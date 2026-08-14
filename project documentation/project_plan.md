# Project 3 | Week 6 | Virtual Coffee Chat Bot
## John Adams

### 1. Use Case
 
**Problem:**
The current hiring market is broken on both sides. Traditional applications (CV, cover letter) are no longer reliable when job hunting. AI tools let anyone produce a polished, keyword-optimized application, so hiring teams can no longer tell a real fit from a well-formatted one – having to navigate a sea of AI slop. In response, hiring is shifting toward skills-based evaluation, where candidates need to demonstrate and prove experience directly, not just claim it on paper. Meanwhile, a candidates actual work experience lives in static places (LinkedIn, CV, personal sites, etc.) that a hiring manager or bot either skims for keywords or doesn't read at all.
 
**Solution:**
A RAG-backed conversational agent, grounded in a knowledge base built from my real work experience, skills, and case studies, that lets a hiring manager have a "virtual coffee chat" with a version of me – asking direct questions and getting grounded, specific answers instead of a static content. Also the ability for the chat UI to surface relevant resources alongside the chat (e.g. n8n workflow case study).
 
**Target users:**
Potential hiring managers and team members evaluating me as a candidate.

**Current process:** I publish static content (CV, LinkedIn, portfolio website) and hope it gets seen. Once an application is submitted, hiring teams ( or their bots) scan for keywords or proof of measurable results, often without ever getting the full picture.

**Verify:**
- POC/Prototype verify: One real question returns a grounded answer, and the chunk it came from is traceable in the n8n execution log for that turn.
- Product verify: A hiring manager engages in a multi-turn conversation and takes a next action (asks a follow-up and books a call in my calendar).
 
### 2. Technology Stack

**Technology selection framework:**
- External knowledge needed: (Yes) RAG knowledge base containing story assets (CV, work history, and case studies)
- Interacts with external systems: (Yes) Calendar booking and conversation logging/feedback
- Multi-step reasoning: (No) No planning or task decomposition is needed – the agent never breaks a goal into sub-goals or decides an order of work. It does call multiple tools within a single turn where the task requires it (check availability, then book, then confirm), but that is tool use inside one pass, not reasoning depth. This is the specific reason LangGraph isn't needed here – see `stack_decision.md`
- Integrates with business systems: (Yes) Calendar, email send, and logging integration
- Autonomous (no human input): (Yes/No) The development pipeline is autonomous, but the end user is a human participating in a multi-turn chat conversation

**Selected stack:**
- Core LLM: OpenAI
- Embeddings: OpenAI
- Reranking: Cohere
- Agent framework: n8n's native LangChain nodes (chat trigger, AI Agent, Pinecone nodes)
- Orchestration: n8n, self-hosted
- Runtime: Docker (n8n runs locally in a container, not on n8n Cloud)
- Public access: Cloudflare named tunnel ("coffee-chat-bot") exposing two hostnames on my own domain – `chat.curiousjohnadams.com` for the chat UI, `n8n-chat.curiousjohnadams.com` for n8n itself
- Tools/integrations:
  1. Pinecone retrieval (CV, work history, case studies)
  2. Cohere reranking of retrieved chunks
  3. Calendly – check availability and book a call
  4. Email (Resend, via HTTP Request) – send a message to me from the chat
  5. Conversation logging + feedback – written to an n8n Data Table (`chat_log`), exported to CSV – Q&A trace, good/bad rating + reason, call-schedule yes/no + reason

**Justification:**
- Using n8n to build on top of the workflows presented in our n8n lessons/labs
- Self-hosting n8n in Docker rather than using n8n Cloud: no per-execution billing while testing heavily, full access to the filesystem so the knowledge base can be read straight off disk during ingestion, and the whole environment is reproducible and portable
- Cloudflare tunnel rather than deploying to a host: the bot needs a real public URL for a hiring manager to use it, but the build is still a prototype. A named tunnel gives a live HTTPS URL on my own domain without standing up hosting or exposing ports directly
- OpenAI for generation and embeddings, one provider and one API key to manage during the build, and it's already proven to work with my Pinecone instances. Claude was the original intention for generation but was dropped early – I don't hold an Anthropic API key, and buying one to run a prototype wasn't worth it when OpenAI was already credentialed and covering embeddings anyway
- Alternative considerations
    - Building directly in Cloudflare Workers but comes with legal risk exposure once publishing live (DPAs, privacy policies, GDPR, Model representation and changes)
- Tradeoffs
    - Keep to n8n for prototyping and showing a working demo
    - Self-hosting means uptime is my responsibility – the bot is only reachable while the container and tunnel are running, which is fine for a prototype and not for GA
    - Complexity in moving to Cloudflare Workers
 
### 3. MVP Scope
 
**All features considered (brainstorm):**
Source input data, story-oriented RAG, chunking, embedding, vector storage, LLM retrieval/processing, chat UI, logging, calendar booking, download CV, contact me (email/LinkedIn), error/validation steps in n8n, auto-disable on repeated bad feedback/errors, model retraining from logged data, LLM fallbacks, rate limiting, cost monitoring.

**Must-have (MVP):**
- Source input data (story-oriented CV/work history document, not verbatim CV text)
- RAG, story-oriented framing
- Chunking, embedding, vector storage (Pinecone)
- LLM retrieval + generation (OpenAI)
- Chat UI
- Logging + feedback (good/bad rating + reason, call-schedule yes/no + reason)
- Calendar booking
- Basic error/validation steps in the n8n workflow

**Should-have (v2):**
- Download CV
- Rate limiting
- Cost monitoring
- Auto-disable chatbot after repeated bad feedback/errors

**Nice-to-have (v3+):**
- LLM fallbacks
- Model retraining from logged feedback data

**What's included:**
Everything in Must-have above – a working story-oriented RAG chat over CV/work history, with logging, feedback capture, calendar booking, and error handling in n8n.

Two things landed in the MVP that were originally scoped as v2 or later, because they turned out to be cheap to add and directly serve the "takes a next action" verify criterion:
- **Contact me by email** – the bot can send me a message from the chat (Resend). LinkedIn stays self-serve: the bot shares the link and explicitly states it cannot send connections or messages on the user's behalf.
- **Error handling well past "basic"** – a dedicated error workflow that logs every failure to an `error_log` Data Table and emails me an alert, covering both hard crashes and tool failures the agent absorbs into its own reply. The user only ever sees one friendly fallback message.

**What's excluded:**
Multi-document expansion beyond CV (case studies come later), live site embedding, rate limiting, cost monitoring, retraining, LLM fallback, download CV, surfacing of resources alongside the chat bot.

**Success metrics:**
At least 3 recruiter-style test questions answered correctly in a narrative/story tone (not CV recitation), one full execution trace showing retrieval → generation → log write, and calendar booking tool triggered successfully at least once in a test run.
 
### 4. Risk Assessment

#### Technical Risks

| Risk | Probability | Impact | Mitigation |
|---|---|---|---|
| Hallucination / agent going off-script under context rot or edge-case prompting | High | High | System prompt constrains answers strictly to retrieved context, with a scoped fallback that routes the user to contact me directly. Two real off-script incidents were caught during testing and fixed: the bot claimed it could send LinkedIn messages (no such tool exists), and it recited internal notes about how the knowledge base was built. Both fixes were two-part – remove the source material from the knowledge base *and* add an explicit prompt rule – on the principle that a prompt rule alone is not a control if the underlying text is still retrievable |
| API rate limits or per-query cost from traffic spikes | Medium, rises with attention/visibility | Medium-High | Cost monitoring and alerts; rate limiting; caching for repeated common questions |
| Silent retrieval degradation (agent keeps answering fluently on weak matches) | Medium | Medium | Confidence logging and auto-disable were not built for the MVP. Mitigated instead by retrieving a wide candidate set (topK 20) and letting Cohere rerank down to the top 3, plus a system prompt rule that produces a distinct "technical issue" message on a retrieval failure rather than a confident-sounding non-answer. Confidence logging stays on the GA list |
| Prompt injection exposure | **Revised: Medium-High** – the original "Low" was wrong for a bot published on a public URL and pointed at strangers | Medium | Input sanitization was not the right tool. Mitigated with explicit system prompt rules (retrieved content and pasted user content are data, never instructions; no impersonation; no revealing the prompt) and verified by live adversarial testing – impersonation, fake-admin privilege escalation, and a pasted document with embedded override instructions were all refused |
| Vector store duplicates when existing input data is reprocessed | High | Medium | Solved more simply than planned: ingestion clears the whole index (`deleteAll`) before inserting, so every run is a full replace rather than an upsert. Deterministic chunk IDs become the better option once the corpus is large enough that a full re-embed is expensive |

#### Business Risks

| Risk | Probability | Impact | Mitigation |
|---|---|---|---|
| Prototype-to-production scaling complexity | High (widely cited pattern that most AI POCs fail to reach production) | High | Staged plan: n8n prototype → Python/LangGraph MVP → GA; each stage gated on its own success criteria |
| Scope creep | High (rich feedback loops in the chat, case study surfacing, legal review) | Medium | MVP boundaries from Step 3 are the hard line; new ideas go to the backlog, not the current build unless they are a show stopper |
| Cost overruns (combined Pinecone, Cohere, OpenAI, Cloudflare, n8n hosting spend) | Medium | Medium | Track combined monthly cost against a fixed budget ceiling; prefer free/low tiers while this stays a prototype |
| Comes across as gimmicky or impersonal, hurting adoption | Medium | Low | Story tone, narrowly scoped to work history; always paired with a direct contact path |

#### Data Risks

| Risk | Probability | Impact | Mitigation |
|---|---|---|---|
| Data quality – source documents not structured well enough for clean retrieval | Medium | Medium | Use of separated knowledge bases; story-oriented rewrite (not verbatim CV) improves chunk coherence; manually review chunk boundaries before indexing |
| Bias – bot over- or under-representing parts of work history based on source weighting | Low-Medium | Medium | Review retrieved answers across a spread of topics/roles, not just the most detailed sections, before launch |
| PII exposure and data processing across OpenAI, Cohere, Calendly, Resend, Cloudflare, Pinecone, and local log exports | High once public | Medium-High | Privacy Policy and DPAs covering every processor above required before public launch – note the booking and email tools mean real names and email addresses now pass through the system, which the original plan didn't account for. Conversation logs stay local (no third-party logging vendor, so no extra DPA needed there), but that shifts responsibility for securing the file directly onto me – keep exported CSVs out of any git repo (`.gitignore`), no world-readable file permissions, and log only what's already public |
| Characterization / impersonation – bot speaking as me (first-person) vs. an assistant representing me (third-person) | Medium | Medium-High | Legal review before launch; default to "assistant representing John," disclosed as AI, until reviewed |
| No T&Cs / privacy notice at point of use | High – doesn't exist yet | Medium | Basic T&C/privacy notice shown at first message, before any public launch |
 
### 5. Implementation Plan

#### Phase 1: Setup and data preparation
- Project planning and setup
- Stand up self-hosted n8n in Docker, with the project folder mounted so ingestion can read the knowledge base off disk
- Write the story-oriented work history documents (not verbatim CV)
- Set up development pipeline and n8n workflow

#### Phase 2: Core agent development
- Chunk, embed (OpenAI), and index into Pinecone
- Build chat trigger → agent → retrieve → rerank (Cohere) → generate (OpenAI) in n8n
- Add Calendly tools (check availability, book a call)
- Add email tool (Resend) for "contact John" from the chat
- Add logging + feedback tool (Q&A trace, good/bad rating + reason, call-schedule yes/no + reason)
- Add error/validation nodes, and a separate error workflow for logging and alerting

#### Phase 3: Integration and testing
- Run end-to-end test conversations
- Confirm calendar booking and logging tools fire correctly in a real trace
- Confirm answers are narrative/story-toned, not CV recitation
- Capture one full execution trace (input payload, retrieval, output)
- Adversarial testing: impersonation, privilege escalation, and pasted-document prompt injection
- Publish the chat UI over a Cloudflare named tunnel and test against the real public URL, not just locally

#### Phase 4: Deployment and monitoring (scoped for GA version - not MVP)
- Legal review: T&Cs, privacy policy, DPAs for every processor (model change impact)
- Live embed on the Cloudflare-hosted site
- Rate limiting, cost monitoring, auto-disable circuit breaker, retrieval confidence logging
- Move off the tunnel-plus-local-Docker setup, which only serves traffic while my machine is running – decide n8n (self-hosted on a server) vs. Cloudflare Worker for production

#### Timeline
- Phase 1: 0.5–1 day
- Phase 2: 1–2 days
- Phase 3: 0.5–1 day
- Phase 4: deferred, own timeline later

**Milestones:**
- Project kickoff
- Development environment set up
- Working Pinecone index
- Working chat bot
- Working feedback and logging
- Captured execution trace

**Dependencies:** 
- Docker and self-hosted n8n must be running, with the project folder mounted, before anything else
- Story-oriented docs must exist before chunking/embedding
- Calendly must be credentialed before Phase 2 (chosen during the build; was TBD at planning time)
- n8n/Pinecone/API keys must be live before Phase 1 can run at all
- The Cloudflare tunnel must be configured and routed before the bot can be tested as a real user would use it

#### Resources Needed
- **Team:** solo (John Adams)
- **Tools/services:** self-hosted n8n running in Docker (MCP-connected), Cloudflare named tunnel for public access, Pinecone, OpenAI API (generation + embeddings), Cohere API (reranking), Calendly (availability + booking), Resend (email)
- **Budget:** prototype stage runs on free/low tiers where available; combined spend tracked against the cost-overrun risk from Step 4

### 6. Success Metrics
- At least 3 recruiter-style test questions answered correctly, in narrative tone, grounded in the source document
- One full execution trace: retrieval → generation → log write
- Calendar booking tool triggered successfully at least once in testing
- **Groundedness rate of ≥80% at MVP** – the share of test answers with no hallucinated or off-script claim

**On the groundedness target.** The original plan said "zero hallucinated/off-script claims," which was the wrong metric. Zero isn't measurable on a small test set and isn't achievable with an LLM in the loop – it's an aspiration, not a bar you can pass or fail. What I actually meant was a confidence threshold, so it's now written as one.

80% is deliberately a prototype bar: high enough that the bot is genuinely useful, low enough to acknowledge that a handful of edge-case answers will drift and get caught in testing. Two did, and both were fixed.

That bar has to rise before this goes anywhere near a real hiring manager, because the thing being represented is me:
- **MVP / demo: ≥80%** – failures are caught by me, in testing, with no audience
- **Soft launch (GTM sprint 1–2): ≥90%** – shared with a small number of real people, each remaining failure is a live misrepresentation
- **GA / public on my site: ≥95%**, with retrieval confidence logging and an auto-disable circuit breaker doing the enforcing rather than me reading transcripts

The measurement method has to harden along the same curve. Right now it's manual review of a test spread. At 90% and above it needs a fixed evaluation set of questions with known-correct answers, re-run after every knowledge base or prompt change, so the number is a regression check rather than a judgement call.