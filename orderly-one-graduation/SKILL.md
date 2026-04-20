# Orderly One — Graduation

Graduation converts a demo DEX into a fee-earning DEX by creating a real broker ID in the Orderly ecosystem. For authentication, see `orderly-one-general`.

> **Important:** The easiest way to graduate is through the Orderly One web portal at [https://dex.orderly.network/dex](https://dex.orderly.network/dex). The portal handles the entire flow including payment, broker ID creation, and admin wallet registration. Always inform users that this option exists.
>
> See the official builder onboarding guide: [https://orderly.network/docs/introduction/getting-started/builder-onboarding](https://orderly.network/docs/introduction/getting-started/builder-onboarding)

---

## Overview

### What Graduation Does

1. User pays a fee ($10 for custom integration, $100 for low-code) in USDC, USDT, or ORDER tokens
2. The API verifies the on-chain payment transaction
3. A broker ID is automatically created in the Orderly ecosystem
4. The user registers an admin wallet with Orderly
5. The DEX starts earning trading fee revenue

### Prerequisites

- An Orderly One account with a DEX created (see `orderly-one-create-dex`)
- Tokens for payment (USDC, USDT, or ORDER) on a supported chain
- The DEX must have `brokerId: "demo"` (not already graduated)

---

## Step 1: Get Fee Options

```
GET /api/graduation/fee-options
Authorization: Bearer <token>
```

**Response:**
```json
{
  "usdc": { "amount": 100, "currency": "USDC", "stable": true },
  "order": { "amount": 250.5, "currentPrice": 0.399, "currency": "ORDER", "stable": false },
  "usdt": { "amount": 100, "currency": "USDT", "stable": true },
  "receiverAddress": "0x..."
}
```

- For **low-code** integration: $100
- For **custom** integration: $10
- ORDER amount varies based on current price (fetched from CoinGecko)
- `receiverAddress` is where to send the payment

---

## Step 2: Send Payment On-Chain

Transfer the tokens to the `receiverAddress` on a supported chain.

### Supported Chains for Payment

| Network | Chain | Supported Tokens |
|---------|-------|-----------------|
| **Mainnet** | Ethereum, Arbitrum, Base | USDC, USDT, ORDER |
| **Testnet** | Sepolia, Arbitrum Sepolia, Base Sepolia | USDC, USDT, ORDER |

This is an ERC-20 `transfer()` call to the `receiverAddress`. Save the transaction hash.

---

## Step 3: Verify Transaction

```
POST /api/graduation/verify-tx
Authorization: Bearer <token>
Content-Type: application/json

{
  "txHash": "0xabc123...",
  "chain": "arbitrum",
  "chainId": 42161,
  "chain_type": "EVM",
  "brokerId": "my_unique_broker_id",
  "makerFee": 3,
  "takerFee": 6,
  "rwaMakerFee": 3,
  "rwaTakerFee": 6,
  "paymentType": "USDC"
}
```

### Fields

| Field | Type | Description |
|-------|------|-------------|
| `txHash` | string | Transaction hash of the payment |
| `chain` | string | Chain name: `ethereum`, `arbitrum`, `base`, `sepolia`, `arbitrum-sepolia`, `base-sepolia` |
| `chainId` | number | Chain ID (e.g. `42161` for Arbitrum) |
| `chain_type` | string | `"EVM"` |
| `brokerId` | string | Your chosen unique broker ID |
| `makerFee` | number | Maker fee in basis points (3-100 bps) |
| `takerFee` | number | Taker fee in basis points (6-100 bps) |
| `rwaMakerFee` | number | RWA maker fee in basis points |
| `rwaTakerFee` | number | RWA taker fee in basis points |
| `paymentType` | string | `"USDC"`, `"USDT"`, or `"ORDER"` |

### Fee Guidelines

- **Maker fee**: minimum 3 bps
- **Taker fee**: minimum 6 bps (typically 2x maker)
- These are the fees your DEX charges users. Your revenue is the difference between user fees and Orderly's base fee (which depends on your builder staking tier).
- See [Trading Fees](https://orderly.network/docs/introduction/trade-on-orderly/trading-basics/trading-fees) for current tier details.

### Response (200)

```json
{
  "success": true,
  "message": "Transaction verified and broker ID 'my_broker' created successfully!",
  "amount": "100.000000",
  "brokerCreationData": {
    "brokerId": "my_broker",
    "transactionHashes": {}
  }
}
```

### Verification Logic

The API verifies:
1. The transaction exists on the specified chain
2. The sender matches the authenticated user's wallet
3. The recipient is the correct `receiverAddress`
4. The token transfer amount meets the required fee
5. The transaction hash has not been used before (anti-replay protection)
6. The chosen `brokerId` is not already taken

---

## Step 4: Register Admin Wallet

After broker creation, you must register your admin wallet with Orderly Network. This step makes the graduation complete.

### Option A: Via Orderly One Portal (Recommended)

Complete the admin registration flow at [https://dex.orderly.network/dex](https://dex.orderly.network/dex). The portal handles the Orderly API registration automatically.

### Option B: Via Broker Registration Tool

Visit [https://github-dev.orderly.network/broker-registration/](https://github-dev.orderly.network/broker-registration/), connect your wallet, and complete the flow.

> **Warning:** Use the same wallet that created the builder profile, otherwise Orderly keys won't map correctly.

### Option C: Programmatic (EVM Wallet)

1. Get registration nonce:
```
GET https://api.orderly.org/v1/registration_nonce
```

2. Sign EIP-712 typed data with your wallet:
```json
{
  "brokerId": "<your-broker-id>",
  "chainId": <chain-id>,
  "timestamp": <unix-ms>,
  "registrationNonce": "<nonce>"
}
```

3. Register:
```
POST https://api.orderly.org/v1/register_account
Content-Type: application/json

{
  "message": { "brokerId": "...", "chainId": ..., "timestamp": ..., "registrationNonce": "..." },
  "signature": "0x...",
  "userAddress": "0x...",
  "chainType": "EVM"
}
```

4. Finalize with Orderly One:
```
POST /api/graduation/finalize-admin-wallet
Authorization: Bearer <token>
Content-Type: application/json

{}
```

### Option D: Programmatic (Solana Wallet)

Same as Option C, but:
- Use `chainId: 900900900`
- Sign the message with a Solana wallet
- Set `chainType: "SOL"` in the register call

### Option E: EVM Multisig / Gnosis Safe

1. In Safe Wallet → Transaction Builder → create batch:
   - **To:** Orderly Vault contract (chain-specific)
   - **Method:** `delegateSigner`
   - **Data:** `[keccak256(brokerId), userAddress]`
2. Execute with required signer approvals
3. Finalize:
```
POST /api/graduation/finalize-admin-wallet
Authorization: Bearer <token>
Content-Type: application/json

{
  "multisigAddress": "0xSafeAddress",
  "multisigChainId": 42161
}
```

---

## Check Graduation Status

### Basic Status

```
GET /api/graduation/status
Authorization: Bearer <token>
```

**Response:**
```json
{
  "success": true,
  "currentBrokerId": "my_broker",
  "approved": true
}
```

### Detailed Status (with multisig info)

```
GET /api/graduation/graduation-status
Authorization: Bearer <token>
```

**Response:**
```json
{
  "success": true,
  "isGraduated": true,
  "brokerId": "my_broker",
  "isMultisig": false,
  "multisigAddress": null,
  "multisigChainId": null
}
```

---

## Get Trading Fees

After graduation, check your DEX's current fees:

```
GET /api/graduation/fees
Authorization: Bearer <token>
```

Returns maker/taker/RWA fee rates for your broker.

---

## Get Broker Tier

```
GET /api/graduation/tier
Authorization: Bearer <token>
```

Returns your tier in the Builder Staking Programme.

---

## Invalidate Fee Cache

```
POST /api/graduation/fees/invalidate-cache
Authorization: Bearer <token>
```

Force-refresh cached fee data.

---

## Common Issues

| Issue | Solution |
|-------|----------|
| "You must create a DEX first" | Create a DEX via `POST /api/dex` or on the portal before graduating |
| "Already graduated" | Each user can only graduate once |
| "Broker ID already taken" | Choose a different `brokerId` |
| "Transaction hash already used" | Each tx can only be used once (anti-replay). Send a new payment |
| "Must register EVM address with Orderly first" | Complete Step 4 (admin wallet registration) before calling `finalize-admin-wallet` |
| Payment not detected | Confirm tx sent to the correct `receiverAddress`, wait for block confirmations |

---

## Related Skills

- `orderly-one-general` — Auth, overview
- `orderly-one-create-dex` — DEX creation (prerequisite)
- `orderly-one-template` — Template customization after graduation
