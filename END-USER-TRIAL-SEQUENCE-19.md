# End-user trial: JokeBase sequence 19

**Artifact tested:** `JokeBase-v1.wasm`, 18,438 bytes, SHA-256 `ca3c9d4d3c0a76767281410e1df5da713f1e5f40e26c5635b3106eb09458580b`.

**Method:** Treat the artifact as a black box. This trial used only its documented host interface: send UTF-8 SQL to `execute`, read results through the exported result functions, and use the documented save/restore interface. No implementation representation or raw binary internals were read.

## Plain verdict

JokeBase feels like a real but very small embedded database experiment, not like a database product a typical developer could pick up and use freely.

It can hold changing state, enforce important rules, answer selected SQL questions, and restore an earlier saved state. Those are real database behaviors.

It is also unusually strict. It accepts only a small declared collection of table shapes and SQL sentence forms. An application developer needs a host program to use it, needs to know the accepted spellings precisely, and gets numeric error codes rather than explanatory messages.

## What a first-time user can successfully do

### Store and change integer rows

This session worked on a fresh instance:

```sql
CREATE TABLE t1(x INTEGER)
INSERT INTO t1 VALUES(10)
INSERT INTO t1 VALUES(-3)
INSERT INTO t1 VALUES(NULL)
SELECT x FROM t1
SELECT x FROM t1 WHERE x > 0
UPDATE t1 SET x = 20 WHERE x = 10
DELETE FROM t1 WHERE x < 0
```

Observed results:

- the table was created;
- the three values, including `NULL`, were stored;
- the first scan returned `10`, `-3`, and `NULL`;
- the filter returned `10`;
- the update changed one row;
- the delete removed one row.

The exact spaces in `UPDATE t1 SET x = 20 WHERE x = 10` mattered. More natural-looking compact versions such as `x=20` were rejected. This is a real usability limitation, not a cosmetic one.

### Enforce uniqueness

These results were observed:

| Action | Result |
| --- | --- |
| Create `t2(y INTEGER PRIMARY KEY)` | accepted |
| Insert `7` twice | second insert returned `-5` |
| Insert `NULL` into `t2` | returned `-10` |
| Create `t3(z INTEGER UNIQUE)` | accepted |
| Insert uppercase `NULL` twice into `t3` | accepted |
| Insert `4` twice into `t3` | second insert returned `-5` |

For membership questions, `SELECT 7 IN t2` returned true. `SELECT 9 IN t3` returned unknown after `t3` contained a `NULL`, matching SQL's three-way truth behavior.

### Store bounded text and copy it

This session worked:

```sql
CREATE TABLE t7(a TEXT UNIQUE)
INSERT INTO t7 VALUES('hello')
INSERT INTO t7 VALUES('world')
INSERT INTO t7 VALUES(null)
CREATE TABLE t8(c TEXT)
INSERT INTO t8 SELECT * FROM t7
SELECT 'hello' IN t7
SELECT 'missing' IN (SELECT * FROM t7)
```

Observed results:

- the copy inserted three rows into `t8`;
- `'hello'` was found;
- a missing value returned unknown because `t7` contains `NULL`;
- a duplicate non-empty text value returned `-5`;
- 64 text rows were accepted, while row 65 returned `-3` and did not change the stored state;
- text with an embedded quote written as `O''Brien` was rejected;
- text longer than the declared bound was rejected.

This makes `t7` useful for bounded presence checks, but not for the everyday task of listing or reading text rows: `SELECT * FROM t7`, `SELECT 'hello' FROM t7`, and `SELECT count(*) FROM t7` were rejected.

### Save and restore

The host saved a snapshot after storing `'before-save'`, inserted `'after-save'`, then restored the saved snapshot.

After restoration:

- `'before-save'` was found;
- `'after-save'` was not found;
- restore returned success.

This is genuine persistence behavior, but it is host-managed rather than a familiar `SAVE` command or file on disk. The application embedding JokeBase must copy snapshot bytes out and back in.

## What a first-time user is likely to try—and cannot do yet

| Natural request | Observed result |
| --- | --- |
| `CREATE TABLE people(id INTEGER, name TEXT)` | rejected with `-10` |
| `CREATE TABLE notes(text TEXT)` | rejected with `-10` |
| `DROP TABLE t7` | rejected with `-10` |
| `CREATE INDEX name_index ON t7(a)` | rejected with `-10` |
| `SELECT * FROM t7` | rejected with `-10` |
| `SELECT count(*) FROM t7` | rejected with `-10` |
| `SELECT * FROM t7 ORDER BY a` | rejected with `-10` |
| Query a table that does not exist | returned `-2` |

`-10` is intentionally a broad “unsupported or malformed request” result. It is safe and stable, but it does not tell a human *why* their request failed.

## Experience assessment

### What feels real

- Data persists across statements.
- Tables have independent state.
- Constraints are enforced.
- Failed capacity and duplicate operations do not partly apply.
- Queries have meaningful `NULL` behavior.
- Save/restore really moves the database back to an earlier state.

### What feels unfinished

- There is no ready-to-use command-line program, web interface, or driver; a programmer must embed the Wasm file.
- SQL is a narrow language with spelling and whitespace sensitivities.
- TEXT can be tested for membership but cannot yet be selected back as rows.
- Table names and schemas are fixed rather than user-chosen.
- Error codes lack human-readable explanations.
- There is no table removal, index creation, sorting, counting, joining, transaction support, or direct disk saving.

## End-user conclusion

JokeBase passes the threshold of “a genuine, bounded database artifact.” It does not pass the threshold of “a useful general database for an ordinary application developer.”

The next improvements that would most change the end-user experience are not more test-count headlines. They are:

1. an extremely small interactive host or driver;
2. readable error messages;
3. flexible table names and simple user-defined schemas;
4. row projection for stored TEXT;
5. less formatting-sensitive SQL;
6. table removal and basic lifecycle operations.

This trial deliberately records both sides. The successful behaviors make the experiment credible; the friction shows what still separates a verified binary experiment from a practical database.
