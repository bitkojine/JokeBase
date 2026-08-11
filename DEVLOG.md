# JokeBase append-only development log

This is the project flight log. New facts are appended at the end. Existing entries are not edited, reordered, deleted, or rewritten when later evidence changes our understanding. Corrections must be new entries that identify the earlier statement being corrected.

The log may contain artifact hashes, byte counts, binary offsets, test inputs, observed behavior, rejected hypotheses, control decisions, and links to retained raw fragments. It must never contain implementation source, WAT, compiler input, generator programs, pseudocode that reconstructs the implementation, or decompiled representations.

Git history is an additional integrity layer, not a substitute for this rule. A work-in-progress entry records an observation, not a promotion claim.

## 2026-08-11 — control record 0001

- Context: unpromoted five-table integer-catalog candidate derived from promoted sequence 18.
- Artifact: 13,903 bytes; SHA-256 `c132d49e1bab440a4b52b726abc671360336f2ad8ca3cd1c22a0f4e779591876`.
- Intervention: append a raw snapshot preflight function and call it before snapshot state mutation.
- Predicted effect: reject duplicate non-NULL values in unique/primary-key table slots and reject NULL primary-key rows; preserve legal duplicates in nonunique tables and multiple NULLs in nullable unique tables.
- Observed effect: four crafted semantic corruptions rejected; four rejections atomic; two nonunique duplicate images accepted; two nullable-unique multiple-NULL images accepted; valid mixed-state round-trip exact.
- Structural campaign: 3,685 wrong lengths, 15 catalog metadata corruptions, and 320 invalid NULL-flag corruptions rejected with zero traps and zero atomicity failures.
- Control conclusion: snapshot import must re-establish semantic database invariants, not only structural memory safety.
- Promotion status: not promoted.

## 2026-08-11 — control record 0002

- Observation: a 10,000-case generated integer membership campaign passed with pinned lowercase `null`, but uppercase `NULL` operands were rejected.
- Intervention: extend the raw membership body by 14 bytes using an existing uppercase token already resident in module memory.
- Artifact: 13,917 bytes; SHA-256 `ea10556fc44e69deb0085b6f5a42433804c2243f1d8cb89a41309270174b5eed`.
- Observed effect: 10,000 mixed-case generated comparisons passed, including 835 uppercase NULL operands and 4,591 expected SQL NULL results.
- Control conclusion: keyword-case variation is a high-value metamorphic input dimension.
- Promotion status: not promoted.

## 2026-08-11 — control record 0003

- Objective: add bounded state for the pinned `t7`, `t8`, `t7n`, and `t8n` TEXT tables without hard-coding query answers.
- Chosen state envelope: four table slots; 64 rows per table; 32 unescaped UTF-8 bytes per non-NULL value; independent schema flags, counts, lengths, NULL flags, and fixed cells.
- Memory interval: `[34368, 43104)`; 8,736 bytes.
- First create probe: 14,458 bytes; SHA-256 `837221b87a33b12ff09e30d120e9057a0a73349615f5078186c0e2d54788e299`.
- Observed effect: all four schemas created independently; duplicate create returned `-1`; old integer membership remained reachable.
- Failure found: `db_reset()` still cleared only the prior integer-catalog interval, so TEXT schema flags survived reset.
- Corrective artifact: 14,458 bytes; SHA-256 `2a98ffaa160ef0d9902171d01d0387ef60a69483999df3554f68fd4af81c83df`.
- Corrective observation: every byte in `[30720, 43104)` was zero after reset and schemas could be recreated.
- Control conclusion: every state-space expansion requires a reset-boundary test before feature tests.
- Promotion status: not promoted.

## 2026-08-11 — control record 0004

- Generic TEXT insertion probe initially accepted a 65th row despite a designed capacity of 64.
- Fault localization: the intended positive `64` was encoded as a one-byte signed LEB128 immediate and decoded as negative `64`.
- Corrective artifact: 15,069 bytes; SHA-256 `9ec930de1b4ecb5b1570695ed705c1a7b6de777170e05bba11996c981b665d5a`.
- Observed effect: row 64 succeeded; row 65 returned `-3`; the stored count remained 64.
- Other measured behavior: empty strings and spaces accepted; lowercase and uppercase NULL accepted; nullable unique tables accepted multiple NULLs; duplicate non-NULL unique values returned `-5`; malformed and overlength inputs returned `-10` without state changes.
- Control conclusion: signed-immediate encoding is an actuator-noise source and boundary constants require direct behavioral probes.
- Promotion status: not promoted.

## 2026-08-11 — control record 0005

- TEXT `INSERT ... SELECT` artifact: 15,651 bytes; SHA-256 `55863514af2b75e77c6e6437a1f012e29f5293a4f630e3b5a04936d9b2c4cfc0`.
- Observed successful copy counts: `t7` to `t8`, 3; `t7` to `t7n`, 3; `t7n` to `t8n`, 4 including one NULL.
- Failure gates: missing source `-2`; missing destination `-2`; capacity `-3`; unique conflict `-5`.
- Atomicity observation: every rejected copy left the complete 8,736-byte TEXT state interval unchanged.
- Promotion status: not promoted.

## 2026-08-11 — control record 0006

- Objective: add table-backed TEXT `IN` and `NOT IN` for direct-table and `SELECT *` subquery RHS forms.
- Rejected candidate 1: 16,449 bytes; SHA-256 `cb1a41335c42da52e0e7f46680e48b43a2f2376b4e30c58392fb108bebb87a20`; WebAssembly validation failed because an `i32.add` lacked an operand.
- Rejected candidate 2: 16,444 bytes; SHA-256 `96da1da89988a268a963aab800f5023242989268b14ce31d69491deaa798fef6`; validation failed because a missing block opcode caused the following byte to be interpreted as an invalid memory index.
- Rejected candidate 3: 16,445 bytes; SHA-256 `fb6a218e3a93878146cfb966f7a67420ea865cb937872858c84032b40d05479c`; validation failed at an invalid subtraction stack boundary.
- Candidate 4 validated but rejected every TEXT query because the direct-table and subquery branches were reversed.
- Candidate 5 accepted direct-table forms, but matches were reported as misses and subquery forms were rejected. Two causes were measured: a subquery-prefix length encoded as 13 instead of 15, and an SQL-relative operand offset passed where an absolute memory address was required.
- Current candidate: 16,445 bytes; SHA-256 `57269c546eed810c65ce35eef3cb9f9aab2c03c8250d136e1f4eb726ee7e19bb`.
- Current observation: 96 fixed comparisons passed across four tables, both operators, both RHS forms, matches, misses, NULL-containing tables, and lowercase/uppercase NULL operands; integer-list fallback remained correct.
- Control conclusion: validation is the fast structural loop; staged parser probes and populated semantic comparisons are separate necessary loops.
- Promotion status: not promoted; TEXT snapshot integration and broader generated testing remain open.

## 2026-08-11 — control record 0007

- Predicted persistence delta: append exactly 8,736 TEXT-state bytes to the existing snapshot suffix, producing a 12,384-byte combined integer+TEXT catalog image; preserve legacy and integer behavior.
- Snapshot candidate: 16,920 bytes; SHA-256 `23d72532254a692718a0fe447430ad7b3dcc1b398cd68c918ca5079b236ec659`.
- Snapshot format identity was advanced so the new reader does not silently interpret the earlier shorter format as current.
- First observation: a mixed legacy, integer-catalog, and TEXT-catalog snapshot was 12,420 bytes; its final 12,384 bytes exactly matched live catalog memory; reset cleared the complete catalog; restore returned success and reproduced every catalog byte; representative integer and TEXT queries retained their results.
- Hostile-image observation: 523 crafted TEXT corruptions were rejected atomically with zero traps. Coverage was 8 schema/count fields, all 256 fixed text-length positions, all 256 fixed NULL-flag positions, one active NULL row with a nonzero length, and duplicate values in both unique TEXT tables.
- Legal-image observation: a duplicate in a nonunique TEXT table and multiple NULLs in each nullable unique TEXT table were accepted and restored byte-exactly.
- Control conclusion: the persistence loop now observes both structural shape and semantic uniqueness across the expanded state space.
- Promotion status: not promoted; exhaustive wrong-length testing, generated query testing, and the pinned-suite frontier measurement remain open.

## 2026-08-12 — control record 0008

- Full pinned-suite candidate: 18,412 bytes; SHA-256 `ecf9cfda4e06165ae776a88cbff695e4a689abe92d02524f326c42a270fa4918`.
- Pinned suite: `tests/upstream/sqlite-sqllogictest-in1.test`; SHA-256 `83d8958a4f86de196a0756e548f24c71db6edf2679d70393d546d542c84fc2fd`.
- Measured result: all 27 SQLite-enabled setup statements and all 187 SQLite-enabled queries executed through final line 1155 with zero failures. This is the complete pinned file, not a contiguous-prefix estimate.
- Dispatch failures encountered before the green result: the TEXT parser initially claimed `NULL` queries for integer tables; an older scalar-list evaluator claimed lowercase NULL-left TEXT lists before the new helper; a temporary function-34 pre-dispatcher then interfered with `t3`; and the first observer hook reused an existing scalar-parser local. Each interference was isolated and removed or given private state before the complete replay passed.
- Control improvement: narrow observers now use a reserved local and are placed immediately before the controller they must override; unrelated syntax continues through the original dispatch chain.
- Known robustness defect: `SELECT null IN ('a',)` is still accepted even though the trailing comma is malformed. It is outside the pinned file but blocks promotion until corrected and regression-tested.
- Remaining gates: correct the malformed-list fallback, rerun snapshot length/corruption campaigns against this exact hash, run generated TEXT membership and stateful campaigns, preserve raw fragments and evidence, and audit the release boundary.
- Promotion status: not promoted.

## 2026-08-12 — control record 0009

- Correction to control record 0008: the malformed trailing-comma fallback was corrected in a later candidate; record 0008 remains the observation for its own earlier artifact.
- Final full-file candidate: 18,438 bytes; SHA-256 `ca3c9d4d3c0a76767281410e1df5da713f1e5f40e26c5635b3106eb09458580b`.
- Pinned suite replay against this exact artifact: all 27 SQLite-enabled setup statements and all 187 SQLite-enabled queries in `tests/upstream/sqlite-sqllogictest-in1.test` executed through final line 1155 with zero failures.
- Malformed-input observation: `SELECT null IN ('a',)` now returns stable malformed-query error `-10` rather than being accepted.
- Nearby regression observations: valid TEXT lists retain SQL NULL behavior; empty-list and numeric-list behavior remain unchanged.
- Control conclusion: a green upstream suite is a necessary outer-loop sensor, but malformed inputs immediately outside the suite remain an independent promotion gate.
- Promotion status: not promoted; persistence and generated-state campaigns must run against this exact hash.

## 2026-08-12 — control record 0010

- Artifact under test: 18,438 bytes; SHA-256 `ca3c9d4d3c0a76767281410e1df5da713f1e5f40e26c5635b3106eb09458580b`.
- Mixed-state snapshot observation: 12,420 bytes total; the final 12,384 bytes exactly matched live integer and TEXT catalog memory; reset zeroed the complete catalog interval; valid restore reproduced the snapshot byte-for-byte.
- Exhaustive length campaign: all 12,420 shorter lengths from 0 through 12,419 and the single overlength 12,421 were rejected, for 12,421 wrong-length cases total.
- TEXT hostile-image campaign: 523 corruptions were rejected—8 schema/count violations, 256 invalid fixed-cell lengths, 256 invalid NULL flags, one active NULL row with a nonzero length, and duplicate values in both unique TEXT tables.
- Header campaign: mutations at each of the first five snapshot bytes were rejected.
- Across every rejected image: zero traps and zero atomicity failures; a newly written snapshot remained byte-identical to the pre-intervention state.
- Control conclusion: promotion evidence must be hash-bound; unchanged implementation regions do not justify carrying evidence forward from a different candidate hash.
- Promotion status: not promoted; generated semantic and stateful gates remain open.

## 2026-08-12 — control record 0011

- Artifact under test: 18,438 bytes; SHA-256 `ca3c9d4d3c0a76767281410e1df5da713f1e5f40e26c5635b3106eb09458580b`.
- Differential campaign: 10,000 generated TEXT membership queries across 100 generated database states matched SQLite 3.51.0 exactly, with zero mismatches.
- Measured semantic coverage: 6,791 SQL NULL results, 1,237 uppercase-NULL operands, 5,005 direct-table forms, and 4,995 subquery forms.
- Test-controller failure found before the accepted run: selecting dimensions from the low bits of a linear congruential stream created severe correlations and misleading coverage. The candidate binary was unchanged; the campaign was rejected and rerun with a deterministic xorshift stream plus explicit coverage counters.
- Stateful model campaign: 40,000 operations with zero failures and zero traps—14,671 inserts, 4,026 atomic table copies, 11,994 modeled membership queries, 4,939 snapshot/save-or-restore operations, 2,363 reset-and-recreate cycles, and 2,007 malformed-statement atomicity probes.
- Control conclusion: deterministic generation is insufficient by itself; generated campaigns need measured dimension coverage so defects in the test signal are observable.
- Promotion status: not promoted; raw-fragment preservation and complete release-evidence audit remain open.

## 2026-08-12 — control record 0012

- Artifact under test: 18,438 bytes; SHA-256 `ca3c9d4d3c0a76767281410e1df5da713f1e5f40e26c5635b3106eb09458580b`.
- Regression corpus: `tests/generated/text-list-seq17-sqlite-3.53.3.jsonl.gz`; compressed SHA-256 `e5ca0679ae89dc5e9594a8d9a59bee6f612bd486bfff281c0abfcd324aa7d1a8`; uncompressed SHA-256 `b3badf8c446d367acf12d036d0baa42036ff8067a382f9e6d9e8626f4dcbaa91`.
- Exact replay result: all 50,000 materialized SQLite-expected records passed with zero failures.
- Preservation action: the complete candidate and 20 exact raw body/data fragments were retained under `artifacts/wip-text-catalog/` with byte lengths and SHA-256 identities in its manifest.
- Control conclusion: a materialized prior corpus is a valuable regression sensor when new dispatch logic touches the same syntax family.
- Promotion status: not promoted; final ABI, capability-boundary, and release-evidence audit remain open.

## 2026-08-12 — control record 0013

- Artifact under test: 18,438 bytes; SHA-256 `ca3c9d4d3c0a76767281410e1df5da713f1e5f40e26c5635b3106eb09458580b`.
- SQL staging-window replay: three supported boundary placements and all nine declared rejected ranges passed; rejected ranges cleared results, did not trap, and left the complete snapshot state unchanged.
- Focused bounded-state replay: all four TEXT tables accepted rows through capacity 64, rejected row 65 with `-3`, and preserved state atomically; both unique TEXT tables rejected pre-capacity duplicate non-NULL values with `-5`; full-capacity `-3` retained precedence over duplicate `-5`.
- Nullable-unique observation: both lowercase and uppercase NULL insertions were accepted multiple times in each unique TEXT table.
- Control conclusion: broad stateful campaigns are supplemented by exact boundary/error-precedence probes because random histories may not isolate the required ordering.
- Promotion status: not promoted; documentation and evidence must be reconciled with the candidate before release.

## 2026-08-12 — control record 0014

- Compatibility intervention: produced a valid 36-byte snapshot from promoted sequence 18 containing legacy row 91, then supplied it to the 18,438-byte candidate while the candidate held independent row 73.
- Observed result: the candidate returned `-10`, did not trap, and retained its complete pre-import snapshot byte-for-byte.
- Compatibility conclusion: the expanded catalog snapshot is intentionally not backward-readable; sequence-18 images are rejected atomically rather than interpreted as the new format.
- Promotion status: not promoted; this breaking snapshot boundary must be explicit in sequence-19 evidence and ABI documentation.

## 2026-08-12 — control record 0015

- Promotion: sequence 19, 18,438 bytes, SHA-256 `ca3c9d4d3c0a76767281410e1df5da713f1e5f40e26c5635b3106eb09458580b`; parent sequence 18 SHA-256 `1d238226c11e693980b61527ced25dde91822a56131243f489e51193231491f1`.
- Local promotion commit: `0ef8111aeb68d0c03f0fd658a180c8774f3fb189` on `main`.
- Publication observation: the public GitHub main branch advanced to that exact commit; a fresh GitHub-served raw `JokeBase-v1.wasm` download hashed to the promoted sequence-19 SHA-256.
- Promotion status: promoted and publicly replicated.

## 2026-08-12 — control record 0016

- Preservation action: added `STORY-BRIEF.md`, a non-technical narrative guide grounded in the promoted artifact, public evidence, capability boundary, correction history, and stated limitations.
- Communication control conclusion: an opaque-artifact project needs an equally precise public story. The brief distinguishes WebAssembly bytecode from native machine code, one pinned test file from whole-suite conformance, evidence from proof, and a real bounded database from a general SQL implementation.
