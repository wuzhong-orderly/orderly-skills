# Orderly SDK DEX Architecture

A comprehensive guide to architecting and scaffolding a complete DEX application using the Orderly Network SDK.

## Overview

This skill covers the complete architecture for building a production-ready DEX:

- Project structure and setup
- Provider hierarchy and configuration
- **Network configuration (REQUIRED)** - mainnet/testnet and supported chains
- Routing and page components
- Runtime configuration
- Build and deployment

**Critical Configuration**: Every DEX must have:
1. `brokerId` - Your Orderly broker ID
2. `networkId` - Either "mainnet" or "testnet"
3. Proper wallet connector setup with matching network

---

## Project Structure

```
my-dex/
├── public/
│   ├── config.js              # Runtime configuration
│   ├── favicon.webp
│   ├── locales/               # i18n translations
│   │   ├── en.json
│   │   ├── zh.json
│   │   └── extend/            # Custom translations
│   └── tradingview/           # TradingView library (optional)
│       └── charting_library/
├── src/
│   ├── main.tsx               # Entry point
│   ├── App.tsx                # Root component with router
│   ├── components/
│   │   ├── orderlyProvider/   # SDK provider setup
│   │   │   ├── index.tsx      # Main provider wrapper
│   │   │   ├── walletConnector.tsx
│   │   │   └── privyConnector.tsx
│   │   ├── ErrorBoundary.tsx
│   │   └── LoadingSpinner.tsx
│   ├── pages/
│   │   ├── Index.tsx          # Home/redirect
│   │   ├── perp/              # Trading pages
│   │   │   ├── Layout.tsx
│   │   │   ├── Index.tsx
│   │   │   └── Symbol.tsx
│   │   ├── portfolio/         # Portfolio pages
│   │   │   ├── Layout.tsx
│   │   │   ├── Index.tsx
│   │   │   ├── Positions.tsx
│   │   │   ├── Orders.tsx
│   │   │   └── Assets.tsx
│   │   ├── markets/           # Markets pages
│   │   └── leaderboard/       # Leaderboard pages
│   ├── utils/
│   │   ├── config.tsx         # App configuration
│   │   ├── walletConfig.ts    # Wallet setup
│   │   ├── runtime-config.ts  # Runtime config loader
│   │   └── storage.ts         # Local storage utils
│   └── styles/
│       └── index.css          # Global styles + Tailwind
├── .env                       # Build-time env vars
├── index.html
├── package.json
├── tailwind.config.ts
├── tsconfig.json
└── vite.config.ts
```

---

## Tech Stack

| Category | Technology |
|----------|------------|
| Framework | React 18 + TypeScript |
| Build Tool | Vite |
| Routing | React Router v6/v7 |
| Styling | Tailwind CSS |
| State | Orderly hooks (SWR + Zustand) |
| UI Components | @orderly.network/ui |

---

## 1. Project Setup

### Initialize Project

```bash
# Create Vite React project
npm create vite@latest my-dex -- --template react-ts
cd my-dex

# Install Orderly SDK packages
npm install @orderly.network/react-app \
            @orderly.network/trading \
            @orderly.network/portfolio \
            @orderly.network/markets \
            @orderly.network/hooks \
            @orderly.network/ui \
            @orderly.network/wallet-connector \
            @orderly.network/i18n \
            @orderly.network/types

# Install routing
npm install react-router-dom

# Install Tailwind
npm install -D tailwindcss postcss autoprefixer
npx tailwindcss init -p

# REQUIRED: Install EVM wallet packages (for MetaMask, WalletConnect, etc.)
npm install @web3-onboard/injected-wallets @web3-onboard/walletconnect

# REQUIRED: Install Solana wallet packages
npm install @solana/wallet-adapter-base @solana/wallet-adapter-wallets
```

### package.json

> **IMPORTANT**: The package.json MUST include both EVM wallet packages (`@web3-onboard/*`) AND Solana wallet packages (`@solana/wallet-adapter-*`). Without these, users cannot connect wallets.

```json
{
  "name": "my-dex",
  "version": "1.0.0",
  "scripts": {
    "dev": "vite",
    "build": "tsc && vite build",
    "preview": "vite preview"
  },
  "dependencies": {
    "@orderly.network/react-app": "^2.8.0",
    "@orderly.network/trading": "^2.8.0",
    "@orderly.network/portfolio": "^2.8.0",
    "@orderly.network/markets": "^2.8.0",
    "@orderly.network/hooks": "^2.8.0",
    "@orderly.network/ui": "^2.8.0",
    "@orderly.network/wallet-connector": "^2.8.0",
    "@orderly.network/i18n": "^2.8.0",
    "@orderly.network/types": "^2.8.0",
    "@web3-onboard/injected-wallets": "^2.x",
    "@web3-onboard/walletconnect": "^2.x",
    "@solana/wallet-adapter-base": "^0.9.x",
    "@solana/wallet-adapter-wallets": "^0.19.x",
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "react-router-dom": "^6.20.0"
  },
  "devDependencies": {
    "@types/react": "^18.2.0",
    "@types/react-dom": "^18.2.0",
    "autoprefixer": "^10.4.16",
    "postcss": "^8.4.31",
    "tailwindcss": "^3.4.0",
    "typescript": "^5.3.0",
    "vite": "^5.0.0"
  }
}
```

---

## 2. Provider Hierarchy

The SDK requires a specific provider nesting order:

```
LocaleProvider (i18n)
└── WalletConnectorProvider (or Privy)
    └── OrderlyAppProvider
        └── ModalProvider (from UI)
            └── TooltipProvider
                └── Your App
```

### Main Provider Component

```tsx
// src/components/orderlyProvider/index.tsx
import { ReactNode, useCallback, Suspense, lazy } from "react";
import { OrderlyAppProvider } from "@orderly.network/react-app";
import { LocaleProvider, LocaleCode, defaultLanguages } from "@orderly.network/i18n";
import type { NetworkId } from "@orderly.network/types";
import { useOrderlyConfig } from "@/utils/config";
import { getRuntimeConfig, getRuntimeConfigBoolean } from "@/utils/runtime-config";

const NETWORK_ID_KEY = "orderly_network_id";

const getNetworkId = (): NetworkId => {
  if (typeof window === "undefined") return "mainnet";
  
  const disableMainnet = getRuntimeConfigBoolean('VITE_DISABLE_MAINNET');
  const disableTestnet = getRuntimeConfigBoolean('VITE_DISABLE_TESTNET');
  
  if (disableMainnet && !disableTestnet) return "testnet";
  if (disableTestnet && !disableMainnet) return "mainnet";
  
  return (localStorage.getItem(NETWORK_ID_KEY) as NetworkId) || "mainnet";
};

const WalletConnector = lazy(() => import("./walletConnector"));

const OrderlyProvider = ({ children }: { children: ReactNode }) => {
  const config = useOrderlyConfig();
  const networkId = getNetworkId();

  const onChainChanged = useCallback(
    (_chainId: number, { isTestnet }: { isTestnet: boolean }) => {
      const currentNetworkId = getNetworkId();
      if ((isTestnet && currentNetworkId === 'mainnet') || 
          (!isTestnet && currentNetworkId === 'testnet')) {
        const newNetworkId: NetworkId = isTestnet ? 'testnet' : 'mainnet';
        localStorage.setItem(NETWORK_ID_KEY, newNetworkId);
        window.location.reload();
      }
    },
    []
  );

  const onLanguageChanged = (lang: LocaleCode) => {
    const url = new URL(window.location.href);
    if (lang === 'en') {
      url.searchParams.delete('lang');
    } else {
      url.searchParams.set('lang', lang);
    }
    window.history.replaceState({}, '', url.toString());
  };

  return (
    <LocaleProvider
      onLanguageChanged={onLanguageChanged}
      locale="en"
      languages={defaultLanguages}
    >
      <Suspense fallback={<LoadingSpinner />}>
        <WalletConnector networkId={networkId}>
          <OrderlyAppProvider
            brokerId={getRuntimeConfig('VITE_ORDERLY_BROKER_ID')}
            brokerName={getRuntimeConfig('VITE_ORDERLY_BROKER_NAME')}
            networkId={networkId}
            onChainChanged={onChainChanged}
            appIcons={config.appIcons}
          >
            {children}
          </OrderlyAppProvider>
        </WalletConnector>
      </Suspense>
    </LocaleProvider>
  );
};

export default OrderlyProvider;
```

### Wallet Connector Setup (REQUIRED)

> **IMPORTANT**: The `WalletConnectorProvider` MUST have both `evmInitial` AND `solanaInitial` configured. Without `evmInitial`, users cannot connect EVM wallets like MetaMask. Without `solanaInitial`, users cannot connect Solana wallets like Phantom.

```tsx
// src/components/orderlyProvider/walletConnector.tsx
import { ReactNode } from 'react';
import { WalletConnectorProvider } from '@orderly.network/wallet-connector';
import { WalletAdapterNetwork } from "@solana/wallet-adapter-base";
import type { NetworkId } from "@orderly.network/types";
import { getEvmInitialConfig, getSolanaWallets } from '@/utils/walletConfig';

interface Props {
  children: ReactNode;
  networkId: NetworkId;
}

const WalletConnector = ({ children, networkId }: Props) => {
  // REQUIRED: EVM wallet config for MetaMask, WalletConnect, etc.
  const evmInitial = getEvmInitialConfig();

  // REQUIRED: Solana wallet config for Phantom, Solflare, etc.
  const solanaInitial = {
    network: networkId === 'mainnet' 
      ? WalletAdapterNetwork.Mainnet 
      : WalletAdapterNetwork.Devnet,
    wallets: getSolanaWallets(networkId),
  };

  return (
    <WalletConnectorProvider
      solanaInitial={solanaInitial}  // REQUIRED
      evmInitial={evmInitial}        // REQUIRED for EVM wallet support
    >
      {children}
    </WalletConnectorProvider>
  );
};

export default WalletConnector;
```

---

## 3. Network Configuration (REQUIRED)

**IMPORTANT**: Every Orderly DEX must configure the network properly. Without proper network configuration, the DEX will not function correctly.

### Supported Networks

Orderly Network supports the following chains:

#### Mainnet Chains (Production)

| Chain | Chain ID | Description |
|-------|----------|-------------|
| Arbitrum | 42161 | Primary mainnet chain |
| Optimism | 10 | OP mainnet |
| Base | 8453 | Base mainnet |
| Ethereum | 1 | Ethereum mainnet |
| BNB Chain | 56 | Binance Smart Chain |
| Polygon | 137 | Polygon mainnet |
| Mantle | 5000 | Mantle mainnet |
| Sei | 1329 | Sei Network |
| Solana | N/A | Solana mainnet |

#### Testnet Chains (Development)

| Chain | Chain ID | Description |
|-------|----------|-------------|
| Arbitrum Sepolia | 421614 | Primary testnet chain |
| Base Sepolia | 84532 | Base testnet |
| Optimism Sepolia | 11155420 | Optimism testnet |
| Solana Devnet | 901901901 | Solana devnet |

#### Default Chain Configuration

```typescript
// Default chains used by the SDK:
// Mainnet: Arbitrum (42161), Base (8453), Optimism (10)
// Testnet: Arbitrum Sepolia (421614)
```

### Network ID Configuration

The `networkId` prop determines whether your DEX connects to mainnet or testnet. **This is required on both OrderlyAppProvider and WalletConnectorProvider**.

```tsx
import type { NetworkId } from "@orderly.network/types";

// Network ID must be "mainnet" or "testnet"
const networkId: NetworkId = "mainnet"; // or "testnet"
```

### Complete Provider Setup with Network Config

```tsx
// src/components/orderlyProvider/index.tsx
import { ReactNode, useCallback, useState } from "react";
import { OrderlyAppProvider } from "@orderly.network/react-app";
import { WalletConnectorProvider } from "@orderly.network/wallet-connector";
import { WalletAdapterNetwork } from "@solana/wallet-adapter-base";
import type { NetworkId } from "@orderly.network/types";

const NETWORK_ID_KEY = "orderly_network_id";

// Get current network from localStorage or default to mainnet
const getNetworkId = (): NetworkId => {
  if (typeof window === "undefined") return "mainnet";
  return (localStorage.getItem(NETWORK_ID_KEY) as NetworkId) || "mainnet";
};

const OrderlyProvider = ({ children }: { children: ReactNode }) => {
  const [networkId] = useState<NetworkId>(getNetworkId);

  // Handle chain changes between mainnet/testnet
  const onChainChanged = useCallback(
    (_chainId: number, { isTestnet }: { isTestnet: boolean }) => {
      const currentNetworkId = getNetworkId();
      if ((isTestnet && currentNetworkId === "mainnet") || 
          (!isTestnet && currentNetworkId === "testnet")) {
        const newNetworkId: NetworkId = isTestnet ? "testnet" : "mainnet";
        localStorage.setItem(NETWORK_ID_KEY, newNetworkId);
        window.location.reload();
      }
    },
    []
  );

  return (
    <WalletConnectorProvider
      // Solana network must match networkId
      solanaInitial={{
        network: networkId === "mainnet" 
          ? WalletAdapterNetwork.Mainnet 
          : WalletAdapterNetwork.Devnet,
        wallets: [], // Add your Solana wallets
      }}
      evmInitial={{
        options: {
          wallets: [], // Add your EVM wallets
        },
      }}
    >
      <OrderlyAppProvider
        brokerId="your_broker_id"
        brokerName="Your DEX Name"
        networkId={networkId}                    // REQUIRED: "mainnet" or "testnet"
        onChainChanged={onChainChanged}          // Handle network switching
      >
        {children}
      </OrderlyAppProvider>
    </WalletConnectorProvider>
  );
};
```

### Chain Filtering (Optional)

To restrict which chains users can select:

```tsx
// Filter to specific chains only
const chainFilter = {
  mainnet: [
    { id: 42161 },  // Arbitrum
    { id: 8453 },   // Base
    { id: 10 },     // Optimism
  ],
  testnet: [
    { id: 421614 }, // Arbitrum Sepolia
  ],
};

<OrderlyAppProvider
  brokerId="your_broker_id"
  networkId={networkId}
  chainFilter={chainFilter}  // Restrict available chains
>
```

### Default Chain (Optional)

Set a default chain for users:

```tsx
const defaultChain = {
  mainnet: { id: 42161 },  // Default to Arbitrum on mainnet
};

<OrderlyAppProvider
  brokerId="your_broker_id"
  networkId={networkId}
  defaultChain={defaultChain}
>
```

### Environment Variables for Network Config

```bash
# .env
VITE_ORDERLY_BROKER_ID=your_broker_id
VITE_ORDERLY_BROKER_NAME=Your DEX

# Chain configuration
VITE_ORDERLY_MAINNET_CHAINS=42161,10,8453
VITE_ORDERLY_TESTNET_CHAINS=421614
VITE_DEFAULT_CHAIN=42161

# Network toggles
VITE_DISABLE_MAINNET=false
VITE_DISABLE_TESTNET=false
```

---

## 4. Runtime Configuration

Use runtime configuration for deployment flexibility:

### public/config.js

```javascript
window.__RUNTIME_CONFIG__ = {
  "VITE_ORDERLY_BROKER_ID": "your_broker_id",
  "VITE_ORDERLY_BROKER_NAME": "Your DEX Name",
  "VITE_DISABLE_MAINNET": "false",
  "VITE_DISABLE_TESTNET": "false",
  "VITE_ORDERLY_MAINNET_CHAINS": "42161,10,8453,56,1",
  "VITE_ORDERLY_TESTNET_CHAINS": "421614,97",
  "VITE_DEFAULT_CHAIN": "42161",
  "VITE_WALLETCONNECT_PROJECT_ID": "your_project_id",
  "VITE_DISABLE_EVM_WALLETS": "false",
  "VITE_DISABLE_SOLANA_WALLETS": "false",
  "VITE_ENABLED_MENUS": "Trading,Portfolio,Markets,Leaderboard",
  "VITE_AVAILABLE_LANGUAGES": "en,zh,ja,ko",
};
```

### Runtime Config Loader

```tsx
// src/utils/runtime-config.ts

export function getRuntimeConfig(key: string): string {
  // Check runtime config first (from public/config.js)
  if (typeof window !== 'undefined' && window.__RUNTIME_CONFIG__?.[key]) {
    return window.__RUNTIME_CONFIG__[key];
  }
  // Fall back to build-time env vars
  return import.meta.env[key] || '';
}

export function getRuntimeConfigBoolean(key: string): boolean {
  return getRuntimeConfig(key) === 'true';
}

export function getRuntimeConfigArray(key: string): string[] {
  const value = getRuntimeConfig(key);
  if (!value) return [];
  return value.split(',').map(s => s.trim()).filter(Boolean);
}

export function getRuntimeConfigNumber(key: string): number | undefined {
  const value = getRuntimeConfig(key);
  const num = parseInt(value, 10);
  return isNaN(num) ? undefined : num;
}

// TypeScript declaration
declare global {
  interface Window {
    __RUNTIME_CONFIG__?: Record<string, string>;
  }
}
```

### Load Config Before App

```tsx
// src/main.tsx
import React, { lazy } from 'react';
import ReactDOM from 'react-dom/client';
import { createBrowserRouter, RouterProvider } from 'react-router-dom';
import App from './App';
import './styles/index.css';

// Load runtime config before rendering
async function loadRuntimeConfig() {
  return new Promise<void>((resolve) => {
    const script = document.createElement('script');
    script.src = '/config.js';
    script.onload = () => resolve();
    script.onerror = () => resolve(); // Fallback to env vars
    document.head.appendChild(script);
  });
}

// Lazy load pages
const TradingPage = lazy(() => import('./pages/perp/Symbol'));
const PortfolioPage = lazy(() => import('./pages/portfolio/Index'));
const MarketsPage = lazy(() => import('./pages/markets/Index'));

const router = createBrowserRouter([
  {
    path: '/',
    element: <App />,
    children: [
      { index: true, element: <Navigate to="/perp/PERP_ETH_USDC" /> },
      {
        path: 'perp',
        children: [
          { index: true, element: <Navigate to="/perp/PERP_ETH_USDC" /> },
          { path: ':symbol', element: <TradingPage /> },
        ],
      },
      { path: 'portfolio', element: <PortfolioPage /> },
      { path: 'markets', element: <MarketsPage /> },
    ],
  },
]);

loadRuntimeConfig().then(() => {
  ReactDOM.createRoot(document.getElementById('root')!).render(
    <React.StrictMode>
      <RouterProvider router={router} />
    </React.StrictMode>
  );
});
```

---

## 5. App Root Component

```tsx
// src/App.tsx
import { Outlet } from "react-router-dom";
import { Suspense } from "react";
import OrderlyProvider from "@/components/orderlyProvider";
import { LoadingSpinner } from "@/components/LoadingSpinner";
import { ErrorBoundary } from "@/components/ErrorBoundary";

export default function App() {
  return (
    <ErrorBoundary>
      <OrderlyProvider>
        <Suspense fallback={<LoadingSpinner />}>
          <Outlet />
        </Suspense>
      </OrderlyProvider>
    </ErrorBoundary>
  );
}
```

---

## 6. Page Components

### Trading Page

```tsx
// src/pages/perp/Symbol.tsx
import { useCallback, useState, useEffect } from "react";
import { useNavigate, useParams } from "react-router-dom";
import { API } from "@orderly.network/types";
import { TradingPage } from "@orderly.network/trading";

export default function PerpSymbol() {
  const { symbol: paramSymbol } = useParams();
  const [symbol, setSymbol] = useState(paramSymbol!);
  const navigate = useNavigate();

  useEffect(() => {
    // Persist last viewed symbol
    localStorage.setItem("lastSymbol", symbol);
  }, [symbol]);

  const onSymbolChange = useCallback(
    (data: API.Symbol) => {
      setSymbol(data.symbol);
      navigate(`/perp/${data.symbol}`);
    },
    [navigate]
  );

  return (
    <TradingPage
      symbol={symbol}
      onSymbolChange={onSymbolChange}
      tradingViewConfig={{
        scriptSRC: "/tradingview/charting_library/charting_library.js",
        library_path: "/tradingview/charting_library/",
      }}
      sharePnLConfig={{
        backgroundImages: ["/pnl-bg-1.png", "/pnl-bg-2.png"],
      }}
    />
  );
}
```

### Portfolio Page

```tsx
// src/pages/portfolio/Index.tsx
import { OverviewModule } from "@orderly.network/portfolio";

export default function PortfolioIndex() {
  return (
    <div className="oui-portfolio-page">
      <OverviewModule.OverviewPage />
    </div>
  );
}
```

### Markets Page

```tsx
// src/pages/markets/Index.tsx
import { MarketsHomePage } from "@orderly.network/markets";
import { useNavigate } from "react-router-dom";

export default function MarketsIndex() {
  const navigate = useNavigate();

  return (
    <MarketsHomePage
      onSymbolClick={(symbol) => navigate(`/perp/${symbol}`)}
    />
  );
}
```

---

## 7. App Configuration

```tsx
// src/utils/config.tsx
import { useMemo } from "react";
import { TradingPageProps } from "@orderly.network/trading";
import { AppLogos } from "@orderly.network/react-app";
import { getRuntimeConfig, getRuntimeConfigBoolean } from "./runtime-config";

export interface OrderlyConfig {
  appIcons: AppLogos;
  tradingPage: {
    tradingViewConfig: TradingPageProps["tradingViewConfig"];
    sharePnLConfig: TradingPageProps["sharePnLConfig"];
  };
  navigation: {
    items: { name: string; href: string }[];
  };
}

const getMenuItems = () => {
  const enabledMenus = getRuntimeConfig("VITE_ENABLED_MENUS");
  const items = enabledMenus ? enabledMenus.split(",") : ["Trading", "Portfolio", "Markets"];
  
  const menuMap: Record<string, { name: string; href: string }> = {
    Trading: { name: "Trading", href: "/perp/PERP_ETH_USDC" },
    Portfolio: { name: "Portfolio", href: "/portfolio" },
    Markets: { name: "Markets", href: "/markets" },
    Leaderboard: { name: "Leaderboard", href: "/leaderboard" },
  };

  return items.map(item => menuMap[item.trim()]).filter(Boolean);
};

export function useOrderlyConfig(): OrderlyConfig {
  return useMemo(() => ({
    appIcons: {
      main: {
        img: "/logo.svg",
      },
      secondary: {
        img: "/logo-small.svg",
      },
    },
    tradingPage: {
      tradingViewConfig: {
        scriptSRC: "/tradingview/charting_library/charting_library.js",
        library_path: "/tradingview/charting_library/",
      },
      sharePnLConfig: {
        backgroundImages: ["/pnl-bg-1.png", "/pnl-bg-2.png"],
      },
    },
    navigation: {
      items: getMenuItems(),
    },
  }), []);
}
```

---

## 8. Wallet Configuration (REQUIRED)

> **CRITICAL**: Both EVM and Solana wallet configurations are required for a functional DEX. The `evmInitial` config enables MetaMask, WalletConnect, and other EVM wallets. The `solanaInitial` config enables Phantom, Solflare, etc. **You MUST pass both to `WalletConnectorProvider`.**

### Required Packages

```bash
# EVM wallet packages (REQUIRED for MetaMask, WalletConnect)
npm install @web3-onboard/injected-wallets @web3-onboard/walletconnect

# Solana wallet packages (REQUIRED for Phantom, Solflare)
npm install @solana/wallet-adapter-base @solana/wallet-adapter-wallets
```

### walletConfig.ts

```tsx
// src/utils/walletConfig.ts
import injectedOnboard from "@web3-onboard/injected-wallets";
import walletConnectOnboard from "@web3-onboard/walletconnect";
import { PhantomWalletAdapter, SolflareWalletAdapter } from "@solana/wallet-adapter-wallets";
import type { NetworkId } from "@orderly.network/types";

// EVM WALLETS - Required for MetaMask, Coinbase, WalletConnect, etc.
export function getOnboardEvmWallets() {
  const walletConnectProjectId = import.meta.env.VITE_WALLETCONNECT_PROJECT_ID;
  const isBrowser = typeof window !== "undefined";
  
  // Always include injected wallets (MetaMask, Coinbase, Rabby, etc.)
  const wallets = [injectedOnboard()];
  
  // Add WalletConnect if project ID is configured
  if (walletConnectProjectId && isBrowser) {
    wallets.push(
      walletConnectOnboard({
        projectId: walletConnectProjectId,
        qrModalOptions: { themeMode: "dark" },
        dappUrl: window.location.origin,
      })
    );
  }

  return wallets;
}

// REQUIRED: Returns EVM wallet configuration for WalletConnectorProvider
export function getEvmInitialConfig() {
  const wallets = getOnboardEvmWallets();
  
  return {
    options: {
      wallets,
      appMetadata: {
        name: import.meta.env.VITE_ORDERLY_BROKER_NAME || "DEX",
        description: "Decentralized Perpetual Exchange",
      },
    },
  };
}

// SOLANA WALLETS - Required for Phantom, Solflare, etc.
export function getSolanaWallets(_networkId: NetworkId) {
  const isBrowser = typeof window !== "undefined";
  if (!isBrowser) return [];

  return [
    new PhantomWalletAdapter(),
    new SolflareWalletAdapter(),
  ];
}
}
```

---

## 9. Styling Setup

### tailwind.config.ts

```ts
import type { Config } from 'tailwindcss';
import { OUITailwind } from '@orderly.network/ui';

export default {
  content: [
    './index.html',
    './src/**/*.{js,ts,jsx,tsx}',
    './node_modules/@orderly.network/**/*.{js,mjs}',
  ],
  presets: [OUITailwind.preset],
  theme: {
    extend: {},
  },
  plugins: [],
} satisfies Config;
```

### src/styles/index.css

> **Note**: Only `@orderly.network/ui` has a CSS file. Other packages like `trading`, `portfolio`, `markets` do NOT have separate CSS files—they use the base UI styles.

```css
/* Only import from @orderly.network/ui - other packages don't have CSS files */
@import '@orderly.network/ui/dist/styles.css';

@tailwind base;
@tailwind components;
@tailwind utilities;

/* Custom styles */
:root {
  --oui-color-primary: #7b61ff;
  --oui-color-success: #00c853;
  --oui-color-danger: #ff5252;
}

body {
  @apply oui-bg-base-9 oui-text-base-contrast;
}
```

---

## 10. Vite Configuration

> **Important**: The wallet connector packages use Node.js built-ins like `Buffer`. You must add polyfills for browser compatibility.

### Install Vite Polyfills

```bash
npm install -D vite-plugin-node-polyfills
```

### vite.config.ts

```ts
// vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import { nodePolyfills } from 'vite-plugin-node-polyfills';
import path from 'path';

export default defineConfig({
  plugins: [
    react(),
    // Required for wallet libraries that use Node.js built-ins (Buffer, etc.)
    nodePolyfills({
      include: ['buffer', 'crypto', 'stream', 'util'],
      globals: {
        Buffer: true,
        global: true,
        process: true,
      },
    }),
  ],
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
    },
  },
  server: {
    port: 3000,
  },
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          'orderly-core': [
            '@orderly.network/hooks',
            '@orderly.network/types',
            '@orderly.network/utils',
          ],
          'orderly-ui': [
            '@orderly.network/ui',
            '@orderly.network/react-app',
          ],
          'orderly-trading': [
            '@orderly.network/trading',
          ],
          'orderly-portfolio': [
            '@orderly.network/portfolio',
          ],
        },
      },
    },
  },
});
```

---

## 11. High-Level Page Widgets

### Available Page Widgets

| Package | Component | Description |
|---------|-----------|-------------|
| `@orderly.network/trading` | `TradingPage` | Full trading interface with orderbook, chart, order entry |
| `@orderly.network/portfolio` | `Portfolio` | Portfolio overview with positions, orders, history |
| `@orderly.network/markets` | `MarketsHomePage` | Markets listing with prices, volumes |
| `@orderly.network/trading-leaderboard` | `LeaderboardPage` | Trading leaderboard |
| `@orderly.network/trading-rewards` | `TradingRewardsPage` | Trading rewards/affiliate program |
| `@orderly.network/vaults` | `VaultsPage` | Vault/earn products |

### TradingPage Props

```tsx
interface TradingPageProps {
  symbol: string;
  onSymbolChange?: (symbol: API.Symbol) => void;
  tradingViewConfig?: {
    scriptSRC: string;
    library_path: string;
    customCssUrl?: string;
    overrides?: Record<string, any>;
  };
  sharePnLConfig?: {
    backgroundImages?: string[];
    color?: "brand" | "profit" | "loss";
  };
}
```

### Portfolio Modules

```tsx
import { 
  OverviewModule,
  PositionsModule,
  OrdersModule,
  AssetsModule,
  HistoryModule,
} from "@orderly.network/portfolio";

// Full overview page
<OverviewModule.OverviewPage />

// Individual sections
<PositionsModule.PositionsPage />
<OrdersModule.OrdersPage />
<AssetsModule.AssetsPage />
<HistoryModule.HistoryPage />
```

---

## 12. Custom Layout with Scaffold

```tsx
import { 
  Scaffold,
  MainNavWidget,
  BottomNavWidget,
  AccountMenuWidget,
  ChainMenuWidget,
} from "@orderly.network/ui-scaffold";

function CustomLayout({ children }) {
  return (
    <Scaffold
      header={
        <MainNavWidget
          logo={<Logo />}
          items={navigationItems}
          trailing={
            <>
              <ChainMenuWidget />
              <AccountMenuWidget />
            </>
          }
        />
      }
      footer={<BottomNavWidget items={bottomNavItems} />}
    >
      {children}
    </Scaffold>
  );
}
```

---

## 13. Environment Variables

### .env (Build-time)

```bash
# Orderly Configuration
VITE_ORDERLY_BROKER_ID=your_broker_id
VITE_ORDERLY_BROKER_NAME=Your DEX

# Network
VITE_DEFAULT_NETWORK=mainnet

# Wallet Connect
VITE_WALLETCONNECT_PROJECT_ID=your_project_id

# Features
VITE_ENABLE_TRADING_VIEW=true
```

### Build vs Runtime Config

| Config Type | When to Use |
|-------------|-------------|
| Build-time (.env) | Static values, API keys (during development) |
| Runtime (config.js) | Deployment-specific values, multi-tenant deployments |

---

## 14. Deployment

### Build Command

```bash
npm run build
```

### Docker Deployment

```dockerfile
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf

# Runtime config is mounted at deployment
# COPY config.js /usr/share/nginx/html/config.js

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

### nginx.conf

```nginx
server {
    listen 80;
    root /usr/share/nginx/html;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    # Cache static assets
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    # Don't cache config.js (runtime config)
    location = /config.js {
        expires -1;
        add_header Cache-Control "no-store, no-cache, must-revalidate";
    }
}
```

---

## 15. Complete File Examples

### index.html

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <link rel="icon" type="image/webp" href="/favicon.webp" />
    <title>My DEX</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>
```

### ErrorBoundary.tsx

```tsx
import { Component, ErrorInfo, ReactNode } from "react";
import { Button, Text, Flex } from "@orderly.network/ui";

interface Props {
  children: ReactNode;
}

interface State {
  hasError: boolean;
  error?: Error;
}

export class ErrorBoundary extends Component<Props, State> {
  state: State = { hasError: false };

  static getDerivedStateFromError(error: Error): State {
    return { hasError: true, error };
  }

  componentDidCatch(error: Error, errorInfo: ErrorInfo) {
    console.error("Error caught:", error, errorInfo);
  }

  render() {
    if (this.state.hasError) {
      return (
        <Flex direction="column" align="center" justify="center" className="h-screen">
          <Text size="xl" weight="bold">Something went wrong</Text>
          <Text color="neutral">{this.state.error?.message}</Text>
          <Button onClick={() => window.location.reload()}>
            Reload Page
          </Button>
        </Flex>
      );
    }

    return this.props.children;
  }
}
```

### LoadingSpinner.tsx

```tsx
import { Spinner, Flex } from "@orderly.network/ui";

export function LoadingSpinner() {
  return (
    <Flex align="center" justify="center" className="h-screen w-full">
      <Spinner size="lg" />
    </Flex>
  );
}
```

---

## Best Practices

### 1. Use Lazy Loading for Pages
```tsx
const TradingPage = lazy(() => import('./pages/perp/Symbol'));
```

### 2. Separate Runtime from Build Config
```tsx
// Runtime config can be changed without rebuild
const brokerId = getRuntimeConfig('VITE_ORDERLY_BROKER_ID');
```

### 3. Code Split by Feature
```ts
// vite.config.ts
manualChunks: {
  'orderly-trading': ['@orderly.network/trading'],
  'orderly-portfolio': ['@orderly.network/portfolio'],
}
```

### 4. Handle Loading States
```tsx
<Suspense fallback={<LoadingSpinner />}>
  <TradingPage />
</Suspense>
```

### 5. Use Error Boundaries
```tsx
<ErrorBoundary>
  <OrderlyProvider>
    <App />
  </OrderlyProvider>
</ErrorBoundary>
```

### 6. Persist User Preferences
```tsx
const symbol = localStorage.getItem("lastSymbol") || "PERP_ETH_USDC";
const networkId = localStorage.getItem("networkId") || "mainnet";
```

### 7. Support SEO (if needed)
```tsx
import { Helmet } from "react-helmet-async";

<Helmet>
  <title>{`${symbol} Trading | My DEX`}</title>
  <meta name="description" content="Trade perpetual futures" />
</Helmet>
```

---

## Checklist for Production

- [ ] Broker ID configured
- [ ] WalletConnect project ID (for mobile wallets)
- [ ] Runtime config.js deployed
- [ ] TradingView library (if using charts)
- [ ] Custom branding (logo, favicon, colors)
- [ ] i18n translations (if multi-language)
- [ ] Error tracking (Sentry, etc.)
- [ ] Analytics integration
- [ ] SSL/HTTPS enabled
- [ ] CORS configured for API
- [ ] Rate limiting (if custom backend)
