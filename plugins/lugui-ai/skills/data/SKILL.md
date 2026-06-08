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

## STEP 0 — Ask the user which access pattern they need

Before generating anything, **ask which of these fits** (it changes auth + how
the page is wired):

- **(a) Interna autenticada** — só pessoas **@lugui** usam a página inteira. App
  ACL `private` (só o dono) ou `shared` (qualquer @lugui). A página inteira
  exige login: chame `lugui.ensureAuth()` no load.
- **(b) Backoffice (admin + links públicos por registro)** — o dono gerencia
  **logado** (rotas de admin com `lugui.data.*`) e gera **links públicos de
  preenchimento, um por registro**, para gente de fora (sem conta @lugui).
  Habilita `allow_public_fill` no app, cria cada registro com `public_fill=true`
  (pega o `fill_token`), e o link é
  `https://pages.lugui.ai/<page>#/p/<fill_token>`. A rota `#/p/<token>` é
  **pública** (sem login) e usa `lugui.public.*`. **Este é o modelo "dual-mode"
  abaixo.**
- **(c) Form público totalmente aberto** (qualquer pessoa cria registros novos
  sem link individual) — **ainda não suportado** (roadmap). Não implemente; se o
  usuário pedir isso, ofereça (b) como alternativa (um link por registro) ou
  explique que está no roadmap.

## The model (explain in plain terms)

- **app** — container que o dono cria. Tem **ACL** (`private`/`shared`) e a flag
  **`allow_public_fill`** (liga os links públicos por registro).
- **collection** — balde de itens dentro do app (ex.: `projetos`, `inscricoes`).
- **item** — um objeto JSON (seu registro). O servidor envolve:
  `{ id, app_key, collection, data, created_by, updated_by, fill_token,
  created_at, updated_at }` — **seus campos ficam em `.data`**.
- **fill_token** — capability token de UM item: quem tem o link lê/grava só
  aquele registro, **sem login**. Aparece em `.fill_token` na resposta
  autenticada (o dono usa pra montar o link).
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
| `await lugui.public.get(app, fillToken)` | `{ id, data, updated_at }` daquele registro |
| `await lugui.public.update(app, fillToken, dataObj)` | PUT público (last-write-wins) |

Erros: link inválido/revogado/expirado → **`LuguiNotFoundError`** (404);
gravações rápidas demais → **`LuguiRateLimitError`** (429, faça backoff e
retente). Essas chamadas **não enviam credenciais**.

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

## What to make clear to the user

- **Pattern (a)** exige login @lugui pra ver/usar a página (o HTML é público, o
  *dado* é protegido).
- **Pattern (b):** o backoffice exige login @lugui; **o link `#/p/<token>` NÃO** —
  quem recebe preenche sem conta, e mexe só naquele registro. Compartilhe o link
  só com quem deve preencher; pra cortar o acesso, `revokeFillToken`.
- Em nenhum caso coloque segredos no HTML (veja `security-checklist`).

## Then publish

Entregue o HTML pronto, on-brand e data-wired pro **`/lugui-ai:pages:publish`**
(que roda os checks de branding + segurança antes do upload).
