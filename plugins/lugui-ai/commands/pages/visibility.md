---
description: Change who can open a page you published on pages.lugui.ai — interna (só login @lugui.ai) ou pública (qualquer pessoa com o link) — without re-publishing it.
argument-hint: "<caminho> <interna|publica>"
---

# /lugui-ai:pages:visibility

Change a published page's visibility. `$ARGUMENTS` carries the page path and,
optionally, the target (e.g. `relatorios/cardinali/q2 interna` or
`maximiza-poa publica`).

- **interna** (`internal`, the default for new pages): pages.lugui.ai only
  serves it to someone logged in with a `@lugui.ai` Google account.
- **pública** (`public`): anyone with the link opens it, **source code
  included**. Only for content meant for people outside Lugui: forms, landing
  pages, terms, client links (`#/p/<token>`), images used in e-mails.

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

1. **Resolve the path.** From `$ARGUMENTS`; if empty, ask which page (offer
   `/lugui-ai:pages:list`, which shows each page's current visibility). Strip
   any leading `https://pages.lugui.ai/` and leading/trailing slashes so you
   have the bare `slug` (which may be nested with `/`).

2. **Resolve the target.** `interna` → `internal`, `publica`/`pública` →
   `public`. If missing, ask:

   > Quem pode abrir `pages.lugui.ai/<caminho>`? **Interna** (só o time, com
   > login @lugui.ai) ou **pública** (qualquer pessoa com o link)?

3. **Before making a page public, check what it exposes.** Fetch it
   (`curl -sS https://pages.lugui.ai/<caminho>`) and look at the HTML,
   including inline scripts and JSON: internal numbers (MRR, metas, salários,
   pipeline), client data or terms, keys or tokens. If any is there, tell the
   user what you found and recommend keeping it **interna** or removing that
   content first. Only go ahead on a clear confirmation.

4. **Change it** — one Bash call, block + request:

   ```sh
   # [bloco lugui-credencial v1, inteiro, aqui]
   lugui_curl -w '\nHTTP %{http_code}\n' -X PATCH -H 'Content-Type: application/json' \
     --data '{"visibility":"<internal|public>"}' "$_lg_api/pages/<caminho>"
   ```

   (Keep the `/` separators in a nested path; the route matches the full path.)

   Map the HTTP status:
   - **200** → done. Confirm in pt-BR, e.g. *"Pronto, `pages.lugui.ai/<caminho>`
     agora é **interna**: só abre com login @lugui.ai."*
   - **404** → that page doesn't exist (check the path, or run
     `/lugui-ai:pages:list`).
   - **403** → você não é o dono dessa página; peça ao dono para mudar.
   - **502** → a visibilidade foi salva, mas o pages.lugui.ai ainda não foi
     avisado. Rode o comando de novo em alguns segundos.
   - **401** → invalid/expired token → on a computer, `/lugui-ai:setup`; with
     `fonte=env` (cloud), fix the `LUGUI_TOKEN` where it is set.

Never print the token or the contents of `~/.lugui/config.json`.
