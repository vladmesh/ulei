# ADR-0051: Memory layers: the archive (C10), the `session_search` index in the C8 journal, "forget" marks, and their cards

Status: accepted
Date: 2026-11-21
Affects: crates/memory, crates/engine, crates/garden, contract:C6, contract:C7, contract:C8, contract:C9, contract:C10, dep:rusqlite, dep:serde_json, dep:rustix, dep:sha2, dep:miniz_oxide

## Context

Three items of the sprint share stores and indexes: the archive (issue:0863875e8c25435f782f,
here d8), the tool `session_search` (issue:658330d9eaf2323d9718, d4) and "forget"
(issue:296e14ac9e104640c96d, d10). Each issue carries the owner's decisions of 2026-11-21
quoted below by their time. Two neighbours only need an interface from this record: the
seedbed, the garden's day bed (issue:af784fb6649fa299fc64, d7), and the source connectors
(issue:0276511448bf4134a435, d9).

The owner's decisions this record implements:

- 11:14 to 11:18: the archive is raw material as it came, immutable, never compressed with loss;
  its index is FTS5 and metadata only (who, when, source), no embeddings, no LLM. The original
  stays in the archive forever; the forest holds the compressed form, made by the gardener's
  night stage (not this record).
- 11:22 to 11:23: the archive is stored locally on the agent's server (compressed raw material,
  FTS5); cold storage only when the disk no longer holds it.
- 12:37: the raw dialogue goes into the archive; context compression during a dialogue stays an
  artifact of the session and goes neither into the archive nor into retrieval.
- 12:39: the archive is not touched without a direct command. "Forget" marks a forest record as
  not current (out of retrieval); the garden clears it itself at the night reconciliation.
- 13:25: the archive and data are not encrypted on the server; only external backups are.
- Search order: garden, then forest, then archive; the archive only when garden and forest find
  nothing; what is found in the archive is lifted into the seedbed.
- 13:29 to 13:30: the old runtime's history and library go into the archive after the sprint.

What exists on main:

- The C8 journal (ADR-0014, `crates/engine/src/journal.rs`, schema version 5) stores each
  message as its canonical C4G serialization in `messages`, keyed `(session, seq)` in a
  `WITHOUT ROWID` table, with `active` and `compacted` flags. It stores no time per message,
  and the rows a compaction inserts (its summary and the head and tail it keeps, ADR-0025) are
  live rows with `compacted` false, which no column tells from the turn's own rows. It has no
  search.
- C6 0.7.0 defines `raw_sessions` ("read raw sessions of the runtime, the forest layer of
  memory") with a time range, a cursor and `RawMessage {session_id, ts, role, content}`; no
  runtime serves it.
- `ulei-memory` (`crates/memory`) is an empty skeleton described as "the forest: the cold archive
  of raw sessions", read through C6 `raw_sessions`. The owner's decisions make the archive (raw,
  immutable) and the forest (compressed) two layers.
- The garden (ADR-0043 to ADR-0047) is a daemon on SQLite with FTS5 serving C7 on a Unix socket;
  its facts carry `source_ref`. The engine has no garden tool yet (ADR-0047 section 9).
- Bundled SQLite 3.54.1 (`libsqlite3-sys` 0.40.0 in `Cargo.lock`, its `build.rs` passes
  `-DSQLITE_ENABLE_FTS5`) keeps FTS5, which `.cargo/config.toml` does not undefine (ADR-0044
  item 8); `.cargo/config.toml` passes `-DSQLITE_OMIT_JSON`, so SQL has no JSON functions.
- The reference of `session_search` (port map section 3, row R9, and its tool list):
  `tools/session_search_tool.py`, `hermes_state.py`, `hermes_state_common.py` and
  `hermes_state_search.py` (`THIRD_PARTY_NOTICES.md#ref-agent`). Its FTS5 index lives in the
  session database next to the messages (`FTS_SQL`), written by `AFTER INSERT` triggers in the
  write's transaction, so a committed message is searchable at once and nothing is rebuilt per
  turn. Search covers live rows and rows a compaction archived (`active = 1 OR compacted = 1`)
  and hides rewound ones. The tool has four shapes chosen by its arguments (discovery, scroll,
  read, browse), clamps `limit` to 1..10 (default 3) and `window` to 1..20 (default 5), and
  bounds each returned message to 4,000 characters (1,200 in bookends). The local patch of row
  R9 makes a failed index maintenance pass after the commit not fail the committed turn.

## Decision

### 1. Layers, owners and crates

| Layer | Holds | Store | Crate and process |
|---|---|---|---|
| C8 journal | The engine's sessions as they run: live and compacted messages, steps, deliveries; the history index of section 4 and the forget marks of section 5 | `engine.journal` | `ulei-engine` |
| Archive | Raw items of every source as they came, immutable, lossless compressed, with metadata and FTS5 | `memory.archive`, new | `ulei-memory`, daemon `ulei-memory serve`, contract C10 |
| Forest | The compressed records the night stage makes from the archive | none in this sprint | `ulei-memory`, when built |
| Garden | Typed facts (ADR-0043) | `garden.store` | `ulei-garden` |
| Seedbed | The day's unprocessed records of the garden | decided by d7 | `ulei-garden` |

The archive lives in `ulei-memory`. Its description becomes "Memory below the garden: the
archive of raw items (immutable, compressed, FTS5), served as C10 by `ulei-memory serve`, and
later the forest made from it"; its `[package.metadata.ulei]` contracts become C10 (served) and
C6 (read through `raw_sessions`, section 3); `crates/memory/AGENTS.md`, `src/lib.rs`, the module
map and the README row are rewritten by card 6. Traces to d8 scope 1 and 2 and the decisions of
11:14 to 11:23.

Reason: the forest is made from the archive, so the crate that holds the originals is where the
forest will read them, and one daemon serves both layers within one budget row. The crate is
empty, so nothing moves.

Rejected:
- A new crate `ulei-archive` beside an empty `ulei-memory`: a second module crate and a second
  daemon budget row for the same owner's data, while the forest crate still has no code.
- The archive inside the `ulei-garden` daemon: the garden holds typed facts under its 4 MiB row
  and its own contract; raw items of every source, their blobs and their deletion audit are
  another store with another write pattern.
- The C8 journal as the archive: the journal belongs to one runtime, changes its flags at every
  compaction, and has no way in for the connectors of d9; the archive is immutable and holds
  every source.

### 2. The archive

Item. An item is the original bytes of one record of one source and its metadata:

| Field | Meaning |
|---|---|
| `id` | Lowercase hex SHA-256 of the original bytes; the dedup key. |
| `source` | The putter: `dialogue` for the C8 dialogue (section 3), a connector's manifest name for d9; `^[a-z][a-z0-9_-]{0,63}$`. |
| `external_id` | The source's id of the record, at most 512 bytes. |
| `who` | Author as the source names it (the C4G role for the dialogue); optional. |
| `at` | Unix time of the original record; optional, absent when the source has none. |
| `kind` | Media type of the original bytes (`application/vnd.ulei.c4g-message+json` for the dialogue). |
| `text` | The extracted text for FTS5, given by the putter; the archive parses no format. |
| `put_at` | The archive's time of the first put. |

Storage: one SQLite file `memory.archive`, created 0600 in a 0700 directory like the journal,
WAL, opened only by the daemon. Table `blobs` (`id`, `len`, `codec` `deflate` or `none`,
`body`), table `entries` (`source`, `external_id`, `id`, `who`, `at`, `kind`, `put_at`, unique
on `(source, external_id)`), the FTS5 table `item_fts` (column `text`, tokenizer `unicode61
remove_diacritics 2`, rowid the blob's row) holding its text so `snippet()` works, table
`cursors` (section 3) and table `deletions` (the audit). The body is the raw deflate stream of
the original at level 6 through `miniz_oxide`, or the original itself (`none`) when deflate does
not make it shorter; a read inflates and checks the SHA-256 before it answers, so the bytes read
are the bytes put. No encryption on the server (13:25).

Dedup and idempotence. A put of bytes whose `id` is stored adds no blob and no FTS row. A put of
a `(source, external_id)` that is stored with the same `id` is a no-op answered `duplicate`; with
another `id` it is refused `409 conflict`, because an archived record never changes its bytes (a
changed record is a new `external_id`, the source's business). A new `(source, external_id)` with
a stored `id` adds an entry pointing to the existing blob. The blob, its FTS row and its entries
are written in one transaction.

Operations, C10 0.1.0 `x-methods`, each `POST /v1/<name>` with JSON on the daemon's Unix socket:

| Method | Input | Output |
|---|---|---|
| `put` | an item's fields with the original as `bytes_b64`; body at most 1 MiB | `id`, `status` `stored` or `duplicate` |
| `search` | `q` (FTS5 query), optional `source`, `who`, `kind`, `since`, `until`, `limit` (default 10, at most 50) | hits: `id`, the matching entries' metadata, `snippet()` of `text`; `synced` (section 3) |
| `read` | `id` | `bytes_b64`, `kind`, `len`, entries |
| `delete` | `id`, `reason` | the audit row |
| `sync` | none | the count of dialogue items put (section 3) |
| `health` | none | `version`, item count, last sync time and error; the ready probe of the budgets (section 7) |

`delete` is the only method that removes anything, and it is a command, not a tool: the engine
exposes no tool that calls it, and it is run as `ulei-memory delete <id> --reason <text>`, the
owner's direct command of 12:39. `delete` removes the blob, its FTS row and its entries and, in the same
transaction, writes a `deletions` row (time, `id`, `len`, the entries' metadata, the reason, the
caller's pid and uid from the socket), which is never removed, and one stderr line. No other
method updates or deletes a row. Traces to d8 scope 3 and the decision of 12:39.

Host and transport. The daemon `ulei-memory serve [--config PATH]` reads `ulei.toml` (keys
`memory.archive`, `memory.socket`, and `engine.c6_socket` for section 3), binds its socket 0600
with the peer uid check through `rustix`, one request per connection, every read and write with a
5 s deadline, errors as C10 `Error` (`invalid_request`, `forbidden`, `unknown_method`,
`not_found`, `conflict`, `too_large`, `store_failed`), as C7 does (ADR-0047 section 2). C10 is a
new contract in `contracts/`, not an extension of C7, because the archive is another store and
another process. Callers: the engine's tools `archive_search` and `archive_read` (section 6), the
d9 runner (it calls `put` for every item a connector emits, `source` its manifest name), and the
owner's `delete`.

Rejected:
- The archive as a library linked into its callers: the engine, the d9 runner in the core and the
  owner's command would each open the file; ADR-0001 section 3 forbids a module crate depending on
  another.
- An MCP server for the archive: the engine connects MCP servers per turn (ADR-0026), and the
  connectors' runner is no MCP client.
- Blobs as content-addressed files beside an index: a put would not be one transaction across the
  file and its rows, and every dialogue message would be a file.
- A contentless FTS5 table: it returns NULL for every column but the rowid (sqlite.org/fts5.html
  section 4.4.1), so `snippet()` has no text to cut.
- Extraction of text by the archive per media type: the source knows its format; the archive
  would carry a parser per connector.

### 3. The raw C8 dialogue into the archive

The journal, schema version 6 (C8 next minor version), gives every message:
- `n`, its position in insertion order over the whole journal, from 1, unique; the cursor of
  `raw_sessions`, the rowid of the history index (section 4) and the message id of
  `session_search`;
- `at`, the Unix time the engine wrote it; absent on rows written before version 6;
- `origin`, `turn` for a row a turn appended (the user prompt, the assistant's messages, tool
  results) and `compaction` for a row of a compacted set (`compact` and the in-turn compaction of
  `append`).

The migration from 5 numbers the existing rows in `(session, seq)` order and sets `origin` to
`compaction` for a row whose bytes equal those of an archived row with a lower `seq` in its
session (a head or tail the compressor kept) or that the compressor's summary recognizer
(ADR-0025) accepts, and to `turn` otherwise. A message the owner repeated word for word is caught
by the first rule too; its text stays indexed and archived under the earlier row.

Path: the archive pulls. The engine serves C6 `raw_sessions` (C6 next minor version) from the
journal: the rows with `origin` `turn`, live and archived, after the cursor in `n` order, at most
`limit`; `RawMessage` gains `seq`, `session_key` and `message`, the canonical C4G serialization,
and `content` is the text the engine extracts for its own index (section 4), so both indexes see
the same text; `since` becomes optional and `ts` is absent for rows without `at`. The archive puts
each row as an item: `source` `dialogue`, `external_id` `<session_key>/<session>/<seq>`, `who` the
role, `at` from `ts`, `kind` `application/vnd.ulei.c4g-message+json`, the bytes of `message`,
`text` from `content`. It stores the page's cursor in `cursors` in the transaction of the page's
items, so a crash repeats at most one page, which dedup makes a no-op.

Moment: at the daemon's start, before every `search`, and on the method `sync`, which the d9
runner's schedule calls like any source. When the engine does not answer, `search` answers from
what is stored with `synced: false` and `health` names the error. Nothing in the archive wakes on
a timer.

Excluded: the rows with `origin` `compaction` (decision 12:37), the steps (request ids, routes,
usage: run metadata), the system prompt (built from files, not said), the turn's ephemeral
context (never journaled) and the delivery records. Forget marks (section 5) do not filter
`raw_sessions`: the archive takes every raw row.

Traces to d8 scope 4 and the decision of 12:37.

Rejected:
- The engine pushing each turn to C10 `put` after its append: the turn path would hold an archive
  client and an outbox for failures, and the archive would depend on one runtime where C6
  `raw_sessions` already is the runtime's read for this layer.
- The archive reading `engine.journal` itself: another module's database is not a wire
  (`crates/memory/AGENTS.md`, ADR-0001 section 3).
- A timer in the archive daemon: wake-ups when idle (ADR-0001).
- Telling compaction rows apart by the summary prefix alone: the kept head and tail are
  byte-identical copies of archived rows, which would enter the archive and the index twice.

### 4. The `session_search` index

The history index lives in the C8 journal: an FTS5 table `messages_fts` (columns `content`,
`tool_name`, `tool_calls`, as `FTS_SQL` of the reference; tokenizer `unicode61`, its default) with
rowid `n`, holding its text. `append` inserts the index rows of its `origin` `turn` rows in its own
transaction, so a committed turn is searchable at once and no turn recomputes anything; the
engine extracts the text from the C4G message in Rust, since SQL here has no JSON functions. A
compaction changes flags only and leaves the index as it is, so archived rows stay searchable, as
in the reference. The migration to version 6 fills the index from the existing `turn` rows in its
transaction. Traces to d4 scope 1, 2 and 4.

Reason: the reference keeps the index inside the session database and writes it with the
message; the engine owns that database, the tool runs in the engine's process with no wire hop,
and the forget marks of section 5 join it in one query.

Against "do not start a second independent history index" (d4 scope 4): `messages_fts` is the one
history index. The forest gets none of its own. The archive's `item_fts` is the index of the
archive store over every source (connectors, the old runtime's history after the sprint), most of
which never passes the journal; its dialogue items are not maintained independently of the
journal: they are written only from the journal through `raw_sessions`, carry the text the
journal indexed and the journal's identity (`<session_key>/<session>/<seq>`), and no read path
asks both indexes the same question. `session_search` reads only `messages_fts`; the archive is
read only as the last tier (section 6), and its dialogue hits are filtered by the journal's
marks.

Rejected:
- The index in `ulei-memory` (the forest): the forest is empty, the tool would cross the wire on
  every call, and the index would need its own feed of the journal.
- The archive's `item_fts` as the history index: the archive is immutable and cannot carry
  forget marks, every call would cross the wire and wait for a sync, and d4 places archive search
  outside its scope.
- `AFTER INSERT` triggers as in the reference: they need the text in SQL, which has no JSON
  functions here.
- A contentless table: no `snippet()` (section 2).

The tool. `session_search` in the engine with the reference's name, four shapes and parameters
(`query`, `limit`, `sort`, `detail`, `session_id`, `around_message_id`, `window`, `role_filter`),
its defaults, clamps, per-message bounds, `snippet(..., '>>>', '<<<', '...', 40)` and its rule
that the current session's live rows are skipped while its archived rows are found. Differences:

| Reference | Engine | Why |
|---|---|---|
| No time filter | optional `since` and `until` (RFC 3339 or `YYYY-MM-DD`, UTC) in the discovery shape | d4 scope 1: filters by time. |
| `session_id` without an anchor reads the session and ignores `query` | `session_id` with `query` is a discovery inside that session; without `query` it reads | d4 scope 1: a filter by session. |
| `profile` and the `@session:<profile>/<id>` link | not ported | One journal; the channel renders no link. |
| `title`, `source`, `model` per session, lineage and its dedup, hidden and demoted sources | `session_key`; dedup by session id; no source classes | C8 has no titles, sources or parent sessions; compaction is in place (ADR-0014 section 6). |
| Compaction summaries indexed and kept out of bookends only | rows with `origin` `compaction` are not indexed and never returned | Decision 12:37. |
| Trigram and CJK tables, the LIKE fallback, the stale-index rebuild and merge passes | not ported | The index is written in the message's transaction and is never stale; the owner writes Cyrillic and Latin, which `unicode61` splits. |
| `when` as local time | RFC 3339 in UTC; absent for rows without `at` | The engine has no time zone database (`chrono` is banned). |
| nothing hidden but rewound rows | rows under a live forget mark are left out of every shape | d4 scope 3, section 5. |

### 5. Forget

Marks. "Forget" marks records; it deletes nothing and is undone by lifting the mark.
- History records (the journal's messages, the forest's stand-in while the forest is empty, d10
  scope 5): table `marks` in the journal (C8 next minor version, record `Mark`): `id`, the message
  `n`, `at`, `reason`, `provenance`, `lifted_at` and `lifted_provenance`. A message is forgotten
  while one of its marks has no `lifted_at`. Marks are never deleted, so every mark and lift keeps
  its time, reason and command.
- Seedbed records: a mark of the same fields beside the record in the seedbed's store, which d7
  defines; the garden serves it.
- The archive is not marked and not changed (12:39).

Provenance is the owner's command: `s<session>-t<turn>` and the `n` of the owner's prompt that
asked.

The handle. An engine tool `forget` with `refs`, `query`, `reason` and `undo`:
- `query` (with the optional `session_id`, `since`, `until` of section 4) marks nothing and answers
  the candidate refs from the history index and from the garden's `search` over the seedbed;
- `refs` with `reason` marks them; `refs` with `undo` lifts their live marks;
- refs are `c8:<n>` for a message, `c8:session:<id>` for every message of a session and the
  seedbed's refs of d7; an `archive:<id>` ref is refused;
- it runs only in a turn whose prompt is the owner's (`policy::Origin`, ADR-0022 section 1); any
  other turn gets a refusal.

Which handles filter. `session_search`, every shape; the engine's `archive_search` drops a
`dialogue` hit whose message is forgotten (the archive is untouched; the engine filters its own
answer); C7 `search` and `pointers` leave out forgotten seedbed records (and forest records once
the forest exists). The `archive_search` filter follows from d10's check that a forgotten record
is found by no retrieval handle while its original stays in the archive. Garden facts are not
marked: the garden clears them at the night reconciliation (12:39).

The night reconciliation queue. C7 gains `forget` (`refs`, `reason`, `provenance`, `undo`), which
the engine's tool calls with every ref it marks or lifts. The garden marks the seedbed records
among them and puts into the table `reconcile_queue` of its store one row per current fact whose
`source_ref` equals one of the refs: `fact_id`, the ref, the mark's time and reason, `queued_at`,
`taken_at` (empty). An `undo` lifts the seedbed marks and removes the queue rows of those refs
that the night stage has not taken. The night stage itself is outside the sprint. Traces to d10
scope 1 to 5.

Rejected:
- Marks in the garden for every kind of record: `session_search` would cross the wire on every
  call, and the history marks would live away from the history index (d10 scope 5).
- Marks in the archive: the archive is not changed without a direct command (12:39).
- Deleting the rows from the index: d10 scope 1 requires the mark to be reversible.
- `forget` marking every hit of a query: a vague query would hide records the owner did not mean;
  the query answers candidates and the marks take explicit refs.

### 6. Search order and lifting into the seedbed

`archive_search` (input: `query`, the filters of C10 `search`, `limit` at most 10) walks the tiers
in order: C7 `search` (the garden with its seedbed), then the history index of section 4 as the
forest's tier while the forest is empty, then C10 `search`. It asks the archive only when both
earlier tiers return nothing, and otherwise returns their hits with the note that the archive was
not searched. `archive_read` returns an item's original through C10 `read` (text kinds as text,
other kinds as length and media type). Traces to d8 scope 5 and the search order decision.

Lifting: every archive hit `archive_search` returns is written into the seedbed through the
seedbed's write method of C7 that d7 defines, with source `archive`, ref `archive:<id>`, the
item's `at` and its text bounded to 1,200 characters (the reference's bookend bound). The
seedbed's storage, dedup and day rules are d7's.

Rejected:
- Three tools left to the model's order: the owner's order would rest on a prompt.
- Lifting only what `archive_read` opens: a hit the model uses from its snippet would never reach
  the seedbed.

### 7. Budgets

- `ulei-memory`: a section in `ci/budgets.sh` driven by a new stdlib `ci/memory_run.py` on the
  runner disk: a synthetic `ulei.toml`, a fresh archive filled through `put` with 10,000 synthetic
  items, a fake C6 socket serving `raw_sessions`. Limits: static musl binary 2,621,440 B, VmHWM
  after 10^4 `put`, `search` and `read` 4,096 kB, median spawn to the first `200` of `health`
  10,000 us, 0 voluntary context switches in 10 s idle (ADR-0001's daemon row); p50 of `put`
  10,000 us and of `search` 20,000 us, connect to last byte (the garden's figures, ADR-0047
  section 7); wchar per stored `put` 131,072 B (ADR-0007). Expected size: the inbox-shaped base of
  ADR-0047 section 1 (1.65 MB), FTS5 (140 KB), `miniz_oxide` (37 KB, section 9) and the archive
  code, about 2.0 MB.
- `session_search`: a line of the `ulei-engine` section driven by a new stdlib
  `ci/session_search_run.py`: `ulei-engine serve` fills a journal of 100 sessions of 100 messages
  through C6 `submit_turn` against a fake gateway, which then asks for 200 `session_search` calls
  (discovery, scroll, read); the gap between the gateway's answer carrying the call and the next
  request is the tool's latency. Limit: p50 20,000 us.
- `ulei-engine` keeps its size row; the card that adds the tools reports the size it measured.

Rejected: a wall-clock assertion in a Rust test (a debug build on a shared runner) and no budget
(d4 scope 2 and d8 scope 5 ask for CI budgets).

### 8. Implementation cards, in order

Each card merges alone with the workspace green, tests on synthetic data only, and rewrites in
place every document that describes what it changes. A card that changes a contract or adds a
dependency to a crate carries the short ADR the ADR gate needs, pointing here. "Next minor"
means the next free version when the card lands.

1. Journal version 6 (d4, d8; about 60 min). C8 next minor: `Message.n`, `Message.at`,
   `Message.origin`; migration 5 to 6 with the origin rule of section 3; `append` and `compact`
   write them. Tests: a migrated version 5 file with a compaction has its summary, head and tail
   as `compaction` and the rest as `turn`; new rows carry `at` and increasing `n`.
2. History index (d4; about 60 min). `messages_fts` written by `append` for `turn` rows, filled by
   the migration; `Journal::search` with role, session, time filters, `sort` and `snippet()`.
   Tests: two sessions with different topics, each query finds its session with id and time;
   compaction rows are not found; archived rows are.
3. `session_search` (d4; about 60 min). The tool's four shapes over `Journal::search` with the
   differences of section 4; registered in the engine; `THIRD_PARTY_NOTICES.md` ported row. Tests:
   discovery, scroll, read and browse on a synthetic journal; current session's live rows skipped;
   clamps and bounds.
4. Latency budget of `session_search` (d4; about 60 min). `ci/session_search_run.py` and its line
   in `ci/budgets.sh`, the engine's size measured. Check: the `budgets` job.
5. Forget in the journal (d10; about 60 min). C8 next minor `Mark`, table `marks`, the tool
   `forget` with `query`, `refs`, `undo`, the owner-turn check; `session_search` leaves marked
   rows out. Tests: a marked message is not found by any shape; undo returns it; an event turn is
   refused; the mark keeps time, reason and provenance.
6. C10 0.1.0 and the `[memory]` keys of C9 (d8; about 60 min). Schema, `x-methods`, `Error`,
   fixtures, generated types, `docs/ulei.example.toml`; the description of `ulei-memory` of
   section 1. Tests: fixtures and round trip, `cargo xtask contracts`.
7. The archive store (d8; about 60 min). `ulei-memory` library: `blobs`, `entries`, `item_fts`,
   `deletions`, deflate and SHA-256; dependencies `rusqlite`, `sha2`, `miniz_oxide`. Tests: the
   original reads back byte for byte; a repeated put stores nothing; a changed record under the
   same external id is refused; FTS5 and every metadata filter find the item; only `delete` removes
   an item and it leaves an audit row.
8. `ulei-memory serve` and `ulei-memory delete` (d8; about 60 min). Daemon, C10 transport, peer
   check (`rustix`, `serde_json`), the `ulei-memory` budget section. Tests: every method over a
   temporary socket, each error code, a foreign uid refused; the `budgets` job.
9. The dialogue into the archive (d8; about 60 min). C6 next minor `raw_sessions` served by the
   engine; the archive's pull at start, before `search` and on `sync`, with its cursor. Tests: a
   session's dialogue appears in the archive with the journal's bytes; compaction rows do not; a
   repeated sync adds nothing; the engine down answers `synced: false`.
10. `archive_search` and `archive_read` (d8; about 60 min). The engine's C10 client and the tier
    order of section 6, the forget filter of dialogue hits. Tests against fake C7 and C10 sockets:
    the archive is not asked when the garden or the history index finds something; a forgotten
    dialogue hit is left out while the archive keeps it.
11. Forget in the garden (d10; about 60 min; after d7's seedbed). C7 next minor `forget`, seedbed
    marks, `reconcile_queue`, `search` and `pointers` filtering; the engine's `forget` calls it.
    Tests: after forget the record is found by no handle, the original is in the archive, the queue
    holds the facts whose `source_ref` names the record; undo returns the record and empties those
    queue rows.
12. Lifting into the seedbed (d8; about 45 min; after card 10 and d7's write method). Tests: an
    archive hit is found by C7 `search` among the seedbed's records with ref `archive:<id>`.

### 9. External facts checked

Checked on 2026-11-21.

| Fact | Source |
|---|---|
| FTS5 in bundled SQLite: `libsqlite3-sys` 0.40.0 (`Cargo.lock`) compiles SQLite 3.54.1 with `-DSQLITE_ENABLE_FTS5` (its `build.rs`); `.cargo/config.toml` keeps it and omits JSON (`-DSQLITE_OMIT_JSON`) | `Cargo.lock`, the crate's `build.rs` and `sqlite3.c`, `.cargo/config.toml` |
| A contentless FTS5 table returns NULL for every column but the rowid; contentless-delete tables exist since SQLite 3.43.0 | sqlite.org/fts5.html, section 4.4 |
| `miniz_oxide` 0.9.1, released 2026-03-13, MIT OR Zlib OR Apache-2.0 (all in `deny.toml`'s allow list), pure Rust, not on its ban list; with `adler2` 2.0.1 | crates.io/crates/miniz_oxide |
| Measured: a throwaway static musl binary (release profile of the workspace, `opt-level = "s"`, fat LTO) that deflates at level 6 and inflates its stdin grows from 385,784 B to 422,648 B, +36,864 B. On the repository's ADR texts (513,253 B) the ratio is 2.91, on its contract JSON (220,026 B) 5.95 | throwaway build, not committed |
| `zstd` 0.14.4, BSD-3-Clause, binds the C library 1.5.7 built with `cc` through `zstd-sys` | crates.io/crates/zstd, docs.rs/crate/zstd-sys |
| `ruzstd` 0.9.1, MIT; its encoder implements only the levels `Uncompressed` and `Fastest` (about zstd level 1) | crates.io/crates/ruzstd, docs.rs/ruzstd `CompressionLevel` |
| `lz4_flex` 0.14.2, MIT, pure Rust | crates.io/crates/lz4_flex |

`miniz_oxide` is chosen: pure Rust, so the musl build needs no further C, the smallest measured
addition, and a ratio near 3 on prose. `zstd` compresses better and faster but adds a C library to
the static build; `ruzstd` cannot compress beyond level 1; `lz4_flex` trades ratio for speed the
archive does not need. No crate of the workspace compresses today.

## Consequences

- The engine's journal grows by the indexed text and three columns per message; a journal of
  version 6 is not read by an older engine.
- History, its index and its marks stay in one file of one process; the archive is a separate
  daemon that pulls from the runtime and never writes back.
- The archive is the only store that keeps raw material forever; it is changed only by `put` and
  by the owner's `delete`, each delete leaving an audit row.
- A dialogue message is in two indexes, the journal's and the archive's, fed by one extraction
  and read by different handles at different tiers.
- A garden fact enters the reconciliation queue only when its `source_ref` names the forgotten
  record.
- Not decided here: the systemd unit, release and install of `ulei-memory`; items above 1 MiB;
  the forest's own records and marks; the night stage.
