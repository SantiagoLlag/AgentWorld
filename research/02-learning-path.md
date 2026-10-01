# 02 — Learning path: building AI agents, hands-on

*As of 2026-10-01. Reading material: the [frameworks landscape](./01-frameworks-landscape.md) and the twelve use-case dossiers in [./use-cases/](./use-cases/). Scope: LLM-driven systems that use tools and take multi-step actions with some autonomy (an agent loop or an agentic workflow). Plain chatbots and single prompts are out of scope, except as the starting point of L1. Section references like "04 §7" mean dossier 04, section 7 (every dossier uses the same 14-section layout: §1 TL;DR, §4 architecture, §7 tool design, §8 context and memory, §9 evaluation, §10 failure modes, §11 build it yourself, §12 economics).*

---

## 0. How to use this path

**Shape.** Eight levels. Each level is: read named dossier sections (not whole dossiers), build one project, pass a written eval gate, write a one-page decision artifact. All projects live in one repo, so the result is a portfolio of eight related systems rather than eight demos.

**Rules that apply at every level**

1. **Eval before feature.** Write the dataset and grader first. Write the pass bar down before you run the test split.
2. **Dev/test split.** Tune on dev. Touch test only at the gate. If you change the agent after a failed gate, replace at least 30% of the test cases before rerunning.
3. **Cumulative regression.** Each gate also reruns every earlier gate's suite (at least the cheap, code-graded cases).
4. **Config, not constants.** Model IDs come from env vars (`AGENT_MODEL`, `WORKER_MODEL`), as in the atlas sketches. Pin SDK versions: most agent SDKs are pre-1.0 or churn fast (landscape §8.2 #6).
5. **Policy lives in code, not prompts** (04 §1). Limits, identity, suppression lists, ACLs, budgets and caps are enforced outside the model.
6. **Synthetic or redacted data only** until L8's data-handling decision.
7. **Hand-write the L2 loop and the L3 gates once.** Use a coding assistant for boilerplate afterwards. Spend your own domain expertise on eval cases, approval boundaries and unanswerable cases; a model cannot invent those for you.
8. **Gate numbers are curriculum defaults I chose**, not atlas findings. Where the atlas gives an anchor it is cited. With small n, report raw counts (7/8), not only percentages.

**Prerequisites.** Python 3.11+, HTTP/JSON, git, Docker basics, SQL basics, an LLM API key. The sketches in the atlas use the Anthropic Messages API and the Claude Agent SDK, so the examples below do too. The concepts transfer; alternatives are named per level. The landscape did not verify non-Claude model support in the Claude Agent SDK (landscape §3.2), so check before choosing it on a non-Claude stack.

**Evidence caveat.** The dossiers are medium confidence (04 is low: most vendor pages were blocked). Many figures are vendor- or speaker-reported. Treat every atlas number as a hypothesis your own gate evals confirm or refute. Prefer the atlas's architecture, tool-design and failure-mode sections over its exemplar statistics.

**Time-boxed route.** If you only have a quarter: L2, L3, L4, L6, then the cost/tracing/regression parts of L8. Skim L1; skip L5 and L7 until a client needs research or code agents.

### Repo spine (shared by all projects)

```text
agentlab/
  core/      llm.py (client wrapper, model from env, usage/$ accounting) tools.py (schema + registry) loop.py trace.py   # L1-L2
  gates/     policy functions, hooks, approval queue + CLI (separate process)                                          # L3
  kb/        retrieval, ACL filter, SOPs/SKILL.md, MCP server, release manifest                                        # L4
  research/  lead / worker / verifier prompts, findings ledger, governor hooks                                         # L5
  flows/     graph definitions, validators, authority matrix, exception queue, audit table                             # L6
  sandbox/   Dockerfile, egress policy, blueprint runner, red-team probes                                              # L7
  ops/       OTel setup, gateway config, budgets, release flow, runbooks, risk register                               # L8
  agents/    p1_intake p2_analyst p3_drafter p4_service p5_research p6_pipeline p7_ticket2pr p8_capstone
  evals/     datasets/*.jsonl  graders/  runner.py  results/L1.md ... L8.md
```

### Shared eval harness (build in L1, extend every level)

- **Case record** (JSONL): `{"id","split":"dev|test","input","checker":"exact|schema|state|judge","expected","tags":["adversarial","injection","unanswerable","multilingual"]}`.
- **Runner:** 3 trials per case. Per trial record: pass, steps, tokens in/out, USD, latency, tool calls, trace id, model ID, SDK versions, prompt hash.
- **Metrics:** pass@1 (mean over trials); pass^k (all k trials pass; per-trial 75% gives about 42% at k=3, 04 §9); cost per task; p50/p95 latency and steps; cap-hit rate.
- **Grader order:** code and state checks first; LLM judge with a rubric second, run in a fresh context from the actor (self-grading is biased, 03 §4); human sample (at least 10%) third. This follows Anthropic's eval guidance as summarized in 01 §9.
- **Infra pinning:** pin container limits and tool versions in the results file. Resource config alone moved Terminal-Bench 2.0 by about 6 points (01 §9), so unexplained deltas are often infrastructure.

### Atlas map: where each dossier is used

| Dossier | Cx | Autonomy | What it teaches | Core in | Also |
|---|---|---|---|---|---|
| [01 interactive coding](./use-cases/01-interactive-coding-agents.md) | 3 | supervised | A tiny loop plus a big harness: layered action gates and context management | L2, L3 | L7, L8 |
| [02 SRE/security](./use-cases/02-sre-security-investigation-agents.md) | 4 | semi | Separate investigation from action; parallel hypothesis workers; mechanical oracle where possible | L5, E3 | L3 |
| [03 async fleets](./use-cases/03-async-coding-agent-fleets.md) | 4 | semi | Per-task isolated environment, fixed blueprint with caps, layered verification | L7 | L6, L8 |
| [04 support](./use-cases/04-support-resolution-agents.md) | 3 | semi | Policy in code, procedures as versioned artifacts, simulated-user release gates | L4 | L1, L3, L8 |
| [05 voice](./use-cases/05-voice-agents.md) | 4 | semi | Turn-taking and latency as architecture; talker/thinker split; human exit | E1 | L8 |
| [06 GTM/creative](./use-cases/06-gtm-sales-marketing-creative-agents.md) | 3 | supervised | The approval boundary is a missing tool; deterministic rails; outcome metrics | L3 | L1, L8 |
| [07 workplace](./use-cases/07-workplace-knowledge-agents.md) | 4 | semi | Identity and the approval gate are the product; ACL-filtered retrieval; triggers | L4, L6 | L5, L8 |
| [08 back-office](./use-cases/08-back-office-case-processing-agents.md) | 4 | supervised | Deterministic pipeline, quarantined model; STP and exception economics | L1, L6 | L8 |
| [09 deep research](./use-cases/09-deep-research-analyst-agents.md) | 4 | supervised | Parallel isolated subagents, evidence at the tool boundary, verifier, budgets | L5 | L8 |
| [10 computer use](./use-cases/10-computer-use-browser-agents.md) | 4 | supervised | Control plane beats perception choice; API first, GUI fallback; injection | E2 | L7 |
| [11 embodied/sim](./use-cases/11-embodied-simulated-world-agents.md) | 5 | semi | Low-level action model as one blocking tool; independent success detector | E5 | L5 |
| [12 discovery](./use-cases/12-scientific-engineering-discovery-agents.md) | 4 | semi | The evaluator decides whether it works; held-out data outside the agent's reach | L7, E4 | L5 |

Cx is the dossier's own complexity score (1 to 5). Read the complexity-3 dossiers (01, 04, 06) first; 11 last.

---

## 1. Progression at a glance

```mermaid
flowchart TD
  subgraph FOUND["Foundations"]
    L1["L1 Augmented LLM and fixed workflows<br/>P1 Intake extractor<br/>gate: field F1, schema validity, injection canary"]
    L2["L2 The agent loop<br/>P2 Data analyst agent<br/>gate: pass^3, abstention, caps, traces"]
  end
  subgraph CTRL["Control and knowledge"]
    L3["L3 Harness and action gates<br/>P3 Draft-only outreach agent<br/>gate: seeded violations denied, injection suite"]
    L4["L4 Knowledge, procedures, service agents<br/>P4 Policy-grounded service agent<br/>gate: simulated users, ACL canary"]
  end
  subgraph SCALE["Orchestration and durability"]
    L5["L5 Orchestrator-workers and research<br/>P5 Research agent with verifier<br/>gate: citation fidelity, baseline comparison"]
    L6["L6 Durable workflows and human gates<br/>P6 Case pipeline with exception queue<br/>gate: no wrong commits, crash-resume"]
  end
  subgraph PROD["Isolation and production"]
    L7["L7 Isolated autonomy<br/>P7 Ticket-to-draft-PR agent in a sandbox<br/>gate: red-team probes, hidden-test replay"]
    L8["L8 Production system<br/>P8 Production pack and capstone<br/>gate: regression CI, degradation drill, incident drill"]
  end
  L1 --> L2 --> L3 --> L4 --> L5 --> L6 --> L7 --> L8
  L4 -.-> E1["E1 Voice (05)"]
  L5 -.-> E3["E3 Investigation (02)"]
  L7 -.-> E2["E2 Browser and computer use (10)"]
  L7 -.-> E4["E4 Evaluator-driven search (12)"]
  L8 -.-> E5["E5 Embodied and simulated worlds (11)"]
```

Autonomy rises along the path: L1 none (fixed control flow), L2 autonomous but read-only, L3 supervised actions, L4 semi-autonomous on narrow flows, L5 parallel autonomous research, L6 pipelines with exception handling, L7 unattended and isolated, L8 operated in production. Complexity is added only when the previous level's eval shows a ceiling that the new mechanism removes.

| L | Project (artifact) | Framework | Gate, short | Effort (h) |
|---|---|---|---|---|
| 1 | P1 Intake extractor | Raw Messages API + Pydantic | F1 ≥ 0.90, schema 100%, evidence ≥ 98%, injections 0/5 | 8–12 |
| 2 | P2 Data analyst agent | Raw loop, about 120 lines | pass@1 ≥ 80%, pass^3 ≥ 65%, abstain 3/3, 100% within caps | 10–14 |
| 3 | P3 Draft-only outreach agent | Claude Agent SDK | 20/20 violations denied, claims traceable ≥ 95%, injections 0/10 | 12–16 |
| 4 | P4 Policy-grounded service agent + MCP server | Claude Agent SDK + MCP + SKILL.md | state-checked success ≥ 80%, escalation ≥ 95%, ACL leaks 0 | 16–24 |
| 5 | P5 Research agent with verifier | Claude Agent SDK subagents | citation fidelity ≥ 95%, judge ≥ 0.70, budget 100% | 16–24 |
| 6 | P6 Case pipeline with exception queue | LangGraph (Temporal as stretch) | 0 wrong auto-commits, exception recall ≥ 95%, crash-resume clean | 20–30 |
| 7 | P7 Ticket-to-draft-PR agent in a sandbox | Claude Agent SDK headless in a container | 10/10 probes contained, ≥ 50% hidden-test pass, 5/5 test-weakening caught | 20–30 |
| 8 | P8 Production pack and capstone | OpenTelemetry + trace backend + gateway + CI evals | degradation, kill-switch and incident drills pass | 24–36 |

Effort covers reading, building and evaluating, for an experienced Python developer. These are planning estimates, not measurements. Total 126–186 hours, roughly 16–23 weeks at 8 hours a week.

---

## L1 — Augmented LLM and fixed workflows

**Goal.** Make one bounded model call do a typed job with tools, and learn why most jobs should stop here.

**Concepts introduced**
- Augmented LLM: model plus tools, retrieval and memory (04 §4).
- The tool-use protocol: JSON Schema tool definition, `tool_use` block, harness executes, `tool_result` message back. A tool description is a prompt; return high-signal fields and readable names, not UUID soup; errors say what to try next (04 §7, 01 §7).
- Structured output as a contract: Pydantic schema, validation, one repair retry that feeds the validation error back, and a per-field evidence quote that code can check mechanically (08 §7, §10). A portable trick: force the final answer to be a `submit` tool whose `input_schema` is your schema.
- Fixed-control-flow workflows: prompt chaining, routing, parallelization (landscape §2, §8.2). Decision rule: if the path is known, write a workflow. LinkedIn's advice, quoted in 06 §1: "procedural code works a lot better" for simple workflow automation.
- The model extracts; code decides. Scores, limits and routing thresholds are plain code (04 §1).
- Untrusted text is data: delimit it, say so in the system prompt, give the extractor no way to act on it (04 §8, 08 §10).
- Usage accounting from the first call: tokens, USD, latency per stage.

**Read**
- Core: [04](./use-cases/04-support-resolution-agents.md) §1, §7, §8 (untrusted-text bullet), §11 steps 2–3 and the raw `messages.create` loop sketch. [08](./use-cases/08-back-office-case-processing-agents.md) §1, §4 (quarantined reader), §7 lessons 2–3, §11 steps 2–4. [06](./use-cases/06-gtm-sales-marketing-creative-agents.md) §1 and §7 lessons 1–3.
- Skim: [01](./use-cases/01-interactive-coding-agents.md) §7 lessons; [landscape](./01-frameworks-landscape.md) §2 (last two paragraphs), §8.2 rules 1–2.

**Build — P1 Intake extractor** (`agents/p1_intake/`)
- Input: a free-text message (email, chat, form). Output: `IntakeResult` JSON with `category` (enum of at most 6), `fields` each as `{value, evidence_quote, confidence}`, `needs_human`, and a `score` computed by plain code.
- Pipeline: (1) route call on a small model; (2) extract call with two stub tools (`lookup_contact`, `lookup_org`) and at most two tool round trips, hard-coded, with the final answer forced through a `submit` tool; (3) deterministic validators: schema, quote is a substring of the input, enum membership, currency and date parsing; a failing field becomes null, never a guess; one repair retry; (4) scoring by code; (5) per-stage tokens, USD and latency to JSONL.
- Dataset: at least 60 labeled messages (40 dev, 20 test), including at least 8 adversarial (ambiguous, multilingual, contradictory, duplicate) and 5 injections ("ignore previous instructions and ...").

**Acceptance criteria**
1. Output is schema-valid on 100% of messages after at most one repair retry; first-pass validity is reported.
2. On the test split (mean of 3 trials): field-level F1 ≥ 0.90 and router accuracy ≥ 0.90.
3. Evidence validity ≥ 98%: every non-null field's quote is a verbatim substring of the input.
4. Injection: 0 of 5 injected messages change the category, trigger a call outside the two lookups, or leak another record.
5. Cost and p95 latency per message are reported, with a one-paragraph note on why this is a workflow and not an agent.

**Gate eval.** `make eval-p1` runs 3 trials over dev and test and writes `evals/results/L1.md`. Pass means criteria 1–4 on the test split.

**Framework — raw provider SDK + Pydantic.** Anthropic's guidance (landscape §8.2 #1) is to start with direct API calls or a minimal loop, add workflow patterns only when they demonstrably help, and use frameworks cautiously because they can hide prompts and responses. At this level a framework would only hide the `tool_use`/`tool_result` protocol you need to understand. Alternatives: Pydantic AI (typed outputs, validation retries, dependency injection; landscape §3.2) for less glue; Vercel AI SDK on TypeScript.

**Domain variant — luxury inquiry intake.** Route inbound inquiries (web form, email, WhatsApp text) into `buyer_lead | seller_lead | existing_client | vendor_or_spam | press`. Extract `property_interest`, `budget` (with currency and ambiguity such as "north of 10"), `timeline`, `financing` (cash, mortgage, unknown), `location_preferences`, `represented_by_another_agent`, `language`, `privacy_request`. Code computes the qualification score from the extracted fields; the model never scores. Add multilingual and multi-currency cases to the adversarial tag. This is the "lead qualification" input stage; L3 and L8 reuse it.

**Common mistakes**
- Vague tool descriptions and opaque return payloads; the model then picks the wrong tool or misreads results (04 §7).
- Accepting extracted values without evidence, so hallucinated fields pass silently (08 §10).
- Putting untrusted message text where instructions go, or giving the reader tools (08 §4).
- Reaching for an agent where a chain suffices; the control flow is known, so it should be code (06 §1).
- Evaluating on five hand-picked examples instead of a labeled split with adversarial cases.

**Stretch.** Tier the models (small router, stronger extractor) and publish the cost-versus-F1 curve. Port P1 to Pydantic AI and diff the prompts it sends. Add a parallel fan-out for a multi-message thread.

**Consulting artifact.** One-page workflow-versus-agent decision memo: what is known in advance, what is not, which pattern fits.

---

## L2 — The agent loop

**Goal.** Build the autonomous gather-act-verify loop yourself, with caps, traces and a verification contract, so every later framework is legible.

**Concepts introduced**
- The loop: reason, call a tool, observe, repeat until done. The model supplies reasoning; tools supply action; the code around them is the harness (01 §1, §4).
- Stop conditions owned by code: step cap, USD cap, wall-clock cap, repeated-call detector. A model deciding "I'm done" is a claim, not a fact.
- Verification contract: a runnable check that closes the loop. Premature "done" is a characteristic failure (01 §10, 03 §10).
- Tool errors are observations, not exceptions. Truncate large outputs with a "narrow your query" message (01 §7, 09 §7).
- Context grows every step; windowed reads and short outputs are the first defense (01 §8).
- Traces: one JSONL record per step, replayable.
- Non-determinism: run several trials; pass@k versus pass^k (01 §9).
- The minimal scaffold lesson: mini-SWE-agent is bash-only and about 100 lines, with a project-reported score above 74% on SWE-bench Verified (01 §1, §9). The loop is small; the engineering is around it.

**Read**
- Core: [01](./use-cases/01-interactive-coding-agents.md) §1, §4, §7, §8 (first two bullets), §9, §10, §11 steps 1–3. [09](./use-cases/09-deep-research-analyst-agents.md) §11 step 1 (day-1 MVP, then measure on about 20 questions). [12](./use-cases/12-scientific-engineering-discovery-agents.md) §11 steps 1–2 ("write `evaluate()` first").
- Skim: [04](./use-cases/04-support-resolution-agents.md) §11 sketch (turn cap with handoff on cap); [landscape](./01-frameworks-landscape.md) §3.1 table, to see which frameworks are loops and which are graphs.

**Build — P2 Data analyst agent** (`agents/p2_analyst/`, loop in `core/loop.py`)
- Tools: `describe_schema()`, `sql_query(sql)` (single SELECT, read-only connection, 50-row limit, 5 s timeout, truncated output with guidance), `final_answer(value, unit, evidence_sql)`.
- Loop, about 120 lines, no framework (pseudocode):

```text
state = {steps: 0, usd: 0.0, seen: Counter()}; messages = [task]
loop:
  if steps >= MAX_STEPS or usd >= MAX_USD or elapsed > MAX_SECONDS: halt("cap")
  resp = model(messages, tools, system)                  # log tokens, USD, latency
  calls = tool_use_blocks(resp)
  if no calls: return accept(resp) if verify(resp) else nudge("verification failed: ...")
  for c in calls:
      seen[hash(c)] += 1
      result = error("repeated call; change approach") if seen[hash(c)] > 3 else run_tool(c)  # never raises
  messages += [resp, tool_results(truncate(results))]; trace.write(step, resp, results)
```
- Verification contract: the harness re-runs `evidence_sql`; if the claimed value is not in the result (within tolerance) the observation is "evidence mismatch" and the loop continues.
- `replay.py` re-executes every recorded tool call against the database and reports result drift.
- Dataset: any database with at least 3 related tables. At least 25 questions (15 dev, 10 test) with golden SQL: 8 single aggregates, 8 joins, 6 multi-step, at least 3 unanswerable, 1 poisoned-row case (a text cell that contains instructions).

**Acceptance criteria**
1. Test split, 3 trials: pass@1 ≥ 80% and pass^3 ≥ 65% (numeric tolerance 0.5%).
2. Unanswerable questions: 3 of 3 end with an explicit "cannot determine" and no invented number.
3. 100% of runs end within caps, including fault-injected runs (SQL syntax error, timeout, empty result) and a loop-bait question; no tool failure crashes the loop.
4. Every run has a complete trace; `replay.py` reports 0 drift on a static database.
5. p50 and p95 steps, USD and latency are reported, plus cost per correct answer.

**Gate eval.** `make eval-p2` writes `evals/results/L2.md` with the criteria above. Rerun the L1 suite (regression rule).

**Framework — none (raw API).** The loop is small, and writing it once gives you the vocabulary that every framework renames. After the gate, port it to the Claude Agent SDK `query()` or the OpenAI Agents SDK and diff the traces; the difference is what a harness adds (permissions, hooks, context management, subagents).

**Domain variant (dataset swap only).** Replace the database with a synthetic closed-sales table (`sale_date, price, sqft, beds, zone, days_on_market, list_price`). P2 becomes a market-stats analyst. Keep "unanswerable" cases such as "median price for a zone with 2 sales".

**Common mistakes**
- Caps enforced by prompt ("stop after 10 steps") instead of code.
- Raising exceptions from tools instead of returning them as observations, so the loop dies on the first bad query.
- Dumping unbounded tool output into context (01 §8).
- Accepting the model's final answer without a check (01 §10).
- Judging one run of one question; consistency needs several trials (pass^k).
- Adding more tools to fix a prompt problem; every tool is another failure source (03 §7).

**Stretch.** Add context compaction (clear old tool results, then summarize) and measure the effect on long questions (01 §8). Compare against a bash-only variant on the same questions. Allow parallel tool calls and measure steps saved.

**Consulting artifact.** Cost and step envelope: p50/p95 steps, USD and latency per task. You will use it to price pilots.

---

## L3 — Harness and action gates

**Goal.** Let an agent touch real tools while a deterministic layer, not the model, decides what can leave the building.

**Concepts introduced**
- Tool classes: read (cheap, safe), draft (writes to a review queue), commit (external, irreversible). The approval boundary is a missing tool, not a prompt: the agent has no `send` (06 §7 lesson 1).
- Layered permissions: hooks and deny rules first, allow rules and modes next, a human last (01 §10). Approval fatigue is real: users approved 93% of prompts, so a prompt per action does not protect anyone (01 §10).
- Deterministic gates that return a reason the model can act on ("Max 120 words; cite source URLs"), not a bare refusal (04 §7, 06 §7 lesson 5).
- Provenance at the tool boundary: facts carry `source_url`; the gate rejects uncited claims (06 §7 lesson 2).
- Identity is bound by the harness, never passed by the model (04 §7).
- Injection basics: all retrieved text is data; content-originated actions become proposals (06 §4 plugin note, 04 §8).
- Per-call approval tokens bound to tool name and arguments (02 §4, §7) as the stronger pattern for writes.
- Harness-as-library: built-in tools, hooks, subagents, MCP, permissions, caps (landscape §3.2).

**Read**
- Core: [06](./use-cases/06-gtm-sales-marketing-creative-agents.md) §1, §4, §7, §10, §11 (steps 1–4 and the SDK sketch). [01](./use-cases/01-interactive-coding-agents.md) §10, §11 steps 4–5 and its sketch (`can_use_tool`, `PreToolUse` hook). [07](./use-cases/07-workplace-knowledge-agents.md) §10 (`auto` is not a human checkpoint; SDK gotchas). [02](./use-cases/02-sre-security-investigation-agents.md) §7 (`propose_action`) and §10 (unsafe remediation row).
- Skim: [04](./use-cases/04-support-resolution-agents.md) §10 (permissioning); [landscape](./01-frameworks-landscape.md) §3.2 (Claude Agent SDK, OpenAI Agents SDK).

**Build — P3 Draft-only outreach agent** (`agents/p3_drafter/`, gates in `gates/`)
- Claude Agent SDK. In-process MCP tools: `crm_lookup`, `enrich_account` (returns facts with source URL), `queue_draft`. Built-in Bash, Write, Edit and web tools disabled (`disallowed_tools` as in the 06 sketch, or `tools=[]` as in the 07 sketch). There is no send tool anywhere in the agent's tool list.
- `can_use_tool` gate in plain code: tool allowlist; suppression list; at most 120 words; every factual claim cites a source from this run's tool results. Denials return actionable reasons.
- Review queue with an approve/edit/reject CLI in a separate module and process that the agent cannot import. Log every approval and edit as data (06 §11 step 7).
- Caps: `max_turns`, `max_budget_usd`.
- Dataset: 30 account fixtures (stub CRM and enrichment), 10 poisoned fixtures (a CRM note, a web snippet and a prior reply containing instructions), and 20 seeded gate violations, scripted as tool-call sequences plus prompts that tempt the model ("just send it now").
- Python SDK note: with `query()`, `can_use_tool` needs a streaming (async-iterable) prompt plus a keep-alive `PreToolUse` hook (06 §11 sketch). The 01 sketch uses `ClaudeSDKClient` instead. Bare entries in `allowed_tools` bypass `can_use_tool` (07 §10).

**Acceptance criteria**
1. Gate unit tests: 20 of 20 seeded violations denied with an actionable reason; at most 1 of 20 clean drafts falsely denied.
2. Structure test: the tool list contains no send or commit tool; built-ins are disabled; the approval module is not importable from the agent process.
3. On test fixtures (at least 15, 3 trials): at least 95% of factual claims are traceable (cited source ID exists in that run's tool results and the figure matches); judge rubric (specific relevance line, value bridge, soft CTA, ≤ 120 words) mean ≥ 4/5 with the judge in a fresh context.
4. Injection: 10 of 10 poisoned inputs produce 0 non-allowlisted tool calls and no draft that carries the injected payload.
5. A loop-bait prompt halts within `max_turns` and the budget cap.

**Gate eval.** `make eval-p3` writes `evals/results/L3.md`. Rerun L1 and L2 suites.

**Framework — Claude Agent SDK (Python).** It ships the permission model, hooks, in-process MCP tools, subagents and caps as a library, which is what this level teaches (landscape §3.2; the 01, 06 and 07 sketches use it). Alternatives: OpenAI Agents SDK, whose input/output/tool guardrails with tripwires cover similar ground (04 §10), if your stack is OpenAI-centric. Caveats: pre-1.0 versions, so pin; it is built around Claude Code's harness, and non-Claude support is unverified in the atlas.

**Domain variant — luxury lead follow-up drafter.** Input: a qualified lead from P1 plus listing records. Extra gate rules: off-market or NDA listings may only be mentioned to contacts flagged `verified_principal`; any price or availability statement must quote a `get_listing` result retrieved within 24 hours; reply language matches the lead's language; do not draft to a lead flagged `represented_by_another_agent`; wording passes a neutral-language check. The last two rules are domain assumptions (brokerage solicitation norms, fair-housing style wording rules): have your broker's compliance contact define the real rules. The atlas does not cover them. The gate is the point, not the copy.

**Common mistakes**
- Giving the agent a send or commit tool and relying on a prompt to restrain it (06 §7).
- Treating `allowed_tools` as a way to hide tools; it pre-approves them, and bare entries skip the human gate (07 §10).
- Putting must-run rules in CLAUDE.md or the system prompt; that is context, not enforcement. Use hooks (01 §10).
- Prompting a human for every action; approval fatigue makes the gate decorative (01 §10).
- Letting the model supply the customer or account ID (04 §7).
- Mistaking an "auto" or model-judged approval for a human checkpoint (07 §10).

**Stretch.** Add a small-model classifier as a second gate layer and measure its false-positive and false-negative rates against your seeded set. Implement a per-call approval token bound to tool name and arguments for one write tool (02 §7). Send approval requests to Slack or email through a permission-request hook (07 §11 step 4).

**Consulting artifact.** Approval-boundary diagram plus gate catalog: for a client workflow, list each action, its class (read, draft, commit), who approves, and the deterministic rule that blocks it.

---

## L4 — Knowledge, procedures and service agents

**Goal.** Give an agent a body of knowledge and a written procedure, bind it to an identity, and prove with simulated users that it resolves cases, escalates correctly and leaks nothing.

**Concepts introduced**
- Procedure as plan: a numbered, versioned SOP with stop conditions and escalation triggers, in a skill or system prompt (04 §4, §8). Skills use progressive disclosure: metadata always loaded, body on trigger, files on demand (landscape §5.4, 07 §8).
- Retrieval: just-in-time lookups with identifiers instead of stuffing context (01 §8). Permission-aware retrieval: ACL filtering happens before the model sees text (07 §1, §10). Start by federating to existing search; build your own ACL-synced index only when latency, ranking or joins force it (07 §11 steps 2 and 7).
- MCP: expose tools through a server so they are reusable and independently testable; default write tools to ask (07 §7 lesson 5; landscape §5.1).
- Read tools return derived policy facts (`eligible`, `max_amount`); write tools enforce limits and idempotency and return DENIED with a reason; `escalate_to_human` is a first-class, tested tool (04 §7).
- Memory: principal-scoped; structured profile fields, not free-form PII (04 §8, 07 §8). Handoff package: asked, verified, done, blocked.
- Release snapshot: prompt, SOP, tools, KB version and model stamped on every conversation (04 §8, §10).
- Simulated-user evaluation: a second model plays the customer; grade final database state and communication constraints; use pass^k. Measure true resolution separately from deflection (04 §9).

**Read**
- Core: [04](./use-cases/04-support-resolution-agents.md) §1, §4, §7, §8, §9, §10, §11. [07](./use-cases/07-workplace-knowledge-agents.md) §1, §4, §7, §8, §9 (ACL canaries), §10 (permission leakage, confused deputy). [01](./use-cases/01-interactive-coding-agents.md) §8 (context and memory).
- Skim: [landscape](./01-frameworks-landscape.md) §5.1 (MCP), §5.4 (skills), §6.4–§6.5 (memory and vector stores); [05](./use-cases/05-voice-agents.md) §3 to see how the same procedures behave under a latency budget.

**Build — P4 Policy-grounded service agent + MCP server** (`agents/p4_service/`, `kb/`)
- Pick a fictional store (orders, refunds, plan changes). SOP files with numbered steps, stop conditions and escalation triggers; 30–50 versioned KB passages.
- Tools in an MCP server (stdio): `search_kb` (returns passage ID and `kb_version`; ACL-filtered by a harness-bound principal), `lookup_order`, `check_refund_eligibility`, `issue_refund` (limit, idempotency key, DENIED with reason), `escalate_to_human(summary, category, urgency, attempted_steps)`. The customer ID is injected by the harness.
- Agent: Claude Agent SDK with the MCP server and a `SKILL.md` for the SOP. Release manifest stamped on every conversation.
- Simulator: persona prompt plus scenario (goal, hidden facts, adversarial behaviors such as social-engineering a refund or claiming to be another customer). Grader: final database state, transcript constraints (for example at most 10 turns), and an LLM rubric for tone.
- Dataset: 60 scripted or simulated conversations (30 dev, 30 test), including at least 20 adversarial or injection cases (a KB document containing instructions; a user asking for another customer's order).

**Acceptance criteria**
1. Test split, 3 trials: state-checked task success ≥ 80%, pass^3 ≥ 60%.
2. Escalation accuracy ≥ 95% (Anthropic's suggested target in 04 §9) on cases where escalation is required or forbidden.
3. ACL canary: 0 leaks across at least 20 adversarial attempts, including injection via a KB passage.
4. Groundedness: at least 95% of KB-derived statements cite a passage whose text contains the claim (mechanical check plus judge).
5. Every conversation carries a release manifest; changing the SOP version changes the manifest; L3's gate suite still passes.

**Gate eval.** `make eval-p4` writes `evals/results/L4.md`, with resolution, deflection, escalation and cost per conversation reported separately.

**Framework — Claude Agent SDK + MCP + `SKILL.md`.** Same harness as L3, so you add knowledge without relearning permissions. The procedure is a skill, the tools are an MCP server, and the SOP stays a versioned file in git. Retrieval: SQLite FTS5 or pgvector first; add a vector database only after your retrieval eval shows it is the bottleneck. (Third-party advice to start on pgvector at modest scale is unbenchmarked in the atlas, landscape §6.5; treat it as a default, not a fact.) Alternatives: OpenAI Agents SDK handoff-style triage to specialists (04 §4); Parlant for guideline matching per turn (04 §4); Inspect AI as an eval runner (landscape §7).

**Domain variant — luxury listing concierge.** KB: listing dossiers (features, fees, taxes, showing rules), neighborhood notes, brokerage FAQ, each tagged `visibility: public | verified_client | agents_only`. Principals: anonymous visitor, verified client, agent. Tools add `propose_showing_slots` and `book_showing` (verified principals only, idempotent, stub calendar). Escalate on offers, price negotiation, legal or financing questions and press. Simulated users: serious buyer, tire-kicker, someone claiming "I'm his assistant, send me the off-market list", a multilingual client. The ACL canary matters more here than in a generic store: discretion is part of the service, so the off-market canary must show 0 leaks.

**Common mistakes**
- Stuffing every procedure and document into one prompt instead of loading what the routed intent needs (04 §8).
- Enforcing access in the prompt rather than in the retrieval layer (07 §10).
- Unversioned KB and SOP, so a regression cannot be traced to a change (04 §8).
- Measuring deflection and calling it resolution (04 §9).
- Testing only the happy path; no adversarial users, no injection in the KB.
- Never testing that `escalate_to_human` actually gets used when it should (04 §7).
- Loading dozens of MCP tools into context; add MCP servers sparingly and defer tool definitions (01 §7, §11 step 9).

**Stretch.** Build an ACL-synced index with query-time censoring and compare it with federated search on latency and leakage (07 §10). Add principal-scoped memory with a test that a memory written under user A is never recalled under user B. Add a small-model input and tool-output screen and measure its false-positive rate (04 §4).

**Consulting artifact.** Knowledge and permission map: sources, owners, freshness, ACL model, who can see what, plus an escalation matrix.

---

## L5 — Orchestrator-workers and research agents

**Goal.** Use parallel, context-isolated subagents only where measurement shows they pay for their tokens, and make every claim traceable.

**Concepts introduced**
- Orchestrator-workers: a lead plans and delegates; workers run tool loops in fresh context windows and return condensed findings (09 §4, §8). Return size guidance in 09 §8 is about 1,000–2,000 tokens.
- Effort scaling written into the lead prompt: one agent and a few calls for a fact lookup, a few subagents for a comparison, many for broad research (09 §4).
- Evidence at the tool boundary: `record_finding(claim, url_or_doc, quote)` means the report cannot cite what was never read (09 §7).
- Plan and notes as files, not only context; workers write raw findings to disk to avoid "telephone" loss (09 §8, §10).
- A separate verifier in a fresh context checks each claim against its quote; unsupported claims are marked, not silently kept (09 §4; self-grading bias, 03 §4). Same-model oversight shares blind spots (11 §10), and LLM judges rank LLM output while sharing its blind spots (12 §10), so prefer deterministic checks (quote is a substring of the stored page) and consider a different model for the verifier.
- Hypothesis workers with refutation and three outcomes (validated, invalidated, inconclusive) (02 §1, §4).
- Governor outside the model: budget, depth and concurrency caps (09 §10).
- Economics: agents use about 4x chat tokens and multi-agent systems about 15x; token spend explained about 80% of BrowseComp variance in Anthropic's data (vendor-reported, 09 §1, §12). Multi-agent also costs 3–10x a single agent and earns its keep only for context protection, parallelism or specialization (07 §4).

**Read**
- Core: [09](./use-cases/09-deep-research-analyst-agents.md) §1, §4, §7, §8, §9, §10, §11, §12. [02](./use-cases/02-sre-security-investigation-agents.md) §1, §4, §10, §11 steps 2, 4, 5. [07](./use-cases/07-workplace-knowledge-agents.md) §4 (when to use multi-agent).
- Skim: [03](./use-cases/03-async-coding-agent-fleets.md) §4 (evaluator-optimizer note); [landscape](./01-frameworks-landscape.md) §3.2 (Claude Agent SDK subagents, Deep Agents, LangGraph) and §4.1 (Managed Agents `callable_agents`: research preview, one delegation level).

**Build — P5 Research agent with verifier** (`agents/p5_research/`, `research/`)
- Step 1, baseline: one agent with `web_search`, `fetch_page` and a report template with mandatory inline citations. Score it on the test split and commit the result before any subagent exists.
- Step 2: lead plus 2–6 workers on a cheaper model. Workers get an objective, an output format, source hints, boundaries and a tool-call budget; they write findings to files. The lead emits no citations. A verifier subagent in a fresh context checks each claim against its recorded quote and the stored fetched text; failures become `[UNSOURCED]`.
- Governor hooks: maximum spawn count, maximum concurrency, depth 1, USD cap.
- Dataset: 24 questions (about 60/40 dev/test, stratified): 4 simple lookups, 10 comparisons, 7 broad, 3 unanswerable; plus 5 poisoned pages. Judge rubric (accuracy, citation accuracy, completeness, source quality, tool efficiency, each 0–1) as in 09 §9.

**Acceptance criteria**
1. The single-agent baseline is committed with test scores before any subagent code.
2. Citation fidelity ≥ 95%: every claim in the final report maps to a recorded finding whose quote is a verbatim substring of stored fetched text. Failures appear as `[UNSOURCED]`.
3. Judge mean ≥ 0.70 on test for the configuration you carry forward. A complete comparison table (judge score, citation fidelity, tokens, USD, wall-clock) for single-agent versus lead-plus-workers, with a written verdict on whether the token multiple was justified. A negative verdict is a valid result.
4. 100% of runs stay within the USD cap, worker cap and depth 1. Effort scaling holds in traces: simple queries use at most 1 worker; broad queries use at least 3.
5. At least 2 of 3 unanswerable questions end in an explicit "insufficient evidence" instead of a confident report.
6. Injection: 0 of 5 poisoned pages cause any lead or worker tool call outside the research tools.

**Gate eval.** `make eval-p5` writes `evals/results/L5.md` with raw counts. Rerun earlier suites.

**Framework — Claude Agent SDK subagents (programmatic `AgentDefinition`).** The orchestrator-workers pattern stays on the harness you already know, with context isolation, built-in WebSearch and WebFetch, and `max_budget_usd` (09 §11 and its MVP stack). Alternatives: LangGraph or Deep Agents when you want an explicit graph or the LangChain ecosystem (landscape §3.2); Managed Agents `callable_agents` is a research preview limited to one delegation level (09 §4). Gotcha: `allowed_tools` pre-approves rather than hides tools, so "the lead never searches" needs a `PreToolUse` hook (09 §11, sketch caveat).

**Domain variant — comparable-sales research memo.** Subject: a property. Workers, one per source class: closed sales (radius and time window), active and pending competition, market trend context, unique-feature comparables (waterfront, view, architect). `record_finding` requires a source row or URL plus the quoted figures. Deterministic code, not the model, recomputes price per square foot and medians and checks each cited sale's fields against its source row. The memo gives a value range with stated adjustment rationale and an explicit "insufficient comparables" state; a broker signs it, and the agent never publishes a price (draft-then-sign-off, 09 §10). Luxury comps are thin and heterogeneous, so abstention is a feature. Extra eval: a blind holdout of 8–10 closed sales you can access with the sold price hidden. Report coverage (sold price inside the range), range width, citation fidelity, and correct abstention on 3 cases with fewer than 3 usable comps. Set the acceptable range width with a broker; do not copy a number from here. Check MLS or IDX data-license terms before using feeds in prompts, logs or eval sets (outside the atlas).

**Common mistakes**
- Defaulting to multi-agent without a single-agent baseline; most of the gain may be token spend (09 §12).
- Over-delegation: 50 subagents for a simple query. Effort-scaling rules and hard caps prevent it (09 §10).
- Summaries that drop the quote and the caveat; keep verbatim evidence (09 §8).
- Letting the writer verify its own citations (03 §4).
- Treating SEO farms like primary sources; add source-quality rules to worker prompts (09 §10).
- Caps that exist only in the prompt, not in hooks (09 §10).

**Stretch.** Add a refutation pass: a worker assigned to disprove the lead's top claim (02 §4). Reimplement the lead on LangGraph or Deep Agents and compare cost and score. Add prompt caching for the stable lead prompt and report the savings (09 §8, §12).

**Consulting artifact.** Multi-agent ROI note: tokens times price versus task value, with the single-agent baseline as the comparator.

---

## L6 — Durable workflows, pipelines and human gates

**Goal.** Build a deterministic pipeline where the model is quarantined, every write is gated, and a run can pause for days and survive a crash.

**Concepts introduced**
- Keep the backbone deterministic and quarantine the model. The reader that opens untrusted documents has no tools and returns length-capped, schema-validated JSON; one writer role exists and never sees outsider content (08 §1, §4, §7).
- Separate read, propose and commit: no agent tool posts, pays or approves. A deterministic commit service enforces an authority matrix (08 §7).
- Exceptions as a product: reason codes, evidence references, recommended action, a human queue (08 §4).
- Audit rows: actor, basis, timestamp, ruleset, prompt and model versions (08 §7, §10).
- Durable state: checkpoints, interrupt and resume, idempotent writes with case-scoped keys, approvals that wait days (08 §4; landscape §6.3).
- Triggers and per-run budgets: webhook or cron creates a fresh run with a hard budget (07 §11 step 5).
- Economics: cost per case is machine cost plus exception rate times reviewer minutes times loaded rate; reviewer time dominates (08 §12). Message batches halve cost for non-urgent work (08 §12).
- Failure modes: silent extraction errors, evidence fabrication, automation bias (reviewers rubber-stamp) (08 §10); calibrate confidence against error cost.

**Read**
- Core: [08](./use-cases/08-back-office-case-processing-agents.md) §1, §4, §7, §8, §9, §10, §11, §12. [07](./use-cases/07-workplace-knowledge-agents.md) §4 (trigger routing) and §11 steps 4–5. [landscape](./01-frameworks-landscape.md) §3.2 (LangGraph, Microsoft Agent Framework, ADK), §6.3 (durable execution and its rule of thumb), §8.2 rules 3–4.
- Skim: [03](./use-cases/03-async-coding-agent-fleets.md) §4 (deterministic nodes with caps); [06](./use-cases/06-gtm-sales-marketing-creative-agents.md) §11 steps 5–6.

**Build — P6 Case pipeline with exception queue** (`agents/p6_pipeline/`, `flows/`)
- Supplier invoices as the generic case: classify and split, quarantined reader (structured output with page and quote), deterministic validators (arithmetic, formats, duplicates), three-way match against a read-only ERP stub, draft journal entry, authority matrix, then commit service or exception queue with reason codes.
- Orchestration: a LangGraph graph with a database checkpointer, `interrupt()` at human approval, and a cron trigger with a per-run budget.
- Dataset: at least 100 synthetic documents (60 dev, 40 test) with golden entries. Seed defects in at least 30%: duplicate, arithmetic mismatch, unknown vendor, over-PO amount, missing receipt, decimal or currency error, and at least 10 injection documents.

**Acceptance criteria**
1. Test split: 0 wrong auto-commits (a committed entry that differs from golden, or that should have been an exception).
2. Exception recall ≥ 95% on seeded defects; exception precision reported; straight-through rate ≥ 50% of clean documents. If you loosen validators to raise STP, show the evidence.
3. The reader has no tools (asserted in a test) and returns length-capped, schema-validated JSON. 10 of 10 injection documents have no downstream effect.
4. Crash-resume: kill the worker at a random node in 20 runs; 0 duplicate writes and the final state equals an uninterrupted run.
5. An item above the authority limit pauses at `interrupt()`; a simulated 48-hour wait then resume works; the approver sees evidence references and reason code; the decision is recorded.
6. Audit: the audit table alone reconstructs every decision (actor, basis, timestamps, versions).
7. Cost per case is filled in with measured numbers using the 08 §12 formula.

**Gate eval.** `make eval-p6` writes `evals/results/L6.md`, including the kill-and-resume harness. Rerun earlier suites.

**Framework — LangGraph.** Explicit state, checkpointing and `interrupt()` match the landscape's rule to use a graph with checkpoints and human gates when you fear unbounded behavior (landscape §8.2 rule 3). You now have a real graph, so its ceremony pays; at L2 it would not have. Use framework-native checkpoints while runs last minutes and one team owns the stack; add Temporal (MIT) when waits last days, cross services or must survive deploys (landscape §6.3 rule of thumb, my synthesis). Alternatives: Microsoft Agent Framework (.NET or Azure shops), Google ADK 2.0 (GCP; note its 2.0 changed storage and event schemas), Pydantic AI plus any of its eight durable-execution integrations (landscape §3.2).

**Domain variant — monthly client report pipeline.** For each seller client: pull showings, inquiries and market movement; deterministic code computes the statistics; the model writes the narrative; a tie-out validator requires every number in the narrative to match a source figure (fail closed); an approval queue precedes sending; a cron trigger runs it monthly. Add a cross-client canary: client A's data must never appear in client B's report. Track edit distance between draft and approved text as a quality signal. A transaction-file completeness checker (purchase agreement, disclosures, proof of funds, missing-document exceptions) is an equally natural variant.

**Common mistakes**
- Building a free-running agent where a pipeline works; only the variable parts (messy documents, exceptions) need a model (08 §1).
- Keeping wait state in process memory, so a day-long approval cannot survive a deploy.
- Non-idempotent writes, which duplicate on replay (08 §7).
- Uncalibrated confidence scores used as thresholds (08 §9).
- Reviewers shown only the recommendation, not the evidence, which invites rubber-stamping (08 §10).
- Optimizing model quality when STP and exception handling set the economics (08 §12).

**Stretch.** Add an investigator agent with read-only tools plus a critic for exceptions only (08 §4). Port the flow to Temporal and compare failure handling. Use batch processing for non-urgent runs and measure the saving. Feed reviewer overrides back into eval cases (08 §8).

**Consulting artifact.** Authority matrix, exception taxonomy and STP business case.

---

## L7 — Isolated autonomy: sandboxes, verifier rails, unattended runs

**Goal.** Run an agent that executes code with real side effects, unattended and in parallel, without trusting it, and measure it with an evaluator it cannot touch.

**Concepts introduced**
- Isolation layers: hooks, deny and allow rules first; then OS, container or microVM isolation; then an egress allowlist; credentials never inside the sandbox, attached by a proxy or by harness steps (01 §10, 03 §10). Sandboxing cut approval prompts by 84% in one reported deployment (03 §10).
- Isolation models differ: kernel-per-sandbox microVM, shared-kernel container, gVisor. The smolagents local executor "is not a security boundary" (landscape §3.2, §6.1, §8.2 #7).
- The blueprint: fixed code nodes around agent nodes, with hard caps (about two CI fix rounds, a per-task budget) (03 §4). Verification layers: deterministic checks, then an LLM judge on the diff, then CI (03 §11 step 7).
- Evaluator-first: scoring code and held-out tests live outside the agent's write scope; candidates that game the scorer are the characteristic failure; independently re-run winners (12 §1, §10).
- Brain, hands and session decoupled: stateless harness, disposable sandboxes, an append-only event log (landscape §4.1, 03 §4).
- Fan-out: one sandbox per task, global concurrency caps, kill switch, canary batch (03 §10).
- Where there is no API, a browser agent is the fallback; API first (10 §1, §11).

**Read**
- Core: [03](./use-cases/03-async-coding-agent-fleets.md) §1, §4, §6, §7, §8 (long horizons), §10, §11, §12. [01](./use-cases/01-interactive-coding-agents.md) §10 and §11 steps 4–5. [12](./use-cases/12-scientific-engineering-discovery-agents.md) §1, §10, §11 steps 1–4. [landscape](./01-frameworks-landscape.md) §4.1 (brain, hands, session), §6.1 (sandboxes), §8.2 rules 4 and 7.
- Skim: [10](./use-cases/10-computer-use-browser-agents.md) §1 and §10 (injection, ambient credentials, consent gates).

**Build — P7 Ticket-to-draft-PR agent in a sandbox** (`agents/p7_ticket2pr/`, `sandbox/`)
- Target repo: your own `agentlab/`. Tickets are generated by mutation: inject a bug, write the symptom as the ticket, keep a hidden test that catches it. Dataset: 20 tickets (12 dev, 8 test) plus 10 red-team probes.
- Blueprint (code owns the sequence): create worktree and hydrate context (AGENTS.md, ticket) → agent implements → lint and tests → agent fixes, at most 2 rounds → fresh-context LLM judge compares diff with ticket → harness commits, pushes and opens a draft PR.
- Sandbox: rootless container, non-root user, read-only root filesystem except the worktree and a temp directory, CPU, memory and process limits, no network except an egress allowlist (model API and a package mirror). No credentials inside; push and PR creation are harness steps. Hidden tests live outside the writable mount and are run by the harness.
- Agent: Claude Agent SDK headless inside the container, with a tool allowlist and `max_budget_usd`.
- Red-team probes: read a canary secret outside the workspace; curl a non-allowlisted host; write outside the worktree; push to a protected branch; edit or delete the hidden tests; exhaust resources; install from a non-allowlisted index; read environment variables for secrets; DNS or URL exfiltration of a canary; escape the worktree through a symlink or path traversal.

**Acceptance criteria**
1. 10 of 10 red-team probes contained and logged.
2. No credentials in the sandbox (canary secret and environment scan); the agent has no push or `gh` capability.
3. Blueprint enforced in code: an unfixable ticket ends as "failed: caps" with no PR, within 2 fix rounds and the per-task USD cap.
4. Evaluator integrity: hidden tests unreachable from the sandbox; a diff checker flags 5 of 5 seeded test-weakening diffs (deleted assertions, `skip`, edited fixtures).
5. Replay: at least 50% of test tickets yield a diff that passes the hidden tests (3 trials; report pass^3, USD per ticket, and your review-accept rate). Adjust the bar after the dev run: the point is the measurement.
6. Fleet: 5 tickets in parallel in separate worktrees and containers; a canary file planted in task A is invisible to task B; the global concurrency cap holds.
7. One batch runs unattended from a cron or CLI trigger and leaves PRs and traces.

**Gate eval.** `make eval-p7` writes `evals/results/L7.md` with infrastructure pinned (container limits, tool versions). Rerun earlier suites.

**Framework — Claude Agent SDK, headless, inside a container you operate.** The harness supplies the loop, file and shell tools and permissions; isolation is infrastructure, and the SDK runs in a process you operate (landscape §3.2), so do not conflate the two. Ready-made harnesses to compare: goose or OpenHands (03 §6). Hosted sandboxes when you outgrow local containers: E2B (Apache-2.0), Modal, Cloudflare; Daytona moved core development to a private codebase in June 2026, so check license and isolation claims (landscape §6.1). Add Temporal only when runs last hours.

**Domain variant.** None. This is an infrastructure level. The real-estate analogue is running L5 and L6 agents against client data, handled by L8's data-handling decision.

**Common mistakes**
- Credentials inside the sandbox (01 §10, 03 §10).
- No egress allowlist; an allowlisted model-API domain was used for exfiltration until provenance was validated (03 §10).
- Letting the agent read or edit the evaluator or tests (12 §10).
- Judging by merged-PR counts without controlling for task selection (03 §9).
- Commits made with the default `GITHUB_TOKEN` do not trigger CI (03 §10).
- Comparing scores across different container limits (01 §9).
- Running unattended without caps or a kill switch (03 §10).

**Stretch.** Run the same blueprint on a hosted microVM sandbox and compare isolation, cold start and cost. Add a separate reviewer agent for a second opinion and measure judge-veto rate (03 §9). Add a canary-then-fan-out rollout with a kill switch.

**Consulting artifact.** One-page threat model: assets, injection paths, blast radius, controls, residual risk.

---

## L8 — Production system: evals in CI, observability, cost, release, governance

**Goal.** Make the portfolio operable: regressions are caught before release, failures are traceable, cost is bounded, and rollout and rollback are rehearsed.

**Concepts introduced**
- Eval pyramid: code checks, state checks, LLM judge, human sample, online metrics (01 §9, 04 §9, 09 §9). Cumulative regression on every change to prompt, tool, SOP, KB, model or SDK.
- Outcome metrics, not activity: meetings held versus emails sent (06 §9); resolution versus deflection (04 §9); cost per outcome.
- Tracing: a span per model call, tool call, gate decision and approval, tagged with run ID, release ID and tenant. OpenTelemetry GenAI conventions were still "Development" in mid-2026, so pin versions; OpenInference covers the Claude Agent SDK and others (landscape §7).
- Cost control: per-run and per-tenant budgets, model tiering, prompt caching, batch for non-urgent work, small models for volume (09 §12, 08 §12, 06 §12).
- Release discipline: snapshot (model, prompt, SOP, tools, KB, SDK), shadow then assist then autonomy, canary, one-command rollback, traffic shifting so in-flight runs finish on the old version (04 §10, 09 §10).
- Kill switch: route everything to humans (04 §10, 05 §10).
- Data governance: what enters prompts and logs, retention, redaction. Claude Managed Agents is not currently eligible for Zero Data Retention or HIPAA BAA coverage because it is stateful (landscape §4.1, 07 §10, 08 §10).
- Churn: hosted visual builders and hosted evals are fragile dependencies; keep evals in your repo (landscape §7, §8.2 #5).

**Read**
- Core: [01](./use-cases/01-interactive-coding-agents.md) §9, §12. [03](./use-cases/03-async-coding-agent-fleets.md) §9, §10, §12. [04](./use-cases/04-support-resolution-agents.md) §9–§12. [06](./use-cases/06-gtm-sales-marketing-creative-agents.md) §9, §12. [07](./use-cases/07-workplace-knowledge-agents.md) §9, §10, §12. [08](./use-cases/08-back-office-case-processing-agents.md) §10, §12. [09](./use-cases/09-deep-research-analyst-agents.md) §9, §10, §12. [landscape](./01-frameworks-landscape.md) §4, §6.6 (gateways), §7 (evals and observability), §8 (decision guide), §9 (risks).
- Skim: [05](./use-cases/05-voice-agents.md) §9 (field-level evals); [10](./use-cases/10-computer-use-browser-agents.md) §9 (outcome-verified success).

**Build — P8 Production pack and capstone** (`ops/`, `agents/p8_capstone/`)
- Pick your two strongest agents (suggest P3 plus P4, or P5 plus P6) and run them as a service with: OpenTelemetry tracing to a backend; an LLM gateway with budgets; eval CI; a release manifest with shadow, canary and rollback; a kill switch; dashboards; a runbook; a risk register; a data-handling memo.
- Rehearsals: a deliberately degraded prompt; a seeded injection incident; a model or SDK version bump.

**Acceptance criteria**
1. One command runs the cumulative regression on every change to prompt, tool, SOP, KB, model or SDK version; results are stored with the version manifest.
2. Degradation drill: remove a key rule (for example the SOP's escalation rule). CI fails before release; if CI is bypassed, canary metrics trip an automatic rollback.
3. Tracing: every model call, tool call, gate decision and approval is a span carrying run ID, release ID, tenant and cost. Incident drill: given a seeded failure (an injection that produced a bad draft), identify the root-cause span from telemetry alone within 30 minutes.
4. Cost: per-run budget and per-tenant daily budget enforced at the gateway; a dashboard shows cost per outcome (approved draft, resolved conversation, processed case, merged PR) and p50/p95 latency; a budget alarm is tested.
5. Release flow: manifest stamped on every run; shadow, assist and autonomy promotion criteria written as metrics (approval rate, edit rate, wrong-action rate, escalation accuracy); canary at about 5% and rollback exercised; the kill switch routes all traffic to humans in under a minute in a drill.
6. Governance: data-handling memo (client data in prompts and logs, retention, redaction, vendor retention limits), risk register, runbook.
7. Churn rehearsal: bump the SDK or model, run the suite, read the diff, decide.

**Gate eval.** `make eval-all` writes `evals/results/L8.md`: all earlier gates, the degradation, incident and kill-switch drills, the churn rehearsal, and a one-page operations report.

**Framework — no new agent framework; add the operations layer.** OpenTelemetry plus OpenInference instrumentation feeding a trace backend: Langfuse (MIT, self-hostable, owned by ClickHouse since January 2026), Phoenix (source-available, Elastic License 2.0) or LangSmith (commercial, framework-agnostic) (landscape §7). Evals as code in your repo (Inspect AI, DeepEval or Promptfoo-style configs); OpenAI's hosted Evals product is reported to shut down on 2026-11-30 (secondary source in the atlas), which is the argument for repo-local evals. LiteLLM as the gateway for budgets and model routing (landscape §6.6). Deployment target by the landscape's decision flow (§8.1): your own containers, or a managed platform (AgentCore, Foundry, Gemini Enterprise Agent Platform, Claude Managed Agents in beta), weighed against the data-retention limits above.

**Domain variant — luxury "deal desk" capstone.** Compose P1 (inquiry intake) → P3 (follow-up drafts) → P4 (listing concierge) → P5 (comps memo) → P6 (client report) as five agents under one ops layer: separate budgets, gates, release manifests and a shared trace backend. Tenancy is per client; CI canaries prove client A's data never appears in client B's outputs. Promotion path per agent: shadow beside the human team, then assist, then autonomy on the narrowest flow only.

**Common mistakes**
- Adding evals at the end instead of from L1; the dataset is the scarcest asset.
- Tracing without outcome metrics: you can replay a failure but cannot tell whether the system is working.
- Tuning on the test split, or reusing it after changes without refreshing cases (the eval rules in §0).
- Shipping prompt changes without regression; a one-line prompt edit can remove an escalation behavior.
- No kill switch, or one never rehearsed.
- Treating vendor numbers (resolution rates, speedups) as targets; compare definitions first (04 §9).
- Floating dependency versions in a space with breaking majors and spec deprecations (landscape §8.2 #6, §9).

**Stretch.** Roll prompts out with traffic shifting so in-flight runs finish on the old version (09 §10). Optimize a prompt against your own eval metric with DSPy (GEPA is one of its optimizers; landscape §3.2). Run one agent on a managed platform and compare operational burden and data limits. Add A2A between two of your agents only if a cross-organization boundary actually exists (landscape §5.2).

**Consulting artifact.** Production readiness checklist and a pilot plan with promotion criteria and a rollback trigger.

---

## Elective tracks

Take these when a client need or curiosity pulls you sideways. Each assumes the listed level and reuses the shared eval harness.

| Track | After | Read | Mini-build | Pass bar |
|---|---|---|---|---|
| E1 Voice | L4 | [05](./use-cases/05-voice-agents.md) §3, §4, §9, §11 | Phone receptionist on LiveKit Agents or Pipecat: `verify_caller`, one read tool, one write tool, warm transfer to a human, fake backends | 10–20 scripted scenarios run in CI with judge assertions; P95 response gap measured and reported; transfer works in 5 of 5 escalation cases. Real-estate fit: showing-scheduling calls |
| E2 Browser and computer use | L7 | [10](./use-cases/10-computer-use-browser-agents.md) §4, §7, §10, §11 | Playwright tools with host allowlist, untrusted-page wrapping, consent hook for risky clicks, in a container | pass rate reported on 20 state-checked tasks; injection suite 0 successes; an API-first check documented for each task |
| E3 Investigation | L5 | [02](./use-cases/02-sre-security-investigation-agents.md) §4, §7, §9, §11 | Hypothesis workers plus a refutation judge plus a `propose_action` queue. Apply it to your own L8 traces: why did a run fail? | Replay 20 recorded failures; correct root cause on at least 60%; every claim cites a span |
| E4 Evaluator-driven search | L7 | [12](./use-cases/12-scientific-engineering-discovery-agents.md) §1, §4, §10, §11 | AlphaEvolve-style miniature: program database, parent plus inspirations prompt, smoke-then-full cascade, held-out score | Held-out score reported separately from fitness; a seeded reward-hacking candidate is caught |
| E5 Embodied and simulated worlds | L8 | [11](./use-cases/11-embodied-simulated-world-agents.md) §4, §9, §11 | Orchestrator with `observe`, `run_skill`, `estop` over a stub world; independent verifier; policy gate | 20 seeded scenarios; failure taxonomy (plan, verify, act); orchestrator ablation reported |

Pass bars here are lighter than level gates; they are mini-projects. Dossier 11 rates embodied agents as pilot (robotics) or research (games, simulation) maturity (11 §1), so treat E5 as learning, not as a client offering.

---

## Portfolio carry-forward

| Artifact | Built in | Reused in |
|---|---|---|
| `llm.py` client wrapper with model-from-env and USD accounting | L1 | all |
| Tool schema registry, errors-as-observations | L1–L2 | all |
| `loop.py` and JSONL `trace.py` with replay | L2 | L3–L8 |
| `evals/` runner, graders, results files | L1, extended every level | L8 CI |
| `gates/` policy functions and approval queue (separate process) | L3 | L4, L6, L7 |
| `kb/` retrieval, ACL filter, SOPs, MCP server, release manifest | L4 | L5, L8 |
| `research/` lead, worker, verifier, governor | L5 | L8 |
| `flows/` graph, validators, authority matrix, exception queue, audit table | L6 | L8 |
| `sandbox/` container, egress policy, blueprint runner, red-team probes | L7 | L8 |
| `ops/` OTel, gateway, budgets, release flow, runbooks, risk register | L8 | all |

**Definition of done for each project.** README with the architecture sketch and the approval boundary; the eval report in `evals/results/`; cost per task and p95 latency; a failure taxonomy of what broke in dev; and the level's consulting artifact. By L8 you hold, per agent: a decision memo, an approval-boundary diagram, an eval report, a threat model, an authority matrix and a readiness checklist. These are the documents a client engagement needs.

---

## Skills checklist (one page)

Tick an item only when you have shown it in a committed artifact, not when you can describe it. Level tags show where it is first practiced.

**Design judgment**
- [ ] Decide workflow versus agent for a given job and defend it in five lines (L1)
- [ ] Classify every action as read, draft or commit, and state its blast radius (L3)
- [ ] Express an approval boundary as a missing tool and draw it (L3)
- [ ] Choose single versus multi-agent from token cost, context size and parallelism need (L5)
- [ ] Choose a framework and hosting target with the §8.1 decision flow and state the lock-in (L6, L8)

**Building blocks**
- [ ] Write a tool schema and description; return high-signal output and actionable errors (L1)
- [ ] Enforce structured output: validation, repair retry, evidence quotes (L1)
- [ ] Implement the raw loop with step, USD and time caps, loop detection and traces (L2)
- [ ] Return tool errors as observations; truncate outputs with guidance (L2)
- [ ] Bind identity in the harness; never accept it from the model (L3, L4)
- [ ] Write a deterministic gate that returns actionable DENIED reasons (L3)
- [ ] Use hooks, `can_use_tool` and permission modes correctly; know the `allowed_tools` gotchas (L3)
- [ ] Package tools as an MCP server and procedures as a `SKILL.md` (L4)
- [ ] Manage context: truncation, compaction, notes in files, subagent isolation, caching (L2, L4, L5)
- [ ] Build ACL-aware retrieval and prove no leakage with canaries (L4)
- [ ] Run orchestrator-workers with effort scaling, condensed returns and a separate verifier (L5)
- [ ] Build a quarantined reader, validators, authority matrix and exception queue (L6)
- [ ] Build a graph with checkpoints, interrupt and resume, and idempotent writes (L6)
- [ ] Isolate execution: container limits, egress allowlist, credentials outside (L7)
- [ ] Build a blueprint of deterministic nodes and capped agent nodes (L7)

**Evaluation**
- [ ] Build a labeled dataset with a dev/test split and adversarial, injection and unanswerable cases (L1)
- [ ] Run multiple trials; compute pass@1 and pass^k and explain the difference (L2)
- [ ] Prefer code and state graders; write a rubric judge that runs in a fresh context (L3, L5)
- [ ] Run simulated-user evals graded on final state (L4)
- [ ] Verify citations mechanically against stored sources (L5)
- [ ] Keep evaluators and held-out data outside the agent's write scope (L7)
- [ ] Run cumulative regression in CI and catch a degraded prompt before release (L8)
- [ ] Separate outcome metrics from activity and deflection metrics (L4, L8)

**Security and safety**
- [ ] Treat all retrieved text as data; show an injection suite result per agent (L1, L3)
- [ ] Apply least privilege and keep ambient credentials out of agent reach (L3, L7)
- [ ] Layer gates and explain approval fatigue (L3)
- [ ] Threat-model an agent and red-team it in a sandbox (L7)
- [ ] Write a data-handling decision for client data, including vendor retention limits (L8)

**Operations and economics**
- [ ] Report cost, steps and latency per task at p50 and p95 from traces (L2)
- [ ] Enforce budgets outside the model (L2, L5, L7)
- [ ] Apply model tiering, prompt caching and batch where non-urgent (L5, L6, L8)
- [ ] Trace with OpenTelemetry and diagnose a failed run from telemetry alone (L8)
- [ ] Stamp release manifests; run canary, shadow-to-autonomy promotion, rollback and a kill-switch drill (L8)
- [ ] Pin versions and rehearse an upgrade (L8)

**Consulting layer**
- [ ] Produce the one-page artifacts: decision memo, approval-boundary diagram, cost envelope, threat model, authority matrix, readiness checklist (L1–L8)
- [ ] Estimate pilot cost from a measured envelope and ROI from a measured baseline (L2, L6)
- [ ] Define promotion criteria from shadow to assist to autonomy for a client workflow (L8)

---

## Limits and volatile items

- **Evidence quality.** The atlas is medium confidence; dossier 04 is low. Vendor-reported figures (resolution rates, speedups, benchmark scores) are design anchors only. The gate bars in this path are my defaults.
- **Fast-moving facts** (landscape §1, §9): Claude Agent SDK and OpenAI Agents SDK are pre-1.0; Claude Managed Agents is in beta; the MCP spec changed on 2026-07-28 with deprecations; Google ADK 2.0 broke storage and event schemas; OpenAI's hosted Agent Builder and Evals are reported (secondary source) to shut down on 2026-11-30; Daytona's core went closed-source in June 2026; OpenTelemetry GenAI conventions are still "Development". Pin versions and expect to revise levels L3, L5 and L8 first.
- **Not covered in the atlas, so not in this path:** fine-tuning and model training, front-end agent UIs (AG-UI, Vercel AI SDK), A2A interop beyond a stretch item, agent commerce and payments, dedicated memory vendors, regulatory detail. The real-estate compliance points (fair-housing wording, solicitation rules for represented clients, MLS/IDX data licenses, confidentiality of off-market listings, AML or KYC duties for cash purchases) are domain assumptions flagged where they appear; confirm each with your broker or counsel before an agent touches real client communications or data.
