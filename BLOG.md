# Building a local AI agent ecosystem: what actually worked

A personal project building three things from scratch on a single consumer GPU — an
agent runtime, a knowledge graph, and a multi-agent system — and measuring each one
honestly. This is the summary of the results. The numbers are in the linked write-ups;
this post is about what they mean.

**Stack:** qwen3:8b and qwen3:14b on Ollama, Neo4j, an RTX 5060 Ti (16 GB VRAM). All
local, no cloud APIs.

---

## The single-agent baseline (A5)

Before building anything multi-agent, I wanted to know what a well-built single agent
actually gets you. A5 ran an ablation across 38 tasks, 2 repeats, 3 models, and 4 agent
variants: a plain loop, then plus planner, then plus verification, then plus retries.

The result was model-specific, and the specifics matter more than the headline:

| Model | plain | + verify | + retry | Best config |
|---|---|---|---|---|
| qwen3:8b | 82% | **91%** | 88% | planner + verify |
| qwen3:14b | 84% | 84% | 86% | planner alone |
| llama3.1:8b | 20% | 13% | **29%** | planner + verify + retry |

**qwen3:8b**: verification gained 9 percentage points. The reason is specific: qwen3:8b
occasionally passes a filename as content (`word_frequency(text="essay.txt")` instead
of reading the file). A deterministic check catches this in one line of code and sends
the answer back. The planner alone can't recover from it; verification can. No tasks
were lost.

**qwen3:14b**: verification costs without helping. It already makes few of the mistakes
the deterministic checks catch. Adding verification burned +30-40% more tokens and
+55-65% more time for no accuracy gain. The right configuration for this model is the
planner alone.

**llama3.1:8b**: verification alone hurt badly (20% → 13%). Without retries, its most
common failure — writing a JSON description of a tool call instead of making the actual
call — gets sent back to the replanner, whose new steps fail the same way. With retries,
the rejection message ("call the tool through the tool interface") fed back to the model
causes it to actually make the call. Net: 29%, its best result, at 7x the tokens of the
plain loop. Still not reliable enough to use for anything serious.

**What it means for the rest of the project:** qwen3:8b with verification, qwen3:14b
with the planner, and a short list of deterministic checks. Reflection (asking the model
"did this step achieve its goal?") helped llama but added noise for the qwen3 models.

Full results: [`agentos/benchmarks/results/a5-ablation.md`](agentos/benchmarks/results/a5-ablation.md)

---

## Structured knowledge: does a graph beat vector? (D4, D7)

DomainGraph holds 10 lectures of MIT 6.006 algorithms as a Neo4j graph: 600 concepts,
3,293 facts, 2,214 relations, with bge-m3 embeddings on both passages and concepts. D4
ran a search benchmark — 102 hand-labeled queries across three search modes — to check
whether the graph structure actually helps.

**D4: vector wins, the graph helps only at the margins.**

| Mode | MRR@10 (test, 50 queries) |
|---|---|
| BM25 (baseline) | 0.596 |
| **Vector** | **0.653** |
| Hybrid (vector + graph re-rank) | 0.618 |
| Graph-only | 0.394 |

Graph-only search is significantly worse than vector (−0.26 MRR, 95% CI excludes zero).
The reason: the graph is good at finding the *topic* but not the *passage*. "AVL tree"
is mentioned in 21 of Lecture 6's 43 passages; a question about the AVL property needs
one or two of them. Concept weights rank all 21 almost identically. Vector, comparing
the question embedding against each passage embedding, finds the right one.

Hybrid — using the graph to re-rank vector's top 50 — picks up R@10 (+3.5 points on
all queries) and helps on procedure, definition and relation questions (+0.05 to +0.10
MRR). On the held-out test half, the net MRR effect is −0.035 with an interval that
spans zero. The graph adds signal, but on rank metrics it isn't measurably better than
vector alone.

One more finding worth noting: BM25 trails vector by only 0.03-0.06 MRR. Lecture
questions use the lecture's vocabulary. The graph doesn't change that.

**D7: fine-tuning the embedding model didn't help either.**

Having found that the search bottleneck was passages-within-concepts rather than
concept matching, I tried fine-tuning bge-m3 on 4,203 pairs generated from the graph
(fact → passage, concept → passages, related concept pairs). On a held-out dev lecture,
MRR improved by +0.07. On the D4 test queries: −0.018 (within noise).

The model learned what the pairs teach — matching a declarative fact statement to its
passage. That is not the same task as answering a student's question. On the D4 queries
that deliberately avoid the lecture's vocabulary (paraphrases), the tuned model got
worse in every variant. A follow-up with question-generated training data (D7b) showed
the same pattern: dev gain doesn't predict test gain when the training questions are
closer to the passage wording than the test queries are.

General search (NanoBEIR) was unchanged (mean nDCG@10: 0.604 → 0.605), so the
fine-tuning didn't degrade the model outside the domain. But it didn't help on the
actual task. Base bge-m3 stayed as the search model.

Full results: [`domaingraph/benchmarks/search/results/d4-search.md`](domaingraph/benchmarks/search/results/d4-search.md),
[`domaingraph/benchmarks/finetune/results/d7-finetune.md`](domaingraph/benchmarks/finetune/results/d7-finetune.md)

---

## Single agent vs. the whole fleet (M6)

M6 ran 16 tasks, 3 repeats each, on two systems: a single AgentOS agent with every
specialist's tools, and the full fleet (orchestrator + Research, Analyst, Verifier,
Knowledge agents over A2A). The fleet was on the same hardware, same models, same data.

**Summary of the final matched-trace-root rerun (`c1e57ad`, 96 runs):**

| | single agent | fleet |
|---|---|---|
| Runs passed | 34/48 (71%) | **48/48 (100%)** |
| Composite tasks (two skills chained) | 3/9 | **9/9** |
| Citations from pages never fetched | 1/12 | 0/20 |
| Shipped unverified numbers/quotes | none | none |
| CUDA errors | 0 | 0 |
| Mean tokens per run | ~9,500 | ~14,600 |
| Median time per run | ~58 s | ~41 s |

**Where the fleet wins: code-enforced rules.**

The single agent cited pages it never fetched in 12-14 of its citing answers in early
runs — every single citing answer. Adding a citation guard in code (Research refuses an
answer that cites a URL no `fetch_url` returned) eliminated this on the fleet. Adding
the same guard to the single agent reduced its violation rate to 1/12, the known
`http://127.0.0.1:11434` artifact (the Ollama API, which isn't a fetchable page).

The same pattern holds for verification: the fleet's Verifier fetches each source and
checks verbatim quotes before signing off. The single agent skipped this in roughly
half its research answers before the code-level check was added.

**Where the fleet loses: chained tasks.**

A task that chains two skills — research a value, then look it up in the knowledge base
— passed 3/9 on the single agent and 9/9 on the fleet in the final rerun. But this was
after substantial tuning on these 16 tasks. Every step in a fleet run sees only the
planner's rewrite of its piece of the task, not the original. Needed inputs went
missing, one question got inverted, and the planner added a knowledge-base write nobody
asked for. The fixes (passing the original task to every step, allowing a `CANNOT:
<reason>` early exit, re-planning on a missing input) were written after reading these
specific failure traces. The 10-run composite evals, before those fixes, put the fleet
at 90-95% on the same tasks — a more honest estimate for new chained tasks.

**The cost:** ~1.5x the tokens and slightly faster wall time (the fleet parallelises
some steps). Roughly 3x tokens over the original M6 baseline, once the citation guards
were added to both.

**What changed during M6's development:**

PRs #8 through #40 closed gaps the eval surfaced. The headline changes: the citation
guard and its compaction-safe reconstruction (PRs #23, #28-#34), the numbers and quotes
checks in code (#24), configurable cancel grace and planner-token accounting (#38), and
an Ollama context-checkpoint crash workaround ([ollama#17534](https://github.com/ollama/ollama/issues/17534))
that eliminated the one class of mid-run crash the hardware produced (#38).

Full write-up: [`agentmesh/benchmarks/results/m6-single-vs-multi.md`](agentmesh/benchmarks/results/m6-single-vs-multi.md)

---

## The end-to-end demo (S1)

With all three systems running, S1 ran the full pipeline: given the 10-lecture knowledge
graph, find the concepts covered only by lecture notes with no web sources, research
those gaps, verify the claims, and write the supported facts back through the M5
security chain.

The task was a single natural-language prompt to the orchestrator. No S1-specific code
was written. The planner produced a four-step plan and the fleet executed it:

1. **Knowledge agent** queried the graph: AVL tree–related concepts (balance factor,
   rotation, rebalancing, height) had no web-sourced facts. Confirmed gaps.
2. **Research agent** fetched Wikipedia and GeeksForGeeks and produced four claims with
   verbatim quotes and source URLs.
3. **Verifier agent** checked each claim against its source. All four: SUPPORTED.
4. **Knowledge agent** called `add_fact` four times. All four: `ok`.

Four new facts now live in DomainGraph (IDs 323–326), gated through the M5
Verifier-attestation write check.

**What running S1 found.** The demo surfaced two bugs in the M5 auth layer that the M6
benchmark's uniform verdict format had never hit. The live Verifier wraps source URLs as
`[<https://...>]` (angle brackets); `_URL`'s `>` stop-character extracted nothing from
that form. The orchestrator numbers its claims and the Verifier re-numbers its output,
producing `1. 1. SUPPORTED`; `_SUPPORTED` required exactly one numeric prefix. Both
were one-line fixes (PR #40). They only surface in an actual end-to-end live run — the
M6 benchmark format is too uniform to hit them, and unit tests on the regex alone
wouldn't have caught the Verifier's natural output style. Running S1 was the right test.

---

## What I learned

**The honest results were the useful ones.** D4 and D7 both came back negative. The
graph doesn't measurably beat vector on these queries, and fine-tuning the embedding
model doesn't help (and hurts paraphrases). Reporting those findings accurately is more
useful than a cherry-picked comparison. They also point directly at what would actually
move the numbers: training questions written at the same distance from the passage as
the test queries, or simply more data.

**Code beats prompts for anything you care about.** Every reliability improvement in
AgentMesh that stuck was code: a deterministic check, a guard that strips or flags
rather than hopes the model won't do it again, a test that reproduces the failure.
Prompt-only fixes regressed within one eval cycle. The citation guard (Research refuses
to answer until it's fetched every URL it cites) is more reliable than any instruction
not to cite unfetched pages.

**Unit tests find most bugs; live runs find the rest.** 289 tests pass before S1. Two
of them were added after S1 found the auth.py parse gaps. End-to-end runs with a real
model producing natural output are not replaceable by unit tests against a fixed format.

**Multi-agent costs are real.** The fleet is 1.5-3x the tokens of the single agent on
these tasks and adds latency for short tasks (coordination overhead). The benefit is
specific: code-enforced specialist rules on single-skill tasks. For a task that doesn't
need them, a single well-configured qwen3:14b agent is faster, cheaper, and barely
less accurate.

---

*Code: [agentos](https://github.com/yqx1412/agentos) · [domaingraph](https://github.com/yqx1412/domaingraph) · [agentmesh](https://github.com/yqx1412/agentmesh)*
