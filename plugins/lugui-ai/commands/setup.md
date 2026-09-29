---
description: One-time setup for the lugui-ai suite — reuses LUGUI_TOKEN when it is already in the environment, otherwise logs in on the web with your @lugui.ai account and saves the token straight from the clipboard (never pasted in chat). In cloud sessions and routines it explains the connector / environment-variable path instead of minting yet another token. Self-service, no admin, no Python.
---

# /lugui-ai:setup

Authenticate the **lugui-ai** suite (page publishing, data apps, videos). The
credential is a **personal access token** (`lgp_...`) obtained by **web login**
with a **@lugui.ai** account. It can live in two places:

- **`LUGUI_TOKEN` in the environment** — the only option in a cloud session
  (routines, claude.ai/code), in CI and on machines without a browser. It also
  overrides the file when both exist. (`LUGUI_PAGES_TOKEN` is an alias;
  `LUGUI_PAGES_API` overrides the API address.)
- **`~/.lugui/config.json`** — the normal case on your own computer, written by
  this command with mode `600`.

The token value must never show up in chat, in a command, or in a process
argument list. That is what the canonical block below guarantees.

## Credential block (canonical — identical in every lugui-ai command)

The Bash tool keeps no shell state between calls, so **every Bash call that
talks to the API starts with this block, pasted as is**, and then uses
`lugui_curl` instead of `curl`. **NEVER write the token value in a command;
always use `lugui_curl`.** Never open `~/.lugui/config.json` with the Read tool
and never print it.

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

- `$_lg_src` records where the token came from: `env` or `file` (the value is
  never printed). `$_lg_api` is the API base URL.
- Exit **3** = no credential; exit **2** = invalid configuration (API address
  that is not `https://`, or a token with unexpected characters). In both cases
  the block already printed a pt-BR explanation.
- `lugui_curl` is `curl -sS` with the `Authorization` header handed over on
  stdin (`-K -`), so it takes every normal curl option. Don't feed it stdin
  (`@-`), and only point it at `$_lg_api`.

## Steps for Claude (do these IN ORDER)

1. **Is there a credential already? Validate it before any login.**

   ```sh
   # [bloco lugui-credencial v1, inteiro, aqui]
   lugui_curl -o /tmp/lugui-me.json -w 'HTTP %{http_code}\n' "$_lg_api/me"
   echo "fonte=$_lg_src api=$_lg_api"; cat /tmp/lugui-me.json; echo; rm -f /tmp/lugui-me.json
   ```

   - **HTTP 200** → already configured. Tell the person who they are (the
     `email` in the response) and where the credential came from (`fonte=env` →
     *"usando o LUGUI_TOKEN do ambiente"*; `fonte=file` → *"usando o token salvo
     neste computador"*). **STOP** — do not log in again. Every login mints a
     brand-new token that never expires; logging in "just in case" is how tokens
     pile up.
   - **HTTP 401 with `fonte=env`** → the `LUGUI_TOKEN` in the environment is
     wrong or was revoked. Tell the person to fix it **where it is set** (in a
     cloud session: the environment's variables, step 3 option B). **STOP** —
     a login here would not fix the environment variable.
   - **HTTP 401 with `fonte=file`** → the saved token is dead. Continue to
     step 2 (the new login replaces it).
   - **Exit 3 (no credential)** → expected on a first setup. Don't repeat the
     block's message to the person; continue to step 2.
   - **`403` with `host_not_allowed`, or `CONNECT tunnel failed, response 403`**
     → a cloud session whose network blocks the API. Explain step 3, option B,
     items 4–5 (or option A, which needs no network change). **STOP.**
   - **Exit 2** → relay the block's message and **STOP**.

2. **Where are we? Only a computer with a browser does the web login.**

   ```sh
   if [ "${CLAUDE_CODE_REMOTE:-}" = true ]; then echo modo=nuvem
   elif [ "$(uname -s 2>/dev/null)" = Darwin ]; then echo modo=mac
   elif command -v cmd.exe >/dev/null 2>&1 || [ -r /dev/clipboard ]; then echo modo=windows
   elif { [ -n "${DISPLAY:-}" ] || [ -n "${WAYLAND_DISPLAY:-}" ]; } && command -v xdg-open >/dev/null 2>&1; then echo modo=linux
   else echo modo=sem-navegador; fi
   ```

   - `modo=nuvem` or `modo=sem-navegador` → step 3. **Do NOT run the web login
     here**: the token would have to travel through the chat, and it would be
     one more never-expiring token for a session that disappears.
   - `modo=mac`, `modo=windows`, `modo=linux` → step 4.

3. **Cloud session (routine, claude.ai/code) or no browser: explain, don't mint.**

   Say this to the person (pt-BR, adapt lightly), then **STOP**:

   > Esta sessão roda na nuvem (ou numa máquina sem navegador). Aqui não existe
   > o arquivo `~/.lugui` do seu computador, e fazer login de novo a cada sessão
   > só cria mais um token. Você configura **uma vez só**, de um destes jeitos:
   >
   > **A) Só para publicar páginas — o conector Lugui Pages** (não precisa de
   > token nem de mexer em rede; por enquanto ele **não** lê nem grava dados de
   > apps, então rotina que mexe em ideias, cadastros ou listas usa a opção B):
   > 1. Abra **claude.ai/customize/connectors**.
   > 2. Clique em **Adicionar conector personalizado** (*Add custom connector*).
   > 3. Nome: `Lugui Pages`. URL: `https://pages-api.coolify.lugui.ai/mcp`. Salve.
   > 4. Clique em **Conectar** e entre com a sua conta **@lugui.ai**.
   > 5. Em **claude.ai/code/routines**, abra a rotina → **Editar** → em
   >    **Conectores**, confira que **Lugui Pages** está incluído → salve.
   >
   > **B) Para rotinas que leem ou gravam dados — um token fixo no ambiente da
   > nuvem:**
   > 1. No seu computador, abra `https://pages-api.coolify.lugui.ai/login`, entre
   >    com a conta @lugui.ai e copie o token. Esse token fica **só para a
   >    rotina**; não gere outro a cada sessão.
   > 2. Em **claude.ai/code/routines**, abra a rotina → **Editar**. Embaixo das
   >    instruções, clique no ícone de nuvem com o nome do ambiente → passe o
   >    mouse no ambiente → clique na engrenagem → **Update cloud environment**.
   > 3. Em **Environment variables**, acrescente uma linha: `LUGUI_TOKEN=` seguido
   >    do token (sem espaços, sem aspas).
   > 4. Em **Network access**, escolha **Custom**. Em **Allowed domains**,
   >    escreva `pages-api.coolify.lugui.ai`. Deixe marcado **Also include default
   >    list of common package managers**.
   > 5. Clique em **Save changes**. Vale a partir da próxima execução.
   >
   > Cuidado: quem usar esse mesmo ambiente consegue ler a variável. Use um
   > ambiente só seu. **Nunca copie a pasta `~/.lugui` para um repositório nem
   > para a nuvem.**

   **Machine without a browser that you control (a server over SSH):** option B
   also works there — `LUGUI_TOKEN` exported in that machine's shell profile — or
   the person logs in on their own computer, copies the token and runs this **in
   the server's terminal, not in the chat** (what they type is not shown):

   ```sh
   (umask 077; mkdir -p ~/.lugui; printf 'Cole o token lgp_ (não aparece na tela) e aperte Enter: '; stty -echo; IFS= read -r t; stty echo; echo; printf '{\n  "pages_api": "https://pages-api.coolify.lugui.ai",\n  "token": "%s"\n}\n' "$t" > ~/.lugui/config.json && chmod 600 ~/.lugui/config.json && echo "Token salvo.")
   ```

4. **Open the login page in the browser** (`$_lg_api` is
   `https://pages-api.coolify.lugui.ai` unless `LUGUI_PAGES_API` or an existing
   config says otherwise):

   ```sh
   open "https://pages-api.coolify.lugui.ai/login"            # modo=mac
   xdg-open "https://pages-api.coolify.lugui.ai/login"        # modo=linux
   start "" "https://pages-api.coolify.lugui.ai/login"        # modo=windows (Git Bash)
   cmd.exe /c start "" "https://pages-api.coolify.lugui.ai/login"   # modo=windows (WSL)
   ```

   If the opener fails, **don't error** — show the URL. Then tell the person:

   > Abri o login no navegador. Entre com a sua conta **@lugui.ai** e clique em
   > **Copy** ao lado do token. **Não precisa colar o token aqui** — é só me
   > avisar ("copiei") que eu leio direto da área de transferência.
   > _(Se não abriu sozinho: `https://pages-api.coolify.lugui.ai/login`)_

5. **Save the token straight from the clipboard.** Run this after the person
   says they copied it:

   ```sh
   _lg_new=
   if command -v pbpaste >/dev/null 2>&1; then _lg_new=$(pbpaste 2>/dev/null)
   elif [ -r /dev/clipboard ]; then _lg_new=$(cat /dev/clipboard 2>/dev/null)
   elif command -v wl-paste >/dev/null 2>&1; then _lg_new=$(wl-paste -n 2>/dev/null)
   elif command -v xclip >/dev/null 2>&1; then _lg_new=$(xclip -o -selection clipboard 2>/dev/null)
   elif command -v xsel >/dev/null 2>&1; then _lg_new=$(xsel -ob 2>/dev/null)
   elif command -v powershell.exe >/dev/null 2>&1; then _lg_new=$(powershell.exe -NoProfile -Command Get-Clipboard 2>/dev/null)
   else echo "lugui: não consegui ler a área de transferência neste computador."; exit 4
   fi
   _lg_new=$(printf '%s' "$_lg_new" | tr -d ' \t\r\n')
   case "$_lg_new" in lgp_*) ;; *) echo "lugui: a área de transferência não tem um token lgp_."; exit 5 ;; esac
   case "$_lg_new" in *[!A-Za-z0-9._-]*) echo "lugui: o conteúdo copiado não parece um token."; exit 5 ;; esac
   _lg_keep=$(sed -n 's/.*"pages_api"[[:space:]]*:[[:space:]]*"\([^"]*\)".*/\1/p' "$HOME/.lugui/config.json" 2>/dev/null | head -n 1)
   _lg_keep="${_lg_keep:-https://pages-api.coolify.lugui.ai}"
   umask 077 && mkdir -p "$HOME/.lugui" && chmod 700 "$HOME/.lugui" \
     && printf '{\n  "pages_api": "%s",\n  "token": "%s"\n}\n' "$_lg_keep" "$_lg_new" > "$HOME/.lugui/config.json.tmp" \
     && mv "$HOME/.lugui/config.json.tmp" "$HOME/.lugui/config.json" && chmod 600 "$HOME/.lugui/config.json" \
     && echo "lugui: token salvo em ~/.lugui/config.json (só você lê)."
   if command -v pbcopy >/dev/null 2>&1; then printf '' | pbcopy
   elif [ -w /dev/clipboard ]; then printf '' > /dev/clipboard
   elif command -v wl-copy >/dev/null 2>&1; then wl-copy --clear 2>/dev/null
   elif command -v xclip >/dev/null 2>&1; then printf '' | xclip -selection clipboard
   elif command -v xsel >/dev/null 2>&1; then xsel -cb
   fi
   ```

   Why this is the least leaky option available: the token goes from the
   clipboard to a shell variable to the file. It never passes through the chat,
   it is not in the command text (so not in the transcript), and it is not in
   any process argument list (`printf` is a shell builtin). The file is created
   already private (`umask 077`, then `chmod 600`) and swapped in with `mv`, and
   the clipboard is cleared right after (tell the person, so they aren't
   surprised). An existing `pages_api` is kept.

   - **Exit 5** → the clipboard doesn't hold a token: ask the person to click
     **Copy** again and tell you.
   - **Exit 4** → no clipboard tool: ask the person to run the one-liner from
     step 3 ("Machine without a browser") **in their own terminal**.
   - **If the person pasted the token in the chat anyway:** do not repeat it and
     do not write it in a command. It is still in their clipboard — run this
     same step.

6. **Validate** with the command from step 1.

   - **HTTP 200** → done. Tell the person: *"Pronto, configurado como
     `<email>`. Agora é só usar `/lugui-ai:pages:publish ./pagina.html`."*
   - **HTTP 401** → wrong or dead token: reopen the login (step 4), copy a fresh
     token and repeat step 5. Do NOT keep a bad token.
   - **Connection error** → check the API address (`$_lg_api`) is reachable.

## Notes

- Never print, `cat` or Read `~/.lugui/config.json`; never echo the token, not
  even partially.
- One token per computer is enough. The web login mints a new token every time
  and tokens don't expire, so don't log in again when step 1 says things work.
  If a token leaks, ask the tech team to revoke it and run `/lugui-ai:setup`.
- Plugins installed on a computer don't exist in cloud sessions. Routines that
  read or write data apps use `LUGUI_TOKEN` in the cloud environment (option B);
  the **Lugui Pages connector** (option A) only publishes pages for now.
