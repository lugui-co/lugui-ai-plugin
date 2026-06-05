---
description: Validate an HTML file and publish it to a public pages.lugui.ai link, using curl + your personal token. No Python, no libraries.
argument-hint: "<arquivo.html>"
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
   - **Start from `skills/branding/template.html`** instead of writing markup
     from scratch.
   - **Use the real tokens from `skills/branding/tokens.css`** (inline the
     `:root`). Background must be **cream**.
   - **Load Plus Jakarta Sans** via Google Fonts:
     `https://fonts.googleapis.com/css2?family=Plus+Jakarta+Sans:wght@300;400;500;600;700;800&display=swap`
   - **NEVER invent tokens.** There is no `--lugui-primary`, `--lugui-accent`,
     `--lugui-bg`, or generic greys like `#1a1a2e`/`#f5f6fa`. The correct names
     and values are exactly: `--lugui-dark #2B4A42`, `--lugui-lime #D4F34A`,
     `--lugui-cream #EDE8DC`, `--lugui-navy #1A2332`, `--lugui-sage #5A7A70`,
     `--lugui-text #1A1A1A`, `--lugui-muted #5F6B68`, `--lugui-border #D9D4C7`,
     plus status/tints — see `tokens.css` for the complete list. If you don't
     remember a value, READ `tokens.css`; do not guess.

   **Brand sanity-check (run right before the upload, every time):** confirm the
   HTML you are about to send (a) loads Plus Jakarta Sans (a
   `fonts.googleapis.com` link containing `Plus+Jakarta+Sans`), and (b) uses the
   real tokens `--lugui-dark` / `--lugui-lime` / `--lugui-cream` with a cream
   background.
   - If it PASSES → continue to the JSON build.
   - If it FAILS (e.g. a pre-existing off-brand file, or invented tokens) →
     **STOP. Do NOT publish a non-branded page silently.** Tell the user the
     page is off-brand and offer to restyle it to the Lugui standard (template +
     tokens + Plus Jakarta Sans) first. Only publish after it passes, or after
     the user explicitly insists on publishing the file as-is.

7. **Build the JSON body in a temp file (do NOT interpolate the HTML inline).**
   The HTML contains quotes and newlines, so escape it as a proper JSON string.
   Write the request body to a temp file, e.g. `/tmp/lugui-ai-publish.json`, with
   this shape:

   ```json
   {
     "html": "<!doctype html>...escaped HTML as a JSON string...",
     "type": "permanent",
     "slug": null
   }
   ```

   - Use `"type": "permanent"` or `"type": "ephemeral"` per the user's choice.
   - Set `"slug"` to the chosen canonical path for permanent pages (it may be a
     nested path like `relatorios/cardinali/q2`), or `null`.
   - The simplest robust way: use the Write tool to author the JSON file with
     the HTML correctly escaped as a JSON string value. `jq`/`python` are NOT
     required — you build the JSON yourself.

8. **POST with curl, reading the body from the temp file:**

   ```bash
   curl -sS -w "\n%{http_code}" -X POST "<pages_api>/pages" \
     -H "Authorization: Bearer <token>" \
     -H "Content-Type: application/json" \
     --data @/tmp/lugui-ai-publish.json
   ```

   The last line of output is the HTTP status code; everything before it is the
   JSON response body.

9. **Report the result.** On **200/201**, parse the response JSON and show the
   public `url` (and `expires_at` if present, for ephemeral pages). Map errors:
   - **401 / 403** → invalid token or no permission → run `/lugui-ai:setup` to
     redo the web login and get a fresh token.
   - **409** → that path is already used by someone else → ask the user for a
     different path and retry.
   - **413** → HTML too large → trim the page and retry.
   - **422** → server rejected the request. Either the path is invalid/reserved
     (ask for a different path) or the HTML is invalid (empty / not HTML /
     contains a secret) → fix and retry.

10. **Clean up.** Remove the temp JSON file (`rm -f /tmp/lugui-ai-publish.json`)
    so the HTML/body doesn't linger.

## Notes

- The request body is `{ "html", "type", "slug" }`; the response is
  `{ url, slug, type, expires_at }`.
- Never paste the token or the contents of `~/.lugui/config.json` into chat.
