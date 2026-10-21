# HWD31 Reverse-Engineering Study — In-Depth Analysis by Aspect

> Building on the facts of the `01` history document, this document answers three questions for each aspect:
> **Why consider this aspect? What is its relationship to achieving SIM Lock? What does the current result mean?**
> `01` is responsible for "what was done and what was obtained"; this document is responsible for "why it was done, what the relationship is, and what the results show".
> Conclusion first: the lock mechanism has been fully resolved; under the constraints of "no disassembly, no official channels, only the running system", there is no usable unlock path at the software layer. This conclusion is not a case of "the analysis not going deep enough", but a structural result to which multiple aspects point jointly.

---

## 1. Document positioning and analytical framework

### 1.1 Division of labour with 01

- `01` records facts, evidence and results by aspect of exploration;
- this document does not repeat the details of `01`, but explains each aspect's **motivation, its logical relationship to SIM Lock, and the meaning of the result**;
- therefore every judgement in this document rests on the facts of `01`; if any fact in `01` is revised, the corresponding conclusions here should be re-examined.

### 1.2 The four conditions for unlocking

Breaking "unlocking" apart, four classes of condition must be satisfied simultaneously:

| Condition | Meaning | Corresponding aspect | Typical failure manifestation |
|---|---|---|---|
| Cognition | Knowing how the code is verified, where it is stored, and how it takes effect | A / D | Not knowing what to read or what to write |
| Reach | Being able to enter a context that can be read / executed / written | C / E / G | A protocol exists, but there is no code execution |
| Operation | Being able to read/write NV, or to submit a lock code | E / F | Configuration can be written, but NV cannot |
| Effect | A write can be persisted to storage, or a submission can be accepted | B / F | Commands are always rejected; installation is uncontrollable |

**The significance of this framework**: many routes appear to "have a vulnerability", but in reality satisfy only one or two of these conditions. For example, WebUI has configuration-write capability (operation) but no execution context (reach); FOTA has download capability (reach), but installation is partition flashing (the object of operation is wrong). The aspect-by-aspect analysis that follows unfolds within this framework.

### 1.3 Two theoretical paths

1. **Read NV**: read the 140 B at 0xC546 (hash+salt) → offline brute-force → a single submission;
2. **Write NV**: directly rewrite the lock record (clear the lock, or write in the hash+salt of a known code) → takes effect on reboot.

Both paths require "reach + operation"; the difference is that reading only needs an execution context, whereas writing additionally needs an unrestricted NV write channel. This difference is the key to understanding every subsequent failure.

---

## 2. Aspect A: the lock mechanism — why "cognition" came first

**Why consider it**: without knowing how the code is verified, which NV item stores it, and how failures are counted, it is impossible to judge whether any route is close to the goal. The lock mechanism is the coordinate system for all the work.

**Relationship to SIM Lock**: it directly determines the definition of "success". It yields three hard constraints:
- the code is a hash preimage written at the production line, not a function of the IMEI;
- the target record lives in modem NV 0xC546 (50502 / E0) and 0xC547 (50503 / E1), with length 0x19B;
- 10 errors cause a permanent lock, so online exhaustive search is impossible; only offline reading plus a single submission is viable.

**Result**: the double SHA256 chain, the salt position, the slot mapping, the 10-attempt limit and the `^DATALOCK` oracle are all empirically established (see `01` §3). The key negative findings are: no `code=f(IMEI)` algorithm exists on the device side, `^NVRDEX` does not exist either, and the NV read path relied upon by external tools fails on this unit.

**What this means**:
- the problem is reduced to "reading out 140 B", rather than "guessing an algorithm";
- static analysis can only prove the mechanism, not replace reading; without the NV record, no submittable code can be produced;
- the 10-attempt limit economically rules out "online code trial"; `^DATALOCK`'s 3 per boot only suffices to verify a small number of candidates, not to search exhaustively;
- consequently, the value of every subsequent aspect can be measured by "does it get closer to those 140 B, or can it rewrite them?"

---

## 3. Aspect B: lock commands and gating — why the "shortest path" was sought first

**Why consider it**: if some AT command could directly submit or rewrite the lock state on a locked device, then neither reading NV nor root would be needed. This is the lowest-cost, most controllable hypothesis, and it had to be ruled out first.

**Relationship to SIM Lock**: it corresponds directly to the "effect" condition. `^CARDLOCK` is the submission port, `^RSFW`/`^SETHWLOCK`/`^TESTHWLOCK` might rewrite the lock state, and `^DATALOCK` is the verification port; if any one of them were usable on a locked device, the problem would be over.

**Result**:
- `^CARDLOCK` SET is always rejected in this image (`+CME ERROR: 3`); only queries work;
- `^DATALOCK` is the submission port for E0, 3 per boot; failure returns `+CME ERROR:16`, but it does not consume the `^CARDLOCK` counter;
- `^RSFW` writing to EFS requires the unlocked state first, and is only opened for SIMLOCK;
- `^SETHWLOCK`/`^TESTHWLOCK` require the unlocked state; FTM writes (`^SSID` etc.) likewise require it;
- `^SPWORD` is a separate 3-attempt limit; the risk outweighs the benefit, and it was never touched throughout.

**What this means**:
- the lock was not "forgotten to be checked" at the AT layer; it is deliberately gated by the modem firmware, and there is no usable bypass at the AT layer;
- the real value of `^DATALOCK` is "reducing the risk of code trials", not "producing candidate codes"; it cannot replace NV reading;
- continuing to look for write bypasses on the AT command surface has a very low marginal return; unless a new firmware version or a new command table appears, this line can be considered closed.

---

## 4. Aspect C: device channels — why "ability to control" is not "ability to execute"

**Why consider it**: any read, write or injection experiment needs a stable channel. Without a channel, the later static conclusions cannot be brought to bear on the device; with a channel, a "static hypothesis" can become a "measured result".

**Relationship to SIM Lock**: indirect but necessary. A channel does not by itself change the lock state; it determines "where one can reach":
- AT/WebUI is the configuration and control plane;
- diag/QCDM is the modem diagnostic plane;
- adb is a potential root execution plane;
- USB composition switching determines whether these planes are exposed.

**Result**:
- WebUI/WiFi can be fully controlled, and is decoupled from the USB mode;
- AT can be connected directly (later COM26 @115200); read-only probing is stable;
- diag is silent in every reachable combination tested; it is a "zombie interface";
- the adb interface exists, but adbd is not running; no shell was obtained;
- the USB mode/composition script executor has been solved, but it cannot conjure a diag channel out of nothing.

**What this means**:
- the project already has a "control plane" but no "execution plane"; WebUI/AT can change configuration but cannot execute arbitrary commands;
- the two most valuable channels (diag, adb) happen to be both dead: one has no data link, the other has no running adbd;
- the channel is not the bottleneck; **"what a channel can reach" is the bottleneck**; further investment repeated only on these two channels will not change the conclusion.

---

## 5. Aspect D: firmware static analysis — why "understanding" is still necessary

**Why consider it**: static analysis is zero-risk; it can provide a mechanism map, locate write targets, and prove that "a certain path simply does not exist". Given that the device cannot be blind-tested, static analysis is the only means of safely expanding the attack surface.

**Relationship to SIM Lock**: it supplies the "cognition" condition and turns channels into operable primitives:
- finding the `/dev/nv` protocol and item mapping tells us what to read;
- finding aboot's fastboot/bootbk/dload branches tells us whether the bootloader has an entry point;
- finding the updater's write backend tells us where "installation" writes;
- finding the WebUI's ACL/path allowlist tells us which interfaces are writable.

**Result**: modem (Hexagon), kernel, aboot, updater and WebUI have all been resolved down to the key branches; at the same time a set of "negative proofs" was obtained:
- the fastboot command table exists, but the whole image contains no call to `fastboot_init`;
- bootbk needs SMEM `RECK`, which nothing in the firmware writes;
- dload has no independent branch; the predicate is always 0;
- `osa_flash_*` is a file backend and does not touch partitions;
- WebUI has no shell import, and the path allowlist is implemented correctly.

**What this means**:
- depth of analysis is no longer the limiting factor; "what is and is not in the firmware" is essentially clear;
- the greatest value of static analysis is not "finding holes" but **proving that certain holes do not exist**, thereby avoiding repeated investment;
- absent new binaries or new partitions, the marginal return of continuing with full decompilation is very low; R4b is the exception, because what it lacks is an image, not analysis.

---

## 6. Aspect E: reading NV — why it is the "straight road"

**Why consider it**: this is the shortest path. As long as the 140 B at 0xC546 can be read, the code can be brute-forced offline and submitted once; no NV write is needed, no root persistence is needed, and not even a change of device state is needed.

**Relationship to SIM Lock**: it corresponds directly to the read operation within "reach + operation". Success means the unlock path holds; failure means one must turn to writing NV or to hardware.

**Result**:
- the `/dev/nv` protocol has been 100% solved: a 12 B header + 1024 B of data, i.e. `open + read({0x22,128,1})` is enough;
- but AP-side code execution is required; across the whole rootfs there are only three fixed users — authconsole, webserver and router;
- `^RSFR` can read SIMLOCK/LOGSAVE, but does not contain the hash; `^NVRDEX` does not exist;
- diag/QCDM is entirely silent; P1 (bbou NV backup → mass) was disproved on the unit (1C20 empty LUN);
- the eUAP console password equals the unlock code, and the tty is an internal UART, forming a circular dependency.

**What this means**:
- the protocol is not the obstacle; **privilege is the obstacle**; reading NV is only "one code-execution context that can call `/dev/nv`" away;
- this also explains why the project long remained at "only one line of code execution short": it is not that we do not know how, but that there is no execution target;
- any future software route must be able to provide AP execution; otherwise only hardware remains.

---

## 7. Aspect F: writing and execution — why "having a write primitive" is still not enough

**Why consider it**: if NV cannot be read, the second path is to write NV (clear the lock, or write in a known hash+salt); in addition, writing configuration may affect update, network and firewall behaviour, and thereby indirectly create execution conditions.

**Relationship to SIM Lock**: it corresponds to the "operation + effect" conditions. The 22-file write primitive is a "controlled write", FOTA is an "official channel", and installation is "partition flashing"; all three could become a springboard for writing NV or for execution, but they need the right write target.

**Result**:
- the `nvram.bak` checksum was broken, yielding a controlled write of 22 configuration files; the path field is strictly validated and cannot be traversed;
- the FOTA download chain was connected (30→40→60), but installation = partition-level flashing (webui is yaffs2, firmware is a raw partition);
- local upgrade-package injection is stopped by a "byte-level content lock"; an official same-version package can reliably reach 90 (5 times in total);
- the file-upload allowlist is `.bin/.zip`, and `cur_path` does not take part in the write target;
- DMZ/IP/firewall injection needs a PPP WAN or server-side validation; the dynamic probes did not take effect.

**What this means**:
- the 22-file write is the software-controlled "high-water mark", but what it writes is configuration, not NV; its blast radius is limited by the path validation and the file list;
- the FOTA download is the official-protocol-controlled "high-water mark", but the installation stage is partition flashing, which cannot serve as an arbitrary write;
- "having a write primitive" is not "being able to write to the target": a write primitive must simultaneously satisfy "the right target + taking effect + not damaging the device";
- the result of this step moved the project from "completely read-only" to "controlled writes", but did not cross the root/NV-write boundary.

---

## 8. Aspect G: the five-layer sweep — why the failure is structural

**Why consider it**: after the direct routes (read NV, write NV, AT bypass) all failed, a systematic check was needed: whether bootloader, kernel, modem, updater and WebUI still hold overlooked entry points. The five-layer sweep is the final audit of "are there any shortcuts left".

**Relationship to SIM Lock**: each layer might supply one link of "reach / operation / effect":
- the bootloader might supply a fastboot/flashing entry point;
- the kernel might supply an SMEM/flash write node;
- the modem might supply an NV backup/restore bypass;
- the updater might supply a partition write;
- the WebUI/app might supply command execution.

**Result**: all five layers are closed, and for different reasons:
- bootloader: fastboot is never called, bootbk has no writer, dload has no branch;
- kernel: `/dev/dload_nark` can write SMEM, but it is 0600 root and has no RECK/UADB;
- modem: `^NVRESTORE` is a closed loop over `0:OEMINFO`'s own backup;
- updater: `osa_flash_*` is a file backend, and its write target is worthless;
- WebUI/app: no shell point, path traversal is blocked, writes go to fixed write targets, DMZ is validated.

**What this means**:
- the failure is not "one hole missed"; all five layers exhibit the same pattern: **the capability exists, but there is no reachable execution/write target**;
- this is stronger than a single route failing: it shows that, under the current firmware and constraints, the software-layer attack surface has been systematically exhausted;
- even with root, an NV write channel is still missing; root is a necessary condition, not a sufficient one.

---

## 9. Aspect H: non-software paths — why "boundaries" must be drawn

**Why consider it**: if the software layer is closed, one must answer "is there any method that does not depend on the running system". This is not for immediate execution, but to delimit the project's conclusion: which paths are "infeasible under the current constraints" and which are "feasible once the constraints are relaxed".

**Relationship to SIM Lock**: non-software paths correspond directly to "physical read/write of NV" or "official acquisition of the code":
- UART/JTAG/EDL/NAND can bypass software privileges and read/write NV directly;
- official/third-party channels can supply the code directly;
- same-model NV backups and production-line tools are indirect options.

**Result**:
- official/third-party channels lie outside the constraints, but the official one is the only highly deterministic path;
- UART/JTAG/EDL/NAND all require disassembly, soldering or specialised tools;
- C6 (offline NAND read) is the only hardware route that "can be closed with existing knowledge": read NV → brute-force → a single submission, but it requires desoldering;
- C7 (fastboot/bootbk/dload) has already been disproved at the software layer and need not be repeated.

**What this means**:
- the project's conclusion is **conditional**: infeasible under "no disassembly, no official channels"; once the constraints are relaxed, the route ordering is C1 → C2 → C6 → C3 → C5/C4;
- the value of non-software paths is not "do it right away", but to let the reader know the cost and success probability of each route;
- this also shows that continuing to look for a "no-hardware-entry" breakthrough at the software layer already has a very low marginal return.

---

## 10. Cross-cutting synthesis: why these results point to the same conclusion

### 10.1 Common pattern: capability separated from write target

Placing the eight aspects together reveals one and the same failure pattern:

| Aspect | Capability already held | Missing write target |
|---|---|---|
| C channels | WebUI/AT control | no execution plane |
| D static analysis | protocols/entry points fully solved | no reachable caller |
| E read NV | `/dev/nv` protocol | no AP execution |
| F write | 22-file write, FOTA download | cannot write NV, cannot execute |
| G five layers | every layer's capability is present | entry point / write target / identity — one missing |

This is not a question of "depth of analysis" but of "device constraints": **the firmware places the critical capabilities in protected regions, and leaves no legitimate entry point reachable from a PC**.

### 10.2 Why root is not a sufficient condition

- Reading NV: root suffices (the `/dev/nv` protocol is solved);
- Writing NV: root does not suffice:
  - `/dev/dload_nark` has no RECK/UADB fields, and the node is `0600 root`;
  - `^NVRESTORE` only replays `0:OEMINFO`'s own backup;
  - rewriting EFS2/modem via `/dev/mtd*` has no recovery means;
- Execution: root requires an entry point first, and entry points are exactly what the five-layer sweep closed.

Therefore the judgement "once we have root it will be easy" must be corrected to: **root can solve reading, but not writing; and reading in turn requires an execution entry point first**.

### 10.3 The two "high-water marks"

- **The 22-file write**: the high-water mark of software control — it can change configuration and influence update/network behaviour, but it cannot write NV and cannot execute;
- **The FOTA download**: the high-water mark of official-protocol control — it can make the device download proactively, but installation is partition flashing and cannot be an arbitrary write.

Both capabilities are real achievements, but neither suffices to cross the boundary of the "protected region". They explain why the project appears to be "only one step short" when in fact it is short of an entire class of privilege.

### 10.4 Two key lessons

- **A static hole ≠ triggerable**: the concatenation points in DMZ/IP/firewall exist statically, but triggering them requires a PPP WAN or server-side validation;
- **A write primitive ≠ a useful write**: the write target, path and effective mechanism of the 22-file write are all restricted, and it cannot be equated with an arbitrary write.

---

## 11. Key corrections to prior understanding

| Old judgement | Evidence | Corrected judgement |
|---|---|---|
| root is enough to unlock | root only solves `/dev/nv` reading; all NV write channels are closed | root is necessary but not sufficient; writing NV still needs an additional channel |
| the fake FOTA dpath allows arbitrary writes | it actually always lands at `/online/online_update.zip` | dpath is not the write target; withdrawn |
| a local upgrade package can inject content | byte-level content lock on the component's compressed stream | local upgrade can only repair the same version, not inject |
| AT can open diag | only 3 composition-script executors; mbimd is the only automatic one | diag is a zombie interface, not triggerable from a PC |
| config injection can reach a shell | needs a PPP WAN / server-side validation | statically holed but not triggerable |
| symbol address = file offset | aboot's 0x28 wrapper header causes base-address misalignment | verify the base address/version before drawing conclusions |
| `cur_path` is the upload write target | the server fixes it to `/nvram.bak` | statically confirm the write target before writing |

These corrections are themselves an important output of the project: they eliminate "seemingly feasible" routes one by one, avoiding continued investment in the wrong direction.

---

## 12. Significance and priority of the remaining directions

> R1 = log extraction, R2 = lock-command static analysis, R3 = non-WAN execution points, R4 = official image baseline; R4b = dload signature engine.

| Direction | What it can test | Cost | Probability of success | Constraint |
|---|---|---|---|---|
| R4b: dload signature engine / NV writer | whether the install stage is exploitable | needs the missing partition / flashing | unknown | hardware |
| Network daemon CVE | whether a WAN-independent RCE exists | high | low | software |
| Memory corruption | whether the `/app` parsers have a controllable crash | very high | low | software |
| UART | whether a root shell can be obtained | medium-high | medium | disassembly/soldering |
| EDL | whether the whole disk can be read/written | extremely high | low | shorting/programmer |
| Offline NAND read | whether the NV record can be obtained directly | high | medium-high | desoldering |
| Official/third-party | whether the code can be obtained directly | low | high | outside the constraints |

**Priority judgement**:
1. R4b has the highest information value, but what it lacks is an image, not analysis; it cannot be advanced without a new partition;
2. the network CVE surface is the only remaining software surface, but it is costly and low-probability, and requires a concrete candidate;
3. the hardware routes (UART/EDL/NAND) can all bypass software, but conflict with the "no disassembly" constraint;
4. official/third-party is the shortest path outside the constraints, with the official one being the most deterministic.

---

## 13. Information sufficiency and confidence

### 13.1 Per-aspect confidence

> Evidence grades: A = empirically established item by item; B = provable statically/from documentation; C = inference or external experience. Confidence is "how reliable this judgement is given the current evidence", not the probability of success.

| Aspect | Key judgement | Evidence grade | Confidence | What is still missing | Would filling it change the conclusion? |
|---|---|---|---|---|---|
| A lock mechanism | double SHA256, NV slots, 10-attempt limit | A | high | whether E0/E1 share the same code; E1's item / read method | may affect the submission strategy, not the mechanism |
| B lock commands | `^CARDLOCK` always rejected; `^DATALOCK` 3 per boot | A | high | no critical gap | no |
| C device channels | WebUI/AT usable; diag a zombie; adb not running | A/B | medium-high | retesting on other firmware/more combinations | may affect channel availability |
| D firmware static analysis | protocols/entry points fully solved; three bootloader routes closed | A | high | new binaries/new partitions | may change the entry-point judgement |
| E reading NV | protocol solved; AP execution missing | A/B | high | root/execution entry point | yes (direct unlock) |
| F writing/execution | 22-file write; FOTA download; no NV/root | A | high | execution entry point, NV write primitive | yes (writing NV can unlock) |
| G five-layer system | all five layers closed | A/B | medium-high | new vulnerabilities, new partitions, new firmware | yes (may reopen an entry point) |
| H non-software | all require hardware/official/third-party | B/C | medium | hardware specification, price, eligibility | does not change "infeasible within the constraints" |

**How to read this**: the aspects graded A/B (mechanism, commands, static analysis, write path) are the firmest; the conclusions of C/G depend on "the combinations/images already tested", and may need retesting on a different version; H is an assessment, not a measurement.

### 13.2 What information would change the conclusion

| Unknown information | Impact | How to fill it | Cost | What it could change |
|---|---|---|---|---|
| whether E0/E1 share the same code | `^DATALOCK` only verifies E0 | static comparison + read-only probing | low | the submission strategy |
| a real dload session | the hard session gate is unsolved | packet capture of a real flash / tool sample | medium-high | may reopen dload |
| the R4b signature engine | exploitability of the install stage | supply a dumped partition | high | may reopen installation/signature |
| network daemon CVE | the only remaining software surface | CVE search + controllability verification | high | may obtain AP execution |
| NAND/EDL hardware | physical read/write of NV | disassembly/programmer | high | may bypass software |
| other firmware versions | the applicability of the conclusion | collect other versions for comparison | medium | may offer different entry points |
| official/third-party | the actual feasibility of unlocking | policy/market research | low | does not change the technical conclusion |

> For the complete list (including "why it matters / how to fill it / priority"), see `01` Appendix C.

### 13.3 Scope of applicability and boundaries

- this conclusion is based on **one device, one firmware version** (see `01` Appendix D); no hardware path was attempted, and no official/third-party channel was tried;
- the conclusion should therefore be phrased as: **under the current firmware, current privileges and current constraints, there is no usable unlock path at the software layer**;
- it is not a proof that "no HWD31 can ever be unlocked"; with a different firmware, a different partition or different hardware conditions, the conclusion needs re-examination.

### 13.4 Information-sufficiency conclusion

- on "understanding the mechanism and the conclusion": the existing evidence is sufficient;
- on "a complete summary of the information obtained": `01` Appendices A/B/D/E add the route matrix, knowledge inventory, version identity and capability inventory;
- on "assessing what is still missing": `01` Appendix C and §13.2 of this document give the list of unknowns and the impact judgement;
- the remaining shortfalls lie mainly in **external information** (official/third-party prices and eligibility, specific NAND/EDL hardware information).

---

## 14. Conclusion: what the project proved and what it did not

**Proved**:
- the lock mechanism, NV slots, command gating and write paths all have code/measured evidence;
- under "no disassembly, no official channels, only the running system", there is no usable unlock path;
- the failure pattern of the five-layer sweep is consistent: the capability exists, but there is no reachable execution/write target;
- the 22-file write, the FOTA download, the `^DATALOCK` oracle and the official baseline restore are real capabilities, but none of them suffices to cross the NV boundary;
- the official image baseline is repeatably reproducible (5 times to 90).

**Not proved**:
- it was not proved that "no software path exists at all" — only that "none exists under the current firmware, current privileges and current constraints";
- it was not proved that hardware paths are infeasible — UART/EDL/NAND were not executed, and they conflict with the constraints;
- it was not proved that official/third-party channels are infeasible — they lie outside the constraints, and the official path is highly deterministic;
- it was not proved that R4b is infeasible — the signature implementation is not in the available images, and belongs to the unknown of the "undumped partition".

**Value of the project**:
- mechanism knowledge: a reusable methodology for reversing SIM Lock;
- toolchain: firmware rebuild, Ghidra projects, nvram.bak encode/decode, FOTA test server;
- negative results: the one-by-one elimination of 41 routes (see `01` Appendix A), sparing later work from repeating the investment;
- baseline: the official image restore procedure is repeatable, the device remains healthy, and the lock stands at 0/10.

**Final judgement**:

> Under the three constraints "no disassembly, no official channels, only the running system", no usable unlock path was found.
> The failure pattern across the five layers is highly consistent: **the capability is always present, and what is always missing is the step of "getting the data into a protected region"**; and that step invariably requires a root execution environment, an unrestricted flash write, or a hardware channel.

---

## 15. Method and decision review

### 15.1 What was done well

1. **Evidence closed loop**: firmware static analysis → device read-only → replay interpretation, with the three lines corroborating each other;
2. **Elimination first + final hearing on the unit**: P1, `cur_path` and injection unreachability were all killed by device procedures rather than by static argument alone;
3. **Black-box and white-box in parallel**: the checksum push used the differential oracle and decompilation-based localisation at the same time;
4. **Reversibility by design**: baseline backups, restart-reset of the restore state machine, and the official package as a fallback;
5. **Self-review**: old conclusions such as "FOTA dpath arbitrary write" were voluntarily withdrawn, avoiding prolonged drift.

### 15.2 What could be improved

1. **Document duplication**: multiple parallel records caused cross-session recovery cost; the division of labour between this document and `01` is intended to reduce duplication;
2. **FOTA gating was discovered rather late**: once write capability was established, key configuration families such as update/network/wifistation should have been enumerated immediately;
3. **Injection triggerability was assessed rather late**: after a static hole was found, "trigger conditions" should have been assessed at the same time, rather than doing device experiments first;
4. **Insufficient base-address/version verification**: aboot, libwebsdk and webserver all once required rework due to base-address/version misalignment;
5. **Insufficient write-target confirmation before writing**: `/nvram.bak` was overwritten with 0 bytes once, and was later rebuilt with `Control=3`; the lesson is "statically confirm the write target before writing".

### 15.3 Three disciplines

1. verify `^CARDLOCK` at every step (currently `1,10,0`, **0/10**);
2. before any write-type operation, statically confirm the write target;
3. restore from the latest backup and merge `sta_policy=2`; online installation with official images only.

---

*This document is an in-depth interpretation of `01` and does not constitute an operating guide; any lock-code submission, flashing or disassembly carries risk.*
