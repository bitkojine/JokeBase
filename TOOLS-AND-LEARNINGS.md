# JokeBase tools and field notes

This document records how JokeBase is being developed, which tools participate, what each tool is trusted to establish, and what the experiment has taught us. The `.wasm` artifact is entertaining; this operational knowledge is the more reusable result.

It is deliberately not an implementation description. It contains no WebAssembly text, source code, generator, compiler input, pseudocode reconstruction, or decompiled representation of JokeBase.

## Experimental boundary

Humans and AI coding agents may work with:

- raw `.wasm` bytes and opaque raw body/data fragments;
- binary container metadata such as section lengths, function indices, byte offsets, memory regions, and hashes;
- the documented embedding ABI and snapshot layout;
- SQL inputs, expected outputs, error codes, and runtime state observable through the public ABI;
- public test-suite data and independent behavioral oracles;
- manifests describing lineage, provenance, capabilities, experiments, and failures.

They may not create, inspect, or retain:

- implementation source in any programming language;
- WebAssembly text (`.wat`);
- a compiler or generator program that emits JokeBase;
- compiler inputs or an intermediate representation;
- disassembly, decompiled source, source-like pseudocode, or a reconstructed control-flow listing.

This is a declared process constraint, not a mathematical proof about everything a participant might have seen. Git can prove which bytes were published; it cannot prove the complete private history of human or agent cognition. Claims about the process must therefore remain narrower than claims established by artifact hashes and repeatable behavior.

## Tool inventory

The version snapshot below was recorded on 2026-08-11 on Darwin arm64. Versions are evidence context, not a promise that every historical promotion used exactly this environment.

| Tool or facility | Recorded version or identity | Permitted role | What it does not prove |
| --- | --- | --- | --- |
| Node.js WebAssembly API | Node.js 26.5.0 | Validate, instantiate, invoke exports, inspect export names/arities, stage SQL bytes, and observe results or traps | General cross-runtime portability or SQL correctness |
| Node.js `DatabaseSync` SQLite | SQLite 3.53.3 in the recorded differential campaigns | Produce differential expected results for generated SQL inputs | Standards conformance independent of SQLite |
| SQLite command-line client | SQLite 3.51.0 in the current local environment | Manual reference checks and environment diagnosis | Reproduction of campaigns explicitly recorded against SQLite 3.53.3 |
| SQLite SQLLogicTest data | Pinned files and SHA-256 values under `tests/upstream/` | Supply public SQL inputs and expected results | Conformance with unexecuted records or the complete suite |
| `xxd` | 2025-08-24 | Render selected raw bytes as hexadecimal and convert explicitly chosen hexadecimal bytes back to binary fragments | Semantic correctness of those bytes |
| BSD `dd` | Darwin system tool | Copy already identified byte ranges into disposable candidate positions | Correct offsets, valid Wasm, or correct behavior |
| `shasum` | 6.04 | Calculate SHA-256 identities for artifacts, fragments, corpora, and pinned inputs | Authentic authorship or correctness of the hashed content |
| `jq` | jq 1.7.1-apple | Validate and query declarative JSON evidence and manifests | Truth of a `PASS` value without the underlying replay evidence |
| Git | 2.50.1, Apple Git-155 | Preserve append-only promoted artifacts, manifests, corrections, and parent relationships | That forbidden representations were never used privately |
| GitHub | Public repository at `bitkojine/JokeBase` | Off-machine backup, public inspection, and remote artifact replication | Authenticity unless commits/releases are signed and independently verified |
| AI coding agent | Codex, model/version dependent on the active session | Reason from English intent, binary metadata, runtime behavior, and test residuals; propose bounded byte mutations | Infallibility, process compliance by assertion, or correctness without external evidence |

Tools intentionally excluded from the JokeBase implementation workflow include language compilers, assemblers that consume WAT, `wasm2wat`, decompilers, disassemblers that reconstruct instructions, and source-generating build systems.

## What the raw-byte workflow looks like

Every change follows a feedback loop:

1. Select one coherent behavioral boundary from a pinned public suite.
2. State the expected behavior, existing invariants, failure semantics, and memory bounds.
3. Record the current promoted artifact hash and the binary regions expected to change.
4. Construct raw binary fragments and splice them only into a disposable candidate.
5. Recalculate every affected container length and index count.
6. Ask a Wasm engine to validate and instantiate the candidate.
7. Run focused success, failure, boundary, atomicity, and regression probes.
8. Run differential and property campaigns capable of exposing plausible incorrect implementations.
9. Reject the candidate on any trap, unexplained state change, parent regression, or documentation mismatch.
10. Preserve useful failed experiments as evidence when they teach a general lesson.
11. Promote only after the artifact, hashes, capability profile, suite boundary, and unsupported behavior agree.

The previous content-addressed artifact is always the safe fallback. A disposable alias is never the only retained copy of promoted work.

## What each evidence layer contributes

No single test result is allowed to carry the whole trust argument.

| Evidence layer | Question it answers |
| --- | --- |
| Wasm validation | Is the binary structurally and type-correct enough for this engine to compile? |
| Focused examples | Does the newly targeted behavior work in representative states? |
| Parent regression | Did the intervention preserve every previously declared behavior? |
| SQLLogicTest | Does behavior agree with pinned public input/output records? |
| Differential testing | Does behavior agree with an established database over a larger generated domain? |
| Metamorphic/property testing | Do relationships that should hold remain true without relying solely on another database's answers? |
| Stateful model testing | Do long operation sequences preserve the modeled state machine? |
| Robustness testing | Are malformed inputs rejected without traps or unintended mutation? |
| Snapshot testing | Can state be serialized, restored in a fresh instance, and rejected atomically when corrupt? |
| Hash and lineage checks | Are we testing and publishing the exact bytes we claim? |
| Independent replay | Can somebody other than the original agent reproduce the recorded result? |

Passing one layer does not imply passing another. In particular, Wasm validation says nothing about SQL semantics, and agreement with SQLite does not independently prove a SQL property.

## Lessons learned from real failures

### 1. WebAssembly validation is necessary and radically insufficient

A candidate can validate and instantiate while containing a semantically wrong address, branch, comparison, or snapshot layout.

During the additional-table work, five positive data addresses were encoded using unsigned LEB128 rules even though `i32.const` consumes signed LEB128. The module validated. At runtime those byte sequences became negative addresses and the first comparison trapped out of bounds. The candidate was discarded before state mutation.

The lesson is to treat validation as a container/type gate, never as a behavioral correctness claim.

### 2. ULEB128 and SLEB128 are different safety domains

Section lengths, vector counts, and function-body sizes use unsigned LEB128. Integer constants use signed LEB128. The same mathematical positive value can require a different final byte and even an additional byte in signed form.

Every changed immediate must therefore be classified by its binary grammar before encoding. “The numeric value looks right” is not an adequate check.

The catalog membership parser exposed both layers of this mistake in succession. It first compared table-name digits with numeric `4`, `5`, and `6` rather than their ASCII byte values. After that was corrected, the ASCII letters `t` and `n` were still encoded as single-byte signed constants. Their byte values have bit 6 set, so signed LEB128 interpreted them as negative numbers. A byte parser must ask both “what byte value represents this character?” and “how must that positive value be encoded for this particular Wasm immediate?”

### 3. One byte changes distant offsets

Adding a function changes the function-section count and payload length. Growing a body changes the code-section length. Extending initialized data changes both the segment length and data-section length. Any of those may change the number of bytes needed to encode the length itself.

Offsets must be recalculated from section boundaries after every size change. Reusing yesterday's downstream offset because the semantic change was “small” is unsafe.

### 4. Append new function indices when possible

Inserting a function in the middle renumbers every later function and can silently redirect existing calls. New helpers are therefore appended to the function vector whenever possible, and the existing fallback is extended to call the new final index.

This does not eliminate risk: the function count, type vector, code-body count, call immediate, and code-section size must still agree.

### 5. A validator can catch stack mistakes before behavior exists

The first generic catalog-insert body was rejected because one address expression reached `i32.add` with only one operand. The disposable module never instantiated. That failure localized the mistake to the new raw body before any SQL test or live state could confuse the diagnosis.

Immediate validation after each coherent splice keeps the feedback delay short.

The next candidate exposed a second stack-contract mistake: the retained integer parser consumes three operands—the SQL pointer, a start offset, and an end offset—rather than two operands containing a pre-added pointer and an end offset. Validation again rejected the module before execution. After that was corrected, behavioral tests found a different class of defect: the NULL recognizer counted the closing parenthesis and accepted only uppercase spelling, while the pinned suite used lowercase `null`. Structural validation found the stack mistakes; only behavioral testing found the SQL mistake.

### 6. Empty-state tests can hide completely wrong memory maps

The first cross-table `x+y` membership candidate used the wrong address for one table and still passed the newly targeted upstream cases because both tables were empty. No loop iteration reached the incorrect load.

Populated differential tests exposed it immediately. Tests must provide informative excitation: absent, empty, populated, full, matching, missing, NULL-containing, boundary-valued, and overflowing states are observably different conditions.

### 7. Arithmetic overflow can manufacture false SQL matches

WebAssembly signed 32-bit addition wraps. SQLite may evaluate an overflowing integer expression in a wider numeric domain. Comparing only the wrapped Wasm result can therefore report a match that does not exist mathematically.

The cross-table campaign deliberately included thousands of overflowing row pairs. Boundary values are part of semantic coverage, not merely parser coverage.

### 8. Persistence multiplies every schema change

Adding a table is not just adding a row array. It changes reset behavior, schema existence, counts, values, NULL flags, snapshot size, writer layout, reader validation, versioning, and restoration atomicity.

For the five-slot integer catalog, the snapshot reader was not accepted until it restored every catalog byte exactly and rejected:

- every truncated length plus a trailing-length case;
- invalid schema flags;
- counts above capacity;
- impossible absent-schema/nonzero-count combinations;
- every one of the 320 fixed NULL-flag positions when corrupted.

All rejection checks ran before live-state mutation.

Structural validity was still not enough. A host can supply a correctly shaped image containing duplicate non-NULL rows in a unique table or a NULL row in a primary-key table—states that SQL insertion would never create. The reader therefore gained a separate preflight pass over every active unique/primary-key row. Crafted duplicates in all three unique slots and a forged primary-key NULL were rejected atomically, while duplicates in nonunique tables and multiple NULLs in nullable unique tables remained legal. Persistence must re-establish semantic invariants, not merely deserialize bytes safely.

### 9. Exported memory changes the threat model

JokeBase has no imports and no ambient filesystem or network capability. That protects the host from ambient module access. It does not protect JokeBase from its host: the exported linear memory can be modified directly.

The SQL staging window limits what `execute` will read, but a hostile or careless embedder can corrupt internal memory before calling JokeBase. This is an ABI trust boundary, not a property the Wasm sandbox can solve internally.

### 10. Documentation is part of the executable contract

A hostile review found that `ABI.md` described `db_snapshot_read(ptr, len)` even though the promoted binary exports a one-parameter `db_snapshot_read(len)` and reads from the fixed buffer returned by `db_snapshot_ptr()`.

The binary behavior was correct; the public contract was not. Export arity, valid restoration, and short-image atomic rejection were measured, and the correction was recorded explicitly rather than rewriting history silently.

### 11. A recorded `PASS` is not automatically reproducible evidence

Early campaigns retained counts, seeds, summaries, and artifact hashes but not every generated input or an executable replay environment. Those records are useful attestations, but an outside reviewer cannot independently reproduce all of them from the repository.

The first improvement was to publish the exact 50,000-record text-list corpus, including every SQL input, SQLite expected value, compressed and uncompressed hashes, and replay totals. The remaining historical campaigns still have a weaker reproducibility status and are labeled honestly.

### 12. Benchmark-guided development needs a holdout

JokeBase is deliberately developed toward the next visible SQLLogicTest boundary. Passing that same boundary demonstrates regression progress, not unseen generalization. Generated hidden cases, metamorphic relations, and tests from independent engines are necessary to detect recognition of known strings without underlying semantics.

Future promotion evidence should distinguish development cases, public regression cases, and uninspected holdout cases.

### 13. Differential testing and property testing answer different objections

SQLite differential testing asks whether JokeBase agrees with SQLite over a generated domain. SQLancer-style properties such as ternary-logic partitioning ask whether internally related queries remain consistent without using SQLite as the expected-answer oracle.

Both are needed. Agreement can reproduce a reference-system quirk; a property can be satisfied by two consistently wrong answers. Independent evidence layers reduce correlated blind spots.

### 14. Public Git history is backup, not complete reproducibility

Content-addressed binaries and parent hashes mean promoted work is recoverable. Raw body fragments preserve valuable intermediate work. Neither automatically provides a reproducible construction process, because the experiment intentionally forbids a conventional source-level build recipe.

This tension should remain visible: artifact preservation is currently stronger than transformation reproducibility.

### 15. Cryptographic identity and behavioral trust are separate

A SHA-256 hash identifies bytes. It does not say those bytes are correct. A signed commit identifies an author's approval more strongly than an unsigned commit. It still does not prove the test claims.

JokeBase needs hashes, signatures, and behavioral evidence; none substitutes for the others.

### 16. Batch statements need a validation phase and a commit phase

`INSERT ... SELECT` can fail because the destination is absent, the source is absent, total capacity is insufficient, a unique value conflicts, or a NULL is incompatible with the destination contract. Copying rows while discovering those conditions would leave a partially applied statement.

The catalog copy helper therefore resolves the complete operation and checks both schemas, total capacity, NULL compatibility, and existing destination conflicts before copying its first row. Adversarial probes snapshot the whole catalog before each rejected operation and compare every byte afterward.

This work also exposed a cross-layer invariant: a copy helper may trust source uniqueness only if every possible state-entry path establishes it. Ordinary insertion does, but a crafted snapshot could bypass that assumption unless the snapshot reader validates unique-table contents. Atomic statement logic and persistence validation cannot be designed independently.

### 17. SQL spelling variants are cheap hidden tests with high value

A 10,000-case membership campaign initially used the pinned lowercase `null` spelling and passed. Repeating the same semantic cases with uppercase `NULL` immediately found an inconsistent parser path: insertion accepted both spellings, but catalog membership accepted only lowercase.

The repair reused an existing uppercase token already present in linear memory and extended the raw comparison path by 14 bytes. A mixed-case 10,000-case campaign then passed, including 835 uppercase NULL operands. Keyword case, whitespace, signed zero, and equivalent parenthesization are useful metamorphic dimensions because they change syntax without changing SQL meaning.

### 18. Control records must separate prediction, intervention, and observation

Raw-byte work becomes harder when the intended effect, actual byte change, and measured behavior are remembered as one story. They are now recorded separately:

- the predicted artifact and behavioral deltas before an intervention;
- the exact parent/candidate hashes and changed binary interval;
- structural validation results;
- behavioral observations, including unexpected effects;
- the promotion decision and remaining uncertainty.

`DEVLOG.md` is the append-only flight log for these facts. Earlier entries are never cleaned up after a later diagnosis; a correction is a new record. This preserves failed hypotheses and makes control quality auditable instead of reconstructing a success narrative after the fact.

### 19. Signed LEB128 makes familiar positive constants non-obvious

WebAssembly instruction immediates are not all unsigned byte values. A one-byte signed LEB128 encoding whose sign bit is set can represent a negative number even when the raw byte looks like a familiar positive boundary. A planned capacity constant of 64 was encoded in one byte and decoded as negative 64, allowing a 65th row through the intended guard.

Constants at signed-encoding boundaries now require direct below/at/above behavioral probes. Reading the intended decimal value from a raw byte is not evidence of the value the engine decodes.

### 20. Parser controllers interfere unless observation points are narrow

Several independently correct query helpers failed when composed because an earlier dispatcher claimed syntax intended for a later one. NULL-left TEXT lists were especially hazardous: a broad observer could repair one case while stealing integer-table or scalar-list queries.

The reliable pattern is to place the smallest possible observer immediately before the controller it must override, give it private state, and prove that unrelated syntax still traverses the original chain. Complete-suite replays are the outer-loop sensor for cross-controller interference.

### 21. Generated tests need coverage telemetry

A deterministic test run can be repeatable and still be badly distributed. Selecting several dimensions from the low bits of a linear congruential stream produced correlated choices: almost every query followed the same combinations, despite a nominally large case count.

The campaign was rejected, its signal generator changed, and the rerun reported counts for NULL outcomes, keyword casing, direct-table syntax, and subquery syntax. Generated-case totals without dimension totals are weak evidence.

### 22. Evidence belongs to an exact artifact hash

It is tempting to carry a passing persistence campaign forward when later changes appear confined to query dispatch. That relies on an implementation argument nobody is permitted to inspect and ignores accidental binary interference.

The full snapshot round-trip, 12,421 wrong-length cases, 523 semantic and structural corruptions, and header mutations were therefore rerun against the exact same hash that passed the complete pinned upstream file. Promotion gates attach to bytes, not to confidence that a region was probably unchanged.

## Failed experiments are retained knowledge

WIP manifests may describe rejected candidates when the failure teaches something reusable. A rejected entry should include:

- candidate byte length and SHA-256;
- whether Wasm validation succeeded;
- the first observed failing behavior;
- whether any state mutation occurred;
- the corrected hypothesis or next control action;
- an explicit statement that the candidate was never promoted.

This turns failures into experimental data without preserving a source representation.

## Current limitations of the method

- Most historical generated campaigns are not yet independently replayable from repository contents.
- There is no repository-enforced continuous promotion gate yet.
- Runtime coverage is concentrated in Node/V8 rather than multiple independent Wasm engines.
- The source-free process is a transparent policy claim, not remotely provable.
- Public development cases and final evaluation cases are not yet adequately separated.
- Commit and release signing are not yet consistently established.
- The repository preserves exact artifacts more strongly than it preserves repeatable transformations between them.

These are research questions and engineering debt, not details to hide behind the joke.

## Documentation required for every future promotion

Every promoted artifact should leave behind:

1. exact artifact byte length and SHA-256;
2. parent artifact hash and capability delta;
3. changed raw-fragment identities and binary metadata;
4. tool/runtime versions relevant to validation and reference answers;
5. pinned public-suite file hashes and the exact passed boundary;
6. materialized generated inputs or an equally strong independent replay path;
7. focused, differential, property, stateful, robustness, and persistence results as applicable;
8. rejected-candidate lessons that affected the final design;
9. ABI and snapshot compatibility statements;
10. explicit unsupported behavior and threat-model boundaries;
11. remote publication verification;
12. signatures or attestations when the publication path supports them.

The experiment succeeds only if its claims become more inspectable as the binary becomes more opaque.
