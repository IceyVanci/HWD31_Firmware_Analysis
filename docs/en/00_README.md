# HWD31 Reverse-Engineering Study — Analysis Document Index

> This document index explains the purpose of the three documents, how they divide the work, the reading order, the terminology and the known information gaps.
> The three documents summarise the progress of this reverse-engineering effort and the information obtained, and assess what still needs to be supplemented.
> Coverage: 2026-10-15 → 2026-10-21; the original evidence, firmware images, scripts and samples are not released with this repository.

---

## 1. Document list

| File | Content | Intended reader |
|---|---|---|
| [`01_analysis_progress_history.md`](./01_analysis_progress_history.md) | **Exploration progress and results by area**: organised by lock mechanism, lock commands, device channels, firmware static analysis, NV reading, writing/execution, the five system layers and non-software paths; Appendices A–E contain the 41-row route matrix, the knowledge and protocol inventories, the unknown-information table, version identity and the capability inventory | Readers who want a systematic account of the conclusions and details in each area |
| [`02_progress_evaluation_and_analysis.md`](./02_progress_evaluation_and_analysis.md) | **In-depth analysis by area**: explains, area by area, why each was considered, its relationship to SIM Lock and what the results mean; §13 gives information sufficiency and confidence, §14 the conclusions, and §15 the methodology and decision retrospective | Readers who want to understand "why it failed, what the results show and how reliable the judgements are" |
| [`03_non_software_paths.md`](./03_non_software_paths.md) | **Non-software path assessment**: the conditions, cost and success rate of channels such as official/third-party, UART/JTAG, EDL and NAND offline reading; §4.11 lists the external information still missing for each channel | Readers who want to understand the possible paths that do not depend on a running system |

`01` presents the facts and results by exploration area, `02` explains the motivation, relationships and meaning of each area and assesses confidence, and `03` discusses non-software paths. The three documents can be read individually as needed; reading `01` §2 (overview) and §12 (conclusions) first, followed by `02` §10/§14, is recommended.

---

## 2. Suggested reading order

| Reader goal | Suggested order |
|---|---|
| A quick view of "how far we got and where we are stuck" | `01` §2 overview → §12 conclusions → `02` §10/§14 |
| Wants to know "what is left to do" | `02` §12 → `03` |
| Wants to understand, for each area, "why it was done and what the results show" | `02` §2–§9 |
| Wants to see whether the information is sufficient and how reliable the judgements are | `02` §13 |
| Needs to verify a particular mechanism conclusion | `01` §3–§8 (mechanism/channel/firmware/reading/writing) → `01` §11; details in `01` Appendix B |
| Wants to understand the five-layer system sweep | `01` §9 |
| Needs to assess what information is still missing | `01` Appendix C → `02` §13.2 |
| Reviewing the quality of historical decisions / retrospective methodology | `02` §15 |
| Wants to understand non-software paths | the whole of `03` |

---

## 3. Document notes

1. The facts in all three documents come from public sources and artefacts obtained by measurement in the project; unverified inference is marked "(inferred)", and key conclusions carry an evidence grade of A/B/C.
2. The device safety red line (`^CARDLOCK: 1,10,0`, 0/10 consumed) held throughout the entire history; no code was ever submitted to `^CARDLOCK`.
3. `01` and `02` have both been rewritten by "exploration area"; `03` is the non-software path assessment.
4. The documents cover 2026-10-15 → 2026-10-21; content added after that should first update the original record and then be written back into the corresponding sections.
5. **Information completeness**: the three documents are summaries, not a complete evidence archive; the original evidence, firmware images, scripts and samples are not released with the repository. Known gaps are in `01` Appendix C and `02` §13; capabilities/tools in `01` Appendix E; versions/images in `01` Appendix D.
6. This document is also available in an English translation; the two editions are kept in sync, with the Chinese edition prevailing where they differ.

---

## 4. Terminology

| Term | Meaning |
|---|---|
| SIM Lock | The device's SIM network lock; the objective of this project |
| NV 0xC546 / 50502 / E0 | Lock record slot; contains hash+salt, 0x19B (411 B) in total |
| NV 0xC547 / 50503 / E1 | Second lock record slot; shares the verification code with E0, but the NV item/slot differs |
| `^CARDLOCK` | Lock query/submission command; SET is always rejected on this image |
| `^DATALOCK` | E0 submission port; 3 attempts per boot, allowing candidate codes to be verified without consuming `^CARDLOCK` attempts |
| `/dev/nv` | AP-side NV read node; 12 B header + 1024 B data |
| dload | Qualcomm/Huawei flashing protocol; HDLC frames + CRC-X25; has a hard session gate |
| five container | Official upgrade package format; 17 components; local upgrade has a byte-level content lock |
| nvram.bak | Device configuration backup; line-by-line base64 + checksum; can restore 22 configuration files |
| FOTA | Over-the-air update; CheckNewVersion → filelist → download → install/report back |
| 22-file write primitive | A controlled write capability that writes 22 predetermined configuration files through a legitimate nvram.bak restore |
| R1–R4 / R4b | R1 log, R2 lock commands, R3 non-WAN execution point, R4 official baseline; R4b = dload signature engine |

---

## 5. Version and language status

| Item | Status |
|---|---|
| Chinese edition | Authoritative version; rewritten by area, with Appendices A–E, information sufficiency and confidence added |
| English edition | Synchronised with the Chinese edition; where they differ, the Chinese edition prevails |
| Coverage | 2026-10-15 → 2026-10-21 |
| Device/firmware | HWD31 (au Speed Wi-Fi NEXT W01); firmware 11.411.13.80.824; WebUI 14.100.03.46.04 |
| Last updated | 2026-10-21 |
| Original evidence | Not released with the repository; key versions/hashes in `01` Appendix D |

---

## 6. Translation provenance

| English file | Chinese source | Size | SHA-256 of the source |
|---|---|---|---|
| `docs/en/00_README.md` | `docs/zh-CN/00_README.md` | 4,888 B | `bbd09c191b5ca2e28b2766c3dc744827b31bce4e19151cf9df3927b46699a1e8` |
| `docs/en/01_analysis_progress_history.md` | `docs/zh-CN/01_分析进度历史.md` | 41,284 B | `dd2a20a6d5a07e09202ca31b82aa4ab40d79c9fb4600d5dd43621a80463a4e73` |
| `docs/en/02_progress_evaluation_and_analysis.md` | `docs/zh-CN/02_历史进度评估与分析.md` | 24,349 B | `ff1508793ce57dcce4c22aa47209b89d25d301aac1456b2d20165b0d574ad907` |
| `docs/en/03_non_software_paths.md` | `docs/zh-CN/03_非软件路径分析.md` | 11,271 B | `ef4acf907fe2f9db79d508cee5f2d4df3fc93126b88b974ae956d15998931709` |

> The Chinese edition remains the authoritative record. The table records the current Chinese source identities; if a Chinese source changes (its SHA-256 no longer matches), re-translate the affected sections rather than editing the English text independently.

---

*Translation of `docs/zh-CN/00_README.md`; the Chinese original remains the authoritative record.*
