# Crypto Market Prices Page — System Design (Coinbase Explore)

# Important Concepts

- CDN + short-TTL caching for high-traffic read-heavy pages
- Pre-computation pipeline (sorted lists, sparklines, trending)
- Write-behind caching (Redis) for price data
- WebSocket for detail page only (not the list page)
- VWAP aggregation across multiple exchange feeds
- Server-side candle aggregation with forming candle streaming (not client-side tick aggregation)
- TimescaleDB continuous aggregates for OHLCV candles
- REST over GraphQL for simplicity — URL-based CDN caching vs. persisted query infrastructure
- Market health aggregation (market cap, BTC dominance, buy/sell ratio)

# Context

We are designing the backend for a page like **Coinbase Explore** — a high-traffic page that displays cryptocurrency market prices for thousands of assets. The page is the #1 entry point for casual users and receives billions of page views per month.

**Key observation**: The **list page** (explore page) does NOT update prices in real time via WebSocket. Prices refresh on page load or via client-side polling (~every 60 seconds). Only the **detail page** (e.g., `/price/bitcoin`) shows a live-ticking price chart via WebSocket. This dramatically simplifies the architecture — the list page is essentially a **CDN-cacheable, pre-computed snapshot**.

Core features:
1. **Asset listing table** — Shows ~400 tradeable assets (Coinbase lists ~383 assets as of 2026) with name, symbol, price, percent changes across multiple time periods (1h, 24h, 7d, 1m, 1y), 24h volume, market cap, and a sparkline chart. Prices refresh on page load or via client-side polling (~every 60 seconds).
2. **Market health overview** — Aggregate market stats: total market cap, 24h trading volume, BTC dominance %, buy/sell ratio, with trend charts for each metric.
3. **Categories & filters** — Filter by "All Assets", "Top Gainers", "Top Losers", "Trending", "New Listings", "DeFi", "Layer 1", etc. Separate filter and sort controls.
4. **Featured assets** — Dedicated BTC and ETH header sections with market cap and volume.
5. **Search** — Instant search across all assets by name or symbol.
6. **Pagination** — Default view shows top ~10 assets per page (Coinbase uses `limit: 10`) with pagination controls. Sorted by rank (market cap) by default.
7. **Price sparkline** — Mini chart for each asset (loaded lazily — can be skipped on initial load for performance).
8. **Detail page** — Single asset view with live-ticking price, OHLCV charts (1d/7d/30d/1y), stats, and description. This is the only page that uses WebSocket.

This is a **read-heavy** system. The write side is a continuous ingestion pipeline from exchanges and market data providers. The read side is serving millions of concurrent users viewing mostly the same pre-computed data.

# Requirements

## Functional Requirements

1. **List assets** — Return a paginated list of crypto assets with current price, percent changes across multiple periods (1h, 24h, 7d, 1m, 1y), volume, and market cap. Support independent filter and sort controls. Prices are a snapshot, not live-updating.
2. **Market health overview** — Aggregate market-level data: total market cap with trend chart, total trading volume, BTC dominance %, buy/sell ratio.
3. **Search** — Search assets by name or ticker symbol with sub-100ms latency.
4. **Asset detail with live price** — Get detailed data for a single asset: live price (WebSocket), 24h/7d/30d/1y charts (OHLCV candles), description, stats.
5. **Trending / Top movers** — Pre-computed lists: top gainers, top losers, trending (by user interest/volume spike), new listings.

**✍️ Note**: We are NOT designing the trading engine, order book, or wallet system. We assume upstream systems (matching engine, exchange integrations) produce a stream of price ticks that we consume.

## Non-Functional Requirements

### Scale

- **10M+ DAU** viewing the explore page.
- **~400 listed assets** tracked (Coinbase tracks ~383 listed assets; the total including unlisted is higher). Each asset gets a price tick every 1-5 seconds from exchanges.
- Read QPS for list page HTTP API: **~100K QPS** (page loads, search, pagination). Peak: **500K QPS** during market volatility events (Bitcoin crash/rally).
- Detail page WebSocket connections: **~200K-500K concurrent** (only a fraction of users drill into a specific asset at any time).
- Data freshness: list page prices can be **30-60 seconds stale** (CDN/polling). Detail page prices must be **< 3 seconds stale** (WebSocket).

### Availability

- **99.99% uptime** — Users check prices during market crashes, which is when load is highest. Downtime during volatility is catastrophic for trust.
- Multi-region deployment. No single region failure should take down the price page.
- Graceful degradation: if the detail page WebSocket fails, fall back to polling every 5 seconds. If price ingestion lag exceeds 30 seconds, show a stale-data warning banner.

### Latency

- List page API (asset list): **< 100ms p99** (CDN-served for most users).
- Search API: **< 50ms p99**.
- Detail page WebSocket price delivery: **< 3 seconds p99** from exchange tick to client screen.
- Chart data API: **< 200ms p99** (pre-computed, cached).

### Consistency

- **Eventual consistency is acceptable** — Price data is inherently approximate (different exchanges show different prices). We target "good enough" freshness, not strict consistency.
- Asset metadata (name, symbol, category tags) is rarely updated and can be strongly consistent with short cache TTL.

## Edge Cases

### Market volatility spikes (thundering herd)

When Bitcoin drops 10% in an hour, traffic spikes 5-10x on the list page. The CDN absorbs most of this. Detail page WebSocket connections surge for BTC. We must handle this gracefully without cascading failures.

### Exchange feed outage

If a price feed from a major exchange goes down, we should fall back to the last known price and display a "delayed" indicator. We should NOT show zero or null prices.

### Stale data display

If our ingestion pipeline falls behind, the client should show a "Prices may be delayed" warning. We track data freshness via a timestamp in the API response — the client checks `updated_at` and shows a warning if it's too old.

### New asset listing

When a new asset is listed, we need to add it to the asset catalog, backfill sparkline data, and make it searchable — all without downtime.

---

# API Design

## REST API

### List Assets

```
GET /v1/assets?page=1&page_size=10&sort=rank&order=asc&filter=listed

Response 200:
{
  "assets": [
    {
      "id": "bitcoin",
      "slug": "bitcoin",
      "symbol": "BTC",
      "name": "Bitcoin",
      "image_url": "https://...",
      "color": "#F7931A",
      "price_usd": "67287.395",
      "percent_change": {
        "hour": -0.45,
        "day": -2.34,
        "week": 5.12,
        "month": 12.5,
        "year": 85.3
      },
      "volume_24h_usd": "28450000000",
      "market_cap_usd": "1345000000000",
      "sparkline_day": [],             // empty if skipSparklines=true
      "is_tradable": true,
      "is_dex": false,
      "is_wallet": true,
      "rank": 1
    },
    ...
  ],
  "market_health": {
    "market_cap": { "value": "2.45T", "percent_change": 1.2, "chart": [...] },
    "trading_volume": { "value": "89.5B", "percent_change": -3.1, "chart": [...] },
    "btc_dominance": { "value": "54.2%", "percent_change": 0.3, "chart": [...] },
    "buy_sell_ratio": { "value": 1.15, "chart": [...] }
  },
  "top_gainers": ["Solana", "Chainlink", "Avalanche"],
  "btc_header": { "market_cap": "1.34T", "volume_24h": "28.4B" },
  "eth_header": { "market_cap": "420B", "volume_24h": "15.2B" },
  "total_assets": 383,
  "page": 1,
  "page_size": 10,
  "updated_at": "2026-03-07T12:00:32Z"
}
```

**Filter and sort are separate concerns** (matching Coinbase's actual pattern):

| Parameter | Values | Description |
|---|---|---|
| `filter` | `listed` (default), `gainers`, `losers`, `trending`, `new`, `defi`, `layer1` | Controls *which* assets appear |
| `sort` | `rank` (default), `price`, `market_cap`, `volume`, `percent_change` | Controls *ordering* |
| `order` | `asc` (default), `desc` | Sort direction |

**✍️ Note**: Coinbase's actual API separates filter from sort — `filter: "LISTED"` selects which assets, while `sort: "RANK"` + `order: "ASC"` controls ordering. This is more flexible than conflating them (e.g., our earlier "gainers" was both a filter AND a sort).

**✍️ Note**: Each filter+sort combination maps to a **separate pre-computed page in Redis**. When the user clicks "Gainers", the client fetches `GET /v1/assets?filter=gainers&sort=percent_change&order=desc&page=1`. This hits CDN → Redis key `asset_list:gainers:pct_desc:page:1` — a pre-built JSON blob. No on-demand sorting at request time.

The total number of pre-computed pages: ~7 filters × ~5 sort options × ~40 pages each ≈ ~1,400 Redis keys, ~15MB total. Still negligible for Redis.

**✍️ Note**: Prices are **decimal strings** (e.g., `"67287.395"`), matching Coinbase's production API. This avoids floating-point precision issues while being more human-readable than integer cents. Percent changes span **5 time periods** (hour, day, week, month, year) in a single response, so the client doesn't need separate calls for each period. The `sparkline_day` field can be empty if `skipSparklines=true` — sparklines are loaded lazily to reduce initial payload size.

**✍️ Note**: The response includes **market-level aggregate data** (market health stats, BTC/ETH headers, top gainers) alongside the asset list. This means a single request loads the entire explore page — no extra API calls needed. This is CDN-cacheable with a 30-60 second TTL.

### ❓ Why REST over GraphQL?

We use REST because the URL-based caching model is trivially CDN-friendly — each filter/sort/page combination is a unique URL that Cloudflare/Fastly can cache without any extra infrastructure. This is ideal for a read-heavy explore page where the response shape is fixed and predictable.

That said, GraphQL is a valid alternative here (and Coinbase actually uses it in production). The key challenge with GraphQL is CDN caching — GraphQL typically uses POST requests with a query body, which CDNs can't cache by default. The solution is **persisted queries**: the client sends a `sha256Hash` of a pre-registered query instead of the full query string. The server maps hash → whitelisted query. This gives you:
- **CDN cacheability** — The hash-based URL is stable and cacheable, restoring REST-like caching behavior
- **Security** — Server only executes whitelisted queries (prevents arbitrary GraphQL abuse)
- **Bandwidth savings** — No query string in every request
- **Client flexibility** — Clients can request only the fields they need (e.g., `skipSparklines: true`)

The trade-off is complexity: persisted queries require build-time query registration, hash management, and a query allowlist. For this explore page where we have a fixed number of well-defined queries, REST is simpler and achieves the same performance. The backend architecture is identical either way — pre-computed Redis lookups behind a CDN.

### Search Assets

```
GET /v1/assets/search?q=eth&limit=10

Response 200:
{
  "results": [
    { "id": "ethereum", "symbol": "ETH", "name": "Ethereum", "price_usd": 345621, "rank": 2 },
    { "id": "ethereum-classic", "symbol": "ETC", "name": "Ethereum Classic", "price_usd": 2543, "rank": 45 },
    ...
  ]
}
```

**✍️ Note**: Search matches on both `name` and `symbol` with prefix matching. With only ~400 listed assets, we can use an in-memory trie or prefix index — no need for Elasticsearch.

### Get Asset Detail

```
GET /v1/assets/{asset_id}?chart_period=7d

Response 200:
{
  "id": "bitcoin",
  "symbol": "BTC",
  "name": "Bitcoin",
  "description": "...",
  "price_usd": 6834521,
  "price_change_24h_pct": -234,
  "price_change_7d_pct": 512,
  "volume_24h_usd": 2845000000,
  "market_cap_usd": 134500000000,
  "circulating_supply": 1960000000000000,  // in smallest unit (satoshis)
  "max_supply": 2100000000000000,
  "all_time_high_usd": 7384200,
  "chart": {
    "period": "7d",
    "interval": "1h",
    "candles": [
      { "t": 1709600000, "o": 6780000, "h": 6850000, "l": 6750000, "c": 6810000, "v": 120000000 },
      ...
    ]
  },
  "updated_at": "2026-03-07T12:00:32Z"
}
```

**✍️ Note**: The detail page loads this REST API for the initial snapshot + chart data, then opens a WebSocket for live price ticks. The REST response is CDN-cacheable with a shorter TTL (~10 seconds) since detail page users expect fresher data.

### Get Trending / Top Movers

```
GET /v1/assets/trending?list=top_gainers&limit=10

Response 200:
{
  "list": "top_gainers",
  "period": "24h",
  "assets": [
    { "id": "solana", "symbol": "SOL", "price_usd": "185.32", "percent_change": { "day": 18.45 } },
    ...
  ],
  "updated_at": "2026-03-07T12:00:00Z"
}
```

**✍️ Note**: Trending lists are pre-computed every 60 seconds and cached at CDN with 60-second TTL. Lists available: `top_gainers`, `top_losers`, `trending`, `highest_volume`, `new_listings`. In Coinbase's actual API, top gainers are embedded in the main explore query response (as `topGainersHeader` containing top 3 names) rather than as a separate endpoint — this reduces round trips.

## WebSocket API (detail page live price + chart data)

The detail page WebSocket serves two types of real-time data:
1. **Live price ticker** — Current price updated every 1-5 seconds
2. **Live candle/kline data** — Pre-aggregated OHLCV candles for the chart the user is viewing

```
Connect: wss://stream.example.com/v1/prices

Client → Server (subscribe to price ticker):
{
  "action": "subscribe",
  "channel": "ticker",
  "asset_id": "bitcoin"
}

Client → Server (subscribe to candle/kline stream):
{
  "action": "subscribe",
  "channel": "candles",
  "asset_id": "bitcoin",
  "interval": "5m"
}

Server → Client (price tick, every 1-5 seconds):
{
  "type": "price_update",
  "data": {
    "id": "bitcoin",
    "symbol": "BTC",
    "price_usd": "68345.21",
    "percent_change": { "hour": -0.45, "day": -2.34 },
    "volume_24h_usd": "28450000000",
    "timestamp": 1709600123
  }
}

Server → Client (candle update, every 1-2 seconds):
{
  "type": "candle_update",
  "data": {
    "asset_id": "bitcoin",
    "interval": "5m",
    "start": 1709600100,
    "open": "68100.50",
    "high": "68350.00",
    "low": "68050.25",
    "close": "68345.21",
    "volume": "124.5",
    "is_closed": false
  }
}
```

**✍️ Note**: The `is_closed` flag is critical. When `is_closed: false`, the client updates the rightmost (forming) bar on the chart. When `is_closed: true`, the client freezes that bar and starts a new one. This is the industry-standard pattern used by Binance (`"x": true/false`), and most trading platforms.

**❓ How does real-time chart data flow work?**

This is a common interview question: *"If I'm on a 5-minute chart, does the client receive 1-second ticks and aggregate them into 5-minute candles? Or does the server send 5-minute candles directly?"*

**Answer: The server sends pre-aggregated candles. The client does NOT aggregate raw ticks.**

The flow:
1. User opens `/price/bitcoin` and selects the 5-minute chart
2. Client fetches historical 5m candles via REST (`GET /v1/assets/bitcoin?chart_period=7d&interval=5m`)
3. Client subscribes to the WebSocket candle stream: `{ channel: "candles", asset_id: "bitcoin", interval: "5m" }`
4. Server sends the **current forming candle** every ~1 second, with updated OHLCV values
5. When the 5-minute period ends, server sends one final message with `is_closed: true`
6. Next message starts a new candle with a new `start` timestamp

**❓ Why server-side aggregation, not client-side?**

| Approach | Server-side aggregation (our design) | Client-side aggregation |
|---|---|---|
| **Client complexity** | Simple — just render the candle object | Complex — must maintain OHLCV state, handle time boundaries, deal with missed ticks |
| **Bandwidth** | 1 candle object per second (~200 bytes) | 1 raw trade per trade (~100 bytes, but 10-100x more messages for active assets) |
| **Consistency** | All clients see identical candles | Different clients may compute slightly different values (missed ticks, clock drift) |
| **Resolution switching** | Client re-subscribes with new interval, server sends the right candle | Client must re-aggregate all raw ticks for the new resolution |
| **Scalability** | Server computes once, fans out to N clients | N clients all independently doing the same computation |

Server-side is the clear winner. This is how Binance (`@kline_5m` stream), Coinbase (candles channel), and all major exchanges work.

**✍️ Note**: Unlike the list page (which uses HTTP polling), the detail page uses WebSocket for a **single asset's** live price and chart data. The user is viewing `/price/bitcoin` — they only need BTC ticks and BTC candles. This keeps the fan-out problem small: each WebSocket client subscribes to exactly 1 asset.

**❓ Why WebSocket on detail page but not on list page?**

| Page | Data needed | Update frequency | Users | Best approach |
|---|---|---|---|---|
| **List page** | 50 assets | Every 30-60 seconds is fine | 10M DAU (huge) | HTTP + CDN (cacheable, simple) |
| **Detail page** | 1 asset (price + chart) | Every 1-2 seconds (live chart) | ~200-500K concurrent | WebSocket (low fan-out, single asset) |

The list page serves **the same data to all users** (top 50 by market cap) — perfect for CDN caching. Opening a WebSocket for every list page visitor would create millions of unnecessary persistent connections for data that can be 60 seconds stale. The detail page needs live ticks for the chart animation, but only for one asset per user — a much simpler fan-out problem.

---

# High Level Design

```
                          ┌────────────────────────────────────────────┐
                          │              CDN / Edge Cache              │
                          │     (list page API, trending, sparklines)  │
                          │         TTL: 30-60 seconds                 │
                          └────────────────────┬───────────────────────┘
                                               │
                                               │ cache miss
                          ┌────────────────────▼───────────────────────┐
                          │          API Gateway / Load Balancer        │
                          └──────┬─────────────────────┬───────────────┘
                                 │                     │
                    ┌────────────▼──────┐    ┌─────────▼──────────────┐
                    │   REST API Fleet  │    │  WebSocket Gateway      │
                    │   (stateless)     │    │  Fleet                  │
                    │                   │    │  (detail page only)     │
                    │ • List assets     │    │                         │
                    │ • Search          │    │ • Single-asset price    │
                    │ • Asset detail    │    │   subscription          │
                    │ • Trending lists  │    │ • Heartbeat             │
                    └────────┬─────────┘    └────────┬────────────────┘
                             │                       │ subscribe to
                             │ read from             │ per-asset channel
                    ┌────────▼─────────┐    ┌────────▼────────────────┐
                    │   Redis Cluster   │    │  Pub/Sub Broker         │
                    │   (price cache)   │    │  (Redis Pub/Sub)        │
                    │                   │    └────────┬────────────────┘
                    │ • Latest prices   │             │ publish
                    │ • Pre-computed    │             │ price ticks
                    │   list pages      │    ┌────────▼────────────────┐
                    │ • Sparkline data  │    │  Price Aggregation      │
                    │ • Trending lists  │    │  Service                │
                    └────────┬─────────┘    │                         │
                             │              │ • Normalizes feeds      │
                             │ write-behind │ • Computes VWAP         │
                             │              │ • Generates sparklines  │
                    ┌────────▼─────────┐    │ • Computes trending     │
                    │   TimescaleDB /  │    │ • Pre-builds list pages │
                    │   PostgreSQL      │    └────────┬────────────────┘
                    │                   │             │ consume
                    │ • OHLCV candles   │             │
                    │ • Historical data │    ┌────────▼────────────────┐
                    │ • Asset metadata  │    │  Exchange Feed Adapters  │
                    └──────────────────┘    │                         │
                                            │ • Binance WebSocket     │
                                            │ • Coinbase Pro WS       │
                                            │ • Kraken WS             │
                                            │ • CoinGecko API         │
                                            └─────────────────────────┘
```

### Exchange Feed Adapters

Stateful connectors that maintain WebSocket connections to major crypto exchanges (Binance, Coinbase Pro, Kraken, etc.) and REST polling to aggregators (CoinGecko, CoinMarketCap). Each adapter normalizes the exchange-specific format into a unified price tick event.

### Price Aggregation Service

The brain of the system. Consumes raw ticks from all exchange adapters, computes the **Volume-Weighted Average Price (VWAP)** across exchanges, and produces the canonical price for each asset. Also pre-computes all derived data: sorted list pages, 24h change %, sparkline arrays, trending lists. Writes everything to Redis. Publishes per-asset price ticks to Pub/Sub for WebSocket consumers.

### Redis Cluster (Price Cache + Pre-Computed Data)

The primary read store for all API requests. Stores pre-computed responses for every API endpoint so the REST API is a simple key lookup.

### Pub/Sub Broker (Redis Pub/Sub)

Distributes per-asset price ticks from the aggregation service to WebSocket gateway instances. Each gateway subscribes only to channels for assets its connected clients are viewing. With ~400 assets and ~200-500K total WebSocket connections, the fan-out per channel is manageable.

### REST API Fleet

Stateless HTTP servers. All reads go to Redis (cache hit rate > 99%). Cache misses fall through to TimescaleDB for chart data. The CDN absorbs 70%+ of list page traffic before it reaches the API fleet.

### WebSocket Gateway Fleet

Maintains WebSocket connections for **detail page users only**. Each user subscribes to one asset at a time. The gateway subscribes to the corresponding Pub/Sub channel and forwards ticks. Much smaller scale than if we had WebSocket on the list page.

### TimescaleDB / PostgreSQL

Time-series database for historical OHLCV candle data. Used for chart rendering (7d, 30d, 1y charts). Not on the hot read path for the list page. Also stores asset metadata (name, description, categories).

### CDN / Edge Cache

The first line of defense. Caches the list page API response at the edge with a 30-60 second TTL. This absorbs the thundering herd during market events. The CDN is the reason the list page doesn't need WebSocket — a 30-60 second refresh cycle is acceptable, and the CDN handles the scale.

---

# Detailed Technical Design

## 1. Price Ingestion Pipeline

```
Exchange WS Feed → Exchange Adapter → Price Aggregation Service → Redis + Pub/Sub
```

### Exchange Feed Adapters

Each adapter maintains a persistent WebSocket connection to one exchange:

- **Reconnection with exponential backoff** — Exchange feeds drop periodically. The adapter auto-reconnects and replays from the last sequence number.
- **Deduplication** — Some exchanges send duplicate ticks. We dedup by `(exchange, asset, sequence_number)`.
- **Validation** — Reject ticks with obviously wrong prices (> 50% deviation from last known price). This guards against exchange feed bugs.
- **Health monitoring** — If no tick received for 30 seconds, mark the feed as degraded and alert.

### Price Aggregation — VWAP Calculation

Different exchanges show slightly different prices for the same asset. We compute a **Volume-Weighted Average Price** across exchanges:

```
VWAP = Σ(price_i × volume_i) / Σ(volume_i)
```

For each asset, we maintain a sliding window of recent ticks from all exchanges. When a new tick arrives, we recalculate the VWAP and publish if the price changed by more than a threshold (e.g., $0.01 for BTC, $0.0001 for low-cap coins).

**✍️ Note**: VWAP is the industry standard for aggregated crypto prices. An alternative is the **median price** across exchanges, which is more robust against a single exchange showing an outlier price. CoinGecko uses a combination of both. For simplicity, we use VWAP with outlier filtering (drop ticks > 2 standard deviations from the running mean).

### Pre-Computation Pipeline

This is the key insight: **the list page serves pre-computed data, not live data**. The aggregation service runs periodic pipelines that write directly to Redis:

```
Every 10-30 seconds:
  → Update price:{asset_id} in Redis for all assets that changed
  → Compute percent changes for 5 periods (1h, 24h, 7d, 1m, 1y) per asset
  → Compute market health stats (total market cap, volume, BTC dominance, buy/sell ratio)
  → For EACH filter+sort combination:
      → Filter + sort the assets accordingly
      → Write pre-built JSON responses (including market_health, top_gainers, btc/eth headers):
          asset_list:{filter}:{sort}:{order}:page:1 ... page:N

Every 5 minutes:
  → Recompute sparkline arrays from candle data (per asset)
  → Recompute market health chart data (market cap chart, volume chart, etc.)
  → Write sparkline:{asset_id}:day to Redis for all assets

Every 60 seconds:
  → Recompute trending scores and top gainers list
```

The REST API for `GET /v1/assets?filter=gainers&sort=percent_change&order=desc&page=1` is literally:
```
return redis.get("asset_list:gainers:pct_desc:page:1")
```

No sorting, no filtering, no database query at request time. Every filter tab the user clicks maps to a different pre-computed key. This is why the API achieves < 100ms p99 — it's a single Redis GET of a pre-built JSON blob.

**✍️ Note**: The pre-computed response includes **market-level aggregate data** (market health, BTC/ETH headers, top gainers) in every page response. This matches Coinbase's actual pattern where a single API call returns everything the explore page needs. While this means some data is duplicated across pages, it eliminates the need for separate API calls and simplifies client logic.

**❓ Why pre-compute entire response pages instead of individual prices?**

If we stored individual prices and sorted/filtered at request time, each API call would:
1. Fetch ~400 individual price keys from Redis.
2. Compute percent changes across 5 time periods.
3. Filter by category and sort by the appropriate metric.
4. Slice to the requested page.
5. Assemble market health data.
6. Serialize to JSON.

At 500K QPS, that's 500K sort+filter operations per second — wasteful because the result is identical for all users within the same 30-second window. Pre-computing the sorted/filtered pages once every 10-30 seconds and storing the complete JSON response means each API request is O(1).

The tradeoff is more storage in Redis (~7 filters × ~5 sorts × ~40 pages × ~15KB = ~21MB total), which is negligible.

## 2. List Page Data Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                         List Page Flow                          │
│                                                                 │
│  User opens explore page                                        │
│       │                                                         │
│       ▼                                                         │
│  Browser → CDN edge                                             │
│       │                                                         │
│       ├── Cache HIT (within 30-60s TTL)                         │
│       │   → Return cached JSON instantly (< 20ms)               │
│       │                                                         │
│       └── Cache MISS                                            │
│           → Forward to API Gateway                              │
│           → REST API does Redis GET("asset_list:mcap:page:1")   │
│           → Return response, CDN caches it                      │
│                                                                 │
│  After 60 seconds, client JS polls the same endpoint            │
│  → CDN serves updated cache (refreshed by other requests)       │
│                                                                 │
│  User clicks "Gainers" filter tab                               │
│  → Client fetches GET /v1/assets?filter=gainers&page=1          │
│  → CDN cache (or Redis GET of pre-computed gainers page)        │
│  → Different filter = different CDN cache key = different Redis  │
│    pre-computed blob. All equally fast.                          │
└─────────────────────────────────────────────────────────────────┘
```

**✍️ Note**: The list page is architecturally similar to a news homepage or product catalog — it's a **cacheable, pre-computed snapshot** that refreshes periodically. The core scaling strategy is CDN + pre-computation, not real-time push. This is a deliberate simplification that works because users don't need second-by-second prices on a list of 50 assets — they need a directional snapshot ("BTC is around $68K, down 2% today").

### CDN Configuration

```
Cache-Control: public, max-age=30, stale-while-revalidate=60

CDN rules:
  /v1/assets?*                → Cache 30s, stale-while-revalidate 60s
  /v1/assets/trending?*       → Cache 60s, stale-while-revalidate 120s
  /v1/assets/search?*         → Cache 10s (search results change with prices)
  /v1/assets/{id}             → Cache 10s  (detail page initial load)
```

**Stale-while-revalidate** is critical: when the TTL expires, the CDN serves the stale response immediately while fetching a fresh one in the background. Users always get a fast response. This eliminates the thundering herd problem entirely — during a Bitcoin crash, millions of users hit the CDN, and the CDN makes exactly 1 request to origin per TTL window.

**❓ What about cache stampede at origin?**

Even with CDN, the origin still receives cache-miss requests (one per TTL window per edge PoP). During extreme traffic, many edge PoPs expire simultaneously. Mitigations:

1. **Request coalescing** — CDN collapses multiple origin requests for the same URL into one (all major CDNs support this).
2. **Cache warming** — The aggregation service proactively pushes updated responses to the CDN via API (Cloudflare Purge + re-cache, or Fastly's surrogate key invalidation) every 30 seconds, before the TTL expires.
3. **Origin shield** — CDN uses a mid-tier cache layer (shield PoP) between edge PoPs and origin. All edge cache misses go to the shield first, which collapses them further.

## 3. Detail Page — WebSocket Live Price & Chart Data

Only the detail page (e.g., `/price/bitcoin`) uses WebSocket for live ticks and chart data. The scope is much smaller than "all explore page visitors":

- **~200K-500K concurrent WebSocket connections** (5-10% of DAU drill into a detail page at any moment).
- Each connection subscribes to **exactly 1 asset** (price ticker + candle stream for the selected chart interval).
- Popular assets (BTC, ETH) have more subscribers; long-tail assets may have 0.

### Architecture

```
                    ┌───────────────────────────────────────┐
                    │       Price Aggregation Service        │
                    │                                        │
                    │  On each VWAP update:                  │
                    │  1. Update running candle OHLCV state  │
                    │     for every active interval (1m,5m,  │
                    │     15m,1h,4h,1d)                      │
                    │  2. PUBLISH prices:BTC { price_tick }  │
                    │  3. PUBLISH candles:BTC:5m { candle }  │
                    │     (with is_closed=false)              │
                    │  4. On period boundary:                 │
                    │     PUBLISH candles:BTC:5m { candle,   │
                    │       is_closed=true }                  │
                    │     → Reset candle state for new period │
                    └─────────────┬─────────────────────────┘
                                  │
                    ┌─────────────▼─────────────┐
                    │  Redis Pub/Sub              │
                    │                             │
                    │  prices:bitcoin    (ticker)  │
                    │  prices:ethereum   (ticker)  │
                    │  candles:bitcoin:1m          │
                    │  candles:bitcoin:5m          │
                    │  candles:bitcoin:1h          │
                    │  candles:ethereum:5m         │
                    │  ...~400 × ~6 intervals     │
                    └──┬──────┬──────┬──────┬────┘
                       │      │      │      │
              ┌────────▼┐ ┌──▼────┐ ┌▼─────┐ ┌▼────────┐
              │ WS GW 1 │ │WS GW 2│ │WS GW3│ │WS GW N  │
              │(subscr  │ │       │ │      │ │         │
              │ BTC     │ │       │ │      │ │         │
              │ ticker  │ │       │ │      │ │         │
              │ +5m     │ │       │ │      │ │         │
              │ candles)│ │       │ │      │ │         │
              └────┬────┘ └──┬────┘ └──┬───┘ └────┬────┘
                   │         │         │          │
              Clients     Clients   Clients    Clients
              viewing     viewing   viewing    viewing
              /price/btc  various   various    various
              (5m chart)
```

**Each WebSocket gateway only subscribes to channels for assets AND intervals its clients are viewing.** When a client connects and sends `subscribe: { channel: "candles", asset_id: "bitcoin", interval: "5m" }`, the gateway:
1. Checks if it's already subscribed to `candles:bitcoin:5m` on Pub/Sub.
2. If not, subscribes. If yes, just adds the client to the local subscriber list.
3. When a candle update arrives on `candles:bitcoin:5m`, the gateway forwards it to all local subscribers.
4. Client updates the rightmost bar on the chart (if `is_closed: false`) or starts a new bar (if `is_closed: true`).

### Server-Side Candle Aggregation (How the Forming Candle Works)

The Price Aggregation Service maintains **running OHLCV state** for every `(asset, interval)` combination:

```python
# Pseudocode — candle aggregation in the Price Aggregation Service
class CandleAggregator:
    def __init__(self, asset_id, interval_seconds):
        self.asset_id = asset_id
        self.interval = interval_seconds  # e.g., 300 for 5m
        self.current_candle = None
        self.reset_candle()

    def reset_candle(self):
        """Start a new candle period."""
        now = time.time()
        period_start = now - (now % self.interval)  # align to interval boundary
        self.current_candle = {
            "start": period_start,
            "open": None, "high": None, "low": None, "close": None,
            "volume": 0, "is_closed": False
        }

    def on_trade(self, price, volume, timestamp):
        """Called on each incoming trade tick after VWAP computation."""
        candle = self.current_candle

        # Check if this trade belongs to a new candle period
        if timestamp >= candle["start"] + self.interval:
            # Close the current candle
            candle["is_closed"] = True
            self.publish(candle)  # final message for this period
            self.persist(candle)  # write closed candle to TimescaleDB
            self.reset_candle()
            candle = self.current_candle

        # Update running OHLCV
        if candle["open"] is None:
            candle["open"] = price
        candle["high"] = max(candle["high"] or price, price)
        candle["low"] = min(candle["low"] or price, price)
        candle["close"] = price
        candle["volume"] += volume

    def emit(self):
        """Called every ~1 second by a timer to push the forming candle."""
        if self.current_candle["open"] is not None:
            self.publish(self.current_candle)  # is_closed=false

    def publish(self, candle):
        redis.publish(f"candles:{self.asset_id}:{self.interval}s", json.dumps(candle))

    def persist(self, candle):
        """Write closed candle to TimescaleDB for historical queries."""
        db.execute(
            "INSERT INTO price_candles (asset_id, interval, bucket_time, open, high, low, close, volume) "
            "VALUES (%s, %s, %s, %s, %s, %s, %s, %s)",
            (self.asset_id, f"{self.interval // 60}m", candle["start"],
             candle["open"], candle["high"], candle["low"], candle["close"], candle["volume"])
        )
```

**Key design decisions:**

1. **Emit cadence**: The forming candle is pushed every ~1 second (via a timer calling `emit()`), regardless of trade frequency. This gives the client smooth chart updates without overwhelming it with per-trade messages.

2. **Candle closure**: On the period boundary (e.g., every 5 minutes for a 5m chart), the service sends one final message with `is_closed: true`, then persists the closed candle to TimescaleDB.

3. **Resolution switching**: When a user switches from 5m to 1h chart, the client unsubscribes from `candles:bitcoin:5m` and subscribes to `candles:bitcoin:1h`. The gateway adjusts its Pub/Sub subscriptions accordingly. Historical candles for the new resolution are fetched via REST.

**✍️ Note**: This is exactly how Binance's `@kline_5m` stream works. Each message is a complete candle snapshot (not a diff), with `"x": false` for forming candles and `"x": true` for closed candles. Coinbase's candles channel works similarly but infers closure from a changed `start` timestamp rather than an explicit flag.

**❓ With 1,000 assets × many intervals, does the server aggregate all combinations independently?**

**No.** Two key optimizations avoid a combinatorial explosion:

### Optimization 1: Hierarchical Roll-Up

Only the **1-minute base interval** is aggregated directly from raw trade ticks. All higher intervals are **derived by rolling up** lower-resolution candles:

```
Raw trade ticks
    │
    ▼
1m candles ← aggregated from ticks (1 aggregator per asset)
    │
    ├── 5m  = roll up 5 × 1m   (max highs, min lows, first open, last close)
    ├── 15m = roll up 3 × 5m
    ├── 30m = roll up 2 × 15m
    ├── 1h  = roll up 2 × 30m
    ├── 4h  = roll up 4 × 1h
    └── 1d  = roll up 24 × 1h
```

Rolling up a 5m candle from five 1m candles is trivial:
```python
def rollup(candles_1m):
    return {
        "open":   candles_1m[0]["open"],         # first open
        "high":   max(c["high"] for c in candles_1m),  # max of highs
        "low":    min(c["low"]  for c in candles_1m),  # min of lows
        "close":  candles_1m[-1]["close"],        # last close
        "volume": sum(c["volume"] for c in candles_1m)
    }
```

So for 1,000 assets, you run **1,000 tick-level aggregators** (1m base), not 1,000 × N. The higher intervals are cheap derivations.

### Optimization 2: Lazy Pub/Sub Emission

The server maintains running OHLCV state for all `(asset, interval)` combinations — but it only **publishes to Pub/Sub** for channels that have active WebSocket subscribers. If nobody is viewing the 15m chart for asset #847, don't emit on `candles:asset847:15m`.

```python
def emit(self):
    """Only publish if there are active subscribers on this channel."""
    channel = f"candles:{self.asset_id}:{self.interval}s"
    if self.has_subscribers(channel):  # check subscriber count
        self.publish(self.current_candle)
```

The in-memory candle state is still maintained (so the server can start streaming instantly when someone subscribes), but the Pub/Sub traffic is proportional to **active viewers**, not total combinations.

### Scale math

| Metric | Count | Cost |
|---|---|---|
| Tick-level aggregators (1m base) | ~1,000 (one per asset) | Each processes ~1 tick/sec |
| Higher-interval candle state | ~1,000 assets × ~12 intervals = ~12,000 | ~100 bytes each = ~1.2 MB total RAM |
| Active Pub/Sub channels | Proportional to users viewing charts | BTC/ETH dominate; long-tail assets have 0 subscribers |
| Standard intervals (industry) | Binance: 16, Coinbase: 6 | Not 100 — exchanges offer ~6-16 standard intervals |

**✍️ Note**: The total in-memory state for all candle aggregators is ~1.2 MB. The actual Pub/Sub load depends on how many users are actively viewing charts. During peak hours, perhaps ~50 assets have active chart viewers across all intervals — that's ~300 active channels, not 12,000. The long tail of assets has zero viewers most of the time.

**❓ How big is the fan-out per asset?**

| Asset | Estimated concurrent viewers | Fan-out per tick |
|---|---|---|
| Bitcoin (BTC) | ~50K-100K | 50K-100K WebSocket pushes |
| Ethereum (ETH) | ~20K-50K | 20K-50K |
| Top 10 assets | ~5K-20K each | Moderate |
| Long-tail (rank 100+) | ~10-500 each | Trivial |

Even for BTC (the worst case), 100K pushes per tick at 1 tick/sec is well within the capacity of a distributed WebSocket gateway fleet. With 10 gateway instances, each handles ~10K BTC subscribers — easy.

**❓ Why Redis Pub/Sub and not NATS or Kafka?**

With lazy emission, the actual Pub/Sub load is much smaller than the theoretical maximum. Total defined channels: ~1,000 ticker channels + ~1,000 assets × ~12 intervals = ~13,000 channels. But with lazy emission, only channels with active subscribers emit — typically ~50-100 assets with chart viewers × ~2 intervals each ≈ **~200-500 active channels** at any time, plus ~200-400 active ticker channels. Realistic load: **~500-1,000 msgs/sec**.

For this scale, Redis Pub/Sub is the simplest choice:
- We already have Redis for the cache layer. No new infrastructure.
- Redis Pub/Sub handles ~1M messages/sec. Our load is ~500-1,000 msgs/sec — 0.1% of capacity.
- Fire-and-forget semantics are perfect for ephemeral price ticks and forming candles.
- Gateway only subscribes to channels for assets AND intervals its clients are viewing — most channels have 0 gateway subscribers.

NATS would also work and is a better choice if the gateway fleet grows beyond ~50 instances (Redis Pub/Sub doesn't scale horizontally as well). But at our current scale, Redis Pub/Sub avoids adding a new dependency.

### Connection Management

- **5-10 gateway instances** × 20K-50K connections each = 200K-500K total.
- Each connection consumes ~10-20KB of memory. 50K connections ≈ 750MB RAM per instance.
- **L4 load balancer** (NLB) distributes new connections across gateways.
- **Heartbeat**: Gateway sends ping every 30 seconds. Client responds with pong. If no pong for 90 seconds, close connection.
- **Reconnection**: Client auto-reconnects with exponential backoff. On reconnect, re-subscribe and fetch latest price via REST to fill the gap.
- If a gateway dies, ~50K clients reconnect across remaining instances within ~2 seconds. During the gap, client shows last known price.

## 4. Search

With only ~400 listed assets (~383 in Coinbase's actual data), search is simple. We do NOT need Elasticsearch.

### Approach: In-Memory Prefix Trie + Redis Sorted Set

The aggregation service maintains a **Redis sorted set** for search:

```
ZADD search_index 1 '{"id":"bitcoin","symbol":"BTC","name":"Bitcoin","price":6834521}'
ZADD search_index 2 '{"id":"ethereum","symbol":"ETH","name":"Ethereum","price":345621}'
...
```

The score is the asset's market cap rank. The API service also maintains an in-memory **trie** for prefix matching:

```
Trie:
  b → bi → bit → bitc → bitcoin (rank 1)
  b → bn → bnb (rank 4)
  e → et → eth → ethereum (rank 2)
  e → et → eth → ethereum-classic (rank 45)
```

On search request `q=eth`:
1. Traverse the trie to find all assets matching "eth" (by name or symbol).
2. Return results sorted by rank (top assets first).
3. Total latency: < 1ms (in-memory trie traversal).

The trie is rebuilt on startup from Redis and updated when assets are added/removed (rare event).

**Alternative: Elasticsearch** — Overkill for ~400 items. If we needed full-text search across descriptions or fuzzy matching, ES would make sense. For prefix matching on name/symbol, a trie is simpler and faster.

## 5. Sparkline and Chart Data

### Sparkline (24h mini chart — list page)

The 24h sparkline is an array of 24 hourly prices. Pre-computed every 5 minutes by the aggregation service:

```python
# Pseudocode
def compute_sparkline(asset_id):
    candles = timescaledb.query(
        "SELECT time_bucket('1 hour', time) AS hour, last(price, time) AS close "
        "FROM price_ticks WHERE asset_id = %s AND time > now() - interval '24 hours' "
        "GROUP BY hour ORDER BY hour",
        asset_id
    )
    return [c.close for c in candles]  # 24 data points
```

The sparkline is embedded in the list page response — no extra API call needed.

### OHLCV Candle Data (for detail page charts)

Stored in **TimescaleDB** — a PostgreSQL extension optimized for time-series data.

```sql
CREATE TABLE price_candles (
    asset_id TEXT NOT NULL,
    interval TEXT NOT NULL,        -- '1m', '5m', '1h', '1d'
    bucket_time TIMESTAMPTZ NOT NULL,
    open BIGINT,
    high BIGINT,
    low BIGINT,
    close BIGINT,
    volume BIGINT,
    PRIMARY KEY (asset_id, interval, bucket_time)
);

-- TimescaleDB hypertable with automatic partitioning
SELECT create_hypertable('price_candles', 'bucket_time');
```

**Continuous aggregates** automatically roll up 1-minute candles into 5m, 1h, and 1d candles:

```sql
CREATE MATERIALIZED VIEW candles_1h
WITH (timescaledb.continuous) AS
SELECT asset_id,
       time_bucket('1 hour', bucket_time) AS bucket_time,
       first(open, bucket_time) AS open,
       max(high) AS high,
       min(low) AS low,
       last(close, bucket_time) AS close,
       sum(volume) AS volume
FROM price_candles
WHERE interval = '1m'
GROUP BY asset_id, time_bucket('1 hour', bucket_time);
```

**✍️ Note**: TimescaleDB's continuous aggregates are incrementally updated as new data arrives — no batch job needed.

### Storage Estimate

- ~400 listed assets × 1 candle/minute × 1,440 min/day = **~576K rows/day** (1-minute candles).
- Each row ~100 bytes → **~58 MB/day** → ~21 GB/year.
- With TimescaleDB compression (typically 10-20x), raw storage is ~1-2 GB/year. Trivial.

## 6. Database Design

### Redis Cluster (Primary Read Store)

| Key pattern | Value | TTL | Update frequency |
|---|---|---|---|
| `asset_list:{filter}:{sort}:{order}:page:{n}` | Pre-built JSON response (includes market health, top gainers, btc/eth headers) | 60s | Every 10-30s |
| `price:{asset_id}` | Latest price + percent changes (5 periods) | 60s | Every 1-5s |
| `market_health` | Aggregate market stats (market cap, volume, BTC dominance, buy/sell ratio + charts) | 60s | Every 30s |
| `sparkline:{asset_id}:day` | Price data points for sparkline chart | 10min | Every 5min |
| `asset_detail:{asset_id}` | Full asset metadata + stats + color + platform flags | 5min | Every 5min |
| `search_index` | Sorted set for search | 1h | On asset catalog change |

Redis Cluster with **6 nodes** (3 primaries + 3 replicas). At peak 500K API QPS with 99%+ CDN hit rate, actual Redis QPS is ~5K-50K (only CDN cache misses reach origin). This is comfortable for a small Redis cluster.

**✍️ Note**: The CDN absorbs the vast majority of read traffic. Redis only serves CDN cache misses + detail page lookups + WebSocket fan-out. This is why we don't need 12-18 Redis nodes — the CDN does the heavy lifting.

### TimescaleDB (Historical Data Store)

Two key tables:

- **`price_candles`** — OHLCV candle data. Partitioned by time (automatic hypertable). Indexed on `(asset_id, interval, bucket_time)`. Stores 1m, 5m, 1h, 1d candles.
- **`assets`** — Asset metadata catalog. Stores `asset_id` (PK), `slug`, `symbol`, `name`, `description`, `categories` (JSONB), `color` (hex for UI), `image_url`, `circulating_supply`, `max_supply`, `is_tradable`, `is_dex`, `is_wallet`, `listed_at`, `updated_at`. ~400 rows. Cached in Redis.

### Why TimescaleDB?

| Option | Pros | Cons |
|---|---|---|
| **TimescaleDB** | PostgreSQL-compatible, continuous aggregates, automatic partitioning, compression | Not as fast as InfluxDB for pure time-series writes |
| InfluxDB | Purpose-built for time-series, fast writes | Custom query language (Flux), no SQL joins |
| ClickHouse | Extremely fast analytical queries, columnar | Overkill for ~2M rows/day |
| Raw PostgreSQL | Simple | No auto partitioning, no continuous aggregates |

**We choose TimescaleDB** because our write volume is modest (~2M rows/day) and the continuous aggregates feature eliminates the need for a separate ETL pipeline.

## 7. Failure Handling

### Exchange feed failure

```
Exchange feed health check:
  Every 10 seconds, check last_tick_time for each exchange adapter.

  If now() - last_tick_time > 30 seconds:
    → Mark exchange feed as DEGRADED
    → Alert on-call
    → Continue using other exchange feeds for VWAP
    → If ALL feeds degraded:
      → Freeze prices at last known value
      → Set updated_at to last known time
      → Client detects stale updated_at and shows warning banner
```

### CDN / API failure

- CDN has built-in redundancy (Cloudflare/Fastly global PoP network).
- If origin API goes down, CDN serves stale content via `stale-if-error` directive. Users see slightly old prices but the page stays up.
- If Redis goes down, API falls back to TimescaleDB. Latency degrades from < 10ms to ~50-100ms.

### WebSocket gateway failure (detail page)

- L4 health check detects failure within 5 seconds.
- ~50K clients auto-reconnect across remaining healthy gateways.
- During the ~2 second gap, client shows last known price.
- No data loss — the next tick fills the gap.

### Price Aggregation Service failure

- Run 3+ replicas. Each consumes from the same exchange adapter output queues.
- Use leader election (via Redis lock) to prevent duplicate computation of pre-built list pages.
- If all replicas fail, Redis continues serving the last-computed data (stale but available). Client sees stale `updated_at` and shows warning.

---

# Other Considerations

## Rate Limiting

| Tier | Rate Limit | Data Freshness |
|---|---|---|
| Anonymous (CDN-served) | Effectively unlimited (CDN handles) | 30-60s stale |
| Authenticated (logged-in) | 100 req/s per user | 10-30s stale |
| API key (developer) | 1000 req/s per key | Real-time (paid tier) |

## Multi-Currency Support

Prices are stored in USD cents. For other fiat currencies (EUR, GBP, JPY):
1. Fetch exchange rates from a forex API (updated every 60 seconds).
2. Store rates in Redis.
3. Server-side conversion when building the pre-computed list pages — generate separate pages per currency (`asset_list:mcap:usd:page:1`, `asset_list:mcap:eur:page:1`).

**Alternative**: Client-side conversion (simpler, fewer cached pages, but potential rounding inconsistencies across clients). For a global product, server-side is better for consistency.

## Monitoring & Alerting

Key metrics:
- **Price freshness** — Time since last tick per asset. Alert if > 30 seconds.
- **CDN hit rate** — Should be > 95%. Drop indicates cache misconfiguration.
- **API latency** — p99 by endpoint. Alert if list API > 200ms or search > 100ms.
- **WebSocket connection count** — Monitor per gateway. Alert if approaching capacity.
- **Redis latency** — p99 GET latency. Alert if > 5ms.
- **Exchange feed health** — Per-exchange uptime. Alert if any feed degraded > 5 minutes.
- **Pre-computation pipeline lag** — Alert if list pages haven't been refreshed in > 60 seconds.

## Scalability

### Scale Overview

| Component | Scale | Architecture |
|---|---|---|
| CDN | Absorbs 95%+ of list page traffic | Global edge network |
| REST API | ~50K QPS at origin (after CDN) | 10-20 stateless instances |
| WebSocket Gateway | 200K-500K concurrent connections | 5-10 instances |
| Redis Cluster | ~5K-50K ops/sec (after CDN) | 6 nodes (3 primaries) |
| Redis Pub/Sub | ~500-1,000 msgs/sec (lazy emit, active channels only) | Same Redis cluster |
| Price Aggregation | ~80K ticks/min ingested | 3 replicas |
| TimescaleDB | ~576K rows/day, ~2 GB/year | Single primary + 2 replicas |

### Multi-Region Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                    Global DNS (GeoDNS)                        │
│              Route users to nearest region                    │
└────────┬──────────────────┬──────────────────┬───────────────┘
         │                  │                  │
    ┌────▼────┐        ┌────▼────┐        ┌────▼────┐
    │ US-EAST │        │ EU-WEST │        │ AP-EAST │
    │         │        │         │        │         │
    │ CDN PoP │        │ CDN PoP │        │ CDN PoP │
    │ REST API│        │ REST API│        │ REST API│
    │ WS GW   │        │ WS GW   │        │ WS GW   │
    │ Redis   │        │ Redis   │        │ Redis   │
    └────┬────┘        └────┬────┘        └────┬────┘
         │                  │                  │
         └──────────────────┼──────────────────┘
                            │
              ┌─────────────▼─────────────┐
              │  Centralized Ingestion     │
              │  (US-EAST primary)         │
              │                            │
              │  Exchange Feed Adapters    │
              │  Price Aggregation Service │
              │  TimescaleDB Primary       │
              └──────────┬─────────────────┘
                         │
              Cross-region replication
              (Redis → Redis via replication)
```

**Design: Centralized ingestion, replicated read layer**

Price data is global and identical for all users. Therefore:

1. **Ingestion is centralized** in one primary region (US-EAST). Exchange feed adapters and the Price Aggregation Service run here. This avoids duplicate exchange connections and price divergence across regions.

2. **Read layer is replicated** to all regions. The Price Aggregation Service writes to US-EAST Redis, which replicates to EU-WEST and AP-EAST Redis clusters. Each region's CDN and REST API read from their local Redis.

3. **Cross-region replication latency**: ~50-100ms (US → EU), ~150-200ms (US → APAC). APAC users see prices ~200ms behind US users — imperceptible for a list page that refreshes every 30-60 seconds.

**❓ What if the primary ingestion region goes down?**

1. A hot standby aggregation service runs in EU-WEST (passive mode).
2. If US-EAST fails, DNS failover promotes EU-WEST within ~30 seconds.
3. During the gap, all regions continue serving the last-known prices from their local Redis + CDN cache. The CDN `stale-if-error` directive ensures pages stay up even if all origins fail.

**❓ Why not run ingestion in every region?**

- Each region would connect to the same exchanges → 3x connections, potential rate limiting.
- VWAP computation would produce slightly different prices per region (network jitter). Users switching regions would see price flickers.
- More operational complexity for no benefit — price data doesn't need regional affinity.

---

# Summary

The core of this design:

1. **CDN-first architecture for the list page** — The explore page is a cacheable snapshot, not a live feed. 30-60 second TTL at CDN absorbs 95%+ of traffic. The API server barely does any work.

2. **Pre-computation over on-demand calculation** — Sorted list pages, trending lists, sparklines, and market health stats are all pre-built by the aggregation service and stored as complete JSON responses in Redis. The REST API is a single Redis GET.

3. **Single response = entire page** — A single API call returns everything the explore page needs: asset list, market health stats, BTC/ETH headers, top gainers. This minimizes round trips.

4. **WebSocket only on the detail page** — Live price ticks are only needed when a user is viewing a single asset's chart. This keeps WebSocket connections at ~200-500K (manageable) instead of millions. Each client subscribes to exactly 1 asset.

5. **Server-side candle aggregation** — The server computes OHLCV candles and streams the forming candle to clients every ~1 second with an `is_closed` flag. Clients never aggregate raw ticks — they just render the candle object. This is the industry standard (Binance `@kline_5m`, Coinbase candles channel).

6. **Simple Pub/Sub for WebSocket fan-out** — Redis Pub/Sub distributes ticker + candle updates to the gateway fleet. With lazy emission (only channels with active subscribers), actual load is ~500-1,000 msgs/sec — trivial for Redis. No need for NATS or Kafka.

7. **Centralized ingestion, replicated reads** — Exchange feeds are consumed in one region. Aggregated prices are replicated globally via Redis cross-region replication. Avoid duplicate exchange connections and price divergence.

8. **Graceful degradation everywhere** — CDN serves stale on origin failure. Redis serves stale on aggregation failure. Client detects stale `updated_at` and warns user. Every failure mode has a fallback, and the page stays up.

9. **Simplicity as a feature** — ~400 listed assets don't need Elasticsearch (use a trie). ~576K candle rows/day don't need sharding (single TimescaleDB). The list page doesn't need WebSocket (CDN + polling). REST over GraphQL because URL-based CDN caching is trivial — no persisted query infrastructure needed. The hardest scaling problem is CDN cache management, not real-time fan-out.
