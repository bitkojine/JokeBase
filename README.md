# JokeBase

JokeBase is an experimental SQL database distributed only as WebAssembly bytecode.

Its defining constraint is absolute: neither humans nor AI agents may read or write implementation source code, WebAssembly text, compiler inputs, generator programs, or decompiled source. Development operates only on raw `.wasm` bytes, binary metadata, runtime behavior, and external tests.

## WebAssembly is bytecode, not native machine code

WebAssembly (Wasm) is a portable, low-level bytecode format. A `.wasm` binary does not directly contain the x86-64 or ARM instructions executed by a particular CPU. A host engine such as V8, SpiderMonkey, or Wasmtime validates the module and typically translates it into host-specific native machine code using just-in-time (JIT) or ahead-of-time (AOT) compilation. Ahead-of-time compilation may happen before the module is loaded for execution.

The conventional compilation flow is:

```text
Source code (Rust, C++, Go, ...)
    │
    ▼  language compiler / LLVM / Emscripten
WebAssembly bytecode (.wasm)
    │
    ▼  Wasm engine JIT or AOT compiler
Native host machine code (x86-64, ARM, ...)
    │
    ▼
CPU execution
```

JokeBase removes the first step. Its `.wasm` bytecode is constructed and evolved directly from English intent, raw binary structure, and observed behavior. Native machine code is still produced later by the host Wasm engine and is outside JokeBase's preserved implementation artifact.

### Key distinctions

- **Portable virtual instructions:** traditional native machine code targets a particular instruction-set architecture. Wasm defines a hardware-independent virtual instruction set that compatible engines can translate for their host architecture.
- **Abstract stack machine:** Wasm semantics are described using an operand stack. This is a conceptual execution model, not a requirement that an engine materialize that stack in hardware. Engines commonly map values and operations onto native registers and instructions.
- **Sandboxed capabilities:** a core Wasm module has no ambient access to the host filesystem, network, devices, or operating-system services. The embedder must explicitly provide such capabilities through imports. The sandbox isolates the module from the host, but it does not magically prevent a program from corrupting its own data layout inside its linear memory.

### Binary and text representations

The WebAssembly standard defines both a compact binary format (`.wasm`) and a human-readable text format (`.wat`). They describe the same kind of module structure, but they serve different purposes: engines distribute and consume the binary representation, while people and tools often use the text representation for inspection, debugging, or hand authoring. Support for parsing the text format is not required of every Wasm embedder.

JokeBase uses only the binary representation. Under the experiment's rules, neither a human nor an AI agent may create, read, preserve, or derive a `.wat` representation of JokeBase. The binary is tested through its externally observable behavior instead.

These descriptions follow the official [WebAssembly specification introduction](https://webassembly.github.io/spec/core/intro/introduction.html), [execution overview](https://webassembly.github.io/spec/core/intro/overview.html), and [project goals](https://webassembly.org/docs/high-level-goals/).

## Current artifact

- File: `JokeBase-v1.wasm`
- Size: 10,762 bytes
- SHA-256: `1d238226c11e693980b61527ced25dde91822a56131243f489e51193231491f1`
- Promoted sequence: 18

The content-addressed copy is preserved under `artifacts/promoted/`. `artifacts/lineage.json` links every retained promoted binary to its parent.

## Current capability boundary

JokeBase currently provides three independent tables, each with capacity for 64 rows: nullable `t1(x INTEGER)`, explicit-key `t2(y INTEGER PRIMARY KEY)`, and nullable-unique `t3(z INTEGER UNIQUE)`. Its SQL interface supports:

- table creation;
- integer and `NULL` insertion;
- ordered full-table scans;
- `<`, `=`, and `>` filters;
- `IS NULL` and `IS NOT NULL` filters;
- predicate-qualified integer updates and deletes;
- all-row integer updates;
- scalar signed-integer/`NULL` literal-list `IN` and `NOT IN` expressions;
- state-dependent signed-integer/`NULL` `IN` and `NOT IN` over `t1` or `SELECT * FROM t1`;
- explicit unique insertion and state-dependent membership for `t2`;
- multiple `NULL` rows, unique non-`NULL` signed integers, and state-dependent membership for `t3`;
- `IN` and `NOT IN` over the populated cross-table expression `SELECT x+y FROM t1,t2`, with NULL propagation and signed-overflow-safe comparisons;
- signed decimal-literal `IN` and `NOT IN` against empty lists and integer `t1`, with one to six fractional digits and NULL-aware semantics;
- ordinary ASCII and signed-integer text-literal membership against empty lists and integer `t1`, including SQLite-compatible integer affinity;
- valid hexadecimal blob-literal membership against empty lists and integer `t1`, with SQLite-compatible type and NULL semantics;
- text-literal `IN` and `NOT IN` over text/`NULL` literal lists, with byte-exact comparison and three-valued semantics;
- deterministic host-storable three-table snapshots with atomic validation and restoration.

The supported embedding interface is versioned in [`ABI.md`](ABI.md). SQL bytes must be staged wholly inside memory addresses `[1024,4096)`, with a maximum length of 3,072 bytes. Sequence 18 rejects every other `execute(ptr,len)` range with `-10` before reading it. Because the linear memory is exported, direct host writes outside that window remain outside the contract and can corrupt module state before JokeBase receives control.

This is not a general SQL implementation. Tables beyond `t1`, `t2`, and `t3`, multiple columns, arbitrary identifiers, stored floating-point/text/blob values, general floating-point/text/cross-table expressions, joins, grouping, ordering, indexes, transactions, internal filesystem I/O, and concurrency remain unsupported. Decimal exponent notation and more than six fractional digits are unsupported. Text literals with embedded quotes are unsupported; text membership against integer `t1` recognizes numeric affinity only for optional-minus integers. `t2` does not yet support NULL rowid allocation, projection, update, or delete; `t3` does not yet support projection, update, or delete. SQL keyword casing is not yet consistently general. The exact contract is recorded in `JokeBase-v1-evidence.json`.

## External evidence

Sequence 18 retains the first 58 queries in the pinned SQLite SQLLogicTest `in1.test` file contiguously, including literal-list membership, independent `t1`/`t2`/`t3` table-backed membership, cross-table `x+y` membership, decimal membership, text membership with integer affinity, hexadecimal blob membership, and text-literal lists. The next unsupported statement creates `t4`, a fourth table.

It also passed:

- 30,000 generated three-table membership comparisons against SQLite 3.53.3;
- 50,000 generated cross-table-sum membership comparisons against SQLite 3.53.3, including 3,031 overflowing row pairs;
- 50,000 generated decimal-membership comparisons against SQLite 3.53.3;
- 50,000 generated text-membership comparisons against SQLite 3.53.3;
- 50,000 generated blob-membership comparisons against SQLite 3.53.3, covering 802,334 payload bytes and 1,542 empty blobs;
- 50,000 generated text-list comparisons against SQLite 3.53.3, covering 200,473 list elements and 25,332 `NULL` elements;
- 20,000 malformed or randomly mutated text-list inputs with zero traps and successful post-campaign recovery;
- 52,500 stateful nullable-database regression operations;
- 220,589 independent model checks;
- 52,500 four-way NULL partitions;
- 250 fresh-instance three-table snapshot round trips;
- 750 corrupt-image atomic rejections;
- capacity, uniqueness, malformed-input, and non-trapping invalid-range checks.
- supported-window edge checks plus nine rejected SQL input ranges, with result clearing, no traps, and no state mutation by `execute`.

These results support only the declared capability profile. They do not imply conformance with the complete SQLLogicTest corpus, SQLite, SQLancer, or SQL generally.

### Current reproducibility boundary

The repository independently preserves and makes reproducible the promoted bytes, their SHA-256 lineage, the pinned upstream test inputs, the declarative ABI cases, and the exact capability boundary. The large generated differential and stateful campaign counts are currently recorded evidence, but their executable generators and runners are not yet published. An external reviewer can verify artifact identity and run the declared examples, but cannot yet reproduce every generated counter from repository contents alone. Closing that gap is a promotion-method priority.

## Developing an opaque database as a feedback-control problem

Without source code, implementation inspection cannot close the development loop. JokeBase instead borrows a practical model from control theory: treat the binary as an unknown, partially observed system; apply a bounded intervention; measure the response; reject unsafe states; and re-plan from the new evidence.

This is an engineering analogy, not a claim that software evolution is a conventional linear plant. The mapping is nevertheless useful:

| Control concept | JokeBase equivalent |
| --- | --- |
| Plant | The candidate `.wasm` binary and its runtime state |
| Control input | A transactional mutation of raw binary bytes |
| Sensors | Wasm validation, SQL results, errors, snapshots, hashes, and test oracles |
| Reference signal | The declared database contract and next pinned-suite behavior |
| Observer | Binary metadata plus hypotheses inferred from permitted input/output experiments |
| Residual | The structured difference between expected and observed behavior |
| Safety supervisor | Deterministic promotion gates and immutable regression requirements |
| Safe fallback | The previous content-addressed promoted binary |

### Observability before confidence

Passing examples do not necessarily make the relevant behavior observable. Different internal mistakes can produce identical outputs when the chosen database state does not exercise them.

Sequence 13 produced a concrete example. The first candidate for `SELECT ... IN (SELECT x+y FROM t1,t2)` used the wrong memory addresses for `t1`. It still passed all four newly targeted SQLLogicTest queries because both tables were empty: the faulty loop never read either address. Populating the tables immediately exposed the defect.

The lesson is the control-theory idea of informative excitation. Every new capability must be driven through states capable of distinguishing plausible failures. Depending on the feature, that includes:

- absent, empty, populated, and full schemas;
- matches, misses, and duplicate values;
- `NULL` and non-`NULL` inputs;
- minimum, maximum, ordinary, and overflowing arithmetic values;
- success, malformed input, and failure atomicity;
- state before and after snapshot restoration;
- every relevant combination of interacting tables.

For sequence 13, the four empty upstream examples were therefore only the first sensor readings. Promotion also required 50,000 populated comparisons against SQLite, including 3,031 row pairs whose mathematical sum exceeded the signed-32-bit range. Those experiments found both the incorrect memory map and the risk of a false match caused by wrapping Wasm arithmetic.

### Receding-horizon mutation

Changes are kept small and feedback is collected after each one:

1. choose one coherent semantic boundary from a pinned suite;
2. form explicit behavioral hypotheses and distinguishing experiments;
3. apply one recoverable raw-byte intervention to a disposable candidate;
4. validate the Wasm container and instruction-stack types immediately;
5. run focused probes, then differential, property, stateful, robustness, and persistence tests;
6. update the behavioral evidence and re-plan from what was actually observed;
7. promote only when all hard residuals are zero.

This resembles receding-horizon control: plan ahead, apply only the next bounded action, observe the real response, and plan again. It prevents unmeasured binary edits from accumulating faster than failures can be isolated.

### A safety supervisor, not a single score

New capability cannot compensate for corruption of an existing invariant. JokeBase therefore does not optimize one blended "quality" number. Promotion uses a lexicographic safety barrier: the candidate must validate, preserve its parent and ABI, remain bounded and non-trapping, pass all declared regressions, preserve statement and snapshot atomicity, reproduce an exact hash, and state unsupported behavior honestly. Only then does a longer test-suite prefix count as progress.

Failed experiments remain disposable. Every accepted state has an immutable content-addressed predecessor, so the previous promoted artifact acts as a known-safe fallback. The public repository then provides an independently retrievable copy whose bytes are verified after publication.

### An observer that must not become a decompiler

Control requires useful state estimates, but the experiment forbids reconstructing implementation source. JokeBase may record binary metadata such as section sizes, opaque function indices and hashes, globals, memory regions, snapshot layouts, test coverage identifiers, and observed mutation-to-behavior relationships. It may not produce WAT, pseudocode, an intermediate representation, decompiled logic, or any other source-like implementation.

The objective is not to identify the hidden implementation uniquely. It is to collect enough informative behavioral evidence to control its evolution and justify each bounded claim. This is close in spirit to data-informativity approaches, where data can be sufficient for a control property even when they do not uniquely identify the entire system.

The control perspective is informed by Åström and Murray's open [Feedback Systems](https://www.cds.caltech.edu/~murray/amwiki/Version_2.10c.html), the paper [Data informativity: a new perspective on data-driven analysis and control](https://research.rug.nl/en/publications/data-informativity-a-new-perspective-on-data-driven-analysis-and-/), and research on [safe exploration under uncertain dynamics](https://proceedings.mlr.press/v120/liu20a.html) and [safe learning with model-predictive control](https://proceedings.mlr.press/v242/buerger24a.html).

## Repository policy

Implementation artifacts in this repository are binaries. Human-readable files describe capabilities, provenance, tests, hashes, and observed behavior; they are not an implementation source representation.

Every promotion must:

1. validate as WebAssembly;
2. preserve its immutable parent artifact;
3. pass declared regression and behavioral campaigns;
4. receive a content-addressed archive name and evidence file;
5. state unsupported behavior honestly.

The repository intentionally contains no build-from-source instructions because no implementation source exists.
