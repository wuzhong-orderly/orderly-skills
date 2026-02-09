# OrderlyOne DEX Creator Skill

This skill allows you to build a DEX using Orderly One, a no-code solution for launching an Orderly Network DEX. It uses a template repository that is pre-wired with the Orderly SDKs and supports extensive configuration via `public/config.js` and CSS variables.

## Prerequisites
- Node.js 18+
- npm or yarn
- Git

## Template Repository
URL: `https://github.com/OrderlyNetworkDexCreator/sample-8205`

## Usage

When asked to "create an Orderly One DEX" or "launch a no-code DEX", follow these steps.

### 1. Clone the Template

Clone the official Orderly One template. Do not start from scratch.

```bash
git clone https://github.com/OrderlyNetworkDexCreator/sample-8205 my-orderly-dex
cd my-orderly-dex
npm install
```

### 2. Configure the DEX

The DEX is configured entirely through `public/config.js`. This file is injected at runtime, allowing you to change settings without rebuilding.

#### Essential Configuration (`public/config.js`)

You **must** update these values in `public/config.js`:

| Parameter | Description | Example |
| :--- | :--- | :--- |
| `VITE_ORDERLY_BROKER_ID` | Your unique Broker ID. Use `demo` for testing. | `"demo"` |
| `VITE_ORDERLY_BROKER_NAME` | The display name of your DEX. | `"My DEX"` |
| `VITE_BROKER_EOA_ADDRESS` | The EVM address of the broker (for profit sharing). | `"0x..."` |

#### Feature Flags & UI Settings

| Parameter | Description |
| :--- | :--- |
| `VITE_DISABLE_MAINNET` | Set `"true"` to disable Mainnet (e.g. for testing). |
| `VITE_DISABLE_TESTNET` | Set `"true"` to disable Testnet. |
| `VITE_WALLETCONNECT_PROJECT_ID` | Required for WalletConnect support. Get one from Reown. |
| `VITE_APP_NAME` | Name used in browser title and meta tags. |
| `VITE_APP_DESCRIPTION` | SEO description. |
| `VITE_ENABLED_MENUS` | Comma-separated list of menus to show. Options: `Trading`, `Portfolio`, `Markets`, `Leaderboard`. |
| `VITE_CUSTOM_MENUS` | Add external links to the nav. Format: `Name,URL;Name2,URL2`. |
| `VITE_TELEGRAM_URL` | Link to your Telegram community. |
| `VITE_DISCORD_URL` | Link to your Discord server. |
| `VITE_TWITTER_URL` | Link to your X/Twitter profile. |

#### Advanced Configuration

- **Restricted Regions**: `VITE_RESTRICTED_REGIONS` (ISO codes, e.g., `"US,CN"`).
- **SEO**: `VITE_SEO_SITE_URL`, `VITE_SEO_TWITTER_HANDLE`, `VITE_SEO_KEYWORDS`.
- **Custom PnL Posters**: Set `VITE_USE_CUSTOM_PNL_POSTERS` to `"true"` and upload images to `public/pnl/`.

### 3. Customize Branding (Logos)

The template checks `VITE_HAS_PRIMARY_LOGO` and `VITE_HAS_SECONDARY_LOGO` in `config.js`.

1. **Primary Logo** (Header):
   - Set `VITE_HAS_PRIMARY_LOGO` to `"true"`.
   - Replace `public/logo.webp`.

2. **Secondary Logo** (Mobile/Favicon):
   - Set `VITE_HAS_SECONDARY_LOGO` to `"true"`.
   - Replace `public/logo-secondary.webp`.

3. **Favicon**: Replace `public/favicon.ico`.

### 4. Customize Theme (`app/styles/theme.css`)

The UI uses CSS variables for theming. Edit `app/styles/theme.css` to match your brand.

#### Key Variables

- **Brand Colors**:
  - `--oui-color-primary`: Main brand color (RGB format: `R G B`).
  - `--oui-color-primary-light`: Lighter shade.
  - `--oui-color-primary-darken`: Darker shade.
  - `--oui-color-link`: Color for links/accents.

- **Status Colors** (Trading):
  - `--oui-color-trading-profit`: Color for profit/buy (Green).
  - `--oui-color-trading-loss`: Color for loss/sell (Red).

- **Backgrounds**:
  - `--oui-color-fill`: Main background color.
  - `--oui-color-base-1` to `--oui-color-base-10`: Grayscale palette for UI elements (cards, inputs).

- **Gradients**:
  - `--oui-gradient-primary-start` / `-end`: Main gradient used in buttons/banners.

**Note:** Colors are often defined as space-separated RGB values (e.g., `176 132 233`) to allow opacity modifiers in the code.

### 5. Run & Build

```bash
# Start Dev Server
npm run dev

# Build for Production
npm run build
```

## Agent Instructions

- **Always** clone the template first.
- **Check `public/config.js`** immediately after cloning. This is the source of truth.
- **If the user asks to change colors**, modify `app/styles/theme.css`. Explain that they need to use RGB triplet format (e.g., `255 0 0` not `#FF0000`) for the core variables.
- **If the user asks to change the menu**, update `VITE_ENABLED_MENUS` in `public/config.js`.

## Troubleshooting

- **"Broker ID not found"**: Ensure `VITE_ORDERLY_BROKER_ID` is set in `public/config.js`.
- **"Logo not showing"**: Ensure `VITE_HAS_PRIMARY_LOGO` is `"true"` and the file exists at `public/logo.webp`.
- **"Theme colors look wrong"**: Check `theme.css`. Ensure you used space-separated RGB values (e.g., `255 255 255`) instead of hex codes for variables starting with `--oui-color-`.
