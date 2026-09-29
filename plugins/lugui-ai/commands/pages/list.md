---
description: List the pages you published to pages.lugui.ai (path, public link, type, created) using curl + your personal token.
---

# /lugui-ai:pages:list

List the published pages you own. Read-only.

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

## Flow for Claude

1. **Fetch the pages** — one Bash call, block + request:

   ```sh
   # [bloco lugui-credencial v1, inteiro, aqui]
   lugui_curl -w '\nHTTP %{http_code}\n' "$_lg_api/pages"
   ```

   The body is a JSON array; each item has:
   `slug` (the canonical path), `url` (public link), `type`
   (`permanent` | `ephemeral`), `size_bytes`, `created_at`, `updated_at`,
   and `expires_at` (only for ephemeral pages).

   - **401 / 403** → invalid/expired token → on a computer, `/lugui-ai:setup`;
     with `fonte=env` (cloud), the `LUGUI_TOKEN` in the environment must be
     fixed where it is set.

2. **Format a readable table** sorted by `updated_at` (most recent first), e.g.:

   | Caminho | Link | Tipo | Criada em |
   |---|---|---|---|
   | `relatorios/cardinali/q2` | https://pages.lugui.ai/relatorios/cardinali/q2 | permanente | 05/06/2026 |

   - Render `type` in pt-BR: `permanent` → **permanente**, `ephemeral` →
     **efêmera**. For ephemeral pages, also show `expires_at` (expira em …).
   - Format dates as `dd/mm/aaaa` (pt-BR).

3. **Empty state.** If the array is empty, say something friendly like: *"Você
   ainda não publicou nenhuma página. Crie uma com `/lugui-ai:pages:publish`."*

## Footer reminder (always show)

> Para **atualizar** uma página, rode `/lugui-ai:pages:publish` no **mesmo
> caminho** (mesmo `slug`) — isso sobrescreve o conteúdo. Para **apagar**, use
> `/lugui-ai:pages:delete <caminho>`.

Never print the token or the contents of `~/.lugui/config.json`.
