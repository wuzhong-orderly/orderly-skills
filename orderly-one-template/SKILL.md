# Orderly One — Template Customization

This skill covers direct customization of the forked DEX template repository — the actual frontend code that runs the deployed DEX. This is the "product" side, as opposed to the "factory" API covered by other orderly-one skills.

> **Note:** For most users, the Orderly One web portal at [https://dex.orderly.network/dex](https://dex.orderly.network/dex) handles all configuration without touching code. This skill is for developers who want to customize the template repo directly (e.g., after forking it, or when building a custom integration).

---

## Template Repository

The official Orderly One DEX template:
- **URL:** `https://github.com/OrderlyNetworkDexCreator/sample-8205` (or whichever template is configured)
- **Stack:** React + Vite + Orderly SDK
- **Deployment:** GitHub Pages via GitHub Actions

When a user creates a DEX through Orderly One (low-code path), the API forks this template and pushes configuration to it.

---

## Getting Started

### Clone & Install

```bash
git clone https://github.com/OrderlyNetworkDexCreator/sample-8205 my-orderly-dex
cd my-orderly-dex
npm install
```

### Run Locally

```bash
npm run dev
```

### Build for Production

```bash
npm run build
```

---

## Configuration: `public/config.js`

This is the primary configuration file. It's loaded at runtime (not build time), so changes don't require a rebuild.

### Essential Parameters

| Parameter | Description | Example |
|-----------|-------------|---------|
| `VITE_ORDERLY_BROKER_ID` | Your broker ID. Use `"demo"` for testing | `"demo"` |
| `VITE_ORDERLY_BROKER_NAME` | Display name of your DEX | `"My DEX"` |
| `VITE_BROKER_EOA_ADDRESS` | Broker EVM address (for profit sharing) | `"0x..."` |

### Feature Flags

| Parameter | Description |
|-----------|-------------|
| `VITE_DISABLE_MAINNET` | `"true"` to disable mainnet |
| `VITE_DISABLE_TESTNET` | `"true"` to disable testnet |
| `VITE_WALLETCONNECT_PROJECT_ID` | Required for WalletConnect. Get from [Reown](https://reown.com/) |
| `VITE_APP_NAME` | Browser title and meta tags |
| `VITE_APP_DESCRIPTION` | SEO description |

### Menu Configuration

| Parameter | Description |
|-----------|-------------|
| `VITE_ENABLED_MENUS` | Comma-separated. Options: `Trading`, `Portfolio`, `Markets`, `Leaderboard`, `Swap`, `Rewards`, `Vaults`, `Points` |
| `VITE_CUSTOM_MENUS` | External links: `"Name,URL;Name2,URL2"` |

### Social Links

| Parameter | Description |
|-----------|-------------|
| `VITE_TELEGRAM_URL` | Telegram community link |
| `VITE_DISCORD_URL` | Discord server link |
| `VITE_TWITTER_URL` | X/Twitter profile link |

### Advanced

| Parameter | Description |
|-----------|-------------|
| `VITE_RESTRICTED_REGIONS` | ISO codes (e.g., `"US,CN"`) for geo-blocking |
| `VITE_SEO_SITE_URL` | Canonical URL for SEO |
| `VITE_SEO_TWITTER_HANDLE` | `"@handle"` for Twitter cards |
| `VITE_SEO_KEYWORDS` | SEO keywords |
| `VITE_USE_CUSTOM_PNL_POSTERS` | `"true"` to enable custom P&L poster images in `public/pnl/` |

---

## Branding: Logos

The template checks flags in `config.js` for logo presence:

### Primary Logo (Header)

1. Set `VITE_HAS_PRIMARY_LOGO` to `"true"` in `config.js`
2. Replace `public/logo.webp`

### Secondary Logo (Mobile / Favicon area)

1. Set `VITE_HAS_SECONDARY_LOGO` to `"true"` in `config.js`
2. Replace `public/logo-secondary.webp`

### Favicon

Replace `public/favicon.ico`.

---

## Theming: `app/styles/theme.css`

The UI is themed via CSS variables. Edit `app/styles/theme.css` to customize.

> **Important:** The dex-creator template uses `app/styles/theme.css`. Other official templates (Vite, Next.js, Remix) may use `src/styles/theme.css` instead. Check your project structure.

### Color Format

Colors use **space-separated RGB values**: `176 132 233` (not `#B084E9`).

This format allows the SDK to apply opacity modifiers in code.

### Key Variables

**Brand Colors:**
```css
--oui-color-primary: 176 132 233;       /* Main brand color */
--oui-color-primary-light: 213 190 244;
--oui-color-primary-darken: 137 76 209;
--oui-color-link: 189 107 237;
```

**Trading Colors:**
```css
--oui-color-trading-profit: 41 233 169;  /* Green for buy/profit */
--oui-color-trading-loss: 245 97 139;    /* Red for sell/loss */
```

**Backgrounds:**
```css
--oui-color-fill: 36 32 47;             /* Main background */
--oui-color-base-1: 93 83 123;          /* Lightest UI element bg */
/* ... base-2 through base-9 get progressively darker ... */
--oui-color-base-10: 14 13 18;          /* Darkest */
```

**Gradients:**
```css
--oui-gradient-primary-start: 40 0 97;
--oui-gradient-primary-end: 189 107 237;
```

For the complete variable reference, see `orderly-one-theming`.

---

## How the API Configures the Template

When Orderly One creates or updates a DEX, it pushes these files to the forked repo:

1. **`public/config.js`** — All VITE_* parameters from the DEX configuration
2. **`app/styles/theme.css`** — The `themeCSS` field content
3. **`public/logo.webp`** — Primary logo (base64-decoded)
4. **`public/logo-secondary.webp`** — Secondary logo
5. **`public/favicon.ico`** — Favicon
6. **`public/pnl/`** — P&L poster images
7. **`.github/workflows/`** — Deployment workflow files
8. **Repository secret: `TEMPLATE_PAT`** — Encrypted deployment token

GitHub Actions then builds and deploys to GitHub Pages automatically.

---

## Custom Development

For developers going beyond configuration:

### Project Structure

```
my-orderly-dex/
├── public/
│   ├── config.js          # Runtime configuration
│   ├── logo.webp          # Primary logo
│   ├── logo-secondary.webp
│   ├── favicon.ico
│   └── pnl/               # P&L poster images
├── app/
│   ├── styles/
│   │   └── theme.css      # Theme CSS variables
│   ├── root.tsx            # App root
│   └── routes/             # Page routes
└── package.json
```

### Orderly SDK Packages Used

The template integrates these Orderly SDK packages:
- `@orderly.network/react-app` — Trading app components
- `@orderly.network/ui` — UI component library
- `@orderly.network/wallet-connector` — Wallet connection
- `@orderly.network/i18n` — Translations

For SDK customization, see the `orderly-sdk-*` family of skills.

---

## Agent Instructions

When helping users customize a DEX template:

1. **Always clone the template first** — don't start from scratch
2. **Check `public/config.js` immediately** — it's the source of truth for runtime config
3. **For color changes**: modify `app/styles/theme.css` using space-separated RGB values
4. **For menu changes**: update `VITE_ENABLED_MENUS` in `config.js`
5. **For logo changes**: replace files in `public/` and set the corresponding `VITE_HAS_*` flags
6. **Inform users** they can also use the Orderly One portal to configure everything visually

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| "Broker ID not found" | Set `VITE_ORDERLY_BROKER_ID` in `config.js`. Use `"demo"` for testing |
| Logo not showing | Set `VITE_HAS_PRIMARY_LOGO` to `"true"` and ensure `public/logo.webp` exists |
| Colors look wrong | Use space-separated RGB (`255 255 255`) not hex (`#FFFFFF`) in `theme.css` |
| Config changes not reflected | `config.js` is runtime — clear browser cache or hard refresh |

---

## Related Skills

- `orderly-one-general` — Overview, auth, chain IDs
- `orderly-one-create-dex` — Create/update DEX via API (pushes config to this template)
- `orderly-one-theming` — Full CSS variable reference, AI theme generation
- `orderly-one-graduation` — Get a real broker ID to replace `"demo"`
