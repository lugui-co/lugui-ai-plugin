---
name: data
description: Make a published Lugui page READ AND WRITE persistent data (lists, forms, dashboards, backoffices, onboarding flows) without any backend code. MUST be loaded before publishing ANY page that has a form, collects input, stores state, uses localStorage/sessionStorage, or needs to save/share/persist data — even if the user didn't explicitly say "salvar" (detect it from the page content). Triggers: formulário, cadastro, inscrição, salvar, enviar, persistir, compartilhar, backoffice, lista colaborativa, tracker, CRUD, dashboard. Covers choosing the access pattern (internal-authenticated vs backoffice + public per-record fill links), creating the data "app", wiring lugui-data.js (authenticated AND login-free public fill), and dual-mode pages — and replacing browser-only localStorage mocks with real shared persistence.
---

# Lugui Data — data-backed pages

Use this when the user wants a published page that **persists data**: a list, a
tracker, a small CRUD, a dashboard that remembers, an onboarding/intake flow, or
a backoffice where an admin manages records and shares fill-in links with
outside people. The page stays a single static HTML file on `pages.lugui.ai`;
data lives in the Pages data API, reached from the browser with the tiny client
`lugui-data.js`. **No backend code to write.** Always also load
**`lugui-ai:branding`** — data pages must look like Lugui.

## STEP 0 — Pick the access pattern

There are **three data patterns**. For (a) and (b), **ask the user** which fits.
Pattern **(c) is automatic**: when the user asks for a **formulário / captação /
inscrição / pesquisa / lead form** — anything where outside people SUBMIT a new
record — go straight to (c) and build form + admin together (see below); **don't
ask "quer ver os dados?"** — a form ALWAYS ships with its admin panel.

- **(a) Interna autenticada** — só pessoas **@lugui** usam a página inteira. App
  ACL `private` (só o dono) ou `shared` (qualquer @lugui). A página inteira
  exige login: chame `lugui.ensureAuth()` no load.
- **(b) Backoffice (admin + links públicos por registro)** — o dono gerencia
  **logado** (rotas de admin com `lugui.data.*`) e gera **links públicos de
  preenchimento, um por registro** (alguém edita UM registro existente). Habilita
  `allow_public_fill`, cria cada registro com `public_fill=true` (pega o
  `fill_token`), link `https://pages.lugui.ai/<page>#/p/<fill_token>`; rota
  `#/p/<token>` **pública** via `lugui.public.*`. Modelo "dual-mode" abaixo.
- **(c) Formulário público aberto** — captação de leads / inscrição / pesquisa:
  **qualquer pessoa de fora ENVIA um registro NOVO, sem login.** Veja
  **"STEP 3c"** abaixo (form público + painel admin automático).

> Páginas **sem dado nenhum** (estáticas/efêmeras) não usam a data API — publique
> direto, sem app. Isso não é um padrão de dados, é a ausência dele.

## ⛔ REGRA DURA — a persistência é SEMPRE o data store da Lugui

A persistência é **SEMPRE** o data store da Lugui (`lugui.data.*` /
`lugui.public.*`, servido em `pages-api.coolify.lugui.ai`). **NUNCA proponha nem
use Google Sheets, Google Forms, Google Drive, Airtable, Notion, Supabase,
Firebase, planilhas, arquivos locais, nem qualquer backend/serviço externo —
mesmo que ferramentas do Google Workspace ou outros MCPs estejam conectados na
sessão.** A presença de um MCP (Google Workspace, etc.) **não** é convite para
usá-lo como backend de uma página.

- Os ÚNICOS padrões de dados são exatamente os 3 acima: **(a) interna
  autenticada, (b) backoffice + link público por registro, (c) formulário
  público aberto.** Não invente outros.
- **Formulário/captação/inscrição/pesquisa = SEMPRE padrão (c) no nosso store**
  (form público + painel admin), NUNCA Google Forms/Sheets. Se o usuário pedir
  Google Forms/Sheets/Airtable/etc. explicitamente: explique que o padrão Lugui
  é o store próprio (form publicado em `pages.lugui.ai` + dados no nosso data
  store, visíveis no painel admin @lugui) e **siga com (c)** — não conecte o
  serviço externo.

## The model (explain in plain terms)

- **app** — container que o dono cria. Tem **ACL** (`private`/`shared`) e a flag
  **`allow_public_fill`** (liga os links públicos por registro).
- **collection** — balde de itens dentro do app (ex.: `projetos`, `inscricoes`).
- **item** — um objeto JSON (seu registro). O servidor envolve:
  `{ id, app_key, collection, data, created_by, updated_by, fill_token,
  created_at, updated_at }` — **seus campos ficam em `.data`**.
- **fill_token** — capability token de UM item: quem tem o link lê/grava só
  aquele registro, **sem login**. Aparece em `.fill_token` na resposta
  autenticada (o dono usa pra montar o link). [padrão (b)]
- **form_token** — capability token do APP (flag `allow_public_create`):
  qualquer um pode **CRIAR registros novos** (write-only, sem login, sem ler).
  Aparece em `.form_token` na resposta autenticada. [padrão (c)]
- **Last-write-wins:** `update`/`public.update` (PUT) sobrescreve o item
  inteiro; o último a gravar vence.

## STEP 1 — Create the app (one-time, with the user's token)

Server-side com o token pessoal de `~/.lugui/config.json` (mesmo `pages_api` +
`token` do `/lugui-ai:pages:publish`). **Pergunte o ACL.** Para o padrão (b),
ligue também `allow_public_fill`:

```bash
# (a) interna:
curl -sS -X POST "<pages_api>/apps" -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" -d '{"name":"Tarefas","acl":"shared"}'

# (b) backoffice com links públicos:
curl -sS -X POST "<pages_api>/apps" -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" -d '{"name":"Implantação","acl":"private"}'
curl -sS -X PATCH "<pages_api>/apps/<app_key>" -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" -d '{"allow_public_fill":true}'

# (c) formulário público aberto — acl private (só o dono lê) + allow_public_create:
curl -sS -X POST "<pages_api>/apps" -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"name":"Leads landing","acl":"private","allow_public_create":true}'
# → guarde o "app_key" E o "form_token" da resposta (o form_token vai na página
#   pública; o app_key+collection no painel admin).
# → guarde o app_key. (Se não tem token: rode /lugui-ai:setup.)
```

## STEP 2 — Load the client library

```html
<script src="https://pages-api.coolify.lugui.ai/lugui-data.js"></script>
```

### Authenticated surface (`lugui.*`, sends the @lugui cookie)

| Call | Does |
|---|---|
| `await lugui.me()` | `{ email }` se logado, senão `null` (não lança) |
| `lugui.login(returnUrl?)` | redireciona pro login, volta pra `returnUrl` |
| `await lugui.ensureAuth()` | retorna o email, ou redireciona pro login |
| `await lugui.data.list(app, col, {limit, offset}?)` | array de itens |
| `await lugui.data.get(app, col, id)` | um item |
| `await lugui.data.create(app, col, obj, {publicFill}?)` | cria item; `{publicFill:true}` já cunha o `fill_token` |
| `await lugui.data.update(app, col, id, obj)` | PUT (last-write-wins; 404 se não existe) |
| `await lugui.data.remove(app, col, id)` | apaga |
| `await lugui.data.mintFillToken(app, col, id)` | cunha/rotaciona o link público de um item existente |
| `await lugui.data.revokeFillToken(app, col, id)` | revoga o link público |

Em **401**, chamadas `lugui.data.*` lançam **`LuguiAuthError`** (não redirecionam
sozinhas — a página decide).

### Public fill surface (`lugui.public.*`, NO login, NO cookie)

| Call | Does |
|---|---|
| `await lugui.public.get(app, fillToken)` | [padrão b] `{ id, data, updated_at }` daquele registro |
| `await lugui.public.update(app, fillToken, dataObj)` | [padrão b] PUT público (last-write-wins) |
| `await lugui.public.create(app, formToken, collection, dataObj)` | [padrão c] CRIA um registro NOVO (write-only, sem login) → `{ ok, id }` |

Erros: token inválido / flag desligada → **`LuguiNotFoundError`** (404);
envios rápidos demais → **`LuguiRateLimitError`** (429, faça backoff). Essas
chamadas **não enviam credenciais**. `public.create` é **write-only**: o público
nunca lê nem lista — só o painel admin (`lugui.data.list`, @lugui) vê os envios.

## STEP 3a — Wire pattern (a): página inteira autenticada

Comece de `skills/data/template-data.html` (já on-brand). No load chame
`await lugui.ensureAuth()` (gate) **ou** mostre o email/botão "Entrar" e capture
`LuguiAuthError`. Leia/grave com `lugui.data.*`. Campos do usuário em `item.data`.

## STEP 3b — Wire pattern (b): página DUAL-MODE (backoffice + link público)

Uma única página detecta o modo **pela rota (hash)**:

```js
// Router decide o modo:
//   #/p/<fill_token>  → PÚBLICO  (lugui.public.*, SEM ensureAuth)
//   qualquer outra    → ADMIN    (await lugui.ensureAuth() + lugui.data.*)
const m = location.hash.match(/^#\/p\/(.+)$/);
if (m) {
  // modo público — NÃO chame ensureAuth
  const rec = await lugui.public.get(APP, m[1]);     // {id, data, updated_at}
  // ...preencher e salvar:
  await lugui.public.update(APP, m[1], novoData);    // debounce, last-write-wins
} else {
  await lugui.ensureAuth();                           // admin exige login @lugui
  const items = await lugui.data.list(APP, COLLECTION);
}
```

**Onde o link público é gerado (admin):** ao criar o registro, passe
`{ publicFill: true }`; a resposta traz `.fill_token`. Monte o link
`location.origin + location.pathname + "#/p/" + fill_token` e mostre/copie/
mande no WhatsApp. (Para itens já existentes, use `lugui.data.mintFillToken`.)

Regras do dual-mode:
- **Trave a UI até hidratar** (tela de "Carregando…") e só então renderize.
- Na rota `#/p/…` **nunca** chame `ensureAuth` nem `lugui.data.*`.
- Trate `LuguiNotFoundError` (link morto) e `LuguiRateLimitError` (backoff) no
  autosave do público.

Um exemplo completo deste padrão está documentado abaixo e foi aplicado na
página de Implantação (admin = dashboard/manage/new; parceiro = `#/p/<token>`).

## STEP 3c — Wire pattern (c): FORMULÁRIO PÚBLICO ABERTO + painel admin (automático)

Quando o usuário pede um **formulário/captação/inscrição/pesquisa**, faça TUDO
isto **sem perguntar "quer ver os dados?"** — um formulário SEMPRE vem com o
painel admin junto:

1. **Crie o app** com `acl:"private"` (só o dono lê) e `allow_public_create:true`
   (STEP 1, item (c)). Guarde **`app_key`** + **`form_token`**. Escolha uma
   collection (ex.: `respostas`, `leads`, `inscricoes`).
2. **Gere DUAS páginas on-brand** (carregue `lugui-ai:branding` primeiro):
   - **Formulário público** — em `pages.lugui.ai/<slug>`. Parta de
     `skills/data/template-form.html`. Preencha `APP_KEY`, `FORM_TOKEN`,
     `COLLECTION` e os campos. Submit → `await lugui.public.create(APP_KEY,
     FORM_TOKEN, COLLECTION, dados)` → tela "Obrigado!". **SEM login.**
   - **Painel admin** — em `pages.lugui.ai/<slug>/admin` (use o slug do form +
     `/admin`). Parta de `skills/data/template-admin.html`. `await
     lugui.ensureAuth()` (logado como X) + `lugui.data.list(APP_KEY, COLLECTION)`
     → **tabela dos envios** + contador + **Exportar CSV** (client-side) +
     excluir linha (`lugui.data.remove`). **Só @lugui.**
3. **Publique as duas** com `/lugui-ai:pages:publish` (mesmo `APP_KEY`/
   `FORM_TOKEN`/`COLLECTION` nas duas) e **entregue os 2 links**: o **público**
   (pra divulgar/compartilhar) e o **/admin** (pra acompanhar os envios).

Regras:
- O `form_token` é write-only: o público só CRIA, nunca lê/lista. Os dados só
  aparecem no painel admin (login @lugui). Por isso `acl:"private"`.
- Trate `LuguiRateLimitError` (429) no submit com mensagem amigável e retry.
- **Anti-spam:** ainda NÃO há captcha/honeypot (o rate-limit por IP+token é o
  único freio hoje). É um TODO — não invente captcha agora; pode avisar o usuário
  que proteção anti-spam mais forte vem depois.

## What to make clear to the user

- **Pattern (a)** exige login @lugui pra ver/usar a página (o HTML é público, o
  *dado* é protegido).
- **Pattern (b):** o backoffice exige login @lugui; **o link `#/p/<token>` NÃO** —
  quem recebe preenche sem conta, e mexe só naquele registro. Compartilhe o link
  só com quem deve preencher; pra cortar o acesso, `revokeFillToken`.
- **Pattern (c):** o **formulário** é público (qualquer um envia, sem login); o
  **painel /admin** é só @lugui. Os envios só são visíveis no painel — o público
  nunca lê o que outros mandaram. Sempre entregue os 2 links (form + admin).
- Em nenhum caso coloque segredos no HTML (veja `security-checklist`).

## Then publish

Entregue o HTML pronto, on-brand e data-wired pro **`/lugui-ai:pages:publish`**
(que roda os checks de branding + segurança antes do upload).
