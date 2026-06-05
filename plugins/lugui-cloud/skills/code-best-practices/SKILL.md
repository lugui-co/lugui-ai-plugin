---
name: code-best-practices
description: Standards for HTML/CSS/JS generated for pages.lugui.ai — semantic markup, basic accessibility, self-contained (no untrusted external deps), CSP-friendly, and no inline secrets. Use when writing or reviewing HTML pages to be published.
---

# Code best practices for published pages

These pages are **public, static, single-file HTML** served from
`pages.lugui.ai` via CloudFront. Optimize for: loads anywhere, no broken
external dependencies, accessible, and safe to expose.

## Hard rules

1. **Single self-contained file.** Inline CSS in `<style>` and any JS in
   `<script>`. Do NOT pull in external CDNs, fonts, analytics, or trackers.
   They break offline, leak referrers, and create supply-chain risk. If a font
   is essential, embed it or fall back to a system stack.
2. **No secrets, ever.** No API keys, tokens, passwords, AWS keys, private
   keys, or signed URLs in the markup or JS. The publish gate rejects these,
   but never rely on the gate — see security-checklist.
3. **No client PII / internal data.** No customer names, CPFs, contract
   details, internal hostnames, or internal API endpoints. Public page = public
   internet.
4. **No remote calls to internal services.** Don't `fetch()` internal Lugui
   APIs from a public page; CORS and auth will fail and it exposes endpoints.

## HTML

- `<!doctype html>` + `<html lang="pt-BR">` (or the page's real language).
- Always include `<meta charset="utf-8">` and the responsive viewport meta.
- Use semantic elements: `header`, `nav`, `main`, `section`, `article`,
  `footer`, `h1`–`h6` in order (one `h1` per page).
- A meaningful `<title>` — the backend also uses it to derive a permanent slug.

## Accessibility (baseline)

- Every `<img>` has a meaningful `alt` (or `alt=""` if purely decorative).
- Color contrast ≥ 4.5:1 for body text against its background.
- Interactive elements are real `<a>`/`<button>` and keyboard-focusable, with a
  visible `:focus-visible` style.
- Don't convey meaning by color alone.
- Respect `prefers-color-scheme` and `prefers-reduced-motion` when adding motion.

## CSS

- Prefer CSS custom properties (use the Lugui brand tokens — see branding).
- Mobile-first, responsive; use `clamp()` for fluid type.
- Avoid heavy box-shadows/filters that cause repaint jank.

## JavaScript (only if truly needed)

- Most published pages need zero JS — prefer static HTML/CSS.
- If used: vanilla ES, no frameworks, no external scripts.
- **CSP-friendly:** keep it in a single `<script>` block; avoid `eval`,
  `new Function`, and inline event-handler attributes (`onclick="..."`) so the
  page survives a strict Content-Security-Policy. Attach handlers via
  `addEventListener`.
- Never store or transmit credentials; never call internal endpoints.

## Pre-publish checklist

- [ ] One file, no external CDN/font/analytics requests.
- [ ] Valid `<title>` and `lang`.
- [ ] Images have `alt`; contrast is adequate; focus styles present.
- [ ] No secrets, no PII, no internal endpoints.
- [ ] Renders correctly at 360px and at desktop widths.
