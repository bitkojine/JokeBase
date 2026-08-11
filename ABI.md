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
