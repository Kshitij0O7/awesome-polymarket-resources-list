# awesome-bitquery-polymarket-v2

Resources for working with **Polymarket** prediction-market data through **Bitquery's V2 GraphQL API**, WebSocket subscriptions, and Kafka.

I was building a small Polymarket odds dashboard and went pretty deep on Bitquery's `PredictionTrades` / `PredictionManagements` / `PredictionSettlements` cubes. Putting the queries that actually worked here so others don't have to dig through five doc pages to piece them together.

> One thing that bit me: Polymarket data lives on the `realtime` dataset on Polygon, which only retains roughly the **last 7 days**. If you want longer history for backtesting, you have to ask sales — it's not in the default plan.

---

## Contents

- [Background (CTF / Polygon)](#background-ctf--polygon)
- [Endpoint & auth](#endpoint--auth)
- [Cubes](#cubes)
- [Trades & prices](#trades--prices)
- [Markets & lifecycle](#markets--lifecycle)
- [Settlements & positions](#settlements--positions)
- [Wallet activity](#wallet-activity)
- [Top markets by volume](#top-markets-by-volume)
- [Themed markets](#themed-markets)
- [CTF Exchange (raw events)](#ctf-exchange-raw-events)
- [Streaming](#streaming)
- [SDKs / tools](#sdks--tools)
- [Links](#links)

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

Generate a token at [account.bitquery.io/user/api_v2/access_tokens](https://account.bitquery.io/user/api_v2/access_tokens?utm_source=github&utm_medium=readme&utm_campaign=awesomelist), pass as `Authorization: Bearer <token>`. Same token works for HTTP and WebSocket. Kafka is separate — contact support if you need it.

```bash
curl -X POST https://streaming.bitquery.io/graphql \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d @- <<'EOF'
{"query":"query { EVM(network: matic) { PredictionTrades(limit: {count: 5}, orderBy: {descending: Transaction_Time}, where: {Trade: {Prediction: {Marketplace: {ProtocolName: {is: \"polymarket\"}}}}}) { Transaction { Hash Time } Trade { Prediction { Question { Title } Outcome { Label } } OutcomeTrade { Buyer Seller Amount CollateralAmount } } } } }"}
EOF
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

### [Latest Polymarket Trades](https://ide.bitquery.io/latest-polymarket-trades)

This resource provides us with the latest trades that occured on Polymarket, which can be useful for trading algorithms and portfolio managers.

### [Latest Odds for Every Active Polymarket](https://ide.bitquery.io/Latest-odds-for-every-polymarket)

This resource provides the latest odds for each Polymarket, which enables technical traders with accurate real time price info.

### [Whale Trade Stream](https://ide.bitquery.io/whale-trade-stream)

This resource tracks trades larger than $10k USD, which is essentially a threshold assumed for whale wallet transactions.

### [Largest Trades in Last 7 Days](https://ide.bitquery.io/Top-100-trades-in-the-past-7-days)

This resource provides the top 100 largest trades on Polymarket for the interval of past 7 days. This could be of great use for market analysis.

### [Top Buyers and Sellers by Volume for Last 5 Days](https://ide.bitquery.io/Top-buyers-and-sellers-on-polymarket)

This resource provides the top buyers and sellers for Polymarket over the past 5 days period, which could prove to be a great metric when running market analysis.

### [Trade count for a wallet](https://ide.bitquery.io/trade-counts-for-a-wallet-on-polymarket)

This resource provides the tally for Polymarket trades by a wallet, which is a great metric to have for portfolio managers and auditors.

---

## Markets & lifecycle

### All market events (Created / Resolved)

```graphql

```

`Question.MarketId` matches `https://gamma-api.polymarket.com/markets/{MarketId}` so you can join with Polymarket's own metadata API for stuff Bitquery doesn't index (descriptions, tags, etc.).

### Filter markets by Asset ID, Condition ID, or slug/keyword

```graphql
query MarketsByAssetId($assetIds: [String!]) {
  EVM(network: matic) {
    PredictionManagements(
      where: {
        Management: {
          Prediction: {
            Marketplace: {ProtocolName: {is: "polymarket"}}
            OutcomeToken: {AssetId: {in: $assetIds}}
          }
        }
      }
    ) {
      Management {
        Prediction {
          Question { MarketId Title }
          Condition { Id }
          OutcomeToken { AssetId SmartContract }
        }
      }
    }
  }
}
```

Swap the inner filter for `Condition: {Id: {in: $conditionIds}}` or `Question: {Title: {includesCaseInsensitive: $keyword}}` to filter by condition or slug instead.

---

## Settlements & positions

```graphql
query {
  EVM(network: matic) {
    PredictionSettlements(
      where: {Settlement: {Prediction: {Marketplace: {ProtocolName: {is: "polymarket"}}}}}
      limit: {count: 50}
      orderBy: {descending: Block_Time}
    ) {
      Settlement {
        EventType  # "Split" | "Merge" | "Redemption"
        Amounts { CollateralAmount CollateralAmountInUSD }
        Prediction { Question { MarketId Title } }
      }
      Transaction { Hash From }
    }
  }
}
```

---

## Wallet activity

Aggregate stats for a single wallet over the last 5 hours.

```graphql
query MyQuery($trader: String) {
  EVM(network: matic) {
    PredictionTrades(
      where: {
        TransactionStatus: { Success: true }
        any: [
          { Trade: { OutcomeTrade: { Buyer: { is: $trader } } } }
          { Trade: { OutcomeTrade: { Seller: { is: $trader } } } }
        ]
        Block: { Time: { since_relative: { hours_ago: 5 } } }
      }
    ) {
      Total_Outcomes_traded: count
      Total_Outcome_Amount: sum(of: Trade_OutcomeTrade_Amount)
      Total_Collateral_Amount: sum(of: Trade_OutcomeTrade_CollateralAmount)
      Total_Markets: count(distinct: Trade_Prediction_Question_MarketId)
    }
  }
}
```

---

## Top markets by volume

```graphql
query questionsByVolume($time_ago: DateTime) {
  EVM(network: matic) {
    PredictionTrades(
      where: {
        TransactionStatus: {Success: true}
        Block: {Time: {since: $time_ago}}
        Trade: {Prediction: {Marketplace: {ProtocolName: {is: "polymarket"}}}}
      }
      limit: {count: 100}
      orderBy: {descendingByField: "sumBuyAndSell"}
      limitBy: {by: Trade_Prediction_Question_Id}
    ) {
      Trade { Prediction { Question { Id Image Title CreatedAt } } }
      buyUSD: sum(
        of: Trade_OutcomeTrade_CollateralAmountInUSD
        if: {Trade: {OutcomeTrade: {IsOutcomeBuy: true}}}
      )
      sellUSD: sum(
        of: Trade_OutcomeTrade_CollateralAmountInUSD
        if: {Trade: {OutcomeTrade: {IsOutcomeBuy: false}}}
      )
      sumBuyAndSell: calculate(expression: "$buyUSD + $sellUSD")
    }
  }
}
```

The `calculate(expression: ...)` field is handy — saves doing the addition client-side.

---

## Themed markets

Polymarket has clusters of similar markets you can filter for. Pattern is mostly `Question.Title` substring or `ResolutionSource` host.

### Bitcoin Up or Down

```graphql
subscription {
  EVM(network: matic) {
    PredictionTrades(
      where: {
        Trade: {
          Prediction: {
            Marketplace: {ProtocolName: {is: "polymarket"}}
            Question: {Title: {includes: "Bitcoin Up or Down"}}
          }
        }
      }
    ) {
      Block { Time }
      Trade {
        OutcomeTrade { Buyer Seller CollateralAmountInUSD Price IsOutcomeBuy }
        Prediction { Question { Title MarketId } Outcome { Label } }
      }
      Transaction { Hash }
    }
  }
}
```

### Sports / cricket / esports

Cricket markets resolve via espncricinfo.com, so filter on `ResolutionSource`:

```graphql
query questionByLiquidity($time_ago: Int!, $limit: Int!) {
  EVM(network: matic) {
    PredictionSettlements(
      where: {
        Block: {Time: {since_relative: {hours_ago: $time_ago}}}
        Settlement: {Prediction: {Question: {ResolutionSource: {includes: "espncricinfo.com"}}}}
      }
      limit: {count: $limit}
      orderBy: {descendingByField: "position"}
      limitBy: {by: Settlement_Prediction_Question_Id}
    ) {
      Settlement {
        Prediction { Question { Image MarketId Title Id CreatedAt ResolutionSource } }
      }
      split: sum(of: Settlement_Amounts_CollateralAmountInUSD if: {Settlement: {EventType: {is: "Split"}}})
      merge: sum(of: Settlement_Amounts_CollateralAmountInUSD if: {Settlement: {EventType: {is: "Merge"}}})
      position: calculate(expression: "$split - $merge")
    }
  }
}
```

### Commodities (Gold / Oil)

Same pattern, different `Question.Title` or `ResolutionSource` filter. See [docs.bitquery.io/docs/examples/polymarket-api/polymarket-commodity-api](https://docs.bitquery.io/docs/examples/polymarket-api/polymarket-commodity-api/?utm_source=github&utm_medium=readme&utm_campaign=awesomelist).

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
- [Bitquery IDE](https://ide.bitquery.io/?utm_source=github&utm_medium=readme&utm_campaign=awesomelist) — Run/share queries.
- [Polymarket Gamma API](https://gamma-api.polymarket.com/markets) — Polymarket's own metadata API. Good to pair with Bitquery (Bitquery has on-chain data; Gamma has descriptions, tags, slugs).

---

## Links

- Polymarket cookbook: [docs.bitquery.io/docs/category/polymarket-api](https://docs.bitquery.io/docs/category/polymarket-api/?utm_source=github&utm_medium=readme&utm_campaign=awesomelist)
- Prediction markets index: [docs.bitquery.io/docs/category/prediction-markets](https://docs.bitquery.io/docs/category/prediction-markets/?utm_source=github&utm_medium=readme&utm_campaign=awesomelist)
- Telegram: [t.me/bloxy_info](https://t.me/bloxy_info)
- GitHub: [github.com/bitquery](https://github.com/bitquery)

PRs welcome.

---

## License

[CC0](https://creativecommons.org/publicdomain/zero/1.0/)
