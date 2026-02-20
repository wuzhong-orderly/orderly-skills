---
name: orderly-sdk-theming
description: Customize the visual appearance of your Orderly DEX with CSS variables, colors, fonts, logos, and component overrides.
argument-hint: <customization-type>
---

# Orderly SDK Theming & Branding

Customize the look and feel of your DEX using CSS variables, the theme provider, and component overrides.

---

## Overview

Orderly UI uses a CSS variable-based theming system with Tailwind CSS integration:

1. **CSS Variables** - Define colors, spacing, border radius
2. **Tailwind Preset** - Orderly's custom Tailwind configuration
3. **OrderlyThemeProvider** - Component-level overrides
4. **Assets** - Logo, favicon, PnL backgrounds

---

## 1. CSS Variables

All UI components use CSS variables prefixed with `--oui-`. Override them in your CSS to customize the theme.

### Create Theme File

```css
/* src/styles/theme.css */

:root {
  /* ============================================
     BRAND COLORS
     ============================================ */
  
  /* Primary - main accent color (buttons, links, highlights) */
  --oui-color-primary: 99 102 241;           /* Indigo: rgb(99, 102, 241) */
  --oui-color-primary-light: 165 180 252;
  --oui-color-primary-darken: 79 70 229;
  --oui-color-primary-contrast: 255 255 255;

  /* Link colors */
  --oui-color-link: 99 102 241;
  --oui-color-link-light: 165 180 252;

  /* ============================================
     SEMANTIC COLORS
     ============================================ */
  
  /* Success - profit, positive actions */
  --oui-color-success: 34 197 94;            /* Green: rgb(34, 197, 94) */
  --oui-color-success-light: 134 239 172;
  --oui-color-success-darken: 22 163 74;
  --oui-color-success-contrast: 255 255 255;

  /* Danger - loss, destructive actions, errors */
  --oui-color-danger: 239 68 68;             /* Red: rgb(239, 68, 68) */
  --oui-color-danger-light: 252 165 165;
  --oui-color-danger-darken: 220 38 38;
  --oui-color-danger-contrast: 255 255 255;

  /* Warning - alerts, caution */
  --oui-color-warning: 245 158 11;           /* Amber: rgb(245, 158, 11) */
  --oui-color-warning-light: 252 211 77;
  --oui-color-warning-darken: 217 119 6;
  --oui-color-warning-contrast: 255 255 255;

  /* ============================================
     TRADING COLORS (orderbook, PnL)
     ============================================ */
  
  --oui-color-trading-profit: 34 197 94;
  --oui-color-trading-profit-contrast: 255 255 255;
  --oui-color-trading-loss: 239 68 68;
  --oui-color-trading-loss-contrast: 255 255 255;

  /* ============================================
     BACKGROUND SCALE (dark theme: 1=lightest, 10=darkest)
     ============================================ */
  
  --oui-color-base-1: 100 116 139;   /* Lightest - disabled text */
  --oui-color-base-2: 71 85 105;
  --oui-color-base-3: 51 65 85;
  --oui-color-base-4: 45 55 72;      /* Borders, dividers */
  --oui-color-base-5: 39 49 66;
  --oui-color-base-6: 30 41 59;      /* Card backgrounds */
  --oui-color-base-7: 24 33 47;
  --oui-color-base-8: 18 26 38;      /* Sidebar background */
  --oui-color-base-9: 15 23 42;      /* Main background */
  --oui-color-base-10: 10 15 30;     /* Darkest - page background */

  /* Foreground (text on base backgrounds) */
  --oui-color-base-foreground: 255 255 255;
  
  /* Line/border color */
  --oui-color-line: 255 255 255;

  /* Fill colors (inputs, buttons) */
  --oui-color-fill: 30 41 59;
  --oui-color-fill-active: 39 49 66;

  /* ============================================
     GRADIENTS
     ============================================ */
  
  --oui-gradient-primary-start: 79 70 229;
  --oui-gradient-primary-end: 139 92 246;

  --oui-gradient-success-start: 22 101 52;
  --oui-gradient-success-end: 34 197 94;

  --oui-gradient-danger-start: 153 27 27;
  --oui-gradient-danger-end: 239 68 68;

  --oui-gradient-brand-start: 199 210 254;
  --oui-gradient-brand-end: 129 140 248;
  --oui-gradient-brand-angle: 135deg;
  --oui-gradient-brand-stop-start: 0%;
  --oui-gradient-brand-stop-end: 100%;

  /* ============================================
     TYPOGRAPHY
     ============================================ */
  
  --oui-font-family: 'Inter', 'SF Pro Display', -apple-system, BlinkMacSystemFont, sans-serif;

  /* ============================================
     BORDER RADIUS
     ============================================ */
  
  --oui-rounded-sm: 2px;
  --oui-rounded: 4px;
  --oui-rounded-md: 6px;
  --oui-rounded-lg: 8px;
  --oui-rounded-xl: 12px;
  --oui-rounded-2xl: 16px;
  --oui-rounded-full: 9999px;

  /* ============================================
     SPACING (for layout components)
     ============================================ */
  
  --oui-spacing-xs: 20rem;
  --oui-spacing-sm: 22.5rem;
  --oui-spacing-md: 26.25rem;
  --oui-spacing-lg: 30rem;
  --oui-spacing-xl: 33.75rem;
}
```

### Import Theme

```css
/* src/styles/index.css */
@import "@orderly.network/ui/dist/styles.css";
@import "./theme.css";
@import "./fonts.css";

@tailwind base;
@tailwind components;
@tailwind utilities;
```

> **Note**: CSS variables use space-separated RGB values (e.g., `99 102 241`) not `rgb()` syntax. This allows Tailwind to apply opacity modifiers.

---

## 2. Color Variable Reference

### Brand Colors

| Variable | Usage |
|----------|-------|
| `--oui-color-primary` | Primary buttons, active states, accents |
| `--oui-color-primary-light` | Hover states, secondary highlights |
| `--oui-color-primary-darken` | Active/pressed states |
| `--oui-color-primary-contrast` | Text on primary backgrounds |
| `--oui-color-link` | Link text color |
| `--oui-color-secondary` | Secondary text |
| `--oui-color-tertiary` | Tertiary/muted text |

### Semantic Colors

| Variable | Usage |
|----------|-------|
| `--oui-color-success` | Profit, positive values, success messages |
| `--oui-color-danger` | Loss, negative values, errors, sell buttons |
| `--oui-color-warning` | Warnings, alerts, caution states |

### Trading Colors

| Variable | Usage |
|----------|-------|
| `--oui-color-trading-profit` | Green for profits in orderbook, PnL |
| `--oui-color-trading-loss` | Red for losses in orderbook, PnL |

### Background Scale

The base colors form a gradient from light to dark:

```
base-1 (lightest) → base-10 (darkest)
```

| Variable | Usage |
|----------|-------|
| `--oui-color-base-1` to `base-3` | Muted/disabled text |
| `--oui-color-base-4` to `base-5` | Borders, dividers |
| `--oui-color-base-6` to `base-7` | Card/panel backgrounds |
| `--oui-color-base-8` to `base-9` | Page backgrounds |
| `--oui-color-base-10` | Darkest background |

---

## 3. Custom Fonts

### Add Custom Font

```css
/* src/styles/fonts.css */

/* Import from Google Fonts or local files */
@import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&display=swap');

/* Or use local font files */
@font-face {
  font-family: 'CustomFont';
  src: url('/fonts/CustomFont-Regular.woff2') format('woff2');
  font-weight: 400;
  font-style: normal;
  font-display: swap;
}

@font-face {
  font-family: 'CustomFont';
  src: url('/fonts/CustomFont-Medium.woff2') format('woff2');
  font-weight: 500;
  font-style: normal;
  font-display: swap;
}

@font-face {
  font-family: 'CustomFont';
  src: url('/fonts/CustomFont-SemiBold.woff2') format('woff2');
  font-weight: 600;
  font-style: normal;
  font-display: swap;
}

@font-face {
  font-family: 'CustomFont';
  src: url('/fonts/CustomFont-Bold.woff2') format('woff2');
  font-weight: 700;
  font-style: normal;
  font-display: swap;
}
```

### Apply Font

```css
/* In theme.css */
:root {
  --oui-font-family: 'CustomFont', 'Inter', sans-serif;
}
```

---

## 4. Logo & Assets

### Configure App Icons

```tsx
// In OrderlyAppProvider
import { OrderlyAppProvider } from "@orderly.network/react-app";

<OrderlyAppProvider
  brokerId="your_broker_id"
  brokerName="Your DEX"
  networkId="mainnet"
  appIcons={{
    // Main logo (header, navigation)
    main: {
      img: "/logo.svg",        // Or "/logo.png"
      component: CustomLogo,    // Or use a React component
    },
    // Secondary logo (smaller contexts)
    secondary: {
      img: "/logo-small.svg",
    },
  }}
>
  {children}
</OrderlyAppProvider>
```

### Logo Component

```tsx
// Custom logo component
function CustomLogo() {
  return (
    <img 
      src="/logo.svg" 
      alt="My DEX" 
      className="h-8 w-auto"
    />
  );
}
```

### Favicon

Place favicon files in `/public/`:

```
public/
├── favicon.ico
├── favicon.svg
├── favicon-16x16.png
├── favicon-32x32.png
├── apple-touch-icon.png
└── site.webmanifest
```

Update `index.html`:

```html
<head>
  <link rel="icon" type="image/svg+xml" href="/favicon.svg" />
  <link rel="icon" type="image/x-icon" href="/favicon.ico" />
  <link rel="apple-touch-icon" sizes="180x180" href="/apple-touch-icon.png" />
</head>
```

---

## 5. PnL Share Backgrounds

Customize the backgrounds for the PnL sharing feature:

### Add Background Images

```
public/
├── pnl/
│   ├── profit-bg-1.png
│   ├── profit-bg-2.png
│   ├── loss-bg-1.png
│   └── neutral-bg.png
```

### Configure in TradingPage

```tsx
<TradingPage
  symbol={symbol}
  tradingViewConfig={config}
  sharePnLConfig={{
    backgroundImages: [
      "/pnl/profit-bg-1.png",
      "/pnl/profit-bg-2.png",
      "/pnl/loss-bg-1.png",
    ],
    // Text colors on the share card
    color: "rgba(255, 255, 255, 0.98)",
    profitColor: "rgb(34, 197, 94)",
    lossColor: "rgb(239, 68, 68)",
    brandColor: "rgb(99, 102, 241)",
  }}
/>
```

---

## 6. TradingView Chart Colors

Customize the TradingView chart to match your theme:

```tsx
<TradingPage
  symbol={symbol}
  tradingViewConfig={{
    scriptSRC: "/tradingview/charting_library/charting_library.js",
    library_path: "/tradingview/charting_library/",
    colorConfig: {
      chartBG: "#0f172a",           // Chart background
      upColor: "#22c55e",           // Bullish candles
      downColor: "#ef4444",         // Bearish candles
      pnlUpColor: "#22c55e",        // Profit color in chart
      pnlDownColor: "#ef4444",      // Loss color in chart
      pnlZoreColor: "#64748b",      // Neutral color
      textColor: "#e2e8f0",         // Text on chart
      qtyTextColor: "#94a3b8",      // Quantity labels
      font: "Inter",                // Chart font
    },
    // TradingView overrides
    overrides: {
      "paneProperties.background": "#0f172a",
      "paneProperties.backgroundType": "solid",
      "scalesProperties.textColor": "#e2e8f0",
      "scalesProperties.lineColor": "#1e293b",
    },
  }}
/>
```

---

## 7. Component Overrides

Use `OrderlyThemeProvider` for component-level customization:

```tsx
import { OrderlyThemeProvider } from "@orderly.network/ui";

<OrderlyThemeProvider
  overrides={{
    // Override button styles
    button: {
      primary: {
        className: "custom-primary-button",
      },
    },
    // Override input styles
    input: {
      className: "custom-input",
    },
  }}
>
  {children}
</OrderlyThemeProvider>
```

---

## 8. Tailwind Configuration

### tailwind.config.ts

```ts
import type { Config } from "tailwindcss";
import { OUITailwind } from "@orderly.network/ui";

export default {
  content: [
    "./index.html",
    "./src/**/*.{js,ts,jsx,tsx}",
    "./node_modules/@orderly.network/**/*.{js,mjs}",
  ],
  // Use Orderly's preset
  presets: [OUITailwind.preset],
  theme: {
    extend: {
      // Add custom colors that work alongside OUI
      colors: {
        brand: {
          50: "#eef2ff",
          100: "#e0e7ff",
          500: "#6366f1",
          600: "#4f46e5",
          700: "#4338ca",
        },
      },
      fontFamily: {
        sans: ["Inter", "sans-serif"],
      },
    },
  },
  plugins: [],
} satisfies Config;
```

---

## 9. Dark/Light Mode

Currently, Orderly UI is designed for dark mode. To implement light mode, override the base colors:

```css
/* Light mode theme */
:root.light, [data-theme="light"] {
  --oui-color-base-1: 100 116 139;
  --oui-color-base-2: 148 163 184;
  --oui-color-base-3: 203 213 225;
  --oui-color-base-4: 226 232 240;
  --oui-color-base-5: 241 245 249;
  --oui-color-base-6: 248 250 252;
  --oui-color-base-7: 249 250 251;
  --oui-color-base-8: 255 255 255;
  --oui-color-base-9: 255 255 255;
  --oui-color-base-10: 255 255 255;

  --oui-color-base-foreground: 15 23 42;
  --oui-color-line: 15 23 42;
  --oui-color-fill: 241 245 249;
  --oui-color-fill-active: 226 232 240;
}
```

Toggle with JavaScript:

```tsx
function toggleTheme() {
  document.documentElement.classList.toggle("light");
}
```

---

## 10. Complete Theme Example

### Blue/Cyan Theme

```css
:root {
  --oui-font-family: 'Inter', sans-serif;

  /* Brand - Cyan */
  --oui-color-primary: 6 182 212;
  --oui-color-primary-light: 103 232 249;
  --oui-color-primary-darken: 8 145 178;
  --oui-color-primary-contrast: 255 255 255;

  --oui-color-link: 34 211 238;
  --oui-color-link-light: 103 232 249;

  /* Trading */
  --oui-color-trading-profit: 34 197 94;
  --oui-color-trading-loss: 239 68 68;

  /* Dark backgrounds */
  --oui-color-base-6: 17 24 39;
  --oui-color-base-7: 17 24 39;
  --oui-color-base-8: 10 15 25;
  --oui-color-base-9: 5 10 20;
  --oui-color-base-10: 0 5 15;

  /* Gradients */
  --oui-gradient-primary-start: 8 145 178;
  --oui-gradient-primary-end: 34 211 238;
}
```

### Purple Theme (Default Orderly)

```css
:root {
  --oui-color-primary: 176 132 233;
  --oui-color-primary-light: 213 190 244;
  --oui-color-primary-darken: 137 76 209;

  --oui-color-success: 41 233 169;
  --oui-color-danger: 245 97 139;
}
```

---

## Best Practices

1. **Use RGB values** - CSS variables use space-separated RGB (e.g., `99 102 241`)
2. **Import order matters** - Import Orderly styles first, then your overrides
3. **Test all states** - Verify hover, active, disabled states look correct
4. **Check accessibility** - Ensure sufficient color contrast (WCAG 2.1)
5. **Test on mobile** - Some components have different mobile styling
6. **Match TradingView** - Keep chart colors consistent with your theme
