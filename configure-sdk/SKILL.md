---
name: configure-sdk
description: Configure the Orderly SDK application settings: Broker ID, Network, Theme, Localization, and Feature Flags.
argument-hint: <config-path>
---

# Configure SDK Skill

Use this skill to update the application's configuration. This includes setting the broker ID (from `.env`), switching networks (testnet/mainnet), and applying themes.

## 1. Environment Variables (.env)
Set the foundational settings.

```bash
# .env.local
VITE_BROKER_ID="<your_broker_id>"  # Get from Orderly Dashboard
VITE_NETWORK="testnet"             # "mainnet" or "testnet"
VITE_APP_NAME="My DEX"
```

## 2. Config Provider (src/orderly/config.ts)
Manage the application theme and icons.

```ts
import { ConfigProvider } from "@orderly.network/react-app";

export const theme = {
  colors: {
    primary: "#00ff00",
    secondary: "#0000ff",
  },
  typography: {
    fontFamily: "Inter, sans-serif",
  },
};

export const appIcons = {
  logo: "/logo.svg",
  favicon: "/favicon.ico",
  pnlShare: {
    backgrounds: ["/bg1.png", "/bg2.png"],
  },
};
```

## 3. Localization (i18n)
Use `@orderly.network/i18n` to handle translations.

```ts
import { LocaleProvider, defaultLanguages } from "@orderly.network/i18n";

// In your App or Provider component:
<LocaleProvider
  locale="en"
  languages={defaultLanguages}
  onLanguageChanged={(lang) => console.log(lang)}
>
  {children}
</LocaleProvider>
```

## 4. Wallet Connector Config
Configure supported wallets (Metamask, Rabby, etc.) via `@orderly.network/wallet-connector` or `@orderly.network/wallet-connector-privy`.

```ts
import { WalletConnector } from "@orderly.network/wallet-connector";

<WalletConnector
  networkId="testnet"
  customWallets={/* custom config */}
>
  {children}
</WalletConnector>
```
