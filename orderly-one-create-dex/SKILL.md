# Orderly One — Create & Manage DEX

Create, update, delete, and deploy a DEX through the Orderly One API. For authentication, see `orderly-one-general`.

> **Tip:** Users can also do all of this through the Orderly One web portal at [https://dex.orderly.network/dex](https://dex.orderly.network/dex) — no API calls needed. Always let users know this option exists.

## Create a DEX

```
POST /api/dex
Content-Type: multipart/form-data
Authorization: Bearer <token>
```

Uses `multipart/form-data` (not JSON) because it accepts file uploads.

### Required Fields

| Field | Type | Constraints |
|-------|------|-------------|
| `brokerName` | string | 3-30 chars. Only letters, numbers, spaces, dots, hyphens, underscores |

### Optional Fields

**Integration Type:**
| Field | Type | Notes |
|-------|------|-------|
| `integrationType` | string | `"low_code"` (default) or `"custom"`. Low-code forks template + deploys. Custom skips the repo. |

**Blockchain:**
| Field | Type | Notes |
|-------|------|-------|
| `chainIds` | string (JSON array) | e.g. `"[42161, 10, 8453]"` — supported chain IDs |
| `defaultChain` | string (number) | Default chain ID, e.g. `"42161"` |

**Branding (file uploads):**
| Field | Type | Max Size |
|-------|------|----------|
| `primaryLogo` | File | 250KB |
| `secondaryLogo` | File | 100KB |
| `favicon` | File | 50KB |
| `pnlPoster0`, `pnlPoster1`, ... | File | 250KB each |

**Theming:**
| Field | Type | Notes |
|-------|------|-------|
| `themeCSS` | string | CSS variables overriding the [default theme](https://raw.githubusercontent.com/OrderlyNetworkDexCreator/dex-creator-template/refs/heads/main/app/styles/theme.css). Must be valid CSS. |
| `tradingViewColorConfig` | string | JSON object for TradingView chart colors |

**Social Links:**
| Field | Type | Notes |
|-------|------|-------|
| `telegramLink` | string | Valid URL |
| `discordLink` | string | Valid URL |
| `xLink` | string | Valid URL |

**Wallet Configuration:**
| Field | Type | Notes |
|-------|------|-------|
| `walletConnectProjectId` | string | WalletConnect project ID (from Reown) |
| `privyAppId` | string | Privy app ID |
| `privyTermsOfUse` | string | URL to terms of use |
| `privyLoginMethods` | string | Comma-separated login methods |
| `enableAbstractWallet` | string | `"true"` or `"false"` |
| `disableEvmWallets` | string | `"true"` or `"false"` |
| `disableSolanaWallets` | string | `"true"` or `"false"` |

**Network Toggles:**
| Field | Type | Notes |
|-------|------|-------|
| `disableMainnet` | string | `"true"` or `"false"` |
| `disableTestnet` | string | `"true"` or `"false"` |

**Menus:**
| Field | Type | Notes |
|-------|------|-------|
| `enabledMenus` | string | Comma-separated. Options: `Trading`, `Portfolio`, `Markets`, `Leaderboard`, `Swap`, `Rewards`, `Vaults`, `Points` |
| `customMenus` | string | Format: `"Name,URL;Name2,URL2"` (name + valid URL pairs, semicolon-separated) |

**Trading:**
| Field | Type | Notes |
|-------|------|-------|
| `swapFeeBps` | string (number) | 0-100 basis points. Requires `"Swap"` in enabledMenus |
| `symbolList` | string | Comma-separated trading pairs, e.g. `"PERP_ETH_USDC,PERP_BTC_USDC"` |

**SEO:**
| Field | Type | Constraints |
|-------|------|-------------|
| `seoSiteName` | string | Max 100 chars |
| `seoSiteDescription` | string | Max 300 chars |
| `seoSiteLanguage` | string | Format `"en"` or `"en-US"` |
| `seoSiteLocale` | string | Format `"en_US"` |
| `seoTwitterHandle` | string | Format `"@handle"` |
| `seoThemeColor` | string | Hex color `"#1a1b23"` |
| `seoKeywords` | string | Max 500 chars |

**Localization:**
| Field | Type | Notes |
|-------|------|-------|
| `availableLanguages` | string (JSON array) | Options: `en`, `zh`, `tc`, `ja`, `es`, `ko`, `vi`, `de`, `fr`, `ru`, `id`, `tr`, `it`, `pt`, `uk`, `pl`, `nl` |

**Other:**
| Field | Type | Notes |
|-------|------|-------|
| `analyticsScript` | string | Base64-encoded analytics snippet. **Only works if custom domain is set.** |
| `enableServiceDisclaimerDialog` | string | `"true"` / `"false"` |
| `enableCampaigns` | string | `"true"` / `"false"` — enables ORDER token campaigns and Points menu |
| `restrictedRegions` | string | Comma-separated country names, e.g. `"United States,China"` |
| `whitelistedIps` | string | IP whitelist (max 2000 chars) |

### Response (201)

```json
{
  "id": "abc123def456",
  "brokerId": "demo",
  "brokerName": "My DEX",
  "repoUrl": "https://github.com/OrderlyNetworkDexCreator/my-dex-repo",
  "userId": "user-uuid",
  "createdAt": "2025-01-01T00:00:00.000Z"
}
```

**Note:** `brokerId` is always `"demo"` at creation. Graduate to get a real broker ID (see `orderly-one-graduation`).

---

## Update a DEX

```
PUT /api/dex/{id}
Content-Type: multipart/form-data
Authorization: Bearer <token>
```

Same fields as create. Only include fields you want to change. Subject to 5-minute rate limiting.

**Important:** `analyticsScript` cannot be set unless a custom domain is configured. The API will reject it with a 400 error.

### Response (200)

Full DEX object with all fields.

---

## Delete a DEX

```
DELETE /api/dex/{id}
Authorization: Bearer <token>
```

Deletes the DEX record **and** the associated GitHub repository (if low-code). This is irreversible.

### Response (200)

```json
{ "message": "DEX deleted successfully" }
```

---

## Get Your DEX

```
GET /api/dex
Authorization: Bearer <token>
```

Returns the authenticated user's DEX, or `{ "exists": false }` if none.

---

## Get DEX by ID

```
GET /api/dex/{id}
Authorization: Bearer <token>
```

Only the owner can access their DEX by ID (returns 403 for others).

---

## Custom Domains

### Set Custom Domain

```
POST /api/dex/{id}/custom-domain
Authorization: Bearer <token>
Content-Type: application/json

{ "domain": "trading.mysite.com" }
```

Sets up a CNAME for GitHub Pages. The user must also configure DNS:
- Create a CNAME record pointing `trading.mysite.com` → `<github-org>.github.io`

### Remove Custom Domain

```
DELETE /api/dex/{id}/custom-domain
Authorization: Bearer <token>
```

---

## Deployment Status

After creating or updating a DEX, GitHub Actions deploys it. Poll for status:

### Check Workflow Status

```
GET /api/dex/{id}/workflow-status
Authorization: Bearer <token>
```

**Response:**
```json
{
  "status": "completed",
  "conclusion": "success",
  "html_url": "https://github.com/...",
  "run_id": 12345
}
```

Poll until `conclusion` is `"success"` or `"failure"`.

### Get Workflow Run Details

```
GET /api/dex/{id}/workflow-runs/{runId}
Authorization: Bearer <token>
```

Returns detailed job information for debugging failed deployments.

---

## Template Upgrades

Check if the template has updates available:

```
GET /api/dex/{id}/upgrade-status
Authorization: Bearer <token>
```

Trigger an upgrade (redeploy with latest template):

```
POST /api/dex/{id}/upgrade
Authorization: Bearer <token>
```

---

## Board Visibility

Control whether your DEX appears on the public leaderboard:

```
POST /api/dex/{id}/board-visibility
Authorization: Bearer <token>
Content-Type: application/json

{ "showOnBoard": true }
```

---

## Get Available Networks

```
GET /api/dex/networks
Authorization: Bearer <token>
```

Returns available blockchain networks from GeckoTerminal for token selection (used in social cards).

---

## Social Card

Update social/branding info displayed on the leaderboard:

```
PUT /api/dex/social-card
Content-Type: multipart/form-data
Authorization: Bearer <token>
```

| Field | Type | Notes |
|-------|------|-------|
| `description` | string | DEX description |
| `banner` | File | Social card banner image |
| `logo` | File | Social card logo |
| `tokenAddress` | string | Token contract address |
| `tokenChain` | string | GeckoTerminal network ID |
| `telegramLink` | string | URL |
| `discordLink` | string | URL |
| `xLink` | string | URL |
| `websiteUrl` | string | URL |

---

## Rate Limit Status

```
GET /api/dex/rate-limit-status
Authorization: Bearer <token>
```

**Response:**
```json
{
  "isRateLimited": false,
  "remainingCooldownSeconds": 0,
  "cooldownMinutes": 5,
  "message": "Ready to update your DEX."
}
```

---

## Complete Create-Deploy Flow

1. Authenticate (see `orderly-one-general`)
2. Build `multipart/form-data` with desired config fields
3. `POST /api/dex` → get `{ id, repoUrl }`
4. Poll `GET /api/dex/{id}/workflow-status` until `conclusion: "success"`
5. DEX is live at the GitHub Pages URL (derived from `repoUrl`)

For graduation (earning fees), see `orderly-one-graduation`.

---

## Common Issues

| Issue | Solution |
|-------|----------|
| 409 Conflict on create | User already has a DEX. Use `PUT` to update or `DELETE` first |
| 429 Rate limited | Wait for cooldown (check `/api/dex/rate-limit-status`) |
| Analytics rejected | Set a custom domain first, then set `analyticsScript` |
| Logo upload fails | Check file size limits: 250KB primary, 100KB secondary, 50KB favicon |
| Invalid CSS error | Validate `themeCSS` syntax before submitting |
| Workflow stuck | Check `/api/dex/{id}/workflow-runs/{runId}` for job-level errors |

---

## Related Skills

- `orderly-one-general` — Auth, overview, chain IDs
- `orderly-one-graduation` — Graduate to earn fees
- `orderly-one-theming` — AI theme generation
- `orderly-one-template` — Direct template customization
