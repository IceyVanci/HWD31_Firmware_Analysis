# HWD31 Non-Software Path Analysis (doc/03)

> Generated: the evening of 2026-10-21. Scope: apart from "finding a software entry point in the running system", what other analysis and unlocking possibilities exist.
> Constraint premise: the user explicitly rules out disassembly and hardware modification, and does not use official channels; this document is an assessment and a record only and performs no hardware operation.
> Evidence grades follow the rest of the document set: A = proven item by item; B = provable statically or from documentation; C = inference or external experience.

---

## 1. One-line conclusion

With both constraints in force at once (no disassembly + no official channels), every non-software path is unreachable. If either constraint is relaxed, in order of value for money:

1. Official/carrier unlock (least effort, most lawful, but requires contract eligibility and a fee);
2. Third-party unlock services (pay for a code; the risks are fraud and re-locking);
3. NAND offline read → obtain hash+salt → offline brute force (requires desoldering the chip, but uses only knowledge this project already has);
4. UART/JTAG debug port (requires soldering; the payoff is a root shell, and the NV write problem still has to be solved);
5. USB EDL (9008) (requires shorting a test point plus a firehose programmer, which this device's package does not contain).

## 2. Pin down exactly what "unlock" lacks

- Lock record: modem NV 0xC546 (50502 / E0) and 0xC547 (50503 / E1), each 0x19B (411 bytes);
- Verification: h2 = SHA256( SHA256(code8) ‖ salt ), h2 == record[0:32], salt = record[0x40:0x60];
- Limit: NV 50005 = 10 attempts; this device reports `^CARDLOCK: 1,10,0`, with 0/10 used;
- Hence only two ways to reach the goal:
  - A. Obtain the 8-digit code → submit once with `AT^CARDLOCK`;
  - B. Rewrite NV (clear the lock state, or write the hash+salt of a known code) → takes effect after a reboot.

Every channel below answers the same question: can the code be obtained, or can the physical capability to write NV be obtained?

## 3. Channel summary table

| Channel | Physical conditions required | What it yields | Difficulty | Risk | Legality | Evidence |
|---|---|---|---|---|---|---|
| C1 Official unlock (au / after-sales) | contract eligibility, fee | unlock code or remote release | low | low | compliant | B |
| C2 Third-party unlock service | IMEI, payment | unlock code | low | fraud / re-locking | grey | C |
| C3 UART debug port | disassembly, soldering test points | root shell (ttyHSL0) | high | medium-high | grey | B |
| C4 JTAG (AP/modem) | disassembly, flying leads | memory/register read and write | very high | high | grey | C |
| C5 USB EDL 9008 | short/test point + firehose | whole-flash read and write | high | very high | grey | B |
| C6 NAND offline read | chip-off/fixture (BGA) | raw NV data (hash+salt) | high | high (chip damage) | grey | B |
| C7 fastboot / bootbk / dload | none (disproved) | — | — | — | — | A (unreachable) |
| C8 Reuse of an NV backup from the same model | another unlocked unit | backup (including salt) | medium | low | grey | B |
| C9 Factory/service tools | internal tools | NV read and write | high | high | grey | C |
| C10 SIM/roaming side | a different SIM or plan | nothing (does not resolve the network lock) | — | — | — | A (irrelevant) |

## 4. Channel-by-channel detail

### C1 Official unlock (au / after-sales)

- Principle: the carrier holds an IMEI→unlock-code database, or can release the unlock remotely;
- This project has established that the code is not a function of the IMEI (no MD5 IV/K table anywhere in any image; every historical formula candidate failed), so the only option is to ask the carrier for it;
- Requires: contract expiry / no arrears / compliance with the unlock policy; the charge is usually a handling fee;
- Assessment: the only path that is certain to work, but it is excluded by the "no official channels" constraint.

### C2 Third-party unlock services

- Principle: vendors obtain codes through internal interfaces/databases, or offer "remote flashing";
- Practical risks: IMEI leakage, taking payment without delivering a code, re-locking after the "unlock"; legal status varies by jurisdiction;
- Assessment: if the constraint is relaxed in future, insist first on "code issued before payment + support for attempt-free verification via `^DATALOCK`" (this device already provides an oracle with 3 attempts per boot, so a code can be verified before `^CARDLOCK` is submitted).

### C3 UART debug port (ttyHSL0)

- Known: the boot image cmdline is console=ttyHSL0,115200,n8; on this device authconsole is attached to exactly that UART (its password = the SIM unlock code, a circular dependency that has been disproved);
- However, the UART's bootloader command line / kernel console may itself provide a root shell or control over boot parameters (androidboot.*, init=/bin/sh and similar), depending on whether the test points are brought out;
- Requires: disassembly, locating the TX/RX/GND test points and soldering to them (most models have 3–4 pads);
- Payoff: root shell → read 0xC546 (140B) from /dev/nv → back to "offline brute force"; this does not unlock the device directly;
- Assessment: high effort, and the payoff is "walking a known path again"; this project has shown that root alone still lacks an NV write channel, but reading NV happens to need only root, so UART is the shortest hardware route to reading the hash.

### C4 JTAG

- Requires: identifying the AP/modem JTAG pins (usually not brought out) and a dedicated debugger;
- Payoff: can halt the CPU and read/write memory and peripherals directly, in theory bypassing every software restriction;
- Risk: mishandling can lock up the SoC; extremely high cost;
- Assessment: not recommended unless there is a successful case and schematics for the same model.

### C5 USB EDL (9008)

- Known: sbl1_sahara.c shows that 9008 is available at the boot stage, but this device's package contains no firehose programmer, and entering 9008 requires shorting/test points or a special USB command;
- Payoff: EDL can read and write all partitions (including modem NV and EFS), making it the "most thorough" hardware route;
- Risk: very high (a wrong flash = a brick, with no means of recovery);
- Assessment: consider it only if a programmer for the corresponding platform (MDM9635) is obtained and the risk of bricking is acceptable.

### C6 NAND chip offline read (chip-off / ISP)

- Principle: remove the NAND (or clip onto it in-circuit with ISP) and read a full-disk image with a programmer, then parse EFS/NV offline to obtain the 411-byte record at 0xC546 directly (hash+salt);
- Then: brute-force the 8 purely numeric digits offline (this project has measured brute.exe at about 5.84M hash/s, roughly 17 seconds per round), obtain the code → submit once;
- Requires: a BGA rework station/reballing, a NAND programmer, and parsing of the yaffs2/Huawei EFS layout (this project has already accumulated a great deal of that);
- Risk: damage during desoldering, unreadable data; high cost;
- Assessment: this is the most feasible path that is "fully offline and does not depend on the device running", and the only hardware route that "closes the loop with existing knowledge" (read → brute force → submit).

### C7 fastboot / bootbk / dload (disproved)

- fastboot: the 12-entry command table and the oem unlock string are both present, but no code in any image references fastboot_init → unreachable;
- bootbk: the only entry is g_recovery == 0x5245434B (RECK), and nothing in the firmware writes that value;
- dload: the aboot_init branch predicate is always 0, and the dload flag only affects DTB reads;
- Assessment: this route is closed for good; do not reinvest effort in it.

### C8 Reuse of an NV backup from the same model

- If an NV backup from another unlocked unit of the same model is obtained: the salt in it is random, and the hash corresponds to that unit's code;
- Writing it straight to this device will not unlock it (the code differs), and writing NV still requires a physical channel;
- The only value: validating the record layout/parsers, or serving as a "control sample for writing NV".

### C9 Factory/service tools

- Huawei/Qualcomm factory tools (such as NV writer and factory download tool) can write NV directly, but they are internal tools and are essentially unobtainable through public channels;
- If a leaked version appears in future, first confirm: MDM9635 support, support for the 0x19B record format, and the ability to validate offline;
- Assessment: usable as a realistic substitute for "supplementing partitions/writing NV", but not something that can be counted on.

### C10 SIM / roaming side

- Changing to an au SIM, international roaming, allow-listed APNs and the like only affect "whether data connectivity works"; they do not change the network-lock state;
- This project has confirmed that `^DATALOCK` can still verify codes when no SIM is present (attempt-free), which shows that the lock is independent of SIM identity;
- Assessment: irrelevant to unlocking; excluded.

### 4.11 External information gaps and how to close them

> The table below is not an operational recommendation; it sets out which pieces of information are still missing in order to turn each channel from an assessment into an executable plan.

| Channel | Information still missing | How to obtain it | Cost/constraint | What it would let us judge |
|---|---|---|---|---|
| C1 Official unlock | contract eligibility, fees, application process | official policy / customer service enquiry | low | whether it is feasible and what it would cost |
| C2 Third party | market price, fraud risk, whether code verification comes first | market research / case reports | low | whether it is worth considering |
| C3 UART | test-point locations, successful cases, soldering risk | teardown / cases for the same model | medium-high | whether a root shell can be obtained |
| C4 JTAG | pin definitions, debugger, successful cases | teardown / tools / case reports | high | whether it is feasible |
| C5 EDL | MDM9635 programmer, short points, brick recovery method | platform tools / leaked packages / case reports | high | whether the whole flash can be read and written |
| C6 NAND | chip model/package, programmer, EFS layout, success rate | teardown identification / chip datasheets | high | whether the NV record can be read directly |
| C7 fastboot/bootbk/dload | nothing needed: already disproved at the software layer | — | — | unreachable |
| C8 Same-model backup | a backup from another unlocked unit | source of a device | medium | whether it can serve as a layout/control sample |
| C9 Factory tools | tool availability, whether MDM9635/0x19B is supported | internal channels / leaked packages | high | whether NV can be written directly |
| C10 SIM/roaming | nothing needed: unrelated to the lock state | — | — | irrelevant |

**Assessment boundary**: C1–C6, C8 and C9 were not executed in this project; C7 has been disproved; C10 is irrelevant. The purpose of the table above is to show "which class of information should be obtained first if work is to continue", not to promise that these channels are viable.

## 5. Decision comparison (if the constraints are relaxed in future)

| Goal | Shortest path | Prerequisite | Cost | Success rate |
|---|---|---|---|---|
| Just want it "usable" | C1 official unlock | contract eligibility | handling fee | high (within policy) |
| No disassembly, willing to pay | C2 third party | IMEI + payment | tens to hundreds of yuan | medium (fraud risk) |
| Willing to disassemble, want to "do it yourself" | C6 NAND offline read | programmer / rework station | high | medium-high (a successful read closes the loop) |
| Willing to solder, want a shell | C3 UART | test points / soldering | medium-high | medium (depends on the test points) |
| Pursuing "total control" | C5 EDL | programmer + short | very high | low (brick risk) |

## 6. Recommended order if the constraints are relaxed

1. Try C1 first (official): the only path that is "highly certain and free of side effects";
2. Then evaluate C2 (third party): accept only "verify the code before paying", using `^DATALOCK` (3 attempts per boot) to verify first, then decide whether to submit;
3. To do it yourself → C6: offline NAND read → extract the 0xC546 record → brute force → single submission; this is the route that this project's body of knowledge can reuse directly;
4. C3 UART as a supplement for "root, but not necessarily NV write";
5. Consider C5 EDL / C4 JTAG only when the corresponding tools are obtained and bricking can be tolerated.

## 7. Explicitly not recommended

- Expecting to enter EDL/fastboot directly over USB without disassembly: already disproved for this device (no call sites, no short condition);
- Buying "unlock software/boxes": mostly fraud, or only capable of unlocking older models;
- Hot-air desoldering a NAND directly: the damage rate is extremely high without a rework station;
- Any third party that wants "payment first, code afterwards": do not submit `^CARDLOCK` while it cannot be verified (there are only 10 attempts).

## 8. Conclusion

- Software layer: R1–R4 are closed and R4b is blocked on an undumped partition; there is no usable unlock path;
- Non-software layer: all of it depends on physical contact (disassembly/soldering/chip reading) or official/third-party channels; under "no disassembly + no official channels" it is equally infeasible;
- The only hardware route that "closes the loop with existing knowledge" is C6 (offline NAND read → brute force → single submission), but it requires desoldering and therefore conflicts with the current constraints;
- The project's final conclusion therefore stands unchanged: under the current constraints the SIM Lock cannot be removed; if they are relaxed in future, follow the order in §6.

---

*This document is an assessment of non-software paths and does not constitute an operating guide; any disassembly, soldering, shorting or flashing carries a risk of damaging the device. For the supporting evidence, see the firmware and system-layer investigation in doc/01 and doc/02 §13/§14.*
