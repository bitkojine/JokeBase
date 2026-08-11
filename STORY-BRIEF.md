# The JokeBase story brief

This is a non-technical communication guide for JokeBase. It is a description of externally observed behavior and evidence, not an implementation representation.

## The one-sentence version

JokeBase is a small experimental database built as raw WebAssembly bytecode, with a deliberately extreme rule: neither people nor AI agents are allowed to read or write its implementation source code.

## The human story

Most software is built by changing source code, compiling it, and reading the source again when something breaks.

JokeBase asks what happens when that safety net is removed.

There is no Rust, C, JavaScript, SQL-engine source, WebAssembly text, compiler input, or generated implementation file behind the current artifact. The durable implementation is an 18,438-byte WebAssembly binary. WebAssembly is portable bytecode, not the host CPU's native machine code; a runtime later translates it for the actual machine.

That makes the project partly absurd on purpose. A database nobody can inspect or normally maintain is a joke.

But the experiment is serious: can an opaque software artifact still be evolved responsibly if every change is small, every claim is externally tested, failures are retained, and every accepted version is content-addressed and published?

## What exists today

The current public release is JokeBase sequence 19.

- It is a real, bounded, stateful SQL database rather than a collection of canned answers.
- It can create the twelve declared one-column table identities, insert integer or bounded TEXT values, enforce the declared unique/primary-key rules, perform selected updates and deletes on its original integer table, run selected `IN` and `NOT IN` queries, and copy data through its declared `INSERT ... SELECT` forms.
- It handles SQL's awkward three-valued logic: an answer can be true, false, or unknown because of `NULL`.
- It can create deterministic snapshots, reject malformed or impossible snapshots before changing live state, and restore valid snapshots atomically.
- Its TEXT tables hold up to 64 rows each, with values up to 32 unescaped UTF-8 bytes. Those limits are intentional and explicit.

It is not SQLite, and it is not a general SQL database. It does not support arbitrary table definitions, general joins, grouping, ordering, transactions, concurrency, filesystem durability, or most of SQL. That honesty is part of the project.

## Why anyone should believe it does anything

We do not ask people to trust a claim that the binary "looks right." There is no readable implementation to inspect.

Instead, trust is built from observable evidence:

- The exact release binary has a public SHA-256 identity and a preserved parent history.
- It executes all 27 setup statements and all 187 SQLite-enabled queries in one pinned SQLite SQLLogicTest file with zero observed failures.
- 10,000 generated membership queries matched SQLite across 100 generated TEXT-table states.
- 40,000 generated model-based state operations completed with zero observed failures or runtime traps.
- A materialized 50,000-query SQLite-derived regression corpus replayed with zero failures.
- Snapshot testing rejected 12,421 wrong lengths, 523 deliberately corrupted TEXT snapshots, and five damaged headers without changing the database state.
- The public GitHub-served binary was downloaded after publication and its hash matched the promoted release.

Those results do not prove the database is universally correct. They make a narrower, testable claim credible: the released artifact behaved correctly over the declared contract and evidence set.

## The real difficulty

The hard part is not simply writing bytes.

One incorrect byte can make WebAssembly reject the whole database before it runs. More dangerously, a binary can be structurally valid yet have the wrong database behavior. A new query handler can accidentally intercept an older query family. A snapshot reader can accept bytes that look well-formed but represent an impossible database state. A single encoded boundary value once made a supposed 64-row limit behave as negative 64.

In ordinary development, source code and a debugger help isolate these problems. Here the test suite must do that job.

## How the project became manageable

JokeBase uses a practical control-theory mindset.

Think of the binary as a machine whose internals cannot be inspected. A proposed byte-level change is an input to the machine. Tests, hashes, snapshots, errors, and runtime validation are sensors. The declared behavior is the target. A failed test is a measured difference between target and reality.

That leads to a disciplined loop:

1. Predict a narrow behavior change before touching the artifact.
2. Make the smallest reversible intervention.
3. Validate that the binary still runs.
4. Test the new behavior and nearby old behavior.
5. Run broader differential, stateful, corruption, and persistence checks.
6. Record the result forever, including failed hypotheses.
7. Promote only an artifact that clears all release gates.

The project records this in an append-only flight log. Old records are not rewritten when later evidence changes the story; corrections are new entries. That makes failures part of the value rather than embarrassing debris.

## The best opening for a talk or post

> What if you tried to build a database but banned everyone—including the AI—from ever reading or writing its source code?
>
> You would not get normal software engineering. You would get a strange experiment in control, testing, and trust.
>
> That is JokeBase.

## Language that is accurate

Prefer:

- "raw WebAssembly bytecode"
- "source-free implementation under a documented rule"
- "a small, bounded, stateful SQL database"
- "passes one complete pinned SQLLogicTest file"
- "evidence-backed behavior within an explicit boundary"
- "WebAssembly is portable bytecode that a runtime translates to native code"
- "the tests are the debugger and part of the specification"

Avoid:

- "native machine code" when referring to the `.wasm` file
- "it passes SQLLogicTest" without naming the one pinned file
- "AI built a database with no code"—the artifact is code in binary form; the unusual claim is that no source representation was used or inspected
- "proves the database is correct"
- "production-ready" or claims of SQLite compatibility beyond the measured cases

## Questions people will reasonably ask

### Is this really a database?

Yes, in the limited but real sense that it accepts SQL inputs, maintains mutable table state, enforces declared constraints, produces query results, and persists/restores state through snapshots. It is deliberately not a general-purpose database.

### Is this really code with no source code?

The executable binary is code. The experiment's rule is narrower and more precise: no human or AI agent reads or writes a source-level implementation, WebAssembly text, compiler input, implementation generator, or decompiled representation. The repository records raw binary fragments, behavior, hashes, and tests instead.

### Did an AI just memorize a database?

The meaningful answer is not a claim about internal AI cognition. The project deliberately requires observable behavior: arbitrary bounded values, evolving state, constraints, snapshots, generated inputs, reference-engine comparisons, and stateful histories make a fixed list of expected query answers inadequate for the declared scope.

### Why make it this hard?

Because normal source code hides how much of software trust comes from readable structure, familiar tools, and reversible development. Removing those tools exposes what must replace them: precise contracts, testing, evidence preservation, and honest limits.

### What is the value if nobody should build production software this way?

The useful output is the method and evidence trail. JokeBase is a stress test for agentic software development, opaque artifacts, testing strategy, reproducibility, and the difference between "it seems to work" and "we have reason to trust this limited claim."

## Current public facts

- Repository: https://github.com/bitkojine/JokeBase
- Current promoted sequence: 19
- Public artifact SHA-256: `ca3c9d4d3c0a76767281410e1df5da713f1e5f40e26c5635b3106eb09458580b`
- Artifact size: 18,438 bytes
- Full evidence: `JokeBase-v1-evidence.json`
- Append-only history of experiments: `DEVLOG.md`
- Tooling and methodological lessons: `TOOLS-AND-LEARNINGS.md`

## Narrative arc in five beats

1. **The dare:** Build a database while forbidding source code.
2. **The twist:** The `.wasm` file is bytecode, not native machine code, and there is no hidden source tree.
3. **The consequence:** You lose the ordinary debugger and have to make tests, snapshots, hashes, and tiny reversible changes do the work.
4. **The evidence:** A tiny but real stateful database now passes a complete pinned upstream SQL test file plus independent generated and corruption campaigns.
5. **The question:** How much evidence is enough to rationally trust an opaque artifact within a declared boundary?

The story should end on that question, not a victory lap.
