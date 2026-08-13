# Lab | Week 5 | Day 4 | Architect the bot someone will actually pay for
## John Adams

**Note:**
This is the pre-project planning baseline I will carry forward into project 3.

### 1. Use Case
 
**Problem:**
The current hiring market is broken on both sides. Traditional applications (CV, cover letter) are no longer reliable when job hunting. AI tools let anyone produce a polished, keyword-optimized application, so hiring teams can no longer tell a real fit from a well-formatted one – having to navigate a sea of AI slop. In response, hiring is shifting toward skills-based evaluation, where candidates need to demonstrate and prove experience directly, not just claim it on paper. Meanwhile, a candidates actual work experience lives in static places (LinkedIn, CV, personal sites, etc.) that a hiring manager or bot either skims for keywords or doesn't read at all.
 
**Solution:**
A RAG-backed conversational agent, grounded in a knowledge base built from my real work experience, skills, and case studies, that lets a hiring manager have a "virtual coffee chat" with a version of me – asking direct questions and getting grounded, specific answers instead of a static content. Also the ability for the chat UI to surface relevant resources alongside the chat (e.g. n8n workflow case study).
 
**Target users:**
Potential hiring managers and team members evaluating me for a role.
 
**Verify:**
- Lab/prototype verify: One real question returns a grounded answer, and the source chunk it came from is traceable.
- Product verify: A hiring manager engages in a multi-turn conversation and takes a next action (asks a follow-up and books a call in my calendar).

**Current process:** I publish static content (CV, LinkedIn, portfolio website) and hope it gets seen. Once an application is submitted, hiring teams ( or their bots) scan for keywords or proof of measurable results, often without ever getting the full picture.
 
### 2. Technology Stack

**Technology selection framework:**
- External knowledge needed: (Yes) RAG knowledge base containing story assets (CV, work history, and case studies)
- Interacts with external systems: (Yes) Calendar booking and conversation logging/feedback
- Multi-step reasoning: (No) Single retrieve-then-generate pass will be used in the workflow
- Integrates with business systems: (Yes) Calendar and logging integration
- Autonomous (no human input): (Yes/No) The development pipeline is autonomous, but the end user is a human participating in a multi-turn chat conversation

**Selected stack:**
- Core LLM: OpenAI
- Embeddings: OpenAI
- Reranking: Cohere
- Agent framework: n8n's native LangChain nodes (chat trigger, AI Agent, Pinecone nodes)
- Orchestration: n8n
- Tools/integrations:
  1. Pinecone retrieval (CV, work history, case studies)
  2. Calendar booking (schedule a call with me)
  3. Conversation logging + feedback – written to CSV export (n8n in-memory store during test runs) – Q&A trace, good/bad rating + reason, call-schedule yes/no + reason

**Justification:**
- Using n8n to build on top of the workflows presented in our n8n lessons/labs
- OpenAI for generation and embeddings, one provider and one API key to manage during the build, and it's already proven to work with my Pinecone instances
- Alternative considerations
    - Building directly in Cloudflare Workers but comes with legal risk exposure once publishing live (DPAs, privacy policies, GDPR, Model representation and changes)
- Tradeoffs
    - Keep to n8n for prototyping and showing a working demo
    - Complexity in moving to Cloudflare Workers
 
### 3. MVP Scope
 
**All features considered (brainstorm):**
Source input data, story-oriented RAG, chunking, embedding, vector storage, LLM retrieval/processing, chat UI, logging, calendar booking, download CV, contact me (email/LinkedIn), error/validation steps in n8n, auto-disable on repeated bad feedback/errors, model retraining from logged data, LLM fallbacks, rate limiting, cost monitoring.

**Must-have (MVP):**
- Source input data (story-oriented CV/work history document, not verbatim CV text)
- RAG, story-oriented framing
- Chunking, embedding, vector storage (Pinecone)
- LLM retrieval + generation (Claude)
- Chat UI
- Logging + feedback (good/bad rating + reason, call-schedule yes/no + reason)
- Calendar booking
- Basic error/validation steps in the n8n workflow

**Should-have (v2):**
- Download CV
- Contact me/send a message (email/LinkedIn)
- Rate limiting
- Cost monitoring
- Auto-disable chatbot after repeated bad feedback/errors

**Nice-to-have (v3+):**
- LLM fallbacks
- Model retraining from logged feedback data

**What's included:**
Everything in Must-have above – a working story-oriented RAG chat over CV/work history, with logging, feedback capture, calendar booking, and basic error handling in n8n.

**What's excluded:**
Multi-document expansion beyond CV (case studies come later), live site embedding, rate limiting, cost monitoring, retraining, LLM fallback, download/contact links, surfacing of resources alongside the chat bot.

**Success metrics:**
At least 3 recruiter-style test questions answered correctly in a narrative/story tone (not CV recitation), one full execution trace showing retrieval → generation → log write, and calendar booking tool triggered successfully at least once in a test run.
 
### 4. Risk Assessment

#### Technical Risks

| Risk | Probability | Impact | Mitigation |
|---|---|---|---|
| Hallucination / agent going off-script under context rot or edge-case prompting | High | High | System prompt constrains answers strictly to retrieved context; explicit fallback script ("I'm not able to answer that directly, please reach out to John directly – he's really nice!"); test against prompt injection/jailbreak attempts before launch |
| API rate limits or per-query cost from traffic spikes | Medium, rises with attention/visibility | Medium-High | Cost monitoring and alerts; rate limiting; caching for repeated common questions |
| Silent retrieval degradation (agent keeps answering fluently on weak matches) | Medium | Medium | Log retrieval confidence per turn; repeated low-confidence turns trigger the auto-disable chatbot (workflow) |
| Prompt injection exposure | Low | Medium | Basic input sanitization |
| Vector store duplicates when existing input data is reprocessed | High | Medium | Use a deterministic chunk ID (hash of filename + chunk index) on every upsert, so reprocessing the same document overwrites existing vectors instead of adding duplicates; alternatively, delete all vectors matching the source document's metadata filter before re-upserting |

#### Business Risks

| Risk | Probability | Impact | Mitigation |
|---|---|---|---|
| Prototype-to-production scaling complexity | High (widely cited pattern that most AI POCs fail to reach production) | High | Staged plan: n8n prototype → Python/LangGraph MVP → GA; each stage gated on its own success criteria |
| Scope creep | High (rich feedback loops in the chat, case study surfacing, legal review) | Medium | MVP boundaries from Step 3 are the hard line; new ideas go to the backlog, not the current build unless they are a show stopper |
| Cost overruns (combined Pinecone, Cohere, Claude, Cloudflare, n8n hosting spend) | Medium | Medium | Track combined monthly cost against a fixed budget ceiling; prefer free/low tiers while this stays a prototype |
| Comes across as gimmicky or impersonal, hurting adoption | Medium | Low | Story tone, narrowly scoped to work history; always paired with a direct contact path |

#### Data Risks

| Risk | Probability | Impact | Mitigation |
|---|---|---|---|
| Data quality – source documents not structured well enough for clean retrieval | Medium | Medium | Use of separated knowledge bases; story-oriented rewrite (not verbatim CV) improves chunk coherence; manually review chunk boundaries before indexing |
| Bias – bot over- or under-representing parts of work history based on source weighting | Low-Medium | Medium | Review retrieved answers across a spread of topics/roles, not just the most detailed sections, before launch |
| PII exposure and data processing across Claude, Cloudflare, Pinecone, and local CSV log exports | High once public | Medium-High | Privacy Policy and DPA covering Claude/Cloudflare/Pinecone required before public launch; CSV logs stay local (no third-party logging vendor, so no extra DPA needed there), but that shifts responsibility for securing the file directly onto me – keep exported CSVs out of any git repo (`.gitignore`), no world-readable file permissions, and log only what's already public |
| Characterization / impersonation – bot speaking as me (first-person) vs. an assistant representing me (third-person) | Medium | Medium-High | Legal review before launch; default to "assistant representing John," disclosed as AI, until reviewed |
| No T&Cs / privacy notice at point of use | High – doesn't exist yet | Medium | Basic T&C/privacy notice shown at first message, before any public launch |
 
### 5. Implementation Plan

#### Phase 1: Setup and data preparation
- Project planning and setup
- Write the story-oriented work history documents (not verbatim CV)
- Set up development pipeline and n8n workflow

#### Phase 2: Core agent development
- Chunk, embed (OpenAI), and index into Pinecone
- Build chat trigger → agent → retrieve → rerank (Cohere) → generate (Claude) in n8n
- Add calendar booking tool (TBD)
- Add logging + feedback tool (Q&A trace, good/bad rating + reason, call-schedule yes/no + reason)
- Add error/validation nodes

#### Phase 3: Integration and testing
- Run end-to-end test conversations
- Confirm calendar booking and logging tools fire correctly in a real trace
- Confirm answers are narrative/story-toned, not CV recitation
- Capture one full execution trace (input payload, retrieval, output)

#### Phase 4: Deployment and monitoring (scoped for GA version - not MVP)
- Legal review: T&Cs, privacy policy, DPA, Anthropic ToS (model change impact)
- Live embed on the Cloudflare-hosted site
- Rate limiting, cost monitoring, auto-disable circuit breaker
- Decide n8n (self-hosted) vs. Cloudflare Worker for production

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
- Story-oriented docs must exist before chunking/embedding
- Calendar service must be chosen and credentialed before Phase 2
- n8n/Pinecone/API keys must be live before Phase 1 can run at all.

#### Resources Needed
- **Team:** solo (John Adams)
- **Tools/services:** self-hosted n8n (MCP-connected), Pinecone, Claude API, OpenAI API (embeddings), Cohere API (reranking), calendar/booking service (TBD)
- **Budget:** prototype stage runs on free/low tiers where available; combined spend tracked against the cost-overrun risk from Step 4

### 6. Success Metrics
- At least 3 recruiter-style test questions answered correctly, in narrative tone, grounded in the source document
- One full execution trace: retrieval → generation → log write
- Calendar booking tool triggered successfully at least once in testing
- Zero hallucinated/off-script claims across all test questions