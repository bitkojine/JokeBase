# JokeBase embedding ABI

This document describes the supported host contract for promoted JokeBase sequence 19. It is an interface specification, not an implementation representation.

## Instantiation

`JokeBase-v1.wasm` is a core WebAssembly module with no imports. It exports one linear memory and the functions listed below. The current module starts with one 64 KiB page. Internal database storage is fixed-capacity, but a host can grow the exported memory; growth does not increase JokeBase table capacity.

## SQL input window

The only supported way to call `execute(ptr, len)` is to place the exact UTF-8 SQL bytes inside this half-open memory interval:

- first supported address: 1024;
- first unsupported address after the window: 4096;
- maximum input length: 3072 bytes;
- required relation: `1024 <= ptr` and `ptr + len <= 4096`.

`execute` returns `-10` and clears the materialized result count when the range is outside this window. It does not read the rejected range.

Linear memory is exported so the host can stage SQL and copy snapshots. Direct host writes outside the input window are outside the ABI contract. Such writes can corrupt database state before JokeBase receives control; WebAssembly cannot protect one region of an exported linear memory from its host.

The input bytes need not be NUL-terminated. JokeBase uses the explicit length.

## SQL execution and results

`execute(ptr, len) -> i32` returns:

- `0` for a successful statement with no affected or returned rows;
- a positive row/change count for a successful statement;
- a documented negative error code on failure.

Every call clears the previous materialized result before parsing. A successful row-producing query exposes its result through:

- `result_count() -> i32`;
- `result_get(index) -> i32`;
- `result_is_null(index) -> i32`, returning `1` for SQL `NULL`, `0` for a non-NULL integer, and `-4` for an invalid index.

The current error codes are:

| Code | Meaning |
| --- | --- |
| `-1` | table already exists |
| `-2` | referenced table does not exist |
| `-3` | fixed row capacity reached |
| `-4` | row or result index is out of bounds |
| `-5` | duplicate non-NULL primary-key or unique value |
| `-10` | unsupported/malformed SQL, unsupported input range, or invalid snapshot input |

## Host diagnostic contract

The binary return code is a compact, stable machine-facing status. It is not a complete end-user diagnostic. Every host, command-line tool, test report, and web interface built for JokeBase **must translate a negative code into a useful diagnostic**. Do not render a bare value such as `Error -2` as the final user-visible outcome.

The required shape is:

```text
error[JB-0002]: table `t1` does not exist in this database instance
  note: this is a fresh, empty instance
  help: create it first with `CREATE TABLE t1(x INTEGER)`
```

The `JB-` identifier is a stable host diagnostic identifier; it maps one-to-one to the ABI return code today (`JB-0002` maps to `-2`). It does not change the WebAssembly ABI or assert that the binary itself contains text diagnostics.

Each displayed error must include all applicable parts:

1. a short, plain-English primary message naming the failed operation and relevant object/value;
2. a stable `JB-` diagnostic identifier and the raw ABI code for programmatic support and evidence;
3. a `note` with relevant observed context (for example, that a new instance was selected, the fixed row limit is 64, or the input was outside the permitted staging window);
4. a concrete `help` action when one is known; and
5. the exact submitted SQL, with a best-effort byte offset or highlighted span where the host can determine one without guessing.

Suggested baseline messages:

| ABI code | Host diagnostic | Primary message | Useful help |
| --- | --- | --- | --- |
| `-1` | `JB-0001` | table already exists | use a new instance, or select another declared table identity |
| `-2` | `JB-0002` | referenced table does not exist in this database instance | create the declared table first; after a new instance, all tables are absent |
| `-3` | `JB-0003` | table has reached its fixed 64-row capacity | delete a row, reset/start a new instance, or use a different declared table |
| `-4` | `JB-0004` | requested row or result index is outside the available range | inspect `result_count()` before requesting an index |
| `-5` | `JB-0005` | duplicate non-NULL value violates this table's unique or primary-key constraint | use a different non-NULL value; SQL `NULL` is not a duplicate for the supported unique tables |
| `-10` | `JB-0010` | statement or host input is unsupported or malformed | use a documented statement form; check exact spacing/casing and the `[1024,4096)` input window |

This policy is deliberately modeled after Rust compiler diagnostics: use simple, concise primary errors; separate explanatory context into `note`; put corrective action in `help`; give recurring errors stable identifiers with extended documentation; and test diagnostic output as a public interface. Rust's official compiler guide describes that structure and style, including structured help/suggestions and exhaustive UI-test expectations. See the [Rust diagnostic guide](https://rustc-dev-guide.rust-lang.org/diagnostics.html), [error-code guidance](https://rustc-dev-guide.rust-lang.org/diagnostics/error-codes.html), and [UI-test guidance](https://rustc-dev-guide.rust-lang.org/tests/ui.html).

### Diagnostic tests

For every supported host integration, test at least one contextual example for each negative code it can emit. Assertions must cover the diagnostic identifier, primary message, contextual note, and help text—not only the raw return value. Treat new bare numeric errors in interactive surfaces as a regression. The source-free rule remains intact: these tests observe only public ABI inputs and outputs.

## Lifecycle and legacy row API

`jokebase_version() -> i32` returns the ABI major version, currently `1`.

`db_reset()` clears all schemas, rows, results, and mutable database state. The exports `db_create`, `db_insert`, `db_count`, `db_get`, and `db_delete` are retained for compatibility with the original one-table integer ABI. New integrations should use SQL through `execute`.

## Host-managed snapshots

JokeBase snapshots are deterministic binary images managed by the host:

- `db_snapshot_ptr() -> i32` returns the module-owned snapshot buffer address;
- `db_snapshot_size() -> i32` returns the exact image length for current state;
- `db_snapshot_write() -> i32` writes the image and returns its length;
- `db_snapshot_read(len) -> i32` atomically validates and restores an image from the module-owned snapshot buffer.

The host copies snapshot bytes out after `db_snapshot_write`, and copies them back to the address returned by `db_snapshot_ptr()` before calling `db_snapshot_read(len)`. The read function does not accept a caller-selected pointer. A rejected image leaves JokeBase state unchanged.

The earlier public revision of this document incorrectly described `db_snapshot_read` as a two-parameter function. The promoted binaries have always exported the one-parameter function documented above; this was a documentation defect, not a binary ABI change. The behavioral correction evidence for sequence 18 is recorded in `tests/snapshot-abi-contract-v1.json`.

Sequence 19 expands the snapshot image to include the bounded integer and TEXT catalogs and advances the snapshot format identity. Snapshots written by sequence 18 and earlier are not accepted by sequence 19. A rejected earlier-format image returns `-10` and leaves the complete sequence-19 state unchanged. Hosts must treat snapshot bytes as version-bound opaque images and retain the exact JokeBase artifact needed to restore an older image.

This is host-managed snapshot persistence, not filesystem durability. JokeBase does not provide filesystem access, `fsync`, write-ahead logging, crash-safe host storage, or concurrent transactions.

## Security boundary

JokeBase has no imports and therefore no ambient filesystem, network, device, or operating-system capabilities. Its WebAssembly sandbox protects the host from ambient module access. It does not protect JokeBase from a host that directly modifies exported memory.
