# agent-mem: Speaker Script

**Target time:** 8:00 minutes, including a short opening on the title slide

**Presentation rule:** The paper explains where the architecture came from. The talk describes the current `agent-mem` scope and does not claim that the complete research design is already implemented.

## Timing map

| Slide | Time | Visual | Source or action |
|---:|---:|---|---|
| 1 | 0:00–0:10 | Title slide | Short introduction |
| 2 | 0:10–1:00 | The memory problem | No external asset |
| 3 | 1:00–1:50 | Research findings | Compact table from the paper review |
| 4 | 1:50–2:35 | The `agent-mem` viewer | `presentation/assets/agent-mem-ui-dashboard-compact-tight.png` |
| 5 | 2:35–4:05 | Persistence dataflow | `agent-mem-storage-dataflow.html` live or `presentation/assets/agent-mem-storage-diagram-tight-en.png` |
| 6 | 4:05–5:30 | Session injection sequence | `agent-mem-session-injection-sequence.html` live or `presentation/assets/agent-mem-session-sequence-tight-en.png` |
| 7 | 5:30–7:25 | Guided Archify walkthrough | Open both HTML files and use the guided views |
| 8 | 7:25–8:00 | Result and limits | No external asset |

**Time check:** 10 + 50 + 50 + 45 + 90 + 85 + 115 + 35 seconds = 480 seconds = 8 minutes.

## Slide 1: Title

**Time: 0:00–0:10**

“Today I am showing `agent-mem`, a local memory service for coding agents. The talk follows one question: how does a stored source return to a session as bounded context?”

## Slide 2: The memory problem

**Time: 0:10–1:00**

“A coding agent can read files, run tools, and propose changes to a software project. During a session, it works with a finite active context. When the session ends, the next session often loses the decisions and reasons from the earlier work.

This becomes especially risky when an earlier statement was later corrected. If the agent retrieves only the old statement, it may continue with a stale configuration or repeat work that was already completed.

The central question is therefore: how do we preserve project history so that an agent can find relevant evidence later and we can still trace that evidence back to its exact source?”

## Slide 3: Research origin

**Time: 1:00–1:50**

“The starting point was a review of nine research papers on long-term memory.

RAG, HippoRAG, and LongMemEval show that retrieval is made up of separate tasks. A system must distinguish what was stored, what the search finds, and what is allowed into the current context.

Generative Agents, MemoryBank, MemGPT, and Reflexion show that original observations, derived experience, context management, and lessons have different roles.

A-MEM and Mem0 show the risk in updates. New information must not silently overwrite older sources. Provenance and identity remain important.

That research led to a larger architecture. For `agent-mem`, I selected the local, source-grounded core: reliable capture, local search, bounded evidence, and traceable sources.”

## Slide 4: The `agent-mem` viewer

**Time: 1:50–2:35**

“Before following the data through the system, I want to show the actual local UI.

This is a read-only viewer over the `agent-mem` state. It shows sessions, sources, spans, jobs, graph sources, local health, and the context-reduction readout.

The recent sources table is the important part. It exposes the host session, role, evidence class, storage state, and span count. The reduction panel compares the stored baseline with the evidence surfaced to the active context. In this snapshot, 566 stored tokens become 15 surfaced tokens, a 97.3 percent reduction.

This view is not a second memory system. It is an inspection surface for the persisted state. The following two diagrams explain how those sources are stored and later returned to a session.”

## Slide 5: Archify, what gets stored

**Time: 2:35–4:05**

“I am opening the Archify dataflow view. It answers the first concrete question: what stays in local storage?

The flow starts with a host event. That can be a prompt, a tool event, or a lifecycle event. The capture hook normalizes and redacts the input.

The central storage object is `source_event`. It contains the normalized event with its payload, project scope, session ID, stage, role, evidence class, timestamps, coverage, and correlation data.

Next to it is `source_span`. A span identifies the exact evidence inside the source: its path, UTF-16 start and end offsets, and SHA-256 digest. This lets the system check later whether a quote really belongs to that stored source.

The source also produces search representations. FTS5 handles exact terms, IDs, and file names. E5 creates a local vector so that similar wording can be found as well.

The important separation is this: the index helps us find a source. It does not replace the stored source text. A technical query trace, such as query ID, injection ID, packet digest, candidate IDs, and budget, supports auditability. It is not inserted as memory text into the session.”

**Archify action:** Select the guided view `Persisted`, then `Search`.

## Slide 6: Archify, what enters the session

**Time: 4:05–5:30**

“Now I switch to the session view.

At `SessionStart`, the adapter uses a configured start question. For a new user prompt, it uses the current prompt as the recall query. The request includes project scope, mode, and a bounded context budget.

The search combines FTS5 and E5. The Context Builder then filters candidates by scope, session exclusions, deadline, and budget. Only approved candidates become the `EvidencePacket`.

For the session, the system serializes an `agent_memory_context` wrapper. The wrapper contains an `injection_id`, the mode, and selected items.

For normal source items, the active context receives the source ID, scope, source class, role, status, capture time, event time, and selected spans with their quotes.

Depending on the integration, the host adds this wrapper as `additionalContext` or as a synthetic context part.

The model therefore sees the current prompt plus selected quotes and metadata. By default, it does not see the complete local store, every search candidate, the E5 vectors, the query trace, or the complete payload of every event.

If more detail is needed, the agent calls `memory_get` explicitly. That read checks the scope, reference, span range, and digest again before returning the complete permitted source.”

**Archify action:** Select the guided view `In the Session`, then open the sequence view in the second diagram.

## Slide 7: Archify as the workflow explanation

**Time: 5:30–7:25**

“I will now walk through the complete workflow.

First, capture. A host event occurs during the work. The adapter sends it to the local IPC broker. The broker checks scope and authentication. The capture layer cleans the input and writes `source_event` and `source_span` to SQLite. The acknowledgement is returned only after the commit.

Second, indexing. The stored source is prepared for FTS5 and E5. FTS5 can find an exact term such as `SQLite`. E5 can also connect a question such as ‘Why is there no external database here?’ to a source that uses different wording.

Third, recall. The adapter sends a query with scope and a byte budget. The search stages produce candidates. The context builder then keeps only source spans that fit the scope and budget.

Fourth, injection. The Context Builder creates the `agent_memory_context` wrapper. The host adds it to the active session. The model receives the current prompt together with the selected quotes.

Fifth, source read. If a quote needs to be checked in detail, the agent calls `memory_get`. The response is then the complete permitted original source, not just a search snippet.

This makes the distinction between storage and context visible. Storage stays detailed and durable. Active context stays small and tied to the current question.”

## Slide 8: Result and limits

**Time: 7:25–8:00**

“The research paper provided the architectural starting point. `agent-mem` implements the verifiable local core from that direction.

It captures sources, preserves their provenance, searches locally with FTS5 and E5, and returns bounded evidence to the active context. Complete sources are loaded only through an explicit read.

`agent-mem` does not yet claim automatic semantic correction or general answer quality. Those require revisions, temporal models, and a separate evaluation.

The key property is simple: the system stores more than it injects. The source remains available while the agent receives only the part needed for the current task.”

## Technical facts for questions

- `SessionStart` and user prompts use separate context profiles of roughly 4,000 and 8,000 UTF-8 bytes. Host limits are separate.
- `token_budget` is a protocol name. The `agent-mem` context profiles measure conservatively in UTF-8 bytes.
- E5 and the optional cross-encoder are local, non-generative models.
- The host adapter passes the context wrapper. The memory service does not generate the final answer.
- `memory_get` loads the complete source explicitly and checks the source reference again.
- Local storage does not automatically mean encryption. The SQLite file and host egress need separate protection.
