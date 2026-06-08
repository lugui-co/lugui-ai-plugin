---
name: data
description: Make a published Lugui page READ AND WRITE persistent data (lists, forms, dashboards that save) without any backend code. MUST be loaded whenever the user asks for a page that saves, shares, or remembers data (todo lists, sign-up sheets, trackers, CRUDs, shared boards). Covers creating the data "app", wiring lugui-data.js, ACL (private vs shared), and handling @lugui login.
---

# Lugui Data — data-backed pages

Use this when the user wants a published page that **persists data** (saves
entries, shows a shared list, has a form that remembers submissions, a tracker,
a small CRUD). The page stays a single static HTML file on `pages.lugui.ai`;
the data lives in the Pages data API and is reached from the browser with the
tiny client `lugui-data.js`. **No backend code to write.**

Always also load **`lugui-ai:branding`** and start from the branded template —
data pages must look like Lugui too (see step 3).

## The model (explain this to the user in plain terms)

- **app** — a named container the user owns. It has an **ACL**:
  - `private` → only the owner (you, signed in) can read/write.
  - `shared` → any signed-in **@lugui.ai** person can read/write.
  - The owner is the only one who can create/rename/delete the app or change its
    ACL. **Ask the user which one they want** before creating it.
- **collection** — a named bucket of items inside an app (e.g. `tarefas`,
  `inscricoes`). Just pick a name; it's created on first write.
- **item** — one JSON object (your record). The server wraps it: the response is
  `{ id, app_key, collection, data, created_by, updated_by, created_at,
  updated_at }` and **your fields live under `.data`**.
- **Last-write-wins:** `update` (PUT) overwrites the whole item; the last writer
  wins. `update` on a missing id is a 404 (create first).

## Step 1 — Create the app (one-time, with the user's token)

The app is created server-side with the user's personal token from
`~/.lugui/config.json` (the same `pages_api` + `token` used by
`/lugui-ai:pages:publish`). **Ask: private (só você) or shared (time @lugui)?**

```bash
# read pages_api + token from ~/.lugui/config.json (never print the token)
curl -sS -X POST "<pages_api>/apps" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"name":"Lista de tarefas","acl":"private"}'
# → { "app_key": "lista-de-tarefas-x7k2", "acl": "private", ... }
```

Save the returned **`app_key`** — it goes into the page. If the user has no
token yet, tell them to run `/lugui-ai:setup` first. To make an existing app
shared later: `curl -X PATCH "<pages_api>/apps/<app_key>" -H "Authorization:
Bearer <token>" -d '{"acl":"shared"}'`.

## Step 2 — Load the client library in the page

```html
<script src="https://pages-api.coolify.lugui.ai/lugui-data.js"></script>
```

This exposes `window.lugui`. Every call sends the browser session cookie
(`credentials: 'include'`), so the page works once the visitor is signed in.

### `lugui` API surface

| Call | Does |
|---|---|
| `lugui.api` | the API base URL (string) |
| `await lugui.me()` | `{ email }` if signed in, else `null` (never throws) |
| `lugui.login(returnUrl?)` | redirects the browser to sign in, returns to `returnUrl` (default: current page) |
| `await lugui.ensureAuth()` | returns the email, or redirects to login if not signed in |
| `await lugui.data.list(app, col, {limit, offset}?)` | array of items |
| `await lugui.data.get(app, col, id)` | one item |
| `await lugui.data.create(app, col, obj)` | creates an item, returns it |
| `await lugui.data.update(app, col, id, obj)` | PUT (last-write-wins; 404 if missing) |
| `await lugui.data.remove(app, col, id)` | deletes it |

On **401**, data calls throw a **`LuguiAuthError`** — they do NOT redirect
silently. The page catches it and calls `lugui.login()`. Other failures throw
`LuguiApiError` (with `.status` and `.body`).

## Step 3 — Wire the page (on-brand)

1. Load **`lugui-ai:branding`** and start from `skills/branding/template.html`
   (cream background, Plus Jakarta Sans, real `--lugui-*` tokens). Then start
   from **`skills/data/template-data.html`** in this skill, which already wires
   the list + form + login into that branding.
2. Set the `APP_KEY` and `COLLECTION` constants at the top of the script.
3. Handle login. Two valid patterns:
   - **Gate on load:** call `await lugui.ensureAuth()` first — visitors are sent
     to sign in immediately. Good for private/owner tools.
   - **Login button:** render the signed-in email or an "Entrar" button that
     calls `lugui.login()`; wrap data calls in `try/catch (LuguiAuthError)` and
     show the button when it fires. Good for shared pages where you want a
     friendly entry point.
4. Read with `lugui.data.list(...)`, write with `create`/`update`/`remove`,
   re-render after each change. Remember user fields are under `item.data`.

## What to make clear to the user

- **Both `private` and `shared` pages require a @lugui login in the browser** —
  the page is public HTML, but the *data* is protected. A logged-out visitor
  sees the login prompt, not the data.
- **`shared` means everyone on the team writes the same data** — a shared list
  is collaborative and persistent for all @lugui users. `private` is just the
  owner's.
- The page itself is still publicly reachable at its URL; only the data is
  gated. Don't put secrets in the HTML (see `security-checklist`).

## Then publish

Hand the finished, branded, data-wired HTML to **`/lugui-ai:pages:publish`**
(which runs the branding + security checks before upload).
