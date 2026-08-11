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
- Size: 8,070 bytes
- SHA-256: `9574030eecd603af64986df52b246003d7294b1c59c7a60daeafbb4e5789658b`
- Promoted sequence: 13

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
- deterministic host-storable three-table snapshots with atomic validation and restoration.

This is not a general SQL implementation. Tables beyond `t1`, `t2`, and `t3`, multiple columns, arbitrary identifiers, floating-point/text/blob values, general cross-table expressions, joins, grouping, ordering, indexes, transactions, internal filesystem I/O, and concurrency remain unsupported. `t2` does not yet support NULL rowid allocation, projection, update, or delete; `t3` does not yet support projection, update, or delete. SQL keyword casing is not yet consistently general. The exact contract is recorded in `JokeBase-v1-evidence.json`.

## External evidence

Sequence 13 passes the first 36 queries in the pinned SQLite SQLLogicTest `in1.test` file contiguously, including literal-list membership, independent `t1`/`t2`/`t3` table-backed membership, and cross-table `x+y` membership. The next unsupported query uses a floating-point literal.

It also passed:

- 30,000 generated three-table membership comparisons against SQLite 3.53.3;
- 50,000 generated cross-table-sum membership comparisons against SQLite 3.53.3, including 3,031 overflowing row pairs;
- 52,500 stateful nullable-database regression operations;
- 220,589 independent model checks;
- 52,500 four-way NULL partitions;
- 250 fresh-instance three-table snapshot round trips;
- 750 corrupt-image atomic rejections;
- capacity, uniqueness, malformed-input, and non-trapping invalid-range checks.

These results support only the declared capability profile. They do not imply conformance with the complete SQLLogicTest corpus, SQLite, SQLancer, or SQL generally.

## Repository policy

Implementation artifacts in this repository are binaries. Human-readable files describe capabilities, provenance, tests, hashes, and observed behavior; they are not an implementation source representation.

Every promotion must:

1. validate as WebAssembly;
2. preserve its immutable parent artifact;
3. pass declared regression and behavioral campaigns;
4. receive a content-addressed archive name and evidence file;
5. state unsupported behavior honestly.

The repository intentionally contains no build-from-source instructions because no implementation source exists.
