---
name: install-dependency
description: Install Orderly SDK packages and related dependencies (hooks, UI, features, wallet connectors) using the preferred package manager.
argument-hint: <package-name>
---

# Install Dependency Skill

Use this skill to add Orderly SDK packages to your project. The SDK is modular, so you only install what you need.

## Package Groups

### 1. Core (Required)
The foundation of any Orderly dApp.
- `@orderly.network/react-app`: Context provider & main app logic.
- `@orderly.network/hooks`: Data fetching & state management hooks.
- `@orderly.network/types`: TypeScript definitions.
- `@orderly.network/ui`: Base UI components.

```bash
npm install @orderly.network/react-app @orderly.network/hooks @orderly.network/types @orderly.network/ui
```

### 2. Feature Widgets (High-Level)
Plug-and-play pages/widgets.
- `@orderly.network/trading`: Full trading interface (chart, orderbook, positions).
- `@orderly.network/portfolio`: Assets & history dashboard.
- `@orderly.network/markets`: Market list & stats.
- `@orderly.network/vaults`: Earn/Vaults page.
- `@orderly.network/affiliate`: Referral system.

```bash
npm install @orderly.network/trading @orderly.network/portfolio
```

### 3. Wallet Connectors
Choose one strategy.

**Option A: Standard (Web3Onboard / Wagmi)**
- `@orderly.network/wallet-connector`: Generic connector.
- `@web3-onboard/*` or `wagmi`: Your choice of library.

**Option B: Privy (Recommended for social login)**
- `@orderly.network/wallet-connector-privy`: Privy integration.
- `@privy-io/react-auth`: Core Privy library.

```bash
# Standard
npm install @orderly.network/wallet-connector

# Privy
npm install @orderly.network/wallet-connector-privy @privy-io/react-auth
```

### 4. Utilities
- `@orderly.network/perpetual`: Perp-specific logic (if building custom UI).
- `@orderly.network/core`: Low-level API client (rarely needed directly).

## Usage
Run the install command in your project root.

```bash
# Example: Install everything for a full DEX
npm install @orderly.network/react-app @orderly.network/trading @orderly.network/portfolio @orderly.network/markets @orderly.network/wallet-connector
```
