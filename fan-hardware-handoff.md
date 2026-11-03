# Brief for a new session — rack fan hardware: 24 V fans, and six of them

You are being handed a **hardware feasibility question**, not a bug. Nobody is
asking you to change firmware yet. Two questions, asked by the owner:

1. **Can 24 V fans be used** instead of the current 12 V ones?
2. **Can one "big" ESP32 drive six fans?**

This file is self-contained: the Rack-Fan firmware is **not** on the Home
Assistant host, so nothing here can be verified by reading that repo. Everything
below was measured from HA's API and by SNMP against the UPSes on 2026-10-16.

---

## 1. The physical setup — read this first, it changes the answer

Two adjacent rooms separated by a **door**:

- a **rack room**, holding the rack
- an **AC room**, cooled by a Faikin/Faikout-controlled split unit
  (`faikout-sottotetto`)

The ESPHome device called **`Rack-Fan` is a transfer fan**. It moves air from the
rack room into the AC room. It is **not** a fan inside the rack, and it is not
the AC's fan. Getting this wrong invalidates most reasoning about it, so state it
back before proposing anything.

**The control split, as decided by the owner** — do not propose changing it
without saying so explicitly:

- the **fans** regulate the **rack room**
- the **AC** regulates the **AC room**
- rack temperature is an **emergency escalation** input for the AC, never its
  normal input

The reason is that the AC cools the AC room, the door makes its authority over
the rack room weak, and letting the AC chase rack temperature would put two
controllers in a fight over coupled variables. The intended coupling is
**physical**: a cooler AC room gives the transfer fan a colder sink.

**The weak link, and it matters for sizing.** The fan's achievable floor is the
AC room temperature. If the AC is off or the AC room is warm, **no fan of any
size fixes the rack room.** A saturated fan PID usually means the sink is too
warm, not that the fans are too small. Keep this in front of you: it is the main
reason "add more/bigger fans" can be the wrong answer.

---

## 2. What is installed today

| | |
|---|---|
| ESPHome device name | `Rack-Fan` |
| Board | **`nodemcu-32s`** (plain ESP32, 38-pin NodeMCU-32S) |
| MAC | `b0:a7:32:db:6e:70` |
| IP | 172.16.80.161 |
| ESPHome version | **2025.4.2** (built 2025-05-15) — old, see §6 |
| Firmware source | fork **`Rocco83/esphome-fan-controller`** of `patrickcollins12/esphome-fan-controller` |
| Temp/humidity sensor | **SHT21**, in the **AC room** |
| Tachometers | **two** inputs wired |
| Control | ESPHome `pid` climate, plus a manual-override fan entity |

The fork already diverges from upstream: SHT21 instead of DHT, a second
tachometer, a web server, and a pinout fix. Expect local changes; do not assume
upstream files match.

Entities in HA:

```
climate.rack_fan_thermostat     modes [off, cool], min 20.0 max 50.0, step 0.1
fan.manual_fan_speed            manual override, percentage 100, currently off
sensor.rack_fan_output_value    PID output, 0-255
sensor.rack_fan_error_value     setpoint error, degC
sensor.rack_fan_p_term / _i_term / _d_term
sensor.rack_fan_fan_speed_1     tach 1, RPM
sensor.rack_fan_fan_speed_2     tach 2, RPM
sensor.rack_fan_is_in_deadband
switch.rack_fan_esp32_restart
```

**Stale duplicates:** a set of `sensor.ac_rack_fan_*` entities exist from an
earlier device name and read `unavailable`. Same single device. Ignore them;
they are on the HA side's cleanup list, not yours.

---

## 3. Live state, 2026-10-16 — the current fans are already saturated

Measured within the same minute:

| reading | value |
|---|---|
| SHT21, AC room (`climate.rack_fan_thermostat` current) | **28.5 °C** |
| PID setpoint | **25.9 °C** |
| PID error | 2.56 °C |
| PID output | **164.9 / 255 ≈ 65 %** |
| `i_term` | **100.0 — pinned at its clamp** |
| `p_term` / `d_term` | 64.9 / 0.0 |
| tach 1 | **1878 RPM** |
| tach 2 | **0 RPM** |
| APC `RackTempUppe_SRT3000` (ups1 idx1) | **33 °C** |
| APC `RackTempLow_SUA750XL` (ups2 idx1) | 29 °C |
| APC `RoomTemp_SUA750XL` (ups2 idx2) | 29 °C |

Two things to take from this:

**The integral is clamped.** `i_term` sits at exactly 100.0, which is its limit,
with a steady 2.5 °C error. The loop has given up: it cannot reach 25.9 °C.
Per §1 the likely cause is the **sink**, not fan capacity — the AC room itself is
28.5 °C. **Before recommending six fans, establish whether the AC was even
running.** Adding fans to a warm sink buys nothing, and this is the single
cheapest way to get the answer wrong.

**Tach 2 reads 0 RPM while tach 1 reads 1878.** Either only one fan is fitted,
or the second fan is dead, or its tach wire is not connected. **Resolve this
before adding four more fans** — it is a one-fan diagnostic that tells you
whether the existing wiring and the second tach input actually work.

---

## 4. Question 1 — 24 V fans

The honest framing: 24 V is a **wiring** decision, not a control decision. Work
out what it actually buys here before any part numbers.

What to settle, in order:

1. **Is there already a 24 V rail?** The house has an alarm panel, a 12 V bus,
   and PoE switches. If a 24 V supply has to be added for this, that is the real
   cost of the idea. Ask.
2. **How long is the cable run** from the supply to the fans? The genuine
   advantage of 24 V is **half the current for the same power**, so half the
   I·R drop and thinner wire. On a short run inside one room, that advantage is
   close to zero.
3. **The PWM and tach interface is the part that must not be guessed.** For
   4-pin fans the control pin is driven by the host at **logic level** and the
   tach is an **open-collector output referenced to fan ground** — on a
   compliant 24 V fan both are still low-voltage signals, so a 3.3 V ESP32 can
   in principle drive and read them directly. **Verify this per datasheet for the
   exact part**, including the maximum voltage the tach transistor tolerates and
   the PWM input's logic thresholds. Do not generalise from a 12 V fan's
   datasheet.
4. **Nominal PWM frequency is ~25 kHz** (the 4-wire fan convention, roughly
   21–28 kHz). Confirm the chosen part's accepted range; some industrial fans
   are fussier than desktop ones.
5. **Grounds.** If the fan supply ground and the ESP32 ground are separate
   returns, the tach reading and the PWM both become unreliable. A single
   star point is the cheap correct answer.

Figures worth checking but **not** worth trusting from memory: Noctua's
industrialPPC line exists in both 12 V and 24 V variants, and the 24 V parts draw
roughly half the current of their 12 V equivalents for similar airflow. Pull the
real numbers from the datasheets rather than quoting these back — the per-fan
current is what drives the supply sizing and the connector choice, so it has to
be right.

**What 24 V does not fix:** the saturated loop in §3. If the sink is the
limit, a 24 V fan is the same fan with different wiring.

---

## 5. Question 2 — six fans on one ESP32

The limit is **not PWM channels.** An ESP32's LEDC peripheral has 16 channels and
25 kHz is comfortably within its range at useful resolution. Six independent PWM
outputs are not the constraint.

The real constraints, in the order they bite:

1. **Tach GPIOs.** Six tachometers need six interrupt-capable GPIOs plus
   pull-ups. On an ESP32 be careful with: input-only pins (GPIO34–39 cannot
   provide a pull-up, so an external resistor is required), strapping pins
   (0, 2, 5, 12, 15), the flash pins (6–11, unusable), and ADC2 (unusable while
   WiFi is on). Count the genuinely free pins on the actual board before
   promising six.
2. **Do you even want six tachs?** Six PWM outputs are cheap; six tach inputs
   cost pins and firmware. If the fans run as one bank, **one PWM output and a
   per-fan tach** is often the right split — control is common, failure
   detection is per-fan. Say which split you are proposing and why.
3. **Power.** Six fans is a supply-sizing and connector question, plus inrush.
   This is where §4's current numbers get used.
4. **Firmware shape.** Six fans as one `output` + six `pulse_counter` sensors
   is straightforward in ESPHome. Six *independent* PID loops is not, and almost
   certainly not wanted — one loop on the rack room with a common output is the
   model that matches §1.

**Consider whether six is even the goal.** Per §1 the fans regulate the rack
room against the AC room as a sink. Six fans raise maximum transfer rate; they do
not lower the floor. If the complaint is the floor, this is the wrong project.
Ask what problem six fans is meant to solve before sizing them.

---

## 6. Two things already on the list, relevant to any rebuild

**Gate the transfer fan on a second sensor.** There is a standing idea, recorded
on the HA side: add a temperature sensor in the **AC room** and run the transfer
fan **only when the AC room is cooler than the rack room**. The fan is a heat
*transfer* device, so its usefulness depends on the **sign of ΔT**; running it
when the sink is warmer actively imports heat, and the present single-sensor PID
cannot tell the difference.

Verified 2026-10-03: this is **not** in `patrickcollins12/esphome-fan-controller`
upstream — it defines a single `dht` and has no differential logic, no related
issue or PR. So it is a genuine addition to the fork. Implement it as a condition
on the PID's **output**, or as a hard interlock on the fan output — **not** by
retargeting the PID, which would fight the control split in §1.

If the hardware is being touched for 24 V or for six fans anyway, **this is the
moment to add that second sensor**, because it needs one.

**ESPHome 2025.4.2 is old.** Any rebuild means an upgrade. Check the fork still
compiles before promising anything, and note that `nodemcu-32s` is a plain ESP32
— if the answer to question 2 is a bigger board, the board change and the
ESPHome upgrade land together, which is two variables at once. Sequence them.

---

## 7. Sensor accuracy, if you need to compare readings

| where | sensor | notes |
|---|---|---|
| AC room | SHT21 on `Rack-Fan` | **authoritative**: ±0.3 °C, two decimals |
| AC room, mid height | APC `RoomTemp_SUA750XL` (ups2 idx2) | cross-check only: ~±1 °C and **integer-only**, so ±0.5 °C quantisation |
| rack upper | APC `RackTempUppe_SRT3000` (ups1 idx1) | the hottest reading in the room |
| rack lower | APC `RackTempLow_SUA750XL` (ups2 idx1) | |

**Co-location experiment, 2026-10-03.** With the APC room probe moved beside the
SHT21, their disagreement fell from ~2.2 °C to a stable **+1.28 °C** (APC reads
high). So ~0.9 °C was genuine positional gradient and ~1.3 °C is instrumental —
and 1.3 °C sits inside the combined tolerance of the two parts, so **neither is
faulty**. Offset versus gain error cannot yet be separated: both sensors have
only ever been seen near 31 °C, and the room's whole 10-day span is 29.6–33.0 °C.
So do not treat the APC probes as a reference, and do not "correct" the SHT21
against them.

The APC probes are **temperature-only**; a 0 % humidity reading from them is by
design, not a fault. On the HA side these three probes are configured but
**await an HA restart**, so they may not exist as entities when you look.

---

## 8. What to report back

Not firmware. A decision document:

1. Whether 24 V is worth it **here**, with the cable run and the existing-rail
   answers that made the case — and say plainly if the answer is "no benefit".
2. A pin budget for the fan count you recommend, on a named board, with the
   unusable-pin caveats of §5 already applied.
3. A current budget and supply recommendation, from datasheets, not memory.
4. Your reading of §3: was the loop saturated because of the sink or because of
   fan capacity? This determines whether the whole project is worth doing.
5. The tach-2-reads-zero finding: one fan or two, and if two, why one is silent.
6. Whether to fold in the second sensor from §6 while the hardware is open.

Flag anything you could not verify rather than filling it in. The owner would
rather hear "needs the datasheet" than a number that turns out to be invented —
a previous session got burned asserting hardware behaviour from a 10-day
recorder window instead of the actual specification.
