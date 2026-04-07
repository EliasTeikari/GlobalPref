# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

GlobalPref is a minimal single-notebook demo (`globalpref.ipynb`) that compares GPT-5 vs Claude Sonnet 4.6 responses using the Rapidata human feedback platform. 5 prompts, 15 votes each — just enough to learn the Rapidata workflow end-to-end.

## Setup

```bash
pip install openai anthropic rapidata python-dotenv
```

API keys go in `.env` (already in `.gitignore`):
```
OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-ant-...
```

## Notebook Pipeline (run cells sequentially)

1. **Setup & Imports** — loads `.env`, validates keys
2. **Generate Prompts** — 5 hardcoded prompts (1 per category) → `prompts.json`
3. **Generate Responses** — calls GPT-5 + Claude Sonnet 4.6, randomizes A/B position → `response_pairs.json`
4. **Create Rapidata Order** — one global compare job via `create_compare_job_definition()` with `data_type="text"` → `order_id.json` (requires Rapidata browser login on first run)
5. **Collect Results** — `get_job_by_id()` + `display_progress_bar()` + `get_results()` → `results_raw.json`
6. **Build Dataset** — tallies all votes per prompt → `dataset.jsonl`

## Key Design Decisions

- **Position randomization** with `random.Random(42)` — which model is response_a vs response_b is randomized per prompt to eliminate position bias.
- OpenAI GPT-5 uses `max_completion_tokens` (not `max_tokens`). Anthropic uses `max_tokens`.
- Anthropic model ID is the alias `claude-sonnet-4-6` (no date suffix).
- `responses_per_datapoint=15` — small enough to complete quickly for a demo.

## Rapidata SDK API (current)

```python
from rapidata import RapidataClient
client = RapidataClient()
audience = client.audience.get_audience_by_id("global")
job_def = client.job.create_compare_job_definition(name=..., instruction=..., datapoints=[[a,b],...], contexts=[...], data_type="text", responses_per_datapoint=15)
job = audience.assign_job(job_def)
# Later:
job = client.job.get_job_by_id(job_id)
results = job.get_results()
```

## Generated Files (all gitignored)

`prompts.json`, `response_pairs.json`, `order_id.json`, `results_raw.json`, `dataset.jsonl`
