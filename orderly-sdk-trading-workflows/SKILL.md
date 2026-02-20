# Orderly SDK Trading Workflows

A comprehensive guide to implementing complete trading workflows in Orderly Network DEX applications, from wallet connection through order execution and position management.

## Overview

This skill covers end-to-end trading workflows:

1. **Connect** → Wallet connection and authentication
2. **Deposit** → Fund your Orderly account
3. **Trade** → Place and manage orders
4. **Monitor** → Track positions and PnL
5. **Withdraw** → Move funds back to wallet

---

## Complete Trading Flow

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Connect    │────▶│   Deposit    │────▶│    Trade     │
│   Wallet     │     │    Funds     │     │   (Orders)   │
└──────────────┘     └──────────────┘     └──────────────┘
                                                 │
                                                 ▼
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Withdraw   │◀────│    Close     │◀────│   Monitor    │
│    Funds     │     │  Positions   │     │  Positions   │
└──────────────┘     └──────────────┘     └──────────────┘
```

---

## 1. Connect & Authenticate

### Check Account State

```tsx
import { useAccount, AccountStatusEnum } from "@orderly.network/hooks";

function TradingGuard({ children }) {
  const { state, createAccount, createOrderlyKey } = useAccount();

  switch (state.status) {
    case AccountStatusEnum.NotConnected:
      return <ConnectWalletPrompt />;
      
    case AccountStatusEnum.Connected:
      return (
        <div>
          <p>Create your Orderly account to start trading</p>
          <Button onClick={() => createAccount()}>
            Create Account
          </Button>
        </div>
      );
      
    case AccountStatusEnum.NotSignedIn:
      return (
        <div>
          <p>Enable trading by creating your trading key</p>
          <Button onClick={() => createOrderlyKey()}>
            Enable Trading
          </Button>
        </div>
      );
      
    case AccountStatusEnum.SignedIn:
      return children;
  }
}
```

### Using AuthGuard (Simpler)

```tsx
import { AuthGuard } from "@orderly.network/ui-connector";

function TradingPage() {
  return (
    <AuthGuard>
      <TradingInterface />
    </AuthGuard>
  );
}
```

---

## 2. Deposit Funds

### Using useDeposit Hook

```tsx
import { useDeposit } from "@orderly.network/hooks";

function DepositForm() {
  const {
    deposit,        // Execute deposit
    balance,        // Wallet balance
    allowance,      // Token allowance
    approve,        // Approve tokens
    isNativeToken,  // Is ETH/SOL
    quantity,       // Deposit amount
    setQuantity,    // Set amount
    depositFee,     // Estimated fee
  } = useDeposit({
    address: "0xUSDC_ADDRESS",
    decimals: 6,
    srcChainId: 42161, // Arbitrum
    srcToken: "USDC",
  });

  const handleDeposit = async () => {
    try {
      // Check and approve if needed
      if (!isNativeToken && allowance < quantity) {
        await approve();
      }
      
      // Execute deposit
      const result = await deposit();
      toast.success("Deposit successful!");
    } catch (error) {
      toast.error(error.message);
    }
  };

  return (
    <div>
      <Input
        value={quantity}
        onChange={(e) => setQuantity(e.target.value)}
        placeholder="Amount"
      />
      <Text>Balance: {balance} USDC</Text>
      <Text>Fee: {depositFee} USDC</Text>
      <Button onClick={handleDeposit}>Deposit</Button>
    </div>
  );
}
```

### Using DepositForm Component

```tsx
import { DepositForm } from "@orderly.network/ui-transfer";

function DepositPage() {
  return (
    <DepositForm
      onDeposit={(result) => {
        toast.success("Deposited successfully!");
      }}
    />
  );
}
```

---

## 3. Check Account Balance

### Using useCollateral

```tsx
import { useCollateral } from "@orderly.network/hooks";

function AccountBalance() {
  const {
    totalCollateral,    // Total collateral value
    freeCollateral,     // Available for trading
    totalValue,         // Total account value
    availableBalance,   // Available balance
    unsettledPnL,       // Unrealized P&L
  } = useCollateral({ dp: 2 });

  return (
    <Card>
      <Text>Total Value: ${totalValue}</Text>
      <Text>Free Collateral: ${freeCollateral}</Text>
      <Text>Unsettled PnL: ${unsettledPnL}</Text>
    </Card>
  );
}
```

### Using useHoldingStream

```tsx
import { useHoldingStream } from "@orderly.network/hooks";

function Holdings() {
  const { data: holdings } = useHoldingStream();

  return (
    <ul>
      {holdings?.map((asset) => (
        <li key={asset.token}>
          {asset.token}: {asset.holding}
        </li>
      ))}
    </ul>
  );
}
```

---

## 4. Place Orders

### Market Order

```tsx
import { useMutation } from "@orderly.network/hooks";
import { OrderType, OrderSide } from "@orderly.network/types";

function MarketOrderButton({ symbol, side, quantity }) {
  const [submitOrder] = useMutation("/v1/order");

  const placeMarketOrder = async () => {
    const order = {
      symbol,
      side,
      order_type: OrderType.MARKET,
      order_quantity: quantity,
    };

    try {
      const result = await submitOrder(order);
      if (result.success) {
        toast.success(`Order placed: ${result.data.order_id}`);
      }
    } catch (error) {
      toast.error(error.message);
    }
  };

  return (
    <Button 
      color={side === OrderSide.BUY ? "buy" : "sell"}
      onClick={placeMarketOrder}
    >
      {side === OrderSide.BUY ? "Buy" : "Sell"} Market
    </Button>
  );
}
```

### Limit Order

```tsx
function LimitOrderForm({ symbol }) {
  const [submitOrder] = useMutation("/v1/order");
  const [price, setPrice] = useState("");
  const [quantity, setQuantity] = useState("");
  const [side, setSide] = useState(OrderSide.BUY);

  const placeLimitOrder = async () => {
    const order = {
      symbol,
      side,
      order_type: OrderType.LIMIT,
      order_price: parseFloat(price),
      order_quantity: parseFloat(quantity),
    };

    const result = await submitOrder(order);
    if (result.success) {
      toast.success("Limit order placed!");
    }
  };

  return (
    <div>
      <Input
        label="Price"
        value={price}
        onChange={(e) => setPrice(e.target.value)}
      />
      <Input
        label="Quantity"
        value={quantity}
        onChange={(e) => setQuantity(e.target.value)}
      />
      <Flex gap={2}>
        <Button color="buy" onClick={() => { setSide(OrderSide.BUY); placeLimitOrder(); }}>
          Buy
        </Button>
        <Button color="sell" onClick={() => { setSide(OrderSide.SELL); placeLimitOrder(); }}>
          Sell
        </Button>
      </Flex>
    </div>
  );
}
```

### Using useOrderEntry (Recommended)

```tsx
import { useOrderEntry, OrderSide, OrderType } from "@orderly.network/hooks";

function OrderEntryForm({ symbol }) {
  const {
    formattedOrder,  // Validated order
    submit,          // Submit function
    maxQty,          // Max quantity
    freeCollateral,  // Available margin
    errors,          // Validation errors
    setValues,       // Update form
    helper,          // Helper functions
  } = useOrderEntry({
    symbol,
    side: OrderSide.BUY,
    order_type: OrderType.LIMIT,
  });

  const handleSubmit = async () => {
    if (Object.keys(errors).length > 0) {
      toast.error("Please fix validation errors");
      return;
    }
    
    try {
      await submit();
      toast.success("Order submitted!");
    } catch (error) {
      toast.error(error.message);
    }
  };

  return (
    <div>
      <Input
        label="Price"
        value={formattedOrder.order_price}
        onChange={(e) => setValues({ order_price: e.target.value })}
        error={errors.order_price}
      />
      <Input
        label="Quantity"
        value={formattedOrder.order_quantity}
        onChange={(e) => setValues({ order_quantity: e.target.value })}
        error={errors.order_quantity}
      />
      <Text size="xs">Max: {maxQty}</Text>
      <Text size="xs">Available: ${freeCollateral}</Text>
      <Button onClick={handleSubmit}>Submit Order</Button>
    </div>
  );
}
```

### Using OrderEntryWidget (Easiest)

```tsx
import { OrderEntryWidget } from "@orderly.network/ui-order-entry";

function TradingPage({ symbol }) {
  return (
    <OrderEntryWidget symbol={symbol} />
  );
}
```

---

## 5. Order Types

### Stop-Loss / Take-Profit Orders

```tsx
import { useTPSLOrder } from "@orderly.network/hooks";

function TPSLManager({ symbol, position }) {
  const [tpslOrder, { update, remove }] = useTPSLOrder(
    symbol,
    position.position_qty
  );

  const setTPSL = async () => {
    await update({
      tp_trigger_price: position.mark_price * 1.05, // 5% profit
      sl_trigger_price: position.mark_price * 0.97, // 3% loss
    });
  };

  return (
    <div>
      <Button onClick={setTPSL}>Set TP/SL</Button>
      <Button onClick={remove}>Remove TP/SL</Button>
    </div>
  );
}
```

### Using TPSL Widget

```tsx
import { PositionTPSLPopover } from "@orderly.network/ui-tpsl";

<PositionTPSLPopover position={position} />
```

---

## 6. Monitor Orders

### Active Orders

```tsx
import { useOrderStream } from "@orderly.network/hooks";

function ActiveOrders({ symbol }) {
  const [orders, { refresh }] = useOrderStream({
    symbol,
    status: "OPEN",
  });

  return (
    <DataTable
      columns={[
        { header: "Symbol", accessorKey: "symbol" },
        { header: "Side", accessorKey: "side" },
        { header: "Type", accessorKey: "type" },
        { header: "Price", accessorKey: "price" },
        { header: "Quantity", accessorKey: "quantity" },
        { header: "Status", accessorKey: "status" },
        {
          header: "Actions",
          cell: ({ row }) => (
            <Button onClick={() => cancelOrder(row.order_id)}>
              Cancel
            </Button>
          ),
        },
      ]}
      data={orders}
    />
  );
}
```

### Cancel Order

```tsx
import { useMutation } from "@orderly.network/hooks";

function useCancelOrder() {
  const [cancel] = useMutation("/v1/order", "DELETE");

  return async (orderId: string, symbol: string) => {
    const result = await cancel({
      order_id: orderId,
      symbol,
    });
    return result;
  };
}
```

### Cancel All Orders

```tsx
const [cancelAll] = useMutation("/v1/orders", "DELETE");

const cancelAllOrders = async (symbol?: string) => {
  await cancelAll({ symbol }); // Optional: filter by symbol
};
```

---

## 7. Monitor Positions

### Real-time Position Stream

```tsx
import { usePositionStream } from "@orderly.network/hooks";

function PositionsTable({ symbol }) {
  const [
    positions,
    positionInfo,
    { loading, refresh }
  ] = usePositionStream(symbol);

  return (
    <DataTable
      columns={[
        { header: "Symbol", accessorKey: "symbol" },
        { header: "Size", accessorKey: "position_qty" },
        { header: "Entry Price", accessorKey: "average_open_price" },
        { header: "Mark Price", accessorKey: "mark_price" },
        {
          header: "Unrealized PnL",
          accessorKey: "unrealized_pnl",
          cell: ({ getValue }) => (
            <Text color={getValue() >= 0 ? "success" : "danger"}>
              ${getValue().toFixed(2)}
            </Text>
          ),
        },
        {
          header: "ROI",
          accessorKey: "unrealized_pnl_ROI",
          cell: ({ getValue }) => (
            <Text color={getValue() >= 0 ? "success" : "danger"}>
              {(getValue() * 100).toFixed(2)}%
            </Text>
          ),
        },
      ]}
      data={positions}
    />
  );
}
```

### Using PositionsWidget

```tsx
import { PositionsWidget } from "@orderly.network/ui-positions";

<PositionsWidget symbol="PERP_ETH_USDC" />
```

---

## 8. Close Positions

### Close Specific Position

```tsx
import { useMutation } from "@orderly.network/hooks";
import { OrderType, OrderSide } from "@orderly.network/types";

function ClosePositionButton({ position }) {
  const [submitOrder] = useMutation("/v1/order");

  const closePosition = async () => {
    const order = {
      symbol: position.symbol,
      side: position.position_qty > 0 ? OrderSide.SELL : OrderSide.BUY,
      order_type: OrderType.MARKET,
      order_quantity: Math.abs(position.position_qty),
      reduce_only: true,
    };

    const result = await submitOrder(order);
    if (result.success) {
      toast.success("Position closed!");
    }
  };

  return (
    <Button color="danger" onClick={closePosition}>
      Close Position
    </Button>
  );
}
```

### Close All Positions

```tsx
function CloseAllPositions() {
  const [positions] = usePositionStream();
  const [submitOrder] = useMutation("/v1/order");

  const closeAll = async () => {
    for (const position of positions) {
      if (position.position_qty !== 0) {
        await submitOrder({
          symbol: position.symbol,
          side: position.position_qty > 0 ? OrderSide.SELL : OrderSide.BUY,
          order_type: OrderType.MARKET,
          order_quantity: Math.abs(position.position_qty),
          reduce_only: true,
        });
      }
    }
    toast.success("All positions closed!");
  };

  return <Button onClick={closeAll}>Close All</Button>;
}
```

---

## 9. Withdraw Funds

### Using useWithdraw Hook

```tsx
import { useWithdraw } from "@orderly.network/hooks";

function WithdrawForm() {
  const {
    withdraw,          // Execute withdrawal
    maxAmount,         // Max withdrawable
    availableWithdraw, // Available to withdraw
    unsettledPnL,      // Pending P&L
  } = useWithdraw({
    srcChainId: 42161,
    token: "USDC",
    decimals: 6,
  });

  const [amount, setAmount] = useState("");

  const handleWithdraw = async () => {
    try {
      await withdraw(parseFloat(amount));
      toast.success("Withdrawal initiated!");
    } catch (error) {
      toast.error(error.message);
    }
  };

  return (
    <div>
      <Input
        value={amount}
        onChange={(e) => setAmount(e.target.value)}
        placeholder="Amount"
      />
      <Text>Available: {availableWithdraw} USDC</Text>
      <Text>Max: {maxAmount} USDC</Text>
      <Button onClick={handleWithdraw}>Withdraw</Button>
    </div>
  );
}
```

### Using WithdrawForm Component

```tsx
import { WithdrawForm } from "@orderly.network/ui-transfer";

<WithdrawForm
  onWithdraw={(result) => {
    toast.success("Withdrawal successful!");
  }}
/>
```

---

## 10. Leverage Management

### Get/Set Leverage

```tsx
import { useLeverage } from "@orderly.network/hooks";

function LeverageControl() {
  const {
    curLeverage,      // Current leverage
    maxLeverage,      // Max allowed
    leverageLevers,   // Available options
    update,           // Update function
    isLoading,
  } = useLeverage();

  const changeLeverage = async (newLeverage: number) => {
    try {
      await update({ leverage: newLeverage });
      toast.success(`Leverage set to ${newLeverage}x`);
    } catch (error) {
      toast.error(error.message);
    }
  };

  return (
    <div>
      <Text>Current: {curLeverage}x</Text>
      <Slider
        value={[curLeverage]}
        onValueChange={([v]) => changeLeverage(v)}
        min={1}
        max={maxLeverage}
        marks={leverageLevers.map(v => ({ value: v, label: `${v}x` }))}
      />
    </div>
  );
}
```

### Using LeverageEditor Widget

```tsx
import { LeverageEditor } from "@orderly.network/ui-leverage";

<LeverageEditor />
```

---

## 11. Risk Monitoring

### Margin Ratio

```tsx
import { useMarginRatio } from "@orderly.network/hooks";

function RiskIndicator() {
  const {
    marginRatio,      // Current margin ratio
    mmr,              // Maintenance margin ratio
    currentLeverage,  // Current effective leverage
  } = useMarginRatio();

  const isAtRisk = marginRatio < mmr * 1.5;

  return (
    <Card>
      <Text>Margin Ratio: {(marginRatio * 100).toFixed(2)}%</Text>
      <Text>MMR: {(mmr * 100).toFixed(2)}%</Text>
      <Text>Leverage: {currentLeverage.toFixed(2)}x</Text>
      {isAtRisk && (
        <Badge color="warning">Approaching Liquidation</Badge>
      )}
    </Card>
  );
}
```

### Liquidation Price

```tsx
import { usePositionStream, useSymbolsInfo } from "@orderly.network/hooks";

function LiquidationWarning({ symbol }) {
  const [positions] = usePositionStream(symbol);
  const symbolsInfo = useSymbolsInfo();
  
  const position = positions?.[0];
  if (!position) return null;

  const liqPrice = position.est_liq_price;
  const markPrice = position.mark_price;
  const distance = Math.abs((liqPrice - markPrice) / markPrice * 100);

  return (
    <div>
      <Text>Liquidation Price: ${liqPrice?.toFixed(2)}</Text>
      {distance < 10 && (
        <Text color="danger">
          Warning: {distance.toFixed(1)}% from liquidation
        </Text>
      )}
    </div>
  );
}
```

---

## 12. Trading History

### Trade History

```tsx
import { useTradeHistory } from "@orderly.network/hooks";

function TradeHistory({ symbol }) {
  const { data: trades, isLoading } = useTradeHistory({
    symbol,
    limit: 50,
  });

  return (
    <DataTable
      columns={[
        { header: "Time", accessorKey: "executed_timestamp" },
        { header: "Symbol", accessorKey: "symbol" },
        { header: "Side", accessorKey: "side" },
        { header: "Price", accessorKey: "executed_price" },
        { header: "Quantity", accessorKey: "executed_quantity" },
        { header: "Fee", accessorKey: "fee" },
      ]}
      data={trades}
      loading={isLoading}
    />
  );
}
```

### Funding Fee History

```tsx
import { useFundingFeeHistory } from "@orderly.network/hooks";

function FundingHistory({ symbol }) {
  const { data } = useFundingFeeHistory({ symbol });

  return (
    <DataTable
      columns={[
        { header: "Time", accessorKey: "created_time" },
        { header: "Symbol", accessorKey: "symbol" },
        { header: "Funding Fee", accessorKey: "funding_fee" },
        { header: "Funding Rate", accessorKey: "funding_rate" },
      ]}
      data={data}
    />
  );
}
```

---

## Complete Trading Page Example

```tsx
import { useState } from "react";
import { useParams, useNavigate } from "react-router-dom";
import { TradingPage } from "@orderly.network/trading";
import { AuthGuard } from "@orderly.network/ui-connector";
import { API } from "@orderly.network/types";

export default function Trading() {
  const { symbol } = useParams();
  const navigate = useNavigate();

  const onSymbolChange = (data: API.Symbol) => {
    navigate(`/trade/${data.symbol}`);
  };

  return (
    <AuthGuard>
      <TradingPage
        symbol={symbol!}
        onSymbolChange={onSymbolChange}
        tradingViewConfig={{
          scriptSRC: "/tradingview/charting_library/charting_library.js",
          library_path: "/tradingview/charting_library/",
        }}
        sharePnLConfig={{
          backgroundImages: ["/pnl-bg-1.png", "/pnl-bg-2.png"],
        }}
      />
    </AuthGuard>
  );
}
```

---

## Error Handling Patterns

### Global Error Handler

```tsx
import { useEventEmitter } from "@orderly.network/hooks";
import { toast } from "@orderly.network/ui";

function ErrorHandler() {
  const ee = useEventEmitter();

  useEffect(() => {
    const handlers = {
      "order:error": (e) => toast.error(`Order failed: ${e.message}`),
      "deposit:error": (e) => toast.error(`Deposit failed: ${e.message}`),
      "withdraw:error": (e) => toast.error(`Withdraw failed: ${e.message}`),
    };

    Object.entries(handlers).forEach(([event, handler]) => {
      ee.on(event, handler);
    });

    return () => {
      Object.entries(handlers).forEach(([event, handler]) => {
        ee.off(event, handler);
      });
    };
  }, [ee]);

  return null;
}
```

### Order-Specific Errors

```tsx
import { useOrderEntry } from "@orderly.network/hooks";

function OrderForm() {
  const { errors, submit } = useOrderEntry({ ... });

  const handleSubmit = async () => {
    // Validation errors
    if (errors.order_price) {
      toast.error(`Price: ${errors.order_price}`);
      return;
    }
    if (errors.order_quantity) {
      toast.error(`Quantity: ${errors.order_quantity}`);
      return;
    }

    try {
      await submit();
    } catch (error) {
      // API errors
      if (error.code === "INSUFFICIENT_BALANCE") {
        toast.error("Insufficient balance. Please deposit more funds.");
      } else if (error.code === "RISK_TOO_HIGH") {
        toast.error("Order rejected: would exceed risk limits.");
      } else {
        toast.error(error.message);
      }
    }
  };
}
```

---

## Best Practices

### 1. Always Check Auth Before Trading
```tsx
const { state } = useAccount();
if (state.status !== AccountStatusEnum.SignedIn) {
  return <AuthGuard>{children}</AuthGuard>;
}
```

### 2. Validate Orders Before Submission
```tsx
const { errors, validated } = metaState;
if (!validated || Object.keys(errors).length > 0) {
  // Don't submit
}
```

### 3. Use Real-time Streams for Position Data
```tsx
// ✅ Good - real-time updates
const [positions] = usePositionStream();

// ❌ Avoid - polling
const { data } = useQuery("/v1/positions", { refreshInterval: 1000 });
```

### 4. Handle Loading States
```tsx
if (isLoading) return <Spinner />;
if (error) return <ErrorMessage error={error} />;
```

### 5. Confirm High-Risk Actions
```tsx
const closePosition = async () => {
  const confirmed = await modal.confirm({
    title: "Close Position",
    content: `Close ${position.symbol} position for ${pnl >= 0 ? "profit" : "loss"}?`,
  });
  if (confirmed) {
    await executeClose();
  }
};
```

### 6. Show Execution Feedback
```tsx
toast.loading("Submitting order...");
try {
  await submit();
  toast.success("Order placed!");
} catch (e) {
  toast.error(e.message);
}
```
