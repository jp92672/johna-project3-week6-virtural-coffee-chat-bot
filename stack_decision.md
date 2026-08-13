# Stack Decision Document
**Author:** John Adams

## Primary stack: n8n

n8n was chosen as the primary orchestration layer for this project.

1. I've worked directly with n8n's workflow model, so we can move fast without allocating extensive time to learning a new framework under deadline pressure.

2. Speed to proof of concept. n8n lets us map the workflow visually and integrate tools (Pinecone, OpenAI, calendar, logging) fast enough to test the idea end to end. If we need to pivot during the project, we are rewiring a node, not rewriting code across a graph.

3. The workflow itself doesn't call what LangGraph is built for. The core loop is one pass: retrieve context, generate an answer, log the interaction, optionally book a call. Feedback capture, workflow auto-disable, and future model retraining all sound like they need reasoning loops, but they don't meet the multi-step decision and complexity requirement where LangGraph would add value. Feedback is a single logged decision. Retraining is a separate process outside the live workflow. Workflow auto-disable is a threshold check across turns, not the model re-planning mid-conversation. None of that needs an agent that loops, checks its own work, and decides what to try next.

## Trade-off

n8n costs more in node configuration and passing data cleanly between steps, and it doesn't provide real validation or unit testing the way code would. For a five-day POC, that trade-off is worth it. Speed and integration matter more right now than long-term maintainability. Scaling decisions will need to be considered when moving from MVP → Beta → GA

## Model choice: OpenAI for generation

Generation runs on OpenAI rather than Claude. Both embeddings and generation now come from the same provider, which means one fewer API key and credential to manage during the build. OpenAI's models are capable enough for the retrieve-then-generate pattern this project needs, so this was a practical build decision, not a quality tradeoff.

## Where LangGraph could matter later

If the bot needed to reason about *what to retrieve next*, like deciding a first pass came back thin and pulling from a second knowledge base before answering, that could justify loop qualification. If the core build lands early and reliably, this is the first thing worth prototyping before the five days are up. Otherwise it moves to the post-MVP roadmap.
