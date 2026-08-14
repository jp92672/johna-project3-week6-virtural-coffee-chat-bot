# Workflow Diagram Document — V2
**Author:** John Adams

The original plan sketched two workflows and a linear chat sequence with a single confidence check. The actual build diverged in three ways:

1. **Chat became a tool-calling agent, not a linear pipeline.** The original plan had retrieve → rerank → generate as three fixed steps. The build uses a LangChain agent that decides when to call retrieval (and four other tools) rather than a fixed sequence — this is what let calendar booking and email get added as actions instead of a scripted "next step" prompt.
2. **A third workflow was added.** Centralized error handling (log + email alert on any failure) wasn't in the original two-workflow plan. It became necessary once the bot had real side effects (bookings, emails) worth knowing about if they silently broke.
3. **The confidence check became grounding rules in the system prompt, not a workflow node.** "Did retrieval find something relevant" turned out to be a judgment call the LLM is better positioned to make (with an exact required fallback line) than a threshold check n8n could evaluate structurally.

This file documents what's actually running today. Node names below match the exported workflow JSON in `workflows/`.

---

## Workflow 1: Ingestion (`Virtural Coffee Chat Bot (RAG Flow)`)

Same shape as originally planned, with one addition: duplicate-prevention wasn't designed in V1 and was added once re-ingestion after KB edits became a repeated need.

1. **Manual Trigger** — run manually whenever `kb/` content changes.
2. **Read Files from KBs** — reads `/data/project/kb/**/*.md`.
3. **Clear Pinecone Index** — wipes the namespace before every run via HTTP Request, so re-ingestion never accumulates duplicate vectors. Runs in parallel with the file read (via a Merge node) so the file payload isn't lost.
4. **Route KBs for Chunking** — a Switch node splitting facts vs. narrative content into two branches with different chunking:
   - **Facts branch**: chunk size 500, overlap 100 (list-like, boundary-sensitive content)
   - **Narrative branch**: chunk size 1000, overlap 100 (flowing prose)
5. **OpenAI Embeddings** (`text-embedding-3-small`, dimension 512) on each branch.
6. **Store in Pinecone** — each branch's Document Loader tags chunks with `source_type: facts` or `source_type: narrative` metadata, matching the original plan's "one index, filtered by metadata" design.

```
┌────────────────┐
│ Manual Trigger │
└───────┬────────┘
        │
        ▼
┌──────────────────────┐        ┌────────────────────────┐
│ Read Files from KBs  ├───────►│ Clear Pinecone Index   │
└───────────┬──────────┘        │ (parallel, side effect)│
            │                   └────────────────────────┘
            ▼
┌──────────────────────────┐
│ Route KBs for Chunking   │
│ (Switch: facts/narrative)│
└─────┬─────────────┬──────┘
      │             │
      ▼             ▼
┌───────────┐  ┌─────────────┐
│ Facts:    │  │ Narrative:  │
│ 500/100   │  │ 1000/100    │
└─────┬─────┘  └──────┬──────┘
      ▼               ▼
┌───────────┐  ┌─────────────┐
│ Embed     │  │ Embed       │
└─────┬─────┘  └──────┬──────┘
      ▼               ▼
┌───────────┐  ┌─────────────┐
│ Store     │  │ Store       │
│ (facts)   │  │ (narrative) │
└───────────┘  └─────────────┘
```

---

## Workflow 2: Chat Flow (`Virtural Coffee Chat Bot (Chat Flow)`)

The original linear pipeline (retrieve → rerank → generate → confidence check → respond → log → feedback → next step) became a tool-calling agent. Retrieval, reranking, calendar, and email are all agent tools called on demand rather than fixed pipeline stages. Confidence checking and next-step offers live in the system prompt's rules, not as workflow nodes.

1. **Chat Interface** — public chat trigger, streaming response, Basic Auth (POC-phase access control).
2. **RAG Agent** — the core. System prompt governs grounding, scope, manipulation resistance, contact/tool rules, technical-failure messaging, and internal-info non-disclosure (19 rules total; see the live system message for full text). Tools available to it:
   - **Retrieve from Pinecone** (as tool) — retrieval + Cohere rerank in one step, matching V1's "retrieve → rerank" but as an on-demand tool call instead of a fixed stage
   - **Check Calendly Availability**
   - **Book Calendly Call**
   - **Email Message Send**
3. **Chat Bot Conversation Logging** — writes every turn (question, answer, thumbs up/down rating + reason) to `chat_log`. This *is* the "structured report" deliverable. Runs as a parallel branch off the agent's success output so it never blocks the streamed response.
4. **Listen for Silent Errors** — checks the agent's own successful output for a fixed failure phrase (the case where a tool call fails but the agent absorbs it and reports the failure in normal text rather than crashing).
5. **Compile Error Data → Send Data to Error Workflow** — three sources converge here: Logging's error output, RAG Agent's error output (real crashes), and Listen for Silent Errors. All route to the centralized alert system (Workflow 3) without ever crashing the chat for the end user.

```
┌────────────────┐
│ Chat Interface │
└───────┬────────┘
        │
        ▼
┌───────────────────────────────────────┐
│ RAG Agent                             │
│ (tools: retrieve/rerank, availability,│
│  book, email)                         │
└──┬─────────────────────────────┬──────┘
   │ success                     │ error (crash)
   ▼                             ▼
┌─────────────────┐          ┌────────────────┐
│ Conversation     │         │ Compile Error  │
│ Logging          ├──error─►│ Data           │
└────────┬─────────┘         └────────┬───────┘
         │ success                    │
         ▼                            ▼
┌───────────────────┐        ┌─────────────────────┐
│ Listen for Silent ├───────►│ Send Data to Error  │
│ Errors            │  match │ Workflow            │
└───────────────────┘        └─────────────────────┘
```

---

## Workflow 3: Error Logging + Alert (`Virtural Coffee Chat Bot (Workflow Error Logging + Alert)`) — new, not in V1

Two entry points, both feeding the same two actions:

1. **Error Trigger** — n8n's native mechanism, fires automatically if any node in Chat Flow crashes without being caught locally.
2. **When Executed by Another Workflow** — manual entry point, called directly by Chat Flow's `Send Data to Error Workflow` node for errors caught locally (never crashes the chat).

Both feed:
- **Error Notification** — email to John via Resend.
- **Error Logging** — writes to `error_log` Data Table.

```
┌───────────────┐     ┌──────────────────────────────┐
│ Error Trigger │     │ When Executed by Another WF  │
└───────┬───────┘     └──────────────┬───────────────┘
        │                            │
        └─────────────┬──────────────┘
                      ▼
          ┌────────────────────────┐
          │   fan out to both:     │
          ├─────────────┬──────────┤
          ▼             ▼
   ┌──────────────┐  ┌───────────────┐
   │ Error        │  │ Error         │
   │ Notification │  │ Logging       │
   │ (email)      │  │ (data table)  │
   └──────────────┘  └───────────────┘
```

---

## Deferred, still not in scope

- Multi-step retrieval reasoning (deciding mid-answer to pull from a second source)
- Surfacing resources, stats, and related info in or near the chat
- Retraining workflow from logged feedback (would be a separate workflow)
- A user-facing fallback when RAG Agent crashes mid-stream, delivered from n8n's side — currently handled client-side in `chat-ui/index.html` instead.
