# Scrape Polymarket Odds: Build a Prediction Market Tracker (and Find Arbitrage)

Prediction markets move billions and their odds are data gold — elections, sports, crypto, weather. Polymarket's public API is rate-limited and thin on history/order books. This tutorial shows the Apify-actor pipeline: live odds, whale trades, and **Kalshi-vs-Polymarket arbitrage detection**.

> 🤖 **Try it live** — chat with a Poe bot that generates the exact pipeline for the markets you track: [Lead List Generator](https://poe.com/leadlistgenerator) · [Job Postings Miner](https://poe.com/jobpostingsminer) · [Amazon Product Miner](https://poe.com/amazonproductminer) · [SERP Rank Tracker](https://poe.com/serpranktracker) · [Social Lead Finder](https://poe.com/socialleadfinder)

## The stack

- **[Apify](https://console.apify.com/sign-up?fpr=dx06p)** — dedicated prediction-market actors, some with 800+ users. Free monthly credits included.
- **Python** — odds → CSV, alerts, dashboards.

## Step 1 — Get your API token

1. [Create an Apify account](https://console.apify.com/sign-up?fpr=dx06p) (free credits included)
2. Console → Settings → Integrations → copy your API token
3. `export APIFY_TOKEN=your_token`

## Step 2 — Pull live market odds

```bash
curl -X POST "https://api.apify.com/v2/acts/scrapemint~polymarket-market-monitor/run-sync-get-dataset-items?token=$APIFY_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"category": "politics", "maxItems": 100}'
```

Fields per market: `question`, `outcomes` (YES/NO prices), `volume`, `liquidity`, `endDate`, `url`.

## Step 3 — Detect mispriced markets

```python
from apify_client import ApifyClient

client = ApifyClient("APIFY_TOKEN")
run = client.actor("scrapemint/polymarket-market-monitor").call(run_input={
    "category": "crypto", "maxItems": 200
})

for m in client.dataset(run["defaultDatasetId"]).iterate_items():
    prices = [o.get("price", 0) for o in m.get("outcomes", [])]
    # binary market where YES + NO != $1 = inefficiency
    if prices and abs(sum(prices) - 1.0) > 0.03:
        print("GAP:", m["question"][:70], "| prices:", prices,
              "| vol:", m.get("volume"))
```

## The arbitrage play

[`congism/polymarket-kalshi-arb-finder`](https://apify.com/congism/polymarket-kalshi-arb-finder) (800+ users — proven demand) compares the same event on both platforms. If Kalshi prices YES at $0.52 while Polymarket prices NO at $0.45, buying both locks ~3% risk-free. The actor outputs the pairs — you execute.

## Whale watching

[`jy-labs/polymarket-whale-tracker`](https://apify.com/jy-labs/polymarket-whale-tracker) surfaces the largest positions. When a wallet that called the last three elections moves $50k, that's a signal worth having *before* the odds move.

## Useful actors

| Actor | Job |
|---|---|
| `scrapemint/polymarket-market-monitor` | Live odds by category (242 users) |
| `congism/polymarket-kalshi-arb-finder` | Cross-platform arbitrage pairs (835 users) |
| `jy-labs/polymarket-whale-tracker` | Largest positions / whale wallets |
| `parsebird/polymarket-scraper` | Trader leaderboard data |
| `logiover/polymarket-events-odds-ai-scraper` | Events + odds API output |

## Who uses this

- **Traders** — arb alerts, momentum on whale entries
- **Journalists/analysts** — prediction odds beat polls for breaking stories
- **Data sellers** — cleaned odds history = a sellable dataset

## Automation

Schedule the monitor hourly in Apify Console → diff snapshots → odds-movement alerts to Slack/Telegram. Cost: a few cents per day on [free credits](https://console.apify.com/sign-up?fpr=dx06p).

## More resources

- [apify-skills](https://github.com/g4dr/apify-skills) — Agent Skills that let Claude Code / Cursor run these pipelines from a prompt: `npx skills add g4dr/apify-skills`
- [awesome-web-scraping](https://github.com/g4dr/awesome-web-scraping) — curated scraping tools, proxies and actors
