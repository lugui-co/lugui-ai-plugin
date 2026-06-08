---
description: List the pages you published to pages.lugui.ai (path, public link, type, created) using curl + your personal token.
---

# /lugui-ai:pages:list

List the published pages you own. Read-only.

## Flow for Claude

1. **Read the config.** Load `~/.lugui/config.json` and extract `pages_api` and
   `token`. If the file is missing or either field is empty, STOP and tell the
   user to run `/lugui-ai:setup` first. Never print the token.

2. **Fetch the pages:**

   ```bash
   curl -sS -H "Authorization: Bearer <token>" "<pages_api>/pages"
   ```

   The response is a JSON array; each item has:
   `slug` (the canonical path), `url` (public link), `type`
   (`permanent` | `ephemeral`), `size_bytes`, `created_at`, `updated_at`,
   and `expires_at` (only for ephemeral pages).

   - **401 / 403** → invalid/expired token → tell the user to run
     `/lugui-ai:setup`.

3. **Format a readable table** sorted by `updated_at` (most recent first), e.g.:

   | Caminho | Link | Tipo | Criada em |
   |---|---|---|---|
   | `relatorios/cardinali/q2` | https://pages.lugui.ai/relatorios/cardinali/q2 | permanente | 05/06/2026 |

   - Render `type` in pt-BR: `permanent` → **permanente**, `ephemeral` →
     **efêmera**. For ephemeral pages, also show `expires_at` (expira em …).
   - Format dates as `dd/mm/aaaa` (pt-BR).

4. **Empty state.** If the array is empty, say something friendly like: *"Você
   ainda não publicou nenhuma página. Crie uma com `/lugui-ai:pages:publish`."*

## Footer reminder (always show)

> Para **atualizar** uma página, rode `/lugui-ai:pages:publish` no **mesmo
> caminho** (mesmo `slug`) — isso sobrescreve o conteúdo. Para **apagar**, use
> `/lugui-ai:pages:delete <caminho>`.

Never print the token or the contents of `~/.lugui/config.json`.
