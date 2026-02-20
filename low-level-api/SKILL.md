---
name: low-level-api
description: Interact directly with Orderly Network's low-level REST API (EVM). Explicitly separates Public and Private endpoints with authentication details.
argument-hint: <command> [params]
---

# Low Level API Skill

Use this skill to access the raw REST API. Includes full authentication details.

## Overview

Orderly Network is a **decentralized perpetual futures trading infrastructure** that enables omnichain trading. This API provides direct access to:

- **Trading**: Create, modify, and cancel orders for perpetual futures
- **Position Management**: Monitor open positions, PnL, liquidation prices
- **Account Management**: Handle deposits, withdrawals, leverage settings
- **Market Data**: Access orderbooks, klines, funding rates, and market info
- **Risk Management**: Set TP/SL orders, manage margin and collateral

### When to Use Low-Level API vs SDK
| Use Case | Recommendation |
| :--- | :--- |
| Building a trading bot | Low-Level API (full control) |
| Building a React trading UI | JS-SDK hooks (simpler integration) |
| Server-side trading logic | Low-Level API |
| Custom integrations | Low-Level API |
| Quick prototyping | JS-SDK |

## Configuration
- **Testnet:** `https://testnet-api.orderly.org`
- **Mainnet:** `https://api.orderly.org`

## Authentication & Signing
Private endpoints require the following headers.

### Required Headers
| Header | Description |
| :--- | :--- |
| `orderly-account-id` | Your unique account ID (keccak256 hash of address + broker) |
| `orderly-key` | Your API public key (ed25519, Base58-encoded with `ed25519:` prefix, e.g., `ed25519:5FdHdjGy1tZRWgUhKUYQUAtatuw4N2kJNhkeuKF5VvWn`) |
| `orderly-timestamp` | Current timestamp in milliseconds |
| `orderly-signature` | Base64URL-encoded ed25519 signature |

### Generating `orderly-key` from Secret Key
The public key must be derived from your secret key and encoded in **Base58** format with the `ed25519:` prefix:

```typescript
import * as ed from "@noble/ed25519";

// Base58 alphabet (Bitcoin-style)
const BASE58_ALPHABET = '123456789ABCDEFGHJKLMNPQRSTUVWXYZabcdefghijkmnopqrstuvwxyz';

function bytesToBase58(bytes: Uint8Array): string {
  // Count leading zeros
  let zeros = 0;
  for (let i = 0; i < bytes.length && bytes[i] === 0; i++) {
    zeros++;
  }
  
  // Convert to base58
  const digits: number[] = [0];
  for (let i = 0; i < bytes.length; i++) {
    let carry = bytes[i];
    for (let j = 0; j < digits.length; j++) {
      carry += digits[j] << 8;
      digits[j] = carry % 58;
      carry = (carry / 58) | 0;
    }
    while (carry > 0) {
      digits.push(carry % 58);
      carry = (carry / 58) | 0;
    }
  }
  
  // Build result string
  let result = '';
  for (let i = 0; i < zeros; i++) {
    result += BASE58_ALPHABET[0];
  }
  for (let i = digits.length - 1; i >= 0; i--) {
    result += BASE58_ALPHABET[digits[i]];
  }
  
  return result;
}

async function getOrderlyKey(secretKey: Uint8Array): Promise<string> {
  const publicKey = await ed.getPublicKeyAsync(secretKey);
  return 'ed25519:' + bytesToBase58(publicKey);
}
```

### Generating `orderly-signature`
1. **Construct the Message:**
   ```
   message = timestamp + method + requestPath + bodyString
   ```
   - `timestamp`: Same as header (e.g., `1680000000000`).
   - `method`: Uppercase (e.g., `POST`, `GET`).
   - `requestPath`: URL path including query params (e.g., `/v1/order?symbol=PERP_ETH_USDC`).
   - `bodyString`: JSON string of body parameters (if any). Empty string if none.

2. **Sign:**
   - Use your **API Secret Key** (ed25519) to sign the `message` bytes.

3. **Encode:**
   - Encode the signature in **Base64URL** format.

### Code Example (TypeScript)
```typescript
import * as ed from "@noble/ed25519";

async function signRequest(
  secretKey: Uint8Array,
  timestamp: number,
  method: string,
  path: string,
  body?: object
): Promise<string> {
  const bodyString = body ? JSON.stringify(body) : "";
  const message = `${timestamp}${method.toUpperCase()}${path}${bodyString}`;
  const signature = await ed.signAsync(
    new TextEncoder().encode(message),
    secretKey
  );
  return btoa(String.fromCharCode(...signature))
    .replace(/\+/g, "-")
    .replace(/\//g, "_")
    .replace(/=+$/, "");
}
```

### Complete Authentication Example (TypeScript)
```typescript
import * as ed from "@noble/ed25519";

// Base58 encoding (required for orderly-key)
const BASE58_ALPHABET = '123456789ABCDEFGHJKLMNPQRSTUVWXYZabcdefghijkmnopqrstuvwxyz';

function bytesToBase58(bytes: Uint8Array): string {
  let zeros = 0;
  for (let i = 0; i < bytes.length && bytes[i] === 0; i++) zeros++;
  
  const digits: number[] = [0];
  for (let i = 0; i < bytes.length; i++) {
    let carry = bytes[i];
    for (let j = 0; j < digits.length; j++) {
      carry += digits[j] << 8;
      digits[j] = carry % 58;
      carry = (carry / 58) | 0;
    }
    while (carry > 0) {
      digits.push(carry % 58);
      carry = (carry / 58) | 0;
    }
  }
  
  let result = BASE58_ALPHABET[0].repeat(zeros);
  for (let i = digits.length - 1; i >= 0; i--) {
    result += BASE58_ALPHABET[digits[i]];
  }
  return result;
}

async function makeAuthenticatedRequest(
  baseUrl: string,
  path: string,
  accountId: string,
  secretKey: Uint8Array,
  method: string = 'GET',
  body?: object
): Promise<any> {
  const timestamp = Date.now();
  
  // Generate signature
  const bodyString = body ? JSON.stringify(body) : "";
  const message = `${timestamp}${method.toUpperCase()}${path}${bodyString}`;
  const signature = await ed.signAsync(new TextEncoder().encode(message), secretKey);
  const signatureBase64Url = btoa(String.fromCharCode(...signature))
    .replace(/\+/g, "-")
    .replace(/\//g, "_")
    .replace(/=+$/, "");
  
  // Derive public key (orderly-key) - must be Base58 with ed25519: prefix
  const publicKey = await ed.getPublicKeyAsync(secretKey);
  const orderlyKey = 'ed25519:' + bytesToBase58(publicKey);
  
  const response = await fetch(`${baseUrl}${path}`, {
    method,
    headers: {
      'Content-Type': 'application/json',
      'orderly-account-id': accountId,
      'orderly-key': orderlyKey,
      'orderly-timestamp': timestamp.toString(),
      'orderly-signature': signatureBase64Url
    },
    body: body ? JSON.stringify(body) : undefined
  });
  
  return response.json();
}

// Example usage:
// const secretKey = base58ToBytes("DmLpCDU8PMTSXfhW5UxzsYjNsg8r6HxKfLqr4scqcHxb");
// const result = await makeAuthenticatedRequest(
//   "https://testnet-api-evm.orderly.org",
//   "/v1/client/holding",
//   "0x...",
//   secretKey
// );
```

## Response Format

### Success Response
```json
{
  "success": true,
  "data": { ... }
}
```

### Error Response
```json
{
  "success": false,
  "code": -1005,
  "message": "order_price must be a positive number"
}
```

### Common Error Codes
| Code | Description |
| :--- | :--- |
| `-1000` | Unknown error |
| `-1001` | Invalid request parameters |
| `-1002` | Unauthorized (invalid signature) |
| `-1003` | Rate limit exceeded |
| `-1004` | Order not found |
| `-1005` | Invalid order parameter |
| `-1006` | Insufficient balance |

## Rate Limits
- **General:** 10 req/s per IP (unless specified otherwise)
- HTTP 429 returned when rate limited

## Symbol Format
Orderly uses `PERP_<BASE>_USDC` format (e.g., `PERP_ETH_USDC`, `PERP_BTC_USDC`).

---

## Private Endpoints (Requires Auth)
These endpoints require a valid signature and API key headers.

### Sub-Accounts

> **Use Case:** Sub-accounts allow you to segregate trading activities, useful for:
> - Running multiple trading strategies separately
> - Managing risk by isolating positions
> - Operating on behalf of multiple clients (for brokers)
>
> Only the **main account** can manage sub-accounts. Each sub-account has its own positions and orders but shares the main account's API keys.

#### `POST` /v1/client/add_sub_account
- **Summary:** Add sub-account
- **Use Case:** Create a new sub-account for isolated trading or strategy segregation
- **Limit:** 10 req/s
- **Request Body:**
  ```json
  {
    "description": "Trading sub-account"
  }
  ```
- **Response Data:**
  | Field | Type | Description |
  | :--- | :--- | :--- |
  | `sub_account_id` | string | The new sub-account's unique identifier |


#### `GET` /v1/client/sub_account
- **Summary:** Get sub-account list
- **Use Case:** Retrieve all sub-accounts to display in account management UI or to iterate over for aggregate stats
- **Limit:** 10 req/s
- **Parameters:** None
- **Response Data:**
  | Field | Type | Description |
  | :--- | :--- | :--- |
  | `rows[].sub_account_id` | string | Sub-account identifier |
  | `rows[].description` | string | Custom description |
  | `rows[].created_time` | number | Creation timestamp (ms) |


#### `POST` /v1/client/update_sub_account
- **Summary:** Update sub-account
- **Use Case:** Modify the description of a sub-account for better organization
- **Limit:** 10 req/s
- **Request Body:**
  ```json
  {
    "sub_account_id": "0x...",
    "description": "Updated description"
  }
  ```


#### `GET` /v1/client/aggregate/positions
- **Summary:** Get aggregate positions (all sub-accounts)
- **Use Case:** Get a consolidated view of all positions across all sub-accounts. Useful for:
  - Portfolio overview dashboards
  - Total exposure calculation
  - Cross-account risk monitoring
- **Limit:** 10 req/s
- **Parameters:** None


#### `GET` /v1/client/aggregate/holding
- **Summary:** Get aggregate holding (all sub-accounts)
- **Limit:** 10 req/s
- **Parameters:** None


#### `POST` /v1/sub_account_settle_pnl
- **Summary:** Settle sub-account PnL
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `orderly_account_id` | header | string | Yes |

---

### Positions

> **Use Case:** Position APIs are essential for:
> - **Real-time Trading UI**: Display current positions, PnL, and risk metrics
> - **Risk Management**: Monitor liquidation prices and margin ratios
> - **Portfolio Tracking**: Calculate total exposure and unrealized profits
> - **Trading Bots**: Check position status before placing new orders
>
> The SDK's `usePositionStream` hook wraps these APIs with WebSocket updates for real-time data.

#### `GET` /v1/positions
- **Summary:** Get All Positions Info
- **Use Case:** Core endpoint for position monitoring. Use this to:
  - Display all open positions in a trading dashboard
  - Calculate portfolio-level PnL and risk metrics
  - Check current exposure before placing new orders
- **Limit:** 3 req/s
- **Parameters:** None
- **Response Data:**
  | Field | Type | Description |
  | :--- | :--- | :--- |
  | `rows[]` | array | List of open positions |
  | `rows[].symbol` | string | Trading pair (e.g., `PERP_ETH_USDC`) |
  | `rows[].position_qty` | number | Position size (positive=long, negative=short) |
  | `rows[].average_open_price` | number | Average entry price |
  | `rows[].mark_price` | number | Current mark price for PnL calculation |
  | `rows[].unrealized_pnl` | number | Unrealized PnL based on mark price |
  | `rows[].unsettled_pnl` | number | PnL not yet settled to balance |
  | `rows[].est_liq_price` | number\|null | Estimated liquidation price (null if no position) |
  | `rows[].leverage` | number | Current leverage for this position |
  | `rows[].mmr` | number | Maintenance margin ratio |
  | `rows[].imr` | number | Initial margin ratio |
  | `rows[].pnl_24_h` | number | PnL in last 24 hours |
  | `rows[].fee_24_h` | number | Fees paid in last 24 hours |
  | `margin_ratio` | number | Account margin ratio (lower = higher risk) |
  | `free_collateral` | number | Available collateral for new positions |
  | `total_collateral_value` | number | Total collateral value in USDC |
  | `total_unreal_pnl` | number | Total unrealized PnL across all positions |
- **Response Example:**
  ```json
  {
    "success": true,
    "data": {
      "margin_ratio": 15.5,
      "free_collateral": 5000.00,
      "total_collateral_value": 10000.00,
      "total_unreal_pnl": 150.50,
      "rows": [
        {
          "symbol": "PERP_ETH_USDC",
          "position_qty": 0.5,
          "cost_position": 750.00,
          "last_sum_unitary_funding": 0.0,
          "pending_long_qty": 0,
          "pending_short_qty": 0,
          "settle_price": 1500.00,
          "average_open_price": 1500.00,
          "unrealized_pnl": 10.50,
          "unsettled_pnl": 5.25,
          "mark_price": 1521.00,
          "est_liq_price": 1200.00,
          "timestamp": 1680000000000,
          "mmr": 0.025,
          "imr": 0.05,
          "leverage": 10,
          "pnl_24_h": 25.00,
          "fee_24_h": 0.75
        }
      ]
    }
  }
  ```


#### `GET` /v1/position/{symbol}
- **Summary:** Get One Position Info
- **Use Case:** Get position details for a specific symbol. More efficient than fetching all positions when you only need one:
  - Check position before closing
  - Validate TP/SL order placement
  - Display single-symbol position widget
- **Limit:** 3 req/s
- **Parameters:**
  | Name | In | Type | Required | Description |
  | :--- | :--- | :--- | :--- | :--- |
  | `symbol` | path | string | Yes | Trading pair (e.g., `PERP_ETH_USDC`) |


### `GET` /v1/position_history
- **Summary:** Get Position History
- **Use Case:** Retrieve historical position data for:
  - Trading performance analysis
  - Tax reporting and record keeping
  - Strategy backtesting validation
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required | Description |
  | :--- | :--- | :--- | :--- | :--- |
  | `symbol` | query | string | No | Filter by trading pair |
  | `limit` | query | number | No | Max results to return |
- **Response Data:**
  | Field | Type | Description |
  | :--- | :--- | :--- |
  | `rows[].symbol` | string | Trading pair |
  | `rows[].realized_pnl` | number | Realized profit/loss |
  | `rows[].avg_entry_price` | number | Average entry price |
  | `rows[].avg_exit_price` | number | Average exit price |
  | `rows[].closed_time` | number | Position close timestamp |


### `GET` /v1/funding_fee/history
- **Summary:** Get Funding Fee History
- **Use Case:** Track funding fee payments for:
  - Cost analysis and optimization
  - Tax reporting
  - Strategy profitability calculation (funding is a major cost in perps)
- **Limit:** 20 req/min
- **Parameters:**
  | Name | In | Type | Required | Description |
  | :--- | :--- | :--- | :--- | :--- |
  | `symbol` | query | string | No | Filter by trading pair |
  | `start_t` | query | string | No | Start timestamp (ms) |
  | `end_t` | query | string | No | End timestamp (ms) |
  | `page` | query | string | No | Page number |
  | `size` | query | string | No | Page size |
- **Response Data:**
  | Field | Type | Description |
  | :--- | :--- | :--- |
  | `rows[].symbol` | string | Trading pair |
  | `rows[].funding_rate` | number | Funding rate at payment time |
  | `rows[].mark_price` | number | Mark price at payment time |
  | `rows[].funding_fee` | number | Fee paid (negative = you paid, positive = you received) |
  | `rows[].payment_type` | string | `PAY` (paid) or `RECEIVE` (received) |
  | `rows[].created_time` | number | Payment timestamp |


### `GET` /v1/client/liquidator_liquidations
- **Summary:** Get Liquidated Positions by Liquidator
- **Use Case:** For liquidator bots to track their liquidation activity and earnings
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required | Description |
  | :--- | :--- | :--- | :--- | :--- |
  | `symbol` | query | string | No | Filter by trading pair |
  | `start_t` | query | number | No | Start timestamp (ms) |
  | `end_t` | query | number | No | End timestamp (ms) |
  | `page` | query | number | No | Page number |
  | `size` | query | number | No | Page size |


### `GET` /v1/client/leverage
- **Summary:** Get Leverage Setting
- **Use Case:** Check current leverage setting before:
  - Displaying leverage slider in UI
  - Calculating max position size
  - Validating order parameters
- **Limit:** 1 req/s
- **Parameters:**
  | Name | In | Type | Required | Description |
  | :--- | :--- | :--- | :--- | :--- |
  | `symbol` | query | string | Yes | Trading pair to check leverage for |
- **Response Data:**
  | Field | Type | Description |
  | :--- | :--- | :--- |
  | `leverage` | number | Current leverage setting (e.g., 10 for 10x) |
  | `max_leverage` | number | Maximum allowed leverage for this symbol |


### `POST` /v1/client/leverage
- **Summary:** Update Leverage Setting
- **Use Case:** Change leverage before placing orders. Higher leverage = larger position with less margin but higher liquidation risk
- **Limit:** 5 req/min
- **Request Body:**
  ```json
  {
    "symbol": "PERP_ETH_USDC",
    "leverage": 10
  }
  ```
- **Body Parameters:**
  | Name | Type | Required | Description |
  | :--- | :--- | :--- | :--- |
  | `symbol` | string | Yes | Trading pair |
  | `leverage` | number | Yes | New leverage (1-50, depends on symbol) |
- **Error Cases:**
  | Error | Cause |
  | :--- | :--- |
  | `leverage exceeds max` | Requested leverage > symbol's max_leverage |
  | `position exists` | Cannot change leverage with open position (close first) |
- **Parameters:** None


### `GET` /v1/liquidations
- **Summary:** Get Liquidated Positions of Account
- **Use Case:** Track your own liquidation history for:
  - Risk analysis and strategy adjustment
  - Understanding what went wrong
  - Insurance fund claim eligibility
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required | Description |
  | :--- | :--- | :--- | :--- | :--- |
  | `symbol` | query | string | No | Filter by trading pair |
  | `start_t` | query | number | No | Start timestamp (ms) |
  | `end_t` | query | number | No | End timestamp (ms) |
  | `page` | query | number | No | Page number |
  | `size` | query | number | No | Page size |
  | `sort_by` | query | string | No | Sort field |
  | `liquidation_id` | query | number | No | Specific liquidation ID |
- **Response Data:**
  | Field | Type | Description |
  | :--- | :--- | :--- |
  | `rows[].liquidation_id` | number | Unique liquidation identifier |
  | `rows[].symbol` | string | Trading pair that was liquidated |
  | `rows[].position_qty` | number | Position size that was liquidated |
  | `rows[].liquidator_fee` | number | Fee paid to liquidator |
  | `rows[].insurance_fund_fee` | number | Fee to insurance fund |
  | `rows[].created_time` | number | Liquidation timestamp |


### `POST` /v1/liquidation
- **Summary:** Claim Liquidated Positions
- **Use Case:** For liquidators to claim positions being liquidated. Liquidators earn fees for providing this service.
- **Limit:** 5 req/s
- **Parameters:** None

---

### Orders

> **Use Case:** Order APIs are the core of trading functionality:
> - **Trading Bots**: Automated order placement and management
> - **Trading UI**: Manual order entry with validation
> - **Order Management**: View, edit, and cancel orders
>
> The SDK's `useOrderEntry` hook provides order validation (max qty, est. liquidation price) before submission.
> 
> **Order Types Explained:**
> | Type | Behavior |
> | :--- | :--- |
> | `LIMIT` | Execute at specified price or better |
> | `MARKET` | Execute immediately at best available price |
> | `IOC` | Immediate-or-Cancel: Fill what's available, cancel rest |
> | `FOK` | Fill-or-Kill: Fill entire order or cancel completely |
> | `POST_ONLY` | Only add liquidity (maker), never take (reject if would take) |
> | `ASK` | Limit sell at best ask price |
> | `BID` | Limit buy at best bid price |

#### `POST` /v1/order
- **Summary:** Create Order
- **Use Case:** Place a new order. This is the primary trading endpoint used by:
  - Trading bots for automated strategies
  - UI order forms for manual trading
  - Arbitrage systems for cross-venue trading
- **Limit:** 10 req/s
- **Request Body:**
  ```json
  {
    "symbol": "PERP_ETH_USDC",
    "order_type": "LIMIT",
    "order_price": 1500.00,
    "order_quantity": 0.1,
    "side": "BUY",
    "client_order_id": "my-order-123",
    "reduce_only": false,
    "visible_quantity": 0.1
  }
  ```
- **Body Parameters:**
  | Name | Type | Required | Description |
  | :--- | :--- | :--- | :--- |
  | `symbol` | string | Yes | Trading pair (e.g., `PERP_ETH_USDC`) |
  | `order_type` | string | Yes | `LIMIT`, `MARKET`, `IOC`, `FOK`, `POST_ONLY`, `ASK`, `BID` |
  | `side` | string | Yes | `BUY` (long) or `SELL` (short) |
  | `order_price` | number | No* | Price for LIMIT orders. Required for LIMIT, optional for MARKET |
  | `order_quantity` | number | No* | Order size in base currency (e.g., 0.1 ETH). Use this OR order_amount |
  | `order_amount` | number | No* | For MARKET orders, size in quote currency (e.g., 100 USDC) |
  | `client_order_id` | string | No | Custom order ID (max 36 chars) for your tracking |
  | `reduce_only` | boolean | No | If true, only reduce existing position, never increase |
  | `visible_quantity` | number | No | For iceberg orders: visible portion (rest is hidden) |
  | `slippage` | number | No | Max slippage % for MARKET orders (e.g., 0.01 = 1%) |
  | `order_tag` | string | No | Custom tag for analytics/tracking |
- **Response Data:**
  | Field | Type | Description |
  | :--- | :--- | :--- |
  | `order_id` | number | Orderly's order ID (use for cancel/edit) |
  | `client_order_id` | string | Your custom ID (if provided) |
  | `order_type` | string | Confirmed order type |
  | `order_price` | number | Confirmed order price |
  | `order_quantity` | number | Confirmed order quantity |
- **Response:**
  ```json
  {
    "success": true,
    "data": {
      "order_id": 12345,
      "client_order_id": "my-order-123",
      "order_type": "LIMIT",
      "order_price": 1500.00,
      "order_quantity": 0.1
    }
  }
  ```
- **Error Cases:**
  | Error Code | Cause |
  | :--- | :--- |
  | `-1005` | Invalid price (negative, zero, or wrong precision) |
  | `-1006` | Insufficient balance/collateral |
  | `order_quantity too small` | Below symbol's `base_min` |
  | `order_quantity too large` | Above symbol's `base_max` |
  | `price out of range` | Price deviation > symbol's `price_range` |


#### `PUT` /v1/order
- **Summary:** Edit Order
- **Use Case:** Modify an existing unfilled/partially filled order without cancel+recreate:
  - Adjust price to improve fill probability
  - Change quantity based on market conditions
  - Faster than cancel+new order (maintains queue position for same price)
- **Limit:** 10 req/s
- **Request Body:**
  ```json
  {
    "order_id": 12345,
    "symbol": "PERP_ETH_USDC",
    "order_price": 1510.00,
    "order_quantity": 0.2
  }
  ```
- **Body Parameters:**
  | Name | Type | Required | Description |
  | :--- | :--- | :--- | :--- |
  | `order_id` | number | Yes | Order ID to edit |
  | `symbol` | string | Yes | Trading pair |
  | `order_price` | number | No | New price |
  | `order_quantity` | number | No | New quantity |
- **Error Cases:**
  | Error | Cause |
  | :--- | :--- |
  | `order not found` | Order ID doesn't exist or already filled |
  | `order already cancelled` | Order was previously cancelled |


#### `DELETE` /v1/order
- **Summary:** Cancel Order
- **Use Case:** Cancel a pending order:
  - Manual order management
  - Risk management (cancel before liquidation)
  - Strategy adjustment
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required | Description |
  | :--- | :--- | :--- | :--- | :--- |
  | `symbol` | query | string | Yes | Trading pair |
  | `order_id` | query | number | Yes | Order to cancel |
- **Response Data:**
  | Field | Type | Description |
  | :--- | :--- | :--- |
  | `order_id` | number | Confirmed cancelled order ID |
  | `status` | string | `CANCELLED` |


#### `GET` /v1/order/{order_id}
- **Summary:** Get Order by order_id
- **Use Case:** Check status of a specific order:
  - Verify order was placed successfully
  - Check fill status and executed quantity
  - Get order details for display
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required | Description |
  | :--- | :--- | :--- | :--- | :--- |
  | `order_id` | path | string | Yes | Order ID to query |
- **Response Data:**
  | Field | Type | Description |
  | :--- | :--- | :--- |
  | `order_id` | number | Order identifier |
  | `symbol` | string | Trading pair |
  | `side` | string | `BUY` or `SELL` |
  | `type` | string | Order type |
  | `price` | number | Order price |
  | `quantity` | number | Order quantity |
  | `executed` | number | Filled quantity |
  | `average_executed_price` | number | Average fill price |
  | `status` | string | `NEW`, `PARTIAL_FILLED`, `FILLED`, `CANCELLED` |
  | `total_fee` | number | Total fees paid |
  | `created_time` | number | Order creation timestamp |
  | `updated_time` | number | Last update timestamp |


#### `GET` /v1/orders
- **Summary:** Get Orders
- **Use Case:** List orders with filtering. Essential for:
  - Order management UI (show pending orders)
  - Order history display
  - Trading bot order tracking
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required | Description |
  | :--- | :--- | :--- | :--- | :--- |
  | `symbol` | query | string | No | Filter by symbol |
  | `side` | query | string | No | `BUY` or `SELL` |
  | `order_type` | query | string | No | `LIMIT`, `MARKET`, etc. |
  | `status` | query | string | No | `NEW`, `FILLED`, `PARTIAL_FILLED`, `CANCELLED` |
  | `start_t` | query | number | No | Start timestamp (ms) |
  | `end_t` | query | number | No | End timestamp (ms) |
  | `page` | query | number | No | Page number (default: 1) |
  | `size` | query | number | No | Page size (default: 25, max: 500) |
- **Response Data:**
  | Field | Type | Description |
  | :--- | :--- | :--- |
  | `rows[]` | array | List of orders |
  | `meta.total` | number | Total matching orders |
  | `meta.current_page` | number | Current page number |


#### `DELETE` /v1/orders
- **Summary:** Cancel All Pending Orders
- **Use Case:** Bulk cancel for:
  - Emergency risk management
  - Strategy reset
  - Position close preparation
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required | Description |
  | :--- | :--- | :--- | :--- | :--- |
  | `symbol` | query | string | No | Cancel only for this symbol (all if omitted) |

---

### Trades

> **Use Case:** Trade history is essential for:
> - **Performance Analysis**: Calculate realized PnL by trade
> - **Tax Reporting**: Track all executed trades for tax purposes
> - **Audit Trail**: Verify order execution quality
> - **Strategy Analysis**: Analyze entry/exit points and slippage

#### `GET` /v1/trades
- **Summary:** Get Trades
- **Use Case:** Retrieve executed trade history. Each order fill creates a trade record.
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required | Description |
  | :--- | :--- | :--- | :--- | :--- |
  | `symbol` | query | string | No | Filter by trading pair |
  | `start_t` | query | number | No | Start timestamp (ms) |
  | `end_t` | query | number | No | End timestamp (ms) |
  | `page` | query | number | No | Page number |
  | `size` | query | number | No | Page size |
- **Response Data:**
  | Field | Type | Description |
  | :--- | :--- | :--- |
  | `rows[].trade_id` | number | Unique trade identifier |
  | `rows[].order_id` | number | Parent order ID |
  | `rows[].symbol` | string | Trading pair |
  | `rows[].side` | string | `BUY` or `SELL` |
  | `rows[].executed_price` | number | Actual execution price |
  | `rows[].executed_quantity` | number | Filled quantity |
  | `rows[].fee` | number | Trading fee for this fill |
  | `rows[].fee_asset` | string | Fee token (usually `USDC`) |
  | `rows[].executed_timestamp` | number | Execution timestamp |
  | `rows[].is_maker` | boolean | True if you were the maker (added liquidity) |

---

### Assets

> **Use Case:** Asset/holding APIs provide collateral and balance information:
> - **Portfolio Display**: Show current token balances
> - **Trading Validation**: Check available collateral before orders
> - **Deposit/Withdraw Flow**: Verify balance changes

#### `GET` /v1/client/holding
- **Summary:** Get Current Holding
- **Use Case:** Get all token balances in your account. The `holding` field shows available balance, `frozen` shows locked in orders/positions.
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required | Description |
  | :--- | :--- | :--- | :--- | :--- |
  | `all` | query | boolean | No | If true, include zero-balance tokens |
- **Response Data:**
  | Field | Type | Description |
  | :--- | :--- | :--- |
  | `rows[].token` | string | Token symbol (e.g., `USDC`) |
  | `rows[].holding` | number | Available balance |
  | `rows[].frozen` | number | Locked in orders or as margin |
  | `rows[].pending_short` | number | Pending withdrawal amount |
  | `rows[].updated_time` | number | Last update timestamp |

---

### Account

> **Use Case:** Account APIs provide user-level information:
> - **Fee Rates**: Know your maker/taker fees for cost calculation
> - **Account Settings**: Max leverage, tier status
> - **User Statistics**: Trading volume, performance metrics

#### `GET` /v1/client/info
- **Summary:** Get Account Information
- **Use Case:** Core endpoint for account setup and fee calculation. Called by SDK's `useAccountInfo` hook.
  - Display account tier and fee rates
  - Calculate max leverage for UI
  - Check IMR (Initial Margin Rate) factors per symbol
- **Limit:** 10 req/min
- **Parameters:** None
- **Response Data:**
  | Field | Type | Description |
  | :--- | :--- | :--- |
  | `account_id` | string | Your unique account identifier |
  | `account_mode` | string | Account mode |
  | `tier` | string | Account tier (affects fees) |
  | `futures_tier` | string | Futures trading tier |
  | `taker_fee_rate` | number | Taker fee (you pay when taking liquidity) |
  | `maker_fee_rate` | number | Maker fee (often rebate for adding liquidity) |
  | `futures_taker_fee_rate` | number | Futures-specific taker fee |
  | `futures_maker_fee_rate` | number | Futures-specific maker fee |
  | `max_leverage` | number | Maximum allowed leverage |
  | `imr_factor` | object | Per-symbol IMR factors `{symbol: factor}` |
  | `max_notional` | object | Per-symbol max notional `{symbol: max}` |
  | `maintenance_cancel_orders` | boolean | Auto-cancel orders in maintenance mode |


#### `GET` /v1/client/statistics
- **Summary:** Get User Statistics
- **Use Case:** Retrieve aggregate trading statistics:
  - Display total volume on profile
  - Calculate VIP tier progress
  - Track overall performance
- **Limit:** 10 req/min
- **Parameters:** None
- **Response Data:**
  | Field | Type | Description |
  | :--- | :--- | :--- |
  | `total_volume` | number | Total trading volume (USDC) |
  | `total_pnl` | number | Total realized PnL |
  | `total_trades` | number | Number of trades executed |


#### `GET` /v1/asset/history
- **Summary:** Get Asset History
- **Use Case:** Track deposits and withdrawals for:
  - Transaction history display
  - Reconciliation
  - Audit purposes
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required | Description |
  | :--- | :--- | :--- | :--- | :--- |
  | `token` | query | string | No | Token symbol (e.g., `USDC`) |
  | `side` | query | string | No | `DEPOSIT` or `WITHDRAW` |
  | `status` | query | string | No | `PENDING`, `COMPLETED`, `FAILED` |
  | `start_t` | query | number | No | Start timestamp (ms) |
  | `end_t` | query | number | No | End timestamp (ms) |
  | `page` | query | number | No | Page number |
  | `size` | query | number | No | Page size |
- **Response Data:**
  | Field | Type | Description |
  | :--- | :--- | :--- |
  | `rows[].id` | string | Transaction ID |
  | `rows[].tx_id` | string | On-chain transaction hash |
  | `rows[].side` | string | `DEPOSIT` or `WITHDRAW` |
  | `rows[].token` | string | Token symbol |
  | `rows[].amount` | number | Transaction amount |
  | `rows[].fee` | number | Transaction fee |
  | `rows[].trans_status` | string | Transaction status |
  | `rows[].chain_id` | string | Source/destination chain ID |
  | `rows[].created_time` | number | Creation timestamp |


### `POST` /v1/claim_insurance_fund
- **Summary:** Claim Insurance Fund
- **Use Case:** After liquidation, if there's remaining collateral in the insurance fund attributable to you, claim it back.
- **Limit:** 5 req/s
- **Parameters:** None


### `POST` /v1/orderly_key
- **Summary:** Add Orderly Key
- **Limit:** 10 req/s
- **Parameters:** None


### `POST` /v1/client/remove_orderly_key
- **Summary:** Remove Orderly Key
- **Limit:** 10 req/s
- **Parameters:** None


### `POST` /v1/order/cancel_all_after
- **Summary:** Cancel All After
- **Limit:** 10 req/s
- **Parameters:** None


### `POST` /v1/batch-order
- **Summary:** Batch Create Order
- **Limit:** 10 req/s
- **Parameters:** None


### `DELETE` /v1/batch-order
- **Summary:** Batch Cancel Orders
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `order_ids` | query | string | Yes |


### `DELETE` /v1/client/order
- **Summary:** Cancel Order By client_order_id
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `symbol` | query | string | Yes |
  | `client_order_id` | query | string | Yes |


### `DELETE` /v1/client/batch-order
- **Summary:** Batch Cancel Orders By client_order_id
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `client_order_ids` | query | string | Yes |


### `GET` /v1/client/order/{client_order_id}
- **Summary:** Get Order by client_order_id
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `client_order_id` | path | string | Yes |


### `GET` /v1/orderbook/{symbol}
- **Summary:** Orderbook Snapshot
- **Use Case:** Get current order book depth for:
  - Display bid/ask ladder in trading UI
  - Calculate slippage for market orders
  - Analyze liquidity distribution
  - Best bid/ask price for limit orders
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required | Description |
  | :--- | :--- | :--- | :--- | :--- |
  | `symbol` | path | string | Yes | Trading pair |
  | `max_level` | query | integer | No | Depth levels to return (default: 100) |
- **Response Data:**
  | Field | Type | Description |
  | :--- | :--- | :--- |
  | `asks` | array | Sell orders `[[price, quantity], ...]` ascending by price |
  | `bids` | array | Buy orders `[[price, quantity], ...]` descending by price |
  | `timestamp` | number | Snapshot timestamp |
- **Response Example:**
  ```json
  {
    "success": true,
    "data": {
      "asks": [[1501.50, 2.5], [1502.00, 5.0], [1503.00, 10.0]],
      "bids": [[1500.00, 3.0], [1499.50, 7.5], [1499.00, 15.0]],
      "timestamp": 1680000000000
    }
  }
  ```


### `GET` /v1/kline
- **Summary:** Get Kline (Candlestick Data)
- **Use Case:** Get OHLCV data for charting:
  - Display candlestick charts
  - Technical analysis indicators
  - Historical price analysis
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required | Description |
  | :--- | :--- | :--- | :--- | :--- |
  | `symbol` | query | string | Yes | Trading pair |
  | `type` | query | string | Yes | Interval: `1m`, `5m`, `15m`, `30m`, `1h`, `4h`, `1d`, `1w`, `1M` |
  | `limit` | query | number | No | Number of candles (default: 100, max: 1000) |
- **Response Data:**
  | Field | Type | Description |
  | :--- | :--- | :--- |
  | `rows[].open` | number | Opening price |
  | `rows[].high` | number | Highest price |
  | `rows[].low` | number | Lowest price |
  | `rows[].close` | number | Closing price |
  | `rows[].volume` | number | Trading volume |
  | `rows[].start_timestamp` | number | Candle start time (ms) |


### `GET` /v1/order/{order_id}/trades
- **Summary:** Get All Trades of Specific Order
- **Use Case:** See all fills for a specific order (an order may fill in multiple trades)
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required | Description |
  | :--- | :--- | :--- | :--- | :--- |
  | `order_id` | path | number | Yes | Order ID to query trades for |


### `GET` /v1/trade/{trade_id}
- **Summary:** Get Trade
- **Use Case:** Get details of a specific trade by ID
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required | Description |
  | :--- | :--- | :--- | :--- | :--- |
  | `trade_id` | path | number | Yes | Trade ID to query |


### `POST` /v1/client/maintenance_config
- **Summary:** Set Maintenance Config
- **Limit:** 10 req/min
- **Parameters:** None


### `GET` /v1/volume/user/stats
- **Summary:** Get User Volume Statistics
- **Limit:** 10 req/s
- **Parameters:** None


### `GET` /v1/volume/user/daily
- **Summary:** Get User Daily Volume
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `start_date` | query | string | Yes |
  | `end_date` | query | string | Yes |


### `GET` /v1/withdraw_nonce
- **Summary:** Get Withdrawal Nonce
- **Use Case:** Get the next nonce for withdrawal requests. Nonces prevent replay attacks.
- **Limit:** 10 req/s
- **Parameters:** None
- **Response Data:**
  | Field | Type | Description |
  | :--- | :--- | :--- |
  | `withdraw_nonce` | number | Next nonce for withdrawal signature |


### `GET` /v1/settle_nonce
- **Summary:** Get Settle PnL Nonce
- **Use Case:** Get nonce for PnL settlement request (required for signature)
- **Limit:** 10 req/s
- **Parameters:** None


### `POST` /v1/withdraw_request
- **Summary:** Create Withdraw Request
- **Use Case:** Withdraw funds from Orderly to your wallet:
  - Transfer USDC back to wallet
  - Must have sufficient available balance (not in positions)
  - Requires nonce from `/v1/withdraw_nonce`
- **Limit:** 10 req/s
- **Request Body:**
  ```json
  {
    "token": "USDC",
    "amount": "100.00",
    "chain_id": 42161,
    "receiver": "0x...",
    "nonce": 1,
    "signature": "..."
  }
  ```
- **Parameters:** None


### `POST` /v1/settle_pnl
- **Summary:** Request PnL Settlement
- **Use Case:** Settle realized PnL to your account balance:
  - Convert unrealized PnL to available balance
  - Required before withdrawing profits
  - Happens automatically periodically, but can request manually
- **Limit:** 1 req/s
- **Parameters:** None


### `GET` /v1/pnl_settlement/history
- **Summary:** Get PnL Settlement History
- **Use Case:** Track PnL settlement events for accounting
- **Limit:** 20 req/s
- **Parameters:**
  | Name | In | Type | Required | Description |
  | :--- | :--- | :--- | :--- | :--- |
  | `start_t` | query | integer | No | Start timestamp |
  | `end_t` | query | integer | No | End timestamp |
  | `page` | query | integer | No | Page number |
  | `size` | query | integer | No | Page size |


### `POST` /v1/internal_transfer
- **Summary:** Create Internal Transfer
- **Use Case:** Transfer funds between sub-accounts or to other Orderly accounts
- **Limit:** 10 req/s
- **Request Body:**
  ```json
  {
    "token": "USDC",
    "amount": "100.00",
    "to_account_id": "0x..."
  }
  ```
- **Parameters:** None


### `GET` /v1/transfer_nonce
- **Summary:** Get Transfer Nonce
- **Use Case:** Get nonce for internal transfer signature
- **Limit:** 10 req/s
- **Parameters:** None


### `GET` /v1/internal_transfer_history
- **Summary:** Get Internal Transfer History
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `status` | query | string | No |
  | `start_t` | query | string | No |
  | `end_t` | query | string | No |
  | `page` | query | number | No |
  | `size` | query | number | No |
  | `side` | query | string | Yes |
  | `from_account_id` | query | string | No |
  | `to_account_id` | query | string | No |
  | `main_sub_only` | query | boolean | No |


### `GET` /v1/client/key_info
- **Summary:** Get Current Orderly Key Info
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `key_status` | query | string | No |


### `GET` /v1/client/orderly_key_ip_restriction
- **Summary:** Get Orderly Key IP Restriction
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `orderly_key` | query | string | Yes |


### `POST` /v1/client/set_orderly_key_ip_restriction
- **Summary:** Set Orderly Key IP Restriction
- **Limit:** 10 req/min
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `orderly_key` | query | string | Yes |
  | `ip_restriction_list` | query | string | Yes |


### `POST` /v1/client/reset_orderly_key_ip_restriction
- **Summary:** Reset Orderly Key IP Restriction
- **Limit:** 10 req/min
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `orderly_key` | query | string | Yes |
  | `reset_mode` | query | string | Yes |


### `GET` /v1/notification/inbox/notifications
- **Summary:** Get All Notifications
- **Limit:** 10 req/min
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `type` | query | string | No |
  | `page` | query | number | No |
  | `size` | query | number | No |


### `GET` /v1/notification/inbox/unread
- **Summary:** Get Unread Notifications
- **Limit:** 10 req/min
- **Parameters:** None


### `POST` /v1/notification/inbox/mark_read
- **Summary:** Set Read Status of Notifications
- **Limit:** 10 req/min
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `flag` | query | number | Yes |
  | `ids` | query | array | Yes |


### `POST` /v1/notification/inbox/mark_read_all
- **Summary:** Set Read Status of All Notifications
- **Limit:** 10 req/min
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `flag` | query | number | Yes |

---

### Algo Orders (TP/SL, Stop Orders)

> **Use Case:** Algo orders are conditional orders that execute when a trigger price is reached:
> - **Take-Profit (TP)**: Automatically close position at profit target
> - **Stop-Loss (SL)**: Limit losses by closing position at specified price
> - **Stop Orders**: Enter or exit positions based on price triggers
> - **Bracket Orders**: Combine entry with TP/SL in one order
>
> **Algo Order Types:**
> | Type | Use Case |
> | :--- | :--- |
> | `STOP` | Basic stop order: triggers when price reaches level |
> | `TP_SL` | Take-Profit or Stop-Loss for reducing position |
> | `POSITIONAL_TP_SL` | TP/SL tied to specific position (auto-cancels if position closes) |
> | `BRACKET` | Entry order with attached TP and SL |
>
> The SDK's `usePositionStream` hook returns positions with TP/SL info attached via `full_tp_sl` and `partial_tp_sl` fields.

#### `POST` /v1/algo/order
- **Summary:** Create Algo Order (Stop-Loss, Take-Profit, etc.)
- **Use Case:** Set up conditional orders for risk management:
  - Protect profits with take-profit
  - Limit losses with stop-loss
  - Enter positions at breakout levels
- **Limit:** 5 req/s
- **Request Body:**
  ```json
  {
    "symbol": "PERP_ETH_USDC",
    "algo_type": "STOP",
    "side": "SELL",
    "order_type": "MARKET",
    "quantity": 0.1,
    "trigger_price": 1400.00,
    "trigger_price_type": "MARK_PRICE",
    "reduce_only": true
  }
  ```
- **Body Parameters:**
  | Name | Type | Required | Description |
  | :--- | :--- | :--- | :--- |
  | `symbol` | string | Yes | Trading pair |
  | `algo_type` | string | Yes | `STOP`, `TP_SL`, `POSITIONAL_TP_SL`, `BRACKET` |
  | `side` | string | Yes | `BUY` or `SELL` |
  | `order_type` | string | Yes | `MARKET` (immediate fill at trigger) or `LIMIT` |
  | `quantity` | number | Yes | Order quantity |
  | `trigger_price` | number | Yes | Price that activates the order |
  | `trigger_price_type` | string | No | `MARK_PRICE` (default, safer) or `LAST_PRICE` |
  | `order_price` | number | No | Required for LIMIT orders |
  | `reduce_only` | boolean | No | Only reduce position (recommended for TP/SL) |
- **Response Data:**
  | Field | Type | Description |
  | :--- | :--- | :--- |
  | `algo_order_id` | number | Unique algo order identifier |
  | `client_order_id` | string | Your custom ID (if provided) |
- **Trigger Price Types:**
  - `MARK_PRICE`: Based on mark price (prevents manipulation, recommended)
  - `LAST_PRICE`: Based on last traded price (may be more volatile)


#### `PUT` /v1/algo/order
- **Summary:** Edit Algo Order
- **Use Case:** Modify trigger price or quantity of existing algo order
- **Limit:** 5 req/s
- **Request Body:**
  ```json
  {
    "algo_order_id": 12345,
    "trigger_price": 1450.00,
    "quantity": 0.2
  }
  ```


#### `DELETE` /v1/algo/order
- **Summary:** Cancel Algo Order
- **Use Case:** Remove pending algo order (e.g., remove outdated TP/SL)
- **Limit:** 5 req/s
- **Parameters:**
  | Name | In | Type | Required | Description |
  | :--- | :--- | :--- | :--- | :--- |
  | `symbol` | query | string | Yes | Trading pair |
  | `order_id` | query | number | Yes | Algo order ID to cancel |


#### `DELETE` /v1/algo/client/order
- **Summary:** Cancel Algo Order By client_order_id
- **Use Case:** Cancel using your custom order ID
- **Limit:** 5 req/s
- **Parameters:**
  | Name | In | Type | Required | Description |
  | :--- | :--- | :--- | :--- | :--- |
  | `symbol` | query | string | Yes | Trading pair |
  | `client_order_id` | query | string | Yes | Your custom order ID |


#### `GET` /v1/algo/orders
- **Summary:** Get Algo Orders
- **Use Case:** List all algo orders for display and management
- **Limit:** 5 req/s
- **Parameters:**
  | Name | In | Type | Required | Description |
  | :--- | :--- | :--- | :--- | :--- |
  | `symbol` | query | string | No | Filter by symbol |
  | `algo_type` | query | string | No | `STOP`, `TP_SL`, etc. |
  | `status` | query | string | No | `NEW`, `FILLED`, `CANCELLED`, `TRIGGERED` |
  | `side` | query | string | No | `BUY` or `SELL` |
  | `is_triggered` | query | string | No | `true` (already triggered) or `false` (waiting) |
  | `start_t` | query | number | No | Start timestamp (ms) |
  | `end_t` | query | number | No | End timestamp (ms) |
  | `page` | query | number | No | Page number |
  | `size` | query | number | No | Page size |
- **Response Data:**
  | Field | Type | Description |
  | :--- | :--- | :--- |
  | `rows[].algo_order_id` | number | Algo order identifier |
  | `rows[].symbol` | string | Trading pair |
  | `rows[].algo_type` | string | Order type |
  | `rows[].side` | string | BUY/SELL |
  | `rows[].trigger_price` | number | Trigger price |
  | `rows[].is_triggered` | boolean | Whether trigger has fired |
  | `rows[].is_activated` | boolean | Whether order is active |
  | `rows[].algo_status` | string | `NEW`, `TRIGGERED`, `FILLED`, `CANCELLED` |
  | `rows[].quantity` | number | Order quantity |
  | `rows[].created_time` | number | Creation timestamp |


#### `DELETE` /v1/algo/orders
- **Summary:** Cancel All Pending Algo Orders
- **Limit:** 5 req/s
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `symbol` | query | string | No |
  | `algo_type` | query | string | No |


#### `GET` /v1/algo/order/{order_id}
- **Summary:** Get Algo Order by order_id
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `order_id` | path | string | Yes |


#### `GET` /v1/algo/client/order/{client_order_id}
- **Summary:** Get Algo Order by client_order_id
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `client_order_id` | path | string | Yes |


#### `GET` /v1/algo/order/{order_id}/trades
- **Summary:** Get All Trades of Specific Algo Order
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `order_id` | path | number | Yes |

---

### Builder/Broker APIs

#### `GET` /v1/volume/broker/daily
- **Summary:** Get Builder's Users' Volumes
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `start_date` | query | string | Yes |
  | `end_date` | query | string | Yes |
  | `page` | query | number | No |
  | `size` | query | number | No |
  | `address` | query | string | No |
  | `order_tag` | query | string | No |
  | `aggregateBy` | query | string | No |
  | `sort` | query | string | No |


### `GET` /v1/broker/leaderboard/daily
- **Summary:** Get Builder's Leaderboard
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `start_date` | query | string | Yes |
  | `end_date` | query | string | Yes |
  | `page` | query | number | No |
  | `size` | query | number | No |
  | `order_tag` | query | string | No |
  | `broker_id` | query | string | No |
  | `sort` | query | string | No |
  | `address` | query | string | No |
  | `aggregateBy` | query | string | No |


### `GET` /v1/client/statistics/daily
- **Summary:** Get User Daily Statistics
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `start_date` | query | string | Yes |
  | `end_date` | query | string | Yes |
  | `page` | query | number | No |
  | `size` | query | number | No |
  | `include_historical_data` | query | boolean | No |


### `GET` /v1/broker/user_info
- **Summary:** Get User Fee Rates
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `account_id` | query | string | No |
  | `address` | query | string | No |
  | `page` | query | string | No |
  | `size` | query | string | No |


### `POST` /v1/broker/fee_rate/set
- **Summary:** Update User Fee Rate
- **Limit:** 1 req/s
- **Parameters:** None


### `POST` /v1/broker/fee_rate/set_default
- **Summary:** Reset User Fee Rate
- **Limit:** 1 req/s
- **Parameters:** None


### `GET` /v1/broker/fee_rate/default
- **Summary:** Get Default Builder Fee
- **Limit:** 1 req/s
- **Parameters:** None


### `POST` /v1/broker/fee_rate/default
- **Summary:** Update Default Builder Fee
- **Limit:** 1 req/s
- **Parameters:** None


### `POST` /v1/referral/create
- **Summary:** Create Referral Code
- **Limit:** 1 req/s
- **Parameters:** None


### `GET` /v1/referral/admin_info
- **Summary:** Get Referral Code Info
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `page` | query | string | No |
  | `size` | query | string | No |
  | `user_address` | query | string | No |
  | `account_id` | query | string | No |
  | `sort_by` | query | string | No |


### `POST` /v1/referral/update
- **Summary:** Update Referral Code
- **Limit:** 1 req/s
- **Parameters:** None


### `POST` /v1/referral/bind
- **Summary:** Bind Referral Code
- **Limit:** 1 req/s
- **Parameters:** None


### `GET` /v1/referral/info
- **Summary:** Get Referral Info
- **Limit:** 10 req/s
- **Parameters:** None


### `GET` /v1/referral/referral_history
- **Summary:** Get Referral History
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `start_date` | query | string | No |
  | `end_date` | query | string | No |
  | `page` | query | integer | No |
  | `size` | query | integer | No |


### `GET` /v1/referral/rebate_summary
- **Summary:** Get Referral Rebate Summary
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `start_date` | query | string | No |
  | `end_date` | query | string | No |
  | `page` | query | integer | No |
  | `size` | query | integer | No |


### `GET` /v1/referral/referee_history
- **Summary:** Get Referee History
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `start_date` | query | string | No |
  | `end_date` | query | string | No |
  | `page` | query | integer | No |
  | `size` | query | integer | No |


### `GET` /v1/referral/referee_info
- **Summary:** Get Referee Info
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `page` | query | integer | No |
  | `size` | query | integer | No |
  | `sort` | query | string | No |


### `GET` /v1/referral/referee_rebate_summary
- **Summary:** Get Referee Rebate Summary
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `start_date` | query | string | Yes |
  | `end_date` | query | string | Yes |


### `POST` /v1/referral/edit_split
- **Summary:** Edit Referral Code Split
- **Limit:** 1 req/s
- **Parameters:** None


### `POST` /v1/referral/auto_referral/update
- **Summary:** Builder admin update auto referral
- **Limit:** 1 req/s
- **Parameters:** None


### `GET` /v1/referral/auto_referral/info
- **Summary:** Builder admin get auto referral info
- **Limit:** 1 req/s
- **Parameters:** None


### `GET` /v1/referral/auto_referral/progress
- **Summary:** Get auto referral progress
- **Limit:** 1 req/s
- **Parameters:** None


### `POST` /v1/referral/edit_referral_code
- **Summary:** Edit Referral Code
- **Limit:** 10 req/s
- **Parameters:** None


### `GET` /v1/client/points
- **Summary:** Get User's Merits
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `address` | query | string | Yes |


### `GET` /v1/client/points/user_statistics
- **Summary:** Get User Statistics
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `stage` | query | number | Yes |


### `GET` /v1/admin/points/stage
- **Summary:** Get Stage Parameters
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `stage_id` | query | number | No |


### `POST` /v1/admin/points/stage
- **Summary:** Create/Update Stage Parameters
- **Limit:** 10 req/s
- **Parameters:** None


### `POST` /v1/delegate_withdraw_request
- **Summary:** Delegate Signer Withdraw Request
- **Limit:** 1 req/s
- **Parameters:** None


### `POST` /v1/delegate_settle_pnl
- **Summary:** Delegate Signer Settle PnL
- **Limit:** 1 req/s
- **Parameters:** None


### `GET` /v1/client/distribution_history
- **Summary:** Get Distribution History
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `status` | query | string | No |
  | `type` | query | string | No |
  | `start_t` | query | string | No |
  | `end_t` | query | string | No |
  | `page` | query | integer | No |
  | `size` | query | integer | No |


### `POST` /v1/client/campaign/sign_up
- **Summary:** Sign Up Campaign
- **Limit:** 10 req/s
- **Parameters:** None


### `GET` /v1/staking/balance
- **Summary:** Get Wallet's Current Staked Balance
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `address` | query | string | Yes |


### `GET` /v1/staking/unstake_details
- **Summary:** Get Unstaking ORDER Details
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `address` | query | string | Yes |


### `GET` /v1/staking/valor2/redeem
- **Summary:** Get Valor2 Redeem Info
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `address` | query | string | Yes |


### `GET` /v1/staking/valor/redeem
- **Summary:** Get Valor Redeem Info
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `address` | query | string | Yes |


### `GET` /v1/staking/esorder/vesting_list
- **Summary:** Get esORDER vesting list
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `address` | query | string | Yes |
  | `is_vesting` | query | boolean | Yes |


### `POST` /v1/sv/sp_orderly_key
- **Summary:** Add SP Orderly Key
- **Limit:** 10 req/s
- **Parameters:** None


### `POST` /v1/sv/sp_settle_pnl
- **Summary:** Request SP PnL Settlement
- **Limit:** 10 req/s
- **Parameters:** None


### `POST` /v1/sv/manual_period_delivery
- **Summary:** Trigger Manual Period Delivery
- **Limit:** 10 req/s
- **Parameters:** None


---

## Public Endpoints (No Auth Required)
These endpoints can be called without authentication headers.

### Market Data

> **Use Case:** Market data endpoints provide trading information:
> - **Trading UI**: Display prices, volumes, price changes
> - **Strategy Analysis**: Analyze market trends and liquidity
> - **Price Discovery**: Get current prices for order placement
>
> The SDK's `useMarkets`, `useMarkPrice`, `useIndexPrice` hooks wrap these endpoints.

#### `GET` /v1/public/futures
- **Summary:** Get Market Info for All Symbols
- **Use Case:** Core endpoint for initializing a trading app:
  - Get all available trading pairs
  - Load trading rules (min/max qty, tick sizes)
  - Display market overview table
- **Limit:** 10 req/s
- **Parameters:** None
- **Response Data:**
  | Field | Type | Description |
  | :--- | :--- | :--- |
  | `rows[].symbol` | string | Trading pair (e.g., `PERP_ETH_USDC`) |
  | `rows[].quote_min` | number | Minimum quote amount |
  | `rows[].quote_max` | number | Maximum quote amount |
  | `rows[].quote_tick` | number | Price tick size (precision) |
  | `rows[].base_min` | number | Minimum order quantity |
  | `rows[].base_max` | number | Maximum order quantity |
  | `rows[].base_tick` | number | Quantity tick size (precision) |
  | `rows[].min_notional` | number | Minimum order value in USDC |
  | `rows[].price_range` | number | Max price deviation from mark (e.g., 0.02 = 2%) |
  | `rows[].base_mmr` | number | Base maintenance margin rate |
  | `rows[].base_imr` | number | Base initial margin rate |
  | `rows[].imr_factor` | number | IMR scaling factor |
  | `rows[].funding_period` | number | Funding interval in hours |
  | `rows[].cap_funding` | number | Max funding rate cap |
  | `rows[].floor_funding` | number | Min funding rate floor |
- **Response:**
  ```json
  {
    "success": true,
    "data": {
      "rows": [
        {
          "symbol": "PERP_ETH_USDC",
          "quote_min": 0,
          "quote_max": 100000,
          "quote_tick": 0.01,
          "base_min": 0.001,
          "base_max": 1000,
          "base_tick": 0.001,
          "min_notional": 1,
          "price_range": 0.02,
          "created_time": 1680000000000,
          "updated_time": 1680000000000,
          "base_mmr": 0.05,
          "base_imr": 0.1,
          "imr_factor": 0.0001
        }
      ]
    }
  }
  ```


#### `GET` /v1/public/market_trades
- **Summary:** Get Recent Market Trades
- **Use Case:** Display recent trade feed (time & sales):
  - Show recent executions for a symbol
  - Analyze trade velocity and size distribution
  - Detect large trades
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required | Description |
  | :--- | :--- | :--- | :--- | :--- |
  | `symbol` | query | string | Yes | Trading pair (e.g., `PERP_ETH_USDC`) |
  | `limit` | query | integer | No | Number of trades (default: 10, max: 100) |
- **Response Data:**
  | Field | Type | Description |
  | :--- | :--- | :--- |
  | `rows[].symbol` | string | Trading pair |
  | `rows[].price` | number | Trade execution price |
  | `rows[].size` | number | Trade size |
  | `rows[].side` | string | `BUY` or `SELL` |
  | `rows[].ts` | number | Trade timestamp (ms) |


#### `GET` /v1/public/market_info/price_changes
- **Summary:** Get Price Info for All Symbols
- **Use Case:** Display price change statistics across all markets:
  - Market overview with 24h change %
  - Sorted leaderboard by gainers/losers
  - Price comparison across symbols
- **Limit:** 10 req/s
- **Parameters:** None
- **Response Data:**
  | Field | Type | Description |
  | :--- | :--- | :--- |
  | `rows[].symbol` | string | Trading pair |
  | `rows[].24h_open` | number | Opening price 24h ago |
  | `rows[].24h_close` | number | Current/latest price |
  | `rows[].24h_high` | number | 24h high |
  | `rows[].24h_low` | number | 24h low |
  | `rows[].24h_volume` | number | 24h trading volume (base) |
  | `rows[].24h_amount` | number | 24h trading volume (quote/USDC) |
  | `rows[].change` | number | 24h price change % |


#### `GET` /v1/public/market_info/traders_open_interests
- **Summary:** Get Open Interests for All Symbols
- **Use Case:** Analyze market positioning and liquidity:
  - See total open interest per symbol
  - Identify most active markets
  - Gauge market liquidity
- **Limit:** 10 req/s
- **Parameters:** None
- **Response Data:**
  | Field | Type | Description |
  | :--- | :--- | :--- |
  | `rows[].symbol` | string | Trading pair |
  | `rows[].open_interest` | number | Total open interest (contracts) |


#### `GET` /v1/public/market_info/history_charts
- **Summary:** Get Historical Price List for All Symbols
- **Use Case:** Display mini price charts on market overview page
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required | Description |
  | :--- | :--- | :--- | :--- | :--- |
  | `time_interval` | query | string | No | Interval for chart data |


### Funding Rates

> **Use Case:** Funding rate endpoints are critical for perpetual futures trading:
> - **Cost Estimation**: Funding payments are a major cost/income in perps
> - **Strategy Selection**: High funding = expensive to hold position
> - **Arbitrage**: Compare funding between exchanges
>
> **Funding Rate Basics:**
> - Paid every 8 hours (or as configured per symbol)
> - Positive rate: Longs pay shorts
> - Negative rate: Shorts pay longs
> - Formula: `funding_fee = position_size × mark_price × funding_rate`

#### `GET` /v1/public/funding_rates
- **Summary:** Get Predicted Funding Rates for All Markets
- **Use Case:** Display funding rates across all markets for comparison
- **Limit:** 10 req/s
- **Parameters:** None
- **Response Data:**
  | Field | Type | Description |
  | :--- | :--- | :--- |
  | `rows[].symbol` | string | Trading pair |
  | `rows[].est_funding_rate` | number | Predicted next funding rate |
  | `rows[].last_funding_rate` | number | Last settled funding rate |
  | `rows[].next_funding_time` | number | Next funding timestamp (ms) |
  | `rows[].sum_unitary_funding` | number | Cumulative funding since position open |


#### `GET` /v1/public/funding_rate/{symbol}
- **Summary:** Get Predicted Funding Rate for One Market
- **Use Case:** Get detailed funding info for a specific symbol before trading
- **Limit:** 30 req/s
- **Parameters:**
  | Name | In | Type | Required | Description |
  | :--- | :--- | :--- | :--- | :--- |
  | `symbol` | path | string | Yes | Trading pair (e.g., `PERP_ETH_USDC`) |


#### `GET` /v1/public/funding_rate_history
- **Summary:** Get Funding Rate History for One Market
- **Use Case:** Analyze historical funding patterns:
  - Calculate average funding cost over time
  - Identify funding rate trends
  - Backtest funding-based strategies
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required | Description |
  | :--- | :--- | :--- | :--- | :--- |
  | `symbol` | query | string | Yes | Trading pair |
  | `start_t` | query | number | No | Start timestamp (ms) |
  | `end_t` | query | number | No | End timestamp (ms) |
  | `page` | query | number | No | Page number |
  | `size` | query | number | No | Page size |
- **Response Data:**
  | Field | Type | Description |
  | :--- | :--- | :--- |
  | `rows[].symbol` | string | Trading pair |
  | `rows[].funding_rate` | number | Funding rate at that period |
  | `rows[].funding_timestamp` | number | Timestamp of funding |


### `GET` /v1/public/futures/{symbol}
- **Summary:** Get Market Info for One Symbol
- **Use Case:** Get detailed trading rules for a specific symbol before placing orders
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required | Description |
  | :--- | :--- | :--- | :--- | :--- |
  | `symbol` | path | string | Yes | Trading pair (e.g., `PERP_ETH_USDC`) |


### `GET` /v1/public/liquidation
- **Summary:** Get Positions Under Liquidation
- **Use Case:** For liquidator bots to find claimable liquidated positions
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required | Description |
  | :--- | :--- | :--- | :--- | :--- |
  | `start_t` | query | number | No | Start timestamp (ms) |
  | `end_t` | query | number | No | End timestamp (ms) |
  | `page` | query | integer | No | Page number |
  | `size` | query | integer | No | Page size |


### `GET` /v1/public/liquidated_positions
- **Summary:** Get Liquidated Positions Info
- **Use Case:** View history of liquidated positions across the platform
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required | Description |
  | :--- | :--- | :--- | :--- | :--- |
  | `symbol` | query | string | No | Filter by symbol |
  | `start_t` | query | string | No | Start timestamp |
  | `end_t` | query | string | No | End timestamp |
  | `page` | query | string | No | Page number |
  | `size` | query | string | No | Page size |


### `GET` /v1/public/info/{symbol}
- **Summary:** Get Order Rules per Symbol
- **Use Case:** Get detailed order constraints for a symbol:
  - Price tick size for order validation
  - Quantity precision
  - Min/max order sizes
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required | Description |
  | :--- | :--- | :--- | :--- | :--- |
  | `symbol` | path | string | Yes | Trading pair |
- **Response Data:**
  | Field | Type | Description |
  | :--- | :--- | :--- |
  | `symbol` | string | Trading pair |
  | `quote_min` | number | Minimum quote amount |
  | `quote_max` | number | Maximum quote amount |
  | `quote_tick` | number | Price tick size |
  | `base_min` | number | Minimum order size |
  | `base_max` | number | Maximum order size |
  | `base_tick` | number | Quantity tick size |
  | `min_notional` | number | Minimum order value |


### `GET` /v1/public/token
- **Summary:** Get Supported Collateral Info
- **Use Case:** Get information about supported tokens for:
  - Deposit UI (show which tokens can be deposited)
  - Collateral value calculation
  - Multi-collateral support
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required | Description |
  | :--- | :--- | :--- | :--- | :--- |
  | `chain_id` | query | string | No | Filter by chain ID |
- **Response Data:**
  | Field | Type | Description |
  | :--- | :--- | :--- |
  | `rows[].token` | string | Token symbol |
  | `rows[].token_hash` | string | Token hash identifier |
  | `rows[].decimals` | number | Token decimals |
  | `rows[].minimum_withdraw_amount` | number | Minimum withdrawal |
  | `rows[].is_collateral` | boolean | Can be used as collateral |
  | `rows[].discount_factor` | number | Collateral discount (haircut) |
  | `rows[].chain_details` | array | Per-chain contract addresses |


### `GET` /v1/public/info
- **Summary:** Get Available Symbols
- **Use Case:** Get list of all tradeable symbols with their trading rules
- **Limit:** 10 req/s
- **Parameters:** None


### `GET` /v1/public/index_price_source
- **Summary:** Get Index Price Source
- **Limit:** 10 req/s
- **Parameters:** None


### `GET` /v1/public/insurancefund
- **Summary:** Get Insurance Fund Info
- **Limit:** 10 req/s
- **Parameters:** None


### `GET` /v1/tv/config
- **Summary:** Get TradingView Localized Config Info
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `locale` | query | string | Yes |


### `GET` /v1/tv/history
- **Summary:** Get TradingView History Bars
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `symbol` | query | string | Yes |
  | `resolution` | query | string | Yes |
  | `from` | query | string | Yes |
  | `to` | query | string | Yes |


### `GET` /v1/tv/symbol_info
- **Summary:** Get TradingView Symbol Info
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `group` | query | string | Yes |


### `GET` /v1/public/leverage
- **Summary:** Get Max Leverage Setting
- **Limit:** 10 req/s
- **Parameters:** None


### `GET` /v1/public/system_info
- **Summary:** Get System Maintenance Status
- **Limit:** 1 req/s
- **Parameters:** None

---

### Account Registration

#### `GET` /v1/registration_nonce
- **Summary:** Get Registration Nonce
- **Limit:** 10 req/s
- **Parameters:** None
- **Response:**
  ```json
  {
    "success": true,
    "data": {
      "registration_nonce": "123456789"
    }
  }
  ```


#### `GET` /v1/get_account
- **Summary:** Check if Wallet is Registered
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required | Description |
  | :--- | :--- | :--- | :--- | :--- |
  | `address` | query | string | Yes | Wallet address |
  | `broker_id` | query | string | Yes | Builder/broker ID |
  | `chain_type` | query | string | No | `EVM` or `SOLANA` |


#### `GET` /v1/get_all_accounts
- **Summary:** Check Account Details of an Address
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `address` | query | string | Yes |
  | `broker_id` | query | string | Yes |
  | `chain_type` | query | string | No |


#### `GET` /v1/get_broker
- **Summary:** Check if Address is Registered
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `address` | query | string | Yes |
  | `chain_type` | query | string | No |


#### `POST` /v1/register_account
- **Summary:** Register Account
- **Limit:** 10 req/s
- **Request Body:**
  ```json
  {
    "message": {
      "brokerId": "demo",
      "chainId": 421614,
      "timestamp": 1680000000000,
      "registrationNonce": "123456789"
    },
    "signature": "0x...",
    "userAddress": "0x..."
  }
  ```


#### `GET` /v1/get_orderly_key
- **Summary:** Get Orderly Key
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `account_id` | query | string | Yes |
  | `orderly_key` | query | string | Yes |

---

### System Configuration

#### `GET` /v1/public/config
- **Summary:** Get Leverage Configuration
- **Limit:** 10 req/s
- **Parameters:** None


#### `GET` /v1/public/chain_info
- **Summary:** Get Supported Chains per Builder
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `broker_id` | query | string | No |


### `GET` /v1/tv/kline_history
- **Summary:** Get Kline History
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `symbol` | query | string | Yes |
  | `resolution` | query | string | Yes |
  | `from` | query | string | No |
  | `to` | query | string | No |
  | `limit` | query | number | No |


### `GET` /v1/public/vault_balance
- **Summary:** Get Vault Balance
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `chain_id` | query | string | No |
  | `token` | query | string | No |


### `POST` /v1/faucet/usdc
- **Summary:** Get Faucet USDC(Testnet Only)
- **Limit:** 10 req/s
- **Parameters:** None


### `GET` /v1/public/account
- **Summary:** Check if Account Exists
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `account_id` | query | string | No |


### `GET` /v1/public/announcement
- **Summary:** Get Announcements
- **Limit:** 10 req/s
- **Parameters:** None


### `GET` /v1/public/referral/check_ref_code
- **Summary:** Check Referral Code
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `account_id` | query | string | Yes |


### `GET` /v1/public/referral/verify_ref_code
- **Summary:** Verify Referral Code
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `referral_code` | query | string | Yes |


### `GET` /v1/public/points/leaderboard
- **Summary:** Get Merits Leaderboard
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `start_r` | query | integer | No |
  | `end_r` | query | integer | No |
  | `epoch_id` | query | integer | No |
  | `page` | query | integer | No |
  | `size` | query | integer | No |


### `GET` /v1/public/points/epoch_dates
- **Summary:** Get Start and End Date of All Epochs
- **Limit:** 10 req/s
- **Parameters:** None


### `GET` /v1/public/points/rankings
- **Summary:** Get Stage Rankings
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `stage` | query | number | Yes |
  | `page` | query | number | No |
  | `size` | query | number | No |
  | `period` | query | string | Yes |


### `GET` /v1/public/points/epoch
- **Summary:** Get Number of Merits for Distribution
- **Limit:** 10 req/s
- **Parameters:** None


### `GET` /v1/public/points/stages
- **Summary:** Get Information About Stages
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `stage_id` | query | number | No |
  | `status` | query | string | No |
  | `broker_id` | query | string | Yes |


### `DELETE` /v1/admin/points/stage
- **Summary:** Delete Stage
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `stage_id` | query | number | Yes |


### `POST` /v1/delegate_signer
- **Summary:** Delegate Signer
- **Limit:** 1 req/s
- **Parameters:** None


### `POST` /v1/delegate_orderly_key
- **Summary:** Add Delegate Signer Orderly Key
- **Limit:** 1 req/s
- **Parameters:** None


### `GET` /v1/public/campaigns
- **Summary:** Get List of Campaigns
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `start_t` | query | integer | No |
  | `end_t` | query | integer | No |
  | `only_show_alive` | query | string | No |
  | `address` | query | string | No |


### `GET` /v1/public/campaign/stats
- **Summary:** Get Campaign Statistics
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `campaign_id` | query | integer | Yes |
  | `broker_id` | query | string | No |
  | `symbol` | query | string | No |


### `GET` /v1/public/campaign/stats/details
- **Summary:** Get Detailed Campaign Info
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `campaign_id` | query | integer | Yes |
  | `broker_id` | query | string | No |
  | `symbols` | query | string | No |
  | `group_by` | query | string | No |


### `GET` /v1/public/campaign/ranking
- **Summary:** Get Campaign Ranking
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `campaign_id` | query | integer | Yes |
  | `broker_id` | query | string | No |
  | `page` | query | integer | No |
  | `size` | query | integer | No |
  | `sort_by` | query | string | No |
  | `min_pnl` | query | integer | No |
  | `min_volume` | query | integer | No |
  | `aggregate_by` | query | string | No |


### `GET` /v1/public/campaign/user
- **Summary:** Get Campaign User Info
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `campaign_id` | query | integer | Yes |
  | `account_id` | query | string | No |
  | `address` | query | string | No |
  | `broker_id` | query | string | No |
  | `symbol` | query | array | No |
  | `order_tag` | query | string | No |


### `GET` /v1/public/campaign/check
- **Summary:** Get Campaign Verification
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `campaign_id` | query | number | Yes |
  | `type` | query | string | Yes |
  | `address` | query | string | Yes |
  | `lower_boundary` | query | number | Yes |
  | `cmp` | query | string | No |


### `GET` /v1/staking/overview
- **Summary:** Get Staking Overview
- **Limit:** 10 req/s
- **Parameters:** None


### `GET` /v1/staking/valor/batch_info
- **Summary:** Get Valor Batch Info
- **Limit:** 10 req/s
- **Parameters:** None


### `GET` /v1/staking/valor/pool_info
- **Summary:** Get Valor Pool Info
- **Limit:** 10 req/s
- **Parameters:** None


### `GET` /v1/staking/valor2/pool_info
- **Summary:** Get Valor2 Pool Info
- **Limit:** 10 req/s
- **Parameters:** None


### `GET` /v1/staking/valor2/batch_info
- **Summary:** Get Valor2 Batch Info
- **Limit:** 10 req/s
- **Parameters:** None


### `GET` /v1/staking/valor2/revenue_buyback
- **Summary:** Get Valor2 Revenue Buyback
- **Limit:** 10 req/s
- **Parameters:** None


### `GET` /v1/public/trading_rewards/epoch_info
- **Summary:** Get Parameters of Each Epoch for All Epochs
- **Limit:** 10 req/s
- **Parameters:** None


### `GET` /v1/public/trading_rewards/epoch_data
- **Summary:** Get Epochs Data
- **Limit:** 10 req/s
- **Parameters:** None


### `GET` /v1/public/trading_rewards/broker_allocation_history
- **Summary:** Get Broker Allocation History
- **Limit:** 10 req/s
- **Parameters:** None


### `GET` /v1/public/trading_rewards/wallet_rewards_history
- **Summary:** Get Wallet Trading Rewards History
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `address` | query | string | Yes |
  | `chain_type` | query | string | No |


### `GET` /v1/public/trading_rewards/account_rewards_history
- **Summary:** Get Account Trading Rewards History
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `address` | query | string | Yes |
  | `chain_type` | query | string | No |


### `GET` /v1/public/trading_rewards/status
- **Summary:** Get the Status of Trading Rewards Programme
- **Limit:** 10 req/s
- **Parameters:** None


### `GET` /v1/public/trading_rewards/symbol_category
- **Summary:** Get Symbol Rewards Category
- **Limit:** 10 req/s
- **Parameters:** None


### `GET` /v1/public/market_making_rewards/epoch_info
- **Summary:** Get Parameters of Each Epoch
- **Limit:** 10 req/s
- **Parameters:** None


### `GET` /v1/public/market_making_rewards/status
- **Summary:** Get the Status of Market Making Rewards Programme
- **Limit:** 10 req/s
- **Parameters:** None


### `GET` /v1/public/market_making_rewards/symbol_params
- **Summary:** Get Symbol Rewards Parameters
- **Limit:** 10 req/s
- **Parameters:** None


### `GET` /v1/public/market_making_rewards/leaderboard
- **Summary:** Leaderboard for market maker rewards
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `epoch` | query | number | Yes |
  | `market` | query | string | No |


### `GET` /v1/public/market_making_rewards/current_epoch_estimate
- **Summary:** Get Market Maker Current Epoch Estimate
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `address` | query | string | Yes |
  | `chain_type` | query | string | No |


### `GET` /v1/public/trading_rewards/current_epoch_broker_estimate
- **Summary:** Get Current Epoch Estimate by Broker
- **Limit:** 10 req/s
- **Parameters:** None


### `GET` /v1/public/trading_rewards/current_epoch_estimate
- **Summary:** Get Current Epoch Estimate
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `address` | query | string | Yes |


### `GET` /v1/public/market_making_rewards/group_rewards_history
- **Summary:** Get Wallet Group Market Making Rewards History
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `address` | query | string | Yes |
  | `symbol` | query | string | No |


### `GET` /v1/public/sv_nonce
- **Summary:** Get Strategy Vault Nonce for Account Transaction
- **Limit:** 10 req/s
- **Parameters:** None


### `GET` /v1/account_sv_transaction_history
- **Summary:** Get Account’s Strategy Vault Transaction history
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `vault_id` | query | string | No |
  | `start_t` | query | string | No |
  | `end_t` | query | string | No |
  | `page` | query | string | No |
  | `size` | query | string | No |
  | `type` | query | string | No |


### `POST` /v1/sv_operation_request
- **Summary:** Create Strategy Vault Deposit/Withdrawal Request with Account
- **Limit:** 10 req/s
- **Parameters:** None


### `GET` /v1/sv/venue_transfer_history
- **Summary:** Get Fund Inflow Allocation
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `period_number` | query | number | No |
  | `page` | query | number | No |
  | `size` | query | number | No |


### `GET` /v1/sv/venue_withdrawal_history
- **Summary:** Get Fund Outflow Allocation
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `period_number` | query | number | No |
  | `page` | query | number | No |
  | `size` | query | number | No |


### `GET` /v1/sv/protocol_revenue_share_history
- **Summary:** Get Orderly Protocol Revenue Sharing History
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `page` | query | number | No |
  | `size` | query | number | No |


### `GET` /v1/sv/liquidation_fees_share_history
- **Summary:** Get Liquidation Fees Sharing History
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `page` | query | number | No |
  | `size` | query | number | No |


### `GET` /v1/sv/internal_transfer_history
- **Summary:** Get Internal Transfer History
- **Limit:** 10 req/s
- **Parameters:**
  | Name | In | Type | Required |
  | :--- | :--- | :--- | :--- |
  | `page` | query | number | No |
  | `size` | query | number | No |


---

## Common Workflows

### Workflow 1: Initialize Trading App
1. `GET /v1/public/futures` - Load all available symbols and trading rules
2. `GET /v1/public/token` - Get supported collateral tokens
3. `GET /v1/client/info` - Get account fee rates and leverage limits (requires auth)
4. `GET /v1/client/holding` - Get current balances (requires auth)
5. `GET /v1/positions` - Get current positions (requires auth)

### Workflow 2: Place a Trade
1. `GET /v1/public/info/{symbol}` - Get order constraints (min qty, tick size)
2. `GET /v1/orderbook/{symbol}` - Get current bid/ask prices
3. `GET /v1/positions` - Check current position (if any)
4. `POST /v1/order` - Place the order
5. `GET /v1/order/{order_id}` - Verify order status

### Workflow 3: Set Up TP/SL After Opening Position
1. `GET /v1/positions` - Confirm position is open
2. `POST /v1/algo/order` with `algo_type: "POSITIONAL_TP_SL"` - Create take-profit
3. `POST /v1/algo/order` with `algo_type: "POSITIONAL_TP_SL"` - Create stop-loss
4. `GET /v1/algo/orders` - Verify algo orders are active

### Workflow 4: Close Position
1. `GET /v1/position/{symbol}` - Get current position size
2. `POST /v1/order` with opposite side and `reduce_only: true` - Close position
3. `DELETE /v1/algo/orders?symbol=...` - Cancel any TP/SL orders

### Workflow 5: Withdraw Funds
1. `GET /v1/client/holding` - Check available balance
2. `POST /v1/settle_pnl` - Settle any unrealized PnL to balance
3. `GET /v1/withdraw_nonce` - Get nonce for signature
4. `POST /v1/withdraw_request` - Submit withdrawal

### Workflow 6: Monitor Portfolio (Real-time)
For real-time updates, use WebSocket subscriptions instead of polling:
- **Private WebSocket:** `wss://ws.orderly.org/v2/ws/private/stream/{accountId}`
  - Subscribe to: `executionreport`, `position`, `balance`
- **Public WebSocket:** `wss://ws.orderly.org/ws/stream/{brokerId}`
  - Subscribe to: `orderbook`, `trade`, `tickers`

---

## API to SDK Hook Mapping

| API Endpoint | SDK Hook | Purpose |
| :--- | :--- | :--- |
| `GET /v1/positions` | `usePositionStream` | Real-time position monitoring |
| `GET /v1/orderbook/{symbol}` | `useOrderbookStream` | Real-time orderbook |
| `POST /v1/order` | `useOrderEntry` | Order placement with validation |
| `GET /v1/client/info` | `useAccountInfo` | Account details and fees |
| `GET /v1/client/holding` | `useCollateral` | Balance and collateral |
| `GET /v1/orders` | `useOrderStream` | Order list and updates |
| `GET /v1/public/futures` | `useMarkets` | Market info and symbols |
| `GET /v1/public/funding_rate/{symbol}` | `useFundingRate` | Funding rate data |
| `POST /v1/client/leverage` | `useLeverage` | Leverage management |
| `GET /v1/kline` | TradingView integration | Chart data |

---

## Best Practices

### Rate Limiting
- Implement exponential backoff when receiving 429 errors
- Cache responses where appropriate (market info, symbol info)
- Use WebSocket for real-time data instead of polling

### Order Management
- Always use `client_order_id` for idempotency and tracking
- Check `reduce_only` flag to prevent accidental position increase
- Validate order parameters against symbol constraints before submission

### Error Handling
```typescript
try {
  const response = await fetch('/v1/order', { ... });
  const data = await response.json();
  
  if (!data.success) {
    switch (data.code) {
      case -1006:
        // Insufficient balance - show deposit prompt
        break;
      case -1005:
        // Invalid order params - show validation error
        break;
      case -1003:
        // Rate limited - retry with backoff
        break;
      default:
        // Unknown error
    }
  }
} catch (error) {
  // Network error - retry
}
```

### Security
- Never expose private keys in client-side code
- Use server-side signing for production applications
- Implement IP restrictions via `/v1/client/set_orderly_key_ip_restriction`
- Rotate API keys periodically