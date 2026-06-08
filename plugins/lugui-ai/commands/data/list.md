---
description: List your Lugui data apps (app_key, name, ACL, public-fill) using curl + your personal token.
---

# /lugui-ai:data:list

List the data apps you own (the backends behind data-driven pages). Read-only.

## Flow for Claude

1. **Read the config.** Load `~/.lugui/config.json` and extract `pages_api` and
   `token`. If the file is missing or either field is empty, STOP and tell the
   user to run `/lugui-ai:setup` first. Never print the token.

2. **Fetch the apps:**

   ```bash
   curl -sS -H "Authorization: Bearer <token>" "<pages_api>/apps"
   ```

   The response is a JSON array; each item has:
   `app_key`, `name`, `acl` (`private` | `shared`), `allow_public_fill`
   (bool), `max_items`, `max_bytes`, `created_at`, `updated_at`.

   - **401 / 403** → invalid/expired token → tell the user to run
     `/lugui-ai:setup`.

3. **Format a readable table**, e.g.:

   | App key | Nome | ACL | Link público |
   |---|---|---|---|
   | `implantacao` | Implantação | private | sim |

   - `acl`: `private` (só você) / `shared` (qualquer @lugui).
   - `allow_public_fill`: render as **sim/não** ("Link público" = permite links
     de preenchimento por registro `#/p/<token>`).

4. **Empty state.** If empty: *"Você ainda não tem apps de dados. Eles são
   criados automaticamente quando você publica uma página com dados — veja a
   skill `lugui-ai:data`."*

## Reminder (always show)

> Os dados de cada app ficam em coleções, em `/data/{app_key}/{collection}`.
> Apps de dados alimentam páginas com `lugui.data.*` (interno @lugui) e, quando
> `allow_public_fill` está ligado, links públicos de preenchimento por registro.

Never print the token or the contents of `~/.lugui/config.json`.
