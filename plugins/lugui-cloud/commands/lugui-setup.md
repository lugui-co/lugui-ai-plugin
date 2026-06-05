---
description: One-time setup for Lugui Cloud — log in on the web with your @lugui.ai account, paste the token shown, and validate it. Self-service, no admin, no Python.
---

# /lugui-setup

Configure this machine to publish to `pages.lugui.ai`. Authentication is a
**personal access token** (`lgp_...`) that you obtain yourself via **web login**:
you open a URL, sign in with your **@lugui.ai** account, and the page shows you a
token to copy. No admin issues it for you. Publishing is done entirely with
`curl` — no Python, no libraries.

## Steps for Claude (do these IN ORDER)

1. **Determine the API base URL.** If `~/.lugui/config.json` exists and has a
   `pages_api`, reuse it. Otherwise use the default
   `https://pages-api.coolify.lugui.ai`. (Don't overwrite an existing
   `pages_api` unless the user asks.)

2. **Check if already set up — don't make the user log in again for nothing.**
   If the config already has a `token`, validate it FIRST:

   ```bash
   curl -sS -o /dev/null -w "%{http_code}" \
     -H "Authorization: Bearer <token>" "<pages_api>/pages"
   ```

   - **200** → already configured. Tell the user they're good to go
     (`/lugui-publish ./page.html`) and **STOP** — skip the login.
   - **401 / no token in config** → continue to step 3 to authenticate.

3. **Open the login page in the user's browser** at **`{pages_api}/login`**.
   Detect the OS and run the matching opener (substitute the real `{pages_api}`):

   ```bash
   open "<pages_api>/login"        # macOS (Darwin)
   xdg-open "<pages_api>/login"    # Linux
   start "" "<pages_api>/login"    # Windows
   ```

   If the opener isn't available or fails, **don't error** — just fall back to
   showing the URL. Then tell the user:

   > Abri o login no navegador. Logue com sua conta **@lugui.ai**, copie o token
   > `lgp_...` que aparecer e cole aqui.
   > _(Se não abriu sozinho, acesse: **`{pages_api}/login`**)_

4. **Ask the user to paste the token** (`lgp_xxxxxxxx...`).
   - Treat the token as a secret. Do NOT echo it back in full in chat and do
     NOT write it anywhere except `~/.lugui/config.json`.

5. **Write / update `~/.lugui/config.json`** (keep the existing `pages_api` if
   there was one):

   ```json
   {
     "pages_api": "https://pages-api.coolify.lugui.ai",
     "token": "lgp_..."
   }
   ```

   Lock it down so other users can't read the token:

   ```bash
   mkdir -p ~/.lugui && chmod 700 ~/.lugui
   # ...write the file...
   chmod 600 ~/.lugui/config.json
   ```

6. **Validate the new token** (same curl as step 2):

   - **200** → done. Tell the user they can now run `/lugui-publish ./page.html`.
   - **401** → wrong/expired token. Reopen `{pages_api}/login`, copy a fresh
     token, and retry. Do NOT keep a bad token.
   - **Connection error** → check the `pages_api` URL is reachable.

## Notes

- Never print the contents of `~/.lugui/config.json` (it holds the token).
- This token authenticates every publish. If it leaks or expires, just redo the
  web login at `{pages_api}/login` to get a new one and run `/lugui-setup`.
