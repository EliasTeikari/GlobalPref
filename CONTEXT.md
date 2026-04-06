# GlobalPref — Build Context for Claude Code

## What You're Building

A Python script that collects a cross-cultural LLM response preference dataset using the Rapidata SDK. The dataset is called **GlobalPref**. It will be published open-source on HuggingFace under Elias's personal HuggingFace account.

The dataset captures how people from different countries prefer responses from different frontier LLMs to the same prompts — filling a real gap in RLHF research, where almost all preference data is US/English-centric.

---

## Dataset Design

### Task Type

Pairwise comparison (binary preference). Annotators see two LLM responses side-by-side to the same prompt and pick which one they prefer. This maps to Rapidata's `create_compare_order`.

### Prompts

Generate **50 prompts** across 5 categories (10 per category):

- **Factual** — e.g., "What causes inflation?" / "How do vaccines work?"
- **Practical advice** — e.g., "How should I negotiate a salary?" / "What's the best way to learn a new skill?"
- **Creative** — e.g., "Write a short poem about rain." / "Describe your ideal city."
- **Opinion/values** — e.g., "What makes a good leader?" / "What does success mean?"
- **Everyday tasks** — e.g., "How do I apologize to a friend?" / "How do I stay motivated?"

Keep prompts culturally neutral in phrasing — avoid US-specific references, idioms, or cultural assumptions. Store prompts in `prompts.json`.

### Model Response Pairs

For each prompt, generate two responses — one from each model:

- **Response A**: `gpt-5` via OpenAI API
- **Response B**: `claude-sonnet-4-6` via Anthropic API

**Critical**: Use an identical, neutral system prompt for both models:

```
"You are a helpful assistant."
```

Do not use any additional instructions, persona prompts, or style guidance. This ensures preference signal reflects the model's underlying character, not prompt engineering. Document this clearly in the data card.

**Do not** use same-model/different-prompt comparisons — the cross-model question (GPT-5 vs Claude Sonnet) is what researchers actually want answered and what makes this dataset publishable.

Store generated pairs in `response_pairs.json`.

### Countries & Languages

Target these 5 countries using Rapidata's country filter:

- 🇺🇸 United States (`US`)
- 🇩🇪 Germany (`DE`)
- 🇯🇵 Japan (`JP`)
- 🇧🇷 Brazil (`BR`)
- 🇮🇳 India (`IN`)

Run a **separate Rapidata order per country** so results are cleanly tagged by country in the output.

### Responses Per Datapoint

`responses_per_datapoint = 25` — gives statistically meaningful signal per prompt per country.

### Total Annotations

50 prompts × 25 responses × 5 countries = **6,250 preference votes**

---

## Rapidata SDK — Key Details

> **Important for Claude Code:** Use the `/rapidata` plugin to get up-to-date guidance on how to write Rapidata SDK code. Run it before writing any Rapidata-related code in this project.

Install: `pip install rapidata`

Authentication: browser-based login via `RapidataClient()` on first run. Stores credentials locally.

```python
from rapidata import RapidataClient, RapidataFilters

rapi = RapidataClient()
```

### Creating a Compare Order

```python
order = rapi.order.create_compare_order(
    name="GlobalPref - US",
    instruction="Which response is more helpful and accurate?",
    datapoints=datapoints,          # list of (text_a, text_b) pairs
    responses_per_datapoint=25,
    filters=RapidataFilters(country=["US"])
).run()
```

`datapoints` format for text comparison: list of dicts or tuples — check `docs.rapidata.ai` for the exact schema for text-vs-text compare orders.

### Retrieving Results

After `.run()`, poll until complete, then export:

```python
results = order.get_results()
```

Results include per-datapoint vote counts. Parse into win rates (% preferring Response A vs B) per prompt per country.

---

## File Structure to Build

```
globalpref/
├── generate_prompts.py       # Writes prompts.json (50 prompts across 5 categories)
├── generate_responses.py     # Calls OpenAI + Anthropic APIs, writes response_pairs.json
├── run_rapidata.py           # Creates and runs one Rapidata order per country
├── collect_results.py        # Polls orders for completion, exports raw results → results_raw.json
├── build_dataset.py          # Merges everything into final HuggingFace-ready dataset → dataset.jsonl
├── prompts.json              # Generated: 50 prompts
├── response_pairs.json       # Generated: 50 prompt → (response_a, response_b) pairs
├── order_ids.json            # Generated: Rapidata order IDs per country (for recovery if needed)
├── results_raw.json          # Generated: raw Rapidata output per country
├── dataset.jsonl             # Final dataset (one record per datapoint per country)
└── README.md                 # HuggingFace data card (see schema below)
```

---

## Final Dataset Schema

Each line of `dataset.jsonl`:

```json
{
    "prompt_id": "factual_003",
    "category": "factual",
    "prompt": "What causes inflation?",
    "response_a": "...",
    "response_b": "...",
    "model_a": "gpt-5",
    "model_b": "claude-sonnet-4-6",
    "system_prompt": "You are a helpful assistant.",
    "country": "JP",
    "votes_a": 14,
    "votes_b": 11,
    "total_votes": 25,
    "preferred": "a",
    "win_rate_a": 0.56
}
```

---

## README / HuggingFace Data Card Template

The `README.md` should follow HuggingFace dataset card format and include:

- **Dataset name**: GlobalPref
- **Description**: Cross-cultural human preference dataset comparing GPT-5 and Claude Sonnet across 5 countries. 6,250 pairwise preference votes across 50 prompts (US, DE, JP, BR, IN), collected via the Rapidata human feedback platform.
- **Use cases**: Cross-model preference analysis, multilingual RLHF, cultural alignment research, reward model training
- **Collection method**: Rapidata pairwise comparison orders with country-level demographic filtering. ~25 annotators per prompt per country. Both models given identical system prompt: "You are a helpful assistant."
- **License**: CC BY 4.0
- **Citation**: Include a BibTeX entry
- **Columns**: document all fields from dataset schema above
- **Limitations**: Annotator demographics within each country are not further controlled. Model identity is not hidden from the dataset (though annotators do not see model names during annotation). Results reflect model character and training, not prompt engineering.

---

## Environment Variables Needed

```
OPENAI_API_KEY=...        # for GPT-5 response generation
ANTHROPIC_API_KEY=...     # for Claude Sonnet response generation
```

Rapidata auth is handled interactively on first run (browser login).

---

## Build Order

Run scripts in this order:

1. `generate_prompts.py` — no API calls, just writes prompts.json
2. `generate_responses.py` — calls OpenAI and Anthropic APIs, writes response_pairs.json
3. `run_rapidata.py` — creates 5 orders (one per country), saves order IDs to `order_ids.json`
4. `collect_results.py` — polls orders until complete, writes results_raw.json
5. `build_dataset.py` — merges all data into dataset.jsonl and generates README.md

---

## Notes & Constraints

- Rapidata orders max out at 100 datapoints — 50 prompts per order is well within limit
- The instruction string passed to Rapidata must be ≤ 400 characters
- Preview each order with `order.preview()` before `.run()` to sanity-check the annotator UI
- If a country order fails, it can be rerun independently — order IDs are saved to `order_ids.json`
- Keep response pairs under ~300 words each so they fit cleanly in the annotator UI
- Randomize which model is Response A vs B across prompts to eliminate position bias — record the mapping in response_pairs.json
