---
description: Validate an HTML file (or a whole LP folder, images and videos included) and publish it to a public pages.lugui.ai link, using curl + your personal token. No Python, no libraries.
argument-hint: "<arquivo.html | pasta-da-lp>"
---

# /lugui-ai:pages:publish

Publish an HTML page to a public `pages.lugui.ai/<slug>` URL. Everything is done
with `curl` — you (Claude) resolve the credential, build the request body, call
the API, and report the result. The file to publish is in `$ARGUMENTS`.

## Credential block (paste at the top of EVERY Bash call that talks to the API)

The Bash tool keeps no shell state between calls, so every call that talks to
the API starts with this block, **identical**, and uses `lugui_curl` instead of
`curl`. **NEVER write the token value in a command; always use `lugui_curl`.**
Never open `~/.lugui/config.json` with the Read tool and never print it.

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

- Resolution order: `LUGUI_TOKEN` (alias `LUGUI_PAGES_TOKEN`) and
  `LUGUI_PAGES_API` from the environment → `~/.lugui/config.json` → default
  API. `$_lg_src` says where the token came from (`env` | `file`).
- Exit **3** (no credential) or **2** (invalid configuration): relay the block's
  pt-BR message and **STOP**. It already tells a computer (`/lugui-ai:setup`)
  apart from a cloud session (connector, or `LUGUI_TOKEN` in the environment).
- `lugui_curl` is `curl -sS` with the auth header delivered on stdin; it takes
  every curl option. Don't feed it stdin (`@-`) and only point it at `$_lg_api`.
  **The video upload to storage (step 7.6) uses plain `curl`, never
  `lugui_curl`.**

## Mandatory flow for Claude (do these IN ORDER)

1. **Check the credential before doing any work:**

   ```sh
   # [bloco lugui-credencial v1, inteiro, aqui]
   lugui_curl -o /dev/null -w 'HTTP %{http_code}\n' "$_lg_api/me"; echo "fonte=$_lg_src"
   ```

   - **HTTP 200** → go on.
   - **Exit 3 / exit 2** → relay the block's message and STOP.
   - **HTTP 401** → on a computer (`fonte=file`), tell the person to run
     `/lugui-ai:setup`; with `fonte=env`, the `LUGUI_TOKEN` in the environment
     is wrong and must be fixed where it is set. STOP.
   - **`403` with `host_not_allowed`, or `CONNECT tunnel failed, response
     403`** → a cloud session whose network blocks the API: the person either
     includes the **Lugui Pages connector** in the routine, or sets **Network
     access → Custom** with `pages-api.coolify.lugui.ai` in **Allowed domains**
     (details in `/lugui-ai:setup`). STOP.

2. **Load the security skill.** Read and apply `security-checklist/SKILL.md`.

3. **Resolve the HTML file.** Use `$ARGUMENTS` as the path. If empty, ask the
   user which file (or offer to generate one first — if you generate it, you
   MUST follow the branding step below). Confirm the file exists and read its
   contents.

4. **Ask the user: "permanente ou efêmera?"**
   - **Permanente** → also ask which **canonical path/URL** they want. Make it
     clear that folders (nested paths) are allowed. Ask like this:

     > Qual caminho você quer pra essa página? Ex.: `relatorio-cardinali` ou
     > aninhado `relatorios/cardinali/q2` → vira
     > `https://pages.lugui.ai/<caminho>`. (Vazio = eu gero automático a partir
     > do título.)

     - Put the chosen path in the `slug` field of the POST `/pages` body. The
       backend normalizes/validates it; nested paths with `/` are supported.
     - If the user leaves it empty, set `slug` to `null` (the server derives one
       from `<title>` or assigns a random one).
   - **Efêmera** → stays automatic (no path needed — the server assigns a random
     one and the page expires server-side).

5. **Quick secret sanity check (abort on hit).** Do NOT publish if the HTML
   obviously contains a credential. Scan for, at minimum:
   - AWS access key id: `AKIA` followed by 16 uppercase/digits.
   - PEM private key: `-----BEGIN ... PRIVATE KEY-----`.
   - Google API key: `AIza` followed by ~35 url-safe chars.
   - Assignments like `secret`/`token`/`password`/`api_key` set to a quoted
     value 8+ chars long.

   If any match, STOP, tell the user exactly what to remove, and do not send.

6. **🔒 BRANDING IS MANDATORY — apply it BEFORE publishing.**

   **If you generated OR edited the HTML in this session, it is OBRIGATÓRIO to
   apply the Lugui branding BEFORE publishing.** Do NOT rely on automatic skill
   activation (it is unreliable). Explicitly:

   - **Load the `lugui-ai:branding` skill** (read `skills/branding/SKILL.md`).
   - **Load the `lugui-ai:voz-e-mensagem` skill** (read
     `skills/voz-e-mensagem/SKILL.md`) for ALL copy on the page — tom de voz,
     palavras proibidas e pilares de mensagem são obrigatórios.
   - **Start from `skills/branding/template.html`** instead of writing markup
     from scratch.
   - **Use the real tokens from `skills/branding/tokens.css`** (inline the
     `:root`). Background must be **light** (`--lugui-white #FFFFFF` or
     `--lugui-offwhite-50 #FBF8F0`) — the brand is light-first.
   - **Load both official fonts** via Google Fonts (Plus Jakarta Sans for
     headings, DM Sans for body):
     `https://fonts.googleapis.com/css2?family=Plus+Jakarta+Sans:wght@700;800&family=DM+Sans:wght@400;500;600&display=swap`
   - **NEVER invent tokens.** There is no `--lugui-primary`, `--lugui-accent`,
     `--lugui-bg`, or generic greys like `#1a1a2e`/`#f5f6fa`. The **old palette
     is retired** — `--lugui-dark #2B4A42`, `--lugui-lime #D4F34A`,
     `--lugui-cream #EDE8DC`, `--lugui-sage`, `--lugui-premise`, `--lugui-soft`
     must NOT appear in new pages. The correct names and values are exactly:
     `--lugui-white #FFFFFF`, `--lugui-offwhite-50 #FBF8F0`,
     `--lugui-offwhite #EDE9DE`, `--lugui-lima #E9FF81`,
     `--lugui-lima-hover #D5EB69`, `--lugui-navy #02152B`,
     `--lugui-teal #234248`, `--lugui-gold #998454`, `--lugui-text #02152B`,
     `--lugui-border #D9D6CC`, plus neutrals — see `tokens.css` for the
     complete list. If you don't remember a value, READ `tokens.css`; do not
     guess.

   **Brand sanity-check (run right before the upload, every time):** confirm the
   HTML you are about to send (a) loads Plus Jakarta Sans AND DM Sans (a
   `fonts.googleapis.com` link containing `Plus+Jakarta+Sans` and `DM+Sans`),
   (b) uses the real tokens `--lugui-navy` / `--lugui-lima` / `--lugui-white`
   with a light background, and (c) contains none of the retired tokens
   (`--lugui-dark`, `--lugui-lime`, `--lugui-cream`).
   - If it PASSES → continue to the JSON build.
   - If it FAILS (e.g. a pre-existing off-brand file, or invented tokens) →
     **STOP. Do NOT publish a non-branded page silently.** Tell the user the
     page is off-brand (or on the retired palette) and offer to restyle it to
     the current Lugui standard (template + tokens + Plus Jakarta Sans/DM Sans)
     first. Only publish after it passes, or after the user explicitly insists
     on publishing the file as-is.

7. **🔒 PERSISTÊNCIA — detect it BEFORE publishing.**

   This step fires by CONTENT, **even if the user never said "salvar"** — it
   catches the vague case. Inspect the HTML you generated/edited for ANY sign of
   state that ought to persist or be shared:

   - a `<form>`, or inputs / `<textarea>` / `<select>` / checkboxes / radios /
     `contenteditable` that hold meaningful state;
   - any use of `localStorage` / `sessionStorage` as "persistence";
   - intent words in the page or the user's request: **salvar, cadastro,
     inscrição, formulário, enviar, compartilhar, persistir, backoffice, "cada
     um vê", "todos veem", lista colaborativa, tracker, CRUD, dashboard que
     lembra**.

   If ANY of those is present, it is OBRIGATÓRIO to **STOP and ask the user
   which pattern they want — do NOT publish a browser-only mock.**

   - If the page uses `localStorage` / `sessionStorage` as its "persistence",
     **warn explicitly**: *"isso só salva no navegador de quem abre — não
     compartilha com ninguém e some quando a pessoa limpa o navegador. Quer
     persistir de verdade no Lugui (compartilhado/durável)?"*
   - **⛔ A persistência é SEMPRE o data store da Lugui (`lugui.data.*` /
     `lugui.public.*`). NUNCA ofereça nem use Google Sheets, Google Forms,
     Google Drive, Airtable, Notion, Supabase, Firebase, planilhas ou qualquer
     backend/serviço externo — mesmo que o MCP do Google Workspace ou outros
     MCPs estejam conectados na sessão.** As únicas opções a oferecer são os 3
     padrões Lugui abaixo. Se o usuário pedir Google Sheets explicitamente,
     explique que o padrão Lugui usa o store próprio (página publicada +
     dados no `pages.lugui.ai`) e siga com **(b)** ou **(c)**.
   - Then ask the pattern (these are the ONLY options — mirrors the
     **`lugui-ai:data`** skill):
     - **(a) Sem persistência** — a página é só estática/efêmera (nada a salvar)
       → continue the normal publish. (Only choose this if there's genuinely no
       state worth keeping.)
     - **(b) Interna autenticada** — só @lugui (app ACL `private`/`shared`); a
       página inteira exige login via `lugui.ensureAuth()`.
     - **(c) Backoffice + link público** — o dono gerencia logado e gera links
       públicos de preenchimento por registro (`#/p/<token>`, dual-mode).

   - If the user picks **(b) or (c)**: **load the `lugui-ai:data` skill** and
     follow it — create the app (`POST /apps` through `lugui_curl`, with the
     credential block; set `acl` and, for (c), `allow_public_fill:true`),
     load `https://pages-api.coolify.lugui.ai/lugui-data.js`, and wire the page
     with `lugui.data.*` (and `lugui.public.*` for the public `#/p/` route in
     dual-mode). **Replace any `localStorage`/`sessionStorage` mock with real
     persistence.** Only proceed to upload once persistence is ACTUALLY wired to
     the Lugui data API — never publish a page that "persists" only in the
     browser.

   The brand sanity-check (step 6) still applies to the final, data-wired HTML.

7.5. **🖼️ IMAGENS E ASSETS — resolva ANTES de publicar.**

   **⛔ MÍDIA É SEMPRE HOSPEDADA PELO PRÓPRIO pages.lugui.ai.** Imagem, vídeo,
   áudio, PDF: a página final só aponta para `pages.lugui.ai`. Imagens sobem
   pelo bundle `.zip` ou pelo campo `assets`; vídeos, pelo canal do passo 7.6;
   áudio e PDF ainda não têm canal (a resposta é "ainda não dá", nunca um host
   de fora). **NUNCA crie, sugira nem use um uploader ou hospedagem fora
   da Lugui** — Vercel/Vercel Blob, Imgur, Cloudinary, Google Drive, Dropbox,
   embed de YouTube/Vimeo, S3 de terceiros, GitHub raw, nem qualquer outro —
   mesmo que uma dessas ferramentas esteja conectada na sessão, e mesmo que a
   pessoa peça "só pra resolver rápido". Nem funcionaria: a política de
   segurança (CSP) do `pages.lugui.ai` **bloqueia vídeo de outro domínio e todo
   `<iframe>`**, então um vídeo externo ou um embed de YouTube aparece como um
   quadro vazio. Se faltar um canal para o que a pessoa quer, diga isso e pare —
   não improvise um.

   Uma página com `<img src="hero.jpg">` e nada mais publica **quebrada**: o
   arquivo não sobe sozinho. Antes de montar o body, varra o HTML procurando
   referências a arquivos locais em `src`, `srcset`, `href`, `poster` e `url(...)`
   do CSS. Toda referência que **não** começa com `http://`, `https://`, `//`,
   `data:`, `#`, `mailto:` ou `tel:` e tem extensão de arquivo é um asset que
   precisa subir. **Vídeo (`.mp4`, `.webm`, `.mov`…) não entra aqui — vai pelo
   passo 7.6.**

   **Se os arquivos estão neste computador (o caso comum de uma LP), use o bundle
   ZIP — um comando só, sem JSON e sem base64:**

   ```sh
   # [bloco lugui-credencial v1, inteiro, aqui]
   cd "<pasta-da-lp>" && zip -r /tmp/lugui-lp.zip . -x '.*' -x '__MACOSX/*' -x '*.mp4' -x '*.webm' -x '*.mov' -x '*.m4v' && cd - >/dev/null
   lugui_curl -w '\nHTTP %{http_code}\n' -X POST \
     -H 'Content-Type: application/zip' --data-binary @/tmp/lugui-lp.zip \
     "$_lg_api/pages/bundle?type=permanent&slug=<caminho>"
   rm -f /tmp/lugui-lp.zip
   ```

   O `.zip` precisa ter o `index.html` (o servidor aceita uma pasta raiz única,
   tipo `minha-lp/index.html`). Para efêmera, `?type=ephemeral` e sem `slug`.
   Página gerada nesta sessão com imagens que estão neste computador também vai
   por aqui: monte uma pasta temporária (`/tmp/lugui-lp/index.html` + as imagens
   nos **mesmos caminhos** que o HTML usa) e mande o `.zip` dela.

   **Sem os arquivos locais** (a imagem só existe num link https), mande cada
   uma no campo `assets` do `POST /pages` (passo 8) com `url`: o servidor baixa e
   hospeda no próprio `pages.lugui.ai`. O `path` tem que ser **idêntico** ao que
   está no HTML.

   Regras que valem sempre:
   - **NUNCA deixe `<img src>` apontando para site de terceiros.** Quebra quando o
     outro lado sai do ar, vaza o tráfego da Lugui e, em página efêmera, entrega o
     link secreto (Referer) para o host de fora. Mande a imagem como asset.
   - **NUNCA escreva base64 com a ferramenta Write nem cole base64 no chat ou no
     HTML** (`data:image/...;base64,...`). É contexto desperdiçado e arquivo
     inflado. Se precisar de `content_base64`, ele é gerado **no shell** (passo 8).
   - `alt` descritivo sempre; `width`/`height` para não pular layout.
   - Formatos aceitos: png, jpg, webp, gif, avif, svg, css, js, json, html, fontes
     (woff2/woff/ttf/otf) e `.ico`. O servidor recomprime toda imagem para WebP,
     redimensiona para no máx. 2000px e remove EXIF/GPS.
   - **Leia `unresolved_references` na resposta.** Se vier preenchido, aquelas
     imagens vão dar 404 — republique mandando os arquivos. Não diga que está
     pronto sem esse campo vazio.

7.6. **🎬 VÍDEO — canal de mídia do pages, NUNCA dentro da página.**

   Dispara quando a página tem (ou a pessoa quer) um vídeo: `<video>`, um
   arquivo `.mp4`/`.webm`/`.mov` na pasta, ou pedidos como "coloca o vídeo",
   "filme", "gravação". O vídeo sobe **direto do computador (ou do navegador)
   para o armazenamento da Lugui** e ganha um endereço permanente
   `https://pages.lugui.ai/_m/<id>/<nome>.mp4` para usar no `<video src>`.

   Regras:
   - **Só `.mp4` (H.264) ou `.webm`, até 600 MiB por vídeo.**
   - `.mov`, HEVC, `.m4v`, `.mkv`, `.avi` → peça para exportar como MP4 (H.264):
     *"No Mac: abra o vídeo no QuickTime Player → Arquivo → Exportar como →
     1080p (se aparecer a opção HEVC, deixe desmarcada)."* Se passar de 600 MiB,
     exporte em 720p. Converter aqui só se a pessoa pedir e o `ffmpeg` existir:
     `ffmpeg -i entrada.mov -c:v libx264 -pix_fmt yuv420p -c:a aac -movflags +faststart saida.mp4`.
   - **NUNCA** transforme vídeo em base64, **NUNCA** coloque vídeo no `.zip` nem
     no campo `assets`, **NUNCA** hospede fora do `pages.lugui.ai` (veja o ⛔ do
     passo 7.5). `<iframe>` de YouTube/Vimeo **não funciona** no
     `pages.lugui.ai` — o CSP bloqueia.
   - Se as ferramentas do conector Lugui Pages estiverem na sessão
     (`preparar_envio_de_video` / `verificar_envio_de_video`), elas fazem o mesmo
     que o fluxo (ii) abaixo.

   **Fluxo (i) — o arquivo está neste computador.** O vídeo sobe **em partes**
   de 64 MiB: cada parte vai sozinha, e uma que falhar é reenviada sem
   recomeçar o vídeo inteiro.

   Passo 1 — pedir o envio em partes. A resposta é um arquivo de configuração
   do curl com uma URL assinada por parte, válidas por 3 horas: **não imprima
   o conteúdo dele**, só as linhas `upload_id`, `part_count` e `split` (o
   comando abaixo já filtra).

   ```sh
   # [bloco lugui-credencial v1, inteiro, aqui]
   _lg_vid="./convite.mp4"      # caminho do vídeo (.mp4 ou .webm)
   _lg_tam=$(wc -c < "$_lg_vid" | tr -d ' ')
   printf '{"filename": "%s", "size_bytes": %s, "multipart": true}\n' "$(basename "$_lg_vid" | sed 's/[\\"]/\\&/g')" "$_lg_tam" > /tmp/lugui-media-req.json
   _lg_code=$(lugui_curl -o /tmp/lugui-upload.cfg -w '%{http_code}' -X POST \
     -H 'Content-Type: application/json' --data-binary @/tmp/lugui-media-req.json \
     "$_lg_api/media/uploads?format=curl")
   rm -f /tmp/lugui-media-req.json
   if [ "$_lg_code" = 201 ]; then
     sed -n -e 's/^# \(upload_id=.*\)/\1/p' -e 's/^# \(part_count=.*\)/\1/p' -e 's/^# \(split=.*\)/\1/p' /tmp/lugui-upload.cfg
   else
     echo "HTTP $_lg_code"; cat /tmp/lugui-upload.cfg; rm -f /tmp/lugui-upload.cfg
   fi
   ```

   **O passo 1 só reserva o endereço. A `public_url` existe no arquivo de
   configuração apenas como comentário e NÃO está utilizável: ainda não existe
   nenhum byte lá, e a página serviria um player quebrado. O `<video src>` vem
   EXCLUSIVAMENTE da resposta do passo 3 com `"status": "ready"` (campo
   `public_url` ou `html`). Nunca do passo 1, nunca adivinhado a partir do
   `upload_id` e do nome do arquivo.**

   Passo 2 — cortar o vídeo e mandar as partes **direto para o armazenamento**,
   com `curl` puro: **sem `lugui_curl` e sem cabeçalho `Authorization`** (cada
   parte já tem a assinatura na URL). O `split` usa exatamente o tamanho da
   linha `split=` do passo 1 (troque `<video>` pelo caminho do vídeo), e o
   `curl` roda **dentro da pasta das partes**, porque o arquivo de configuração
   cita cada parte pelo nome (`lugui-parte.aaa`, `lugui-parte.aab`, …). Rode
   este comando **em segundo plano** (Bash com `run_in_background`): um vídeo
   grande passa dos 10 minutos de uma chamada comum. Espere o aviso de que
   terminou, sem ficar consultando.

   ```sh
   rm -rf /tmp/lugui-partes && mkdir -p /tmp/lugui-partes && cd /tmp/lugui-partes \
     && split -b <part_size> -a 3 "<caminho-absoluto-do-video>" lugui-parte. \
     && curl -sS --fail-with-body --retry 3 -K /tmp/lugui-upload.cfg -o /dev/null -w 'parte HTTP %{http_code}\n'
   echo "curl saiu com $?"
   ```

   Uma linha `parte HTTP 200` por parte = todas chegaram. Vá ao passo 3.

   **Faltou alguma parte** (algum `HTTP` diferente de `200`, `curl` saiu com
   erro, ou a chamada foi interrompida): **não recomece do passo 1** — as
   partes que chegaram ficam guardadas. Peça só as que faltam, com URLs novas,
   e mande de novo da mesma pasta (também em segundo plano):

   ```sh
   # [bloco lugui-credencial v1, inteiro, aqui]
   _lg_code=$(lugui_curl -o /tmp/lugui-resto.cfg -w '%{http_code}' \
     "$_lg_api/media/uploads/<upload_id>/parts?format=curl")
   if [ "$_lg_code" = 200 ]; then
     sed -n 's/^# \(received=.*\)/\1/p' /tmp/lugui-resto.cfg
     cd /tmp/lugui-partes && curl -sS --fail-with-body --retry 3 -K /tmp/lugui-resto.cfg -o /dev/null -w 'parte HTTP %{http_code}\n'
   else
     echo "HTTP $_lg_code"; cat /tmp/lugui-resto.cfg
   fi
   rm -f /tmp/lugui-resto.cfg
   ```

   Repita até não faltar nada (o passo 3 também avisa: `409` com "Faltam N de M
   partes"). Ao terminar, apague as partes e as assinaturas:
   `rm -rf /tmp/lugui-partes /tmp/lugui-upload.cfg`.

   **Este computador não alcança o armazenamento** — `curl` saiu com `6`
   (`Could not resolve host`), `7` (`Failed to connect`), `28` (timeout) ou
   qualquer outro erro de rede, com `HTTP 000` e nenhuma resposta: a máquina não
   chega a `lugui-pages-production.s3.amazonaws.com` (sandbox, rede restrita,
   sessão na nuvem). Repetir não resolve e **NÃO recomece do passo 1** — o envio
   já existe. Pegue o `browser_url` DESSE MESMO envio (está nas linhas de
   comentário do arquivo), entregue para a pessoa exatamente como no fluxo (ii)
   e, quando ela avisar o "Pronto", vá ao passo 3 com o mesmo `upload_id`. Só
   então apague o arquivo:

   ```sh
   sed -n 's/^# \(browser_url=.*\)/\1/p' /tmp/lugui-upload.cfg && rm -rf /tmp/lugui-upload.cfg /tmp/lugui-partes
   ```

   Passo 3 — finalizar: o servidor confere as partes, junta tudo num arquivo só
   e confere que é mesmo um vídeo MP4/WEBM. Use o `upload_id` do passo 1 (é o mesmo endereço do
   `complete_url`, montado sobre `$_lg_api` para o token nunca ir a outro lugar):

   ```sh
   # [bloco lugui-credencial v1, inteiro, aqui]
   lugui_curl -X POST -w '\nHTTP %{http_code}\n' "$_lg_api/media/uploads/<upload_id>/complete"
   ```

   **Só um `HTTP 200` cujo corpo diz `"status": "ready"` libera o vídeo.** `409`
   = o arquivo (ou parte dele) não está lá: mande as partes que faltam (passo 2)
   ou espere o "Pronto" do navegador, e chame o passo 3 de novo — não publique. Qualquer outro código: veja a lista
   de erros no fim desta seção.

   **Fluxo (ii) — o arquivo NÃO está neste computador** (sessão na nuvem, vídeo
   anexado no chat, arquivo no celular ou no computador de outra pessoa).
   Também é o caminho quando o upload direto não é possível.

   ```sh
   # [bloco lugui-credencial v1, inteiro, aqui]
   printf '{"filename": "%s"}\n' "convite.mp4" > /tmp/lugui-media-req.json
   _lg_code=$(lugui_curl -o /tmp/lugui-upload.cfg -w '%{http_code}' -X POST \
     -H 'Content-Type: application/json' --data-binary @/tmp/lugui-media-req.json \
     "$_lg_api/media/uploads?format=curl")
   if [ "$_lg_code" = 201 ]; then
     sed -n -e 's/^# \(upload_id=.*\)/\1/p' -e 's/^# \(browser_url=.*\)/\1/p' /tmp/lugui-upload.cfg
   else
     echo "HTTP $_lg_code"; cat /tmp/lugui-upload.cfg
   fi
   rm -f /tmp/lugui-media-req.json /tmp/lugui-upload.cfg
   ```

   Entregue o `browser_url` para a pessoa:

   > Abra este link no navegador (entre com a sua conta @lugui.ai se ele pedir),
   > escolha o vídeo e espere aparecer **"Pronto"**. Vídeos grandes podem levar
   > alguns minutos — não feche a aba. Depois me avise por aqui.
   > `<browser_url>`

   Quando a pessoa avisar, rode o **passo 3** do fluxo (i) com o `upload_id`. O
   link vale 3 horas e serve uma vez só.

   **Resposta do passo 3** (`200`): `{"upload_id", "status": "ready",
   "public_url", "size_bytes", "html"}`. É daqui — e só daqui — que sai o
   endereço do vídeo. Use o `html` pronto na página:

   ```html
   <video controls playsinline preload="metadata" src="https://pages.lugui.ai/_m/<id>/convite.mp4"></video>
   ```

   Passo 4 — prova. Antes de colocar a URL na página, confirme que o endereço
   responde: a saída tem que ser `200` (`403` quer dizer que o arquivo não está
   lá — volte ao passo 2):

   ```sh
   curl -sS -o /dev/null -w '%{http_code}\n' -I "https://pages.lugui.ai/_m/<id>/convite.mp4"
   ```

   O publish (passo 9 — e também o `/pages/bundle` e o conector) **RECUSA com
   `HTTP 422`** qualquer página cujo `<video src>` aponte para um vídeo que não
   terminou de subir, não existe ou foi recusado. Se isso acontecer, repita o
   `detail` para a pessoa e termine o envio (passos 2 e 3); **não** edite o HTML
   para contornar (tirar o vídeo, trocar o endereço, apontar para fora).

   Capa opcional: `poster="capa.jpg"`, com `capa.jpg` subindo como imagem
   comum (passo 7.5). Pasta de LP que já tinha o vídeo: troque a referência
   local pelo `public_url` e deixe o vídeo fora do `.zip` (o comando do bundle
   acima já exclui).

   Erros do canal de vídeo:
   - **400** → formato não aceito (`.mov` etc.): peça para exportar como MP4
     (H.264) e recomece.
   - **413** → maior que 600 MiB: exporte em resolução menor.
   - **409** no passo 3 → o arquivo (ou alguma parte) ainda não chegou: mande o
     que falta (passo 2, "Faltou alguma parte") ou espere o "Pronto" do fluxo
     (ii), e tente de novo.
   - **422** no passo 3 → não é um MP4/WEBM válido, ou as partes não batem com
     o envio combinado (o arquivo foi descartado): confira o `split` e recomece
     do passo 1.
   - **410** → o prazo de 3 horas acabou, ou o envio em partes foi cancelado:
     recomece do passo 1.
   - **401 / 403 / 404** → token inválido, ou o envio é de outra pessoa.
   - **422** no publish (passo 9) → "vídeo ainda não chegou / não existe / foi
     recusado": a página aponta para um vídeo sem arquivo. Termine os passos 2 e
     3 (até o `"status": "ready"`), passe no passo 4 e republique.
   - Repita o `detail` da resposta para a pessoa: ele já vem em português.

8. **Build the JSON body in a temp file (do NOT interpolate the HTML inline).**
   The HTML contains quotes and newlines, so escape it as a proper JSON string.
   Write the request body to a temp file, e.g. `/tmp/lugui-ai-publish.json`, with
   this shape:

   ```json
   {
     "html": "<!doctype html>...escaped HTML as a JSON string...",
     "type": "permanent",
     "slug": null,
     "assets": [
       { "path": "hero.jpg", "url": "https://exemplo.com/foto-original.jpg" }
     ]
   }
   ```

   - Use `"type": "permanent"` or `"type": "ephemeral"` per the user's choice.
   - Set `"slug"` to the chosen canonical path for permanent pages (it may be a
     nested path like `relatorios/cardinali/q2`), or `null`.
   - `"assets"` é opcional; omita quando a página não tem arquivo. Cada item leva
     exatamente **um** entre `url` (https) e `content_base64`, e o `path` tem que
     ser idêntico à referência no HTML.
   - Write the HTML part with the Write tool (HTML correctly escaped as a JSON
     string). `jq`/`python` are NOT required — you build the JSON yourself.
   - **`content_base64` is never typed by you.** When a local file really has to
     go in the JSON (prefer the bundle of step 7.5), write only the head of the
     body with the Write tool — `{"html": "...", "type": "permanent", "slug": null`
     with **no closing brace** — to `/tmp/lugui-ai-publish.head.json`, and let
     the shell append the base64:

     ```sh
     {
       cat /tmp/lugui-ai-publish.head.json
       printf ', "assets": ['
       _lg_sep=
       for _lg_p in "hero.jpg" "img/logo.png"; do      # caminhos IGUAIS aos do HTML
         printf '%s{"path": "%s", "content_base64": "' "$_lg_sep" "$_lg_p"
         base64 < "<pasta-do-html>/$_lg_p" | tr -d '\n'
         printf '"}'; _lg_sep=', '
       done
       printf ']}\n'
     } > /tmp/lugui-ai-publish.json && rm -f /tmp/lugui-ai-publish.head.json
     ```

9. **POST with `lugui_curl`, reading the body from the temp file:**

   ```sh
   # [bloco lugui-credencial v1, inteiro, aqui]
   lugui_curl -w '\nHTTP %{http_code}\n' -X POST \
     -H 'Content-Type: application/json' --data-binary @/tmp/lugui-ai-publish.json \
     "$_lg_api/pages"
   ```

   The last line of output is the HTTP status code; everything before it is the
   JSON response body.

10. **Report the result.** On **200/201**, parse the response JSON and show the
    public `url` (and `expires_at` if present, for ephemeral pages). Também
    reporte, quando vierem preenchidos:
    - `assets` — quantos arquivos subiram e a economia (`bytes_in` → `bytes_out`);
    - `inline_images_extracted` — imagens base64 que viraram arquivo;
    - `unresolved_references` — **imagens que vão dar 404**: avise e ofereça
      republicar mandando os arquivos;
    - `skipped_files` — o que o `.zip` deixou de fora, e por quê (vídeo aqui
      quer dizer que ele não passou pelo passo 7.6).

    Map errors (repeat the response's `detail` to the person — it is already in
    Portuguese):
    - **401 / 403** → invalid token or no permission → on a computer, run
      `/lugui-ai:setup`; with `fonte=env` (cloud), fix the `LUGUI_TOKEN` where it
      is set.
    - **409** → that path is already used by someone else → ask the user for a
      different path and retry.
    - **413** → **a file or the package is too big** (not "the HTML"): the
      `detail` names which one and the limit. Reference: final HTML up to 2 MiB;
      each file up to 12 MiB; all assets together up to 40 MiB after
      compression; the whole request (JSON or `.zip`) up to 48 MiB; the `.zip`
      expanded up to 96 MiB. Shrink or remove the file named in `detail`; a
      video never belongs here (step 7.6).
    - **422** → server rejected the request. Either the path is invalid/reserved
      (ask for a different path), the HTML is invalid (empty / not HTML /
      contains a secret), or a `<video src>` points to a video that has not
      finished uploading (`detail` names the upload; finish steps 2–3 of 7.6 and
      republish — never edit the HTML around it) → fix and retry.

11. **Clean up.** Remove the temp files (`rm -f /tmp/lugui-ai-publish.json
    /tmp/lugui-lp.zip /tmp/lugui-upload.cfg`) so the HTML/body and any upload
    signature don't linger.

## Notes

- The request body is `{ "html", "type", "slug", "assets" }`; the response is
  `{ url, slug, type, expires_at, assets, assets_bytes_in, assets_bytes_out,
  inline_images_extracted, unresolved_references, skipped_files }`.
- Para pasta de LP existe `POST /pages/bundle` (corpo = o `.zip` cru, `?type=` e
  `?slug=` na query) — ver passo 7.5.
- Os assets são servidos sob a própria página (`pages.lugui.ai/<caminho>/_a/…`), e
  excluir a página exclui os arquivos junto. Vídeos ficam em
  `pages.lugui.ai/_m/…` e não somem quando a página é republicada.
- Canal de vídeo: `POST /media/uploads` (`?format=curl` devolve o arquivo de
  config do curl), `POST /media/uploads/{id}/complete`, `GET /media/uploads/{id}`,
  `GET /media` (seus vídeos) e `DELETE /media/uploads/{id}`.
- Never paste the token or the contents of `~/.lugui/config.json` into chat, and
  never write the token in a command — `lugui_curl` exists for that.
