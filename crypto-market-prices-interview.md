# Crypto Market Prices Page — System Design (Coinbase Explore)

# Important Concepts

- WebSocket push vs polling for real-time price updates
- Write-behind caching (Redis) for price data
- Fan-out on write vs fan-out on read for price distribution
- Rate limiting and tiered data freshness
- CDN and edge caching for static asset lists
- Aggregation pipeline for market data (OHLCV candles, volume, market cap)

# Context

We are designing the backend for a page like **Coinbase Explore** — a high-traffic page that displays real-time cryptocurrency market prices for thousands of assets. The page is the #1 entry point for casual users and receives billions of page views per month.

Core features:
1. **Asset listing table** — Shows ~1,500+ tradeable assets with name, symbol, price, 24h change %, 24h volume, market cap, and a sparkline chart.
2. **Real-time price updates** — Prices tick in real time (every 1-5 seconds) without page refresh.
3. **Categories & filters** — Filter by "Trending", "Top Gainers", "Top Losers", "New Listings", "DeFi", "Layer 1", etc.
4. **Search** — Instant search across all assets by name or symbol.
5. **Pagination / infinite scroll** — Default view shows top 50 by market cap; user scrolls or paginates for more.
6. **Price sparkline** — 24h mini chart for each asset.

This is a **read-heavy** system. The write side is a continuous ingestion pipeline from exchanges and market data providers. The read side is serving millions of concurrent users viewing the same data.

# Requirements

## Functional Requirements

1. **List assets** — Return a paginated list of crypto assets with current price, 24h price change, volume, and market cap. Support sorting and filtering by category.
2. **Real-time price stream** — Push price updates to connected clients in real time (target: every 1-5 seconds per asset).
3. **Search** — Search assets by name or ticker symbol with sub-100ms latency.
4. **Asset detail** — Get detailed data for a single asset: price, 24h/7d/30d/1y charts (OHLCV candles), description, stats.
5. **Trending / Top movers** — Pre-computed lists: top gainers, top losers, trending (by user interest/volume spike), new listings.

**✍️ Note**: We are NOT designing the trading engine, order book, or wallet system. We assume upstream systems (matching engine, exchange integrations) produce a stream of price ticks that we consume.

## Non-Functional Requirements

### Scale

- **10M+ DAU** viewing the explore page. Peak concurrent WebSocket connections: **2M+**.
- **~1,500 assets** tracked. Each asset gets a price tick every 1-5 seconds from exchanges.
- Read QPS for HTTP API: **~100K QPS** (page loads, search, pagination). Peak: **500K QPS** during market volatility events (Bitcoin crash/rally).
- WebSocket fan-out: each price tick must be pushed to **~2M connected clients** within 1-5 seconds.
- Data freshness: prices must be **< 3 seconds stale** for logged-in users, **< 10 seconds** for anonymous/CDN-cached.

### Availability

- **99.99% uptime** — Users check prices during market crashes, which is when load is highest. Downtime during volatility is catastrophic for trust.
- Multi-region deployment. No single region failure should take down the price page.
- Graceful degradation: if real-time WebSocket fails, fall back to polling every 5 seconds. If price ingestion lag exceeds 30 seconds, show a stale-data warning banner.

### Latency

- Initial page load (asset list API): **< 100ms p99** (CDN-served for anonymous users).
- WebSocket price update delivery: **< 2 seconds p99** from exchange tick to client screen.
- Search API: **< 50ms p99**.
- Sparkline chart data: **< 200ms p99** (pre-computed, cached).

### Consistency

- **Eventual consistency is acceptable** — Price data is inherently approximate (different exchanges show different prices). We target "good enough" freshness, not strict consistency.
- Asset metadata (name, symbol, category tags) is rarely updated and can be strongly consistent with short cache TTL.

## Edge Cases

### Market volatility spikes (thundering herd)

When Bitcoin drops 10% in an hour, traffic spikes 5-10x. WebSocket connections surge. The price ingestion pipeline gets flooded with ticks. We must handle this gracefully without cascading failures.

### Exchange feed outage

If a price feed from a major exchange goes down, we should fall back to the last known price and display a "delayed" indicator. We should NOT show zero or null prices.

### Stale data display

If our ingestion pipeline falls behind, the client should show a "Prices may be delayed" warning. We track data freshness via a heartbeat mechanism.

### New asset listing

When a new asset is listed, we need to add it to the asset catalog, backfill sparkline data, and make it searchable — all without downtime.

---

# API Design

## Public API (served by API Gateway)

### List Assets

```
GET /v1/assets?page=1&page_size=50&sort=market_cap_desc&category=defi

Response 200:
{
  "assets": [
    {
      "id": "bitcoin",
      "symbol": "BTC",
      "name": "Bitcoin",
      "price_usd": 6834521,           // in cents ($68,345.21)
      "price_change_24h_pct": -234,    // basis points (-2.34%)
      "volume_24h_usd": 2845000000,   // in cents
      "market_cap_usd": 134500000000, // in cents
      "sparkline_24h": [68100, 68050, 67900, ...],  // hourly prices (24 points)
      "categories": ["layer-1", "store-of-value"],
      "rank": 1
    },
    ...
  ],
  "total": 1523,
  "page": 1,
  "page_size": 50
}
```

**✍️ Note**: Prices are in integer cents (never float) to avoid floating-point precision issues. The `sparkline_24h` array contains 24 hourly price points — small enough to inline, avoids a separate API call per asset.

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

**✍️ Note**: Search matches on both `name` and `symbol` with prefix matching. With only ~1,500 assets, we can use an in-memory trie or prefix index — no need for Elasticsearch for search alone.

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
  }
}
```

### Get Trending / Top Movers

```
GET /v1/assets/trending?list=top_gainers&limit=10

Response 200:
{
  "list": "top_gainers",
  "period": "24h",
  "assets": [
    { "id": "solana", "symbol": "SOL", "price_usd": 18532, "price_change_24h_pct": 1845 },
    ...
  ],
  "updated_at": "2026-03-07T12:00:00Z"
}
```

**✍️ Note**: Trending lists are pre-computed every 60 seconds and cached. Lists available: `top_gainers`, `top_losers`, `trending`, `highest_volume`, `new_listings`.

## WebSocket API (real-time price stream)

```
Connect: wss://stream.example.com/v1/prices

Client → Server (subscribe):
{
  "action": "subscribe",
  "channels": ["prices:top50"]    // or "prices:all", "prices:BTC,ETH,SOL"
}

Server → Client (price tick):
{
  "type": "price_update",
  "data": {
    "id": "bitcoin",
    "symbol": "BTC",
    "price_usd": 6834521,
    "price_change_24h_pct": -234,
    "volume_24h_usd": 2845000000,
    "timestamp": 1709600123
  }
}
```

**✍️ Note**: We support channel-based subscriptions. The default `prices:top50` channel covers what's visible on the first page load. Users who scroll down or filter can subscribe to additional channels. This reduces unnecessary fan-out — no need to push 1,500 asset updates to a user only viewing the top 50.

**Alternative: Server-Sent Events (SSE) vs WebSocket**

| | WebSocket | SSE |
|---|---|---|
| Direction | Bidirectional | Server → Client only |
| Protocol | ws:// | HTTP |
| Reconnection | Manual | Auto-reconnect built in |
| Load balancer support | Needs sticky sessions or L4 | Works with standard HTTP LBs |
| Browser support | Universal | Universal (except old IE) |

For this use case, SSE is a strong alternative because we only need server-to-client push. WebSocket's bidirectional capability is useful for subscription management (subscribe/unsubscribe channels) but SSE can handle this via URL params. **We choose WebSocket** because the subscribe/unsubscribe pattern for channels is cleaner with bidirectional messaging, and most CDN/edge platforms now support WebSocket well.

---

# High Level Design

```
                          ┌────────────────────────────────────────────┐
                          │              CDN / Edge Cache              │
                          │     (static asset list, sparkline data)    │
                          └────────────────────┬───────────────────────┘
                                               │
                                               │ cache miss
                          ┌────────────────────▼───────────────────────┐
                          │          API Gateway / Load Balancer        │
                          └──────┬─────────────────────┬───────────────┘
                                 │                     │
                    ┌────────────▼──────┐    ┌─────────▼──────────────┐
                    │   REST API Fleet  │    │  WebSocket Gateway      │
                    │   (stateless)     │    │  Fleet (sticky conn)    │
                    │                   │    │                         │
                    │ • List assets     │    │ • Price subscriptions   │
                    │ • Search          │    │ • Channel management    │
                    │ • Asset detail    │    │ • Heartbeat             │
                    │ • Trending lists  │    └────────┬────────────────┘
                    └────────┬─────────┘             │
                             │                       │ subscribe to
                             │ read from             │ price channels
                    ┌────────▼─────────┐    ┌────────▼────────────────┐
                    │   Redis Cluster   │    │  Pub/Sub Broker         │
                    │   (price cache)   │    │  (Redis Pub/Sub or      │
                    │                   │    │   NATS / Kafka)         │
                    │ • Latest prices   │    └────────┬────────────────┘
                    │ • Sparkline data  │             │ publish
                    │ • Trending lists  │             │ price ticks
                    │ • Search index    │    ┌────────▼────────────────┐
                    └────────┬─────────┘    │  Price Aggregation      │
                             │              │  Service                 │
                             │ write-behind │                         │
                             │              │ • Normalizes feeds      │
                    ┌────────▼─────────┐    │ • Computes VWAP         │
                    │   TimescaleDB /  │    │ • Generates sparklines  │
                    │   PostgreSQL      │    │ • Computes trending     │
                    │                   │    └────────┬────────────────┘
                    │ • OHLCV candles   │             │ consume
                    │ • Historical data │             │
                    │ • Asset metadata  │    ┌────────▼────────────────┐
                    └──────────────────┘    │  Exchange Feed Adapters  │
                                            │                         │
                                            │ • Binance WebSocket     │
                                            │ • Coinbase Pro WS       │
                                            │ • Kraken WS             │
                                            │ • CoinGecko API         │
                                            └─────────────────────────┘
```

### Exchange Feed Adapters

Stateful connectors that maintain WebSocket connections to major crypto exchanges (Binance, Coinbase Pro, Kraken, etc.) and REST polling to aggregators (CoinGecko, CoinMarketCap). Each adapter normalizes the exchange-specific format into a unified price tick event.

### Price Aggregation Service

The brain of the ingestion pipeline. Consumes raw ticks from all exchange adapters, computes the **Volume-Weighted Average Price (VWAP)** across exchanges, and produces the canonical price for each asset. Also computes derived data: 24h change %, sparkline arrays, trending lists. Publishes aggregated price ticks to the Pub/Sub broker and writes to Redis.

### Redis Cluster (Price Cache)

The primary read store for all API requests. Stores:
- Latest price per asset (key: `price:{asset_id}`) — updated every 1-5 seconds.
- Pre-computed asset list pages (key: `asset_list:page:{n}:sort:{sort}`) — updated every 5-10 seconds.
- Sparkline arrays (key: `sparkline:{asset_id}:24h`) — updated every 5 minutes.
- Trending lists (key: `trending:{list_name}`) — updated every 60 seconds.
- Search index (sorted set: `search_index` scored by rank, or a simple hash map).

### Pub/Sub Broker (Redis Pub/Sub or NATS)

Distributes price ticks from the aggregation service to all WebSocket gateway instances. Each WebSocket gateway subscribes to price channels. When a new price tick arrives, the gateway pushes it to all connected clients subscribed to that channel.

### REST API Fleet

Stateless HTTP servers that serve the list, search, detail, and trending APIs. All reads go to Redis first (cache hit rate > 99% for price data). Cache misses fall through to TimescaleDB.

### WebSocket Gateway Fleet

Maintains long-lived WebSocket connections with clients. Each gateway instance handles ~50K-100K concurrent connections. Subscribes to price channels on the Pub/Sub broker and fans out to connected clients. Sticky connections — a client stays on the same gateway for the duration of the session.

### TimescaleDB / PostgreSQL

Time-series database for historical OHLCV candle data. Used for chart rendering (7d, 30d, 1y charts) and backfill. Not on the hot read path — Redis serves most reads. Also stores asset metadata (name, description, categories).

### CDN / Edge Cache

Caches the initial page load API response (top 50 assets by market cap) at the edge with a short TTL (5-10 seconds). This absorbs the thundering herd during market events. Anonymous users always hit the CDN; logged-in users bypass CDN for fresher data.

---

# Detailed Technical Design

## 1. Price Ingestion Pipeline

The most critical component. A price tick flows through the system in this order:

```
Exchange WS Feed → Exchange Adapter → Price Aggregation Service → Redis + Pub/Sub → WebSocket Gateway → Client
```

**Target: < 2 seconds end-to-end latency from exchange tick to client screen.**

### Exchange Feed Adapters

Each adapter maintains a persistent WebSocket connection to one exchange. Design:

```
┌────────────────────────────────┐
│      Exchange Adapter (per exchange)          │
│                                               │
│  ┌──────────┐    ┌────────────┐    ┌────────┐ │
│  │ WS Client│───▶│ Normalizer │───▶│ Output │ │
│  │ (reconnect│   │ (format,   │    │ Queue  │ │
│  │  + backoff)│  │  dedup,    │    │        │ │
│  └──────────┘   │  validate) │    └────────┘ │
│                  └────────────┘               │
└───────────────────────────────────────────────┘
```

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

### Derived Data Computation

The aggregation service also computes:
- **24h price change %** — Compare current VWAP to VWAP from 24 hours ago (looked up from Redis or TimescaleDB).
- **24h volume** — Sum of trade volume across exchanges in the past 24 hours.
- **Market cap** — `price × circulating_supply` (circulating supply is updated daily from CoinGecko).
- **Sparkline** — 24 hourly data points; re-computed every 5 minutes by reading the last 24 hours of candle data.
- **Trending lists** — Sorted by 24h price change % (gainers/losers), by 24h volume (highest volume), by 1h price momentum (trending).

## 2. Real-Time Price Distribution (Fan-Out)

This is the hardest scaling challenge: pushing price updates to **2M+ concurrent WebSocket clients** within seconds.

### Architecture: Two-Level Fan-Out

```
                    ┌───────────────────────────┐
                    │  Price Aggregation Service │
                    │  (single logical stream)   │
                    └─────────────┬─────────────┘
                                  │ publish to Pub/Sub
                                  │ (one message per asset tick)
                    ┌─────────────▼─────────────┐
                    │    Pub/Sub Broker Layer     │
                    │  (NATS / Redis Pub/Sub)     │
                    │                             │
                    │  Channel: prices:BTC        │
                    │  Channel: prices:ETH        │
                    │  Channel: prices:top50      │
                    └──┬──────┬──────┬──────┬────┘
                       │      │      │      │
              ┌────────▼┐ ┌──▼────┐ ┌▼─────┐ ┌▼────────┐
              │ WS GW 1 │ │WS GW 2│ │WS GW3│ │WS GW N  │
              │ (50K    │ │(50K   │ │(50K  │ │(50K     │
              │  conns) │ │ conns)│ │conns)│ │ conns)  │
              └────┬────┘ └──┬────┘ └──┬───┘ └────┬────┘
                   │         │         │          │
              50K clients  50K     50K        50K clients
```

**Level 1: Pub/Sub broker → WebSocket Gateways**
- The aggregation service publishes one message per price tick to a Pub/Sub channel (e.g., `prices:BTC`).
- Each WebSocket gateway subscribes to channels that its connected clients care about.
- With 40 gateway instances (2M / 50K per instance), one Pub/Sub message is delivered to ~40 subscribers. This is easy for NATS or Redis Pub/Sub.

**Level 2: WebSocket Gateway → Clients**
- Each gateway iterates through its local subscription map and pushes the price update to subscribed clients.
- A gateway with 50K clients subscribed to `prices:top50` needs to send 50K WebSocket messages per tick. At 1 tick/second, that's 50K messages/second per gateway — achievable with efficient I/O (epoll, io_uring).

**❓ Why not use Kafka for the Pub/Sub layer?**

Kafka is designed for durable, ordered, replayed message streams. For real-time price fan-out, we need:
- **Low latency** (< 100ms from publish to all subscribers).
- **Ephemeral** messages — if a tick is missed, the next tick supersedes it. No need for replay.
- **Topic fan-out** — Kafka's consumer group model doesn't support broadcasting the same message to all consumers in a group.

NATS or Redis Pub/Sub are better fits. They are fire-and-forget, low-latency, and support broadcast fan-out natively. The tradeoff is no durability — but price ticks are ephemeral by nature.

**Alternative: Kafka Streams for price aggregation, NATS for fan-out.** Kafka is still useful in the ingestion pipeline (exchange → aggregation) where we want durability and replay. The fan-out from aggregation to WebSocket gateways uses NATS.

### Connection Management

**❓ How do we handle 2M concurrent WebSocket connections?**

- **40 gateway instances** × 50K connections each.
- Each connection consumes ~10-20KB of memory (buffer + metadata). 50K connections ≈ 750MB RAM per instance. Comfortable on a 4GB instance.
- We use an **L4 load balancer** (NLB) that distributes new connections across gateway instances. Once established, connections are sticky.
- **Heartbeat**: Gateway sends a ping every 30 seconds. Client responds with pong. If no pong for 90 seconds, close the connection.
- **Reconnection**: Client-side logic auto-reconnects with exponential backoff (1s, 2s, 4s, max 30s). On reconnect, re-subscribe to channels and fetch latest prices via HTTP API to fill the gap.

**❓ How do we handle a gateway instance going down?**

- The L4 load balancer detects the unhealthy instance (TCP health check fails).
- ~50K clients immediately disconnect and auto-reconnect (client-side logic). The load balancer distributes them across remaining healthy instances.
- During the ~2 second reconnection window, clients show the last known price (cached locally in the browser). No data loss — just a brief stale display.

### Bandwidth Optimization

Each price tick JSON message is ~200 bytes. For the `prices:top50` channel, that's 50 ticks/second (50 assets × 1 tick/sec) × 200 bytes = 10KB/s per client. For 50K clients per gateway, that's 500MB/s outbound — too high.

**Optimization 1: Batch updates**
Instead of sending one message per asset, batch all price changes into one message every 1 second:

```json
{
  "type": "price_batch",
  "data": [
    { "id": "BTC", "p": 6834521, "c": -234 },
    { "id": "ETH", "p": 345621, "c": -178 },
    ...
  ],
  "ts": 1709600123
}
```

This reduces 50 messages to 1 message per second. Batched payload: ~2KB per client per second. For 50K clients: **100MB/s** outbound. Achievable with a 1Gbps NIC.

**Optimization 2: Delta encoding**
Only send fields that changed since the last update. If BTC's price changed but volume didn't, only send the price field.

**Optimization 3: Binary protocol (MessagePack / Protobuf)**
JSON is verbose. MessagePack reduces payload size by ~30%. Protobuf by ~50%. Tradeoff: debugging is harder, and client-side decoding adds complexity. For a browser-based client, MessagePack is a good middle ground.

## 3. Caching Strategy

Redis is the heart of the read path. Every API response is served from Redis.

### Cache Structure

| Key pattern | Value | TTL | Update frequency |
|---|---|---|---|
| `price:{asset_id}` | Latest price JSON | 30s | Every 1-5s |
| `asset_list:mcap:page:{n}` | Pre-sorted page of 50 assets | 10s | Every 5-10s |
| `asset_list:gainers:page:1` | Top gainers list | 60s | Every 60s |
| `sparkline:{asset_id}:24h` | Array of 24 hourly prices | 5min | Every 5min |
| `asset_detail:{asset_id}` | Full asset metadata + stats | 5min | Every 5min |
| `search_index` | Sorted set (score=rank, member=asset_json) | 1h | On asset catalog change |
| `trending:{list}` | Pre-computed trending list | 60s | Every 60s |
| `candles:{asset_id}:{interval}` | OHLCV candle data | 5min | Every 5min |

### Pre-Computation Pipeline

The Price Aggregation Service runs a periodic pipeline:

```
Every 1-5 seconds:
  → Update price:{asset_id} in Redis for all assets that changed

Every 5-10 seconds:
  → Re-sort all assets by market cap
  → Write asset_list:mcap:page:{1..30} to Redis (30 pages × 50 assets)

Every 60 seconds:
  → Compute top gainers, top losers, trending, highest volume lists
  → Write trending:{list} to Redis

Every 5 minutes:
  → Recompute sparkline arrays from candle data
  → Write sparkline:{asset_id}:24h to Redis for all assets
```

**✍️ Note**: Pre-computing and caching the sorted asset list pages means the REST API is a simple Redis GET — no sorting or filtering at request time. This is the key to achieving < 100ms p99 latency.

### CDN Layer

For anonymous users (70%+ of traffic), the CDN serves a cached version of the asset list API with a 5-10 second TTL. This absorbs the thundering herd during volatile markets.

```
CDN edge → Cache HIT (5s TTL) → Return cached response
CDN edge → Cache MISS → Forward to API → Cache response → Return
```

Even 5-second staleness is acceptable for the explore page. Users who need real-time prices are on the WebSocket stream.

**❓ What about cache stampede?**

When the CDN TTL expires, many concurrent requests hit the origin simultaneously. We mitigate with:
1. **Stale-while-revalidate** — CDN serves stale content while fetching fresh data in the background. Users always get a fast response.
2. **Request coalescing** — CDN collapses multiple origin requests for the same URL into one.
3. **Background refresh** — Our API proactively pushes updated responses to the CDN (cache warming) before the TTL expires.

## 4. Search

With only ~1,500 assets, search is simple. We do NOT need Elasticsearch.

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
1. Traverse the trie to find all assets starting with "eth" (by name or symbol).
2. Return results sorted by rank (top assets first).
3. Total latency: < 1ms (in-memory trie traversal).

The trie is rebuilt on startup from Redis and updated when assets are added/removed (rare event).

**Alternative: Elasticsearch**

Overkill for 1,500 items. Adds operational complexity (cluster management, JVM tuning) for no benefit. If we needed full-text search across descriptions or fuzzy matching, ES would make sense. For prefix matching on name/symbol, a trie is simpler and faster.

## 5. Sparkline and Chart Data

### Sparkline (24h mini chart)

The 24h sparkline is an array of 24 hourly prices. It's pre-computed every 5 minutes by the aggregation service:

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

The sparkline is embedded in the asset list response — no extra API call needed.

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

**Continuous aggregates** in TimescaleDB automatically roll up 1-minute candles into 5m, 1h, and 1d candles:

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

**✍️ Note**: TimescaleDB's continuous aggregates are incrementally updated as new data arrives — no batch job needed. This gives us real-time candle charts without manual ETL.

### Storage Estimate

- 1,500 assets × 1 candle/minute × 1,440 min/day = **~2.16M rows/day** (1-minute candles).
- Each row ~100 bytes → **~216 MB/day** → ~79 GB/year.
- With TimescaleDB compression (typically 10-20x), raw storage is ~4-8 GB/year. Trivial.

## 6. Database Design

### Redis Cluster (Primary Read Store)

No schema needed — key-value store. See the cache structure table above.

Redis Cluster with **6+ nodes** (3 primaries + 3 replicas). Each primary handles ~50K ops/sec. Total cluster capacity: ~150K ops/sec. At peak 500K API QPS with 99% cache hit rate, actual Redis QPS is ~500K (each API call makes 1-2 Redis GETs). We may need a larger cluster (12-18 nodes) at peak.

**✍️ Note**: We use Redis as the primary read store, NOT PostgreSQL. This is a deliberate design choice. The explore page is extremely read-heavy with data that changes every few seconds. Redis gives us < 1ms read latency. PostgreSQL would require constant index scans on rapidly changing data — unnecessary overhead.

### TimescaleDB (Historical Data Store)

Two key tables:

- **`price_candles`** — OHLCV candle data. Partitioned by time (automatic hypertable). Indexed on `(asset_id, interval, bucket_time)`. Stores 1m, 5m, 1h, 1d candles.
- **`assets`** — Asset metadata catalog. Stores `asset_id` (PK), `symbol`, `name`, `description`, `categories` (JSONB), `circulating_supply`, `max_supply`, `logo_url`, `listed_at`, `updated_at`. ~1,500 rows. Cached in Redis with 1h TTL.

### Why TimescaleDB (and not raw PostgreSQL, InfluxDB, or ClickHouse)?

| Option | Pros | Cons |
|---|---|---|
| **TimescaleDB** | PostgreSQL-compatible (familiar SQL), continuous aggregates, automatic partitioning, compression | Not as fast as InfluxDB for pure time-series writes |
| InfluxDB | Purpose-built for time-series, fast writes | Custom query language (Flux), no SQL joins, harder to store asset metadata alongside |
| ClickHouse | Extremely fast analytical queries, columnar | Overkill for our write volume (~2M rows/day), operational complexity |
| Raw PostgreSQL | Simple | No automatic partitioning, no continuous aggregates, manual vacuuming of time-series data |

**We choose TimescaleDB** because it gives us PostgreSQL compatibility (familiar, battle-tested) with time-series superpowers (continuous aggregates, compression, automatic partitioning). Our write volume is low enough that InfluxDB's write performance advantage doesn't matter.

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
      → Display "Prices may be delayed" banner on client
      → Switch client to polling mode (fetch from cache every 10s)
```

### WebSocket gateway failure

- L4 health check detects failure within 5 seconds.
- Clients auto-reconnect to a healthy gateway.
- During the ~2 second gap, client shows last known prices.
- No data loss: prices are ephemeral; the next tick fills the gap.

### Redis failure

- Redis Cluster tolerates individual node failures (replica promotes to primary).
- If an entire Redis cluster goes down (unlikely), the API falls back to TimescaleDB. Latency degrades from < 10ms to ~50-100ms. Still acceptable.
- The CDN continues serving cached responses, absorbing most traffic.

### Price Aggregation Service failure

- Run 3+ replicas. Each consumes from the same exchange adapter output queues.
- If one replica fails, others continue processing. Redis and Pub/Sub are updated by the surviving replicas.
- Use leader election (via Redis lock or ZooKeeper) to prevent duplicate computation of trending lists.

---

# Other Considerations

## Rate Limiting

Tiered rate limiting to protect the system:

| Tier | Rate Limit | Data Freshness |
|---|---|---|
| Authenticated (logged-in) | 100 req/s per user | Real-time WebSocket |
| Anonymous (no auth) | 20 req/s per IP | CDN-cached (5-10s stale) |
| API key (developer) | 1000 req/s per key | Real-time (with paid plans) |

WebSocket connections are also rate-limited: max 5 subscribe/unsubscribe messages per second per connection.

## Multi-Currency Support

Prices are stored in USD cents. For other fiat currencies (EUR, GBP, JPY), we:
1. Fetch exchange rates from a forex API (updated every 60 seconds).
2. Store exchange rates in Redis.
3. The client-side converts USD prices to the user's local currency using the cached exchange rate.

**Alternative**: Server-side conversion. This increases Redis storage (every price × N currencies) but reduces client-side complexity. For a global product, server-side conversion is better because it ensures consistency.

## Monitoring & Alerting

Key metrics:
- **Price freshness** — Time since last tick per asset. Alert if > 30 seconds.
- **WebSocket connection count** — Monitor per gateway. Alert if a single gateway exceeds 80% capacity (40K connections).
- **Pub/Sub latency** — Time from publish to gateway delivery. Alert if p99 > 500ms.
- **Redis latency** — p99 GET latency. Alert if > 5ms.
- **API latency** — p99 by endpoint. Alert if list API > 200ms or search > 100ms.
- **Exchange feed health** — Per-exchange uptime. Alert if any feed degraded > 5 minutes.
- **CDN hit rate** — Should be > 95%. Drop indicates cache misconfiguration.

## Scalability

### Scale Overview

| Component | Scale | Architecture |
|---|---|---|
| REST API | ~500K QPS peak | 50+ stateless instances behind LB |
| WebSocket Gateway | 2M concurrent connections | 40+ instances, 50K conn each |
| Redis Cluster | ~500K ops/sec peak | 12-18 nodes (6-9 primaries) |
| Pub/Sub (NATS) | ~1,500 msgs/sec (one per asset) | 3-node NATS cluster |
| Price Aggregation | ~300K ticks/min ingested | 3+ replicas |
| TimescaleDB | ~2M rows/day, ~80 GB/year | Single primary + 2 read replicas |
| CDN | Absorbs 70%+ of read traffic | Global edge network |

### Multi-Region Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                    Global DNS (GeoDNS)                        │
│              Route users to nearest region                    │
└────────┬──────────────────┬──────────────────┬───────────────┘
         │                  │                  │
    ┌────▼────┐        ┌────▼────┐        ┌────▼────┐
    │ US-EAST │        │ EU-WEST │        │ AP-EAST │
    │ Region  │        │ Region  │        │ Region  │
    │         │        │         │        │         │
    │ REST API│        │ REST API│        │ REST API│
    │ Fleet   │        │ Fleet   │        │ Fleet   │
    │         │        │         │        │         │
    │ WS GW   │        │ WS GW   │        │ WS GW   │
    │ Fleet   │        │ Fleet   │        │ Fleet   │
    │         │        │         │        │         │
    │ Redis   │        │ Redis   │        │ Redis   │
    │ Cluster │        │ Cluster │        │ Cluster │
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
              (Redis → Redis, NATS → NATS)
```

**Design: Centralized ingestion, replicated read layer**

Unlike payments (where data must stay in region), price data is global and identical for all users. Therefore:

1. **Ingestion is centralized** in one primary region (US-EAST). Exchange feed adapters and the Price Aggregation Service run here. This avoids the complexity of multi-region deduplication of the same exchange ticks.

2. **Read layer is replicated** to all regions. The Price Aggregation Service publishes to a cross-region Pub/Sub (NATS with gateway mode, or Redis cross-region replication). Each region's Redis cluster and WebSocket gateways receive the same price updates.

3. **Cross-region replication latency**: ~50-100ms (US → EU), ~150-200ms (US → APAC). This means APAC users see prices ~200ms behind US users — acceptable for a price display page (not a trading engine).

**❓ What if the primary ingestion region goes down?**

Failover strategy:
1. A hot standby Price Aggregation Service runs in EU-WEST.
2. Exchange Feed Adapters in EU-WEST are in passive mode (connected but not publishing).
3. If US-EAST goes down, a DNS failover promotes EU-WEST's ingestion pipeline within ~30 seconds.
4. During the ~30s gap, all regions continue serving the last known prices from their local Redis cache.

**❓ Why not run ingestion in every region?**

Running independent ingestion in each region would mean:
- Each region connects to the same exchanges → 3x the connections, potential rate limiting from exchanges.
- VWAP computation would produce slightly different prices in each region (due to network jitter). Users switching regions would see price flickers.
- More operational complexity for no real benefit.

Centralized ingestion with replicated reads is the standard pattern for market data distribution systems.

---

# Summary

The core of this design for a Coinbase Explore-like page at scale:

1. **Read-heavy architecture** — Redis as the primary read store, not PostgreSQL. Pre-compute everything (sorted lists, sparklines, trending). The API server is a thin Redis proxy.

2. **Two-level fan-out for real-time prices** — Price Aggregation → NATS Pub/Sub → WebSocket Gateways → Clients. Channel-based subscriptions limit fan-out to what each client actually views.

3. **CDN as the first line of defense** — 5-10 second TTL on the asset list API absorbs 70%+ of traffic (anonymous users). Stale-while-revalidate prevents cache stampedes.

4. **Pre-computation over on-demand calculation** — Trending lists, sparklines, sorted pages, and VWAP are all computed in the background and pushed to Redis. Request-time computation is near zero.

5. **Centralized ingestion, replicated reads** — Exchange feeds are consumed in one region. Aggregated prices are replicated globally via cross-region Pub/Sub. This avoids duplicate exchange connections and price divergence across regions.

6. **Graceful degradation** — Exchange feed failure → use remaining feeds. WebSocket failure → client falls back to polling. Redis failure → fall through to TimescaleDB. Primary region failure → hot standby promotion. Every failure mode has a fallback.

7. **TimescaleDB for historical data** — Continuous aggregates automatically roll up candle data. Compression keeps storage trivial (~4-8 GB/year). PostgreSQL compatibility means no new query language to learn.

8. **Simplicity where possible** — In-memory trie for search (1,500 assets don't need Elasticsearch). Redis sorted sets for rankings. No sharding needed for TimescaleDB at this write volume. Complexity is reserved for the fan-out problem, which is the genuine hard part.
