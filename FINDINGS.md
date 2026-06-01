# FINDINGS — Managed Agents experiment (durable log)

**Purpose:** compaction-proof running log of every test + result. Updated continuously. Source of truth for the post.
Last updated: 2026-06-01.

Keys in repo `.env`: `GEMINI_API_KEY` (dev endpoint `generativelanguage.googleapis.com/v1beta`), `ANTHROPIC_API_KEY`.
Managed Agents dev endpoint confirmed live. Agent: `antigravity-preview-05-2026` (Gemini 3.5 Flash). Header: `Api-Revision: 2026-05-20`.

---

## A. Core battery (225 runs, n=25/cell) — DONE
Data: `data/runs_full.jsonl` | Report: `outputs/results.md`, `outputs/phase-run_full.md`

- **Reliability:** 100% pass on all tasks/arms (zero failures in 225 runs). NOTE: tasks were within-capability, so this is a cost/control/portability finding, not a reliability-cliff claim.
- **Harness tradeoff (L1):** same model, Managed Agents (Google runs the loop) vs self-orchestrated gemini-3.5-flash (I run the loop):
  - Cost ~5x (per-task 5.7x / 6.4x / 3.4x). Latency ~4x (4.1x / 4.3x / 3.7x).
  - Arm A executed real sandbox code on compute tasks (T1 100%, T2 88% of runs); T3 reasoning used 0% code-exec.
- **Portability (L10):** ONE AGENTS.md format rule fed to 3 runtimes (Google managed agent via inline AGENTS.md, Gemini direct via system prompt, Claude via system prompt) -> **100% compliance on all three**.
- Total est. cost for 225 runs: $0.57.

## B. Translator (Gemini <-> Anthropic, API layer) — DONE
Code: `scripts/translator.py` | Report: `outputs/phase-translator.md` | 4/4 live tests passed.
- Gemini-format request ran on Claude; Anthropic request ran on Gemini and returned in Anthropic shape; tool defs translate both ways (functionDeclarations <-> tools/input_schema; functionCall <-> tool_use).
- Scope: REST request/response only. Google's managed runtime (loop+sandbox) is NOT translatable to a format shim (would be "build a harness").

## C. Advanced primitives retest (firsthand, dev endpoint) — DONE
Code: `scripts/probe_advanced.py` | Report: `outputs/phase-advanced.md` | Data: `data/advanced_*.json`

| Primitive | Result | Evidence |
|---|---|---|
| **Custom SKILL.md** | **CONFIRMED (PASS)** | Registered agent with inline `.agents/skills/haiku-maker/SKILL.md`; ran "6x7?"; agent returned 42 AND applied the skill (appended `HAIKU:`). Source-defined skills work. |
| **MCP as `tools[]` (dev endpoint)** | **REJECTED (400)** | `"The value 'mcp_server' is not supported for 'type' at 'tools[...]'"`. So `{type: mcp_server, url, headers}` is **Enterprise/Vertex-only** (`aiplatform.googleapis.com`), NOT on the free dev API. Dev-side MCP is the `npx add-mcp` CLI-into-sandbox path. |
| **Secrets via egress proxy** | **INCONCLUSIVE (timed out)** | Registered agent with `base_environment.network.allowlist=[{domain:httpbin.org, transform:{X-Secret-Probe: Bearer ...}}]`; run timed out >300s. RETEST with simpler task + longer timeout. Documented mechanism (Phil Schmid): "Secrets never exist inside the sandbox" — injected as HTTP header transforms at the egress proxy. |
| **Subagents** | **NOT EXPOSED** | Step types only: thought / code_execution_call / code_execution_result / model_output. No sub-agent/spawn step type. |
| **Scorers / eval** | Separate product | "Agent evaluation" is a separate platform feature, not inline in the managed-agent definition (deprioritized per Pawel). |

## D. Agent definition schema (reconciled from docs + live)
Two surfaces, DIFFERENT capabilities:
- **Dev API** (`generativelanguage.googleapis.com/v1beta`, free, what I tested): `POST /v1beta/agents` with `{id, base_agent, system_instruction, base_environment:{type:remote, sources:[...], network:{allowlist:[...]}}}`. Sources types seen: `inline` (target + content), `repository` (source repo + target), `skill_registry`, `gcs`. Skills auto-discovered from `.agents/skills/<name>/SKILL.md`. NO `tools[]` mcp_server.
- **Enterprise/Vertex** (`aiplatform.googleapis.com/.../agents`): adds `tools[]` = `{type: code_execution|google_search|url_context|filesystem|mcp_server}`. MCP servers via `{type:mcp_server, name, url, headers}`. PATCH with update_mask.
- Run: `POST /v1beta/interactions` `{agent, input:[{type:text,text}], environment:{type:remote}}` -> Interaction `{id, status, usage, environment_id, steps[], ...}`. Final output = last `model_output.content[].text`. Steps: thought / function_call / function_result / code_execution_call / code_execution_result / model_output.

Sources (docs): blog.google managed-agents; ai.google.dev/gemini-api/docs/managed-agents-quickstart + /coding-agents; docs.cloud.google.com/gemini-enterprise-agent-platform/build/managed-agents (+ /create-manage); philschmid.de/gemini-managed-agents-developer-guide.

---

## E. OPEN / IN PROGRESS (new direction 2026-06-01)
Pawel: "biggest unlock = defining agents in SOURCE CODE with skills + MCPs and deploying them (or running locally), agents accessing local files (.py, .md). Anthropic has no UI — skills uploaded via code. Explore all paths."

- [x] **E1. Google code-defined agent reads/runs local .py + .md — CONFIRMED (PASS).** Mounted `spec.md` + `solution.py` (with a deliberate `mul` bug) via inline sources at `/workspace/`. Agent read both, executed the code (code_execution steps), caught the bug, gave the one-line fix. Report: `outputs/phase-explore-google.md`, data `data/explore_codefiles_*.json`.
- [~] **E2. Secrets/egress proxy — INCONCLUSIVE.** `base_environment.network.allowlist=[{domain,transform:{header:value}}]` is ACCEPTED at registration on the dev endpoint, but the injected header did NOT appear in httpbin's echo -> could not confirm egress injection firsthand on the free dev API (documented mechanism likely Enterprise/Vertex). Report stands as "field accepted, injection unconfirmed on dev."
- [x] **E3. Anthropic LOCAL code-defined agent — CONFIRMED (PASS).** `scripts/anthropic_local_agent.py` (Messages API + tool-use loop). Agent defined in code; skill loaded from `anthropic-local/.claude/skills/bug-finder/SKILL.md` VIA CODE (no UI); ran locally; used tools (list_files/read_file/run_python, 2 tool-use rounds) on local `solution.py`+`spec.md`; caught the mul bug in the skill's exact `BUG:/FIX:` format. Data: `data/anthropic_local_agent.json`. NOTE: `claude -p --dangerously-skip-permissions` was BLOCKED by the auto-mode classifier (won't bypass); the Messages-API loop is the clean equivalent.
- [x] **E4. Same definition, two deploy targets — CONFIRMED.** One pattern (instructions + SKILL.md + local .py/.md, find-the-bug task) ran on BOTH Google's managed cloud sandbox (E1) and Claude locally (E3). Both read the files, ran the code, applied the code-defined skill, caught the bug. The runtime is a deploy target; the agent definition (markdown + files) is the portable asset.
- [~] **MCP status:** Google dev endpoint REJECTS `tools[]` `mcp_server` (400) -> Vertex/Enterprise-only there; dev-side MCP = `npx add-mcp` into the sandbox. Anthropic: MCP is code-definable via `.mcp.json` / Agent SDK / Messages API MCP connector. NOT live-tested here (no MCP server wired) -> next step if wanted.
- [x] **E5. Post updated** with the code-defined + deploy-vs-local + portability findings. See `drafts/managed-agents-experiment.md` + `outputs/post.md`.

---

## G. Candidate posts (DIVERGE — do not collapse into one)
Net-new from this research = the GOOGLE + cross-vendor + translator + security angles. The Claude/Anthropic agent story is ALREADY covered by `Articles/Claude-Subagents/draft-v3.md` (subagent primitive, 5 flavors, Agent SDK, skills, claude -p, MCP inheritance, runtime-verified) — DO NOT redo it; cite it.

| # | Post | Thesis | Lens | Data status |
|---|------|--------|------|-------------|
| A | **Google Managed Agents: the cost of renting the loop** | letting Google run the loop = ~5x cost, ~4x latency vs self-orchestrating the same model | L1 | HAVE (225 runs). Strongest data-experiment post. |
| B | **The agent is the repo, not the runtime** | the code-defined definition is the portable asset; runtime is a swappable deploy target | L10 | HAVE + drafted (article-x.md). Cross-vendor tentpole. Cite subagents piece for Claude depth. |
| C | **Secrets never touch the sandbox** | Google injects creds at an egress proxy; the agent uses a token it can't read | L5 | PARTIAL (field accepted on dev; injection unconfirmed — needs Vertex). Sharp short post or fold into B. |
| D | **Google managed (cloud) vs Google local (Gemini CLI)** | two Google agent modes; when each wins | L1/L10 | NEEDS TEST (Gemini CLI local not run yet). |
| E | **Google's managed bet vs Anthropic's code-only bet** | managed+UI vs code+no-UI — same idea, different wager | L10/L16 | HAVE enough (cites subagents piece). Thought-piece riding A/B. |
| F | **The Gemini<->Anthropic translator** | one agent definition runs on both providers via a thin shim | L10 | HAVE (4/4). Short builder note or fold into B. |

ALREADY COVERED (don't redo): Claude subagents / Agent SDK / skills / claude -p / local Claude agents -> `Articles/Claude-Subagents/draft-v3.md`.

Recommendation: A and B are the two strongest standalone posts, both fully backed by data, and should stay SEPARATE (compounding > compression). C needs Vertex to confirm or reframe as "the design." D needs one Gemini-CLI test. E/F ride A/B.

## L. WHAT "MANAGED AGENT" MEANS PER PROVIDER (2026-06-01, Pawel pressure-test) — they are NOT the same kind of thing
- **Google Managed Agents** (`/v1beta/interactions`): a real managed runtime. Provisions a Linux sandbox, runs the FULL multi-step loop server-side, returns the trace. Persistent agents/sessions/environments. = "give us your agent, we run the whole loop."
- **Anthropic Managed Agents** (`/v1/agents` + `/v1/sessions`): a real managed runtime. Container per session, full loop on Anthropic's orchestration layer, streamed events. Plus graders/vaults/subagents. = "give us your agent, we run the whole loop."
- **OpenAI: NO equivalent.** Responses API is request/response — YOU orchestrate the loop ("developers are now responsible for orchestrating tool use"); they host the model + individual tools (code_interpreter runs server-side). The Assistants API (closest to a managed loop, stateful hosted runs) is **DEPRECATED, sunset 2026-08-26**. AgentKit = a visual builder/toolkit, not a hosted agent-loop runtime. So my OpenAI "managed" arm = one Responses call + hosted code tool, NOT a managed agent. Relabel honestly: OpenAI's bet is "you keep the loop, we host the tools + give you a builder."
- **Implication for the comparison:** Google/Anthropic columns = managed-loop-vs-local (apples to apples). OpenAI column = hosted-code-tool-vs-local-code-exec (the 16x is the code_interpreter container fee, not a managed-runtime tax). The post + README + infographic must explain this per-provider, not call all three "managed."
- Sources: openai.com/index/new-tools-for-building-agents, introducing-agentkit; Assistants API deprecation (sunset 2026-08-26).

## K. CLEAN comparison (2026-06-01) — managed loop vs TRUE local loop, confounds fixed. AUTHORITATIVE.
Report: `outputs/comparison-clean.md`. Data: `data/clean_runs.jsonl`. Code: `scripts/clean_comparison.py` (real native-tool-use local loops for all 3 providers; code runs locally via subprocess = free; managed = provider sandbox loop). N=8/cell, same T1-T3, 100% pass everywhere. ~150 runs, $0.50.

| Provider | managed $ | local $ | cost (managed/local) | managed lat | local lat | latency |
|---|---|---|---|---|---|---|
| Anthropic (haiku-4.5) | $0.00250 | $0.00225 | **1.1x (~equal)** | 6.52s | 2.38s | managed 2.7x slower |
| Google (gemini-3.5-flash) | $0.00416 | $0.00079 | **5.3x** | 12.14s | 2.54s | managed 4.8x slower |
| OpenAI (gpt-5-mini) | $0.01054 | $0.00066 | **16.1x** | 3.74s | 5.61s | managed 0.7x (FASTER) |

**Corrected findings (supersede §I/§J cost claims):**
- Running the loop LOCALLY is cheaper on all three (local tools = free CPU; you skip the provider sandbox/loop overhead). Magnitude varies hugely by vendor architecture: Anthropic ~neutral (its auto-caching keeps managed close), Google ~5x flat tax, OpenAI ~16x (driven by the code_interpreter container fee ~$0.03/use, NOT tokens — without it, OpenAI managed ~= local on tokens).
- Latency does NOT track cost: Google + Anthropic managed are 2.7-4.8x SLOWER; OpenAI managed is FASTER (0.7x) because its code runs server-side vs my local round-trips.
- RETRACTED: "Anthropic managed is cheaper / inverse of Google" (was a caching artifact). Corrected: ~cost-neutral, slower.
- Caveats: cost = token estimate at tier rates + OpenAI $0.03 container estimate; managed auto-caches its big context, local context is lean (reported as-used); local loops chose to run code more often (100% vs managed 33-67%); tasks within capability (100% pass), so no reliability-cliff data.

## J. CONFOUNDS (2026-06-01, Pawel pressure-test) — the cost comparison is NOT clean. Do not publish the cost claims as-is.
1. **Caching not held constant -> "Anthropic managed cheaper" is an ARTIFACT.** Prompt caching is the standard 5-min ephemeral cache, a PLAIN-API feature. Verified firsthand: a plain `messages.create` with `cache_control` on a >2048-token (Haiku minimum) system prefix -> call 1 cache_create=5202, calls 2-3 cache_read=5202, input 15. The managed agent auto-caches its big context; my DIY single arm set no cache_control, so it paid full price. Fair comparison (both cache) -> the 0.5x advantage largely disappears. RETRACT "Anthropic managed is cheaper / inverse of Google." Likely the Google 5x is also caching-confounded (not controlled).
2. **"Managed" means different things per provider.** Google/Anthropic: a distinct loop-running API (interactions/sessions). OpenAI: the SAME Responses API with the code tool on vs off -> not a managed-runtime distinction at all; the +$0.03 is the code_interpreter container fee (a tool cost).
3. **Nothing ran LOCALLY.** All "single/hosted" arms executed code in the PROVIDER's remote sandbox (gemini code_execution / anthropic code_execution / openai code_interpreter). The only true local agent built = `anthropic_local_agent.py` (subprocess tools + Messages API for inference). So I never tested managed-vs-local cost.
**Implication:** the cost-comparison post is confounded; don't ship the strong cost claims. Clean experiment = managed loop (server) vs TRUE local loop (my orchestration + local tool exec), caching held constant, same tasks. The DEFENSIBLE findings remain: portability (one definition runs on Google+Anthropic+local), reliability 100% within-capability, and the qualitative architectures. Cost nuances -> repo caveats, not headlines.

## I. 3-PROVIDER managed-vs-non-managed comparison (2026-06-01) — DONE (see § J: cost claims confounded)
Same T1-T3 battery on each vendor's flash tier, hosted/managed loop vs single call. 100% pass everywhere (within-capability). Report: `outputs/comparison-3provider.md`. Data: `data/{runs_full,anthropic_runs,openai_runs}.jsonl`.
- **Google (gemini-3.5-flash):** Managed Agents = **~5x cost, ~4x latency** vs single generateContent. Always-on sandbox loop.
- **Anthropic (claude-haiku-4-5):** Managed Agents = **~0.5x cost (CHEAPER), ~1.4-2.4x latency** vs single messages+code-exec. Prompt-cache reuse across sessions makes managed cheaper per run. INVERSE of Google. Total $0.28.
- **OpenAI (gpt-5-mini):** No separate managed layer; Responses API invokes code_interpreter per-task. Code fired only on T1 (compute) -> +~$0.03 container surcharge ($0.031 vs $0.005), but 2x faster than plain reasoning (17s vs 35s). T2/T3: no code, hosted==single. Total $0.45.
- **HEADLINE:** "managed agents cost more" is FALSE as a universal. Three different architectural bets (Google always-on tax / Anthropic cache-cheaper / OpenAI conditional-sandbox). The portable part across all three = the agent definition; the runtime + its cost curve is a per-vendor deploy choice.
- Runners: `scripts/anthropic_managed_runner.py`, `scripts/openai_runner.py` (+ smoke `anthropic_managed_smoke.py`). Anthropic SDK 0.105.2 (beta.agents/sessions/environments), OpenAI SDK 2.38 (responses+code_interpreter).
- **Facade implication:** 3 real hosted targets now, each with a different cost model -> the "define once, deploy anywhere" facade is more compelling (the L10 thesis as a tool).

## H. CORRECTION (2026-06-01): Anthropic Managed Agents EXISTS and is hosted
Earlier claim "no Anthropic managed cloud runtime" was WRONG (Pawel corrected). Verified firsthand: created a real agent `POST /v1/agents` (beta header `managed-agents-2026-04-01`) -> 200, `agent_019DCHcQsj119e3qYFXmzsSy`. It is a HOSTED runtime ("provisions a container per session; the agent loop runs on Anthropic's orchestration layer").
- **Define:** `POST /v1/agents` {name, model, system, tools[], skills[], mcp_servers[], multiagent}. Run: `POST /v1/sessions` (agent ID) then stream events until terminated.
- **Scorers: YES** — success criteria + a separate grader in its own context, rubric-graded iterate loop (`user.define_outcome`). Lighthouse demo iterated 62->78->96. **This is the feature Google's Managed Agents lacks inline** (answers Pawel's original "scorers for anthropic managed agents" question).
- **MCP: native** (`mcp_servers[]`); **secrets via Vaults API** (`vault_ids`, auto-refresh OAuth; agent declares `{type,name,url}` only, no auth inline).
- **Subagents: YES** (`multiagent: {type:"coordinator", agents:[...]}`). Google: not exposed.
- **NOT yet run firsthand:** a full session (SSE streaming) -> no Anthropic-managed cost/latency data yet. The `anthropic` python SDK is NOT installed.
- **Net:** Anthropic Managed Agents is MORE feature-complete than Google's (scorers, vaults, multiagent, native MCP). Google's bet is the sandbox + code-exec + Gemini. Both hosted.
- **Impact on posts:** Post A ("managed agents cost") is GOOGLE-ONLY data -> currently a Google-specific claim, not a cross-vendor one. To be fair it needs Anthropic-managed measured too. The Google-vs-Anthropic *managed-vs-managed* comparison is now a real, strong post.

## F. The cross-target comparison (code-defined agents)
| Primitive | Google Managed Agents | Anthropic (local, code) |
|---|---|---|
| Define in code | yes (`agents.create` spec + sources) | yes (script + CLAUDE.md + SKILL.md) |
| UI | yes (AI Studio) | **no UI — code only** (Pawel's point, confirmed) |
| Where it runs | Google cloud sandbox (managed loop) | your machine (you run the loop) |
| Custom skills (SKILL.md) | CONFIRMED | CONFIRMED |
| Reads local .py/.md | CONFIRMED (mounted via sources) | CONFIRMED (direct local FS) |
| MCP | Vertex tools[] mcp_server / dev `npx add-mcp` (not live-tested) | `.mcp.json`/SDK/connector (not live-tested) |
| Secrets | egress-proxy header transform ("never in sandbox"); dev field accepted, injection unconfirmed | your env / MCP config (you hold them) |
| Subagents | not exposed | yes (Task/subagents) |
| Cost/latency vs self-orchestrated same model | ~5x cost, ~4x latency (you rent the loop) | you own the loop |
