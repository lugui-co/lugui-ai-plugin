---
description: Permanently delete a page you published on pages.lugui.ai. Asks for confirmation first.
argument-hint: "<caminho>"
---

# /lugui-ai:pages:delete

Permanently delete a published page. The page path (slug) to delete is in
`$ARGUMENTS` (e.g. `relatorio-cardinali` or `relatorios/cardinali/q2`).

## Credential block (paste at the top of EVERY Bash call that talks to the API)

The Bash tool keeps no shell state between calls, so the block goes into the
same call as the request. **NEVER write the token value in a command; always use
`lugui_curl`.** Never open `~/.lugui/config.json` with the Read tool.

```sh
# --- lugui-credencial v1: cole IGUAL no topo de cada chamada Bash que fala com a API ---
# NUNCA escreva o valor do token num comando: use sempre lugui_curl (o token vai por stdin, fora do argv).
_lg_tok="${LUGUI_TOKEN:-${LUGUI_PAGES_TOKEN:-}}"; _lg_api="${LUGUI_PAGES_API:-}"; _lg_src=none
[ -n "$_lg_tok" ] && _lg_src=env
_lg_cfg="$HOME/.lugui/config.json"
if [ -r "$_lg_cfg" ]; then
  if [ -z "$_lg_tok" ]; then
    _lg_tok=$(sed -n 's/.*"token"[[:space:]]*:[[:space:]]*"\([^"]*\)".*/\1/p' "$_lg_cfg" | head -n 1)
    [ -n "$_lg_tok" ] && _lg_src=file
  fi
  [ -z "$_lg_api" ] && _lg_api=$(sed -n 's/.*"pages_api"[[:space:]]*:[[:space:]]*"\([^"]*\)".*/\1/p' "$_lg_cfg" | head -n 1)
fi
_lg_api="${_lg_api:-https://pages-api.coolify.lugui.ai}"; _lg_api="${_lg_api%/}"
case "$_lg_api" in https://*|http://127.0.0.1:*|http://localhost:*) ;;
  *) printf 'lugui: endereço da API recusado (%s): precisa ser https://\n' "$_lg_api" >&2; exit 2 ;; esac
case "$_lg_tok" in *[!A-Za-z0-9._-]*)
  printf 'lugui: o token (fonte: %s) tem caracteres inesperados; gere outro.\n' "$_lg_src" >&2; exit 2 ;; esac
if [ -z "$_lg_tok" ]; then
  if [ "${CLAUDE_CODE_REMOTE:-}" = true ]; then
    printf '%s\n' "lugui: sessão na nuvem (rotina ou claude.ai/code) sem credencial. Aqui não existe ~/.lugui e rodar /lugui-ai:setup não resolve (cada login cria mais um token)." \
      "Configure uma vez: LUGUI_TOKEN nas variáveis do ambiente de nuvem + Network access Custom liberando pages-api.coolify.lugui.ai." \
      "Só para publicar páginas, o conector Lugui Pages (https://pages-api.coolify.lugui.ai/mcp) também serve; ele ainda não lê nem grava dados de apps." >&2
  else
    printf '%s\n' "lugui: não achei credencial (nem LUGUI_TOKEN no ambiente, nem ~/.lugui/config.json)." \
      "Neste computador: rode /lugui-ai:setup uma vez. Sessão sem navegador (servidor, CI): defina LUGUI_TOKEN no ambiente." >&2
  fi
  exit 3
fi
lugui_curl() { printf 'header = "Authorization: Bearer %s"\n' "$_lg_tok" | curl -sS -K - "$@"; }
# --- fim lugui-credencial ---
```

- Exit **3** (no credential) or **2** (invalid configuration): relay the block's
  pt-BR message to the person and **STOP**. It already tells a computer
  (`/lugui-ai:setup`) apart from a cloud session (connector or `LUGUI_TOKEN`).
- `403` with `host_not_allowed`, or `CONNECT tunnel failed, response 403`: a
  cloud session whose network blocks the API → point the person to
  `/lugui-ai:setup`'s cloud instructions (connector, or Network access
  **Custom** allowing `pages-api.coolify.lugui.ai`).

## Flow for Claude (do these IN ORDER)

1. **Resolve the path.** Use `$ARGUMENTS`. If empty, ask which page (offer
   `/lugui-ai:pages:list` to see the options). Strip any leading
   `https://pages.lugui.ai/` and leading/trailing slashes so you have the bare
   `slug` (which may be nested with `/`).

2. **Confirm with the user before deleting** (destructive, irreversible):

   > Apagar `pages.lugui.ai/<caminho>`? Isso é **permanente** e o link para de
   > funcionar. (s/N)

   Only proceed if the user clearly confirms. Otherwise abort.

3. **Delete** — one Bash call, block + request:

   ```sh
   # [bloco lugui-credencial v1, inteiro, aqui]
   lugui_curl -o /dev/null -w 'HTTP %{http_code}\n' -X DELETE "$_lg_api/pages/<caminho>"
   ```

   (Keep the `/` separators in a nested path; the route matches the full path.)

   Map the HTTP status:
   - **204** → deleted. Confirm: *"Pronto, `pages.lugui.ai/<caminho>` foi
     apagada."*
   - **404** → that page doesn't exist (check the path, or run
     `/lugui-ai:pages:list`).
   - **403** → você não é o dono dessa página, não dá pra apagar.
   - **401** → invalid/expired token → on a computer, `/lugui-ai:setup`; with
     `fonte=env` (cloud), fix the `LUGUI_TOKEN` where it is set.

Never print the token or the contents of `~/.lugui/config.json`.
