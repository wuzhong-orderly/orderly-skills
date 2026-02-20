# Orderly SDK Hooks

A comprehensive guide to all React hooks available in the Orderly Network SDK for building decentralized exchange applications.

## Overview

The `@orderly.network/hooks` package provides a collection of React hooks for interacting with Orderly Network's trading infrastructure. These hooks handle real-time data streaming, order management, account operations, and more.

## Installation

```bash
npm install @orderly.network/hooks
# or
yarn add @orderly.network/hooks
# or
pnpm add @orderly.network/hooks
```

## Peer Dependencies

```json
{
  "react": "^18.x",
  "@orderly.network/core": "^2.x",
  "@orderly.network/net": "^2.x",
  "@orderly.network/types": "^2.x",
  "@orderly.network/utils": "^2.x"
}
```

## Provider Setup (Required)

Before using any hooks, wrap your application with the `OrderlyConfigProvider`:

```tsx
import { OrderlyConfigProvider } from "@orderly.network/hooks";

function App() {
  return (
    <OrderlyConfigProvider
      brokerId="your_broker_id"
      networkId="mainnet" // or "testnet"
    >
      <YourApp />
    </OrderlyConfigProvider>
  );
}
```

---

## Hook Categories

### 1. Core Data Fetching Hooks

These are the foundational hooks for data fetching, built on top of SWR.

#### `useQuery`

Fetches public data from Orderly REST API with automatic caching and revalidation.

```tsx
import { useQuery } from "@orderly.network/hooks";

const { data, error, isLoading, mutate } = useQuery<ResponseType>(
  "/v1/public/info",
  {
    revalidateOnFocus: false,
    dedupingInterval: 5000,
  }
);
```

**Parameters:**
| Parameter | Type | Description |
|-----------|------|-------------|
| `path` | `string` | API endpoint path |
| `options` | `SWRConfiguration` | SWR configuration options |

**Returns:** `SWRResponse<T>`

---

#### `usePrivateQuery`

Fetches authenticated/private data requiring user signature.

```tsx
import { usePrivateQuery } from "@orderly.network/hooks";

const { data, error, isLoading } = usePrivateQuery<AccountInfo>(
  "/v1/client/info"
);
```

**Parameters:**
| Parameter | Type | Description |
|-----------|------|-------------|
| `path` | `string` | Private API endpoint path |
| `options` | `SWRConfiguration` | SWR configuration options |

---

#### `useMutation`

Performs POST/PUT/DELETE operations to the API.

```tsx
import { useMutation } from "@orderly.network/hooks";

const [doMutate, { data, error, isMutating, reset }] = useMutation(
  "/v1/order",
  "POST"
);

// Usage
await doMutate({ symbol: "PERP_ETH_USDC", side: "BUY", ... });
```

**Parameters:**
| Parameter | Type | Description |
|-----------|------|-------------|
| `path` | `string` | API endpoint path |
| `method` | `'POST' \| 'PUT' \| 'DELETE'` | HTTP method (default: POST) |

---

#### `useLazyQuery`

Fetches data on-demand rather than automatically.

```tsx
import { useLazyQuery } from "@orderly.network/hooks";

const [fetch, { data, loading, error }] = useLazyQuery<DataType>("/v1/endpoint");

// Trigger manually
const result = await fetch();
```

---

### 2. Account & Authentication Hooks

#### `useAccount`

Core hook for account state and authentication management.

```tsx
import { useAccount } from "@orderly.network/hooks";

const {
  account,           // Account instance
  state,             // AccountState enum
  createOrderlyKey,  // Function to create trading key
  createAccount,     // Function to create Orderly account
  disconnect,        // Disconnect wallet
  setAddress,        // Set wallet address
} = useAccount();
```

**Account States:**
- `NotConnected` - Wallet not connected
- `Connected` - Wallet connected
- `NotSignedIn` - Needs Orderly account creation
- `SignedIn` - Fully authenticated

---

#### `useAccountInfo`

Fetches detailed account information.

```tsx
import { useAccountInfo } from "@orderly.network/hooks";

const { data: accountInfo, isLoading } = useAccountInfo();
// Returns: { account_id, email, tier, ... }
```

---

#### `useAccountInstance`

Access the low-level Account instance for advanced operations.

```tsx
import { useAccountInstance } from "@orderly.network/hooks";

const account = useAccountInstance();
```

---

### 3. Market Data Hooks

#### `useMarkets`

Fetches all available trading markets/symbols.

```tsx
import { useMarkets, MarketsType } from "@orderly.network/hooks";

const [markets, { isLoading }] = useMarkets(MarketsType.ALL);
```

**Parameters:**
| Parameter | Type | Description |
|-----------|------|-------------|
| `type` | `MarketsType` | `ALL`, `FAVORITES`, `NEW_LISTING`, `RECENT` |

---

#### `useMarketsStream`

Real-time streaming of all market ticker data.

```tsx
import { useMarketsStream } from "@orderly.network/hooks";

const { data: marketsData } = useMarketsStream();
// Returns real-time prices, 24h changes, volumes for all symbols
```

---

#### `useMarket`

Get market info for a specific symbol.

```tsx
import { useMarket } from "@orderly.network/hooks";

const data = useMarket("PERP_ETH_USDC");
// Returns: { symbol, base, quote, price, volume24h, ... }
```

---

#### `useSymbolsInfo`

Fetches detailed symbol configuration and trading rules.

```tsx
import { useSymbolsInfo } from "@orderly.network/hooks";

const symbolsInfo = useSymbolsInfo();
const ethInfo = symbolsInfo["PERP_ETH_USDC"]();
// Returns: { base_tick, quote_tick, min_notional, ... }
```

---

#### `useMarkPrice`

Real-time mark price for a symbol.

```tsx
import { useMarkPrice } from "@orderly.network/hooks";

const markPrice = useMarkPrice("PERP_ETH_USDC");
// Returns: number
```

---

#### `useIndexPrice`

Real-time index price for a symbol.

```tsx
import { useIndexPrice } from "@orderly.network/hooks";

const indexPrice = useIndexPrice("PERP_ETH_USDC");
// Returns: number
```

---

#### `useTickerStream`

Real-time ticker data for a specific symbol.

```tsx
import { useTickerStream } from "@orderly.network/hooks";

const ticker = useTickerStream("PERP_ETH_USDC");
// Returns: { symbol, open, high, low, close, volume, change, ... }
```

---

### 4. Order Book & Trades Hooks

#### `useOrderbookStream`

Real-time order book data with depth aggregation.

```tsx
import { useOrderbookStream } from "@orderly.network/hooks";

const [
  data,    // { asks: OrderBookItem[], bids: OrderBookItem[] }
  { 
    onDepthChange,    // Function to change depth
    depth,            // Current depth setting
    allDepths,        // Available depth options
    isLoading,
    onItemClick,      // Handler for clicking a price level
  }
] = useOrderbookStream("PERP_ETH_USDC", undefined, { 
  level: 10  // Number of price levels
});
```

**OrderBookItem Format:** `[price, quantity, cumulativeQty, cumulativeAmount]`

---

#### `useMarketTradeStream`

Real-time market trades stream.

```tsx
import { useMarketTradeStream } from "@orderly.network/hooks";

const { data: trades } = useMarketTradeStream("PERP_ETH_USDC");
// Returns: Array<{ price, size, side, timestamp, ... }>
```

---

### 5. Position Management Hooks

#### `usePositionStream`

Real-time position data stream.

```tsx
import { usePositionStream } from "@orderly.network/hooks";

const [
  positions,  // Array of position objects
  positionInfo,  // Aggregate position info
  { 
    loading, 
    refresh, 
    showAllSymbol,  // Toggle to show all symbols
    setShowAllSymbol 
  }
] = usePositionStream("PERP_ETH_USDC");
```

**Position Object:**
```typescript
{
  symbol: string;
  position_qty: number;
  average_open_price: number;
  mark_price: number;
  unrealized_pnl: number;
  unrealized_pnl_ROI: number;
  mmr: number;
  imr: number;
  // ...
}
```

---

#### `useMaxQty`

Calculate maximum order quantity based on available margin.

```tsx
import { useMaxQty } from "@orderly.network/hooks";

const maxQty = useMaxQty(
  "PERP_ETH_USDC",
  "BUY",      // side
  reduceOnly  // boolean
);
```

---

### 6. Order Management Hooks

#### `useOrderStream`

Real-time order updates stream.

```tsx
import { useOrderStream } from "@orderly.network/hooks";

const [orders, { refresh, isLoading }] = useOrderStream({
  symbol: "PERP_ETH_USDC",
  status: "OPEN",  // "OPEN" | "FILLED" | "CANCELLED" | "PENDING"
});
```

---

#### `useOrderEntry`

Comprehensive hook for order entry with validation.

```tsx
import { useOrderEntry } from "@orderly.network/hooks";

const {
  formattedOrder,  // Validated order object
  submit,          // Submit order function
  maxQty,          // Maximum quantity
  freeCollateral,  // Available collateral
  errors,          // Validation errors
  setValues,       // Update form values
  // ...
} = useOrderEntry({
  symbol: "PERP_ETH_USDC",
  side: "BUY",
  order_type: "LIMIT",
});
```

---

#### `useTPSLOrder`

Manage Take-Profit/Stop-Loss orders.

```tsx
import { useTPSLOrder } from "@orderly.network/hooks";

const [
  tpslOrder, 
  { update, remove }
] = useTPSLOrder("PERP_ETH_USDC", positionQty);
```

---

### 7. Collateral & Margin Hooks

#### `useCollateral`

Get collateral information and account equity.

```tsx
import { useCollateral } from "@orderly.network/hooks";

const {
  totalCollateral,    // Total collateral value
  freeCollateral,     // Available for new positions
  totalValue,         // Total account value
  availableBalance,   // Available balance
  unsettledPnL,       // Unrealized P&L
  positions,          // Position value
} = useCollateral({ dp: 2 });
```

**Options:**
| Option | Type | Description |
|--------|------|-------------|
| `dp` | `number` | Decimal precision (default: 2) |

---

#### `useMarginRatio`

Calculate current margin ratio.

```tsx
import { useMarginRatio } from "@orderly.network/hooks";

const {
  marginRatio,      // Current margin ratio
  mmr,              // Maintenance margin ratio
  currentLeverage,  // Current effective leverage
} = useMarginRatio();
```

---

#### `useLeverage`

Manage account leverage settings.

```tsx
import { useLeverage } from "@orderly.network/hooks";

const {
  curLeverage,      // Current leverage
  maxLeverage,      // Maximum allowed leverage
  leverageLevers,   // Available leverage options [1, 5, 10, 15, 20]
  update,           // Update leverage function
  isLoading,
} = useLeverage();

// Update leverage
await update({ leverage: 10 });
```

---

### 8. Funding Rate Hooks

#### `useFundingRate`

Get funding rate for a specific symbol.

```tsx
import { useFundingRate } from "@orderly.network/hooks";

const fundingRate = useFundingRate("PERP_ETH_USDC");
// Returns: { rate, next_funding_time, ... }
```

---

#### `useFundingRates`

Get funding rates for all symbols.

```tsx
import { useFundingRates } from "@orderly.network/hooks";

const fundingRates = useFundingRates();
// Returns: { "PERP_ETH_USDC": { rate, ... }, ... }
```

---

### 9. Asset Management Hooks

#### `useDeposit`

Handle deposits to Orderly from wallet.

```tsx
import { useDeposit } from "@orderly.network/hooks";

const {
  deposit,        // Execute deposit function
  balance,        // Current wallet balance
  allowance,      // Current token allowance
  approve,        // Approve token spending
  isNativeToken,  // Is native token (ETH/SOL)
  quantity,       // Deposit quantity state
  setQuantity,    // Set deposit quantity
  depositFee,     // Estimated deposit fee
  dst,            // Destination chain info
} = useDeposit({
  address: tokenAddress,
  decimals: 6,
  srcChainId: 1,
  srcToken: "USDC",
});
```

**DepositOptions:**
| Option | Type | Description |
|--------|------|-------------|
| `address` | `string` | Token contract address |
| `decimals` | `number` | Token decimals |
| `srcChainId` | `number` | Source chain ID |
| `srcToken` | `string` | Source token symbol |
| `dstToken` | `string` | Destination token symbol |
| `swapEnable` | `boolean` | Enable cross-chain swap |

---

#### `useWithdraw`

Handle withdrawals from Orderly to wallet.

```tsx
import { useWithdraw } from "@orderly.network/hooks";

const {
  withdraw,          // Execute withdraw function
  maxAmount,         // Maximum withdrawable amount
  availableWithdraw, // Available to withdraw
  unsettledPnL,      // Unsettled P&L affecting withdrawal
  dst,               // Destination chain info
} = useWithdraw({
  srcChainId: 1,
  token: "USDC",
  decimals: 6,
});
```

---

#### `useHoldingStream`

Real-time holdings/balances stream.

```tsx
import { useHoldingStream } from "@orderly.network/hooks";

const { data: holdings, refresh } = useHoldingStream();
// Returns: [{ token: "USDC", holding: 1000, ... }, ...]
```

---

#### `useTransfer`

Transfer funds between Orderly and wallet.

```tsx
import { useTransfer } from "@orderly.network/hooks";

const { transfer, isLoading } = useTransfer();
```

---

#### `useConvert`

Convert between assets.

```tsx
import { useConvert } from "@orderly.network/hooks";

const { convert, conversionRate, isLoading } = useConvert();
```

---

### 10. Chain & Network Hooks

#### `useChains`

Get list of supported chains.

```tsx
import { useChains } from "@orderly.network/hooks";

const [chains, { isLoading }] = useChains();
// Returns: [{ chain_id, name, native_token, ... }, ...]
```

---

#### `useChain`

Get current connected chain information.

```tsx
import { useChain } from "@orderly.network/hooks";

const { chain, chains, setChain } = useChain();
```

---

#### `useNetworkInfo`

Get network status and information.

```tsx
import { useNetworkInfo } from "@orderly.network/hooks";

const networkInfo = useNetworkInfo();
```

---

### 11. WebSocket Hooks

#### `useWS`

Low-level WebSocket connection hook.

```tsx
import { useWS } from "@orderly.network/hooks";

const ws = useWS();

// Subscribe to a topic
ws.subscribe("orderbook:PERP_ETH_USDC", (data) => {
  console.log(data);
});
```

---

#### `useWsStatus`

Monitor WebSocket connection status.

```tsx
import { useWsStatus } from "@orderly.network/hooks";

const { connected, reconnecting, error } = useWsStatus();
```

---

### 12. Historical Data Hooks

#### `useAssetsHistory`

Fetch asset transaction history.

```tsx
import { useAssetsHistory } from "@orderly.network/hooks";

const { data, isLoading, loadMore, hasMore } = useAssetsHistory({
  startDate: "2024-01-01",
  endDate: "2024-12-31",
});
```

---

#### `useFundingFeeHistory`

Fetch funding fee history.

```tsx
import { useFundingFeeHistory } from "@orderly.network/hooks";

const { data: fundingHistory } = useFundingFeeHistory({
  symbol: "PERP_ETH_USDC",
  page: 1,
  pageSize: 50,
});
```

---

#### `useTradeHistory`

Fetch trade execution history.

```tsx
import { useTradeHistory } from "@orderly.network/hooks";

const { data: trades, isLoading } = useTradeHistory({
  symbol: "PERP_ETH_USDC",
  limit: 100,
});
```

---

#### `useLiquidationHistory`

Fetch liquidation history.

```tsx
import { useLiquidationHistory } from "@orderly.network/hooks";

const { data: liquidations } = useLiquidationHistory();
```

---

### 13. Fee Hooks

#### `useFeeState`

Get current fee tier and rates.

```tsx
import { useFeeState } from "@orderly.network/hooks";

const { fee, tier, makerRate, takerRate } = useFeeState();
```

---

### 14. Utility Hooks

#### `useLocalStorage`

Persistent local storage with React state.

```tsx
import { useLocalStorage } from "@orderly.network/hooks";

const [value, setValue] = useLocalStorage("key", defaultValue);
```

---

#### `useEventEmitter`

Subscribe to Orderly event bus.

```tsx
import { useEventEmitter } from "@orderly.network/hooks";

const ee = useEventEmitter();

ee.on("order:created", (order) => {
  console.log("New order:", order);
});
```

---

#### `useConfig`

Access Orderly configuration.

```tsx
import { useConfig } from "@orderly.network/hooks";

const config = useConfig();
// Returns: { brokerId, networkId, ... }
```

---

#### `usePreLoadData`

Preload essential data on app initialization.

```tsx
import { usePreLoadData } from "@orderly.network/hooks";

const { isLoaded, error } = usePreLoadData();
```

---

#### `useBoolean`

Simple boolean state management.

```tsx
import { useBoolean } from "@orderly.network/hooks";

const [isOpen, { toggle, setTrue, setFalse }] = useBoolean(false);
```

---

### 15. Referral & Rewards Hooks

#### `useReferral`

Manage referral program.

```tsx
import { useReferralInfo } from "@orderly.network/hooks";

const { referralCode, referralLink, referralStats } = useReferralInfo();
```

---

#### `useTradingRewards`

Access trading rewards information.

```tsx
import { useTradingRewards } from "@orderly.network/hooks";

const { rewards, claimableAmount, claim } = useTradingRewards();
```

---

### 16. API Key Management Hooks

#### `useApiKeys`

Manage API keys for programmatic access.

```tsx
import { useApiKeys } from "@orderly.network/hooks";

const { keys, createKey, deleteKey, isLoading } = useApiKeys();
```

---

### 17. Sub-Account Hooks

#### `useSubAccount`

Manage sub-accounts.

```tsx
import { useSubAccount } from "@orderly.network/hooks";

const { subAccounts, createSubAccount, switchAccount } = useSubAccount();
```

---

## Import Summary by Package

### Main Package: `@orderly.network/hooks`

```tsx
import {
  // Core
  useQuery,
  useLazyQuery,
  useMutation,
  usePrivateQuery,
  useInfiniteQuery,
  
  // Account
  useAccount,
  useAccountInstance,
  useAccountInfo,
  
  // Market Data
  useMarkets,
  useMarketsStream,
  useMarket,
  useSymbolsInfo,
  useMarkPrice,
  useIndexPrice,
  useTickerStream,
  
  // Order Book
  useOrderbookStream,
  useMarketTradeStream,
  
  // Positions
  usePositionStream,
  useMaxQty,
  
  // Orders
  useOrderStream,
  useOrderEntry,
  useTPSLOrder,
  
  // Collateral
  useCollateral,
  useMarginRatio,
  useLeverage,
  useMaxLeverage,
  
  // Funding
  useFundingRate,
  useFundingRates,
  
  // Assets
  useDeposit,
  useWithdraw,
  useHoldingStream,
  useTransfer,
  useConvert,
  
  // Chains
  useChains,
  useChain,
  useNetworkInfo,
  
  // WebSocket
  useWS,
  useWsStatus,
  
  // History
  useAssetsHistory,
  useFundingFeeHistory,
  useTradeHistory,
  useLiquidationHistory,
  
  // Fees
  useFeeState,
  
  // Utilities
  useLocalStorage,
  useEventEmitter,
  useConfig,
  usePreLoadData,
  useBoolean,
  
  // Provider
  OrderlyConfigProvider,
  WalletConnectorContext,
} from "@orderly.network/hooks";
```

---

## Best Practices

### 1. Always Check Account State
```tsx
const { state } = useAccount();
if (state !== AccountState.SignedIn) {
  return <ConnectWallet />;
}
```

### 2. Handle Loading and Error States
```tsx
const { data, error, isLoading } = useMarkets();
if (isLoading) return <Loading />;
if (error) return <Error message={error.message} />;
```

### 3. Use Appropriate Decimal Precision
```tsx
const { totalCollateral } = useCollateral({ dp: 2 });
```

### 4. Unsubscribe from Streams on Cleanup
```tsx
useEffect(() => {
  const unsubscribe = ws.subscribe("topic", handler);
  return () => unsubscribe();
}, []);
```

### 5. Leverage SWR Caching
```tsx
const { data, mutate } = useQuery(endpoint, {
  revalidateOnFocus: false,
  dedupingInterval: 5000,
});
```

### 6. Validate Before Submitting Orders
```tsx
const { errors, formattedOrder, submit } = useOrderEntry({...});
if (Object.keys(errors).length === 0) {
  await submit();
}
```

### 7. Use Type Imports for TypeScript
```tsx
import type { OrderType, OrderSide, WSMessage } from "@orderly.network/types";
```

### 8. Handle Real-Time Data Efficiently
```tsx
// useOrderbookStream already handles depth aggregation
const [data] = useOrderbookStream(symbol, undefined, { level: 10 });
```

### 9. Preload Data on App Start
```tsx
function App() {
  const { isLoaded } = usePreLoadData();
  if (!isLoaded) return <SplashScreen />;
  return <MainApp />;
}
```

### 10. Use Event Emitter for Cross-Component Communication
```tsx
const ee = useEventEmitter();
ee.emit("custom:event", { data });
```

---

## TypeScript Support

All hooks are fully typed. Import types from `@orderly.network/types`:

```tsx
import type {
  Order,
  Position,
  OrderType,
  OrderSide,
  OrderStatus,
  AccountState,
  MarketInfo,
  SymbolInfo,
  ChainInfo,
} from "@orderly.network/types";
```

---

## Common Patterns

### Complete Trading Flow

```tsx
import { 
  useAccount, 
  useCollateral, 
  useOrderEntry 
} from "@orderly.network/hooks";

function TradingComponent() {
  const { state } = useAccount();
  const { freeCollateral } = useCollateral();
  const { submit, errors, setValues, maxQty } = useOrderEntry({
    symbol: "PERP_ETH_USDC",
    side: "BUY",
    order_type: "LIMIT",
  });

  const handleTrade = async () => {
    if (state !== AccountState.SignedIn) return;
    if (Object.keys(errors).length > 0) return;
    
    await submit();
  };

  return (
    <div>
      <p>Available: {freeCollateral}</p>
      <p>Max Qty: {maxQty}</p>
      <button onClick={handleTrade}>Trade</button>
    </div>
  );
}
```

### Real-Time Market Dashboard

```tsx
import { 
  useMarketsStream, 
  useOrderbookStream,
  useMarketTradeStream 
} from "@orderly.network/hooks";

function MarketDashboard({ symbol }: { symbol: string }) {
  const { data: markets } = useMarketsStream();
  const [orderbook] = useOrderbookStream(symbol);
  const { data: trades } = useMarketTradeStream(symbol);

  return (
    <div>
      <Ticker data={markets[symbol]} />
      <OrderBook data={orderbook} />
      <TradeHistory trades={trades} />
    </div>
  );
}
```

---

## Related Packages

| Package | Description |
|---------|-------------|
| `@orderly.network/core` | Core SDK functionality |
| `@orderly.network/types` | TypeScript type definitions |
| `@orderly.network/utils` | Utility functions |
| `@orderly.network/net` | Network/HTTP layer |
| `@orderly.network/perp` | Perpetual trading calculations |
| `@orderly.network/trading` | Pre-built trading components |
| `@orderly.network/portfolio` | Pre-built portfolio components |
| `@orderly.network/markets` | Pre-built markets components |

---

## Error Handling

All hooks can throw `SDKError` from `@orderly.network/types`:

```tsx
import { SDKError } from "@orderly.network/types";

try {
  await deposit();
} catch (error) {
  if (error instanceof SDKError) {
    console.error(`SDK Error [${error.code}]: ${error.message}`);
  }
}
```

---

## Version Compatibility

- **Hooks Package**: v2.8.x
- **React**: ^18.0.0
- **Node.js**: >= 16.x
- **TypeScript**: >= 5.0.0

---

## Additional Resources

- [Orderly Network Documentation](https://orderly.network/docs)
- [SDK GitHub Repository](https://github.com/OrderlyNetwork/js-sdk)
- [API Reference](https://orderly.network/docs/api)
