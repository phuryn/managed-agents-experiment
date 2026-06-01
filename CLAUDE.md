# Managed Agents Experiment

A firsthand, reproducible comparison of **managed agent runtimes vs running the agent loop yourself**, across **Google, Anthropic, and OpenAI** — plus a Gemini↔Anthropic format translator and a portability demo (one agent definition running on multiple runtimes).

Run 2026-06-01 by Paweł Huryn. Every claim is backed by the run logs in `data/`. Inspect the artifacts, don't trust screenshots.

## What it shows

- **Managed vs local loop (the clean, fair comparison):** [`outputs/comparison-clean.md`](outputs/comparison-clean.md). Same tasks, each provider's managed runtime vs a true local loop (model via API, code runs locally on your machine). Running the loop locally was cheaper on all three; the margin (about equal / ~5x / ~16x) and the latency direction are vendor-specific.
- **Portability:** one agent definition (instructions + `SKILL.md` + files) runs on Google Managed Agents (cloud sandbox) and on Claude locally; one `AGENTS.md` format rule obeyed 100% across Google Managed Agents, Gemini direct, and Claude.
- **Translator:** [`scripts/translator.py`](scripts/translator.py) — a Gemini ↔ Anthropic Messages API adapter so the same agent definition runs on either provider.
- **Advanced primitives (Google Managed Agents):** custom skills, MCP, secrets via egress proxy, subagents — what's actually supported ([`outputs/phase-advanced.md`](outputs/phase-advanced.md)).

## Setup

1. Python 3.11+.
2. `pip install -r requirements.txt` (the Anthropic and OpenAI SDKs; Gemini uses only the standard library).
3. Copy `.env.example` to `.env` and add your keys:
   - `GEMINI_API_KEY` — Gemini API, with Managed Agents preview access
   - `ANTHROPIC_API_KEY` — with the managed-agents beta enabled
   - `OPENAI_API_KEY`

   Scripts read `.env` from the repo root automatically (they walk up from `scripts/`).

## How to run

| Command | What it does |
|---|---|
| `python scripts/gemini_client.py` | Verify your Gemini key + generation. |
| `python scripts/clean_comparison.py --n 8` | **The headline comparison:** managed loop vs true local loop, all three providers → `outputs/comparison-clean.md`. |
| `python scripts/runner.py --tasks T1,T2,T3,T4 --arms A,B,C --n 25` | Google: Managed Agents vs self-orchestrated + portability task (T4). |
| `python scripts/anthropic_managed_runner.py --n 12` | Anthropic: Managed Agents vs single call. |
| `python scripts/openai_runner.py --n 12` | OpenAI: Responses + code_interpreter vs single call. |
| `python scripts/translator.py` | Gemini ↔ Anthropic translator, live round-trip tests. |
| `python scripts/probe_advanced.py` | Google Managed Agents advanced primitives (skills / MCP / secrets / subagents). |
| `python scripts/explore_codedef.py` | A Google agent reading + running local `.py`/`.md` files mounted as sources. |
| `python scripts/anthropic_local_agent.py` | A code-defined **local** Claude agent (skill loaded from `SKILL.md`, local file tools). |
| `python scripts/analyze.py --tag full` | Build a results report from a run log. |

Costs are small — the full set runs to a couple of dollars on the cheap model tiers. Each script writes structured logs to `data/` and a report to `outputs/`.

## Headline result (managed vs local loop, N=8, same tasks)

| Provider | cost: managed vs local | latency | why |
|---|---|---|---|
| Anthropic (claude-haiku-4-5) | ~1.1x (about equal) | managed 2.7x slower | caching keeps managed close |
| Google (gemini-3.5-flash) | ~5.3x more | managed 4.8x slower | flat tax for the always-on hosted sandbox |
| OpenAI (gpt-5-mini) | ~16x more | managed faster (0.7x) | it's the code-sandbox container fee, not tokens |

**Caveats (important):** tasks were within model capability, so every cell hit 100% pass — this is a cost/latency/architecture finding, not a reliability-cliff result. Cost is a token-based estimate at published tier rates (plus the OpenAI code_interpreter container fee and Anthropic cache pricing). Managed runtimes auto-cache a large context; the local loop's context is lean — reported as-used. Full log, corrections, and the methodology are in [`FINDINGS.md`](FINDINGS.md).

## Layout

```
scripts/          runners, provider clients, the translator
data/             raw per-run logs (.jsonl) + API responses (the receipts)
outputs/          human-readable reports + writeups
anthropic-local/  a code-defined LOCAL Claude agent (CLAUDE.md + .claude/skills + files)
FINDINGS.md       durable running log of every test, result, and correction
```

## License

MIT. See [LICENSE](LICENSE).
