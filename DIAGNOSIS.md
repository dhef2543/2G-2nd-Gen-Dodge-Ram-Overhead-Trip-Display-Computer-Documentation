# Overhead console (CMTC — Compass Mini-Trip Computer) — no-save fault diagnosis

> Module type: Chrysler **CMTC**. The EEPROM holds compass calibration
> (hard/soft-iron coefficients + the 1–15 magnetic variance zone), user prefs,
> CCD config, and the trip accumulators — and the compass cal is rewritten
> **continuously while driving**, so those cells see far more write cycles than
> key-off alone. That heavy duty cycle is consistent with the worn-cell failure.

## Symptom (from README)
Boots to a fixed ghost state (≈1723 mi trip, 19.7 avg mpg, 68:53 ET). Values can
be reset and work normally while the truck runs, but revert to that exact state
after a key-off / key-on cycle. Truck has ~333k miles.

## Board inventory (CONFIRMED after VFD removal)
Board # **P14213-1A, REV H**; module label ≈ **56050349**. VFD is `2-BT-2716N`
(getter still silvered after removal → vacuum intact, display should survive).

- **MCU:** **NEC μPD78214GCK-07** (78K/II 8-bit, mask ROM; customer code
  `85-9262-7`, date `9808HP006`). Firmware is in mask ROM — not reprogrammable —
  but the trip *data* lives in the external EEPROM below, which is what matters.
- **EEPROM (the suspect): Microchip `93LC56A-I/SN`** — Microwire 3-wire serial
  EEPROM, **2 Kbit = 256 bytes, x8** ("A" = fixed byte organization), 2.5–5.5 V,
  industrial, SOIC-8. Sits between the MCU and the VFD driver.
- **VFD driver:** **OKI `7272B04-2`** (QFP).
- **Aux:** wide SOIC **`04833637 / 8042 / 9809h`** (Chrysler house number,
  likely bus/interface or glue logic); **`M 10.0G`** 10.0 MHz clock; LM2574M-ADJ
  switcher + LM2931/LM2940 LDOs; RA1032AF driver arrays.

### 93LC56A pin map (verified against the board)
Pins 1–4 (all to accessible back-side vias / MCU): **CS, CLK, DI, DO**.
Pins 5–8: **VSS (gnd), pin6→gnd, pin7→MCU control line, VCC (bypassed)**.
Matches the Microwire signature exactly (4 signal lines one side, power/config
the other). Note **pin 7 routes to the NEC MCU**, so a write failure *could*
sit on the MCU/control side, not the chip — the write-test (below) distinguishes.

## What the symptom tells us
Booting to a *specific, non-trivial* state (not blanks/zeros) means the module
**is** reading real non-volatile data at startup, and that snapshot is stuck.
That argues **against** a simple keep-alive-RAM loss (which comes up default) and
**for** one of:
- (A) a **write-failed / write-locked non-volatile store** (the README's theory), or
- (B) the **save is never triggered** (shutdown-detect or constant-power path
  broken), so the NVM faithfully holds an old good save.

(A) and (B) look identical from the driver's seat but need different fixes. A
chip swap only helps for (A).

## Status
1. **Power — confirmed present.** The module receives constant key-off B+ straight
   from the battery, so the "dead keep-alive feed" branch (B, power side) is ruled
   out. Fault is the 93LC56A EEPROM **or** the MCU's save/control path (pin 7).
2. **NVM — confirmed discrete:** the `93LC56A` (not MCU-internal), so a chip
   swap is physically possible. VFD is off; the chip is fully accessible.

## Repair path (do the write-test before declaring victory)
Read + **write-test** the 93LC56A — same effort as a read, but decisive:
- Off-board is cleanest now that it's exposed: desolder the 8-pin, read/write on a
  programmer (XGecu T48/T56 or TL866-II; select **93LC56, x8**; or bit-bang
  Microwire off an Arduino). No MCU contention that way.
- In the dump, look for the frozen packet — **1723 = 0x06BB**, **197 (=19.7×10) =
  0x00C5** — to prove the chip holds the ghost state.
- **Write a byte, read it back:**
  - won't stick → worn/dead cells → fit a fresh **`93LC56A-I/SN`** (match the
    "A"/x8 org — a `93LC56B` is x16 and will misalign the MCU's transactions).
    Blank is fine: the MCU re-inits trips to defaults on next run.
  - writes fine → the chip is innocent; the fault is the MCU save routine or the
    pin-7 control line, and a swap would re-freeze after one drive.

## Confirmed from the EEPROM dump + repair path
- Dump reads cleanly; **trip miles = 1723** (LE16 @ 0x8E, whole miles, cross-checked
  by the 68:53 ET → 25 mph avg). A VIN-like ASCII string sits @ 0xCF (public web
  says the CMTC doesn't store VIN; this rev appears to anyway — immaterial here).
- **"Write same image, verify OK" proves nothing** — a stuck cell holding its
  current value passes verify because no bit had to change. The real test is a
  **diff-write**: program the bitwise complement (or 00/FF/AA/55 passes) and
  verify. Only a bit forced to flip exposes a worn cell.
  - diff-write **fails** → cells worn → chip confirmed bad → replace.
  - diff-write **passes clean** → chip is fine; fault is MCU-side (pin-7 control
    line / save routine) → a swap won't help; use a good used console instead.
- **Replace by CLONING this dump onto the new 93LC56A — do not install a blank.**
  Blank loses compass calibration (stuck reading until CAL re-run) and risks a CCD
  handshake gripe. Cloning carries cal + config across; the stale 1723 trip data
  overwrites itself on the first key-off once the healthy chip can write.
  Re-run compass CAL + set the variance zone only if it misbehaves.

## Notes / fallback
- Most likely mechanism is **worn cells**, not a "permanent read-only lock" (that
  is largely a myth). The trip bytes are rewritten every key-off (and likely
  periodically while running); over 333k miles that easily exceeds the older
  part's write endurance, leaving those cells stuck at the last value that
  programmed — the 1723 ghost. Effectively permanent → replacement fixes it.
- A **blank replacement EEPROM just resets trip A/B, avg mpg, ET, prefs to
  defaults** (acceptable). It does **not** hold the master odometer (cluster/PCM).
- If you're going in anyway, **socket the EEPROM** so any future access is a
  plug-pull, not another VFD removal.
- Fallback: at 333k miles a good used console (part 56050349 / board P14213) is
  cheap insurance if the write-test points at the MCU side.

## Notes / fallback
- The overhead trip computer stores trip A/B, avg mpg, ET, and user prefs — a
  **blank replacement EEPROM just resets those to defaults** (acceptable). It
  does **not** hold the master odometer (that lives in the cluster/PCM).
- The trap: "EEPROM faithfully holds an old save because the write is never
  issued" mimics a dead chip — reading the chip is the ~zero-cost test that
  tells (A) from (B) before any desoldering.
- At 333k miles, a good used overhead console (cross-ref part 56050349 / board
  P14213) is a cheap pragmatic fallback if the memory turns out MCU-internal.

## UPDATE — chip exonerated; fault is the key-off save path (not the memory)
The 93LC56A **bench-tests good**: on the XGecu (T48) it erases, writes, and
verifies repeatedly, including a full bitwise-complement diff-write (all 2048
bits forced to flip) — so every cell can change state. The earlier "wrote the
same image, verify OK" was inconclusive; the diff-write is what cleared it.

Therefore the no-save fault is **not the EEPROM** — it is in the module's
key-off save path. Constant power is present (12.56 V at the connector via the
IOD fuse), so it is not the always-on feed either.

Working theory — the trip record is committed **at key-off**, and each 93LC56
byte-write takes ~6 ms (tWC), so saving the record needs the logic rail held up
~100+ ms after the key drops. Suspects, in order:
1. **Logic/5 V reservoir (hold-up) cap** degraded → rail collapses before the
   shutdown write finishes → save truncated → reverts. (NOTE: the leaking orange
   cap is on the **VFD** boost rail per trace-out, so it is *not* this — replace
   it for the leak, but the hold-up cap is a different one on the logic supply.)
2. **Ignition-sense / power-down-detect input** to the MCU — the "save now"
   trigger. If its divider/transistor failed, the MCU never starts the save.
   Probe it while cycling the key; it should transition.
3. **EEPROM control-line continuity** (CS/CLK/DI + pin-7), MCU→chip — especially
   after the VFD-removal heat. Reads use CS/CLK/DO, so a cracked **DI**/pin-7
   joint blocks writes while reads still work. Reflow + buzz out.

The NEC μPD78214 MCU is an unlikely cause: every other function (display,
compass, CCD, live trip updates) works, so the core/firmware/I-O are healthy.

**Fast triage:** does anything else persist across a key cycle (compass
calibration, the 1–15 variance zone, US/metric units)? If compass cal holds but
trips revert, the general write path works and only the *key-off* commit fails →
points at (1)/(2). If nothing persists, the whole commit path is down.

## Board architecture (two-board assembly)

The console is two stacked PCBs with a clean **analog-up / digital-down** split,
joined by ribbon/flex interconnects:

**Top board (P14213, "sensor/power" board)** — the one that takes the truck
connector, carries the VTSS light, and sits level in the cab:
- **Compass analog front-end (CONFIRMED by ID):** 2× **Philips/NXP KMZ51** AMR
  magnetic-field sensors (8-pin SOIC) mounted orthogonally = the **X/Y compass**
  (level mounting is why the compass hardware is on this board). Their millivolt
  Wheatstone-bridge outputs are conditioned by **2× ROHM BA10324F** quad
  ground-sense op-amps (8 op-amps total = 2 axes × diff-amp + comp/buffer, with
  the KMZ51's integrated flip/compensation coils run closed-loop per the NXP
  compass app-note). Ground-sense op-amps suit the single-5 V, near-ground bridge
  signals. Output → MCU ADC on the bottom board → heading via the EEPROM's
  hard/soft-iron cal.
- **Main power input / protection / bulk storage**, including a **series MOSFET**
  (most likely reverse-battery / load-dump protection in the B+ path — lower drop
  than a diode on an always-powered IOD feed) and an **LM2940** LDO.
- Two 3-pin SOT-23 devices near the input (marked `25` / `33`) — provisional
  **voltage references / small regulators** for the analog rails (feed the
  on-board compass analog plus route to the other board). Identity TBD — the
  2-digit SOT-23 marks are ambiguous; the measured output rail voltage is
  definitive.

**Bottom board (the "digital" board)** — MCU **NEC μPD78214**, the **93LC56A**
EEPROM, **OKI VFD driver**, the VFD, and its **own local regulation** (the
**LM2574-ADJ** switcher + inductor visible on this board).

### J1 truck connector pinout (traced in KiCad)
| Pin | Net | Pin | Net |
|---|---|---|---|
| 1 | TMP SNS+ | 6 | FUSED B+ (always-on, IOD) |
| 2 | TMP SNS- | 7 | **IGN ST/RN** (ignition sense) |
| 3 | SNS GND | 8 | VTSS (theft light) |
| 4 | CCD+ | 9 | MT |
| 5 | CCD- | | |

Note **pin 7 IGN ST/RN is the key-state (save-trigger) input**, and **pin 6
FUSED B+ is the always-on feed** — the two power/timing signals central to the
key-off save. TMP SNS+/- is an external temperature-sensor input.

## LEADING SUSPECT — cap-glue via corrosion (found 2026-09)

The **only visible damage** on the assembly is a **corroded via** eaten by
**electrolytic-capacitor retaining glue** — a textbook 1990s failure mode (the
tan/brown glue turns conductive *and* corrosive with age and eats copper). The
via is **confirmed open** (multimeter): a trace from the top board's `33`-marked
SOT-23 device routes to the bottom board, runs on the back side, comes up just
off the bottom-left of the **LM2574**, heads SE, and drops through this via —
which no longer connects to the bottom-side plane. The owner's deduction that the
target plane is **not** GND (it sits *next to* a GND plane; two GND planes would
already be tied) is sound → it's a **signal/power/reference** net, not ground.

**Why this is the prime suspect:** a single broken connection in an otherwise
fully-working module is exactly the signature of "one function (the key-off
save) dead while display / compass / live trips all work" — i.e. a rail,
reference, or control/sense line that is only *needed during the save sequence*
has an alternate path for normal running but is broken for the commit.

**Plan:** repair the via/trace, clean **all** cap glue off (inspect for further
glue-corroded traces/vias nearby — they rarely come alone), then re-test whether
the trip data survives a key cycle. If it does, this was the fault and the Saleae
capture becomes confirmation rather than diagnosis. If not, fall back to the bus
capture (CLK/DI/DO + Vcc during key-off) per the plan above.

## RESOLVED (2026-09) — the corroded via was the save-trigger path

**Repairing the cap-glue-corroded via fixed it.** After re-establishing the
broken connection, the console **saves trip state across a key cycle** for the
first time in years — the frozen ghost state had persisted **since at least
2017** (across two owners), so this was a long-standing, genuinely stuck fault.

**Confirmed root cause:** electrolytic-capacitor retaining glue corroded a via
**open** over the years, breaking a **detection/trigger signal** the module needs
to commit the save at key-off. Everything downstream of the trigger was healthy
all along — which is why the diagnosis kept pointing here:
- the **93LC56A bench-tested good** (diff-write passed) → not the memory;
- **constant power present** at the connector → not the always-on feed;
- the **VFD-boost cap** was a separate (leak) issue, not the hold-up path;
- display / compass / live trip updates all worked → the MCU, firmware, and
  write path were fine.

The one thing left was **"the save is never triggered,"** and the open via was
exactly that trigger line going nowhere. The earlier branch-(B) hypothesis
("shutdown-detect / power-down signal broken, so the MCU never starts the save")
was correct; the break was a corroded interconnect via rather than a bad part.

**Takeaway for these consoles:** on a no-save Chrysler overhead console with a
chip that tests good and power present, **suspect cap-glue corrosion on the
key-state / power-down-detect interconnect before replacing the EEPROM.** Clean
all glue and ring out the trigger path across the board-to-board vias.
