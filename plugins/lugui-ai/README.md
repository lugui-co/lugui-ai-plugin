# lugui-ai (Claude Code plugin)

**lugui-ai** is Lugui's internal productivity suite for Claude Code — a growing
set of features behind a single plugin. The first feature is **page
publishing** (generate on-brand, secret-scanned HTML and publish it to a public
link at `pages.lugui.ai/<slug>`, backed by the `pages-publisher` service in this
repo). More will follow (pages → skills → …), all sharing the same one-time
web-login auth.

> **Self-service, no admin, no Python.** Publishing is just Claude running
> `curl`. Authentication is a **personal access token** (`lgp_...`) you get
> yourself via **web login** (`{pages_api}/login` — sign in with @lugui.ai). On
> your computer `/lugui-ai:setup` reads it straight from the clipboard into
> `~/.lugui/config.json`; in cloud sessions it comes from `LUGUI_TOKEN` in the
> environment (or you use the connector). The plugin ships only command
> instructions and skills — there is no executable code to install.

## What you get

- **Commands**
  - `/lugui-ai:setup` — one-time auth for the whole suite: reuses a valid
    credential if there is one, otherwise web login on this computer (the token
    goes from the clipboard to `~/.lugui/config.json`, never through the chat).
    In a cloud session it explains the connector / `LUGUI_TOKEN` path instead.
  - `/lugui-ai:pages:publish <arquivo.html | pasta-da-lp>` — loads the security
    skill, asks permanent/ephemeral (+ optional canonical path), sanity-checks for
    secrets, resolves the page's **images/assets** and **videos**, then publishes
    via `curl`. Uma pasta de LP vai inteira num `.zip` (`POST /pages/bundle`);
    imagens só em link https vão no campo `assets`; vídeos vão pelo canal de
    mídia (`POST /media/uploads`). O servidor recomprime toda imagem (WebP, máx.
    2000px, sem EXIF) e reescreve os caminhos do HTML.
  - `/lugui-ai:pages:list`, `/lugui-ai:pages:delete <caminho>`,
    `/lugui-ai:data:list` — manage what you published.
- **Skills** (auto-applied by Claude when relevant)
  - `branding` — identidade visual oficial (curadoria Marketing & Design):
    paleta lima/navy light-first, tipografia Plus Jakarta Sans + DM Sans, logo
    SVG oficial, tokens + template HTML copiável.
  - `voz-e-mensagem` — o que a Lugui diz e como diz: tom de voz, público,
    pilares de mensagem, palavras proibidas e frases-chave da marca.
  - `code-best-practices` — semantic, accessible, self-contained, CSP-friendly HTML/CSS/JS.
  - `security-checklist` — what must never go on a public page, and the rule
    that media is always hosted by `pages.lugui.ai`.
  - `data` — pages that read and write shared data (forms, backoffices).

## Install

From a Claude Code session, add the public mirror as a marketplace and install
the plugin:

```
/plugin marketplace add lugui-co/lugui-ai-plugin
/plugin install lugui-ai@lugui-marketplace
```

For local development against a checkout, point the marketplace at the path:

```
/plugin marketplace add /absolute/path/to/lugui-ai-powered
/plugin install lugui-ai@lugui-marketplace
```

That's the whole install — no `pip`, no dependencies. Then run `/lugui-ai:setup`.

**Updating** (to get a new version, e.g. 0.10.1):

```
/plugin marketplace update lugui-marketplace
```

## Setup (web login → token)

Run `/lugui-ai:setup`. It first checks whether a valid credential already
exists and stops if so — every web login mints a new token that never expires,
so there is no reason to log in again when things work.

On your computer, Claude opens **`{pages_api}/login`** (default
`https://pages-api.coolify.lugui.ai/login`) in your browser. Sign in with your
**@lugui.ai** account and click **Copy** next to the token — you don't paste it
in the chat. Claude reads it from the clipboard, writes `~/.lugui/config.json`
(mode `600`), clears the clipboard and validates the token against the API.

`~/.lugui/config.json` looks like:

```json
{
  "pages_api": "https://pages-api.coolify.lugui.ai",
  "token": "lgp_..."
}
```

Never commit, share, print or copy it anywhere else — it holds your token. If
the token leaks, ask the tech team to revoke it and run `/lugui-ai:setup` again.

### How commands use the credential

Every command pastes the same short shell block at the top of each call that
talks to the API. It resolves, in order: `LUGUI_TOKEN` (alias
`LUGUI_PAGES_TOKEN`) and `LUGUI_PAGES_API` from the environment →
`~/.lugui/config.json` → the default API. Then it defines `lugui_curl`, which
hands the `Authorization` header to `curl` on stdin — the token never appears in
the chat, in the command text or in `ps`. Without a credential it explains what
to do (computer vs cloud session) and stops.

## Rotinas e sessões na nuvem

Rotinas e sessões em claude.ai/code rodam numa máquina da Anthropic que não é
o seu computador: lá **não existe o seu `~/.lugui`** e **o plugin instalado no
seu computador não vai junto**. Fazer login a cada sessão só cria mais um token.
Configure uma vez, de um destes jeitos:

- **Rotina que lê ou grava dados (ideias, cadastros, listas) — `LUGUI_TOKEN`
  no ambiente da nuvem.** Gere **um** token no seu
  computador (`https://pages-api.coolify.lugui.ai/login`) só para a rotina. Na
  rotina: **Editar** → ícone de nuvem com o nome do ambiente → engrenagem →
  **Update cloud environment**:
  - **Environment variables**: `LUGUI_TOKEN=` seguido do token;
  - **Network access**: **Custom**, com `pages-api.coolify.lugui.ai` em
    **Allowed domains** e **Also include default list of common package
    managers** marcado (sem isso, a chamada volta `403 host_not_allowed`).

  Quem usa o mesmo ambiente consegue ler a variável — use um ambiente só seu.
- **Só publicar páginas — o conector Lugui Pages.** Em
  `claude.ai/customize/connectors` → **Adicionar conector personalizado** → URL
  `https://pages-api.coolify.lugui.ai/mcp` → **Conectar** com a conta @lugui.ai.
  Depois, na rotina (**Editar** → **Conectores**), confira que ele está
  incluído. Não precisa de token nem de liberar rede, mas por enquanto ele não
  lê nem grava dados de apps.

**Nunca copie a pasta `~/.lugui` para um repositório nem para a nuvem.**

## Vídeos

Vídeo na página é **sempre hospedado pelo próprio `pages.lugui.ai`** e nunca vai
dentro do HTML, do `.zip` ou do campo `assets`:

- Formatos: **`.mp4` (H.264) ou `.webm`, até 600 MiB**. `.mov`/HEVC: exporte
  como MP4 (no Mac, QuickTime → Arquivo → Exportar como → 1080p).
- Arquivo no seu computador: o `/lugui-ai:pages:publish` pede o envio em partes
  (`POST /media/uploads?format=curl` com `"multipart": true`), corta o vídeo em
  partes de 64 MiB, manda cada uma direto para o armazenamento com `curl` e
  finaliza (`.../complete`). Se a conexão cair, só as partes que faltam são
  reenviadas (`.../parts`).
- Arquivo fora do computador (sessão na nuvem, anexo no chat): o Claude te dá um
  link; você abre, escolhe o vídeo, espera o **"Pronto"** e avisa.
- O resultado é um endereço `https://pages.lugui.ai/_m/<id>/<nome>.mp4` para
  `<video controls playsinline preload="metadata" src="…">`.
- **A URL só vale depois do "ready".** Pedir o envio reserva o endereço, mas o
  vídeo só existe quando o `.../complete` (ou o "Pronto" do navegador) responde
  `"status": "ready"`. Antes disso a URL serve um player quebrado — e a
  publicação **recusa (422)** qualquer página que aponte para um vídeo que não
  terminou de subir, não existe ou foi recusado. Terminar o envio resolve;
  editar o HTML para contornar, não.
- Se o computador não alcança o armazenamento (sandbox, rede restrita, sessão
  na nuvem), o envio **não** é refeito: o Claude te entrega o link de navegador
  do mesmo envio — você abre, escolhe o arquivo, espera o **"Pronto"** e avisa.
- **Nada de Vercel, Google Drive, Dropbox, YouTube/Vimeo ou outro host**: o CSP
  do `pages.lugui.ai` bloqueia vídeo de outro domínio e `<iframe>`, então nem
  tocaria.

## Usage

```
/lugui-ai:setup                          # once per computer — web login, token from the clipboard
/lugui-ai:pages:publish ./landing.html   # asks permanent/ephemeral, checks, publishes
```

## How publishing works

`POST {pages_api}/pages` (authenticated through `lugui_curl`) with body
`{ "html", "type", "slug", "assets" }`; response `{ url, slug, type, expires_at,
assets, unresolved_references, … }`. Para publicar uma pasta de LP inteira (com
`img/`, `css/`, fontes), o endpoint é `POST {pages_api}/pages/bundle` com o `.zip`
no corpo cru. Vídeos usam o canal de mídia (seção **Vídeos**).

Claude reads the HTML, asks permanent vs ephemeral, runs a quick secret sanity
check, writes the JSON body (with the HTML safely escaped) to a temp file, and
POSTs it with `lugui_curl --data-binary @<tmpfile>`. For **permanent** pages you
choose the canonical URL — including nested paths like
`relatorios/cardinali/q2` → `https://pages.lugui.ai/relatorios/cardinali/q2`
(leave it empty to auto-derive from the title); it goes in the `slug` field.
Server errors (401/403/409/413/422) are mapped to clear messages — a `413` means
a file or the whole package is too big, and the response's `detail` says which.
Validation (final HTML ≤ 2 MiB, looks-like-HTML, secret scan) is enforced
server-side; the local check is a fast courtesy gate.
