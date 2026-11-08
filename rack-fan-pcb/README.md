# Rack fan controller board

Controller board for the rack's 19" fan tray: six 4-pin PWM fans, up to eight DS18B20 probes, wired Ethernet. KiCad 10 project, through-hole parts only, **manufactured PCB** (it started as a perfboard design).

- Firmware: [`../rack-fan.yaml`](../rack-fan.yaml) + [`../rack-fan-channel.yaml`](../rack-fan-channel.yaml)
- Project overview, fan choice and control logic: [`../README-rack-fan.md`](../README-rack-fan.md)
- This file: every decision taken for the board, why, and what the alternatives were.

## Contents

- [Summary](#summary)
- [Decisions](#decisions)
- [Pin assignment](#pin-assignment)
- [Board layout](#board-layout)
- [Parts](#parts)
- [Library sources](#library-sources)
- [Firmware coupling](#firmware-coupling)
- [Verification status](#verification-status)
- [Open items](#open-items)
- [Sources](#sources)

## Summary

| Area | Decision |
|---|---|
| Controller | WT32-ETH01 (ESP32 + LAN8720), **mounted face down** in two 1 × 12 female sockets |
| Fans | 6 channels (Noctua NF-F12 industrialPPC-24V-3000 planned), each with its own PWM and tach |
| PWM stage | 2N3904 open collector per fan: 4.7 kΩ base, 100 kΩ pull-down, inverting |
| Tach | Direct to a GPIO per fan, 10 kΩ pull-up to 3.3 V, hardware pulse counter |
| Probes | DS18B20 on two 1-Wire buses (GPIO5, GPIO17), 4 JST XH connectors per bus |
| Power in | 24 V, Mean Well GST90A24-P1M (5.5/2.5 jack J5), or an XLG LED driver on J1 |
| Protection | 5×20 T2A fuse (PTF78 holder), 1N5822 reverse-polarity diode, 470 µF bulk |
| Logic supply | Traco TSR 1-2450 (24 V → 5 V) into the WT32's 5V pin; 3.3 V from the WT32 |
| Boot robustness | C3 470 µF on EN; fans fail to 100% (except FAN1, see pin 26) |
| Board | Manufactured PCB, outline drawn by the user on Edge.Cuts (≈ 66.5 × 87 mm), parts on a 0.635 mm grid |
| Grounding | Star ground at the power input; logic ground enters the WT32 through **one** GND pin (WT32 GND and 3V3 pins declared as jumper groups) |
| Mounting | 4 × M3 holes (MountingHole_3.2mm_M3), 4 mm in from the corners, **not connected** |

## Decisions

### 1. Controller: WT32-ETH01, socketed, face down

- **Wired Ethernet** for a rack device: no Wi-Fi dependency.
- **Sockets:** the module plugs into two 1 × 12 female headers, so it can be removed to flash it over serial.
- **Pins 13 (LINK) and 14 (GND) have no socket.** They are the pair at the RJ45 end, and with the RJ45 overhanging the board they fall ~0.5 mm past the edge. The edge can't move out to include them: face down, the RJ45 body must clear the board, and it starts only ~1.7 mm beyond the edge. LINK was unused and pin 14 is one of five GND pins, so nothing is lost. The module's two pins simply hang beside the edge. The pads are removed from the footprint and the pins from the symbol; the other pins keep their numbers.
- **Face down:** the module sits components-towards-the-board. That mirrors its two pin rows, so the project uses its own footprint `rack-fan:WT32-ETH01_Socket_FaceDown`. The RJ45 overhangs the left board edge.
- **Row spacing:** 22.86 mm (9 holes), cross-checked against an independent KiCad footprint ([egnor/wt32-eth01](https://github.com/egnor/wt32-eth01)). Measure the module with calipers before soldering the sockets.
- **Pins left:** Ethernet takes GPIO0, 16, 18, 19, 21, 22, 23, 25, 26 and 27. Only 14 GPIOs remain on the header: 1, 2, 3, 4, 5, 12, 14, 15, 17, 32, 33, 35, 36, 39. That budget drives most of the decisions below.

### 2. Six fans, not eight

- 6 PWM + 6 tach + 2 probe buses use all 14 pins.
- **8 fans would need a tach multiplexer.** A 74HC4052 reads one fan at a time, so RPM would update only every ~10 s. Rejected: instant tach readings were preferred.
- **Other ways to 8 fans, both rejected:**
  - a second microcontroller counting the tachs and reporting over I2C
  - an I2C fan-controller chip (MAX31790, EMC2305), which is surface-mount and has no ESPHome support
- **PCA9685 (on hand): rejected.** Its PWM tops out at 1526 Hz (about 3 kHz with an external clock), and the fans need 21–28 kHz. Its I2C also costs 2 pins, and the PW package is surface-mount.

### 3. PWM stage: 2N3904 open collector

```
GPIO ──┬──[4.7 kΩ]── 2N3904 base           fan's internal pull-up
       │             collector ─────────────────┴── fan PWM wire (pin 4)
   [100 kΩ]          emitter ── GND
       │
      GND
```

Why it's needed:
- **Levels:** the fan pulls its PWM wire up to as much as 5 V, above the ESP32's 3.3 V rating, and 3.3 V does not reach the 4.75 V high threshold of some fans (San Ace).

Why this circuit:
- **Drive:** 3.3 V through 4.7 kΩ gives about 0.56 mA of base current. That saturates the transistor at 0.1–0.2 V for the ≤ 5 mA a fan input sinks, guaranteed at 3.3 V. At 25 kHz, turn-off delay costs about 1% of duty.
- **Inverting:** the firmware sets `inverted: true`, so the output level equals the fan duty.
- **Fail-safe:** GPIO low, floating or unpowered means the transistor is off, the wire floats high, and the fan runs at 100%.
- **Strapping pins:** the 100 kΩ pull-down keeps GPIO2 and GPIO12 low at boot.

Alternatives rejected:

| Alternative | Why not |
|---|---|
| 2N7000 MOSFET | Worst-case gate threshold 3 V, too close to the 3.3 V drive |
| 74LVC06A (open-drain inverter) | Surface-mount only |
| BSS138 level-shifter module | Its 3.3 V-side pull-up holds GPIO2/GPIO12 high at boot (GPIO12 high = no boot) |
| 74AHCT125 5 V buffer | Non-inverting: a dead GPIO stops the fan |
| TXB0108 / TXS0108 | Auto-direction shifters misbehave with the fans' pull-ups |
| ULN2003 / ULN2803 | Darlington saturation ~0.9 V, above the 0.4 V low threshold of some fans |
| Direct GPIO | Over-voltage on the pin; insufficient high level |
| 4.7 kΩ pull-downs on pins with a reset pull-up | Would make those fans fail-safe at boot, but rejected: no resistor value changes. Fixed instead by choosing which pins carry PWM (see pin assignment) |

### 4. Tach inputs: direct, one GPIO per fan

- **Circuit:** open collector, 2 pulses per revolution, one 10 kΩ pull-up per fan to **3.3 V** (never 5 V). The pull-up is required on GPIO35/36/39, which have no internal pull-up.
- **Counting:** a hardware pulse counter per fan (6 of the ESP32's 8), live RPM.
- **Filter:** `internal_filter: 13us` (the hardware maximum, 12.8 µs). It also removes the ~80 ns low glitches that ESP32 erratum GPIO-3.11 puts on GPIO36/39 when the ADC powers on. The firmware doesn't use the ADC; the filter is a safeguard.
- **Not on the module's LED pins:** the WT32 has LEDs on GPIO5 and GPIO17. As a tach input, the LED path would add up to ~1.5 mA, against the Noctua's 2 mA tach rating.

### 5. Temperature probes: DS18B20 on two 1-Wire buses

- **DS18B20 chosen over NTC (both on hand):**
  - NTCs need analog inputs, and the ADC1 pins (32/33/35/36/39) are all used.
  - An I2C ADC (ADS1115) would need 2 more pins.
  - The I2C multiplexer (also on hand) was for SHT21 probes sharing one address; DS18B20s share a bus without it.
- **Two buses, 4 connectors each, odd/even:** J21–J24 are TEMP1, 3, 5, 7 (bus A, ONEWIRE_A) and J25–J28 are TEMP2, 4, 6, 8 (bus B, ONEWIRE_B). The hot zone has probes on both buses, so a bus fault (crushed cable, shorted probe) still leaves a hot-zone reading; it also keeps each star of cables short.
  - Bus A: TEMP1 Rear High, TEMP3 Rear Low, TEMP5 Front Low, TEMP7 Inlet.
  - Bus B: TEMP2 Rear Mid, TEMP4 Front High, TEMP6 Outlet, TEMP8 Other Room.
- **Pins GPIO5 and GPIO17:** the module's LED3/LED4 sit there (3V3 → LED → 1 kΩ → pin). For 1-Wire that's harmless: about 1.3 mA of extra pull-up, well within the DS18B20's 4 mA sink, and the LEDs blink with traffic. Those pins were bad for PWM and tach, so 1-Wire is their best use.
- **Pull-up:** 4.7 kΩ per bus (R31, R32); use 2.2 kΩ for long cables.
- **Connectors:** JST XH 3-pin vertical (B3B-XH-A), pin order 1 GND, 2 DQ, 3 VDD.
  - 5.08 mm screw terminals were the first choice but **did not fit**: four in a row need ~17 holes plus spacing, and only ~18 were free beside the fan resistors.
  - XH needs a crimped plug on each probe cable.
- **Probes and names:** eight probes, named after their connector: TEMP1 Rear High, TEMP2 Rear Mid, TEMP3 Rear Low, TEMP4 Front High, TEMP5 Front Low, TEMP6 Outlet, TEMP7 Inlet, TEMP8 Other Room. Cables: 5 m on TEMP4, 6 and 8, 2 m on the others. ESPHome shows them as `Rack Fan TEMP1 Rear High` … `Rack Fan TEMP8 Other Room`. Each name is bound to a probe address in the firmware, so each probe must go into the connector of its name.

### 6. Power

**Input supply: Mean Well GST90A24-P1M** (24 V, 3.75 A, 90 W desktop adapter).
- True constant-voltage supply, and **no mains wiring** on the project at all.
- The **P1M plug is 5.5/2.5 mm, centre positive** (Mean Well datasheet), so J5 must be a **2.5 mm-pin** barrel jack. The common DC-005 jack comes in both pin sizes with the same body, so the footprint fits either.
- **No jack takes both 2.1 and 2.5 reliably.** A 2.1 pin accepts a 2.5 plug but with loose contact, which isn't acceptable at ~26 W. A 2.5 pin doesn't accept a 2.1 plug.
- In Italy, generic adapters are more often 5.5/2.1. If that's preferred, use the GST90A24-P1J with a 2.1 jack.
- **Alternatives:** Mean Well XLG-100-24-A or XLG-75-24-A LED drivers on **J1** (5.08 mm screw terminal). Below their current setting they act as a 24 V supply; leave the potentiometer at maximum. Their flying mains leads need an earthed enclosure wired by someone qualified. Fit J1 **or** J5, not both.

**Load:** 6 × 4.32 W ≈ 26 W (1.1 A) for the fans, plus ~1.5 W of logic.

On the board:

| Ref | Part | Why |
|---|---|---|
| F1 | 5×20 fuse **T2A** (slow-blow) in a Stelvio-Kontek PTF78 holder | Protects the wiring on a short. The footprint is the same one used in espfreezanz. It replaced an earlier resettable PTC fuse |
| D1 | 1N5822 Schottky (3 A) | Reverse polarity. Drops ~0.5 V, so the fans see ~23.5 V |
| C1 | 470 µF 35 V | Bulk capacitance on the fan rail: fan start-up and PWM current pulses |
| U2 | Traco TSR 1-2450 | 24 V → 5 V, 1 A, input 6.5–36 V, fixed output. Preferred to an LM2596 module, which is adjustable and can be set wrong, and has no grid-friendly footprint |
| C2 | 47 µF 10 V | TSR output capacitor |
| C3 | 470 µF 10 V on **EN** | Boot robustness, see below |

**3.3 V** for the pull-ups and probes comes from the WT32's own regulator (pin 10), well under 50 mA.

**C3 on EN, and why not a bigger capacitor on 24 V instead:**
- Some WT32-ETH01 boards hang at power-up when the supply ramps slowly. The ESP32 leaves reset while its 3.3 V is still rising (adapter → TSR → module regulator).
- A capacitor on EN holds the ESP32 in reset until the supply has settled. 470 µF is the value users reported fixing it, with ~1.5–2 s of extra boot delay ([WLED forum](https://wled.discourse.group/t/wt32-eth01-hangs-on-boot-after-psu-power-up/2937)).
- A bigger capacitor on the 24 V line would make the ramp **slower**, the condition that causes the hang. C1 stays for the fan rail; C3 is a different job.
- A smaller C3 (e.g. 100 µF) fits the same footprint and boots faster, but nobody has reported testing it.

### 7. Board type: manufactured PCB

The board started as a perfboard layout (Eurocard, everything on the 2.54 mm hole grid). It is now a **manufactured PCB**:
- The outline is the rectangle the user drew on Edge.Cuts, about 66.5 × 87 mm.
- Parts sit on a 0.635 mm (25 mil) placement grid instead of the 2.54 mm hole grid. That allows the tighter resistor packing and channel pitch below.

### 8. Grounding: star ground, one WT32 GND pin

- **The concern:** the WT32's five GND pins (2, 9, 11, 14, 23; 14 is not fitted) are all tied to the module's internal ground plane. Linking them again on the board only adds a small parallel path, which is harmless in itself. The real risk is the **fan return current** (up to ~1.1 A, pulsed at 25 kHz) entering the module at one GND pin and leaving at another. The ESP32 and the Ethernet PHY would then sit on a bouncing ground.
- **Decision: star ground** at the power input (J5/J1, C1):
  - **Power branch:** fan returns (J11–J16 pin 1), the 2N3904 emitters and the 100 kΩ pull-downs go straight to the star point.
  - **Logic branch:** WT32, TSR 1-2450 ground, C2, C3, the probe GND and R31/R32 go to the star point as their own branch. That branch enters the module through **one** GND pin.
  - Fan current then never crosses the module.
- **How KiCad accepts it:** the WT32 symbol and footprint declare **jumper groups**: GND (2, 9, 11, 23), 3V3 (3, 10). EN (1, 4) was a group too, but KiCad 10's *Update PCB from Schematic* then split the EN net in two (`/EN` on pin 1, `/EN_1` on pin 4 and its tracks), so it was removed: pin 1 carries a no-connect flag and pin 4 is the routed EN pin. The module ties them internally.
  - KiCad 9/10 then treats each group as connected inside the part, so routing one pin per group satisfies the ratsnest and DRC.
  - Tokens: `jumper_pin_groups` in the symbol, `jumper_pad_groups` in the footprint, as written by KiCad's own file formatter.
  - KiCad 10.0.4 bug #25063 affects only pins defined as a list, not this part.
- **Which GND pin:** route the logic branch to the WT32 GND pin nearest the 5V/3V3 pins (pin 9 or 11), so the module's supply and its return are next to each other.

### 9. Mounting holes

- **What:** 4 × stock `MountingHole:MountingHole_3.2mm_M3` (same footprint as espfreezanz), centres 4 mm in from each corner of the outline.
- **Not connected, deliberately.** Metal holes tied to GND would make the chassis a second ground path, a real ground loop.
- **No schematic symbol:** they are board-only footprints (H1–H4).

### 10. Serial

- UART0 (GPIO1/GPIO3) carries FAN1's PWM and tach, so **there is no serial console at runtime**. Logs come over the API and the web server.
- **Flashing:** over the network after the first flash. For a serial flash (first flash, or recovery) use the three headers below, or take the module out of its sockets.
- **J2 UART** (1 × 3, 2.54 mm): 1 GND, 2 RX = IO3/RXD0 (net TACH1), 3 TX = IO1/TXD0 (net PWM1). Labels are from the ESP32's side: the adapter's TX goes to pin 2, its RX to pin 3. Use a **3.3 V** adapter and **unplug FAN1** first, since its tach would drive RX. The PWM1 stage (4.7 kΩ into Q1's base, 100 kΩ pull-down) only loads TX lightly.
- **J3 RESET** (1 × 3): 1 EN, 2 GND, 3 not connected. A jumper on 1-2 holds the ESP32 in reset; pin 3 is the parking place (jumper on 2-3). Shorting EN also discharges C3 (470 µF) through the jumper, which is harmless at 3.3 V.
- **J4 BOOT** (1 × 3): 1 IO0, 2 GND, 3 not connected. Jumper on 1-2 at power-on (or while releasing RESET) = serial download mode; park it on 2-3 for normal use.
- **IO0 is the Ethernet 50 MHz clock input.** The track to J4 is a stub on that clock: keep it **as short as possible**, J4 right next to U1 pin 24, and never leave the BOOT jumper on in normal use. A long IO0 track can break Ethernet.
- **Flash procedure:** FAN1 unplugged, adapter on J2, BOOT jumper on 1-2, power on (or RESET jumper on 1-2 and off), flash, BOOT jumper back to 2-3, reset.
- **Placement:** J2–J4 were added outside the board outline (right side), with nets assigned; place and route them by hand.

## Pin assignment

### Rules that decide every pin

From the [ESP32 datasheet](https://www.espressif.com/sites/default/files/documentation/esp32_datasheet_en.pdf) (strapping pins, IO_MUX reset states) and the [WT32-ETH01 V1.4 schematic](https://github.com/egnor/wt32-eth01):

**Must-haves:**
- **GPIO12 (MTDI)** must be low at boot, or the flash gets 1.8 V and the board won't boot. It can only carry a pulled-down PWM.
- **GPIO2** must be low or floating to enter serial download mode. It can only carry a pulled-down PWM.
- **GPIO35/36/39** are input-only with no pull resistors, so they can only be tachs.
- **GPIO1 (U0TXD)** carries the ROM boot message and is pulled up at reset. It can't be a tach (driven at boot).
- **GPIO0** is the Ethernet clock: unusable.

**What makes a pin good or bad for a job:**
- **PWM is only fail-safe** on pins that are low or floating during reset: GPIO2, GPIO4, GPIO12 (internal pull-down) and GPIO32, GPIO33 (no pull). That's only **5 pins for 6 PWMs**. The 6th PWM necessarily sits on a pin that switches its fan off during reset and boot.
- **Pins with an internal pull-up at reset:** GPIO3, GPIO5, GPIO14, GPIO15 (and GPIO1). Harmless for a tach or 1-Wire; for PWM, the fan stops while the ESP32 is in reset.
- **Module LED pins (GPIO5, GPIO17):** good for 1-Wire. Tight for a tach (current), not fail-safe for PWM.
- **Routing:** the face-down mount swaps the rows. Assignments were chosen to keep each signal on the row nearest its destination, to avoid crossings between the two rows.

### Final assignment

Pins 14–26 are one header row, 1–13 the other. "Pin" is the WT32 header pin, which is also the footprint pad.

| Pin | GPIO | Function | Why this pin | Alternatives for this pin | Caveat |
|---|---|---|---|---|---|
| 26 | IO1 | **PWM1** | Closest pin to FAN1. It couldn't be a tach, and moving 1-Wire to the LED pins freed it | 1-Wire. Not a tach | FAN1 stops for ~1 s at each boot/OTA reboot (reset pull-up, ROM message, then idle high until `on_boot` sets 100%), and stays stopped if the ESP32 is stuck rebooting or held in reset. Accepted: the 6th PWM has to be on such a pin |
| 25 | IO3 | **TACH1** | Pull-up at reset = tach idle level. RX is unused at runtime | PWM (fan would stop during reset), 1-Wire | None |
| 24 | IO0 | **BOOT (J4)** | Ethernet clock input; strapping pin (low = download mode) | — | Only a very short track to J4 |
| 23 | — | GND | | | |
| 22 | IO39 | **TACH2** | Input-only: tach is the only use | Another tach only | Erratum GPIO-3.11 glitch removed by the filter |
| 21 | IO36 | **TACH3** | Input-only: tach is the only use | Another tach only | Erratum GPIO-3.11 glitch removed by the filter |
| 20 | IO15 | **TACH4** | Strapping pin (boot log) whose default is high; the tach pull-up keeps it there | 1-Wire. PWM would stop its fan during reset (pull-up) and silence the boot log | Hobby sources claim a short pulse burst at boot; Espressif documents output disabled (see open items) |
| 19 | IO14 | **TACH5** | Pull-up after reset = tach idle level | 1-Wire. PWM would stop its fan during reset | Same undocumented boot-burst claim |
| 18 | IO12 | **PWM2** | Must stay low at boot: only a pulled-down PWM qualifies | Another PWM only. Never a tach or 1-Wire (pull-up → no boot) | None while the 100 kΩ pull-down is fitted |
| 17 | IO35 | **TACH6** | Input-only: tach is the only use | Another tach only | None |
| 16 | IO4 | **PWM3** | Pull-down at reset: fail-safe PWM | Tach, 1-Wire | None |
| 15 | IO2 | **PWM4** | Must stay low/floating for serial download: only a pulled-down PWM qualifies. Pull-down at reset: fail-safe | Another PWM only. Never a tach or 1-Wire | None |
| 14 | — | GND (no socket) | | | Past the board edge, not fitted |
| 1 | EN | — | Same signal as pin 4 | | Unconnected |
| 2 | — | GND | | | |
| 3 | 3V3 | — | Same rail as pin 10 | | Unconnected |
| 4 | EN | **EN** | C3 to GND | | Adds ~1.5–2 s of boot delay |
| 5 | IO32 | **PWM5** | No pull at reset: fail-safe PWM | Tach, 1-Wire | None |
| 6 | IO33 | **PWM6** | No pull at reset: fail-safe PWM | Tach, 1-Wire | None |
| 7 | IO5 | **ONEWIRE_A** | Module LED3: the best use is 1-Wire. Strapping pin whose default is high, kept by the bus pull-up | Tach (tight: LED current against the 2 mA tach rating). PWM would stop its fan during reset (pull-up + LED) | LED3 blinks with bus traffic |
| 8 | IO17 | **ONEWIRE_B** | Module LED4: the best use is 1-Wire | Tach (tight, as above). PWM would stop its fan during reset (LED pull-up) | LED4 blinks with bus traffic |
| 9 | — | GND | | | |
| 10 | 3V3 | **3.3V** | Supply for pull-ups and probes | | < 50 mA |
| 11 | — | GND | | | |
| 12 | 5V | **+5V** | From U2 (TSR 1-2450) | | |
| 13 | LINK | — (no socket) | | | Past the board edge, not fitted |

Result:
- Every fan except FAN1 runs at 100% from power-on until the firmware takes over, and if the ESP32 is in reset.
- There's no strapping conflict.
- The module LEDs only blink on the probe buses.

**How the assignment evolved**, for the record:
1. **First pass:** probe buses on GPIO1/GPIO3, tachs partly on GPIO5/GPIO17.
2. **Second pass:** after the module was turned face down, the assignment was rearranged for routing. FAN5/FAN6 went to the short row, and each fan's PWM and tach went on the row nearest its connector.
3. **Final:** the reset-state analysis and the WT32 schematic (LEDs on GPIO5/GPIO17) moved 1-Wire to the LED pins and the pull-up pins (IO3, IO14, IO15) to tachs. That left only the unavoidable 6th PWM on GPIO1, which is closest to FAN1. A swap of ONEWIRE_B onto GPIO14 was considered and dropped, because it moved signals between rows.

## Board layout

- **Outline:** the user's Edge.Cuts rectangle (≈ 66.5 × 87 mm).
  - The fan headers sit on the right edge.
  - The RJ45 overhangs the left edge on purpose.
  - The barrel jack J5 extends past the top edge (see open items).
- **Grid:** 0.635 mm (25 mil) placement grid. The part's pin 1 is on the grid; pins with their own pitch (XH 2.50 mm, fuse holder 22.6 mm, C1 5.00 mm) follow from it.
- **Fan channels:** one column, **FAN1 at the top of the board**, pitch **11.43 mm**. The fan header's housing is 11.1 mm, so they don't touch. Every channel copies the user-defined channel-6 pattern, aligned:
  - **fan header** (pad 1 at the top: GND, +24V, TACH, PWM)
  - **2N3904** to its left, its **bottom pin (collector) level with the fan header's bottom pin (PWM)**
  - **three resistors packed together** to the left, rows **3.175 mm** apart (0207 courtyard 3.0 mm, so no overlaps), the bottom row level with the fan header's bottom pin:
    - 100 kΩ pull-down (top)
    - 4.7 kΩ base (middle)
    - 10 kΩ tach pull-up (bottom)
  - the three resistors share one column (pad 1 x), the transistors another, the fan headers another
- **TEMP1–TEMP8 (JST XH):** two rows of four at equal spacing (9.525 mm), rows 12.07 mm apart, bottom-left.
- **R32, R31, C3:** one row under the WT32. C3 sits just clear of the module outline.
- **Power:** top-left (J1/J5, F1, D1, C1, U2, C2), snapped to the grid without changing the user's arrangement.
- **Silkscreen:**
  - connectors show their function: FAN1–FAN6, TEMP1–TEMP8, 24V IN
  - **back silkscreen (B.SilkS, mirrored):**
    - **the value of every part:** resistors, capacitors, Q1–Q6 (2N3904), D1, U1, U2 and F1. For F1 and the connectors, whose value is printed on the front, it's a board text copy, so their footprints still match the library.
    - **the use of every connector pin:** FAN1–FAN6 GND / +24V / TACH / PWM; TEMP1–TEMP8 GND / DQ / 3V3; J1 + / GND.
    - checked: no back text overlaps another, a pad or the board edge
    - pin labels are 0.7 mm high, values 0.8 mm. The board's minimum text height rule is set to **0.7 mm** to match (it was 0.8 mm). Check that your fab accepts 0.7 mm silkscreen text; if not, enlarge the pin labels instead.
  - reference designators are on the fabrication layer
  - "LOW" (above FAN1) and "HIGH" (below FAN6) mark the zones, so the tray is mounted with FAN1 at the bottom
- **Tracks:** routed by the user (238 segments, both layers). KiCad 10 DRC: no errors, warnings only.
- **Net classes** (project file):

  | Class | Nets | Track | Clearance | Why |
  |---|---|---|---|---|
  | Power | +24V, GND, VIN_RAW, VIN_FUSED | 1.0 mm | 0.2 mm | ~1.1 A of fan current (6 × 0.18 A), more under a fault before the T2A fuse opens. 1.0 mm on 1 oz copper carries ~2.4 A at 10 °C rise (IPC-2221); 0.2 mm would run ~24 °C warmer at 1.1 A |
  | Supply_5V | +5V | 0.5 mm | 0.2 mm | WT32 supply, ~0.3 A |
  | Default | everything else | 0.2 mm | 0.2 mm | Signals, 3.3 V (< 50 mA) |

- **Existing tracks were widened** to their class width where clearance allows. Clearance stays ≥ 0.2 mm and the edge distance ≥ 0.5 mm:
  - **1.0 mm:** 41 GND and 19 +24V segments, and all VIN_RAW/VIN_FUSED.
  - **0.5 mm:** the per-fan +24V branches (0.18 A each, fine).
  - **0.2–0.5 mm:** the probe-side GND, which carries only milliamps, and part of +5V. Both are fine.
- **Three GND bottlenecks** on the fan return path still need a small reroute (see open items).


## Parts

| Ref | Value / part | Footprint |
|---|---|---|
| U1 | WT32-ETH01, face down, 2 × 1×12 female sockets (no 13/14) | `rack-fan:WT32-ETH01_Socket_FaceDown` |
| U2 | Traco TSR 1-2450 | `Converter_DCDC:Converter_DCDC_TRACO_TSR-1_THT` |
| J1 | 24V IN screw terminal (XLG option) | `TerminalBlock_Phoenix:…MKDS-1,5-2-5.08…` |
| J5 | 24V IN barrel jack 5.5/2.5 (GST90A24-P1M), pin 3 = switch, unused | `Connector_BarrelJack:BarrelJack_Horizontal` |
| F1 | Fuse T2A 5×20 + PTF78 holder | `Fuse:Fuseholder_Cylinder-5x20mm_Stelvio-Kontek_PTF78_Horizontal_Open` |
| D1 | 1N5822 | `Diode_THT:D_DO-201AD_P12.70mm_Horizontal` |
| C1 | 470 µF 35 V | `Capacitor_THT:CP_Radial_D10.0mm_P5.00mm` |
| C2 | 47 µF 10 V | `Capacitor_THT:CP_Radial_D5.0mm_P2.50mm` |
| C3 | 470 µF 10 V | `Capacitor_THT:CP_Radial_D6.3mm_P2.50mm` |
| J11–J16 | FAN1–FAN6, 4-pin PWM fan header | `Connector:FanPinHeader_1x04_P2.54mm_Vertical` |
| Q1–Q6 | 2N3904 | `Package_TO_SOT_THT:TO-92_Inline_Wide` |
| R1–R6 | 4.7 kΩ (base) | `Resistor_THT:R_Axial_DIN0207_L6.3mm_D2.5mm_P7.62mm_Horizontal` |
| R11–R16 | 100 kΩ (pull-down) | same |
| R21–R26 | 10 kΩ (tach pull-up) | same |
| R31, R32 | 4.7 kΩ (1-Wire pull-ups, bus A/B) | same |
| J2–J4 | UART, RESET, BOOT: 1 × 3 pin headers | `Connector_PinHeader_2.54mm:PinHeader_1x03_P2.54mm_Vertical` |
| J21–J28 | TEMP1, 3, 5, 7, 2, 4, 6, 8 (odd bus A, even bus B), JST XH 3-pin (B3B-XH-A) | `Connector_JST:JST_XH_B3B-XH-A_1x03_P2.50mm_Vertical` |
| H1–H4 | M3 mounting holes, not connected (board-only) | `MountingHole:MountingHole_3.2mm_M3` |

Off the board: the fans, the GST90A24-P1M, the DS18B20 probes with JST XHP-3 plugs, a 3.3 V USB-serial adapter for the first flash. The full BOM with order status is in [`../README-rack-fan.md`](../README-rack-fan.md#bill-of-materials).

## Library sources

Every symbol and footprint comes from the **stock KiCad libraries**, with one exception:

| Ref | Library item | Stock? |
|---|---|---|
| **U1** | symbol `rack-fan:WT32-ETH01`, footprint `rack-fan:WT32-ETH01_Socket_FaceDown` | ⚠️ **No.** KiCad's official libraries have no WT32-ETH01, checked in the current kicad-symbols and kicad-footprints repositories (`RF_Module`). Both items live in this project's own library (`rack-fan.kicad_sym`, `rack-fan.pretty/`), shipped with the project. The pad layout was cross-checked against an independent footprint (egnor/wt32-eth01) |
| All other parts | `Device`, `Connector`, `Connector_Generic`, `Motor`, `Diode`, `Regulator_Switching`, `Transistor_BJT`, `power` symbols; `Capacitor_THT`, `Connector`, `Connector_BarrelJack`, `Connector_JST`, `Converter_DCDC`, `Diode_THT`, `Fuse`, `MountingHole`, `Package_TO_SOT_THT`, `Resistor_THT`, `TerminalBlock_Phoenix` footprints | Yes |

How the check was done:
- Every library item was compared against the stock libraries shipped with KiCad 9.0.2. All items exist there under the same names.
- KiCad 10 keeps these libraries, but if KiCad 10 reports a missing library item when opening the project, that's the place to look.

**Symbols changed to match the parts** (footprints, pin numbers and nets unchanged):
- J5: `Connector:Barrel_Jack` → `Connector:Barrel_Jack_Switch`, so the symbol has the footprint's third pad (switch contact, not connected).
- J21–J28: `Connector:Screw_Terminal_01x03` → `Connector_Generic:Conn_01x03`, since they're XH headers now.
- J11–J16: `Connector_Generic:Conn_01x04` → `Motor:Fan_4pin`, which shows the fan with pins −, +, Tacho, PWM.

**U1 stays custom (decision):** the project's own symbol and footprint keep the module outline, the pin names and single-part placement.

**Stock alternative for U1 (not applied):** two `Connector_Generic:Conn_01x13` symbols on `Connector_PinSocket_2.54mm:PinSocket_1x13_P2.54mm_Vertical` footprints, placed 22.86 mm apart and mirrored for the face-down mount. That would make the project 100% stock, at the cost of the outline, pin names and single-part placement.

## Firmware coupling

- **Target:** the latest ESPHome (2026.10.1) on the **ESP-IDF** framework. Compiled successfully; not yet run on hardware.
- **Pin map:** matches the table above. The `packages:` entries in `rack-fan.yaml` carry a comment with the WT32 pin numbers.

Workarounds the board relies on:
- **`on_boot` (priority 700):** all fans to 100% immediately. ESPHome creates the LEDC outputs at level 0, which with `inverted: true` switches the transistors on and would stop every fan until the first control update.
- **`CONFIG_ESP_CONSOLE_NONE` and `CONFIG_BOOTLOADER_LOG_LEVEL_NONE`:** keep ESP-IDF's messages off GPIO1 (FAN1's PWM). Only the short ROM boot message remains, because pin 20 holds the strapping pin high.
- **`logger: baud_rate: 0`:** UART0 is used by FAN1.
- **`internal_filter: 13us`** on every tach.
- **Strapping warnings:** `ignore_strapping_warning` on GPIO2/12 (PWM), GPIO15 (tach) and GPIO5 (1-Wire). The ESPHome warning on GPIO0 is the Ethernet clock: expected.

## Verification status

**Done:**
- Every pin against the ESP32 datasheet (strapping, IO_MUX reset states), errata and the WT32-ETH01 V1.4 schematic.
- Schematic and board agree on all 150 pins, with zero mismatches.
- No shorts; minimum clearance 0.2 mm (project rule) after widening; tracks ≥ 0.5 mm from the edge.
- KiCad 10 DRC run by the user: no errors, warnings only (before widening).
- **Library match:** J1 and F1 graphics replaced with the KiCad 10.0.8 library versions (fixes the DRC warning "does not match copy in library"). Pads, nets and positions are unchanged.
- **Courtyard overlaps:** only the 4 mounting holes against parts at the corners (see open items). The resistor overlaps are gone with the 3.175 mm packing.
- **Outline:** U1's RJ45 overhangs the left edge on purpose. J5 extends past the top edge.
- **WT32 jumper groups:** present in the schematic, the board and both project-library files.
- Firmware validated and compiled with ESPHome 2026.10.1 / ESP-IDF.

**Not done here:**
- **KiCad's own ERC and DRC.** The files were edited as text from a machine with KiCad 9, which can't open KiCad 10 files. Run both in KiCad 10 after opening.
- **Hardware test.**

## Open items

- **Ordered** from AISLER on 2026-11-03 (project ID `ESREGXHU`, in the board's title block). The exact files sent are in `production/` (Gerber zip, BOM, positions, IPC netlist).

- **Test rig:** [`../test-rig/`](../test-rig/README.md): 2-fan breadboard rig and firmware to check every PWM and tach pin before the PCB is built. Temporary; remove after the test.
- **GND bottlenecks** (fan return, ~1.1 A), each needing a small reroute:
  - **Between U2 pin 2 (GND) and pin 3 (+5V)** (B.Cu, y ≈ 58.0 mm): 0.2 mm, limited by pin 3. Routing the trunk *through* U2 pin 2 (it's GND) instead of between the pins leaves room for 1.0 mm.
  - **From J5 towards J1/C1** (B.Cu, y ≈ 54.3 mm): 0.2 mm, limited by J1 pin 1 (VIN_RAW). This is the input return when J5 is used. Move it away from J1 pin 1 or onto F.Cu.
  - **Right-edge GND chain between the fan headers:** 0.3–0.4 mm, between the headers' peg holes and the 0.5 mm edge rule. It carries the sum of all fans. Alternative: a GND bus left of the headers (between the Q and J columns) at 1.0 mm, with short stubs to each header's pin 1.
  - **A GND fill on B.Cu** would also solve all three, but it ties the probe-side GND to the fan-side GND across the board instead of only through the module (see grounding decision).
- **Mounting holes collide with parts** at 4 mm from the corners. The user will adjust by hand:
  - H1 top-left vs J5 (0.33 mm)
  - H2 top-right vs U2 (0.63 mm)
  - H3 bottom-left vs TEMP5/J25 (2.5 mm)
  - H4 bottom-right vs FAN6/J16 (2.6 mm)

  Options: move the parts, grow the outline, or move the holes along the edges.
- **J5 past the top edge:** confirm it's intended (jack mouth flush with or beyond the edge).
- **Measure the WT32 row spacing** (22.86 mm expected) before soldering the sockets.
- **GPIO14/15 boot pulses:** hobby sources claim them; Espressif documents the pins as not driven. Low risk for TACH4/TACH5. A logic-analyser capture of a bare module at power-up would settle it.
- **C3 value:** 470 µF is the reported fix. Smaller values are untested.
- **DS18B20 offsets:** calibrate (addresses are set).
- **Noctua NF-F12 24V-3000:** confirm whether it stops at 0% duty.
- **J2–J4 (UART, RESET, BOOT):** placed outside the outline; place and route them, IO0 track as short as possible. Back-silkscreen pin labels to be added once placed.
- **F1:** pad rotation reset to 0 to match the KiCad library.

## Sources

- [ESP32 datasheet v5.3](https://www.espressif.com/sites/default/files/documentation/esp32_datasheet_en.pdf): strapping pins, IO_MUX reset states
- [ESP32 errata](https://docs.espressif.com/projects/esp-chip-errata/en/latest/esp32/03-errata-description/index.html): GPIO-3.11
- [WT32-ETH01 V1.4 schematic and footprint reference (egnor/wt32-eth01)](https://github.com/egnor/wt32-eth01)
- [WT32-ETH01 datasheet (Seeed)](https://files.seeedstudio.com/products/102991455/WT32-ETH01_datasheet_V1.1-%20en.pdf)
- [WT32-ETH01 hangs on boot after PSU power-up (WLED forum)](https://wled.discourse.group/t/wt32-eth01-hangs-on-boot-after-psu-power-up/2937)
- [WT32-ETH01 GPIO12 problem (WLED forum)](https://wled.discourse.group/t/wt32-eth01-gpio12-problem/6566)
- [Mean Well GST90A datasheet](https://www.meanwellaustralia.com.au/sites/default/files/2024-12/MEAN%20WELL%20GST90A%20DATA%20SHEET.pdf): P1M plug 2.5 × 5.5 mm, centre positive
- [Noctua PWM specifications white paper](https://noctua.at/pub/media/wysiwyg/Noctua_PWM_specifications_white_paper.pdf)
- [ESPHome source (release branch)](https://github.com/esphome/esphome/tree/release/esphome/components): `ledc`, `pulse_counter`, `logger`
