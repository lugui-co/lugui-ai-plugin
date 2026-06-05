---
name: branding
description: Lugui brand tokens, typography, voice and a copyable on-brand HTML/CSS base template. Use whenever generating HTML/pages that will be published to pages.lugui.ai so the output looks like Lugui, not generic AI output.
---

# Lugui Branding

Apply this whenever you generate an HTML page destined for `pages.lugui.ai`.
The goal: pages should immediately read as **Lugui** — clean, trustworthy,
modern (Lugui is a real-estate / rentals fintech, B2B + consumer facing).

> **TODO: preencher com o brand guide oficial.** The exact hex values,
> fonts and logo asset below are placeholders. Confirm them against the
> official Lugui brand guide and replace every `/* TODO */` before treating a
> page as production-final. Until then, the structure is correct and safe to
> ship internally.

## Design tokens (CSS custom properties)

```css
:root {
  /* Colors — TODO: confirmar hex oficiais com o brand guide */
  --lugui-primary: #1a1a2e;        /* TODO: cor primaria de marca */
  --lugui-accent: #00d1b2;         /* TODO: cor de destaque/CTA */
  --lugui-bg: #ffffff;             /* fundo padrao */
  --lugui-surface: #f5f6fa;        /* cards / superficies elevadas */
  --lugui-text: #1a1a2e;           /* texto principal */
  --lugui-text-muted: #5b5f6e;     /* texto secundario */
  --lugui-border: #e3e5ec;         /* divisores / contornos */
  --lugui-success: #2bb673;
  --lugui-warning: #f5a623;
  --lugui-danger:  #e2483d;

  /* Typography — TODO: confirmar familia tipografica oficial */
  --lugui-font-sans: "Inter", system-ui, -apple-system, "Segoe UI", Roboto, sans-serif;
  --lugui-font-display: var(--lugui-font-sans); /* TODO: fonte de display, se houver */

  /* Spacing scale (4px base) */
  --space-1: 0.25rem; --space-2: 0.5rem; --space-3: 0.75rem;
  --space-4: 1rem;    --space-6: 1.5rem; --space-8: 2rem;
  --space-12: 3rem;   --space-16: 4rem;

  /* Radius & shadow */
  --radius: 12px;
  --radius-sm: 8px;
  --shadow-sm: 0 1px 2px rgba(16, 18, 35, 0.06);
  --shadow-md: 0 6px 24px rgba(16, 18, 35, 0.10);

  /* Layout */
  --content-max: 64rem;
}
```

## Voice & tone

- **Português brasileiro** por padrão (público interno e clientes BR). English
  só se a página for explicitamente internacional.
- Claro, direto, confiável — é uma fintech imobiliária. Sem jargão técnico em
  páginas voltadas a cliente/CEO.
- Confiança sem hype: evite superlativos vazios ("revolucionário"); prefira
  benefícios concretos.
- CTAs com verbo de ação no imperativo ("Começar agora", "Ver proposta").

## Logo

- TODO: incluir o asset oficial do logo (SVG inline de preferência) e a área de
  respiro mínima. Enquanto não houver, use o wordmark textual abaixo no header.
- Não distorça, não recolora fora da paleta, não aplique sombra no logo.

## Base HTML template (copiável)

Use this as the starting skeleton. It is self-contained (no external requests),
responsive, accessible, and uses the tokens above. Replace the placeholder
content. Keep CSS inlined in `<style>` so the page is a single deployable file.

```html
<!doctype html>
<html lang="pt-BR">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>TODO: titulo da pagina</title>
  <meta name="description" content="TODO: resumo de 1 linha (usado em previews)" />
  <style>
    :root {
      --lugui-primary: #1a1a2e;   /* TODO brand guide */
      --lugui-accent: #00d1b2;    /* TODO brand guide */
      --lugui-bg: #ffffff;
      --lugui-surface: #f5f6fa;
      --lugui-text: #1a1a2e;
      --lugui-text-muted: #5b5f6e;
      --lugui-border: #e3e5ec;
      --lugui-font-sans: "Inter", system-ui, -apple-system, "Segoe UI", Roboto, sans-serif;
      --radius: 12px;
      --shadow-md: 0 6px 24px rgba(16, 18, 35, 0.10);
      --content-max: 64rem;
    }
    * { box-sizing: border-box; }
    html { -webkit-text-size-adjust: 100%; }
    body {
      margin: 0;
      font-family: var(--lugui-font-sans);
      color: var(--lugui-text);
      background: var(--lugui-bg);
      line-height: 1.6;
    }
    .container { max-width: var(--content-max); margin: 0 auto; padding: 0 1.25rem; }
    header.site {
      border-bottom: 1px solid var(--lugui-border);
      padding: 1rem 0;
    }
    .brand { font-weight: 700; font-size: 1.25rem; color: var(--lugui-primary); letter-spacing: -0.01em; }
    main { padding: 3rem 0; }
    h1 { font-size: clamp(1.75rem, 4vw, 2.75rem); line-height: 1.15; margin: 0 0 1rem; letter-spacing: -0.02em; }
    p.lead { font-size: 1.125rem; color: var(--lugui-text-muted); max-width: 48ch; }
    .card {
      background: var(--lugui-surface);
      border: 1px solid var(--lugui-border);
      border-radius: var(--radius);
      padding: 1.5rem;
      box-shadow: var(--shadow-md);
    }
    .btn {
      display: inline-block; font-weight: 600; text-decoration: none;
      background: var(--lugui-accent); color: #06302a;
      padding: 0.75rem 1.25rem; border-radius: var(--radius);
    }
    .btn:focus-visible { outline: 3px solid var(--lugui-primary); outline-offset: 2px; }
    footer.site { border-top: 1px solid var(--lugui-border); padding: 1.5rem 0; color: var(--lugui-text-muted); font-size: 0.875rem; }
    @media (prefers-color-scheme: dark) {
      :root { --lugui-bg: #0f1020; --lugui-surface: #16182b; --lugui-text: #f2f3f7; --lugui-text-muted: #a7abbd; --lugui-border: #2a2d44; }
    }
  </style>
</head>
<body>
  <header class="site">
    <div class="container">
      <span class="brand">Lugui</span> <!-- TODO: trocar por <img> do logo oficial -->
    </div>
  </header>
  <main>
    <div class="container">
      <h1>TODO: titulo principal</h1>
      <p class="lead">TODO: subtitulo / proposta de valor em 1-2 frases.</p>
      <p><a class="btn" href="#">TODO: chamada para acao</a></p>
    </div>
  </main>
  <footer class="site">
    <div class="container">© Lugui — TODO: ano / aviso legal se necessario.</div>
  </footer>
</body>
</html>
```

## Rules of use

- One self-contained file; no external scripts/fonts/CDNs (see code-best-practices).
- Never inline real data, secrets or client PII (see security-checklist).
- Keep the token names; only change values to match the official brand guide.
