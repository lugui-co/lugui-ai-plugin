---
name: branding
description: Lugui's real design system — Plus Jakarta Sans, the cream/deep-green palette with lime as the signature accent, and component patterns extracted from a production Lugui page. Use whenever generating HTML/pages for pages.lugui.ai so the output looks unmistakably Lugui, not generic AI output.
---

# Lugui Branding

This skill is the **design system for any page you generate** for
`/lugui-ai:pages:publish`. It was extracted from a real production Lugui page
("Implantação · Lugui"), so following it makes output look like Lugui — warm,
structured, fintech-for-real-estate — and avoids generic "AI slop".

Two companion files live next to this one:

- **`tokens.css`** — the full `:root` (colors, radius, shadow, transitions,
  fonts). The source of truth. Copy/`@import` it.
- **`template.html`** — a clean, on-brand starting page (header + hero + card +
  footer) wired to the tokens. **Start every new page from this.**

## The feel in one line

Cream paper, deep forest green for structure, and a single electric **lime**
accent used like a highlighter. Confident, calm, human. Never neon-everywhere,
never flat white, never purple-gradient SaaS.

## Palette (and when to use each)

| Token | Hex | Use it for |
|---|---|---|
| `--lugui-dark` | `#2B4A42` | Structure: headers, primary buttons, dark cards, headings, text on cream. The workhorse. |
| `--lugui-lime` | `#D4F34A` | **Signature accent — sparingly.** Primary CTA text, the `lugui.` dot, active nav, one hero highlight, progress fill, focus glow. If everything is lime, nothing is. |
| `--lugui-cream` | `#EDE8DC` | The default **page background**. Pages sit on cream paper, not white. |
| `--lugui-white` | `#FFFFFF` | Elevated surfaces only — cards, tables, inputs — floating on the cream. |
| `--lugui-navy` | `#1A2332` | Deepest tone: primary-button hover, max contrast, shadow tint. |
| `--lugui-sage` | `#5A7A70` | Muted green for soft secondary accents / hover hints. |
| `--lugui-text` | `#1A1A1A` | Body copy. |
| `--lugui-muted` | `#5F6B68` | Secondary text, labels, hints, eyebrows. |
| `--lugui-border` | `#D9D4C7` | Default borders & dividers (warm, never cold grey). |
| `--lugui-border-strong` | `#B5AE9A` | Stronger/dashed borders (secondary buttons, file inputs, empty states). |
| `--lugui-premise` | `#F4EFD8` | Read-only / locked fields, callout background. |
| `--lugui-soft` | `#F7F4EA` | Soft neutral fill: hovers, table headers, chips, badges. |
| `--lugui-danger` | `#B8412A` | Errors, required `*`, destructive. |
| `--lugui-warn` | `#C7861B` | Warnings, in-progress. |
| `--lugui-success` | `#3F7A4A` | Success, sent/confirmed. |

Borders and shadows are **warm/navy-tinted**, never neutral grey. Shadows are
soft and low: `--shadow-sm/md/lg` use `rgba(26,35,50,…)`.

## Typography

- **Plus Jakarta Sans** for everything (weights 300–800), loaded from Google
  Fonts. Mono is `ui-monospace, "SF Mono", Monaco, monospace` (tokens, URLs).
- Base body: `15px`, line-height `1.55`, color `--lugui-text` on cream.
- Headings: weight `700`, `letter-spacing: -.01em`, color `--lugui-dark`.
- Scale seen in the product:
  - h1 / page title: `clamp(28px, 4vw, 38–40px)`, line-height `1.1`.
  - section title (`.block__title`, modal `h3`): `22px`.
  - card/empty title: `18px`.
  - body: `15px`; secondary/`small`: `13px`; hints: `12px`.
  - KPI value: `32px` weight `800`, `letter-spacing: -.02em`.
- **Eyebrow** pattern (recurring): `11px`, `letter-spacing: .14em`,
  `text-transform: uppercase`, weight `600`, color `--lugui-muted`. Use it
  above titles for context ("CUSTOMER SUCCESS · LUGUI").

## Logo / wordmark

The logo is **text, not an image**: the lowercase wordmark `lugui` followed by a
**lime period** — the dot is the brand signature.

```html
<a class="brand" href="/">lugui<span class="brand__dot">.</span></a>
```
```css
.brand { font-weight: 800; font-size: 20px; letter-spacing: -.02em; color: var(--lugui-dark); text-decoration: none; }
.brand__dot { color: var(--lugui-lime); }
```
On dark backgrounds the wordmark turns cream and the dot stays lime. Don't
recolor, distort, or add effects to it.

## Components (use these snippets, all token-driven)

### Buttons
```css
.btn { display:inline-flex; align-items:center; justify-content:center; gap:8px; padding:10px 18px; font:600 14px/1 var(--lugui-font-sans); border-radius:8px; border:1px solid transparent; cursor:pointer; transition: transform .08s ease, background var(--t-fast), border-color var(--t-fast); }
.btn:active { transform: translateY(1px); }
.btn-primary   { background: var(--lugui-dark); color: var(--lugui-lime); }   /* lime ON green = the CTA */
.btn-primary:hover { background: var(--lugui-navy); }
.btn-secondary { background: var(--lugui-white); color: var(--lugui-dark); border-color: var(--lugui-border-strong); }
.btn-secondary:hover { border-color: var(--lugui-dark); }
.btn-sm { padding:6px 12px; font-size:13px; border-radius:6px; }  .btn-lg { padding:14px 24px; font-size:15px; }
```
The hero CTA is **lime text on deep green** — that pairing is the most "Lugui"
thing on the page. Use one primary button per view.

### Card
```css
.card { background: var(--lugui-white); border: 1px solid var(--lugui-border); border-radius: var(--radius-lg); padding: 24px; }
.card--dark { background: var(--lugui-dark); color: var(--lugui-cream); border-color: transparent; }  /* closing/CTA cards */
.card--hover:hover { transform: translateY(-2px); box-shadow: var(--shadow-md); }  /* clickable cards */
```

### Badge / status pill
```css
.badge { display:inline-flex; align-items:center; gap:6px; padding:4px 10px; font:700 11px/1 var(--lugui-font-sans); letter-spacing:.06em; text-transform:uppercase; border-radius:999px; background: var(--lugui-soft); color: var(--lugui-dark); }
.badge--success { background:#E0EDDD; color: var(--lugui-success); }
.badge--warn    { background:#FBE9C5; color: var(--lugui-warn); }
.badge--danger  { background:#F5D6CE; color: var(--lugui-danger); }
.badge--accent  { background: var(--lugui-lime); color: var(--lugui-dark); }   /* e.g. "revisado" */
```

### Callout (premise / important note)
```css
.callout { padding:16px 20px; background: var(--lugui-premise); border-left: 3px solid var(--lugui-lime); border-radius:6px; font-size:14px; line-height:1.55; }
.callout strong { color: var(--lugui-dark); }
```
A premise/important inline highlight is a **lime pill** (`.premise-badge`:
lime bg, dark text, uppercase 10px) — reuse the lime sparingly here too.

### Table
```css
.table-wrap { background: var(--lugui-white); border:1px solid var(--lugui-border); border-radius: var(--radius-lg); overflow:hidden; }
table.tbl { width:100%; border-collapse:collapse; }
table.tbl th { padding:14px 18px; text-align:left; font-size:11px; letter-spacing:.12em; text-transform:uppercase; color:var(--lugui-muted); font-weight:600; background: var(--lugui-soft); border-bottom:1px solid var(--lugui-border); }
table.tbl td { padding:14px 18px; font-size:14px; border-bottom:1px solid var(--lugui-border); }
table.tbl tbody tr:hover { background: var(--lugui-soft); }
```

### Inputs (when a page needs a form)
```css
input, select, textarea { width:100%; padding:10px 12px; font:14px var(--lugui-font-sans); color: var(--lugui-text); background: var(--lugui-white); border:1px solid var(--lugui-border); border-radius: var(--radius-sm); transition: border-color var(--t-fast), box-shadow var(--t-fast); }
input:focus, select:focus, textarea:focus { outline:none; border-color: var(--lugui-dark); box-shadow: var(--lugui-focus-ring); }  /* lime glow */
```

## Layout & spacing

- Sticky white **appbar** (wordmark left, nav right) over a cream body.
- Content widths: full app `1280px`; reading/forms `880px`. Center with
  `margin: 0 auto`.
- Generous vertical rhythm: page container `padding: 32–48px 24px 96px`; cards
  `24–36px`. Use `.stack > * + * { margin-top: 12–20px }` for vertical flow and
  CSS grid (`grid-2/3/4`, gap `16px`) for columns — collapse to 1 column under
  ~880px.
- Hero signature: an `<h1>` with one `<em>` whose `font-style` is reset and that
  gets a **lime background highlight** (`background: var(--lugui-lime); padding:
  0 8px; border-radius: 4px`). One per page.

## Radius & shadows

- Radius: `sm 6px` (inputs/pills-of-text), `md 10px` (small cards/inputs),
  `lg 16px` (cards/tables/modals), `xl 24px` (large feature blocks). Pills use
  `999px`.
- Shadows: low and navy-tinted (`--shadow-sm/md/lg`). Cards are usually flat
  (border only); reserve shadow for hover, modals, toasts, floating bars.
- Transitions: `--t-fast .15s` for hovers, `--t-base .25s` for expand/collapse.

## Voice & tone

Inferred from the product copy (a client onboarding form): **professional,
clear, and genuinely warm — plain Brazilian Portuguese, zero jargon.**

- Default language **pt-BR**; talk *to* the person ("Você já pode...", "Reserve
  um tempo para preencher com calma").
- Reassuring and low-pressure: "Não se preocupe em deixar perfeito de primeira",
  "a CS revisa com vocês", "Se não se aplica, escreva 'Não se aplica' e siga".
- Concrete over hype. Short sentences. Use `—` em dashes and a light human touch
  ("— Equipe Lugui"). Sign off as a partner, not a vendor.
- No tech jargon on client/CEO-facing pages.

## Rules — do / don't

**Do**
- Start from `template.html`; pull values from `tokens.css`. Keep token names.
- Always **Plus Jakarta Sans**; cream background; lime as a *rare* accent.
- Lead with hierarchy: clear eyebrow → big h1 → muted subtitle. Strong contrast,
  intentional spacing, warm borders, one focal CTA.
- Self-contained single file (inline `<style>`, no external CSS/JS deps beyond
  the Google Fonts link). Keep it accessible (semantic tags, `alt`, focus ring).
- Sweat the details: the lime dot on the wordmark, the `.14em` eyebrow tracking,
  the navy-tinted shadows. Those small things are what read as "Lugui".

**Don't**
- ❌ Pure flat white pages — Lugui sits on **cream**.
- ❌ Lime everywhere — it's a highlighter, not a fill. One or two hits per view.
- ❌ AI-slop tells: purple/blue gradients, Inter/Arial/system-default look,
  emoji-as-icons everywhere, generic centered "hero + 3 cards" with no point,
  neutral grey borders/shadows, cold corporate copy.
- ❌ Image logos or recolored wordmarks — it's the text `lugui` + lime dot.
- ❌ Tiny cramped type or weak hierarchy.

## Instruction for Claude

When generating an HTML page for `/lugui-ai:pages:publish`:
1. **Start from `template.html`** and bring in `tokens.css` (inline the `:root`).
2. Use the component snippets above; don't invent off-brand styles.
3. Background = cream; structure = deep green; **one** lime accent moment.
4. Write copy in warm, plain pt-BR per the voice section.
5. Then hand off to the `security-checklist` and `code-best-practices` skills
   before publishing.
