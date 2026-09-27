Starting with a collection of pictures of PCB internals to the overhead module in the cap. 

Fault with this unit: The overhead display has its own tracker, and always loads to the same state - miles is at like 1723, mpg is 19.7, distance till empty is correct based off tank % and that 19.7 mpg, and the elapsed time counter is set at like 68:53 (h:m). I can reset it all once I start the truck and it all works normally, however once I turn the truck off and back on it returns to that 1723 miles on the trip counter state. I believe it has power aince lights come on when door opens but I'll check it at the board, then I spect the board itself. Seems like the unit just isn't saving it's state at all. But whatever RAM is onboard the unit works fine until truck is off. And obviously it can read from something on startup. But the truck has 333k miles so I'm sure whatever flash / epprom that's on that board has been through many thousands of uses


Expected issue:
The Cause: Corrupted NVRAM / EEPROM Write ProtectionThe microcontroller inside this 2nd Gen overhead module depends on an onboard Non-Volatile RAM (NVRAM) or EEPROM chip to save its last known state right before power down.When these automotive flash memories fail due to age, high write-cycles, or extreme cabin heat over 20+ years, they typically lock up into a Read-Only / Write-Protected permanent state.Why it loads 1723 / 19.7: At some exact microsecond thousands of miles ago, the EEPROM chip reached its physical end-of-life and suffered a internal logic failure. It successfully burned that exact data packet into its permanent memory cells.Why it acts normal while driving: While the ignition is ON, the system bypasses the EEPROM and calculates everything in temporary, volatile RAM.Why it reverts: When the key turns off, the microchip attempts to write the new data over the old sectors. Because the EEPROM is physically fried and locked up, the write command fails. When you turn the truck back on, it reads the only valid sector it can still find—the ghost image from 1,723 miles ago.



UPDATE — chip exonerated; fault is the key-off save path (not the memory)
The 93LC56A bench-tests good: on the XGecu (T48) it erases, writes, and verifies repeatedly, including a full bitwise-complement diff-write (all 2048 bits forced to flip) — so every cell can change state. The earlier "wrote the same image, verify OK" was inconclusive; the diff-write is what cleared it.

Therefore the no-save fault is not the EEPROM — it is in the module's key-off save path. Constant power is present (12.56 V at the connector via the IOD fuse), so it is not the always-on feed either.

Working theory — the trip record is committed at key-off, and each 93LC56 byte-write takes ~6 ms (tWC), so saving the record needs the logic rail held up ~100+ ms after the key drops. Suspects, in order:
Ignition-sense / power-down-detect input to the MCU — the "save now" trigger. If its divider/transistor failed, the MCU never starts the save. Probe it while cycling the key; it should transition.


LEADING SUSPECT — cap-glue via corrosion (found 2026-09)
The only visible damage on the assembly is a corroded via eaten by electrolytic-capacitor retaining glue — a textbook 1990s failure mode (the tan/brown glue turns conductive and corrosive with age and eats copper). The via is confirmed open (multimeter): a trace from the top board's 33-marked SOT-23 device routes to the bottom board, runs on the back side, comes up just off the bottom-left of the LM2574, heads SE, and drops through this via — which no longer connects to the bottom-side plane. The owner's deduction that the target plane is not GND (it sits next to a GND plane; two GND planes would already be tied) is sound → it's a signal/power/reference net, not ground.

Why this is the prime suspect: a single broken connection in an otherwise fully-working module is exactly the signature of "one function (the key-off save) dead while display / compass / live trips all work" — i.e. a rail, reference, or control/sense line that is only needed during the save sequence has an alternate path for normal running but is broken for the commit.

Plan: repair the via/trace, clean all cap glue off (inspect for further glue-corroded traces/vias nearby — they rarely come alone), then re-test whether the trip data survives a key cycle. If it does, this was the fault and the Saleae capture becomes confirmation rather than diagnosis. If not, fall back to the bus capture (CLK/DI/DO + Vcc during key-off) per the plan above.

RESOLVED (2026-09) — the corroded via was the save-trigger path
Repairing the cap-glue-corroded via fixed it. After re-establishing the broken connection, the console saves trip state across a key cycle for the first time in years — the frozen ghost state had persisted since at least 2017 (across two owners), so this was a long-standing, genuinely stuck fault.

Confirmed root cause: electrolytic-capacitor retaining glue corroded a via open over the years, breaking a detection/trigger signal the module needs to commit the save at key-off. Everything downstream of the trigger was healthy all along — which is why the diagnosis kept pointing here:

the 93LC56A bench-tested good (diff-write passed) → not the memory;
constant power present at the connector → not the always-on feed;
the VFD-boost cap was a separate (leak) issue, not the hold-up path;
display / compass / live trip updates all worked → the MCU, firmware, and write path were fine.
The one thing left was "the save is never triggered," and the open via was exactly that trigger line going nowhere. The earlier branch-(B) hypothesis ("shutdown-detect / power-down signal broken, so the MCU never starts the save") was correct; the break was a corroded interconnect via rather than a bad part.

Takeaway for these consoles: on a no-save Chrysler overhead console with a chip that tests good and power present, suspect cap-glue corrosion on the key-state / power-down-detect interconnect before replacing the EEPROM. Clean all glue and ring out the trigger path across the board-to-board vias.<img width="1848" height="2312" alt="image" src="https://github.com/user-attachments/assets/990d72d8-63a0-42e4-9eca-d943cf899c68" />
