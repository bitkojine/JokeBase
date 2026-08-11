# JokeBase

JokeBase is an experimental SQL database distributed only as WebAssembly machine code.

Its defining constraint is absolute: neither humans nor AI agents may read or write implementation source code, WebAssembly text, compiler inputs, generator programs, or decompiled source. Development operates only on raw `.wasm` bytes, binary metadata, runtime behavior, and external tests.

## Current artifact

- File: `JokeBase-v1.wasm`
- Size: 4,955 bytes
- SHA-256: `f33be846e8ff2b13667703817e6dad19614cdea9cd090687f80b0b530566de9b`
- Promoted sequence: 10

The content-addressed copy is preserved under `artifacts/promoted/`. `artifacts/lineage.json` links every retained promoted binary to its parent.

## Current capability boundary

JokeBase currently provides one fixed table, `t1(x INTEGER)`, with capacity for 64 nullable signed 32-bit values. Its SQL interface supports:

- table creation;
- integer and `NULL` insertion;
- ordered full-table scans;
- `<`, `=`, and `>` filters;
- `IS NULL` and `IS NOT NULL` filters;
- predicate-qualified integer updates and deletes;
- all-row integer updates;
- scalar signed-integer/`NULL` literal-list `IN` and `NOT IN` expressions;
- state-dependent signed-integer/`NULL` `IN` and `NOT IN` over `t1` or `SELECT * FROM t1`;
- deterministic host-storable snapshots with atomic validation and restoration.

This is not a general SQL implementation. Multiple tables and columns, arbitrary identifiers, strings, joins, grouping, ordering, indexes, transactions, internal filesystem I/O, and concurrency remain unsupported. The exact contract and unsupported behavior are recorded in `JokeBase-v1-evidence.json`.

## External evidence

Sequence 10 passes the first 16 queries in the pinned SQLite SQLLogicTest `in1.test` file contiguously, including both literal-list and table-backed membership. The next unsupported statement creates a second table.

It also passed:

- 20,000 generated scalar-expression comparisons against SQLite 3.51.0;
- 20,000 generated table-membership comparisons against SQLite 3.51.0;
- 70,200 stateful nullable-database regression operations;
- 70,000 independent model checks;
- 7,521 fresh-instance snapshot round trips;
- earlier NULL partition, differential, malformed-input, capacity, and corrupt-image campaigns preserved in the evidence history.

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
