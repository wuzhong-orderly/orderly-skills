---
name: scaffold-dex
description: Initialize a new DEX project using Vite + React + Orderly SDK. Sets up the OrderlyProvider, Router, and essential dependencies.
argument-hint: <project-name>
---

# Scaffold DEX Skill

Use this skill to create a new DEX project. It sets up the foundational structure required by the Orderly SDK.

## Tech Stack
- **Framework:** Vite + React (TypeScript)
- **Routing:** React Router (v6/v7)
- **Styling:** Tailwind CSS
- **SDK:** `@orderly.network/react-app` (Provider)

## Output Structure

```text
my-dex/
├── .env                 # Broker ID, Network ID
├── index.html
├── vite.config.ts
├── tailwind.config.ts
└── src/
    ├── main.tsx         # Entry point
    ├── App.tsx          # Router setup
    ├── orderly/         # SDK Configuration
    │   ├── config.ts    # Theme, icons
    │   └── provider.tsx # <OrderlyAppProvider> wrapper
    └── pages/           # Route components
```

## Key Components

### 1. `src/orderly/provider.tsx`
Wraps the app with `OrderlyAppProvider` and `WalletConnector`.

```tsx
import { OrderlyAppProvider } from "@orderly.network/react-app";
import { WalletConnector } from "@orderly.network/wallet-connector";

export const OrderlyProvider = ({ children }) => (
  <OrderlyAppProvider
    brokerId={import.meta.env.VITE_BROKER_ID}
    networkId="testnet"
    appIcons={...}
  >
    <WalletConnector>
      {children}
    </WalletConnector>
  </OrderlyAppProvider>
);
```

### 2. `src/App.tsx`
Sets up the router and layout.

```tsx
import { OrderlyProvider } from "./orderly/provider";

export default function App() {
  return (
    <OrderlyProvider>
      <Router>
        <Routes>
          <Route path="/" element={<Navigate to="/trading/PERP_ETH_USDC" />} />
          <Route path="/trading/:symbol" element={<TradingPage />} />
        </Routes>
      </Router>
    </OrderlyProvider>
  );
}
```

## Dependencies to Install
- `@orderly.network/react-app`
- `@orderly.network/wallet-connector`
- `@orderly.network/types`
- `react-router-dom`
- `tailwindcss`
