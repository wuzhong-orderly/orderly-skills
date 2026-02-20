# Orderly SDK Wallet Connection

A comprehensive guide to integrating wallet connection in Orderly Network DEX applications, supporting both EVM (Ethereum, Arbitrum, etc.) and Solana wallets.

## Overview

Orderly Network supports **omnichain trading**, meaning users can connect wallets from multiple blockchain ecosystems:

- **EVM Chains**: Ethereum, Arbitrum, Optimism, Base, Polygon, BSC, Avalanche, etc.
- **Solana**: Mainnet and Devnet

The SDK provides a unified wallet connection layer that abstracts the differences between these ecosystems.

## Wallet Connector Package

> **CRITICAL**: The `@orderly.network/wallet-connector` package is a wrapper. You MUST also install the underlying wallet libraries for it to work.

```bash
# Main connector package
npm install @orderly.network/wallet-connector

# REQUIRED: EVM wallet packages (for MetaMask, Coinbase, WalletConnect, etc.)
npm install @web3-onboard/injected-wallets @web3-onboard/walletconnect

# REQUIRED: Solana wallet packages (for Phantom, Solflare, etc.)
npm install @solana/wallet-adapter-base @solana/wallet-adapter-wallets
```

### Required Dependencies Summary

| Package | Purpose | Required For |
|---------|---------|--------------|
| `@web3-onboard/injected-wallets` | MetaMask, Coinbase Wallet, Rabby, etc. | EVM wallet connection |
| `@web3-onboard/walletconnect` | WalletConnect protocol | Mobile & multi-platform wallets |
| `@solana/wallet-adapter-base` | Solana wallet adapter base | All Solana wallets |
| `@solana/wallet-adapter-wallets` | Phantom, Solflare, Ledger adapters | Solana wallet connection |

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    WalletConnectorProvider                  │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                   SolanaProvider                     │   │
│  │  (ConnectionProvider + WalletProvider + ModalProvider)│   │
│  │  ┌───────────────────────────────────────────────┐  │   │
│  │  │                   InitEvm                      │  │   │
│  │  │  (Web3OnboardProvider for EVM wallets)        │  │   │
│  │  │  ┌─────────────────────────────────────────┐ │  │   │
│  │  │  │                  Main                    │ │  │   │
│  │  │  │  (WalletConnectorContext - unified API) │ │  │   │
│  │  │  │  ┌─────────────────────────────────┐   │ │  │   │
│  │  │  │  │         Your App                │   │ │  │   │
│  │  │  │  └─────────────────────────────────┘   │ │  │   │
│  │  │  └─────────────────────────────────────────┘ │  │   │
│  │  └───────────────────────────────────────────────┘  │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

---

## Basic Setup

**IMPORTANT**: The `networkId` must be consistent between `WalletConnectorProvider` (Solana network) and `OrderlyAppProvider`. When `networkId="mainnet"`, use `WalletAdapterNetwork.Mainnet`. When `networkId="testnet"`, use `WalletAdapterNetwork.Devnet`.

### 1. WalletConnectorProvider

Wrap your app with the `WalletConnectorProvider`:

```tsx
import { WalletConnectorProvider } from "@orderly.network/wallet-connector";
import { WalletAdapterNetwork } from "@solana/wallet-adapter-base";
import { OrderlyAppProvider } from "@orderly.network/react-app";
import type { NetworkId } from "@orderly.network/types";

function App() {
  // REQUIRED: Set network - "mainnet" for production, "testnet" for development
  const networkId: NetworkId = "mainnet";
  
  return (
    <WalletConnectorProvider
      solanaInitial={{
        // Solana network MUST match networkId
        network: networkId === "mainnet" 
          ? WalletAdapterNetwork.Mainnet 
          : WalletAdapterNetwork.Devnet,
        wallets: getSolanaWallets(),
      }}
      evmInitial={{
        options: {
          wallets: getEvmWallets(),
          appMetadata: {
            name: "My DEX",
            description: "Decentralized Exchange",
          },
        },
      }}
    >
      <OrderlyAppProvider
        brokerId="your_broker_id"    // REQUIRED
        brokerName="Your DEX Name"   // REQUIRED
        networkId={networkId}         // REQUIRED: "mainnet" or "testnet"
      >
        <YourApp />
      </OrderlyAppProvider>
    </WalletConnectorProvider>
  );
}
```

### 2. Configure EVM Wallets

```tsx
import injectedOnboard from "@web3-onboard/injected-wallets";
import walletConnectOnboard from "@web3-onboard/walletconnect";
import binanceWallet from "@binance/w3w-blocknative-connector";

export function getEvmWallets() {
  const walletConnectProjectId = "YOUR_WALLETCONNECT_PROJECT_ID";

  return [
    // Injected wallets (MetaMask, Rabby, Coinbase, etc.)
    injectedOnboard(),
    
    // Binance Web3 Wallet
    binanceWallet({ options: { lng: "en" } }),
    
    // WalletConnect (for mobile wallets)
    walletConnectOnboard({
      projectId: walletConnectProjectId,
      qrModalOptions: {
        themeMode: "dark",
      },
      dappUrl: window.location.origin,
    }),
  ];
}
```

### 3. Configure Solana Wallets

```tsx
import { WalletAdapterNetwork } from "@solana/wallet-adapter-base";
import {
  PhantomWalletAdapter,
  SolflareWalletAdapter,
  LedgerWalletAdapter,
} from "@solana/wallet-adapter-wallets";
import {
  SolanaMobileWalletAdapter,
  createDefaultAddressSelector,
  createDefaultAuthorizationResultCache,
} from "@solana-mobile/wallet-adapter-mobile";

export function getSolanaWallets(networkId: "mainnet" | "testnet") {
  return [
    new PhantomWalletAdapter(),
    new SolflareWalletAdapter(),
    new LedgerWalletAdapter(),
    new SolanaMobileWalletAdapter({
      addressSelector: createDefaultAddressSelector(),
      appIdentity: {
        uri: `${location.protocol}//${location.host}`,
      },
      authorizationResultCache: createDefaultAuthorizationResultCache(),
      chain: networkId === "mainnet" 
        ? WalletAdapterNetwork.Mainnet 
        : WalletAdapterNetwork.Devnet,
    }),
  ];
}
```

---

## Using Wallet Connection

### Access Wallet Context

The SDK provides a unified `WalletConnectorContext` that works with both EVM and Solana:

```tsx
import { useContext } from "react";
import { WalletConnectorContext } from "@orderly.network/hooks";

function WalletStatus() {
  const {
    connect,        // Connect wallet function
    disconnect,     // Disconnect wallet function
    connecting,     // Boolean: connection in progress
    wallet,         // Connected wallet info
    connectedChain, // Current chain info
    setChain,       // Switch chain function
    namespace,      // "evm" | "solana" | null
  } = useContext(WalletConnectorContext);

  return (
    <div>
      {wallet ? (
        <>
          <p>Connected: {wallet.accounts[0].address}</p>
          <p>Chain: {connectedChain?.id}</p>
          <button onClick={disconnect}>Disconnect</button>
        </>
      ) : (
        <button onClick={() => connect({ chainId: 42161 })}>
          Connect Wallet
        </button>
      )}
    </div>
  );
}
```

### Connect to Specific Chain

```tsx
const { connect } = useContext(WalletConnectorContext);

// Connect to EVM chain (Arbitrum)
await connect({ chainId: 42161 });

// Connect to Solana
await connect({ chainId: 900900900 }); // Solana mainnet chain ID
await connect({ chainId: 901901901 }); // Solana devnet chain ID
```

### Switch Chains

```tsx
const { setChain, connectedChain } = useContext(WalletConnectorContext);

// Switch to Optimism
await setChain({ chainId: "0xa" }); // Hex format for EVM

// Switch to Base
await setChain({ chainId: "0x2105" });
```

---

## Account State Machine

After wallet connection, users need to complete Orderly account setup. The account has several states:

```
┌─────────────────┐
│  NotConnected   │ ← Wallet not connected
└────────┬────────┘
         │ connect()
         ▼
┌─────────────────┐
│    Connected    │ ← Wallet connected, no Orderly account
└────────┬────────┘
         │ createAccount()
         ▼
┌─────────────────┐
│  NotSignedIn    │ ← Orderly account exists, needs trading key
└────────┬────────┘
         │ createOrderlyKey()
         ▼
┌─────────────────┐
│    SignedIn     │ ← Fully authenticated, can trade
└─────────────────┘
```

### Using useAccount Hook

```tsx
import { useAccount, AccountStatusEnum } from "@orderly.network/hooks";

function AccountStatus() {
  const {
    account,           // Account instance
    state,             // Current account state
    createOrderlyKey,  // Create trading key
    createAccount,     // Create Orderly account
    disconnect,        // Disconnect wallet
  } = useAccount();

  switch (state.status) {
    case AccountStatusEnum.NotConnected:
      return <ConnectWalletButton />;
      
    case AccountStatusEnum.Connected:
      return (
        <button onClick={() => createAccount()}>
          Create Orderly Account
        </button>
      );
      
    case AccountStatusEnum.NotSignedIn:
      return (
        <button onClick={() => createOrderlyKey()}>
          Enable Trading
        </button>
      );
      
    case AccountStatusEnum.SignedIn:
      return <TradingInterface />;
  }
}
```

---

## UI Components for Wallet Connection

### WalletConnectorWidget

Pre-built wallet connection UI:

```tsx
import { WalletConnectorWidget, WalletConnectorModalId } from "@orderly.network/ui-connector";
import { modal } from "@orderly.network/ui";

// Show wallet connect modal
function ConnectButton() {
  return (
    <button onClick={() => modal.show(WalletConnectorModalId)}>
      Connect Wallet
    </button>
  );
}

// Or use widget directly
<WalletConnectorWidget />
```

### AuthGuard

Wrap content that requires authentication:

```tsx
import { AuthGuard } from "@orderly.network/ui-connector";

function TradingPage() {
  return (
    <AuthGuard fallback={<ConnectPrompt />}>
      <OrderEntry symbol="PERP_ETH_USDC" />
    </AuthGuard>
  );
}
```

### useAuthGuard Hook

```tsx
import { useAuthGuard } from "@orderly.network/ui-connector";

function TradeButton() {
  const { 
    isAuthenticated, 
    triggerAuth // Shows connect modal if needed
  } = useAuthGuard();

  const handleClick = () => {
    if (!isAuthenticated) {
      triggerAuth();
      return;
    }
    // Proceed with trade
  };

  return <button onClick={handleClick}>Trade</button>;
}
```

### useAuthStatus Hook

```tsx
import { useAuthStatus, AuthStatusEnum } from "@orderly.network/ui-connector";

function StatusIndicator() {
  const status = useAuthStatus();

  switch (status) {
    case AuthStatusEnum.NotConnected:
      return <Badge>Not Connected</Badge>;
    case AuthStatusEnum.Connected:
      return <Badge>Connected</Badge>;
    case AuthStatusEnum.SignedIn:
      return <Badge color="success">Ready to Trade</Badge>;
  }
}
```

---

## Chain Selector

### ChainSelectorWidget

```tsx
import { ChainSelectorWidget, ChainSelectorDialogId } from "@orderly.network/ui-chain-selector";
import { modal } from "@orderly.network/ui";

// Show chain selector
<button onClick={() => modal.show(ChainSelectorDialogId)}>
  Select Chain
</button>
```

### useChains Hook

```tsx
import { useChains } from "@orderly.network/hooks";

function ChainList() {
  const [chains, { isLoading }] = useChains();

  return (
    <ul>
      {chains.map(chain => (
        <li key={chain.chain_id}>
          {chain.name} (ID: {chain.chain_id})
        </li>
      ))}
    </ul>
  );
}
```

---

## Supported Chains

### EVM Mainnet Chains

| Chain | Chain ID | Network |
|-------|----------|---------|
| Ethereum | 1 | mainnet |
| Arbitrum | 42161 | mainnet |
| Optimism | 10 | mainnet |
| Base | 8453 | mainnet |
| Polygon | 137 | mainnet |
| BSC | 56 | mainnet |
| Avalanche | 43114 | mainnet |
| Mantle | 5000 | mainnet |
| SEI | 1329 | mainnet |

### EVM Testnet Chains

| Chain | Chain ID | Network |
|-------|----------|---------|
| Arbitrum Sepolia | 421614 | testnet |
| BSC Testnet | 97 | testnet |
| Monad Testnet | 10143 | testnet |

### Solana

| Network | Chain ID (Internal) |
|---------|---------------------|
| Mainnet | 900900900 |
| Devnet | 901901901 |

---

## Chain Filtering

Restrict which chains users can connect to:

```tsx
<OrderlyAppProvider
  brokerId="your_broker_id"
  networkId="mainnet"
  chainFilter={{
    mainnet: [
      { id: 42161 }, // Arbitrum only
      { id: 10 },    // Optimism
    ],
    testnet: [
      { id: 421614 }, // Arbitrum Sepolia
    ],
  }}
>
```

---

## Handling Chain Changes

```tsx
<OrderlyAppProvider
  brokerId="your_broker_id"
  networkId="mainnet"
  onChainChanged={(chainId, { isTestnet }) => {
    console.log(`Switched to chain ${chainId}`);
    
    // Reload if switching between mainnet/testnet
    if (isTestnet && networkId === "mainnet") {
      localStorage.setItem("network", "testnet");
      window.location.reload();
    }
  }}
>
```

---

## Privy Integration (Alternative)

For social login support, use Privy:

```bash
npm install @orderly.network/wallet-connector-privy
```

```tsx
import { WalletConnectorPrivy } from "@orderly.network/wallet-connector-privy";

<WalletConnectorPrivy
  appId="YOUR_PRIVY_APP_ID"
  loginMethods={["email", "google", "twitter"]}
>
  <OrderlyAppProvider ...>
    <App />
  </OrderlyAppProvider>
</WalletConnectorPrivy>
```

---

## Complete Integration Example

```tsx
// src/providers/WalletProvider.tsx
import { ReactNode } from "react";
import { WalletConnectorProvider } from "@orderly.network/wallet-connector";
import { WalletAdapterNetwork } from "@solana/wallet-adapter-base";
import { PhantomWalletAdapter, SolflareWalletAdapter } from "@solana/wallet-adapter-wallets";
import injectedOnboard from "@web3-onboard/injected-wallets";
import walletConnectOnboard from "@web3-onboard/walletconnect";

interface Props {
  children: ReactNode;
  networkId: "mainnet" | "testnet";
}

export function WalletProvider({ children, networkId }: Props) {
  const solanaInitial = {
    network: networkId === "mainnet" 
      ? WalletAdapterNetwork.Mainnet 
      : WalletAdapterNetwork.Devnet,
    wallets: [
      new PhantomWalletAdapter(),
      new SolflareWalletAdapter(),
    ],
  };

  const evmInitial = {
    options: {
      wallets: [
        injectedOnboard(),
        walletConnectOnboard({
          projectId: process.env.WALLETCONNECT_PROJECT_ID!,
          qrModalOptions: { themeMode: "dark" },
        }),
      ],
      appMetadata: {
        name: "My DEX",
        description: "Decentralized Perpetual Exchange",
      },
    },
  };

  return (
    <WalletConnectorProvider
      solanaInitial={solanaInitial}
      evmInitial={evmInitial}
    >
      {children}
    </WalletConnectorProvider>
  );
}
```

```tsx
// src/App.tsx
import { WalletProvider } from "./providers/WalletProvider";
import { OrderlyAppProvider } from "@orderly.network/react-app";

export default function App() {
  return (
    <WalletProvider networkId="mainnet">
      <OrderlyAppProvider
        brokerId="your_broker_id"
        networkId="mainnet"
      >
        <Router>
          <Routes>
            <Route path="/trade/:symbol" element={<TradingPage />} />
          </Routes>
        </Router>
      </OrderlyAppProvider>
    </WalletProvider>
  );
}
```

---

## Error Handling

```tsx
import { useEventEmitter } from "@orderly.network/hooks";

function WalletErrorHandler() {
  const ee = useEventEmitter();

  useEffect(() => {
    const handleError = (error: { message: string }) => {
      toast.error(error.message);
    };

    ee.on("wallet:connect-error", handleError);
    
    return () => {
      ee.off("wallet:connect-error", handleError);
    };
  }, [ee]);

  return null;
}
```

---

## Best Practices

### 1. Check Wallet Connection Before Actions
```tsx
const { wallet } = useContext(WalletConnectorContext);
if (!wallet) {
  modal.show(WalletConnectorModalId);
  return;
}
```

### 2. Use AuthGuard for Protected Content
```tsx
<AuthGuard>
  <ProtectedTradingUI />
</AuthGuard>
```

### 3. Handle Network Mismatches
```tsx
const { connectedChain } = useContext(WalletConnectorContext);
const expectedChainId = networkId === "mainnet" ? 42161 : 421614;

if (connectedChain?.id !== expectedChainId) {
  return <SwitchNetworkPrompt />;
}
```

### 4. Persist Network Selection
```tsx
const NETWORK_KEY = "orderly_network_id";

function getNetworkId(): NetworkId {
  return localStorage.getItem(NETWORK_KEY) as NetworkId || "mainnet";
}

function setNetworkId(id: NetworkId) {
  localStorage.setItem(NETWORK_KEY, id);
}
```

### 5. Support Both EVM and Solana
```tsx
// Disable wallets via config if needed
const disableEVM = process.env.DISABLE_EVM === "true";
const disableSolana = process.env.DISABLE_SOLANA === "true";

<WalletConnectorProvider
  evmInitial={disableEVM ? undefined : evmConfig}
  solanaInitial={disableSolana ? undefined : solanaConfig}
>
```

---

## TypeScript Types

```tsx
import type { 
  ChainNamespace,  // "evm" | "solana"
  NetworkId,       // "mainnet" | "testnet"
} from "@orderly.network/types";

import type { WalletConnectorContextState } from "@orderly.network/hooks";
```

---

## Related Packages

| Package | Purpose |
|---------|---------|
| `@orderly.network/wallet-connector` | Core wallet connection |
| `@orderly.network/wallet-connector-privy` | Privy social login |
| `@orderly.network/ui-connector` | Wallet UI components |
| `@orderly.network/ui-chain-selector` | Chain selection UI |
| `@orderly.network/default-evm-adapter` | EVM wallet adapter |
| `@orderly.network/default-solana-adapter` | Solana wallet adapter |
