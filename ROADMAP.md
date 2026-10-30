# Roadmap: Local AI Agent Ecosystem

Three Python projects, built as one stack. Milestones are ordered, not scheduled.
Each milestone ends in a demo and a "done when" check. Goal: both a portfolio for
AI-engineer roles and a system for everyday personal use.

**Local LLM → MCP → Knowledge Graph → Embeddings → RAG → A2A → Multi-Agent System**

- **MCP** gives agents capabilities (agent → tool/data).
- **A2A** lets agents collaborate (agent → agent).

Wherever a specific library or model is named below, it is a starting pick, not a
fixed decision.

---

## Shared foundation

### M0: Environment
- Install Git, uv, Ollama and Docker Desktop (for Neo4j).
- Pull `qwen3:8b`, `qwen3:14b` and `llama3.1:8b`.
  Hardware: RTX 5060 Ti with 16 GB VRAM and 32 GB RAM, so 7–14B models at Q4 run
  fully on the GPU.
- Create three repos under `Projects/`: `agentos`, `domaingraph` and `agentmesh`.
  Each gets uv, ruff, pytest, pre-commit and GitHub Actions CI.
- **Demo:** `uv run pytest` passes in each repo, and `ollama run qwen3:8b` answers
  on the GPU.

**Rules for every repo**
- Tool inputs and outputs are Pydantic models.
- Every run writes a JSONL trace.
- Every milestone that makes a claim adds a benchmark result.

---

## Project 1: AgentOS (`agentos`)

**Question it answers:** How reliably can a local open-weight model plan and carry
out multi-step tasks, and which parts of the system make the difference?

### A1: Basic agent loop
- **Build:** An abstraction over the model (Ollama first, easy to swap later), a
  simple observe → think → call tool loop, and a tool registry that turns Pydantic
  models into JSON tool schemas. Start with 3 built-in tools: read file, write file,
  calculator.
- **Demo:** "Read notes.txt, count the words, and write the count to result.txt."
- **Done when:** A malformed tool call is caught and returned to the model as an
  error message, not a crash.

### A2: MCP client
- **Build:** Connect to MCP servers over stdio using the official `mcp` SDK. List
  their tools and merge them into the registry. Server configuration lives in
  `agentos.toml`.
- **Demo:** The agent uses the reference filesystem MCP server and one small MCP
  server you write yourself.
- **Done when:** Adding a new MCP server takes a config edit only, no code change.

### A3: Benchmark harness (build this early)
- **Build:** 30–50 tasks in YAML (file operations, multi-step reasoning, tool
  chaining), each with an automatic checker. A runner produces a table of success
  rate, number of steps and tokens per model.
- **Demo:** `agentos bench --models qwen3:8b,llama3.1:8b` prints the table.
- **Done when:** The baseline results for the plain loop are recorded in the README.

### A4: Planner + task graph
- **Build:** The planner breaks a task into a graph of steps with dependencies, and
  the executor runs it. When a step fails, the plan is revised.
- **Demo:** A 5+ step task, with the plan and execution trace shown.
- **Done when:** The benchmark shows the planner's effect compared with A3's plain
  loop. It is kept only where it helps.

### A5: Verification + retries
- **Build:** Check tool results against their schema and against per-step
  conditions. Add a reflection step that asks "did this step achieve its goal?",
  with retries capped by a budget.
- **Done when:** The benchmark shows what each part adds: loop only, then + planner,
  then + verification, then + retries. This table is the headline result of
  Project 1.

### A6: Memory
- **Build:** Short-term memory as a rolling summary of the conversation. Long-term
  memory in SQLite, with facts plus past task outcomes and a simple way to look them
  up. (DomainGraph replaces this in Project 2.)
- **Demo:** A task in session 2 reuses a fact learned in session 1.

### A7: Sandboxing + permissions
- **Build:** Shell and code run in a subprocess with a working-directory jail, a
  timeout and a list of allowed commands. Each tool gets a permission level (read,
  write, dangerous), and dangerous ones require a human "yes".
- **Done when:** A test suite shows common escape attempts being blocked.

### A8: Runtime extras (optional, only if you'll use them)
- Model routing (small model for simple steps, larger model for planning),
  background tasks, scheduling, and an event-log viewer.

**Portfolio output:** a README with the architecture diagram, the ablation table
from A5 and a 2-minute demo GIF.

---

## Project 2: DomainGraph (`domaingraph`)

**Question it answers:** How can an agent keep structured, domain-aware, multimodal
long-term knowledge, and does a graph beat vector-only search?

### D1: Ingest pipeline
- **Build:** Video → audio (ffmpeg) → faster-whisper on the GPU → transcript split
  into chunks with timestamps kept. Also accept plain text, Markdown and PDF.
- **Demo:** A 1-hour lecture produces chunks with timestamps.

### D2: Knowledge extraction
- **Build:** The local LLM extracts concepts, relations and facts as Pydantic
  structured output, each with a confidence score and a pointer back to its source
  and timestamp. Duplicate concepts are merged (normalized names plus embedding
  similarity).
- **Done when:** 2–3 lectures are hand-labeled as a test set, and extraction
  precision and recall are measured.

### D3: Graph store
- **Build:** Neo4j 5 in Docker with a defined schema: `Domain`, `Concept`, `Source`
  and `Fact` nodes, plus `RELATED_TO`, `PART_OF` and `MENTIONED_IN` edges.
  Embeddings are stored in Neo4j's own vector index (starting model: bge-small or
  bge-m3). Loading the same data twice doesn't duplicate anything.
- **Demo:** A Cypher query walks from a concept to its sources with timestamps.

### D4: Hybrid search + evaluation
- **Build:** Three ways to search: vector-only, graph-only (expanding from matched
  concepts) and hybrid (combined and re-ranked).
- **Done when:** A query set of about 100 items gives recall@k and MRR for all
  three. This table is the headline result of Project 2.

### D5: MCP server
- **Build:** An MCP server (FastMCP) with the tools `search`, `get_concept`,
  `add_fact`, `related_concepts` and `ingest_source`.
- **Demo:** AgentOS answers "What did lecture 7 say about Raft?" and cites the
  timestamp.
- **Done when:** AgentOS's long-term memory (from A6) can use DomainGraph instead of
  SQLite.

### D6: Obsidian export
- **Build:** A one-way export from the graph to a vault: one note per concept with
  front matter (domain, type, confidence), `[[links]]` to related concepts, and
  sources with timestamps. Re-running it updates notes in place.
- **Demo:** A screenshot of the Obsidian graph view after ingesting 10 lectures.

### D7: ML — fine-tuned embedding model
- **Build:** Generate training pairs from the graph (concept ↔ source passage,
  related concept pairs) and fine-tune with sentence-transformers.
- **Done when:** On the same D4 test set, the fine-tuned model is compared with the
  base model, including a check that it didn't get worse on general search.

### D8: ML — domain classifier (optional)
- Compare a zero-shot LLM, a fine-tuned small encoder and TF-IDF + logistic
  regression on accuracy and speed. Keep it only if the comparison turns out to be
  interesting.

**Portfolio output:** the extraction scores, the search comparison table, the
before/after table for the fine-tuned model and the Obsidian screenshot.

---

## Project 3: AgentMesh (`agentmesh`)

**Question it answers:** When do specialized agents working together over A2A beat a
single agent, and what does it cost?

### M1: Single A2A agent
- **Build:** One agent as an A2A server using `a2a-sdk`, internally running an
  AgentOS instance. It publishes an Agent Card listing its skills, and supports the
  task lifecycle: submitted → working → completed or failed.
- **Demo:** A plain A2A client finds the agent and gives it a task.

### M2: Specialist agents
- **Build:** Research (web search and fetching), Analyst (technical analysis),
  Verifier (checks claims against sources) and Knowledge (the only one allowed to
  write to DomainGraph, through MCP). Each runs as its own process with its own
  model and tool settings.

### M3: Orchestrator
- **Build:** Finds agents by reading their Agent Cards, then splits a task, hands out
  the pieces, collects the results and combines them. Agents are chosen by what
  their cards say they can do, not hard-coded.
- **Demo:** "Research local LLM inference and update my knowledge base" produces a
  trace across all 4 agents.

### M4: Async + reliability
- **Build:** Streaming progress updates and push notifications, timeouts, retries,
  fallback to another agent, and cancellation.
- **Done when:** Killing an agent mid-task leads to recovery or clean failure, and a
  test proves it.

### M5: Security
- **Build:** Token-based login between agents, with limits on which agent may call
  which skill. The Knowledge agent refuses writes that don't come from the Verifier.

### M6: Single vs. multi-agent evaluation
- Run the same task set with one AgentOS agent and with the whole system. Compare
  quality, time, tokens and failure modes.
- **Done when:** A write-up says honestly where the multi-agent setup helps and where
  it doesn't. This is the headline result of Project 3.

---

## Showcase (spans all three repos)

### S1: End-to-end demo
- "Analyze this 2-hour lecture, update my knowledge base, find the concepts I'm
  missing and research them."
- Video agent → DomainGraph → finding gaps → Research and Verifier agents →
  Knowledge agent → Obsidian updates.
- **Output:** A recorded run, a trace visualization and a blog post linking the
  headline results from A5, D4, D7 and M6.

---

## Build order and dependencies

```text
M0 → A1 → A2 → A3 → A4 → A5 → A6 → A7
                         │
                         └──► D1 → D2 → D3 → D4 → D5 → D6 → D7
                                                 │
                                                 └──► M1 → M2 → M3 → M4 → M5 → M6 → S1
```

- Once A3 exists, A4–A7 can run alongside D1–D3, because the ingest pipeline doesn't
  need the agent.
- A8 and D8 are optional.
- From D5 onward, point your own notes and lectures at the system and use it every
  day. What you run into will tell you better than this plan which optional
  milestones are worth doing.
