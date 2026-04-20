# Orderly One — Theming

Generate and customize DEX themes using AI or manual CSS. For authentication, see `orderly-one-general`.

> **Tip:** The Orderly One web portal at [https://dex.orderly.network/dex](https://dex.orderly.network/dex) has an interactive theme editor with live preview. Users can customize themes visually without writing CSS.

---

## AI Theme Generation

### Modify Theme

Generate 3 CSS theme variants from a text description:

```
POST /api/theme/modify
Authorization: Bearer <token>
Content-Type: application/json

{
  "prompt": "Ocean blue theme with teal accents",
  "currentTheme": "<optional: current CSS to base modifications on>"
}
```

**Response:**
```json
{
  "themes": [
    ":root { --oui-color-primary: 41 128 185; ... }",
    ":root { --oui-color-primary: 52 152 219; ... }",
    ":root { --oui-color-primary: 44 62 80; ... }"
  ]
}
```

- Returns exactly 3 variants with different temperatures (0.7, 0.8, 0.9)
- If `currentTheme` is omitted, uses the default Orderly theme as base
- Rate limited per user
- Uses Cerebras AI (Qwen model) for CSS generation

### Fine-Tune Elements

Generate CSS overrides for specific UI elements:

```
POST /api/theme/fine-tune
Authorization: Bearer <token>
Content-Type: application/json

{
  "prompt": "Make this section darker with a subtle gradient border",
  "html": "<div class='trading-panel'>...</div>",
  "elements": [
    {
      "elementSelector": ".trading-panel",
      "computedStyles": {
        "background-color": "rgb(36, 32, 47)",
        "color": "rgb(255, 255, 255)"
      }
    }
  ],
  "cssVariables": {
    "--oui-color-primary": "176 132 233",
    "--oui-color-fill": "36 32 47"
  },
  "existingOverrides": ".trading-panel { border: 1px solid rgb(93 83 123); }"
}
```

**Response:**
```json
{
  "overrides": [
    ".trading-panel { background: linear-gradient(...); ... }",
    ".trading-panel { ... variant 2 ... }",
    ".trading-panel { ... variant 3 ... }"
  ]
}
```

Returns 3 CSS override variants. If `existingOverrides` is provided, all existing overrides are preserved and combined with new ones.

---

## Default Theme CSS Variables

All colors use **space-separated RGB values** (e.g., `176 132 233` not `#B084E9`).

### Brand Colors

```css
--oui-color-primary: 176 132 233;       /* Main accent (purple) */
--oui-color-primary-light: 213 190 244; /* Lighter shade */
--oui-color-primary-darken: 137 76 209; /* Darker shade */
--oui-color-primary-contrast: 255 255 255; /* Text on primary bg */

--oui-color-link: 189 107 237;
--oui-color-link-light: 217 152 250;
```

### Status Colors

```css
--oui-color-success: 41 233 169;         /* Green - profit */
--oui-color-success-light: 101 240 194;
--oui-color-success-darken: 0 161 120;

--oui-color-danger: 245 97 139;          /* Red - loss */
--oui-color-danger-light: 250 167 188;
--oui-color-danger-darken: 237 72 122;

--oui-color-warning: 255 209 70;         /* Amber */
--oui-color-warning-light: 255 229 133;
--oui-color-warning-darken: 255 152 0;
```

### Trading Colors

```css
--oui-color-trading-profit: 41 233 169;         /* Green */
--oui-color-trading-profit-contrast: 255 255 255;
--oui-color-trading-loss: 245 97 139;           /* Red */
--oui-color-trading-loss-contrast: 255 255 255;
```

### Background & Base Colors (Dark → Darkest)

```css
--oui-color-fill: 36 32 47;             /* Main background */
--oui-color-fill-active: 40 46 58;      /* Active state */

--oui-color-base-1: 93 83 123;          /* Lightest (borders, muted text) */
--oui-color-base-2: 81 72 107;
--oui-color-base-3: 68 61 69;
--oui-color-base-4: 57 52 74;
--oui-color-base-5: 51 46 66;
--oui-color-base-6: 43 38 56;
--oui-color-base-7: 36 32 47;           /* Scrollbar tracks */
--oui-color-base-8: 29 26 38;
--oui-color-base-9: 22 20 28;
--oui-color-base-10: 14 13 18;          /* Darkest */

--oui-color-base-foreground: 255 255 255;
--oui-color-line: 255 255 255;
```

### Gradients

```css
--oui-gradient-primary-start: 40 0 97;
--oui-gradient-primary-end: 189 107 237;

--oui-gradient-secondary-start: 81 42 121;
--oui-gradient-secondary-end: 176 132 233;

--oui-gradient-success-start: 1 83 68;
--oui-gradient-success-end: 41 223 169;

--oui-gradient-danger-start: 153 24 76;
--oui-gradient-danger-end: 245 97 139;

--oui-gradient-brand-start: 231 219 249;
--oui-gradient-brand-end: 159 107 225;
--oui-gradient-brand-stop-start: 6.62%;
--oui-gradient-brand-stop-end: 86.5%;
--oui-gradient-brand-angle: 17.44deg;

--oui-gradient-warning-start: 152 58 8;
--oui-gradient-warning-end: 255 209 70;

--oui-gradient-neutral-start: 27 29 24;
--oui-gradient-neutral-end: 38 41 46;
```

### Spacing & Rounding

```css
--oui-rounded-sm: 2px;
--oui-rounded: 4px;
--oui-rounded-md: 6px;
--oui-rounded-lg: 8px;
--oui-rounded-xl: 12px;
--oui-rounded-2xl: 16px;
--oui-rounded-full: 9999px;

--oui-spacing-xs: 20rem;
--oui-spacing-sm: 22.5rem;
--oui-spacing-md: 26.25rem;
--oui-spacing-lg: 30rem;
--oui-spacing-xl: 33.75rem;
```

### Font

```css
--oui-font-family: 'Manrope', sans-serif;
```

---

## TradingView Chart Colors

The `tradingViewColorConfig` field accepts a JSON string for chart styling:

```json
{
  "upColor": "#29E9A9",
  "downColor": "#F5618B",
  "backgroundColor": "#1D1A26",
  "textColor": "#FFFFFF",
  "gridColor": "#2B2638",
  "borderColor": "#332E42"
}
```

Submit as a JSON string in the `tradingViewColorConfig` field when creating/updating DEX.

---

## Applying Themes

### Via API

Include `themeCSS` in `POST /api/dex` or `PUT /api/dex/{id}` (multipart/form-data):

```
themeCSS: ":root { --oui-color-primary: 41 128 185; --oui-color-fill: 15 20 30; ... }"
```

### Via Template

Edit `app/styles/theme.css` directly (see `orderly-one-template`).

---

## Theme Design Rules

When creating themes (manually or via AI):

1. **Always dark backgrounds, light text** — this is a dark trading platform
2. **Base colors progress from lightest (base-1) to darkest (base-10)**
3. **Scrollbar visibility** — `base-7` (track) must be visibly different from `base-10` (background)
4. **Trading colors must be bright** — green for profit and red for loss must be visible on dark backgrounds
5. **Use space-separated RGB** — `176 132 233` not `#B084E9` or `rgb(176, 132, 233)`
6. **Don't change spacing values** — the spacing variables should stay as-is
7. **Preserve all variable names** — don't add or remove CSS variables

---

## Common Issues

| Issue | Solution |
|-------|----------|
| "Invalid CSS syntax" on submit | Validate CSS before sending. Use `validateCSS()` locally or test in browser |
| Theme looks wrong | Check that you're using space-separated RGB, not hex |
| Text unreadable | Ensure sufficient contrast — light text on dark backgrounds |
| AI returns bad theme | Try a more specific prompt. The AI generates 3 variants — pick the best one |
| Rate limited | Wait for cooldown, then retry |

---

## Related Skills

- `orderly-one-general` — Auth, overview
- `orderly-one-create-dex` — Apply themes via `themeCSS` field
- `orderly-one-template` — Edit `theme.css` directly in the template
