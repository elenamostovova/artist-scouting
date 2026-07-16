# Art for Breakfast — Artist Scouting System

An AI-powered pipeline for discovering and evaluating emerging Portuguese painters as candidates for hand-painted ceramic tile collaborations. Built as both a Python application and an n8n visual workflow.

---

## What it does

The scouting system evaluates artists from Portuguese art schools against a set of collaboration criteria specific to the Art for Breakfast ceramics platform. For each artist it:

1. Filters by medium — painters only (oil, acrylic, mixed media). Non-painters are excluded before the LLM is called.
2. Filters by graduation year — emerging artists within the last 5 years, working from Portugal.
3. Calls Claude via the Anthropic API to score each artist 1–5 across five dimensions.
4. Generates a markdown report and a structured JSON export per run.
5. Sends traces to Dash0 via OpenTelemetry so scoring behaviour can be observed over time.

---

## Scoring criteria (LLM-evaluated)

| Dimension | What it measures |
|---|---|
| Style compatibility | Abstract, expressionism, figurative — styles that translate to ceramic tile |
| Material | Oil and acrylic preferred for tile reproduction quality |
| Institutional recognition | Prizes, residencies, gallery representation (Sovereign Art Foundation PT, CPS Young Art Prize, ARCOlisboa) |
| Market traction | Pricing signal, collector interest, recent exhibition activity |
| Tile collaboration potential | Can this artist's work be translated into a painted tile format by an artisan workshop? |

---

## Architecture

### Python version (with OTel instrumentation)

```
portuguese_artists.csv
        ↓
Filter: medium = painting only
        ↓
For each painter:
    Claude API call → score 1–5 + reasoning
    OTel span emitted → sent to Dash0
        ↓
Scout_Report_PT_[date].md    ← markdown report
Scout_Data_PT_[date].json    ← structured export for database update routine
```

Each artist evaluation is wrapped in an OpenTelemetry span with GenAI semantic conventions:

- `gen_ai.input_tokens` — tokens consumed per evaluation
- `gen_ai.output_tokens` — tokens in Claude's response
- `artist.score` — the 1–5 rating
- `artist.style` — detected style category
- `artist.material` — detected material
- `artist.fit_judgment` — Claude's free-text reasoning

This means scoring behaviour is observable over time. If Claude starts scoring inconsistently across runs (eval drift), it shows up in Dash0 traces before it corrupts the artist database.

### n8n visual workflow version

Built in n8n cloud to demonstrate the same logic in a visual automation tool.

```
Manual Trigger
      ↓
Read CSV from Google Drive
      ↓
Split into items (one per artist)
      ↓
IF node: medium contains "paint"?
      → No  → Stop (excluded)
      → Yes → continue
              ↓
        Claude AI node (scoring prompt)
              ↓
        Google Sheets node (write results)
              ↓
        Gmail node (send summary email)
              ↓
        Error handler → Gmail alert if Claude call fails
```

> **Screenshot:** [add n8n workflow canvas screenshot here]

The error handling branch is intentional — it alerts via email if any Claude call fails silently, which matters in a production agentic workflow where silent failures corrupt downstream data.

---

## What the OTel instrumentation adds vs. the n8n version

| Capability | Python + Dash0 | n8n |
|---|---|---|
| See scoring decision | Yes — as span event | Yes — in sheet output |
| Token cost per artist | Yes — span attribute | No |
| Latency per evaluation | Yes — span duration | No |
| Detect eval drift over time | Yes — compare runs in Dash0 | No |
| Visual workflow editor | No | Yes |
| Non-developer usable | No | Yes |

---

## Sources used for artist data

- Faculdade de Belas-Artes Lisboa (FBAUL)
- Faculdade de Belas-Artes Porto (FBAUP)
- ESAD Caldas da Rainha
- AR.CO Lisboa
- Sovereign Art Foundation Portuguese Art Prize
- CPS Young Art Prize
- ARCOlisboa Art Fair emerging artist list

---

## Tech stack

- Python 3.14
- Anthropic Claude API (claude-sonnet-4-5)
- OpenTelemetry SDK (`opentelemetry-api`, `opentelemetry-sdk`, `opentelemetry-exporter-otlp-proto-http`)
- Dash0 (OTel-native observability, trace ingestion via OTLP HTTP)
- n8n cloud (visual workflow version)
- Google Drive, Google Sheets, Gmail (n8n integrations)

---

## Key learning from building this

The most interesting discovery was the observability gap in agentic workflows: when an AI agent reads external data and acts on it, you need to trace not just what it did but what instructions it received and why it made each decision. The Dash0 integration made that possible — each Claude call is a span with the full reasoning captured as a span event, not just the output score.

This directly maps to a real product problem: as AI workflows become more complex, the gap between "it ran" and "it behaved correctly" requires the same kind of infrastructure that distributed systems observability solved for microservices.

---

*Part of the [Art for Breakfast](https://artforbreakfast.com) agentic automation stack.*
