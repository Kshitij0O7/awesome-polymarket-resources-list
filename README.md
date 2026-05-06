# Awesome Polymarket Data API

Resources for working with **Polymarket** prediction-market data through **Bitquery's V2 GraphQL API**, WebSocket subscriptions, and Kafka.

I was building a small Polymarket odds dashboard and went pretty deep on Bitquery's GraphQL APIs. Putting the queries that actually worked here so others don't have to dig through five doc pages to piece them together.

> One thing that bit me: Polymarket data lives on the `realtime` dataset on Polygon, which only retains roughly the **last 7 days**.

---

## Background (CTF / Polygon)

Quick refresher in case it's useful:

- Polymarket runs on Polygon, with USDC as collateral.
- Markets use the **Conditional Token Framework (CTF)**: each market has a `conditionId`, and outcomes are ERC-1155 tokens identified by `assetId`.
- Trade flow on-chain: **Created** (`PredictionManagements`) → **Buy/Sell** (`PredictionTrades`) → **Split / Merge / Redemption** (`PredictionSettlements`).
- Bitquery parses all of this — you don't have to decode CTF events manually.

---

## Endpoint & auth

V2 endpoint: `https://streaming.bitquery.io/graphql`

Generate a token at [account.bitquery.io/user/api_v2/access_tokens](https://account.bitquery.io/user/api_v2/access_tokens?utm_source=github&utm_medium=readme&utm_campaign=awesomelist_polymarket), pass as `Authorization: Bearer <token>`. Same token works for HTTP and WebSocket. Kafka is separate — contact support if you need it.

```bash
curl --location 'https://streaming.bitquery.io/graphql' \
--header 'Content-Type: application/json' \
--header 'Authorization: Bearer <Your Access Token>' \
--data '{"query":"query questionByLiquidity($time_ago: Int!, $limit: Int!) {\n  EVM(network: matic) {\n    PredictionSettlements(\n      where: {\n        Block: {Time: {since_relative: {hours_ago: $time_ago}}}\n        Settlement: {Prediction: {Question: {ResolutionSource: {includes: \"espncricinfo.com\"}}}}\n      }\n      limit: {count: $limit}\n      orderBy: {descendingByField: \"position\"}\n      limitBy: {by: Settlement_Prediction_Question_Id}\n    ) {\n      Settlement {\n        Prediction { Question { Image MarketId Title Id CreatedAt ResolutionSource } }\n      }\n      split: sum(of: Settlement_Amounts_CollateralAmountInUSD if: {Settlement: {EventType: {is: \"Split\"}}})\n      merge: sum(of: Settlement_Amounts_CollateralAmountInUSD if: {Settlement: {EventType: {is: \"Merge\"}}})\n      position: calculate(expression: \"$split - $merge\")\n    }\n  }\n}","variables":"{\n  \"time_ago\": 24,\n  \"limit\": 100\n}"}'
```

---

## Cubes

Three cubes do most of the work, all under `EVM(network: matic, dataset: realtime)`:

| Cube | Returns | Common filters |
|---|---|---|
| `PredictionManagements` | Market lifecycle (`Created`, `Resolved`) | `Marketplace.ProtocolName: "polymarket"`, `Condition.Id`, `Question.Title` |
| `PredictionTrades` | Buy/sell of outcome tokens (price, USDC amount, taker/maker) | `OutcomeTrade.IsOutcomeBuy`, `Buyer/Seller`, `CollateralAmountInUSD` |
| `PredictionSettlements` | Splits / merges / redemptions | `EventType: "Split" / "Merge" / "Redemption"` |

`Marketplace.ProtocolName: "polymarket"` filter is what scopes a query to Polymarket specifically (vs other prediction markets that may get added to the same cubes later).

---

## Trades & prices

### [Latest Polymarket Trades](https://ide.bitquery.io/latest-polymarket-trades/?utm_source=github&utm_medium=readme&utm_campaign=awesomelist_polymarket)

This resource provides us with the latest trades that occured on Polymarket, which can be useful for trading algorithms and portfolio managers.

### [Latest Odds for Every Active Polymarket](https://ide.bitquery.io/Latest-odds-for-every-polymarket/?utm_source=github&utm_medium=readme&utm_campaign=awesomelist_polymarket)

This resource provides the latest odds for each Polymarket, which enables technical traders with accurate real time price info.

### [Whale Trade Stream](https://ide.bitquery.io/whale-trade-stream/?utm_source=github&utm_medium=readme&utm_campaign=awesomelist_polymarket)

This resource tracks trades larger than $10k USD, which is essentially a threshold assumed for whale wallet transactions.

### [Largest Trades in Last 7 Days](https://ide.bitquery.io/Top-100-trades-in-the-past-7-days/?utm_source=github&utm_medium=readme&utm_campaign=awesomelist_polymarket)

This resource provides the top 100 largest trades on Polymarket for the interval of past 7 days. This could be of great use for market analysis.

### [Top Buyers and Sellers by Volume for Last 5 Days](https://ide.bitquery.io/Top-buyers-and-sellers-on-polymarket/?utm_source=github&utm_medium=readme&utm_campaign=awesomelist_polymarket)

This resource provides the top buyers and sellers for Polymarket over the past 5 days period, which could prove to be a great metric when running market analysis.

### [Trade count for a wallet](https://ide.bitquery.io/trade-counts-for-a-wallet-on-polymarket/?utm_source=github&utm_medium=readme&utm_campaign=awesomelist_polymarket)

This resource provides the tally for Polymarket trades by a wallet, which is a great metric to have for portfolio managers and auditors.

---

## Markets & lifecycle

### [All Market Creation and Resolution Events](https://ide.bitquery.io/Latest-market-creation-and-resolution-events/?utm_source=github&utm_medium=readme&utm_campaign=awesomelist_polymarket)

THis resource provides the latest market creation and resolution events, which can prove to be very useful for algo trading applications like sniper trading, and for portfolio tracking applications as well.


### Filter markets by Question Title](https://ide.bitquery.io/markets-by-question-title/?utm_source=github&utm_medium=readme&utm_campaign=awesomelist_polymarket)

This resource allows us to get Polymarkets which includes a specific Question, for example - "Bitcoin Up or Down". This could be very useful for people targeting a very specific kind of Polymarket.

---

## Settlements and Positions for a Wallet](https://ide.bitquery.io/settlements-and-resolutions-for-a-wallet/?utm_source=github&utm_medium=readme&utm_campaign=awesomelist_polymarket)

This event returns the latest settlements and positions for a wallet on Polymarket, making it very useful for portfolio managing applications.

---

## [Wallet Stats](https://ide.bitquery.io/trader-stats-for-past-5-hours/?utm_source=github&utm_medium=readme&utm_campaign=awesomelist_polymarket)

Aggregate stats for a single wallet over the last 5 hours to determine wallet performance.

---

## [Top markets by volume](https://ide.bitquery.io/top-markets-by-volume-for-past-24-hours/?utm_source=github&utm_medium=readme&utm_campaign=awesomelist_polymarket)

This resource helps us by sorting out the top markets based on trading volume so that users could target their funds in active markets.

---

## Themed markets

Polymarket has clusters of similar markets you can filter for. Pattern is mostly `Question.Title` substring or `ResolutionSource` host.

### [Bitcoin Up or Down](](https://ide.bitquery.io/markets-by-question-title/?utm_source=github&utm_medium=readme&utm_campaign=awesomelist_polymarket)

### [Sports / Cricket/ Football/ Esports](https://ide.bitquery.io/top-cricket-markets-by-liquidity_1/?utm_source=github&utm_medium=readme&utm_campaign=awesomelist_polymarket)

### Commodities (Gold / Oil)

Same pattern, different `Question.Title` or `ResolutionSource` filter. See [docs.bitquery.io/docs/examples/polymarket-api/polymarket-commodity-api](https://docs.bitquery.io/docs/examples/polymarket-api/polymarket-commodity-api/?utm_source=github&utm_medium=readme&utm_campaign=awesomelist_polymarket).

---

## CTF Exchange (raw events)

If the parsed cubes don't have what you need, you can drop down to raw events from the Polymarket CTF Exchange contract via `EVM.Events`. Useful for `OrderFilled`, `OrdersMatched`, `TokenRegistered`.

Price math reminder: USDC is 6 decimals, outcome tokens are 6 decimals. Price = `makerAmountFilled / takerAmountFilled` (or inverse depending on side).

Reference: [docs.bitquery.io/docs/examples/polymarket-api/polymarket-ctf-exchange](https://docs.bitquery.io/docs/examples/polymarket-api/polymarket-ctf-exchange/?utm_source=github&utm_medium=readme&utm_campaign=awesomelist)

---

## Streaming

| Channel | Endpoint / topic | Notes |
|---|---|---|
| GraphQL Subscription | `wss://streaming.bitquery.io/graphql` | Just change `query` → `subscription` in any of the queries above |
| Kafka — `matic.predictions.proto` | private creds | Raw prediction events |
| Kafka — `matic.broadcasted.predictions.proto` | private creds | Mempool prediction events |

---

## SDKs / tools

- [bitquery/polymarket-api](https://github.com/bitquery/polymarket-api) — Official npm SDK with typed wrappers around the queries above (`getNewQuestions`, `getResolvedQuestions`, `getAllTrades`, `streamAllTrades`, etc.). Quicker than writing the GraphQL yourself if you're in JS/TS:
  ```bash
  npm install polymarket-api
  ```
  ```js
  import { getAllTrades, streamAllTrades } from 'polymarket-api';
  const trades = await getAllTrades(token, 50);
  ```
- [Bitquery IDE](https://ide.bitquery.io/?utm_source=github&utm_medium=readme&utm_campaign=awesomelist_polymarket) — Run/share queries.

---

## Links

- Polymarket cookbook: [docs.bitquery.io/docs/category/polymarket-api](https://docs.bitquery.io/docs/category/polymarket-api/?utm_source=github&utm_medium=readme&utm_campaign=awesomelist_polymarket)
- Prediction markets index: [docs.bitquery.io/docs/category/prediction-markets](https://docs.bitquery.io/docs/category/prediction-markets/?utm_source=github&utm_medium=readme&utm_campaign=awesomelist_polymarket)
- Telegram: [t.me/bloxy_info](https://t.me/bloxy_info)
- GitHub: [github.com/bitquery](https://github.com/bitquery)

PRs welcome.

---

## License

[CC0](https://creativecommons.org/publicdomain/zero/1.0/)
