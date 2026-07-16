# Art for Breakfast — Commission & Inventory Tracing System

A Python application that simulates and traces the full lifecycle of a bespoke ceramic tile commission — from client intake through inventory check, production briefing, logistics calculation, and confirmation. Built with OpenTelemetry and visualised in Dash0.

---

## What it does

Art for Breakfast commissions hand-painted ceramic tile murals with Portuguese artisan workshops. Each commission involves multiple interdependent decisions: tile specifications, artist and workshop availability, raw material inventory, weight-based logistics, and pricing. This application models that full flow as a traced transaction so every step is observable, measurable, and debuggable.

---

## The business flow (as a trace)

```
commission-booking (parent span)
├── validate-client
│     └── client type, contact, history
├── define-product-specifications
│     ├── product type (mural / single panel / set)
│     ├── tile dimensions and count
│     ├── thickness and indoor/outdoor flag
│     ├── UV protection requirement
│     └── total weight calculation
├── check-inventory
│     ├── raw tile stock vs. required quantity
│     └── restock lead time if partial
├── check-artist-availability
│     └── next available slot, production start date
├── check-mounting-company-availability
│     └── workshop that produces ceramic backing
├── calculate-logistics
│     ├── total weight (determines if specialist logistics needed)
│     ├── packing cost
│     └── delivery cost flag (>30kg = specialist required)
├── calculate-price
│     └── production + materials + logistics + packing
└── send-confirmation
      └── timeline, price breakdown, artist assigned
```

Each step is an OpenTelemetry span. The parent span captures the full commission. Child spans capture each decision point independently — so if inventory fails or the mounting company is unavailable, the failure appears exactly where it happened in the Dash0 waterfall.

---

## Span attributes (what Dash0 captures per commission)

| Span | Key attributes traced |
|---|---|
| `validate-client` | `client.type`, `client.id`, `client.returning` |
| `define-product-specifications` | `product.type`, `tile.count`, `tile.dimensions`, `tile.thickness_mm`, `tile.uv_protected`, `tile.location` (indoor/outdoor), `weight.total_kg` |
| `check-inventory` | `inventory.required`, `inventory.available`, `inventory.status`, `restock.lead_weeks` |
| `check-artist-availability` | `artist.name`, `artist.available_from`, `artist.style` |
| `check-mounting-company` | `workshop.name`, `workshop.available`, `workshop.lead_weeks` |
| `calculate-logistics` | `logistics.specialist_required` (bool, triggered at >30kg), `logistics.cost_eur`, `packing.cost_eur` |
| `calculate-price` | `price.production`, `price.materials`, `price.logistics`, `price.total` |
| `send-confirmation` | `confirmation.sent`, `confirmation.timeline_weeks` |

---

## What Dash0 shows

Each run produces a trace waterfall showing:

- Duration of each step — the LLM scoring spans are P99 (slowest), expected given API call latency
- Success vs. failure — payment has a 10% simulated failure rate to demonstrate red vs. green spans
- Full attribute set per span — clicking any span shows the full business context
- Span events — key decisions (e.g. inventory partial, specialist logistics triggered) recorded as events on the span

> **Screenshot:** [add Dash0 trace waterfall screenshot here]

---

## Why this was built

The goal was to understand OpenTelemetry from a product perspective — not just what it monitors, but what product decisions it makes observable that would otherwise be invisible. The insight: tracing a business workflow (not just a technical one) means you can answer questions like "how often does inventory availability block a commission?" or "what's the p99 latency for our artist availability check?" from production data, not from manual tracking.

This directly informed the approach to the Dash0 product evaluation — understanding the tool by using it on a real problem, not a tutorial example.

---

## Tech stack

- Python 3.14
- OpenTelemetry SDK (`opentelemetry-api`, `opentelemetry-sdk`, `opentelemetry-exporter-otlp-proto-http`)
- Dash0 (OTel-native observability, OTLP HTTP ingestion at `europe-west4.gcp.dash0.com`)
- VS Code

---

*Part of the [Art for Breakfast](https://artforbreakfast.com) agentic automation stack.*
