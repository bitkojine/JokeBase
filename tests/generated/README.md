# Materialized generated test data

This directory preserves generated test cases as immutable data rather than implementation or generator source.

`text-list-seq17-sqlite-3.53.3.jsonl.gz` contains 50,000 UTF-8 JSON Lines records. Each record includes its stable identity, complete SQL input, and expected SQLite scalar result. The adjacent manifest pins compressed and uncompressed sizes, SHA-256 hashes, field meanings, generation identity, aggregate coverage, and independent replay results.

After standard gzip decompression, consumers read one JSON object per line in ascending `id` order. To replay a record against JokeBase, stage the UTF-8 bytes of `sql` inside the input window specified by `ABI.md`, call `execute`, require one materialized result, and compare SQL `NULL` through `result_is_null` or the integer payload through `result_get` with `expected`.

The corpus intentionally contains no JokeBase implementation representation and no generated expected answers from JokeBase. Expected values were produced by SQLite 3.53.3 and then replayed independently against the promoted binary.

`text-catalog-seq19-sqlite-3.51.0.jsonl.gz` adds 10,000 stateful TEXT-table differential records for promoted sequence 19. Each record carries the complete ordered `setup` array for its declared database state, one membership query, and the independently produced SQLite expected result. The records are grouped by `database`; a replay creates a fresh JokeBase instance, executes that group's setup once in order, and then checks its 100 records. Its adjacent manifest pins the corpus identity, reference version, coverage, and successful sequence-19 replay.
