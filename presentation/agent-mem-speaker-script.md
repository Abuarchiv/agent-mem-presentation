# agent-mem: Speaker Script

**Target:** 5:35 minutes of spoken content, leaving up to 2:25 for the live Archify walkthrough and questions inside an eight-minute slot.

The talk has one thread: a coding agent should be able to retrieve a past decision and inspect the source behind it.

## Timing map

| Slide | Time | Purpose | Visual or action |
|---:|---:|---|---|
| 1 | 0:00–0:10 | One-sentence introduction | Title slide |
| 2 | 0:10–0:45 | The problem | No external asset |
| 3 | 0:45–1:25 | The design decision from the research | Three-part explanation |
| 4 | 1:25–2:00 | Show the real product state | Dashboard screenshot |
| 5 | 2:00–3:25 | Explain persistence | Storage diagram, then Archify `Persisted` |
| 6 | 3:25–4:55 | Explain session injection | Sequence diagram, then Archify `In the Session` |
| 7 | 4:55–5:35 | Close with result and limits | Final slide |

## Slide 1: Title

“This is `agent-mem`, a local memory service for coding agents. I will show how a stored source becomes bounded context in a later session.”

## Slide 2: The gap between sessions

“A coding agent can read files, run tools, and change a software project. During the task, the prompt, files, tool results, and decisions are available in the current context.

When that session ends, the next session may know the code but not the reasoning behind an earlier decision. It may also retrieve an old statement after that statement was corrected.

The question is simple: how does the next session retrieve evidence that we can still verify?”

## Slide 3: The design decision

“The research review led to one design decision: keep the source complete and keep injected context small.

That creates three separate responsibilities. The source stores the original event and its exact span. Local search finds candidates with FTS5 for exact terms and E5 for similar wording. The context layer selects an `EvidencePacket` that fits the current scope and budget.

This separation is the important idea from the research. Storage, retrieval, and context handoff are related, but they are not the same operation.”

## Slide 4: The `agent-mem` viewer

“Before looking at the internals, this is the actual local viewer.

It exposes the current state of `agent-mem`: sessions, sources, spans, jobs, graph sources, local health, and the evidence-reduction readout.

The recent sources table shows which host and session produced the evidence, its role, evidence class, storage state, and span count. The reduction panel compares the stored baseline with what reaches the active context. In this snapshot, 566 stored tokens become 15 surfaced tokens, a measured reduction of 97.3 percent.

This gives us an inspectable product state before we follow the data through the system.”

## Slide 5: Persistence, the source stays local

“The first Archify view answers: what is stored?

A host event enters the capture hook. The hook normalizes and redacts the input before committing it to SQLite.

`source_event` stores the event payload and its metadata: project scope, session, stage, role, evidence class, timestamps, coverage, and correlation. `source_span` stores the exact source location with UTF-16 offsets and a SHA-256 digest.

The system also builds two local search representations. FTS5 handles exact terms, identifiers, and file names. E5 helps find semantically similar wording.

The index is only a way to find evidence. It does not replace the stored source. That is why a later result can still point back to the original text.”

**Live action:** Open `archify/agent-mem-storage-dataflow.html`. Select the guided view `Persisted`, then `Search`. Point out `source_event`, `source_span`, `FTS5 + E5`, `EvidencePacket`, and `memory_get`.

## Slide 6: Session handoff, only selected evidence enters

“The second Archify view answers: what reaches the model?

At `SessionStart`, the adapter uses a configured start question. For a new user prompt, it uses the current prompt as the recall query. The request carries the project scope, mode, and a bounded context budget.

FTS5 and E5 produce candidates. The Context Builder filters them by scope, session exclusions, deadline, and budget. Only the approved source spans become the `EvidencePacket`.

The host then serializes an `agent_memory_context` wrapper. It carries the injection ID, source IDs, scope, role, status, timestamps, and selected quotes from the spans.

The model receives the current prompt plus those selected quotes and metadata. It does not receive the complete local store, every candidate, the vectors, or the query trace.

If a quote needs to be checked, the agent calls `memory_get`. The read validates the scope, source reference, span range, and digest before returning the complete permitted source.”

**Live action:** Open `archify/agent-mem-session-injection-sequence.html`. Select `In the Session`, then show the sequence from `SessionStart` through recall, injection, and `memory_get`.

## Slide 7: Result and limits

“The result is a clear handoff. A later session can retrieve a past decision and inspect the source behind it.

Today, `agent-mem` provides local capture, FTS5 and E5 retrieval, and bounded source-linked context. The next extension is revision and time handling for stale or conflicting evidence.

The system stores more than it injects. That keeps the active context small while preserving the evidence needed to verify a decision.”

## Technical backup for questions

- `SessionStart` and user prompts use separate context profiles of roughly 4,000 and 8,000 UTF-8 bytes. Host limits are separate.
- `token_budget` is a protocol name. The context profiles measure conservatively in UTF-8 bytes.
- E5 and the optional cross-encoder are local, non-generative models.
- The host adapter passes the context wrapper. The memory service does not generate the final answer.
- `memory_get` loads the complete source explicitly and checks the source reference again.
- Local storage does not automatically mean encryption. The SQLite file and host egress need separate protection.

## Research references for follow-up questions

The review grouped the papers by the decision they inform: RAG, HippoRAG, and LongMemEval for retrieval; Generative Agents, MemoryBank, MemGPT, and Reflexion for memory roles; A-MEM and Mem0 for provenance and updates.
