# HWD31 Reverse-Engineering Study — Exploration Progress and Results by Aspect

> This document is organised by "exploration aspect", not chronologically.
> Target: the SIM network lock of the HWD31 (au Speed Wi-Fi NEXT W01 / KD02 / MIZUHO).
> Coverage: the main work from 2026-10-15 → 2026-10-21; raw evidence, firmware images, scripts and samples are not published with this repository.
> Conclusion first: the lock mechanism has been fully resolved; under the constraints of "no disassembly, no official channels, only the running system", no usable unlock path was found at the software layer; every non-software path requires disassembly, soldering, or official/third-party channels.

---

## 1. Objective, constraints and core difficulty

**Objective**: derive the 8-digit SIM unlock code offline and submit it only once; 10 wrong attempts lock the device permanently.

**Constraints**:
- no disassembly, no soldering, no use of official/paid unlock channels;
- no blind attempts against `^CARDLOCK`, no touching `^SPWORD`;
- read-only probing first; any write operation must be rollback-capable.

**Core difficulty**:
- the unlock code is a hash preimage written at the production line, not a function of the IMEI;
- the hash+salt required for verification exist only in modem NV 0xC546 (50502 / E0) and 0xC547 (50503 / E1);
- the only theoretical path is therefore: read the NV record → brute-force offline → submit once.

---

## 2. Exploration aspects at a glance

| Aspect | Core question | Main result | Status | Details |
|---|---|---|---|---|
| A — Lock mechanism | How is the code verified, and where does it live | Double SHA256; NV 0xC546/0xC547; 10-attempt limit | ✅ Resolved | §3 |
| B — Lock commands and gating | Which commands can submit or write | `^CARDLOCK` SET always rejected; `^DATALOCK` attempt-free oracle; `^RSFW`/`^SETHWLOCK` gating | ✅ Resolved, none usable | §4 |
| C — Device channels | How to reach and control the device reliably | USB 1442/1441; AT COM26; WebUI/WiFi; zombie diag; adb not running | ⚠️ Partially usable | §5 |
| D — Firmware statics | What is inside the firmware | modem Hexagon, `/dev/nv` protocol, aboot, updater, WebUI ACL | ✅ Resolved | §6 |
| E — Reading NV | Can hash+salt be read out | Protocol fully decoded but AP execution missing; `^RSFR` has no hash; `^NVRDEX` does not exist; P1 disproved; eUAP loop | ❌ Not achieved | §7 |
| F — Writing/executing | Can NV be written or commands executed | 22-file configuration write; FOTA download; no root/NV write | ⚠️ Partial capability | §8 |
| G — Five system layers | Is there an entry point at each layer | bootloader/kernel/modem/updater/WebUI all closed | ❌ Closed | §9 |
| H — Non-software paths | Can it be unlocked without software | all require disassembly, soldering, or official/third-party channels | ❌ Unreachable within the constraints | §10 |

---

## 3. Aspect A: SIM Lock mechanism and verification chain

### 3.1 Verification chain

```text
code (8 digits)
  → SHA256 → h1 (32 B)
  → SHA256(h1 ‖ salt (32 B)) → h2 (32 B)
  → h2 == NV 0xC546[0:32] ?
success → unlock flag *(GP+0x18fd)=0
failure → global counter +1 (limit 10, NV 50005)
```

- record length 0x19B (411 B); salt sits at record `0x40:0x60`;
- the validator at `0x020623e8` requires 8 pure digits; the core comparison is at `0x02062678`;
- slots: `"OEM" → 0xC546` (50502 / E0), `"SIMLOCK" → 0xC547` (50503 / E1);
- `^DATALOCK` is the submission port for E0: parameters `{code,1,IMEI}`; a failure returns `+CME ERROR:16`; the local RAM counter allows 3 per boot and **does not consume `^CARDLOCK` attempts**;
- the SET form of `^CARDLOCK` is always rejected in this image (`+CME ERROR: 3`); the command can only query;
- key negative findings: no `code=f(IMEI)` algorithm exists on the device; `^NVRDEX` does not exist in this device's firmware; the NV read path that external tools rely on is dead.

### 3.2 Counting and submission discipline

- the lock state has long been `^CARDLOCK: 1,10,0` (10 attempts, **0/10 consumed**);
- `^MAXLCKTMS=10`; `^SPWORD` is an independent 3-attempt limit, riskier than it is useful, and was never touched;
- no unlock code has ever been submitted; `^DATALOCK` was used only for attempt-free checking of historical candidate codes, all of which returned CME16.

**Conclusion for this aspect**: the mechanism is fully resolved; the only viable path is to read the 140 B record (hash+salt) of NV 0xC546 and, after offline brute force, submit once.

---

## 4. Aspect B: Lock commands, write gating and the submission surface

### 4.1 AT command surface

- `dsat_huawei_sim`: `^CARDLOCK` (query + SET always rejected), `^AUTHVER`, `^SPWORD`, `^GLASTERR`, `^RDCUST` etc.;
- `dsat_huawei_ext`: `^DATALOCK`, `^SETHWLOCK`/`^TESTHWLOCK`, `^SIMLOCK`, `^RSFR`/`^RSFW`, `^SSID`/`^WIKEY`/`^WUPWD`/`^SKU` etc.;
- `dsat_huawei_dload`: `^DLOADVER`/`^DLOADINFO`/`^NVBACKUP`/`^NVRESTORE`/`^NVRSTSTTS`/`^AUTHVERIFY`/`^GODLOAD`/`^SUPINFO`/`^SN` etc.

### 4.2 Write gating

| Channel | Gating | Usable on a locked device |
|---|---|---|
| `AT^DATALOCK="code"` | stateless gating; 3 per boot | ✅ submittable, does not consume CARDLOCK (oracle) |
| `AT^CARDLOCK="code"` | rejected by design | ❌ |
| `^SETHWLOCK`/`^TESTHWLOCK` | requires the unlocked state | ❌ |
| `^RSFW` (EFS write) | requires datalock to be removed first; only SIMLOCK writable | ❌ |
| FTM write (`^SSID` etc.) | requires unlocked | ❌ |
| `^RSFR` (read) | statically ungated | ✅ |

### 4.3 Conclusion

- a locked device has no lock command that can write NV directly or bypass the 10-attempt counter;
- `^DATALOCK` is the only low-cost code-testing port, but it writes RAM state only and does not unlock;
- `^CARDLOCK` must never be tried blindly; any candidate code must first be verified through `^DATALOCK` or another read-only means.

---

## 5. Aspect C: Device channels and the control plane

### 5.1 USB modes and enumeration

- initially `12d1:1F01` (CD-ROM); the SCSI CDB `11 06 00 00 00 00` → `12d1:1442` (service mode: COM13=AT / COM14=application port);
- through the three PC→device switching channels — CDB, the vendor control request `C0 9A`, and app_pcui — the combinations 1441, 1566, 1001, 155E, 157E etc. are reachable;
- there are only 3 composition-script executors: the factory shell, boot-time `start_usb` (fixed 1F01), and `mbimd` (the only automatic one);
- conclusion: USB mode switching itself is controllable, but it cannot conjure a diag channel out of nothing.

### 5.2 AT channel

- service mode COM13/COM14; later a direct AT connection at COM26 @115200;
- read-only probing: `^RSFR`, `^NVRDEX`, `^MAXLCKTMS`, `^DATALOCK?`, `^CARDLOCK?`, `^SFEATURE?`;
- `^RSFR` can read SIMLOCK and 14 classes of LOGSAVE logs, but contains no hash;
- `^NVRDEX` returned ERROR in both tests, confirming that the firmware has no such command.

### 5.3 WebUI / WiFi

- the device WiFi hotspot and the WebUI are fully controllable and decoupled from the USB mode;
- login uses a `password_type=4` challenge hash; the form field `csrf_token` needs the `csrf:` prefix, while AJAX uses a request header;
- usable endpoints: `/api/device/mode`, `/api/device/control`, `/api/pin/simlock`, `/nvram.bak` download, `/api/nvramul.cgi` restore, `/api/filemanager/upload`, `/api/dialup/*` etc.;
- result codes: `125003` = no permission/path denied; `100001` = parameter or content error; `100005` = parameter validation failed.

### 5.4 adb

- MI_04 is marked as a standard ADB interface; `adbd` is not running (init is stuck at `f_mtp`, `persist.service.adb.enable` is unset);
- static evidence shows that adbd is root/userdebug, so it would be a root shell once started;
- a direct WinUSB OUT times out and a hand-built CNXN gets no response; no shell was obtained.

### 5.5 diag / QCDM

- ordinary compositions do not execute composition scripts; `echo diag > f_diag/clients` exists only inside composition scripts;
- among the tested reachable compositions 1442/1441/1001/155E etc., diag is completely silent towards the PC; the ROM holds another 157E composition (with diag+pcui), but no script triggers it.
- conclusion: the `f_diag` data link is not registered/enabled and diag is a "zombie interface"; reviving it needs mbimd or a composition script, and the PC can trigger neither.

### 5.6 Conclusion

- a stable WebUI/WiFi + AT control plane has been obtained;
- there is no diag, no adb and no root shell;
- the channel itself is not the bottleneck; the bottleneck is what the channel can reach.

---

## 6. Aspect D: Firmware unpacking and static reversing

### 6.1 Images and tools

- the official upgrade ZIP, unpacking scripts, modem image, boot image, and the rootfs/app/WebUI/ramdisk extraction trees;
- the modem was rebuilt as `modem_full.elf` (about 48 MB); the kernel as `boot5_vmlinux.elf` (68,908 kallsyms symbols);
- tools: Ghidra 12.1.4, JDK 21, REA and custom scripts; AI-assisted organisation.

### 6.2 modem

- the instruction set is Hexagon (QDSP6); segment #17 is the log descriptor database (2.66 MB, 166,355×16 B, 1,486 modules);
- the NV service sits at `0x0205DFxx–0x0205E5xx`; the item map has 16 entries: `0x22 → 0xC546`, `0x2A → 0xC4F2`, `0x2B → 0xC542` etc.;
- SMEM 0x86 magic: read `0x2014AA55`, write `0x2014A5A5`, success `0x201455AA`;
- 12 dload/tool command handlers; the `^CARDLOCK` query and SET chains; the `^RSFR`/`^RSFW` handlers.

### 6.3 Kernel and `/dev/nv`

- `/dev/nv` protocol: 12 B header + 1024 B data `{item,len,status,data}`; reading needs only `open + read({0x22,128,1})`;
- the kernel `huawei_nv_read_item` whitelist is 0..0x2B, proxied to the modem through SMEM 0x86 and QMI;
- whole-rootfs sweep: there are only 3 fixed NV consumers (authconsole 0x22, webserver 0x2A, router 0/4-9/0x21);
- conclusion: the protocol itself is not the obstacle; the obstacle is code execution on the AP side.

### 6.4 aboot

- the `s002` stream is a 0x28-byte wrapper header + the image body; after the base address was corrected, the string-reference hit rate went from 71/757 to 447/757;
- the `fastboot` command table has all 12 entries, but no code in the whole image references `fastboot_init` → unreachable;
- the only entry to `bootbk` is `g_recovery == 0x5245434B` (RECK), and nothing in the whole firmware writes that value;
- dload has no independent branch; `target_is_emmc_boot()` is a stub returning the constant 0;
- conclusion: all three bootloader boot routes are closed.

### 6.5 Updater

- `cl_gz_uncompress_to_Flash` has no signature/hash/version check, the only gate being `dev == 6`;
- but `osa_flash_*` is a file backend: device indices 0..5 map to `/online/*.bin` and `update_info.bin`, and no partition is touched;
- the local-upgrade version check = the last segment of the in-package version string == the last segment of `AT+CGMR`, and two bypaths control it, `check_rule` and the board version prefix;
- the component type is decided by the uploaded filename.

### 6.6 WebUI / app

- `authority.xml` has five tables; the path allowlist `FUN_000288B0` is prefix matching + realpath;
- `webserver` imports no `system/popen/exec*`; all 17 binaries are PIE + stack-protected + FORTIFY;
- static-file `..` variants return 404; path-type endpoints outside `/sdcard` always return `125003`;
- the form `csrf:` prefix and the AJAX request header are the key to admission.

### 6.7 Conclusion

- the static understanding is complete: mechanism, protocol, gating, write paths and the WebUI ACL all have corresponding code evidence;
- the bottleneck at the static layer is not "we cannot understand it" but "there is no reachable execution/write target".

---

## 7. Aspect E: Paths for reading NV

### 7.1 Objective

Read the 140 B (hash+salt) of NV 0xC546, or 0xC547; this is the precondition for brute force.

### 7.2 `/dev/nv` protocol

- the protocol is 100% decoded; but it requires AP-side code execution (a root shell, injection, or a call from a legitimate process);
- every NV consumer has been enumerated: authconsole, webserver, router; there is no hidden "read 0x22" process.

### 7.3 AT surface

- `^RSFR` can read SIMLOCK and LOGSAVE logs, without the hash;
- `^NVRDEX` does not exist;
- `^NVBACKUP`/`^NVRESTORE` are a closed loop over `0:OEMINFO`'s own backup and cannot be exported to the PC.

### 7.4 diag / QCDM

diag is silent in every reachable composition; there is no NV read channel.

### 7.5 P1: bbou NV backup → mass

- static chain: agent/bbou is triggered through `AT^NVBACKUP`; dispatcher state 90/100 + event 0x3c;
- hardware verification: the full official same-version upgrade flow + USB sampling at 1-second intervals; only a 1C20 [mass] empty LUN (0 bytes) appeared, and the [pcui,mass] data shape never did;
- conclusion: the NV backup never enters the USB-exposed surface; P1 is disproved.

### 7.6 eUAP / authconsole

- the authconsole password = the NV 0x22 verification = the SIM unlock code (same algorithm, same layout);
- after login, `EUAP>` is root execution through `system()`, but the tty is the internal UART `ttyHSL0`, unreachable from the PC;
- this forms a circular dependency "password = unlock code" and cannot serve as an entry point.

### 7.7 Conclusion

- there is no PC-reachable NV read path;
- to read 0xC546 one must first obtain AP code execution (root) or go through hardware (UART/EDL/NAND);
- this is why "one line of code execution away" remained the bottleneck for so long.

---

## 8. Aspect F: Write and execution paths

### 8.1 dload

- framing `7E | payload | ~CRC16-X25 LE | 7E`; the CRC is inverted and little-endian;
- `^GODLOAD` can enter the dload/HDLC experimental segment, but "installing" hangs and needs a power cycle to recover;
- 1,200+ blind frames all NAK → hard session gate; the official upgrade flow does not use dload, it uses HTTP upload;
- conclusion: the protocol is fully decoded, but no valid session can be established.

### 8.2 Upgrade-package injection

- the official five container's 17 components are fully decoded; the validation system is MD5-only (RSA is used only for log reporting);
- a scheme to inject `nvdump` (304 B, open/read `/dev/nv`) into the `/system` cpio is constructible;
- but byte-level experiments prove: local upgrade requires the component's compressed stream to be byte-identical to the original → any content modification fails at the apply stage (state 70);
- an official same-version package reliably reaches 90 (5 runs in total);
- conclusion: container and validation fully decoded, but the local upgrade channel cannot carry modifications.

### 8.3 nvram.bak configuration write

- `/nvram.bak` can be downloaded and decoded from base64 line by line; `Control=3` regenerates it; `/api/nvramul.cgi` restores it;
- checksum = Σ `cl_CRC16`(line including `\r\n`) mod 2^32; the last line does not take part; CRC16-CCITT poly 0x1021;
- the restore pipeline has three stages; the path field is strictly validated; only the 22 established configuration files can be written;
- verification: `dialup_filterlist`, `toolsflag` etc. wrote back successfully;
- conclusion: a "controlled-content write across 22 files" capability was obtained, but not arbitrary file write or path traversal.

### 8.4 Online update (FOTA)

- `update/config.xml` server keys are writable; `sta_policy=2` bypasses the network gating; a fake server can make any content pass validation;
- CheckNewVersion → filelist (`dpath`, `md5` uppercase, `size`, `spath`; a component needs `version`; the validate MD5 short-circuit requires changing the content);
- the download chain 30→40→60 works; the file always lands at `/online/online_update.zip`;
- install = partition-level flashing: webui is a yaffs2 image, firmware1/2/iso are raw FW; a corrupted webui = loss of the control channel;
- conclusion: the download chain is usable, but it cannot serve as an arbitrary write/execute channel; install is kept only for "official image recovery/baseline" purposes.

### 8.5 File upload and configuration injection

- `filemanager/upload` allowlist is `.bin/.zip`; `cur_path` does not take part in the write target; `/sdcard` reads back 404;
- `/api/nvramul.cgi` has the fixed write target `/nvram.bak`, and `cur_path` does not take part;
- DMZ/IP/firewall/npdaemon: concatenation points exist statically, but server-side validation or trigger conditions (PPP WAN) make them unreachable; the dynamic probes `telnetd`/`;reboot` had no effect;
- conclusion: there is no reachable "configuration → root execution" path.

### 8.6 Conclusion

- obtained: 22-file configuration write, FOTA download, the `^DATALOCK` oracle, direct AT/WebUI;
- still missing: root execution, NV write, arbitrary file/partition write.

---

## 9. Aspect G: Five-layer system sweep

### 9.1 bootloader

- fastboot: command table/handlers/`oem unlock` are all present, but nothing calls `fastboot_init`;
- bootbk: needs SMEM `+0xB8 == "RECK"`, and nothing in the whole firmware writes it;
- dload: no independent branch, the predicate is constantly 0; the eMMC path is dead code.

### 9.2 Kernel

- `/dev/dload_nark` can write six SMEM fields, but the node is `0600 root`, and the table has no RECK/UADB;
- `/dev/mtd*` is readable, but there is no safe way to write it; rewriting EFS2/modem is high risk with no recovery.

### 9.3 modem NV

- `^NVBACKUP`/`^NVRESTORE` are a closed loop over `0:OEMINFO`'s own backup; magic `0x3C4D5E6F`; version check;
- no external injection is possible; exploiting it would first require writing `0:OEMINFO`, and that partition is not writable.

### 9.4 Updater

- `cl_gz_uncompress_to_Flash` has no signature/hash; but `osa_flash_*` is a file backend that writes only `/online/*.bin`;
- the version lock is forgeable, but the write target is worthless.

### 9.5 WebUI / app

- no shell import; path traversal fully rejected; fixed `/nvram.bak` write; DMZ returns `100005`; the login-free POST surface is still subject to the allowlist;
- all 17 binaries are PIE + stack-protected + FORTIFY.

### 9.6 Conclusion

- all five layers are closed; the failure mode is the same everywhere: the capability exists, but there is no entry point that "lands data in a protected area";
- such an entry point needs a root execution environment, unrestricted flash writes, or a hardware channel.

---

## 10. Aspect H: Non-software paths

| Channel | Requires | Result | Assessment |
|---|---|---|---|
| C1 official/operator unlock | contract eligibility, fee | code obtainable or remote release | the only certain path, outside the constraints |
| C2 third-party unlock | IMEI, payment | code obtainable, risk of fraud/second lock | outside the constraints, verify the code first |
| C3 UART | disassembly, soldering | root shell; still needs NV write | the shortest hardware path for reading the hash |
| C4 JTAG | disassembly, debugger | memory/register access | high cost, high risk |
| C5 EDL 9008 | short pins + firehose | full-disk read/write | extremely high bricking risk |
| C6 NAND off-line read | desoldering/programmer | the NV record directly | the loop can be closed with existing knowledge, but it needs disassembly |
| C7 fastboot/bootbk/dload | none (already disproved) | — | closed |
| C8 same-model NV backup | another unlocked device | different salt/hash, cannot be used directly | reference only |
| C9 production-line tools | internal tools | NV read/write | unpredictable |
| C10 SIM/roaming | another SIM/plan | does not change the lock state | irrelevant |

**Conclusion**: under "no disassembly + no official channels", all non-software paths are unreachable; if the constraints were relaxed, the recommended order is C1 → C2 → C6 → C3 → C5/C4.

---

## 11. Key route outcomes

> The table above is consolidated by exploration aspect; the complete numbering and per-route outcomes of the 41 routes are preserved in the original analysis records and are not expanded in this repository.

| Aspect | Route | Result |
|---|---|---|
| A | Static reversing of the lock mechanism | ✅ double SHA256, NV slots, 10-attempt limit all empirically proven |
| A | `^DATALOCK` oracle | ✅ 3 per boot, does not consume CARDLOCK; writes RAM only |
| B | `^CARDLOCK` SET | ❌ always rejected (CME3) |
| B | `^RSFW`/`^SETHWLOCK` | ❌ require the unlocked state |
| C | USB modes/CDB | ✅ 1442/1441/1001/155E/157E etc. reachable |
| C | WebUI/WiFi | ✅ complete session, configuration read/write, backup download |
| C | diag/QCDM | ❌ zombie interface, silent in every composition |
| C | adb | ❌ adbd not running, no root shell |
| D | modem/NV | ✅ item map, protocol, command handlers fully decoded |
| D | aboot | ✅ all three routes fastboot/bootbk/dload unreachable |
| E | `/dev/nv` direct read | ✅ protocol fully decoded; ❌ no AP execution |
| E | `^RSFR` | ⚠️ can read SIMLOCK/LOGSAVE; no hash |
| E | `^NVRDEX` | ❌ not present in the firmware |
| E | P1 NV backup→mass | ❌ disproved on hardware (1C20 empty LUN) |
| E | eUAP | ❌ password = unlock code; internal UART |
| F | dload | ⚠️ framing/CRC fully decoded; hard session gate |
| F | upgrade-package injection | ❌ byte-level content lock; official package 5 runs to 90 |
| F | nvram.bak | ✅ checksum cracked + 22-file write primitive |
| F | FOTA | ⚠️ download 30→40→60; install = partition flashing |
| F | configuration injection | ❌ needs PPP WAN / server-side validation |
| G | bootloader | ❌ fastboot/bootbk/dload closed |
| G | kernel dload_nark | ❌ 0600 root; no RECK/UADB |
| G | modem NV restore | ❌ `0:OEMINFO` closed loop |
| G | updater | ❌ file backend; the write target is worthless |
| G | WebUI/app | ❌ no shell point / path traversal / fixed target |
| H | UART/EDL/NAND | ❌ need disassembly; excluded by the constraints |
| R1–R4 | logs/lock commands/non-WAN/baseline | ✅ all closed loops; no lock content / no entry / 5 runs to 90 |
| R4b | dload signature engine | ⚠️ not in the available images; needs an additional partition |

---

## 12. Conclusions and remaining directions

**Mechanism**: 100% understood; lock-code verification, NV slots, command gating and write paths all have code or measured evidence.

**Routes**: 41 routes systematically explored; R1–R4 closed, R4b blocked, and a re-check of newly built update packages found no new entry point.
R1 = log extraction, R2 = static lock commands, R3 = non-WAN execution points, R4 = official image baseline; R4b = dload signature engine.

**Final judgement**: under "no disassembly, no official channels, only the running system", no usable unlock path was found; the software layer is closed.

**Why root is not a sufficient condition**:
- the SIM lock lives in modem NV; AP root can only read `/dev/nv`, and an NV write is still required;
- `/dev/dload_nark` has no RECK/UADB fields, and the node is `0600 root`;
- `^NVRESTORE` only restores `0:OEMINFO`'s own backup;
- rewriting `/dev/mtd*` EFS2/modem has no recovery means.

**Remaining directions**:
1. R4b: the signature implementation is not in the available images and needs an additional partition (flashing/hardware);
2. network daemon CVEs (hostapd 2.1 / dnsmasq 2.51) — high cost, low probability;
3. hardware UART/EDL/NAND — needs disassembly, excluded by the constraints;
4. memory corruption — evaluated only when a concrete candidate appears.

**Project value**: mechanism knowledge, a toolchain, reproducible negative results, and an official image baseline.

---

## 13. Safety red line and device state

- `^CARDLOCK: 1,10,0` (10 attempts, **0/10 consumed**); `^DATALOCK: 1`; `^MAXLCKTMS: 10`.
- no unlock code has ever been submitted; `^SPWORD` was never touched; all historical `^DATALOCK` candidates returned CME16.
- experiment discipline: read-only first, then controlled writes; back up before writing; roll back on failure; when the state machine wedges → reboot to reset.
- side effects that occurred: one `/nvram.bak` was overwritten with 0 bytes and later rebuilt with `Control=3`; no device damage.
- current state: USB 1442; WebUI/WiFi normal; firmware state 90; lock 1,10,0.

---

## 14. Evidence and reading notes

- evidence grades: A = empirically proven item by item; B = provable statically or from documents; C = inference or external experience; unverified points are marked "(inferred)".
- function addresses/offsets/structures correspond to the firmware version analysed in this project; other versions may differ.
- raw evidence, firmware images, scripts, samples and the working directory are not published with this repository; this document keeps only publishable conclusions and key evidence.
- this document is not an unlock guide; any lock-code submission, flashing or disassembly carries risk.

---

## Appendix A: 41-Route Matrix

> This table keeps the original route numbering; the `Aspect` column corresponds to §3–§10 of the main text; the outcome is the current state (✅ achieved / ⚠️ partial or closed / ❌ disproved or unusable / ⏸ not done).

| # | Aspect | Route | Goal | Action/method | Outcome |
|---|---|---|---|---|---|
| 1 | E | AT hidden-command sweep | find an NV read command | read-only sweep of ~60 commands | every read-only entry tested; the executable `^NVBACKUP`/`^NVRSTSTTS` work; no direct NV hash read |
| 2 | E | `^RSFR` EFS read | read SIMLOCK/log configuration | `AT^RSFR="SIMLOCK","SIMLOCK"` | ✅ success; only the PLMN lock/LOGSAVE logs, no hash |
| 3 | E | `^NVRDEX` | relied on by external tools | 50502/50503 both tested | ❌ ERROR ×2; the firmware has no such command (settled) |
| 4 | C | COM14 QCDM | diag probing | multiple baud rates + version frames | ❌ not diag (read pipe 31) |
| 5 | C | B2b kernel PNP | switch to a composition containing diag | kallsyms-wide decompilation | ✅ system/channels fully decoded (`C0 9A` etc.) |
| 6 | C | B2a1 CDB sweep | diag-bearing mode | green=0/1 | ✅ 1442/14DB; reachable compositions either have no diag or cannot use the modem port |
| 7 | C | `AT^SETMODE` | →1441 (with diag) | — | ❌ PCUI is not forwarded by ATFWD; modem port missing; app mode is not a diag switch |
| 8 | C | WiFi/WebUI | decoupled control plane | hotspot + mode switching | ✅ full control (reboot/mode/simlock query/nvram.bak) |
| 9 | C | iface2 diag raw | QCDM NV read | full WinUSB bulk matrix | ❌ silent in all three configurations → `f_diag` dead |
| 10 | C | gateway mode 1001/155E | hidden diag | CDB `11 06 30…` | ✅ reached; diag still silent |
| 11 | C | adb (P2) | root shell | WinUSB + CNXN | ❌ adbd not running (`f_mtp` stuck); MI_04 marked ADB and adbd is root/userdebug |
| 12 | E | bbou NV backup (P1) | make the device back itself up → read over mass | static trigger chain + hardware | ❌ disproved on hardware (the NV backup does not enter the USB-exposed surface, 1C20 empty LUN) |
| 13 | F | GODLOAD/dload | official flashing protocol | HDLC + CRC reversing | ✅ framing fully decoded; ⚠️ installing hangs (recoverable); hard session gate |
| 14 | F | dload blind probing | handshake/commands | 1,200+ frames × 5 header models | ❌ all NAK (hard session gate) → a real session capture is needed |
| 15 | E | hwtest/factory | flash read | chain reversing | ❌ needs a bootloader flag; flashtester has no NV read |
| 16 | E | eUAP console | root system() | password = unlock code | ❌ circular dependency; `ttyHSL0` internal UART (settled) |
| 17 | E | `/dev/nv` direct read | protocol + execution | kernel and user space fully decoded | ✅ protocol 100% decoded; ❌ still missing an AP-side code-execution entry |
| 18 | G | network-plane CVE | RCE | dnsmasq/hostapd/miniupnpd | ⚠️ miniupnpd ruled out (not online); hostapd 2.1 WMM = medium-risk fallback (unverified) |
| 19 | F | upgrade-package re-packaging | inject nvdump | five container + MD5-only + nvdump | ⚠️ container/validation fully decoded, direct upload works; content lock → local upgrade cannot inject |
| 20 | F | nvram.bak configuration plane | arbitrary file write | download/restore pipeline + checksum | ✅ checksum cracked (Σ `cl_CRC16`) + 22-file write primitive; ⚠️ path field strictly validated |
| 21 | F | content injection (DMZ/IP/protocol) | →shell | 22-file write injection + trigger chain | ❌ static validation dead / the application trigger needs a PPP WAN (unreachable without a SIM); DMZ measured at 100005 |
| 22 | F | online update (fake FOTA) | arbitrary write/execute | STA gating bypass + FOTA protocol + download | ⚠️ download 30→40→60 works; install = partition flashing (high risk, no blind flashing); official baseline 5 runs to 90 |
| 23 | F | filemanager/upload | arbitrary write target | `cur_path`/allowlist | ⚠️ `.bin/.zip` pass but the write target is unverified; `cur_path` ineffective; everything outside `/sdcard` returns 125003 |
| 24 | B | `^DATALOCK` oracle | attempt-free code testing | E0 hash verification + CME16 | ✅ 3-per-boot boundary; does not consume CARDLOCK; all historical candidates rejected |
| 25 | E | `^RSFR` LOGSAVE logs | log extraction | AT read-only | ✅ completed by R1 (#36); no lock content |
| 26 | B | static lock commands | find an NV write bypass | modem b12 handler analysis | ✅ completed by R2 (#37); gating = RAM flag |
| 27 | H | EDL short-pin | full-chip dump | disassembly | ⚠️ explicitly avoided by the user, last resort only |
| 28 | G | aboot boot decision | find the fastboot/bootbk/dload entries | full `s002` decompilation + instruction-by-instruction checking | ❌ all three rejected: fastboot has no call site; bootbk needs SMEM `RECK` and nobody writes it; dload has no independent branch |
| 29 | G | kernel `/dev/dload_nark` | write SMEM 0x86 | misc device + ioctl command table | ⚠️ can write 6 fields, but the node is 0600 root and there is no RECK/UADB field |
| 30 | G | `^NVRESTORE`/`^NVBACKUP` | rewrite NV | modem decompilation + image tracing | ❌ closed loop: the image is `0:OEMINFO`'s own backup, and the version must match |
| 31 | G | updater write path | arbitrary partition write | `osa_flash_*` + version-check reversing | ❌ only writes `/online/*.bin` staging files; the version lock is forgeable but the write target is worthless |
| 32 | G | WebUI path traversal | arbitrary read | static files + `sdfile` + `Check_file_exist` | ❌ static `..` variants 404; path-type endpoints outside `/sdcard` always 125003 |
| 33 | F | `api/nvramul.cgi` fixed write target | arbitrary write | `csrf:` prefix + multipart | ⚠️ admission passes; the write target is compile-time fixed at `/nvram.bak`, `cur_path` does not take part |
| 34 | F | DMZ injection | configuration → root execution | submit an injection string to `/api/security/dmz` | ❌ 100005 (server-side IP validation); and not inside the 22-file write primitive |
| 35 | G | login-free POST surface | find a credential-free write entry | parse the five tables of `authority.xml` | ⚠️ broad surface (including upload, `sdcard/*`, `pin/*`), but path-type endpoints are still allowlisted; `pin/verify-simlock` must not be touched |
| 36 | E | `^RSFR` logs (R1) | read-only runtime information | chunk protocol reversing + per-chunk fetch | ✅ captured UPDATE_LOG_A/B, WIFI_XML, STAT, CBT; MLOG sampled; no lock-related content |
| 37 | B | static lock commands (R2) | possible attempt-free code verification | AT table entries + handler + slot-table decompilation | ✅ gating = RAM flag; a locked device always gets ERROR; `^SETHWLOCK` changing the code does not unlock |
| 38 | G | non-WAN shell points (R3) | controllable command execution | import scan of 51 binaries + 28 functions decompiled | ✅ closed: no WebUI/AT-controllable site; npdaemon configuration sources still need a PPP WAN |
| 39 | F | official image baseline (R4) | recoverable baseline | official ZIP local upgrade (5 runs in total) | ✅ all reached state 90; version unchanged, lock 0/10, the recovery channel is repeatable |
| 40 | G | dload signature/NV writer (R4b) | install-stage exploitability | package 15 streams + firmware dump + modem-wide search | ⚠️ the engine is in none of the available images (plain/compressed/UTF-16: 0 hits) → needs an additional partition |
| 41 | D | `ats` and updated-build re-check | AT parsing and new-build re-check | Ghidra 145 functions + import scan | ✅ `ats` input uses bounded copies; the updated build's webserver/router have no shell point, device has only a constant string |

---

## Appendix B: Knowledge and Protocol Inventory

> This appendix gathers the reusable knowledge obtained from this reversing work, grouped by protocol/subsystem; function addresses and offsets correspond to the firmware version analysed in this project.

### B.1 The three-layer NV chain and item mapping

```text
AP user space read(/dev/nv, item)
  → kernel huawei_nv_read_item (whitelist 0..0x2B)
  → SMEM 0x86 {item@+0xE0, magic@+0xE8}
  → huawei_notify_kernel_A2M(5)
  → modem qmi_huawei_server FUN_0205e50c
  → FUN_0205e3b0 → FUN_0205e37c (map table) → read/write
```

- item map (16 entries): `0x22→0xC546` (50502/E0), `0x2A→0xC4F2`, `0x2B→0xC542`, `0x00→0x1246`, `0x01→0xC3AB`, `0x04-09→0xC472-77`;
- SMEM 0x86 magic: read `0x2014AA55`, write `0x2014A5A5`, success `0x201455AA`;
- there are only 3 fixed NV consumers on the AP side: authconsole (0x22), webserver (0x2A), router (0/4-9/0x21).

### B.2 SIMLOCK verification system

- the code is 8 pure digits; `h2 = SHA256(SHA256(code) ‖ salt)`; `h2 == NV 0xC546[0:32]`; salt sits at record `0x40:0x60`;
- success: `*(GP+0x18fd)=0`; failure: global counter +1, limit NV 50005=10;
- slots: `"OEM" → 0xC546` (50502/E0), `"SIMLOCK" → 0xC547` (50503/E1); record length 0x19B (411 B);
- `^DATALOCK`: the E0 submission port, parameters `{code,1,IMEI}`, local RAM counter 3 per boot, failure `+CME ERROR:16`, does not consume `^CARDLOCK`;
- `^SPWORD`: independent 3-attempt limit, never touched throughout.

### B.3 dload / HDLC

- framing: `7E | payload | ~crc16_x25 LE | 7E`; the CRC is inverted and little-endian;
- NAK example: `7e 03 00 06 9e4c 7e`;
- 1,200+ frames × 5 header models blind-probed, all NAK → hard session gate; the official upgrade flow does not use dload, it uses HTTP upload;
- there are only 3 composition-script executors: the factory shell, boot-time `start_usb` (fixed 1F01), and `mbimd` (the only automatic one).

### B.4 five upgrade container and content lock

- format: `[prefix][main header 55AA5AA5][per stream: header + comp_size + CRC16-X25 header check + chunk table + zlib stream + pad]`; 17 components;
- validation: runtime MD5 re-read; local upgrade requires the component's compressed stream to be byte-identical to the original (content lock); the expected MD5 for online upgrade comes from the server and is forgeable;
- the official same-version ZIP has been locally upgraded 5 times in total, reaching 90; content modifications return 70 at the apply stage.

### B.5 nvram.bak configuration plane

- line-by-line base64 (each line = inner line + terminator); ends with `<checksum>decimal</checksum>`;
- checksum = Σ `cl_CRC16`(raw line including `\r\n`) mod 2^32; the last line (the checksum line) does not take part;
- `cl_CRC16`: CRC16-CCITT table-driven MSB-first, poly 0x1021, init 0, xorout 0;
- the restore pipeline `0x736C` has three stages (failure codes 103016/103017/103018); success value 5, reboot on success;
- the path field is strictly validated: only the 22 established files can be written, path traversal is rejected (103018).

### B.6 Online update / fake FOTA

- `update/config.xml` server keys (ip/port/ssl/virtual_directory/query_url/test_url/submit_url/proxy_*) are writable;
- gating `FUN_00013340`: passes only when any of dialup/wifi/cradle == 0x385 (901);
- CheckNewVersion → `<status>0` + component{N} (WEBUI/ISO/FIRMWARE1/2/MIX) `<url>` → `increment|full/filelist.xml`;
- filelist field order: `dpath`, `md5` (uppercase), `size`, `spath`; a component needs `<version>`; use `<component>`; the validate MD5 short-circuit requires changing the content;
- state machine: 10/11/12/13/14/20/30/40/50/60/70/90/100; once the download completes, POST `/china/UrlCommand/UpdateDone.aspx`;
- the download always lands at `/online/online_update.zip`; install = partition-level flashing (webui=yaffs2, firmware1/2/iso=raw FW).

### B.7 Writable configuration family and STA gating

- 22 files: `userdata/{dialup,lan,firewall,device,global,webserver,upnp,wifi,wifistation,statistic,update,network,cradle}/config.xml`, several XML files; `/data/tool/{toolsflag,toolsipfilter*,dialup_filterlist}.xml`;
- `wifistation/config.xml <sta_policy>=2` → STA 901 → the online-check gating passes; the dedicated private-network IP is allowed.

### B.8 aboot image structure and boot decision

- the `s002` stream = 0x28-byte wrapper header + image body; `+0x0C` load address 0x07E00000, `+0x10/+0x14` size 0x1CCB0;
- after the base address was corrected, the string-reference hit rate went from 71/757 to 447/757;
- `aboot_init`: key scan → `set_newboot_from_smem` (SMEM 0x86, 0xCD0) → choose the boot exit;
- `bootbk` entry: `g_recovery == 0x5245434B` (RECK); nothing in the whole firmware writes that value;
- `target_is_emmc_boot()` is a stub returning the constant 0; the fastboot command table exists, but nothing calls `fastboot_init`.

### B.9 Kernel dload / nark

- `dload_init` → `nark_api_init`: `huawei_smem_alloc(0x86, 0xCD0)`, mapping `+0x2C/+0xA8/+0xB0/+0xD0/+0xDC/+0x508`;
- `misc_register` (`/dev/dload_nark`, minor 255) + `/proc/dload_nark`;
- `visit_dload_smem` selector: 0→`+0x2C="NFLD"`, 3→`+0xA8="PUDS"`, 6→`+0xD0="PSND"`, 9/0xB→`+0xB0="RNDS"/"RNNO"`, 0xF→`+0xDC="DLCF"`;
- boundaries: the node defaults to `0600 root:root`; `/proc` is read-only; the table has no `+0xB8`/`+0xC8`.

### B.10 Updater write path and version lock

- `cl_gz_uncompress_to_Flash(fd, dev, out)`: gunzip straight to the target, no signature/hash/version check, the only gate `dev == 6`;
- but `osa_flash_*` is a file backend: indices 0..5 → `/online/firmware1.bin`, `firmware2.bin`, `iso.bin`, `web_ui.bin`, `/data/userdata/update_info.bin`, `/online/update_state.bin`;
- local-upgrade version check: the last segment of the in-package version string == the last segment of `AT+CGMR`; controlled by two bypaths, `check_rule` and the board version prefix;
- the component type is decided by the uploaded filename (`HILINK` + whether it contains `_`).

### B.11 WebUI access-control model

- `authority.xml` five tables: `limited_res_type_list`, `limited_fs_res_type_list`, `not_need_login_visited_res_list`, `need_login_get_restful_list`, `not_need_login_post_restful_list`;
- path allowlist `FUN_000288B0(path, mode)` = prefix match; when `mode==2` a `realpath` is done first;
- the form `csrf_token` field value must carry the `csrf:` prefix; AJAX uses the `__RequestVerificationToken` request header;
- result codes: 125003 = no permission/path denied; 100001 = parameter or content error; 100005 = parameter validation failed (IP); 100002 = SD/file unavailable.

---

## Appendix C: Unknowns and Missing Information

> This table gathers "what we still do not know". `Priority` is ordered by "could it change the conclusion": P0 = could change the conclusion; P1 = affects scope/feasibility; P2 = affects only reproducibility or wording.

| # | Unknown | Why it matters | How to fill it | Cost/constraint | What it could change once filled | Priority |
|---|---|---|---|---|---|---|
| 1 | whether E0 (50502) and E1 (50503) share the same code; the NV item/read method of E1 | `^DATALOCK` verifies only E0; whether the final submission covers E1 is unconfirmed | statically compare the two verification chains, the slot table and the NV item map; read-only probing | low | affects the reliability of "oracle verification → single submission" | P0 |
| 2 | the handshake/command sequence of a real dload session | 1,200+ blind frames established no session; the hard session gate is unsolved | capture a real flash session, or obtain a sample of the corresponding flashing tool | medium-high | could reopen the dload flashing path | P0 |
| 3 | which partition holds the R4b signature implementation | whether the install stage is exploitable is unknown | dump the missing partition, or obtain an image containing the signature engine | high (flashing/hardware) | could reopen the install/signature bypass | P0 |
| 4 | whether the network daemons (hostapd 2.1 / dnsmasq 2.51) have a usable RCE | this is the only remaining software surface | CVE search + exact version/configuration check + controllability verification | high | an RCE that does not depend on the WAN could yield AP execution | P0 |
| 5 | NAND model/package/test points/programmer | C6 is the only hardware route whose loop can be closed with existing knowledge | disassemble to identify the chip model, find the test points, prepare a programmer | high | could turn the non-software path from "assessed" into "executable" | P1 |
| 6 | the EDL firehose programmer for the MDM9635 | C5 allows full-disk read/write, but no programmer is in the package | find platform tools/leaked packages/same-platform cases | high | could yield full-disk read/write | P1 |
| 7 | whether `0:OEMINFO` is writable; whether the `^NVRESTORE` version check has a bypass | the current NV write path is judged a closed loop | static analysis + find a primitive that writes 0:OEMINFO | medium | could reopen the NV write path | P1 |
| 8 | whether the same holds for other firmware versions | the current conclusions rest on a single device/single firmware | collect ZIPs/devices of other versions and diff them | medium | affects the applicability of the conclusions | P1 |
| 9 | eligibility, price and success rate of official/third-party services | the shortest path outside the constraints | official policy/customer-service enquiries; market research | low | does not change the technical conclusions, but affects "whether it can actually be unlocked" | P1 |
| 10 | the publication boundary of raw evidence/scripts/samples | affects reproducibility and credibility | decide what can be published and what must be redacted or withheld | low | does not change the conclusions, but affects verifiability | P2 |
| 11 | side effects/lock state after `^SETHWLOCK` changes the code | the R2 conclusion is "changes the code only, no unlock", but the side effects were not measured item by item | pure static + controlled experiment (must be rollback-capable) | medium | could affect the lock-state judgement (low probability) | P2 |
| 12 | the triggerability of specific hostapd WMM / dnsmasq CVEs | the network-plane assessment currently stops at "medium-risk candidate" | a reproduction environment for the specific CVE + device configuration check | medium-high | could turn "candidate" into "usable/unusable" | P2 |

---

## Appendix D: Versions, Images and Reproducibility

### D.1 Device and firmware

| Item | Value | Note |
|---|---|---|
| Device | HWD31 / au Speed Wi-Fi NEXT W01 / KD02 / MIZUHO | different names for the same device |
| USB forms | `12d1:1F01` (CD-ROM) → `12d1:1442` (service mode) → 1441/1566/1001/155E etc. | composition switching |
| Firmware version | `11.411.13.80.824` | the device's current firmware |
| WebUI version | `14.100.03.46.04` | corresponds to the official ZIP |
| Lock state | `^CARDLOCK: 1,10,0`; `^DATALOCK: 1` | 0/10 consumed |
| Update state | `CurrentComponentStatus=90` | after reproducing the official baseline |
| Coverage | 2026-10-15 → 2026-10-21 | the period covered by this analysis |

### D.2 Official materials and images

| File | Size (B) | Hash | Note |
|---|---:|---|---|
| official upgrade ZIP `Mizuho_UPDATE_11.411.13.80.824_WEBUI_14.100.03.46.04.ZIP` | 92,970,520 | MD5 `4713e319c171cd899468c4ac212374cb`; SHA-256 `2d9260c269de40438db4da60a2c47dbce1bd4e5afba3d677256afab335434c54` | matches the device's current version |
| `解包.zip` | 102,162,305 | SHA-256 `210b1c7ccf397bed8c2d90cf62488fe699fd44220a57060b441212bf92f19b87` | 17 members, including the file12 firmware package |
| `modem_full.elf` | 48,737,368 | SHA-256 `d494d044e8e9a240309c4f882433f63f042307e4171efe4264b1ec3e45113e11` | rebuilt from file12, 21 program headers |
| `boot5_vmlinux.elf` | 13,511,512 | SHA-256 `9e744fdfed1dc38b7f7d6bb4aa48b2a57a52f4d6e6b83278e396ea1822de18e0` | 68,908 kallsyms symbols |
| `modem33_full.elf` | 48,813,340 | SHA-256 `140958c4c98bfab50088ba9dc61376157b42bb2e775c9dfc515d2cf1802b3d5d` | second modem image |
| `boot5_kernel.bin` | 4,799,176 | SHA-256 `2947453d8c2e0c29d9fabb5933214144aca36aba50ee208263a36576b60bc1ac` | raw kernel image |
| `boot5_ramdisk.img` | 855,368 | SHA-256 `4a9654bb91d1b9997a1e66f1e66e2ada4483d9c213f18273e2a26addbd8ea7e4` | raw ramdisk image |

> Hash method: SHA-256; the official ZIP additionally records its original MD5. Image hashes not listed here belong to the "missing information" (see Appendix C).

### D.3 Tool environment

| Tool | Version/note |
|---|---|
| JDK | Temurin `21.0.12.1+1` |
| Ghidra | `12.1.4 PUBLIC` (Hexagon support) |
| Decompilation/analysis | Ghidra projects + REA + custom scripts |
| Device-side tools | AT terminal, WinUSB/raw-bulk scripts, WebUI automation scripts, FOTA test server |

### D.4 Reproducibility notes

**Reproducible**:
- the baseline flow of a local upgrade to 90 with the official ZIP (5 runs in total);
- AT read-only queries (`^CARDLOCK?`, `^DATALOCK?`, `^RSFR` etc.);
- `/nvram.bak` download, decoding, restore and the 22-file write primitive;
- the CheckNewVersion/filelist/download chain of the fake FOTA;
- the static conclusions on the lock mechanism, NV mapping, aboot/updater/WebUI (on the same firmware version).

**Not reproducible from this repository alone**:
- the original scripts, samples, Ghidra projects and binary artifacts were not published with the repository;
- hardware paths such as UART/EDL/NAND were not executed;
- unlisted image hashes are missing information (see Appendix C);
- different firmware versions may differ and must be re-checked.

**Verification advice**:
1. first compare the image hashes in D.2 to confirm that the firmware version matches;
2. establish the 90 baseline with the official ZIP before starting any write operation;
3. respect the `^CARDLOCK` red line: no blind attempts, no submission of unverified codes.

---

## Appendix E: Capabilities, Tools and Data Inventory

### E.1 Capabilities obtained

| Capability | Status | Note |
|---|---|---|
| Direct AT channel | ✅ | COM26 @115200; read-only probing is stable |
| Complete WebUI/WiFi session | ✅ | login, configuration read/write, `/nvram.bak` download, upload, FOTA configuration |
| 22-file configuration write | ✅ | checksum cracked + restore pipeline; path field strictly validated |
| FOTA download chain | ✅ | CheckNewVersion → filelist → download → write to disk → report back (30→40→60) |
| `^DATALOCK` attempt-free oracle | ✅ | 3 per boot; does not consume CARDLOCK |
| Official image baseline | ✅ | official ZIP local upgrade, 5 runs in total, reaching 90 |
| AP code execution (root) | ❌ | not obtained through adb/diag/configuration injection |
| Read NV 0xC546 (140 B) | ❌ | protocol decoded, AP execution missing |
| Write NV / unlock | ❌ | all write channels closed |

### E.2 Tools and data (not published with the repository)

| Tool/data | Purpose | Key metric/status |
|---|---|---|
| Brute-force tool | offline brute force over 8 pure digits | about 5.84M hash/s, about 17 seconds per run |
| `nvdump` | 304 B ARM EABI static program, open/read `/dev/nv` | can read the 140 B of 0x22 (needs an execution environment) |
| nvram.bak codec | base64/checksum computation, generating a legal backup | already used for the 22-file write primitive |
| Differential-oracle samples | checksum fitting/verification | `checksum_probe_dataset` etc. |
| Fake FOTA server | CheckNewVersion/filelist/download/report | 30→40→60 already working |
| WebUI automation | login, csrf, upload, STA, status polling | supports batch experiments |
| five container tool | unpack/repack/validate | 17 components; used for the content-lock experiments |
| Ghidra projects | modem/kernel/aboot/updater/WebUI decompilation | 20+ projects |
| Firmware extraction trees | modem/kernel/rootfs/app/WebUI/ramdisk | used for static analysis and rebuilding |
| Configuration samples | nvram.bak baseline, crafted_*, five_mod_v1/v2 | used for restore/injection experiments |

> The tools and samples above were used only for this project's analysis and were not published with the repository; to reproduce anything, first check the firmware version per Appendix D and observe the safety red line.
