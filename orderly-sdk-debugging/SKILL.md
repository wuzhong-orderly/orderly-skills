---
name: orderly-sdk-debugging
description: Debug and troubleshoot common issues with the Orderly SDK including errors, WebSocket issues, authentication problems, and trading failures.
argument-hint: <error-type>
---

# Orderly SDK Debugging & Error Handling

A comprehensive guide to debugging common issues, handling errors, and troubleshooting problems with the Orderly SDK.

---

## 1. Build & Setup Errors

### Buffer is not defined

```
Uncaught ReferenceError: Buffer is not defined
```

**Cause**: Wallet libraries use Node.js built-ins (Buffer, crypto) that don't exist in browsers.

**Solution**: Add `vite-plugin-node-polyfills`:

```bash
npm install -D vite-plugin-node-polyfills
```

```ts
// vite.config.ts
import { nodePolyfills } from 'vite-plugin-node-polyfills';

export default defineConfig({
  plugins: [
    react(),
    nodePolyfills({
      include: ['buffer', 'crypto', 'stream', 'util'],
      globals: {
        Buffer: true,
        global: true,
        process: true,
      },
    }),
  ],
});
```

### CSS Import Not Found

```
ENOENT: no such file or directory, open '@orderly.network/trading/dist/styles.css'
```

**Cause**: Only `@orderly.network/ui` has a CSS file. Other packages (`trading`, `portfolio`, `markets`) do NOT have separate CSS files.

**Solution**: Only import from `@orderly.network/ui`:

```css
/* ✅ Correct - only ui package has CSS */
@import '@orderly.network/ui/dist/styles.css';

/* ❌ Wrong - these don't exist */
/* @import '@orderly.network/trading/dist/styles.css'; */
/* @import '@orderly.network/portfolio/dist/styles.css'; */
```

### process is not defined

```
Uncaught ReferenceError: process is not defined
```

**Cause**: Same as Buffer issue—Node.js globals missing.

**Solution**: Same fix—add `vite-plugin-node-polyfills` (see above).

---

## 2. Error Types

### SDK Errors

```tsx
import { SDKError, ApiError } from "@orderly.network/types";

try {
  await submitOrder(order);
} catch (error) {
  if (error instanceof SDKError) {
    // SDK-level error (configuration, state, validation)
    console.error(`SDK Error: ${error.message}`);
  } else if (error instanceof ApiError) {
    // API-level error (server response)
    console.error(`API Error [${error.code}]: ${error.message}`);
  } else {
    // Unknown error
    console.error("Unexpected error:", error);
  }
}
```

---

## 3. Common Error Codes

### API Error Codes

| Code | Message | Cause | Solution |
|------|---------|-------|----------|
| `-1000` | Unknown error | Server error | Retry request |
| `-1001` | Disconnected | Network issue | Check connection, retry |
| `-1002` | Unauthorized | Invalid/expired key | Re-authenticate |
| `-1003` | Too many requests | Rate limit | Implement backoff |
| `-1004` | Not found | Invalid endpoint/symbol | Check symbol format |
| `-1102` | Invalid parameter | Wrong order params | Validate inputs |
| `-1103` | Invalid signature | Signing failed | Check key configuration |

### Order Error Codes

| Code | Message | Cause | Solution |
|------|---------|-------|----------|
| `-2001` | Insufficient balance | Not enough USDC | Deposit more funds |
| `-2002` | Order would trigger liquidation | Risk too high | Reduce position size |
| `-2003` | Symbol not trading | Market closed | Wait or use different symbol |
| `-2004` | Price out of range | Price too far from mark | Adjust limit price |
| `-2005` | Order quantity too small | Below minimum | Increase quantity |
| `-2006` | Order quantity too large | Exceeds limit | Reduce quantity |
| `-2007` | Reduce only order invalid | No position to reduce | Check position exists |
| `-2008` | Duplicate client order id | Order ID reused | Generate unique ID |
| `-2010` | Order would exceed position limit | Max position reached | Close existing positions |

### Withdrawal Error Codes

| Code | Message | Cause | Solution |
|------|---------|-------|----------|
| `-3001` | Insufficient balance | Not enough available | Check unsettled PnL |
| `-3002` | Withdrawal amount too small | Below minimum | Increase amount |
| `-3003` | Pending withdrawal exists | Previous in progress | Wait for completion |
| `-3004` | Withdrawal disabled | Maintenance | Try again later |

---

## 3. WebSocket Connection

### Monitor Connection Status

```tsx
import { useWsStatus, WsNetworkStatus } from "@orderly.network/hooks";

function ConnectionIndicator() {
  const wsStatus = useWsStatus();
  
  return (
    <div className="connection-status">
      {wsStatus === WsNetworkStatus.Connected && (
        <span className="text-green-500">● Connected</span>
      )}
      {wsStatus === WsNetworkStatus.Unstable && (
        <span className="text-yellow-500">● Reconnecting...</span>
      )}
      {wsStatus === WsNetworkStatus.Disconnected && (
        <span className="text-red-500">● Disconnected</span>
      )}
    </div>
  );
}
```

### WebSocket Status Values

| Status | Description |
|--------|-------------|
| `Connected` | WebSocket is connected and working |
| `Unstable` | Connection dropped, attempting reconnect (≥3 attempts) |
| `Disconnected` | Connection lost, not reconnecting |

### Debug WebSocket Events

```tsx
import { useWS } from "@orderly.network/hooks";

function WebSocketDebugger() {
  const ws = useWS();
  
  useEffect(() => {
    const handleStatusChange = (status: any) => {
      console.log("WS Status:", status.type, status.isPrivate ? "(private)" : "(public)");
    };
    
    ws.on("status:change", handleStatusChange);
    return () => ws.off("status:change", handleStatusChange);
  }, [ws]);
  
  return null;
}
```

---

## 4. Account State Issues

### Check Account State

```tsx
import { useAccount, AccountStatusEnum } from "@orderly.network/hooks";

function AccountDebugger() {
  const { state, account } = useAccount();
  
  useEffect(() => {
    console.log("Account State:", {
      status: state.status,
      address: state.address,
      userId: state.userId,
      accountId: state.accountId,
      hasOrderlyKey: !!account?.keyStore?.getOrderlyKey(),
    });
  }, [state, account]);
  
  // Common issues:
  switch (state.status) {
    case AccountStatusEnum.NotConnected:
      return <p>Wallet not connected</p>;
    case AccountStatusEnum.Connected:
      return <p>Wallet connected, not signed in</p>;
    case AccountStatusEnum.NotSignedIn:
      return <p>Need to sign message to create Orderly key</p>;
    case AccountStatusEnum.SignedIn:
      return <p>Fully authenticated</p>;
    case AccountStatusEnum.DisabledTrading:
      return <p>Trading disabled for this account</p>;
  }
}
```

### Common Account Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Stuck on "Connected" | User didn't sign | Prompt for signature |
| Key expired | 365-day expiry | Re-authenticate |
| Wrong network | Testnet vs mainnet mismatch | Check `networkId` in provider |
| No user ID | Account not registered | Complete signup flow |

---

## 5. Order Submission Errors

### Validate Before Submit

```tsx
import { useOrderEntry } from "@orderly.network/hooks";

function OrderDebugger() {
  const { formattedOrder, metaState, helper } = useOrderEntry("PERP_ETH_USDC");
  
  // Check for validation errors
  if (metaState.errors) {
    console.log("Order Errors:", metaState.errors);
    // Example: { quantity: "Below minimum", price: "Required for limit order" }
  }
  
  // Check order readiness
  console.log("Order Ready:", {
    canSubmit: !metaState.errors && formattedOrder,
    maxQty: helper.maxQty,
    estLiqPrice: helper.estLiqPrice,
    estLeverage: helper.estLeverage,
  });
}
```

### Debug Order Rejection

```tsx
async function submitOrderWithDebug(order: OrderParams) {
  try {
    const result = await submit();
    console.log("Order submitted:", result);
    return result;
  } catch (error: any) {
    console.error("Order failed:", {
      code: error.code,
      message: error.message,
      order: order,
    });
    
    // Common fixes
    if (error.code === -2001) {
      console.log("Fix: Deposit more USDC or reduce order size");
    } else if (error.code === -2002) {
      console.log("Fix: Reduce leverage or position size");
    } else if (error.code === -2004) {
      console.log("Fix: Use market order or adjust limit price");
    }
    
    throw error;
  }
}
```

---

## 6. Provider Configuration Errors

### Missing Provider

```
Error: useAccount must be used within OrderlyConfigProvider
```

**Solution**: Ensure proper provider hierarchy:

```tsx
<WalletConnectorProvider>
  <OrderlyAppProvider brokerId="..." networkId="mainnet">
    {/* Your app */}
  </OrderlyAppProvider>
</WalletConnectorProvider>
```

### Invalid Broker ID

```
Error: Invalid broker ID
```

**Solution**: 
- Use `"demo"` for testing
- Get production broker ID from Orderly Network

### Network Mismatch

```
Error: Chain not supported
```

**Solution**: Ensure chain is supported for the network:

```tsx
// Mainnet chains
const mainnetChains = [1, 42161, 10, 8453, 56]; // Ethereum, Arbitrum, Optimism, Base, BSC

// Testnet chains
const testnetChains = [421614, 84532, 11155420]; // Arbitrum Sepolia, Base Sepolia, OP Sepolia
```

---

## 7. Deposit/Withdrawal Errors

### Debug Deposit

```tsx
import { useDeposit } from "@orderly.network/hooks";

function DepositDebugger() {
  const { deposit, balance, allowance, approve } = useDeposit();
  
  const handleDeposit = async (amount: string) => {
    console.log("Deposit Debug:", {
      amount,
      walletBalance: balance,
      currentAllowance: allowance,
      needsApproval: Number(amount) > Number(allowance),
    });
    
    try {
      if (Number(amount) > Number(allowance)) {
        console.log("Approving USDC...");
        await approve(amount);
      }
      
      console.log("Depositing...");
      const result = await deposit(amount);
      console.log("Deposit success:", result);
    } catch (error: any) {
      console.error("Deposit failed:", error);
      
      if (error.message.includes("user rejected")) {
        console.log("User rejected transaction");
      } else if (error.message.includes("insufficient")) {
        console.log("Insufficient balance or gas");
      }
    }
  };
}
```

### Debug Withdrawal

```tsx
import { useWithdraw } from "@orderly.network/hooks";

function WithdrawDebugger() {
  const { withdraw, availableWithdraw, unsettledPnL } = useWithdraw();
  
  console.log("Withdraw Debug:", {
    available: availableWithdraw,
    unsettledPnL: unsettledPnL,
    canWithdraw: Number(availableWithdraw) > 0,
  });
  
  // If available is 0 but you have funds:
  // 1. Check for open positions (margin locked)
  // 2. Check for pending orders (margin reserved)
  // 3. Check unsettledPnL (need to settle first)
}
```

---

## 8. Debugging Hooks

### Enable Debug Mode

```tsx
// Log all hook state changes
function useDebugHook<T>(hookName: string, value: T): T {
  useEffect(() => {
    console.log(`[${hookName}]`, value);
  }, [value, hookName]);
  return value;
}

// Usage
const positions = useDebugHook("positions", usePositionStream().positions);
```

### Debug SWR Cache

```tsx
import { useSWRConfig } from "swr";

function SWRDebugger() {
  const { cache } = useSWRConfig();
  
  // Log cache keys
  console.log("SWR Cache Keys:", Array.from((cache as Map<string, any>).keys()));
  
  return null;
}
```

---

## 9. Network Issues

### CORS Errors

```
Access to fetch at 'https://api.orderly.network/...' has been blocked by CORS
```

**Solutions**:
1. This usually indicates wrong API usage
2. SDK handles CORS automatically
3. Check you're not calling API directly without SDK

### Rate Limiting

```tsx
// Implement exponential backoff
async function withRetry<T>(
  fn: () => Promise<T>,
  maxRetries = 3,
  baseDelay = 1000
): Promise<T> {
  for (let attempt = 0; attempt < maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error: any) {
      if (error.code === -1003 && attempt < maxRetries - 1) {
        const delay = baseDelay * Math.pow(2, attempt);
        console.log(`Rate limited, retrying in ${delay}ms...`);
        await new Promise(resolve => setTimeout(resolve, delay));
      } else {
        throw error;
      }
    }
  }
  throw new Error("Max retries exceeded");
}
```

---

## 10. Error Boundary

Wrap your app with an error boundary:

```tsx
import { ErrorBoundary } from "@orderly.network/react-app";

// Or create custom:
class OrderlyErrorBoundary extends React.Component<
  { children: React.ReactNode },
  { hasError: boolean; error?: Error }
> {
  state = { hasError: false, error: undefined };

  static getDerivedStateFromError(error: Error) {
    return { hasError: true, error };
  }

  componentDidCatch(error: Error, errorInfo: React.ErrorInfo) {
    console.error("Orderly Error:", error, errorInfo);
    // Send to error tracking (Sentry, etc.)
  }

  render() {
    if (this.state.hasError) {
      return (
        <div className="error-fallback">
          <h2>Something went wrong</h2>
          <pre>{this.state.error?.message}</pre>
          <button onClick={() => window.location.reload()}>
            Reload Page
          </button>
        </div>
      );
    }
    return this.props.children;
  }
}
```

---

## 11. Logging Best Practices

### Structured Logging

```tsx
const logger = {
  order: (action: string, data: any) => {
    console.log(`[ORDER] ${action}:`, data);
  },
  account: (action: string, data: any) => {
    console.log(`[ACCOUNT] ${action}:`, data);
  },
  ws: (action: string, data: any) => {
    console.log(`[WEBSOCKET] ${action}:`, data);
  },
  error: (context: string, error: any) => {
    console.error(`[ERROR] ${context}:`, {
      message: error.message,
      code: error.code,
      stack: error.stack,
    });
  },
};

// Usage
logger.order("submit", { symbol, quantity, price });
logger.error("order-submit", error);
```

### Development vs Production

```tsx
const isDev = import.meta.env.DEV;

const log = isDev
  ? console.log.bind(console)
  : () => {}; // No-op in production

const logError = (context: string, error: any) => {
  if (isDev) {
    console.error(`[${context}]`, error);
  } else {
    // Send to error tracking service
    sendToErrorTracking(context, error);
  }
};
```

---

## 12. Debugging Checklist

### Order Not Submitting

- [ ] Account status is `SignedIn`?
- [ ] Symbol format correct? (e.g., `PERP_ETH_USDC`)
- [ ] Sufficient balance?
- [ ] Order quantity above minimum?
- [ ] Limit price within range?
- [ ] No validation errors in `metaState.errors`?

### Wallet Not Connecting

- [ ] WalletConnectorProvider configured?
- [ ] Correct wallet adapters installed?
- [ ] Chain supported for network?
- [ ] User approved connection in wallet?

### Data Not Loading

- [ ] WebSocket connected?
- [ ] Correct networkId (mainnet vs testnet)?
- [ ] User authenticated for private data?
- [ ] Check browser console for errors?

### Deposit/Withdraw Failing

- [ ] Correct chain selected?
- [ ] USDC approved for deposit?
- [ ] Sufficient gas for transaction?
- [ ] No pending withdrawals?
- [ ] Available balance covers withdrawal?

---

## 13. Useful Debug Components

### Full State Debugger

```tsx
function OrderlyDebugPanel() {
  const { state } = useAccount();
  const wsStatus = useWsStatus();
  const { data: accountInfo } = useAccountInfo();
  
  if (!import.meta.env.DEV) return null;
  
  return (
    <div className="fixed bottom-4 right-4 bg-black/80 text-white p-4 rounded-lg text-xs max-w-sm">
      <h3 className="font-bold mb-2">Debug Panel</h3>
      <div>Account: {state.status}</div>
      <div>WS: {wsStatus}</div>
      <div>Address: {state.address?.slice(0, 10)}...</div>
      <div>Balance: {accountInfo?.freeCollateral?.toFixed(2)} USDC</div>
    </div>
  );
}
```

Add to your app in development:

```tsx
function App() {
  return (
    <OrderlyProvider>
      <Routes />
      {import.meta.env.DEV && <OrderlyDebugPanel />}
    </OrderlyProvider>
  );
}
```
