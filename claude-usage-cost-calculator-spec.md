# Spec: Claude subscription usage -> API cost -> Anthropic cost calculator

Prices and figures are as of 2026-11-09, gathered from public web sources. Values marked ASSUMPTION are not confirmed. Make all inputs in sections 1-3 editable constants in the UI.

## Goal

Given token counts from Claude Code `/usage` (per model: input, output, cache read, cache write), a plan tier, and a number of days, compute:

1. API-equivalent cost (what the usage would cost at list API prices)
2. Estimated Anthropic cost to serve it (API cost x (1 - margin))
3. Ratios against the subscription fee paid

## 1. Plan prices (USD/month)

| Plan | Price |
|---|---|
| Pro | 20 |
| Max 5x | 100 |
| Max 20x | 200 |

Prorate the fee for partial periods: `fee = price * days / 30`.

## 2. API rates (USD per 1M tokens)

| Model | Input | Output | Cache read | Cache write 5 min | Cache write 1 h |
|---|---|---|---|---|---|
| Opus 5.5 | 4.00 | 20.00 | 0.20 | 5.00 | 8.00 |
| Sonnet 5.5 | 2.00 | 10.00 | 0.10 (see note A) | 2.50 | 4.00 |
| Haiku 5.5 | 0.10 | 0.50 | ASSUMPTION 0.01 (10% of input) | ASSUMPTION 0.125 (1.25x input) | ASSUMPTION 0.20 (2x input) |
| Fable 5.1 | 10.00 | 50.00 | 0.25 | ASSUMPTION 12.50 (1.25x input) | ASSUMPTION 20.00 (2x input) |
| Opus 5 and 4-series (legacy) | 5.00 | 25.00 | 0.50 | 6.25 | ASSUMPTION 10.00 (2x input) |

Note A: Sonnet 5.5 cache read was 0.20 at launch. One source reports a cut to 0.10 from 2026-10-07, and several recent pricing pages list 0.10. Provide a toggle between 0.10 and 0.20, default 0.10.

Modifiers (optional toggles):
- Batch API: 50% off input and output. Not confirmed how it combines with cache pricing. Interactive Claude Code usage is not billed this way.
- Opus 5.5 fast mode: 2x price (input 8, output 40).
- Thinking tokens are billed as output tokens. Claude Code `/usage` output counts should already include them, so do not add them again.

Cache write TTL: default to 5 min. Let the user switch to 1 h.

## 3. Margin assumptions

Anthropic cost = API cost x (1 - margin). These are company-wide figures from press and analyst reports, not per-user or API-specific figures. Treat as scenarios.

| Scenario | Margin | Basis |
|---|---|---|
| Low | 0.40 | 2025 gross margin for paying users (The Information, Jan 2026) |
| Low-mid | 0.50 | Round figure between 40% and mid-60s |
| Mid (default) | 0.65-0.70 | Overall gross margin, mid-60s by mid-2026 (SemiAnalysis) |
| High | 0.85 | Pure inference margin H1 2026, excludes training (third-party reading of an S-1) |
| Ceiling | 0.88 | SemiAnalysis estimate for the inference business |
| With training | 0.45 | "Compute gross margin" including training (same S-1 analysis) |

Show all scenarios side by side. Default to Mid.

## 4. Formulas

```python
# USD per 1M tokens: (input, output, cache_read, cache_write_5m, cache_write_1h)
RATES = {
    "opus-5.5":   (4.00, 20.00, 0.20, 5.00, 8.00),
    "sonnet-5.5": (2.00, 10.00, 0.10, 2.50, 4.00),
    # Haiku, Fable, legacy Opus: see section 2, mark ASSUMPTION values in the UI
}

def api_cost(model, inp, out, cache_read, cache_write, ttl="5m"):
    i, o, cr, cw5, cw1 = RATES[model]
    cw = cw5 if ttl == "5m" else cw1
    return (inp * i + out * o + cache_read * cr + cache_write * cw) / 1e6

def anthropic_cost(api, margin):
    return api * (1 - margin)

def plan_fee(price, days):
    return price * days / 30

def ratios(api, margin, fee):
    cost = anthropic_cost(api, margin)
    return {"api_vs_fee": api / fee, "anthropic_cost_vs_fee": cost / fee}
```

Inputs are raw token counts (not millions). Sum `api_cost` across all models used.

## 5. Test case

18 days of usage, Pro plan ($20/month, so fee = 12.00).

| Model | Input | Output | Cache read | Cache write |
|---|---|---|---|---|
| Opus 5.5 | 104,000 | 98,000,000 | 10,800,000,000 | 305,000,000 |
| Sonnet 5.5 | 39,500 | 14,600,000 | 3,100,000,000 | 45,000,000 |

Expected results (5 min TTL, Sonnet cache read 0.10):
- Opus 5.5 API cost: 5,645.42
- Sonnet 5.5 API cost: 568.58
- Total API-equivalent: 6,214.00
- Anthropic cost at 0.50 / 0.70 / 0.88 margin: about 3,107 / 1,864 / 746
- With Sonnet cache read at 0.20: total 6,524.00
- With 1 h cache writes: about 7,196.50

## 6. Caveats to show in the UI

- "API cost" is the retail value of the usage at list price, not what Anthropic pays. API prices include Anthropic's margin.
- Margins are company-wide averages. Per-user serving cost could be lower, because much of the capacity is a fixed cost.
- Subscription limits are not published, so the calculator only converts the token counts entered. It does not tell what a plan allows.
- Usage billed separately (extra usage, API credits) is not subsidized and should be excluded from the subsidy comparison.
- Cache reads and output usually make up most of the cost in agentic use. Small changes in the output share swing the total a lot.
- Rates change. Show a "prices as of" date and keep all rates in one config object.

## 7. Sources

- Opus 5.5 pricing: https://www.developersdigest.tech/models/claude-opus-5-5
- Sonnet 5.5 pricing: https://www.developersdigest.tech/models/claude-sonnet-5-5
- Sonnet 5.5 rate card: https://www.eesel.ai/blog/claude-sonnet-5-5-pricing
- Sonnet 5.5 cache read cut: https://smartscope.blog/en/blog/claude-sonnet-5-5-cache-read-price-cut-2026/
- Opus 5.5 and Fable cache rates: https://www.orcarouter.ai/blog/claude-opus-5-5-pricing
- Opus 5.5 release: https://thenewstack.io/?p=22837560
- Gross margin forecast (The Information): https://www.itiger.com/news/1154519858
- SemiAnalysis inference margins: https://cryptobriefing.com/anthropic-inference-margins-semianalysis-88-percent/
- S-1 margin analysis: https://yage.ai/share/anthropic-s1-six-month-shelf-life-en-20261030.html
