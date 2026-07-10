# Actron Wired Controller Replacement — Design Document

Goal: replace the Actron wired controller with an ESP32/ESP8266 running ESPHome, integrated into Home Assistant, that reproduces the exact voltages the indoor unit's control board expects on the button-sense line.

## 1. What your measurements tell us

Your table is a single analog line where each button pulls the voltage to a different level, with a resting ("IDLE") level near the top of the range. That pattern is the signature of a **pull-up + resistor-to-ground ladder**: the indoor unit's control board has an internal pull-up resistor (to roughly 5V) on the sense pin, and the wired controller normally just closes a switch through a different resistor value for each button, dividing that pull-up down to a distinct voltage. With nothing pressed, no resistor is connected, so the line floats up to essentially the full pull-up voltage — that's your 4.982V IDLE reading (the small ~18mV shortfall from a clean 5.000V is consistent with the board's own ADC input leakage current).

This is a well-documented pattern on Actron indoor unit "multi-function" boards. Actron's own wired-controller manuals describe a 4-core connection to this board with wires for **12V/5V supply, E (common/ground), Y (signal), and X (secondary signal)** — so the wire you're measuring is very likely their "Y" line, referenced to their "E" ground. Your specific controller (button set: ON/OFF, FAN, Temp Up/Down, Timer, UP/DOWN, Zone 1–4) looks like a simpler ducted-zone wired sensor rather than Actron's full LCD "WC-02" remote, but the same electrical convention applies. **Confirm your actual harness's pin labels against your board's silkscreen before wiring anything** — don't assume the WC-02 pinout maps 1:1 to your board.

Two things worth flagging before you build:

- **`Vcc / pull-up value is assumed, not measured.`** Everything below assumes the pull-up rail is ~5.00V. Verify this (see §7) before finalizing.
- **`FAN` appears twice** (0.452V and 3.22V). That's either two genuinely different functions your labeling didn't distinguish (e.g., fan-mode cycle vs. fan-speed cycle), or a transcription slip. Re-test both physical buttons and relabel before you commit them to firmware — sending the wrong one won't damage anything, but it will confuse your automations.

## 2. Circuit model

```
   Vcc (~5V, on the indoor unit board)
        │
      [R_pullup]  (internal to indoor unit — value unknown)
        │
        ●────────────────────────► Y (sense line, to ADC on indoor unit board)
        │
   [button's resistor]   (only connected while a button is held)
        │
       GND / E
```

Because you don't know `R_pullup`, you can't cleanly calculate the per-button resistor values from voltage alone (that needs two knowns and you only have one). Rather than reverse-engineer the ladder, the design below sidesteps it entirely.

## 3. Design decision: drive the line actively with a DAC — as a third bus node, not a replacement

Instead of switching resistors, an ESP-controlled DAC (MCP4725, 12-bit, I2C) **actively outputs** the target voltage directly onto the Y line for the length of a "press." A DAC's output impedance (tens of ohms) is 100–1000x lower than a typical pull-up resistor (usually 10kΩ–100kΩ on this kind of board), so it simply overpowers the existing pull-up and sets the node to whatever voltage you command — you don't need to know `R_pullup` at all.

**Revision:** you're keeping one or both of the existing wired controllers connected, so this DAC can't just sit there permanently holding IDLE (or any voltage) the way a full replacement could — see §3a below, which is why this design now includes a switchable connection and a listening ADC input rather than a permanent drive.

### 3a. Why "always drive" doesn't work here — the bus is shared

Your two existing wired controllers stay in sync with each other purely by listening: each one has its own ADC on the Y line, sampling continuously, even when it's not the one pressing a button. When controller A drives a button voltage onto Y, that voltage is visible to *everyone* on the bus — the indoor unit, and controller B — and B decodes it against the same voltage table to update its own state. There's no separate feedback wire because the command line doubles as a broadcast; Actron's own manual confirms dual-controller setups are a designed-for feature (it lists a "main unit and secondary unit indicator" on the display).

If your ESP32 permanently drives the bus (as a straight replacement design would), it would swamp any button press from a controller you've left connected — your DAC's low output impedance beats their resistor network every time, so their commands would never reach the indoor unit, and they'd stop syncing correctly. To coexist as a genuine third node, your controller needs to do the same two things the OEM controllers do: listen continuously, and drive only for the ~300ms of an actual press — never idle-holding the line.

## 4. Hardware

### Bill of materials

| Part | Purpose | Notes |
|---|---|---|
| ESP32 or ESP8266 dev board | You already have one | Any ESPHome-supported board works — I2C is universal |
| MCP4725 breakout (e.g. Adafruit #935) | 12-bit I2C DAC, generates the sense-line voltage | Power its VDD from 5V, not 3.3V, to reach the ~5V range you measured |
| Bi-directional I2C level shifter (e.g. TXS0102 module) | Protects the ESP's 3.3V I2C pins from the MCP4725's 5V-referenced bus | Required — ESPHome's own MCP4725 docs call this out explicitly for VDD > 3.3V |
| Small SPST reed relay (5V coil) | Disconnects the DAC from the bus except during an actual press | A mechanical relay is the simplest way to get a true, leak-free disconnect across the full 0–5V range — cheaper/simpler than an analog-switch IC that would need its own 5V-compatible logic level |
| NPN transistor (e.g. 2N2222) + 1kΩ base resistor + 1N4148 flyback diode | Drives the relay coil from a 3.3V GPIO | Standard low-side relay driver |
| 1x 220–330Ω resistor | Series resistor between the relay and the Y wire | Limits fault current if the line is ever shorted |
| 1x 5.1V (or appropriate) TVS/Zener clamp | ESD/surge protection on the Y line | Cheap insurance since you're tapping into another system's board |
| 1x 100kΩ + 1x 180kΩ resistor | Voltage divider tapping the Y line down to the ESP32's 0–3.3V ADC range | High total impedance (280kΩ) keeps this tap from meaningfully loading the bus |
| Small ceramic cap (~100nF) | Light filtering on the ADC input, GPIO34 to GND | Smooths bus noise without slowing down detection of a ~300ms pulse |
| Jumper wire / connector matching your harness | Physical connection to the indoor unit's Y (KEY) and E (ground) pins | Match your board's actual connector, not assumed from the manual |

### Wiring

1. ESP32/ESP8266 I2C (SDA/SCL) → level shifter low side (3.3V) → level shifter high side (5V) → MCP4725 SDA/SCL.
2. MCP4725 VDD → the **same 5V rail the indoor unit board uses for its pull-up**, if you can tap it from the connector (this is important — see §7/§9). If you can't tap it, use a well-regulated external 5V supply and re-measure/recalibrate the table in §5 against it.
3. MCP4725 VOUT → relay contact (normally open) → 220–330Ω series resistor → indoor unit's **Y / KEY** pin. The relay coil is driven by a GPIO through the transistor driver; the relay is **open (disconnected) by default** and only closes for the ~300ms of a virtual press.
4. Off the same Y/KEY node (on the connector side of the relay, so it reads the true bus voltage regardless of the relay's state), tap the 100k/180k divider down into an ESP32 ADC-capable pin (e.g. GPIO34), with the 100nF cap from that pin to GND.
5. MCP4725 GND, ESP GND, relay driver GND, and the indoor unit's **E** pin → common ground.
6. Leave both existing wired controllers connected exactly as they are — this design is meant to sit alongside them as a third node, not replace them.

## 5. Voltage → DAC level mapping

ESPHome's `output.set_level` for the MCP4725 takes a percentage (0–100%) of VDD (its `output.set_level` reference), not a raw 12-bit code, so that's what you configure in YAML. The percentages below assume VDD = 5.00V — **recompute them once you've measured the real rail (§7)**, since the whole scheme is self-calibrating as long as MCP4725's VDD matches the board's actual pull-up rail exactly.

| Button | Target voltage | Level (%) | 12-bit code (of 4095) |
|---|---|---|---|
| ON/OFF | 0.000 V | 0.00% | 0 |
| FAN (verify which) | 0.452 V | 9.04% | 370 |
| Temp Up | 1.509 V | 30.18% | 1236 |
| Timer | 1.854 V | 37.08% | 1518 |
| UP | 2.234 V | 44.68% | 1830 |
| DOWN | 2.530 V | 50.60% | 2072 |
| Temp Down | 2.872 V | 57.44% | 2352 |
| FAN (verify which) | 3.220 V | 64.40% | 2637 |
| Zone 1 | 3.650 V | 73.00% | 2989 |
| Zone 3 | 3.875 V | 77.50% | 3174 |
| Zone 2 | 4.237 V | 84.74% | 3470 |
| Zone 4 | 4.681 V | 93.62% | 3834 |
| IDLE (default/rest) | 4.982 V | 99.64% | 4080 |

## 6. Behavior / timing model

A real button press isn't a static state — it's IDLE → pressed voltage → back to IDLE. With two other controllers sharing the bus, "back to IDLE" now means **disconnecting**, not driving IDLE yourself — the moment you let go, the indoor unit's own pull-up puts the line back at IDLE on its own, exactly as it does for the OEM controllers.

1. **Listen first.** Continuously sample the Y/KEY line voltage (§7's `key_line_raw` sensor). If it's not at IDLE, something else is already transmitting — wait.
2. **Carrier-sense before transmitting.** Only close the relay once the bus reads IDLE (with a short timeout — if it's never free, give up and report "bus busy" rather than forcing a collision).
3. **Close the relay, set the DAC to the button's level, hold ~300ms.** The indoor unit board almost certainly debounces/samples on some interval, so this just needs to be comfortably longer than one sample period — tune after bench testing.
4. **Open the relay.** Don't drive IDLE — just disconnect, and let the bus's own pull-up take over again.
5. **Queue/ignore new virtual presses while one is in progress**, same as before — two overlapping presses can't produce a voltage that maps to any real button.
6. **Decode what you hear.** Whenever the bus isn't IDLE and it wasn't you driving it, match the voltage against the same table (§5) to figure out which OEM controller/button just fired, and publish that into Home Assistant — this is the "free" state feedback mentioned in §3a: something you couldn't get in a single-controller replacement design.

For ON/OFF specifically, add a software lockout (e.g., 3 minutes) between toggles — most compressors need a minimum off-time before restart, and this protects the equipment regardless of what triggers the toggle (Home Assistant *or* a physical controller you're now also observing).

## 7. ESPHome configuration

This version adds the relay switch, the ADC listening tap, carrier-sense-before-transmit, and a decoded "what's on the bus right now" text sensor. It's schema-validated against the real ESPHome compiler (`esphome config` → `Configuration is valid!`) — the lambda C++ bodies aren't syntax-checked at that stage, so compile and flash it, then iterate; that's expected for embedded work like this, not a sign something's wrong with the plan.

```yaml
i2c:
  sda: 21          # adjust to your board's wiring
  scl: 22
  scan: true

output:
  - platform: mcp4725
    id: dac_output
    address: 0x60   # confirm with the i2c scan log at boot

switch:
  - platform: gpio
    id: bus_relay
    pin: GPIO25      # drives the relay coil via the transistor, see BOM in §4
    inverted: false  # relay energizes = DAC connected to the bus

sensor:
  - platform: adc
    id: key_line_raw
    pin: GPIO34            # analog tap, downstream of the relay (see wiring §4)
    attenuation: 12db       # full 0-3.3V range
    update_interval: 20ms
    name: "Actron KEY Line Voltage"
    filters:
      - multiply: 1.5556    # undoes the 100k/180k divider -> real bus voltage
      - sliding_window_moving_average:
          window_size: 3
          send_every: 1

text_sensor:
  - platform: template
    id: key_line_state
    name: "Actron KEY Line State"
    update_interval: 20ms
    lambda: |-
      float v = id(key_line_raw).state;
      if (std::isnan(v)) return {"UNKNOWN"};
      if (std::fabs(v - 0.000) < 0.10) return {"ON_OFF"};
      if (std::fabs(v - 0.452) < 0.10) return {"FAN_A"};      // confirm which physical "FAN" this is
      if (std::fabs(v - 1.509) < 0.10) return {"TEMP_UP"};
      if (std::fabs(v - 1.854) < 0.10) return {"TIMER"};
      if (std::fabs(v - 2.234) < 0.10) return {"UP"};
      if (std::fabs(v - 2.530) < 0.10) return {"DOWN"};
      if (std::fabs(v - 2.872) < 0.10) return {"TEMP_DOWN"};
      if (std::fabs(v - 3.220) < 0.10) return {"FAN_B"};      // confirm which physical "FAN" this is
      if (std::fabs(v - 3.650) < 0.10) return {"ZONE_1"};
      if (std::fabs(v - 3.875) < 0.10) return {"ZONE_3"};
      if (std::fabs(v - 4.237) < 0.10) return {"ZONE_2"};
      if (std::fabs(v - 4.681) < 0.10) return {"ZONE_4"};
      if (v > 4.80) return {"IDLE"};
      return {"UNKNOWN"};

binary_sensor:
  - platform: template
    id: bus_idle
    name: "Actron Bus Idle"
    lambda: |-
      return id(key_line_raw).state > 4.80;   # comfortably above every button, below true IDLE margin

script:
  - id: press_button
    mode: single      # a press in progress blocks new presses until it finishes
    parameters:
      level: float
    then:
      - wait_until:
          condition:
            binary_sensor.is_on: bus_idle
          timeout: 2s
      - if:
          condition:
            binary_sensor.is_on: bus_idle
          then:
            - switch.turn_on: bus_relay
            - output.set_level:
                id: dac_output
                level: !lambda 'return level;'
            - delay: 300ms
            - switch.turn_off: bus_relay   # disconnect — don't drive IDLE, let the board's own pull-up do it
          else:
            - logger.log: "Bus busy - command not sent"

button:
  - platform: template
    name: "Actron On/Off"
    on_press:
      - script.execute: { id: press_button, level: 0.00% }
  - platform: template
    name: "Actron Fan"
    on_press:
      - script.execute: { id: press_button, level: 9.04% }   # confirm which physical "FAN" this is
  - platform: template
    name: "Actron Temp Up"
    on_press:
      - script.execute: { id: press_button, level: 30.18% }
  - platform: template
    name: "Actron Timer"
    on_press:
      - script.execute: { id: press_button, level: 37.08% }
  - platform: template
    name: "Actron Up"
    on_press:
      - script.execute: { id: press_button, level: 44.68% }
  - platform: template
    name: "Actron Down"
    on_press:
      - script.execute: { id: press_button, level: 50.60% }
  - platform: template
    name: "Actron Temp Down"
    on_press:
      - script.execute: { id: press_button, level: 57.44% }
  - platform: template
    name: "Actron Fan 2"
    on_press:
      - script.execute: { id: press_button, level: 64.40% }   # confirm which physical "FAN" this is
  - platform: template
    name: "Actron Zone 1"
    on_press:
      - script.execute: { id: press_button, level: 73.00% }
  - platform: template
    name: "Actron Zone 3"
    on_press:
      - script.execute: { id: press_button, level: 77.50% }
  - platform: template
    name: "Actron Zone 2"
    on_press:
      - script.execute: { id: press_button, level: 84.74% }
  - platform: template
    name: "Actron Zone 4"
    on_press:
      - script.execute: { id: press_button, level: 93.62% }
```

This gives you 13 button entities plus two live sensors in Home Assistant: `Actron KEY Line State` (what's on the bus right now — including presses from either OEM controller) and `Actron Bus Idle`. Once these work reliably, the natural next step is wrapping the buttons in a proper `climate` component (mapping HA's target-temperature/mode calls into sequences of Temp Up/Down presses) and using `key_line_state` to keep that climate entity's reported state in sync with what the physical controllers are doing — worth doing once the voltages and decode tolerance are validated, not before.

This design is no longer command-only: the `key_line_state` text sensor gives you visibility into button presses from either OEM controller, sourced from the same bus-snooping mechanism they use to sync with each other (see §3a). It still won't tell you the AC's actual running state at boot (temperature, mode) until *something* presses a button — there's no "give me current status" query on this bus as far as your measurements show.

## 8. Bring-up and verification plan

Do this before connecting anything to the live indoor unit:

1. **Measure the real pull-up rail.** With the wired controller unplugged and the indoor unit powered, measure DC voltage between Y and E with a multimeter. Confirm it reads close to your IDLE measurement (4.982V) with nothing connected — this is your true Vcc/reference. Recompute §5's percentages against this exact value.
2. **Measure `R_pullup`** if you can safely do so (power off, meter across Y to the 12V/5V pin in resistance mode after isolating the board from other circuits) — this tells you how much current your 220–330Ω series resistor will draw and confirms it won't meaningfully load the line.
3. **Bench-test the DAC alone.** Power the MCP4725 from a bench 5V supply, run the ESPHome firmware, and verify each button's output with a multimeter before it's anywhere near the AC unit. Check actual output against the target voltages in §5 within a few millivolts.
4. **Confirm I2C level shifting works** — verify with a scope or logic analyzer (or at minimum, confirm `i2c: scan: true` sees the MCP4725 at boot) before trusting it long-term with 5V on the bus.
5. **Verify the listening path before the transmit path.** With both OEM controllers connected and your ESP's relay left open, confirm `key_line_state` correctly reports IDLE at rest and correctly names the button whenever you press something on either physical controller. Get this rock-solid before you ever close the relay.
6. **Verify the relay is actually silent when open** — with it open, confirm on a scope/meter that the bus voltage is completely unaffected by your circuit (no measurable pull from the divider or leakage through the open relay/DAC path).
7. **Connect to the live unit one button at a time**, starting with something low-risk and observable (e.g., a Zone button, since you can visually confirm a damper moves) before testing ON/OFF.
8. **Test collision handling deliberately** — press a physical button at the same moment you trigger a virtual one, and confirm your carrier-sense either defers cleanly or (worst case) fails safe rather than sending a garbled voltage.
9. **Verify compressor lockout behavior** by rapidly toggling ON/OFF from Home Assistant and confirming the firmware lockout prevents short-cycling.

## 9. Open items you need to confirm

- Real Vcc/pull-up rail voltage (assumed 5.00V above).
- Which physical button corresponds to each of the two "FAN" entries.
- Your actual connector's pin labels (don't assume the WC-02 manual's Y/E/X/12V layout applies verbatim to your board — confirm against your board's silkscreen or service manual).
- Whether your indoor unit board's ADC input impedance is high enough that the 220–330Ω series resistor introduces negligible error (should be fine for any typical ADC input, but worth a quick sanity check once you know `R_pullup`).
- The `bus_idle` threshold (4.80V) and per-button decode tolerance (±0.10V) in §7 are starting points based on your measured deltas (smallest gap is 0.225V, so ±0.10V leaves reasonable margin either side) — tune both once you can see real bus noise and timing on a scope.
- Whether the two existing controllers actually implement carrier-sense themselves, or just tolerate rare collisions — you may see occasional missed/garbled commands from them independent of anything you add, since that's a property of their design, not yours.
- Relay switching speed/bounce — reed relays typically settle in a few ms, comfortably inside your ~300ms press window, but worth confirming with your specific part.
