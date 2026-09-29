---
description: List your Lugui data apps (app_key, name, ACL, public-fill) using curl + your personal token.
---

# /lugui-ai:data:list

List the data apps you own (the backends behind data-driven pages). Read-only.

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

1. **Fetch the apps** — one Bash call, block + request:

   ```sh
   # [bloco lugui-credencial v1, inteiro, aqui]
   lugui_curl -w '\nHTTP %{http_code}\n' "$_lg_api/apps"
   ```

   The body is a JSON array; each item has:
   `app_key`, `name`, `acl` (`private` | `shared`), `allow_public_fill`
   (bool), `max_items`, `max_bytes`, `created_at`, `updated_at`.

   - **401 / 403** → invalid/expired token → on a computer, `/lugui-ai:setup`;
     with `fonte=env` (cloud), fix the `LUGUI_TOKEN` where it is set.

2. **Format a readable table**, e.g.:

   | App key | Nome | ACL | Link público |
   |---|---|---|---|
   | `implantacao` | Implantação | private | sim |

   - `acl`: `private` (só você) / `shared` (qualquer @lugui).
   - `allow_public_fill`: render as **sim/não** ("Link público" = permite links
     de preenchimento por registro `#/p/<token>`).

3. **Empty state.** If empty: *"Você ainda não tem apps de dados. Eles são
   criados automaticamente quando você publica uma página com dados — veja a
   skill `lugui-ai:data`."*

## Reminder (always show)

> Os dados de cada app ficam em coleções, em `/data/{app_key}/{collection}`.
> Apps de dados alimentam páginas com `lugui.data.*` (interno @lugui) e, quando
> `allow_public_fill` está ligado, links públicos de preenchimento por registro.

Never print the token or the contents of `~/.lugui/config.json`.
