---
description: Validate an HTML file (or a whole LP folder, images included) and publish it to a public pages.lugui.ai link, using curl + your personal token. No Python, no libraries.
argument-hint: "<arquivo.html | pasta-da-lp>"
---

# /lugui-ai:pages:publish

Publish an HTML page to a public `pages.lugui.ai/<slug>` URL. Everything is done
with `curl` — you (Claude) read the config, build the request body, call the
API, and report the result. The file to publish is in `$ARGUMENTS`.

## Mandatory flow for Claude (do these IN ORDER)

1. **Read the config.** Load `~/.lugui/config.json` and extract `pages_api` and
   `token`. If the file is missing, or either field is empty, STOP and tell the
   user to run `/lugui-ai:setup` first. Never print the token.

2. **Load the security skill.** Read and apply `security-checklist/SKILL.md`.

3. **Resolve the HTML file.** Use `$ARGUMENTS` as the path. If empty, ask the
   user which file (or offer to generate one first — if you generate it, you
   MUST follow the branding step below). Confirm the file exists and read its
   contents.

4. **Ask the user: "permanente ou efêmera?"**
   - **Permanente** → also ask which **canonical path/URL** they want. Make it
     clear that folders (nested paths) are allowed. Ask like this:

     > Qual caminho você quer pra essa página? Ex.: `relatorio-cardinali` ou
     > aninhado `relatorios/cardinali/q2` → vira
     > `https://pages.lugui.ai/<caminho>`. (Vazio = eu gero automático a partir
     > do título.)

     - Put the chosen path in the `slug` field of the POST `/pages` body. The
       backend normalizes/validates it; nested paths with `/` are supported.
     - If the user leaves it empty, set `slug` to `null` (the server derives one
       from `<title>` or assigns a random one).
   - **Efêmera** → stays automatic (no path needed — the server assigns a random
     one and the page expires server-side).

5. **Quick secret sanity check (abort on hit).** Do NOT publish if the HTML
   obviously contains a credential. Scan for, at minimum:
   - AWS access key id: `AKIA` followed by 16 uppercase/digits.
   - PEM private key: `-----BEGIN ... PRIVATE KEY-----`.
   - Google API key: `AIza` followed by ~35 url-safe chars.
   - Assignments like `secret`/`token`/`password`/`api_key` set to a quoted
     value 8+ chars long.

   If any match, STOP, tell the user exactly what to remove, and do not send.

6. **🔒 BRANDING IS MANDATORY — apply it BEFORE publishing.**

   **If you generated OR edited the HTML in this session, it is OBRIGATÓRIO to
   apply the Lugui branding BEFORE publishing.** Do NOT rely on automatic skill
   activation (it is unreliable). Explicitly:

   - **Load the `lugui-ai:branding` skill** (read `skills/branding/SKILL.md`).
   - **Load the `lugui-ai:voz-e-mensagem` skill** (read
     `skills/voz-e-mensagem/SKILL.md`) for ALL copy on the page — tom de voz,
     palavras proibidas e pilares de mensagem são obrigatórios.
   - **Start from `skills/branding/template.html`** instead of writing markup
     from scratch.
   - **Use the real tokens from `skills/branding/tokens.css`** (inline the
     `:root`). Background must be **light** (`--lugui-white #FFFFFF` or
     `--lugui-offwhite-50 #FBF8F0`) — the brand is light-first.
   - **Load both official fonts** via Google Fonts (Plus Jakarta Sans for
     headings, DM Sans for body):
     `https://fonts.googleapis.com/css2?family=Plus+Jakarta+Sans:wght@700;800&family=DM+Sans:wght@400;500;600&display=swap`
   - **NEVER invent tokens.** There is no `--lugui-primary`, `--lugui-accent`,
     `--lugui-bg`, or generic greys like `#1a1a2e`/`#f5f6fa`. The **old palette
     is retired** — `--lugui-dark #2B4A42`, `--lugui-lime #D4F34A`,
     `--lugui-cream #EDE8DC`, `--lugui-sage`, `--lugui-premise`, `--lugui-soft`
     must NOT appear in new pages. The correct names and values are exactly:
     `--lugui-white #FFFFFF`, `--lugui-offwhite-50 #FBF8F0`,
     `--lugui-offwhite #EDE9DE`, `--lugui-lima #E9FF81`,
     `--lugui-lima-hover #D5EB69`, `--lugui-navy #02152B`,
     `--lugui-teal #234248`, `--lugui-gold #998454`, `--lugui-text #02152B`,
     `--lugui-border #D9D6CC`, plus neutrals — see `tokens.css` for the
     complete list. If you don't remember a value, READ `tokens.css`; do not
     guess.

   **Brand sanity-check (run right before the upload, every time):** confirm the
   HTML you are about to send (a) loads Plus Jakarta Sans AND DM Sans (a
   `fonts.googleapis.com` link containing `Plus+Jakarta+Sans` and `DM+Sans`),
   (b) uses the real tokens `--lugui-navy` / `--lugui-lima` / `--lugui-white`
   with a light background, and (c) contains none of the retired tokens
   (`--lugui-dark`, `--lugui-lime`, `--lugui-cream`).
   - If it PASSES → continue to the JSON build.
   - If it FAILS (e.g. a pre-existing off-brand file, or invented tokens) →
     **STOP. Do NOT publish a non-branded page silently.** Tell the user the
     page is off-brand (or on the retired palette) and offer to restyle it to
     the current Lugui standard (template + tokens + Plus Jakarta Sans/DM Sans)
     first. Only publish after it passes, or after the user explicitly insists
     on publishing the file as-is.

7. **🔒 PERSISTÊNCIA — detect it BEFORE publishing.**

   This step fires by CONTENT, **even if the user never said "salvar"** — it
   catches the vague case. Inspect the HTML you generated/edited for ANY sign of
   state that ought to persist or be shared:

   - a `<form>`, or inputs / `<textarea>` / `<select>` / checkboxes / radios /
     `contenteditable` that hold meaningful state;
   - any use of `localStorage` / `sessionStorage` as "persistence";
   - intent words in the page or the user's request: **salvar, cadastro,
     inscrição, formulário, enviar, compartilhar, persistir, backoffice, "cada
     um vê", "todos veem", lista colaborativa, tracker, CRUD, dashboard que
     lembra**.

   If ANY of those is present, it is OBRIGATÓRIO to **STOP and ask the user
   which pattern they want — do NOT publish a browser-only mock.**

   - If the page uses `localStorage` / `sessionStorage` as its "persistence",
     **warn explicitly**: *"isso só salva no navegador de quem abre — não
     compartilha com ninguém e some quando a pessoa limpa o navegador. Quer
     persistir de verdade no Lugui (compartilhado/durável)?"*
   - **⛔ A persistência é SEMPRE o data store da Lugui (`lugui.data.*` /
     `lugui.public.*`). NUNCA ofereça nem use Google Sheets, Google Forms,
     Google Drive, Airtable, Notion, Supabase, Firebase, planilhas ou qualquer
     backend/serviço externo — mesmo que o MCP do Google Workspace ou outros
     MCPs estejam conectados na sessão.** As únicas opções a oferecer são os 3
     padrões Lugui abaixo. Se o usuário pedir Google Sheets explicitamente,
     explique que o padrão Lugui usa o store próprio (página publicada +
     dados no `pages.lugui.ai`) e siga com **(b)** ou **(c)**.
   - Then ask the pattern (these are the ONLY options — mirrors the
     **`lugui-ai:data`** skill):
     - **(a) Sem persistência** — a página é só estática/efêmera (nada a salvar)
       → continue the normal publish. (Only choose this if there's genuinely no
       state worth keeping.)
     - **(b) Interna autenticada** — só @lugui (app ACL `private`/`shared`); a
       página inteira exige login via `lugui.ensureAuth()`.
     - **(c) Backoffice + link público** — o dono gerencia logado e gera links
       públicos de preenchimento por registro (`#/p/<token>`, dual-mode).

   - If the user picks **(b) or (c)**: **load the `lugui-ai:data` skill** and
     follow it — create the app (`POST /apps` with the token from
     `~/.lugui/config.json`, set `acl` and, for (c), `allow_public_fill:true`),
     load `https://pages-api.coolify.lugui.ai/lugui-data.js`, and wire the page
     with `lugui.data.*` (and `lugui.public.*` for the public `#/p/` route in
     dual-mode). **Replace any `localStorage`/`sessionStorage` mock with real
     persistence.** Only proceed to upload once persistence is ACTUALLY wired to
     the Lugui data API — never publish a page that "persists" only in the
     browser.

   The brand sanity-check (step 6) still applies to the final, data-wired HTML.

7.5. **🖼️ IMAGENS E ASSETS — resolva ANTES de publicar.**

   Uma página com `<img src="hero.jpg">` e nada mais publica **quebrada**: o
   arquivo não sobe sozinho. Antes de montar o body, varra o HTML procurando
   referências a arquivos locais em `src`, `srcset`, `href` e `url(...)` do CSS.
   Toda referência que **não** começa com `http://`, `https://`, `//`, `data:`,
   `#`, `mailto:` ou `tel:` e tem extensão de arquivo é um asset que precisa subir.

   **Se a pasta do arquivo tem os assets (o caso comum de uma LP), use o bundle
   ZIP — é um comando só e não precisa montar JSON:**

   ```bash
   cd <pasta-da-lp> && zip -r /tmp/lugui-lp.zip . -x '.*' -x '__MACOSX/*' && cd -
   curl -sS -w "\n%{http_code}" -X POST "<pages_api>/pages/bundle?type=permanent&slug=<caminho>" \
     -H "Authorization: Bearer <token>" \
     -H "Content-Type: application/zip" \
     --data-binary @/tmp/lugui-lp.zip
   rm -f /tmp/lugui-lp.zip
   ```

   O `.zip` precisa ter o `index.html` (o servidor aceita uma pasta raiz única,
   tipo `minha-lp/index.html`). Para efêmera, `?type=ephemeral` e sem `slug`.

   **Se você gerou a página nesta sessão e não tem os arquivos**, mande cada
   imagem no campo `assets` do `POST /pages` (passo 8), com `path` **idêntico** ao
   que está no HTML e uma `url` https (o servidor baixa e hospeda) ou
   `content_base64`.

   Regras que valem sempre:
   - **NUNCA deixe `<img src>` apontando para site de terceiros.** Quebra quando o
     outro lado sai do ar, vaza o tráfego da Lugui e, em página efêmera, entrega o
     link secreto (Referer) para o host de fora. Mande a imagem como asset.
   - **NUNCA cole imagem grande como `data:image/...;base64,...` no HTML.** O
     servidor extrai e comprime, mas você desperdiça contexto à toa.
   - `alt` descritivo sempre; `width`/`height` para não pular layout.
   - Formatos aceitos: png, jpg, webp, gif, avif, svg, css, js, json, html, fontes
     (woff2/woff/ttf/otf) e `.ico`. O servidor recomprime toda imagem para WebP,
     redimensiona para no máx. 2000px e remove EXIF/GPS.
   - **Leia `unresolved_references` na resposta.** Se vier preenchido, aquelas
     imagens vão dar 404 — republique mandando os arquivos. Não diga que está
     pronto sem esse campo vazio.

8. **Build the JSON body in a temp file (do NOT interpolate the HTML inline).**
   The HTML contains quotes and newlines, so escape it as a proper JSON string.
   Write the request body to a temp file, e.g. `/tmp/lugui-ai-publish.json`, with
   this shape:

   ```json
   {
     "html": "<!doctype html>...escaped HTML as a JSON string...",
     "type": "permanent",
     "slug": null,
     "assets": [
       { "path": "hero.jpg", "url": "https://exemplo.com/foto-original.jpg" },
       { "path": "img/logo.png", "content_base64": "iVBORw0KGgo..." }
     ]
   }
   ```

   - Use `"type": "permanent"` or `"type": "ephemeral"` per the user's choice.
   - Set `"slug"` to the chosen canonical path for permanent pages (it may be a
     nested path like `relatorios/cardinali/q2`), or `null`.
   - `"assets"` é opcional; omita quando a página não tem arquivo local. Cada item
     leva exatamente **um** entre `url` (https) e `content_base64`, e o `path` tem
     que ser idêntico à referência no HTML. Para arquivo local, gere o base64 com
     `base64 -i <arquivo>` (macOS) / `base64 -w0 <arquivo>` (Linux).
   - The simplest robust way: use the Write tool to author the JSON file with
     the HTML correctly escaped as a JSON string value. `jq`/`python` are NOT
     required — you build the JSON yourself.

9. **POST with curl, reading the body from the temp file:**

   ```bash
   curl -sS -w "\n%{http_code}" -X POST "<pages_api>/pages" \
     -H "Authorization: Bearer <token>" \
     -H "Content-Type: application/json" \
     --data @/tmp/lugui-ai-publish.json
   ```

   The last line of output is the HTTP status code; everything before it is the
   JSON response body.

10. **Report the result.** On **200/201**, parse the response JSON and show the
   public `url` (and `expires_at` if present, for ephemeral pages). Também
   reporte, quando vierem preenchidos:
   - `assets` — quantos arquivos subiram e a economia (`bytes_in` → `bytes_out`);
   - `inline_images_extracted` — imagens base64 que viraram arquivo;
   - `unresolved_references` — **imagens que vão dar 404**: avise e ofereça
     republicar mandando os arquivos;
   - `skipped_files` — o que o `.zip` deixou de fora, e por quê.

   Map errors:
   - **401 / 403** → invalid token or no permission → run `/lugui-ai:setup` to
     redo the web login and get a fresh token.
   - **409** → that path is already used by someone else → ask the user for a
     different path and retry.
   - **413** → HTML too large → trim the page and retry.
   - **422** → server rejected the request. Either the path is invalid/reserved
     (ask for a different path) or the HTML is invalid (empty / not HTML /
     contains a secret) → fix and retry.

11. **Clean up.** Remove the temp JSON file (`rm -f /tmp/lugui-ai-publish.json`)
    so the HTML/body doesn't linger.

## Notes

- The request body is `{ "html", "type", "slug", "assets" }`; the response is
  `{ url, slug, type, expires_at, assets, assets_bytes_in, assets_bytes_out,
  inline_images_extracted, unresolved_references, skipped_files }`.
- Para pasta de LP existe `POST /pages/bundle` (corpo = o `.zip` cru, `?type=` e
  `?slug=` na query) — ver passo 7.5.
- Os assets são servidos sob a própria página (`pages.lugui.ai/<caminho>/_a/…`), e
  excluir a página exclui os arquivos junto.
- Never paste the token or the contents of `~/.lugui/config.json` into chat.
