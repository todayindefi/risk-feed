# TID Risk Score Feed

Machine-readable **asset** and **protocol** risk scores, published as static JSON on GitHub Pages. Read-only, public, no auth.

## Endpoints

| URL | Contents |
|-----|----------|
| [`feed.json`](https://todayindefi.github.io/risk-feed/feed.json) | Manifest — read this **first** |
| [`assets.json`](https://todayindefi.github.io/risk-feed/assets.json) | 126 asset score records |
| [`protocols.json`](https://todayindefi.github.io/risk-feed/protocols.json) | 58 protocol score records |

Base URL: `https://todayindefi.github.io/risk-feed/`

## Score scale

All scores are floats on a **0–10 scale where higher = lower risk (safer)**. Never invert.

- **Assets:** `volatility_score`, `liquidity_score`, `structural_score`, `redemption_score`, `issuer_score` (optional — present only where a distinct issuer is assessed), `overall_score`
- **Protocols:** `contract_score`, `economic_score`, `project_score`, `overall_score`

## Update cadence

The feed is rebuilt on a **daily cron at 01:45 UTC** and republished **only when score data changed**. On days with no change, nothing is pushed and `generated_at` does **not** advance — this is expected, not staleness. Treat the data as valid until the next build regardless of `generated_at` age. Don't poll more than once/hour; cache keyed on `generated_at`.

## Manifest (`feed.json`)

```json
{
  "schema_version": "1.0",
  "generated_at": "2026-07-22T01:45:00Z",
  "score_scale": "0-10, higher = lower risk",
  "endpoints": { "assets": "…/assets.json", "protocols": "…/protocols.json" },
  "assets":    { "count": 126, "score_fields": { … } },
  "protocols": { "count": 58,  "score_fields": { … } }
}
```

Pin your client to `schema_version` (current `"1.0"`). If it changes, do **not** assume field compatibility — surface it.

## Record shapes

Each file is `{ "<assets|protocols>": [ …records… ], "alias_map": { … } }`.

**Asset record:**
```json
{
  "asset": "Aave V3", "slug": "aave-v3", "chains": ["eth", "arbitrum"],
  "category": "vault-share", "assessment_type": "light|full",
  "date": "YYYY-MM-DD", "last_verified": "YYYY-MM-DD",
  "issuer": "...", "peg_mechanism": "...", "underlying_assets": ["USDe"],
  "yield_bearing": true, "market_cap_approx": "...", "audited_reserves": "...",
  "default_scores": { "volatility_score": 0, "liquidity_score": 0, "structural_score": 0,
                      "redemption_score": 0, "issuer_score": 0, "overall_score": 0 },
  "chain_scores": { "eth": { "…same keys…": 0 } },
  "report_file": "assets/<slug>.md"
}
```

**Protocol record:** same envelope; scores are `contract_score`, `economic_score`, `project_score`, `overall_score`; plus `tvl_gross`, `tvl_net`, `tvl_borrowed`.

## Resolution rules

1. **Look up by name/ticker** → map through `alias_map` first (e.g. `"aave" → "aave-v3"`), then index the array by `slug`. Build `{r["slug"]: r for r in data[key]}` once.
2. **Per-chain scores** → use `chain_scores[chain]` when a specific chain matters; fall back to `default_scores` when the chain is absent or unspecified. `default_scores` applies to all chains unless overridden.
3. `issuer_score` is **optional** (assets only, where applicable).
4. `report_file` is a path inside a **private** source repo — **not** a fetchable URL. Do not attempt to retrieve it.

## Reference client (Python)

```python
import requests
BASE = "https://todayindefi.github.io/risk-feed"

meta = requests.get(f"{BASE}/feed.json").json()          # check schema_version / generated_at
pr   = requests.get(f"{BASE}/protocols.json").json()
by_slug = {p["slug"]: p for p in pr["protocols"]}
alias   = pr["alias_map"]

def protocol_score(name, chain=None, field="overall_score"):
    rec = by_slug[alias.get(name.lower(), name.lower())]
    scores = (rec.get("chain_scores") or {}).get(chain) or rec["default_scores"]
    return scores[field]

protocol_score("aave", "eth")   # -> 8.0
```
