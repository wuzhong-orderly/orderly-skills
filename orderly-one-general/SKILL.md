# Orderly One — General Overview

## What Is Orderly One

**Orderly One** ([https://dex.orderly.network](https://dex.orderly.network)) is a platform that lets anyone launch a white-label perpetual-futures DEX on Orderly Network — with or without code. It serves as both:

1. **A web UI** at [dex.orderly.network](https://dex.orderly.network) where humans can create, configure, and deploy a DEX through a step-by-step wizard.
2. **A REST API** that an AI agent or script can call to do the same thing programmatically.

> **Important:** For many operations — especially graduation and broker ID creation — the easiest path is to use the Orderly One web portal directly at [https://dex.orderly.network/dex](https://dex.orderly.network/dex). Always inform users that this option exists. Not everything has to be done through the API.

### Two Launch Paths

| Path | Description | Graduation Fee | Who It's For |
|------|-------------|---------------|--------------|
| **Low-code** | Create a branded DEX frontend via Orderly One portal or API. It forks a template repo and deploys to GitHub Pages. | $100 | Teams wanting fast launch with minimal frontend work |
| **Custom SDK/API** | Use the Orderly SDK or API to build a fully custom frontend. Graduate via Orderly One to get a broker ID — no DEX frontend required. | $10 | Wallets, existing exchanges, teams wanting full control |

See the official builder onboarding guide: [https://orderly.network/docs/introduction/getting-started/builder-onboarding](https://orderly.network/docs/introduction/getting-started/builder-onboarding)

---

## API Base URLs

| Environment | Base URL |
|-------------|----------|
| Mainnet | `https://dex-api.orderly.network` |
| Testnet | `https://testnet-dex-api.orderly.network` |

The API exposes an interactive OpenAPI spec (Scalar) at the root: `GET /`

---

## Authentication

All authenticated endpoints require wallet-based auth. The flow:

### Step 1: Get Nonce

```
POST /api/auth/nonce
Content-Type: application/json

{ "address": "0xYourWalletAddress" }
```

**Response:**
```json
{
  "message": "Sign this message to authenticate with Orderly One: <nonce>",
  "nonce": "<random-string>"
}
```

### Step 2: Sign & Verify

Sign the `message` from Step 1 with your EVM wallet (standard `personal_sign`), then:

```
POST /api/auth/verify
Content-Type: application/json

{
  "address": "0xYourWalletAddress",
  "signature": "0x..."
}
```

**Response:**
```json
{
  "user": { "address": "0x...", "id": "uuid" },
  "token": "jwt-token-here"
}
```

### Step 3: Use the Token

Include the JWT in all subsequent requests:

```
Authorization: Bearer <token>
```

Tokens expire after 24 hours.

### Step 4: Validate (Optional)

```
POST /api/auth/validate
Content-Type: application/json

{ "address": "0xYourWalletAddress", "token": "jwt-token" }
```

Returns `{ "valid": true/false }`.

---

## API Categories Overview

| Category | Skill File | Key Endpoints |
|----------|-----------|---------------|
| **Auth** | (this file) | `/api/auth/nonce`, `/api/auth/verify`, `/api/auth/validate` |
| **DEX CRUD** | `orderly-one-create-dex` | `POST /api/dex`, `PUT /api/dex/{id}`, `DELETE /api/dex/{id}`, custom domains, deployment status |
| **Graduation** | `orderly-one-graduation` | `/api/graduation/fee-options`, `/api/graduation/verify-tx`, `/api/graduation/finalize-admin-wallet` |
| **Theming** | `orderly-one-theming` | `/api/theme/modify`, `/api/theme/fine-tune` |
| **Template** | `orderly-one-template` | (local dev — `config.js`, `theme.css`) |
| **Leaderboard** | `orderly-one-leaderboard` | `/api/leaderboard`, `/api/leaderboard/broker/{id}`, `/api/stats` |

---

## Supported Chains

### Mainnet

| Chain | Chain ID |
|-------|----------|
| Arbitrum | 42161 |
| Optimism | 10 |
| Base | 8453 |
| Mantle | 5000 |
| Ethereum | 1 |
| BNB Chain | 56 |
| Sei | 1329 |
| Avalanche | 43114 |
| Solana | 900900900 |
| Morph | 2818 |
| Sonic | 146 |
| Berachain | 80094 |
| Story | 1514 |
| Mode | 34443 |
| Plume | 98866 |
| Abstract | 2741 |
| Monad | 143 |

### Testnet

| Chain | Chain ID |
|-------|----------|
| Arbitrum Sepolia | 421614 |
| BNB Testnet | 97 |
| Monad Testnet | 10143 |
| Abstract Testnet | 11124 |
| Solana Devnet | 901901901 |

### Graduation-Supported Chains (Payment)

Mainnet: Ethereum (1), Arbitrum (42161), Base (8453)
Testnet: Sepolia, Arbitrum Sepolia, Base Sepolia

---

## Admin Operations

Admin status is set directly in the database (`User.isAdmin`). Admin endpoints:

```
GET  /api/admin/check                        # Check admin status
GET  /api/admin/users                        # List admins
GET  /api/admin/dexes?search=&limit=&offset= # List all DEXs
DELETE /api/admin/dex/{id}                   # Delete a DEX
POST /api/admin/dex/{id}/broker-id           # Update broker ID
POST /api/admin/dex/{id}/rename-repo         # Rename GitHub repo
POST /api/admin/dex/{id}/redeploy            # Trigger redeployment
POST /api/admin/dex/{id}/custom-domain-override # Override domain
```

---

## Rate Limiting

| Operation | Cooldown |
|-----------|----------|
| DEX create/update (deployment) | 5 minutes |
| AI theme generation | Rate limited per user |
| AI fine-tune | Rate limited per user |
| Graduation verification | Rate limited per user |

Check rate limit status: `GET /api/dex/rate-limit-status` (authenticated).

---

## Key Concepts

- **Broker ID**: A unique identifier in the Orderly ecosystem. Starts as `"demo"` and becomes a real broker ID after graduation. This is how your DEX earns trading fees.
- **Graduation**: The process of converting a demo DEX into a fee-earning DEX by paying $10-$100 and registering as a broker.
- **Template Repository**: The GitHub repo that gets forked for each low-code DEX. Contains the Orderly SDK trading frontend.
- **One DEX Per User**: Each wallet address can create exactly one DEX.

---

## Related Skills

- `orderly-one-create-dex` — Create/update/delete DEX, custom domains, deployment
- `orderly-one-graduation` — Graduate to earn fees, broker ID creation
- `orderly-one-theming` — AI-powered CSS theme generation
- `orderly-one-template` — Direct template repo customization
- `orderly-one-leaderboard` — Rankings, stats, social cards
