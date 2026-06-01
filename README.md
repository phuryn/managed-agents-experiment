# Managed Agents Experiment

A firsthand, reproducible comparison of **"managed agents" vs running the agent loop yourself**, across **Google, Anthropic, and OpenAI** — plus a Gemini↔Anthropic format translator and a portability demo (one agent definition running on multiple runtimes).

Run 2026-06-01 by [Paweł Huryn](https://www.productcompass.pm). Every claim is backed by the run logs in [`data/`](data/). Inspect the artifacts, don't trust screenshots.

> **TL;DR** — Running the agent loop locally was cheaper than the managed runtime on all three providers, but by wildly different margins (≈equal / ~5× / ~16×), and "managed agent" isn't even the same product across vendors. Google and Anthropic sell a managed-agent runtime; OpenAI doesn't (its Responses API loops over its own hosted tools, but there's no standalone managed-agent product).

## First: "managed agent" means three different things

This is the part most comparisons skip. The providers are not offering the same thing.

| Provider | What "managed" actually is |
|---|---|
| **Google** — Managed Agents (`/v1beta/interactions`) | A real managed runtime. You register an agent; Google provisions a Linux sandbox and runs the **entire multi-step loop** server-side (reason → call tool → run code → observe → repeat → answer). One request, they do every step. Persistent agents, sessions, environments. |
| **Anthropic** — Managed Agents (`/v1/agents` + `/v1/sessions`) | Also a real managed runtime. A container per session, the full loop on Anthropic's orchestration layer, streamed events. Plus success-criteria graders, a Vaults secrets API, and subagents. |
| **OpenAI** — *no standalone product* | The **Responses API** *does* run a server-side loop over its **hosted tools** (a `code_interpreter` sandbox iterates until done in a managed container). What's missing vs the other two is a **persistent managed-agent product** (named agent + session + environment objects). The stateful **Assistants API** (closest to that) is **deprecated, sunset 2026-08-26**. **AgentKit** is a visual builder, not a run-my-agent runtime. |

So when this repo says "managed" for OpenAI, it means *one Responses call with a hosted code tool* — not a managed agent loop. That distinction drives the cost result below.

## Headline result — managed vs a true local loop (N=8, same tasks)

Fair test: the provider's managed runtime vs a loop on my own machine (model via API, **code executes locally for free**), same tasks, caching read as-used.

| Provider (model) | What "managed" is here | Cost: managed vs local | Latency | Why |
|---|---|---|---|---|
| Anthropic (claude-haiku-4-5) | full managed loop + container | **~1.1× (about equal)** | managed 2.7× slower | caching keeps managed close |
| Google (gemini-3.5-flash) | full managed loop + cloud sandbox | **~5.3× more** | managed 4.8× slower | flat tax for the always-on sandbox |
| OpenAI (gpt-5-mini) | Responses loops over its hosted code tool; no persistent product | **~16× more** | faster (0.7×) | the `code_interpreter` container fee, not tokens (hosted-tool economics, not managed-agent) |

Plus: a single agent **definition** (instructions + `SKILL.md` + files) ran on Google's managed cloud sandbox **and** on Claude locally and caught the same bug; one `AGENTS.md` format rule was obeyed 100% across Google Managed Agents, Gemini direct, and Claude. The portable thing is the definition; the runtime is a deploy choice.

**Caveats (important):** tasks were within model capability, so every cell hit 100% pass — this is a cost/latency/architecture finding, **not** a reliability-cliff result. Cost is a token-based estimate at published tier rates (plus OpenAI's `code_interpreter` container fee and Anthropic cache pricing). Managed runtimes auto-cache a large context; the local loop's context is lean (reported as-used). Full log + corrections in [`FINDINGS.md`](FINDINGS.md).

## Setup

```bash
pip install -r requirements.txt          # anthropic + openai SDKs; Gemini uses stdlib
cp .env.example .env                      # then add GEMINI_API_KEY, ANTHROPIC_API_KEY, OPENAI_API_KEY
```

`GEMINI_API_KEY` needs Managed Agents preview access; `ANTHROPIC_API_KEY` needs the managed-agents beta. Scripts read `.env` from the repo root automatically.

## Run it

```bash
python scripts/clean_comparison.py --n 8           # THE headline: managed loop vs local loop, all 3 providers
python scripts/runner.py --tasks T1,T2,T3,T4 --arms A,B,C --n 25   # Google battery + portability task
python scripts/anthropic_managed_runner.py --n 12  # Anthropic managed vs single call
python scripts/openai_runner.py --n 12             # OpenAI Responses+code_interpreter vs single call
python scripts/translator.py                       # Gemini <-> Anthropic translator, live tests
python scripts/probe_advanced.py                   # Google Managed Agents: skills / MCP / secrets / subagents
```

Each script writes structured logs to `data/` and a report to `outputs/`. Full per-script guide: [`CLAUDE.md`](CLAUDE.md).

## Layout

```
scripts/          runners, provider clients, the translator
data/             raw per-run logs (.jsonl) + API responses (the receipts)
outputs/          human-readable reports — start with comparison-clean.md
anthropic-local/  a code-defined LOCAL Claude agent (CLAUDE.md + .claude/skills + files)
FINDINGS.md       durable running log of every test, result, and correction
```

## License

MIT — see [LICENSE](LICENSE).
