---
name: add-feature-page
description: Add a high-level feature page (Trading, Portfolio, Vaults) to the DEX project using Orderly SDK widgets.
argument-hint: <feature-type> (trading|portfolio|vaults|markets)
---

# Add Feature Page Skill

Use this skill to add specialized pages using the SDK's high-level widgets. These widgets are full-page experiences (orderbook, chart, positions, etc.).

## 1. Trading Page

**Import:** `@orderly.network/trading`
**Component:** `<TradingPage />`

```tsx
// src/pages/Trading.tsx
import { TradingPage } from "@orderly.network/trading";
import { useParams, useNavigate } from "react-router-dom";

export const Trading = () => {
  const { symbol } = useParams();
  const navigate = useNavigate();

  const onSymbolChange = (data) => {
    navigate(`/trading/${data.symbol}`);
  };

  return (
    <TradingPage 
      symbol={symbol} 
      onSymbolChange={onSymbolChange}
      tradingViewConfig={/* optional */}
      sharePnLConfig={/* optional */}
    />
  );
};
```

## 2. Portfolio Page

**Import:** `@orderly.network/portfolio`
**Component:** `<Portfolio />`

```tsx
// src/pages/Portfolio.tsx
import { Portfolio } from "@orderly.network/portfolio";

export const MyPortfolio = () => {
  return (
    <div className="p-4">
      <h1 className="text-2xl font-bold mb-4">My Assets</h1>
      <Portfolio />
    </div>
  );
};
```

## 3. Markets Page

**Import:** `@orderly.network/markets`
**Component:** `<MarketsPage />`

```tsx
// src/pages/Markets.tsx
import { MarketsPage } from "@orderly.network/markets";

export const Markets = () => {
  return <MarketsPage />;
};
```

## 4. Vaults (Earn) Page

**Import:** `@orderly.network/vaults`
**Component:** `<Vaults />`

```tsx
// src/pages/Vaults.tsx
import { Vaults } from "@orderly.network/vaults";

export const VaultsPage = () => {
  return <Vaults />;
};
```

## Setup Steps

1.  **Install Package:** `npm install @orderly.network/[feature]`
2.  **Create Page Component:** `src/pages/[Feature].tsx`
3.  **Add Route:** Update `src/App.tsx` (e.g., `<Route path="/portfolio" element={<Portfolio />} />`)
4.  **Add Navigation:** Add link to Navbar (if exists).
