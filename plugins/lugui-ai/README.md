# lugui-ai (Claude Code plugin)

**lugui-ai** is Lugui's internal productivity suite for Claude Code — a growing
set of features behind a single plugin. The first feature is **page
publishing** (generate on-brand, secret-scanned HTML and publish it to a public
link at `pages.lugui.ai/<slug>`, backed by the `pages-publisher` service in this
repo). More will follow (pages → skills → …), all sharing the same one-time
web-login auth.

> **Self-service, no admin, no Python.** Publishing is just Claude running
> `curl`. Authentication is a **personal access token** (`lgp_...`) you get
> yourself via **web login** (`{pages_api}/login` — sign in with @lugui.ai and
> copy the token shown), pasted once into `~/.lugui/config.json`. The plugin
> ships only command instructions and skills — there is no executable code to
> install.

## What you get

- **Commands**
  - `/lugui-ai:setup` — one-time auth for the whole suite: web login, paste the
    `lgp_...` token, saved to `~/.lugui/config.json` and validated.
  - `/lugui-ai:pages:publish <arquivo.html>` — loads the security skill, asks
    permanent/ephemeral (+ optional canonical path), sanity-checks for secrets,
    then publishes via `curl`.
- **Skills** (auto-applied by Claude when relevant)
  - `branding` — Lugui design tokens + a copyable on-brand HTML template.
  - `code-best-practices` — semantic, accessible, self-contained, CSP-friendly HTML/CSS/JS.
  - `security-checklist` — what must never go on a public page.

## Install

From a Claude Code session, add the public mirror as a marketplace and install
the plugin:

```
/plugin marketplace add lugui-co/lugui-ai-plugin
/plugin install lugui-ai@lugui-marketplace
```

For local development against a checkout, point the marketplace at the path:

```
/plugin marketplace add /absolute/path/to/lugui-ai-powered
/plugin install lugui-ai@lugui-marketplace
```

That's the whole install — no `pip`, no dependencies. Then run `/lugui-ai:setup`.

## Setup (web login → token)

Run `/lugui-ai:setup`. Claude opens the login URL — **`{pages_api}/login`**
(default `https://pages-api.coolify.lugui.ai/login`) — in your browser. Sign in
with your **@lugui.ai** account, copy the `lgp_...` token shown, and paste it
back. Claude writes `~/.lugui/config.json` and validates the token against the
API. It's fully self-service — no admin issues the token.

`~/.lugui/config.json` looks like:

```json
{
  "pages_api": "https://pages-api.coolify.lugui.ai",
  "token": "lgp_..."
}
```

The file is written with mode `600`. Never commit or share it — it holds your
token. If the token leaks or expires, just redo the web login at
`{pages_api}/login` to get a fresh one and run `/lugui-ai:setup` again.

## Usage

```
/lugui-ai:setup                          # once per machine — web login, paste lgp_ token
/lugui-ai:pages:publish ./landing.html   # asks permanent/ephemeral, checks, publishes
```

## How publishing works

`POST {pages_api}/pages` with `Authorization: Bearer <lgp_token>` and body
`{ "html", "type", "slug" }`; response `{ url, slug, type, expires_at }`.

Claude reads the HTML, asks permanent vs ephemeral, runs a quick secret sanity
check, writes the JSON body (with the HTML safely escaped) to a temp file, and
POSTs it with `curl --data @<tmpfile>`. For **permanent** pages you choose the
canonical URL — including nested paths like `relatorios/cardinali/q2` →
`https://pages.lugui.ai/relatorios/cardinali/q2` (leave it empty to auto-derive
from the title); it goes in the `slug` field. Server errors
(401/403/409/413/422) are mapped to clear messages. Validation (size ≤ 2 MiB,
looks-like-HTML, secret scan) is enforced server-side; the local check is a
fast courtesy gate.
