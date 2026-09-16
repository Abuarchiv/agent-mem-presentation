# agent-mem: Technical Learning Guide

This document is a deep technical guide for understanding the current `agent-mem` project and answering questions about it during the hackathon presentation.

The presentation should not claim that an LLM magically remembers everything. The accurate claim is:

> Agent Mem is a local, source-grounded context layer for coding agents. It captures source events, builds local search representations, and injects only a bounded, traceable context slice into a later session.

## 0. Current project boundary

The implementation discussed here is the separate repository:

```text
/Users/abu/Lernen/agent-mem
```

The presentation repository contains the deck, the speaker script, and this guide. The product repository is a local-first V1 release. Its central path is:

```text
host event
  -> host adapter
  -> authenticated local broker
  -> SQLite source store
  -> FTS5 / local E5 search index
  -> bounded EvidencePacket
  -> host context injection
```

V1 does include:

- local capture from the configured Codex CLI and OpenCode CLI paths;
- SQLite storage for source events, sessions, scopes, spans, jobs, and provenance;
- lexical FTS5 search;
- bundled multilingual E5 semantic search;
- optional local cross-encoder reranking;
- bounded structural source-graph expansion;
- source-linked explicit reports through `memory_write`;
- four stdio MCP tools;
- a local read-only viewer;
- purge, recovery, scope, egress, and integrity checks.

V1 does not use a generative LLM or a cloud provider to decide what a memory means. The source tree contains older schemas and compatibility code for extraction, revisions, graphs, and derived artifacts, but a fresh V1 runtime disables the generative extraction path.

One documentation detail matters: older planning files describe a smaller headless scope without a viewer. The current README, V1 TypeScript build, CLI, and `src/view/` implement a local read-only viewer. The accurate current statement is therefore: **the core has no general REST API; the viewer uses a small local HTTP server to serve an embedded read-only snapshot.**

## 1. The mental model

```text
Original evidence       Search representations       Model context
------------------      ----------------------       -------------
source_event            search_document / FTS5      agent_memory_context
source_span             vector_chunk                selected quotes
payload + metadata      vector_embedding            IDs and timestamps
provenance              optional reranker            bounded byte budget
```

### Important terms

| Term | Meaning | Why it exists |
|---|---|---|
| Host | The coding agent that emits prompts, tool events, and responses | Agent Mem integrates with a host instead of replacing it |
| Adapter | Host-specific integration code | Codex and OpenCode expose different event shapes |
| Scope | An authorization boundary, normally one project/worktree | Prevents unrelated project data from leaking into recall |
| Session | One concrete host session inside a scope | Preserves temporal and native identity |
| Source event | The original normalized host event | Canonical evidence; never replaced by a summary |
| Source span | An exact text range inside a source event | Makes returned text addressable and verifiable |
| FTS5 index | Exact-term search representation | Good for identifiers, paths, symbols, and error codes |
| E5 embedding | Local semantic search representation | Good for paraphrased wording |
| Agent report | Explicit compact note written through `memory_write` | Carries a handoff or decision without pretending it is verified truth |
| EvidencePacket | Structured bounded recall result | Separates retrieval from host-specific injection |
| Broker | Single local owner process for database, model, and scheduler | Coordinates multiple clients and prevents duplicate owners |
| MCP tool | Agent-callable interface | Gives the host narrow read/write/delete capabilities |

The central distinction is:

```text
Capture answers:  What did the host actually emit?
Search answers:   Which stored source might be relevant?
Packet answers:   Which evidence fits this context budget?
Get answers:      Can I inspect the original source in detail?
Write answers:    Which explicit report should survive as a compact note?
```

## 2. The complete capture flow

### 2.1 Host event enters the adapter

Examples of native events are:

- Codex `SessionStart`;
- Codex `UserPromptSubmit`;
- Codex `PostToolUse`;
- Codex `Stop`, `PreCompact`, and `PostCompact`;
- OpenCode `chat.message`;
- OpenCode `tool.execute.before` and `tool.execute.after`;
- OpenCode `experimental.text.complete`;
- OpenCode session and message-part events.

The adapter maps these into a neutral event contract. The neutral contract records:

- a `stage`, such as `prompt_submitted`, `tool_result`, or `message_part`;
- a `role`: `user`, `assistant`, `tool`, or `system`;
- an `evidence_class`: `prompt`, `assistant_output`, `tool_input`, `tool_output`, `lifecycle`, or `diagnostic`;
- native IDs such as session, turn, message, part, and tool-call IDs;
- the bounded payload and optional text;
- capture and occurrence timestamps;
- coverage and correlation information.

The adapter also resolves the current working directory against an explicitly configured workspace route. A path outside the configured project is rejected. The scope is not selected by a prompt or by untrusted payload fields.

### 2.2 Redaction and contract validation

Before persistence, `redactCaptureInput` walks the bounded JSON input. It removes secret-shaped keys and known token formats, strips private blocks, and preserves a redaction policy version.

The contract layer then validates:

- JSON depth, size, node count, and cycles;
- event stage and stage-specific role/evidence class;
- required and optional text fields;
- native IDs and correlation data;
- scope and host identity;
- capture timestamps and truncation metadata.

This is deliberately fail-closed at the storage boundary. A malformed or overlarge event is not partially stored.

### 2.3 Authenticated local broker

The adapter does not open SQLite directly. It authenticates to the broker using a per-connection credential over a local TLS-PSK channel.

The broker checks:

- the binding ID;
- the secret selected during TLS;
- whether native-session connections are allowed;
- the requested native session ID;
- the derived session binding;
- scope and session registration.

The broker uses bounded newline-delimited JSON frames with request IDs and sequence numbers. Replayed or malformed frames are rejected.

### 2.4 One SQLite transaction

The source commit creates the canonical source row and related metadata in one transaction. Conceptually it writes:

1. `source_event` — original payload, event metadata, provenance, timestamps, coverage, and identity;
2. `capture_acceptance` — when the store accepted it and how long it is retained;
3. `source_span` — exact text addresses and digests;
4. `search_document` — the FTS5 representation for each searchable span;
5. an `embed` job when the V1 runtime has the E5 embedding task enabled;
6. the monotonic commit counter and data epoch.

If the process crashes before commit, SQLite rolls back the transaction. If the source is committed but embedding has not finished, the source remains usable for exact search and the embedding job can be retried after restart.

### 2.5 What a source span means

A span is an authenticated pointer into the sanitized source:

```json
{
  "span_id": "span-uuid",
  "root": "payload",
  "path": "/text",
  "start_utf16": 0,
  "end_utf16": 42,
  "digest": "sha256-of-the-excerpt"
}
```

The `root` can be `payload` or `event`. The `path` is a JSON Pointer. UTF-16 offsets match JavaScript string indexing, and the code rejects ranges that split a surrogate pair.

When a source is read later, Agent Mem resolves the path again, extracts the exact range, hashes it, and compares it with the stored digest. An index row cannot silently substitute unrelated text.

## 3. What is stored, and what is only derived?

### 3.1 Canonical source data

`source_event` stores the evidence that came from the host. It includes the sanitized payload and normalized event metadata, not just a generated summary.

Important fields include:

- `capture_id`;
- `scope_id` and `session_id`;
- `adapter_version`;
- `observed_stage`;
- `role` and `evidence_class`;
- native session, turn, message, part, and tool-call IDs;
- `captured_at` and optional `occurred_at`;
- `payload_json` and `event_json`;
- `truncation_json`, `redaction_json`, and `coverage_json`;
- `commit_seq` and `data_epoch`.

`session` stores the host identity for the session. `scope` stores the project-level boundary, owner, data epoch, and privacy epoch.

### 3.2 Search data

Search data is rebuildable derived state:

- `search_document` contains searchable source-span text;
- `search_fts` is the SQLite FTS5 index;
- `vector_chunk` stores E5-sized chunks and their source mapping;
- `vector_embedding` stores the 384-dimensional float vector;
- `vector_generation` identifies the active vector generation;
- `job` tracks background embedding work.

The search index is not authoritative. It helps locate candidates, but every candidate is hydrated and revalidated against canonical source rows before it can be returned.

### 3.3 Explicit agent reports

`memory_write` writes a compact `agent_report` as a `search_enrichment` derived artifact. It is source-linked through dependency rows.

Example:

```json
{
  "format": "agent_memory_record_v1",
  "origin": "agent_report",
  "kind": "decision",
  "key": "database-choice",
  "summary": "SQLite remains the local storage backend.",
  "next_steps": [],
  "source_ids": ["capture-uuid"]
}
```

An agent report is not automatically verified. Its source dependencies make it auditable and make it disappear from current recall when the supporting source is purged.

When a report with the same kind and key changes, the new write must identify the current revision through `replaces`. The previous revision is blocked rather than silently overwritten.

### 3.4 Query traces and sidecar state

`query_trace` stores search metadata, not query or source text. It records IDs, scope epochs, candidate IDs, output IDs, diagnostics, mode, budget, packet digest, and delivery state.

The private `search-state.json` sidecar stores:

- explicit feedback labels;
- procedure rules that point to existing captures;
- purge epochs;
- short-lived search reports used by feedback.

This sidecar is not a second source database. It is bounded ranking and control state.

## 4. How retrieval works

### 4.1 Query preparation

The request contains plain query text, a trusted or narrowed scope set, a mode, and a budget. Long or complex queries are reduced to bounded terms while preserving technical anchors such as paths, identifiers, and quoted phrases.

The request is validated against the authenticated binding. A tool argument cannot add a new scope or a new output target.

### 4.2 Lexical FTS5 search

The lexical path converts the query into quoted FTS terms and searches `search_fts`. The database applies scope, output grant, commit watermark, purge, and current-session filters before the result limit.

Each hit is hydrated from canonical source data. The span digest is checked and the returned quote must equal the indexed document text.

Lexical search is especially valuable for:

- function names;
- file paths;
- exact error messages;
- UUID-like identifiers;
- release names and technical anchors.

### 4.3 E5 semantic search

The bundled model is `Xenova/multilingual-e5-small`.

```text
dimensions:   384
maximum:      512 model tokens
dtype:        q8
device:       CPU
input:        query: ... / passage: ...
pooling:      masked mean
normalization: L2
```

Stored source spans are split into tokenizer-checked chunks. The worker never silently truncates an overlong unit. It stores the chunk text, source span, character range, digest, model profile, tokenizer version, chunker version, and generation.

At query time, the same model embeds `query: <user query>`. Vector distance is cosine distance. The embedding model finds semantic similarity; it does not generate an answer and does not determine factual truth.

If the E5 model cannot load or produce a valid vector, capture still works and retrieval falls back to FTS5.

### 4.4 Hybrid fusion

When both signals are available, lexical and vector rank lists are combined using Reciprocal Rank Fusion with fixed `k = 60`:

```text
score(rank) = 1 / (60 + rank + 1)
```

The system deduplicates at source level so repeated chunks from one source do not crowd out other sources. Candidates are overfetched before fusion and remain bounded.

### 4.5 Source intelligence

The ranking layer adds bounded signals for:

- lexical match;
- semantic match;
- structural graph membership;
- explicitly registered procedure hints;
- recency.

The query is classified as an identifier, recent, relation, procedure, or semantic query. A relation-style query can trigger the bounded source graph. Explicit feedback can adjust weights after enough samples, but the feedback does not create facts.

### 4.6 Structural graph expansion

The active source graph is deliberately conservative. It can connect sources through:

- shared native message IDs;
- shared tool-call IDs;
- the same explicit file path;
- the immediate previous or next event in a session.

The relation search is limited to two hops and a small node budget. Connected graph paths are admitted atomically into the packet. If the budget or deadline prevents completion, the packet reports `graph_incomplete`.

This is not a claim that the system understands arbitrary semantic relations. The older `entity` and `semantic_edge` tables are compatibility infrastructure and are not automatically populated by fresh V1 capture.

### 4.7 Optional reranking

The optional local reranker is a q8 CPU cross-encoder:

```text
cross-encoder/mmarco-mMiniLMv2-L12-H384-v1
```

It reranks only a small candidate pool. It cannot create new sources. If the extra is absent or fails, the baseline ranking remains available.

## 5. Exactly what enters a new session

### 5.1 Automatic Codex flow

For Codex, the adapter handles events such as `SessionStart` and `UserPromptSubmit`.

At `SessionStart`, the configured query is normally:

```text
recent project work
```

At `UserPromptSubmit`, the current user prompt becomes the retrieval query.

The current prompt is captured first. Automatic recall then excludes current-session prompts so the newly stored prompt cannot immediately return as fake historical context.

Codex receives the serialized packet through the hook result field:

```text
hookSpecificOutput.additionalContext
```

### 5.2 Automatic OpenCode flow

OpenCode observes native message and part events through its plugin and a persistent bridge helper.

The transform path only injects after the actual `chat.message` prompt has been captured and acknowledged. It mutates the native messages array by appending one synthetic text part containing the `agent_memory_context` wrapper.

Previously recognized Agent Mem synthetic parts are removed before a new one is appended. Recognition is based on an authenticated packet or injection identity, not on an arbitrary marker substring.

OpenCode also reconciles native message-part history when an event arrives without enough live text. Missing coverage remains explicit rather than being guessed.

### 5.3 The compact injected wrapper

The host does not receive the whole vault. It receives a bounded JSON wrapper similar to:

```json
{
  "version": 1,
  "kind": "agent_memory_context",
  "injection_id": "uuid",
  "mode": "timeline",
  "items": [
    {
      "item_id": "capture-uuid",
      "capture_id": "capture-uuid",
      "scope_id": "scope-uuid",
      "source_class": "assistant_output",
      "role": "assistant",
      "status": "candidate",
      "captured_at": "2026-09-15T10:00:00Z",
      "occurred_at": null,
      "spans": [
        {
          "span_id": "span-uuid",
          "quote": "SQLite remains the local storage backend."
        }
      ]
    }
  ]
}
```

The full internal EvidencePacket also contains source provenance such as JSON paths, UTF-16 ranges, digests, commit sequence, epochs, diagnostics, and usage. The compact model wrapper keeps only the context fields needed for the host plus the selected quotes and IDs.

### 5.4 Default egress classes

The default reader grant allows:

```text
prompt
assistant_output
```

Tool input, tool output, lifecycle events, and diagnostics can be stored and inspected in the local viewer, but they are not automatically injected into the agent reader context by default.

A source-linked report is injected as a `record` item. It carries its compact report content and source references; it does not pretend that the report itself is an original source quote.

### 5.5 Context budgets

The active host profiles use explicit UTF-8 byte budgets.

| Host path | Session-start profile | Prompt/transform profile |
|---|---:|---:|
| Codex | about 4,000 bytes | about 8,000 bytes |
| OpenCode | about 4,000 bytes for session start | about 8,000 bytes for transform |
| Copilot CLI adapter | 600 token-shaped compatibility budget | 1,500 token-shaped compatibility budget |

Some compatibility fields are still named `token_budget`, but the active host context profile measures UTF-8 bytes. This is deliberately conservative and does not claim a universal token-to-byte ratio.

The packet builder protects the latest task and a compact handoff before older optional material. If the budget is insufficient, it drops optional evidence and reports a diagnostic instead of silently pretending that all matches were delivered.

### 5.6 What is not injected

The automatic wrapper does not contain:

- the complete SQLite database;
- every candidate found by search;
- embedding vectors;
- query text in the durable query trace;
- arbitrary tool output under the default reader policy;
- an automatically generated factual summary;
- unbounded raw history.

If the agent needs the original source, it can use `memory_get` with an ID from recall.

## 6. The four MCP tools

The current MCP catalog contains exactly four tools:

```text
memory_recall
memory_get
memory_write
memory_forget
```

They are exposed through `tools/list`. Older schemas for correction, history, or diagnosis are not current V1 catalog entries.

### 6.1 `memory_recall`

Purpose: search the local store and return a bounded EvidencePacket.

Example:

```json
{
  "query": "database migration decision",
  "mode": "current",
  "max_bytes": 8000
}
```

Inputs:

- required `query`, up to 4,096 characters;
- optional `scope_ids`, which can only narrow trusted scopes;
- `mode`: `current`, `historical`, or `timeline`;
- optional `valid_at` and `known_at_seq` for temporal-compatible reads;
- `max_bytes`, from 1 to 32,000, default 8,000;
- legacy alias `token_budget`, also interpreted as the byte limit.

Returns:

- `query_id`;
- `injection_id`;
- watermark and scope epochs;
- selected source items;
- exact source provenance;
- packet mode and diagnostics;
- used and allowed budget;
- optional intelligence metadata;
- up to ten omitted source references for follow-up reads.

`memory_recall` is read-only. It does not write a new memory.

### 6.2 `memory_get`

Purpose: load one permitted original source or one saved report.

Original source:

```json
{
  "scope_id": "scope-uuid",
  "reference": {
    "kind": "source",
    "capture_id": "capture-uuid"
  }
}
```

The source result contains the sanitized payload, event metadata, coverage, stage, role, evidence class, and verified spans.

Saved report:

```json
{
  "scope_id": "scope-uuid",
  "reference": {
    "kind": "record",
    "revision_id": "revision-uuid"
  }
}
```

The read boundary checks scope, egress grant, purge status, source existence, span range, and digest. A random or unauthorized ID returns an error rather than granting access.

### 6.3 `memory_write`

Purpose: save a compact source-linked handoff, decision, preference, or procedure report.

Example:

```json
{
  "scope_id": "scope-uuid",
  "key": "release-decision",
  "kind": "decision",
  "summary": "The release uses a local SQLite vault.",
  "next_steps": ["Verify the packaged smoke test"],
  "source_ids": ["capture-uuid"]
}
```

Rules:

- a key and stable kind are required;
- summary is bounded;
- up to five next steps are allowed;
- at least one source ID is required;
- the cited source must exist and be visible to the binding;
- no second model is called;
- the report is explicitly marked as an `agent_report`, not verified truth.

If the same report changes, `replaces` must identify the current revision. The old revision becomes blocked and is excluded from new current recall.

The `procedure` kind is a source-linked note. It is not an executable workflow and is not automatically promoted into an authoritative procedure.

### 6.4 `memory_forget`

Purpose: purge selected source captures and their managed dependents.

Example:

```json
{
  "version": 1,
  "scope_id": "scope-uuid",
  "capture_ids": ["capture-uuid"],
  "expected_privacy_epoch": "0"
}
```

The operation can also carry a stable `operation_id` for retry.

The purge writes a durable barrier and tombstones before physical cleanup. It removes or invalidates source rows, spans, FTS documents, vectors, jobs, dependent reports, dependency rows, feedback, and related traces as defined by the purge inventory.

If runtime cleanup or model reset is still pending, the tool returns `state: pending` with a phase and pending reasons. Pending does not mean the deleted source is still eligible for new recall; the tombstone already blocks replay.

## 7. Operator CLI versus MCP tools

The four MCP tools are what the agent sees. The operator CLI has additional control commands:

```text
agent-mem install
agent-mem repair
agent-mem connect / disconnect codex|opencode|copilot-cli
agent-mem start
agent-mem stop
agent-mem view
agent-mem status
agent-mem pause / resume
agent-mem forget
agent-mem feedback
agent-mem procedure add / list / remove
agent-mem extras list / install / remove reranker
agent-mem mcp --connection <id>
```

`install` detects or accepts host selections, verifies the package and model assets, writes host configuration, starts the owned broker, performs an MCP smoke check, and records a journal so a failed installation can be repaired.

The host configuration creates:

- a project scope ID;
- a host connection binding ID;
- a private per-connection secret;
- a fixed workspace route;
- a broker path;
- the MCP command;
- the host hook or plugin configuration.

Host files are backed up before owned blocks are changed, and concurrent foreign edits are not overwritten.

## 8. Feedback and procedure hints

Feedback is not model training. The operator can label a recent search candidate as useful or not useful using its `query_id` and `capture_id`.

After a minimum number of samples for a scope and search kind, the sidecar adjusts bounded ranking weights. Reports expire from in-memory search state after a short TTL, and persistent feedback remains bounded.

Procedure hints point to existing source captures and matching terms. A matching registered procedure can receive a ranking boost. It does not create authority, execute code, or turn arbitrary text into a verified workflow.

## 9. Recovery and integrity

### Lost ACKs and retries

When a transport ACK is lost, the adapter retries the same immutable normalized event. It does not generate a new event with a new semantic meaning.

Correlated native events use stable deterministic IDs. Unknown correlation stays explicit and receives a fresh capture ID rather than being guessed from similar text.

### Watermarks and epochs

- `commit_seq` is the monotonic store ordering and query watermark;
- `data_epoch` changes as data changes;
- `privacy_epoch` changes across purge operations;
- `known_at_seq` lets compatible reads ask what was known by a specific commit point.

Retrieval takes a snapshot and revalidates it before delivery. If a source is purged or its authorization changes during the operation, the packet is rejected or replaced with a safe empty result rather than returning a stale successful response.

### Append-only and dependencies

Source-linked derived artifacts retain dependencies to their supporting spans. A correction or purge can invalidate dependents transitively. The current V1 report path uses this dependency structure even though generative extraction is disabled.

## 10. Security and privacy

The correct statement is **local by default**, not **automatically secure**.

Protections include:

- private data directories and config files;
- rejection of unsafe symlinks and permissions;
- authenticated local TLS-PSK broker transport;
- bounded JSON and frame sizes;
- redaction before persistence;
- trusted scope and output bindings;
- digest-authenticated source spans;
- hash-verified model artifacts;
- no provider calls from the V1 runtime.

Limits include:

- SQLite is not encrypted at rest;
- redaction cannot recognize every possible secret format;
- local backups and OS snapshots are outside the purge guarantee;
- the host may send recalled context to its own model provider;
- stored memory text is untrusted data and must not be executed as instructions.

The host adapter is fail-open from the user's perspective: if Agent Mem is unavailable or misses its deadline, the coding agent continues without memory context.

## 11. The local viewer

`agent-mem view` activates a separate `local_ui` output grant and builds a read-only snapshot.

The viewer can show:

- dashboard health;
- projects and scopes;
- sessions;
- source events and spans;
- memory records;
- jobs and embedding state;
- source and semantic graph slices;
- privacy grants and purge operations;
- query traces;
- measured evidence reduction.

The snapshot is embedded into the HTML response. The viewer has GET-only routes, no general REST API, and no write path into the vault.

The viewer is useful in the presentation because it makes the invisible pipeline inspectable. It is not the memory engine itself.

## 12. Research paper to implementation

The research review did not prove one universal memory architecture. It extracted design consequences and then narrowed V1.

| Research direction | What it motivated | What V1 actually does |
|---|---|---|
| RAG | Separate retrieval from generation and preserve source passages | Local retrieval and source-backed packets; no answer generator |
| Generative Agents | Distinguish observations, reflections, and plans | Preserve observations; no automatic reflective LLM writes |
| MemoryBank | Use multiple time scales and representations carefully | Keep source evidence and explicit reports separate; no automatic decay profile |
| Reflexion | Feedback can affect later attempts | Optional explicit feedback changes bounded ranking weights |
| MemGPT | Context is finite and explicit recall is useful | Bounded packet plus `memory_get` point reads |
| HippoRAG | Relations can help multi-hop retrieval | Small structural graph expansion, not free-form entity inference |
| LongMemEval | Indexing, retrieval, and reading must be measured separately | Source/index/packet boundaries, budgets, traces, and diagnostics |
| A-MEM | Notes need links and evolution | Source-linked agent reports with revisions and dependencies |
| Mem0 | Updates need explicit operations | `memory_write` with `replaces`; no automatic truth resolver |

The strongest research-derived principle is:

> Keep the original evidence available, keep injected context small, and never let a derived representation lose its source lineage.

## 13. Questions developers may ask

### “Is this an LLM memory system?”

Not in V1. The runtime uses a local embedding model for semantic search, but it does not call a generative model to summarize or decide truth. The host model receives retrieved evidence and remains responsible for producing the final answer.

### “Why not inject the whole transcript?”

Because context is finite and full transcripts contain noise. Agent Mem preserves source material locally but sends only relevant, bounded spans into the current model context.

### “Why both FTS5 and embeddings?”

FTS5 is strong for exact technical anchors. Embeddings are strong for paraphrases. Hybrid fusion combines both signals.

### “Can a retrieved result still be wrong?”

Yes. A retrieval hit means that the stored source matches the query. Provenance verifies where the text came from; it does not prove that the host's original statement was factually correct.

### “How do you prevent stale results?”

The system uses commit watermarks, data and privacy epochs, purge tombstones, scope/output checks, source-span digests, and final snapshot revalidation before delivery.

### “What if the E5 model fails?”

Capture is still committed. FTS5 remains available and the packet reports degraded semantic search.

### “What does the graph understand?”

The active graph is structural: shared native IDs, file paths, and nearby session events. It is bounded and does not infer arbitrary semantic truth from similarity.

### “What exactly does `memory_recall` return?”

It returns a bounded `memory_recall_result` containing an EvidencePacket: selected source items, IDs, quotes, provenance, scope epochs, budget usage, mode, diagnostics, and optional ranking metadata.

### “Why do we need `memory_get`?”

Recall is a compact candidate result. `memory_get` is a deliberate point read that loads the original permitted source and verifies its spans again.

### “Why does `memory_write` require source IDs?”

To prevent an ungrounded report from entering the memory path. A handoff or decision can be compact, but it must point back to the evidence that supports it.

### “Does `memory_write` create a verified fact?”

No. It creates an explicit source-linked agent report. The report is auditable but not automatically certified as true.

### “What happens when the same decision changes?”

The new report uses `replaces` to identify the current revision. The old report is blocked rather than silently overwritten, and new current recall excludes it.

### “Does local mean encrypted?”

No. The vault is local but SQLite is not encrypted at rest. Filesystem permissions, backups, and the host's later provider egress still matter.

### “Does the system send data to OpenAI, Anthropic, or OpenRouter?”

The V1 runtime itself does not make provider calls. The host may later send injected context to its configured provider as part of normal answer generation.

### “Is the viewer the database?”

No. The viewer reads a local snapshot. Capture, search, MCP, and purge are implemented in the runtime and work independently of the viewer.

### “Which integrations are production-proven?”

The release evidence verifies the Codex CLI and OpenCode CLI paths. The Copilot CLI adapter exists, but its end-to-end release status is conservative/unverified. Codex Desktop and the Copilot app are outside the demonstrated V1 boundary.

### “Why is there old extraction or correction code in the repository?”

The repository retains compatibility schemas and repositories for older vaults and future paths. The V1 runtime passes `extraction_enabled: false` and exposes only the four current MCP tools.

## 14. Presentation cheat sheet

### 30-second explanation

> Coding agents lose the reasoning behind previous work when a session ends. Agent Mem captures the host's original source events locally, keeps exact source spans, and builds FTS5 and E5 representations for retrieval. When a new session starts, it returns only a bounded EvidencePacket with selected quotes and provenance. The host can inspect an original source with `memory_get`; it can write a compact source-linked handoff with `memory_write`; and it can remove data with `memory_forget`. V1 itself does not generate or verify facts.

### Five words to remember

```text
Capture → Provenance → Retrieval → Bounded context → Verification
```

### Four tool verbs

```text
recall = find evidence
get    = inspect one source or report
write  = save an explicit source-linked report
forget = purge selected sources and dependents
```

### Phrases to avoid

- “The AI remembers everything.”
- “The memory is automatically true.”
- “The local database is encrypted.”
- “The graph understands the project.”
- “Every host integration is production verified.”

### Strong closing line

> Agent Mem does not ask a model to invent a memory. It preserves what happened, finds the relevant evidence locally, and gives the next session only the part that fits and can still be traced back to its source.

## 15. Recommended reading order in the product repository

1. `README.md` — current user-facing V1 boundary;
2. `src/host/tool-schemas.ts` — the exact four-tool MCP catalog;
3. `src/host/mcp.ts` — MCP protocol and tool dispatch;
4. `src/host/broker.ts` — authenticated local transport;
5. `src/core/capture.ts` and `src/core/redact.ts` — source admission;
6. `src/store/schema.sql` — canonical tables and policy state;
7. `src/context/source-only.ts` — retrieval orchestration;
8. `src/context/packet.ts` — packet budget and injection wrapper;
9. `src/retrieval/lexical.ts`, `vector.ts`, `fusion.ts`, and `source-intelligence.ts` — ranking;
10. `src/app/runtime.ts` and `src/worker/main.ts` — model ownership and background work;
11. `adapters/codex/index.ts` and `adapters/opencode/plugin-runtime.ts` — host-specific injection;
12. `src/view/server.ts` and `src/view/model.ts` — read-only inspection UI.
