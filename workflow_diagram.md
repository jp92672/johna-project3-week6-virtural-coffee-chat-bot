# Workflow Diagram Document
**Author:** John Adams

Two separate n8n workflows: document ingestion (run when source docs change) and the live chat flow (runs per conversation).

## Workflow 1: Document ingestion

1. Manual trigger
   - Run by hand when a source doc is added or updated (docs change rarely, so no automated watch trigger is needed)
2. Read file from disk
   - Read/Write Files from Disk node, pointed at the kb/ folder
3. Chunk document
4. Embed (OpenAI)
5. Store in Pinecone, tagged with `source_type` metadata (`facts` or `narrative`)

Source docs: CV, cover letter, Clifton Strengths results (tagged `facts`); story-oriented narrative docs (tagged `narrative`).

One Pinecone index, filtered by metadata at query time. No need for separate indexes.

```
┌───────────────────┐
│  Manual Trigger   │
└─────────┬─────────┘
          │
          ▼
┌───────────────────┐
│  Read File from   │
│  Disk             │
└─────────┬─────────┘
          │
          ▼
┌───────────────────┐
│  Chunk Document   │
└─────────┬─────────┘
          │
          ▼
┌───────────────────┐
│  Embed (OpenAI)   │
└─────────┬─────────┘
          │
          ▼
┌──────────────────────┐
│  Store in Pinecone   │
│  (tagged: facts /    │
│  narrative)          │
└──────────────────────┘
```

## Workflow 2: Chat flow

1. Chat trigger
   - End user opens the chat bot on the portfolio site
2. Retrieve
   - Pull relevant chunks from Pinecone (facts and/or narrative, filtered by metadata as needed)
3. Rerank
   - Cohere reranks results for relevance
4. Generate (OpenAI)
   - Answer grounded strictly in retrieved context
5. Check confidence
   - Did retrieval find something relevant?
      - No → fallback: "I can't answer that, I suggest reaching out to John directly"
      - Yes → continue
6. Respond
   - Send the answer
7. Log
   - Write the Q&A pair to the log (this is the report data)
8. Ask for feedback
   - Good/bad rating + reason
9. Offer next step
   - Ask if they want to book a call or send a message

```
┌───────────────────┐
│  Chat Trigger     │
└─────────┬─────────┘
          │
          ▼
┌───────────────────┐
│  Retrieve         │
└─────────┬─────────┘
          │
          ▼
┌───────────────────┐
│  Rerank           │
└─────────┬─────────┘
          │
          ▼
┌─────────────────────┐
│  Generate (OpenAI)  │
└─────────┬───────────┘
          │
          ▼
   ┌──────────────────┐
   │ Check confidence │
   └──┬────────────┬──┘
      │            │
     No           Yes
      │            │
      ▼            ▼
┌──────────┐  ┌───────────┐
│ Fallback │  │  Respond  │
└──────────┘  └──────┬────┘
                     │
                     ▼
               ┌───────────┐
               │    Log    │
               └─────┬─────┘
                     │
                     ▼
               ┌───────────┐
               │  Feedback │
               └─────┬─────┘
                     │
                     ▼
               ┌─────────────┐
               │  Next step  │
               └─────────────┘
```

## Deferred (not in MVP scope)

- Multi-step retrieval reasoning (deciding mid-answer to pull from a second source)
   - Reasoning will only be considered based on complexity, proven working POC, and time
- Proactively suggesting relevant info not directly asked for
- Surface a case study link or fact if relevant
- Retraining workflow from logged feedback
   - Separate workflow file
