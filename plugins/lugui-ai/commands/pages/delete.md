---
description: Permanently delete a page you published on pages.lugui.ai. Asks for confirmation first.
argument-hint: "<caminho>"
---

# /lugui-ai:pages:delete

Permanently delete a published page. The page path (slug) to delete is in
`$ARGUMENTS` (e.g. `relatorio-cardinali` or `relatorios/cardinali/q2`).

## Flow for Claude (do these IN ORDER)

1. **Read the config.** Load `~/.lugui/config.json` and extract `pages_api` and
   `token`. If the file is missing or either field is empty, STOP and tell the
   user to run `/lugui-ai:setup` first. Never print the token.

2. **Resolve the path.** Use `$ARGUMENTS`. If empty, ask which page (offer
   `/lugui-ai:pages:list` to see the options). Strip any leading
   `https://pages.lugui.ai/` and leading/trailing slashes so you have the bare
   `slug` (which may be nested with `/`).

3. **Confirm with the user before deleting** (destructive, irreversible):

   > Apagar `pages.lugui.ai/<caminho>`? Isso é **permanente** e o link para de
   > funcionar. (s/N)

   Only proceed if the user clearly confirms. Otherwise abort.

4. **Delete:**

   ```bash
   curl -sS -o /dev/null -w "%{http_code}" -X DELETE \
     -H "Authorization: Bearer <token>" \
     "<pages_api>/pages/<caminho>"
   ```

   (Keep the `/` separators in a nested path; the route matches the full path.)

   Map the HTTP status:
   - **204** → deleted. Confirm: *"Pronto, `pages.lugui.ai/<caminho>` foi
     apagada."*
   - **404** → that page doesn't exist (check the path, or run
     `/lugui-ai:pages:list`).
   - **403** → você não é o dono dessa página, não dá pra apagar.
   - **401** → invalid/expired token → run `/lugui-ai:setup`.

Never print the token or the contents of `~/.lugui/config.json`.
