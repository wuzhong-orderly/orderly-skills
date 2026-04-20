# Orderly One — Leaderboard & Statistics

Read-only endpoints for DEX rankings and platform statistics. No authentication required for the leaderboard; stats endpoints are public.

---

## Leaderboard

### Get DEX Rankings

```
GET /api/leaderboard?sort=volume&period=30d&limit=20&offset=0
```

No authentication required.

**Query Parameters:**

| Param | Type | Default | Options |
|-------|------|---------|---------|
| `sort` | string | `volume` | `volume`, `pnl`, `fee` |
| `period` | string | `30d` | `daily`, `weekly`, `30d`, `90d` |
| `limit` | number | 20 | Max results per page |
| `offset` | number | 0 | Pagination offset |

**Response:**
```json
{
  "data": [
    {
      "id": "abc123",
      "brokerId": "my_broker",
      "brokerName": "My DEX",
      "primaryLogo": "data:image/webp;base64,...",
      "dexUrl": "https://trading.mysite.com",
      "totalVolume": 15234567.89,
      "totalPnl": 234567.12,
      "totalBrokerFee": 12345.67,
      "totalFee": 45678.90,
      "lastUpdated": "2025-01-01T00:00:00.000Z",
      "description": "A perpetual futures DEX",
      "banner": "data:image/webp;base64,...",
      "logo": "data:image/webp;base64,...",
      "tokenAddress": "0x...",
      "tokenChain": "ethereum",
      "tokenSymbol": "TOKEN",
      "tokenName": "My Token",
      "tokenPrice": 1.23,
      "tokenMarketCap": 12345678,
      "tokenImageUrl": "https://...",
      "telegramLink": "https://t.me/...",
      "discordLink": "https://discord.gg/...",
      "xLink": "https://x.com/...",
      "websiteUrl": "https://mysite.com"
    }
  ],
  "meta": {
    "sortBy": "volume",
    "period": "30d",
    "limit": 20,
    "offset": 0,
    "total": 45
  }
}
```

**Notes:**
- Only graduated DEXs appear (broker ID ≠ `"demo"`)
- DEXs with `showOnBoard: false` are excluded
- Results cached for 1 minute
- Token data (price, market cap, image) fetched from CoinGecko/GeckoTerminal
- `dexUrl` is derived from: custom domain override > custom domain > GitHub Pages URL

---

## Broker Stats

### Get Detailed Stats for a Specific DEX

```
GET /api/leaderboard/broker/{brokerId}?period=30d
```

No authentication required.

**Path Parameter:** `brokerId` — the broker's unique ID

**Query Parameter:** `period` — `daily`, `weekly`, `30d`, or `90d` (default: `30d`)

**Response:**
```json
{
  "data": {
    "dex": {
      "id": "abc123",
      "brokerId": "my_broker",
      "brokerName": "My DEX",
      "primaryLogo": "data:image/webp;base64,...",
      "dexUrl": "https://trading.mysite.com",
      "description": "...",
      "banner": "...",
      "logo": "...",
      "tokenAddress": "0x...",
      "tokenChain": "ethereum",
      "telegramLink": "...",
      "discordLink": "...",
      "xLink": "...",
      "websiteUrl": "..."
    },
    "aggregated": {
      "brokerId": "my_broker",
      "brokerName": "My DEX",
      "totalVolume": 15234567.89,
      "totalPnl": 234567.12,
      "totalBrokerFee": 12345.67,
      "totalFee": 45678.90,
      "lastUpdated": "2025-01-01T00:00:00.000Z",
      "tokenSymbol": "TOKEN",
      "tokenName": "My Token",
      "tokenPrice": 1.23,
      "tokenMarketCap": 12345678,
      "tokenImageUrl": "https://..."
    },
    "daily": [
      {
        "brokerId": "my_broker",
        "brokerName": "My DEX",
        "date": "2025-01-01",
        "perp_volume": 500000,
        "perp_taker_volume": 300000,
        "perp_maker_volume": 200000,
        "realized_pnl": 12345.67,
        "broker_fee": 789.01,
        "total_fee": 2345.67
      }
    ]
  }
}
```

- Returns DEX info, aggregated stats, and daily breakdown
- Cached for 1 minute
- Returns empty data (zeroes) if broker has no trading activity

---

## Platform Statistics

### Get DEX Creation Stats

```
GET /api/stats?period=30d
```

No authentication required.

**Query Parameter:** `period` — `daily`, `weekly`, `30d`, or `90d` (default: `30d`)

**Response:**
```json
{
  "total": 150,
  "graduated": 45,
  "demo": 105,
  "newInPeriod": 12,
  "newGraduatedInPeriod": 3
}
```

| Field | Description |
|-------|-------------|
| `total` | All-time DEX count |
| `graduated` | DEXs with real broker IDs |
| `demo` | DEXs still using `"demo"` |
| `newInPeriod` | New DEXs created in the selected period |
| `newGraduatedInPeriod` | New graduations in the selected period |

### Get Swap Fee Config

```
GET /api/stats/swap-fee-config
```

Returns swap fee rates for all brokers. Cached for 5 minutes.

**Response:**
```json
{
  "brokers": {
    "my_broker": { "swapFeeBps": 30 },
    "another_broker": { "swapFeeBps": 50 }
  }
}
```

---

## Social Cards

DEX owners can set social/branding info that appears on the leaderboard. See `orderly-one-create-dex` for the `PUT /api/dex/social-card` endpoint.

Social card fields:
- `description` — DEX description text
- `banner` — Banner image for the card
- `logo` — Logo image for the card
- `tokenAddress` + `tokenChain` — Links to a project token (validated against GeckoTerminal)
- Social links (Telegram, Discord, X, website)

---

## Data Sources

- **Volume, PnL, fees**: Aggregated from Orderly Network's MySQL database (real trading data)
- **Token data**: CoinGecko / GeckoTerminal API
- **DEX metadata**: Orderly One's PostgreSQL database
- **Cache**: 1-minute TTL for leaderboard, 5-minute for stats

---

## Related Skills

- `orderly-one-general` — Overview, auth, chain IDs
- `orderly-one-create-dex` — Social card updates, board visibility toggle
- `orderly-one-graduation` — Required before appearing on leaderboard
