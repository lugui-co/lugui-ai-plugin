---
name: branding
description: >
  Identidade visual oficial da Lugui, curada pelo time de Marketing e Design —
  paleta com hexas exatos (lima #E9FF81 como acento, navy #02152B, light-first),
  tipografia (Plus Jakarta Sans + DM Sans), logo oficial em SVG e mockups de
  produto (WhatsApp no iPhone). MUST be loaded BEFORE generating or editing ANY
  HTML destined for pages.lugui.ai (e.g. for /lugui-ai:pages:publish) e sempre
  que for criar, desenhar ou revisar qualquer peça visual da Lugui: layouts,
  telas, slides, mockups, escolha de cor, tipografia ou aplicação do logo. Não
  cobre o texto/copy — para isso, use a skill voz-e-mensagem.
---

# Identidade visual — lugui.

Esta skill governa **como a Lugui aparece**: cor, tipografia, logo e mockups.
Para **o que dizer e como dizer**, use a skill `voz-e-mensagem` (no plugin:
`lugui-ai:voz-e-mensagem`).

Três arquivos-companheiros vivem ao lado deste:

- **`tokens.css`** — o `:root` completo (cores, radius, shadow, transições,
  fontes). A fonte da verdade dos tokens. Copie/`@import` ao gerar páginas.
- **`template.html`** — página inicial on-brand (header + hero + card + footer)
  já ligada aos tokens e com o logo oficial inline. **Toda página nova para
  `pages.lugui.ai` começa deste arquivo.**
- **`assets/`** — os SVGs oficiais do logo (ver §5). Nunca recrie o logo.

## ⛔ TOKENS CANÔNICOS — copie de `tokens.css`, NUNCA invente nomes/valores

```
--lugui-white: #FFFFFF          --lugui-navy: #02152B         --lugui-text: #02152B
--lugui-offwhite-50: #FBF8F0    --lugui-teal: #234248         --lugui-text-secondary: #234248
--lugui-offwhite: #EDE9DE       --lugui-gold: #998454         --lugui-text-tertiary: #606A73
--lugui-lima: #E9FF81           --lugui-border: #D9D6CC       --lugui-text-disabled: #9AA0A1
--lugui-lima-hover: #D5EB69     --lugui-hairline: #E7E4DB
```

Fontes: **Plus Jakarta Sans** (display, `--lugui-font-display`) e **DM Sans**
(texto, `--lugui-font-sans`), ambas via Google Fonts. Radius
`--radius-sm/md/lg/xl`, sombras `--shadow-sm/md/lg` (tingidas de navy),
transições `--t-fast/--t-base`, foco `--lugui-focus-ring`.

**Estes tokens NÃO existem — usá-los é o bug que esta skill previne:**
`--lugui-primary`, `--lugui-accent`, `--lugui-bg`, `--lugui-secondary`, e
também os tokens da **paleta antiga (aposentada)**: `--lugui-dark #2B4A42`,
`--lugui-lime #D4F34A`, `--lugui-cream #EDE8DC`, `--lugui-sage`,
`--lugui-premise`, `--lugui-soft`. Se encontrar uma página usando a paleta
antiga, ela está **off-brand** e precisa ser migrada. Se estiver prestes a
inventar um nome de token, PARE e leia `tokens.css`.

---

## 1. Filosofia visual — LEIA ANTES DE CRIAR

A estética da Lugui é **light-first, editorial e tipográfica**. Referência: uma
landing page limpa estilo Enbios — fundo claro quase branco, tipografia bold
criando hierarquia, cor entrando como acento pontual. **Não é dark, não é navy
dominante, não é card sobre fundo escuro.**

- **O fundo padrão de qualquer layout é claro** (branco ou off-white). Seções
  escuras (navy) são exceção — contraste em momentos específicos, nunca o
  padrão da página.
- **Lima `#E9FF81` é acento**, não fundo de página — e tem regra de contraste
  própria (ver §3): funciona como cor cheia com texto escuro por cima, não como
  texto/linha fina sobre fundo claro.
- **Navy `#02152B`** é tipografia de títulos, rodapé e seções de contraste
  isoladas — não é fundo padrão.

Personalidade: `Geométrica` · `Bold` · `Minimalista` · `Tech` · `Contemporânea`
· `Editorial` · `Light`.
Referências de estilo: **Enbios**, **Uber**, **Kron** — tipografia como
protagonista, muito espaço em branco, cor como acento.

---

## 2. Paleta oficial

**Não use cores fora deste sistema** (a paleta abaixo + os neutros desta seção
+ os estados definidos na §3).

| Nome | Token | Hex | Papel |
|---|---|---|---|
| **white** | `--lugui-white` | `#FFFFFF` | Fundo primário de páginas e documentos |
| **offwhite-50** | `--lugui-offwhite-50` | `#FBF8F0` | Seções de destaque claro, contextos formais |
| **offwhite** | `--lugui-offwhite` | `#EDE9DE` | Fundo terciário — opção para variar |
| **lima** | `--lugui-lima` | `#E9FF81` | Acento principal — preenchimento de bloco/botão/tag com texto escuro por cima (ver §3) |
| **navy** | `--lugui-navy` | `#02152B` | Títulos, rodapé, seções de contraste; texto sobre fundo claro |
| **teal** | `--lugui-teal` | `#234248` | Ícones e elementos secundários |
| **gold** | `--lugui-gold` | `#998454` | Apoio pontual — nunca dominante. Só em composições baseadas em white/offwhite/lima/teal; **nunca junto de navy** |

**Hierarquia de fundos claros:** white é o padrão. **offwhite-50** entra em
seções de destaque claro e contextos formais. **offwhite** é uma terceira opção
de fundo, para variar. Empilhe nessa ordem para criar profundidade sem usar
navy.

### Neutros (texto e bordas)
Cinzas **quentes**, derivados de navy ↔ offwhite — use estes, nunca um cinza
neutro genérico.

| Papel | Token | Hex |
|---|---|---|
| Texto primário | `--lugui-text` | `#02152B` (navy) |
| Texto secundário / ícones | `--lugui-text-secondary` | `#234248` (teal) |
| Texto terciário · legenda · metadado | `--lugui-text-tertiary` | `#606A73` |
| Texto desabilitado · placeholder | `--lugui-text-disabled` | `#9AA0A1` |
| Borda de card · divisor forte | `--lugui-border` | `#D9D6CC` |
| Divisor · hairline sutil | `--lugui-hairline` | `#E7E4DB` |

### Combinações aprovadas

**Padrão principal — fundo claro com color-blocking de lima:**
- Fundo: **white** ou **offwhite-50**.
- Texto: **navy** (títulos e corpo) e **teal** (secundário/ícones).
- Destaque: blocos, botões, tags e pílulas com **preenchimento lima** e **texto
  navy por cima**.
- ⚠️ Como lima e o fundo claro têm pouco contraste de borda, faça o bloco lima
  ser uma **forma cheia e generosa** (área + padding), para ler como campo de
  cor intencional — não um leve tom. Quem cria o contraste é o **texto navy**,
  não a borda do bloco.

**Contraste pontual — seção escura:**
- Seção **navy** + texto **white ou lima** (lima aqui pode ser texto/linha,
  porque o fundo é escuro).
- Máximo **1 seção navy por página**.

**Botão / CTA:**
- Preenchimento **lima** + texto **navy**.
- Sobre fundo **claro**, dê um **contorno fino navy** ao botão (1–1,5px) para
  separá-lo do fundo — a lima sozinha tem pouco contraste de borda no claro.
- Sobre fundo **escuro**, a lima já se separa: sem contorno necessário.
- Hover: lima mais fechada `#D5EB69` (`--lugui-lima-hover`).

### Proibido
- ❌ Página inteira com fundo navy — pesa e não é o estilo da marca
- ❌ Card navy sobre fundo navy — ilegível, sem respiro
- ❌ Lima como fundo de página inteira
- ❌ Cores fora da paleta (inclui a paleta antiga `#2B4A42`/`#D4F34A`/`#EDE8DC`)
- ❌ Gold dominando qualquer composição
- ❌ Gold junto de navy — o gold só entra em composições claras (white/offwhite/lima/teal)

---

## 3. Cores por situação

### Lima — contraste e destaque

A lima `#E9FF81` é clara: **não contrasta o suficiente como texto, ícone fino
ou linha sobre fundo claro** (white/offwhite). O destaque dela vem de usá-la
como **cor cheia** — color-blocking confiante, no espírito de como a Nubank usa
o roxo.

**✅ Como dar destaque com lima:**
- Como **preenchimento** de botão, tag, pílula, badge ou bloco — sempre com
  **texto navy ou teal por cima** (o escuro é que cria o contraste).
- Como **marca-texto / grifo** atrás de uma palavra em fundo claro — a palavra
  fica **navy**, a lima é o realce atrás.
- Como **texto ou acento sobre fundo escuro** (navy/teal) — aí sim a lima
  contrasta e pode ser texto, linha ou ícone.

**❌ Evite:**
- Lima como **cor de texto** sobre white/offwhite — some.
- Lima em **linhas finas, bordas finas ou ícones pequenos** sobre fundo claro —
  contraste insuficiente.
- Lima como fundo de página inteira.

> Regra rápida: em fundo claro, a lima é **preenchimento com texto escuro por
> cima**. Em fundo escuro, a lima pode ser o **próprio texto/acento**.

### Estado hover do botão lima
- Hover do botão lima: **`#D5EB69`** (lima levemente mais fechada). O texto
  continua **navy**.
- Demais estados de interação (pressed, disabled, foco): **a definir** — virão
  do design system da Lugui.

### Gráficos / data viz
- O universo de cores é a **própria paleta da marca**: navy, teal, lima e gold
  nas séries; navy/teal em eixos, grades e rótulos.
- **Lima = cor do dado-herói** (a série que você quer destacar). Evite lima em
  fatias ou elementos pequenos, onde some.
- Precisando de mais séries do que a paleta comporta, derive **tons da mesma
  família** (variações de teal/navy) antes de introduzir qualquer cor nova.

### Semânticos (erro, sucesso, alerta, info)
> ⏳ **A definir** — serão importados do design system da Lugui. Até lá, não
> criar vermelho/verde/âmbar avulsos. Sinalize estados com texto navy/teal +
> ícone/rotulagem clara, dentro da paleta.

---

## 4. Tipografia

Duas famílias, ambas no **Google Fonts**: **Plus Jakarta Sans** para títulos
(display) e **DM Sans** para texto.

```
https://fonts.googleapis.com/css2?family=Plus+Jakarta+Sans:wght@700;800&family=DM+Sans:wght@400;500;600&display=swap
```

| Nível | Fonte | Tamanho | Peso | Uso |
|---|---|---|---|---|
| H1 | Plus Jakarta Sans ExtraBold | 56–72px | 800 | Headlines de impacto — grandes, dominantes |
| H2 | Plus Jakarta Sans ExtraBold | 36–48px | 800 | Títulos de seção |
| H3 | Plus Jakarta Sans Bold | 22–28px | 700 | Subtítulos, títulos de card |
| Body | DM Sans Regular | 16–18px | 400 | Corpo de texto e parágrafos |
| Small | DM Sans Regular | 13–14px | 400 | Legendas, notas, metadados |
| Label | DM Sans Medium | 11–12px | 500 | Labels, tags, categorias |
| CTA | DM Sans SemiBold | 15–16px | 600 | Botões e calls-to-action |

**Hierarquia — como criar impacto:**
- Headlines grandes e pesadas (Plus Jakarta Sans 800) criam o drama visual;
  corpo pequeno e leve (DM Sans 400) cria contraste e respiro.
- Mantenha o contraste de peso: títulos em 700–800, corpo em 400 — evite pesos
  intermediários competindo no meio.

**Espaçamento entre linhas:**
- Headlines multilinha (H1/H2): `line-height: 1.2` — respiro controlado, sem
  virar vão.
- Corpo e descrições: `line-height: 1.2–1.5` — legível sem abrir demais.

---

## 5. Logo

A marca tipográfica é **"lugui."** — sempre em **caixa baixa**, com **ponto
final**. Em documentos e apresentações, o logo vai **sempre no canto superior
esquerdo**.

### Arquivos oficiais (use sempre estes, nunca recrie o logo)

**Wordmark "lugui." (marca tipográfica):**
- `assets/lugui-logo-teal.svg` — versão **teal** `#234248`.
- `assets/lugui-logo-lima.svg` — versão **lima** `#E9FF81`.
- Ambos com `viewBox="0 0 3500 1315"` → proporção **≈ 2,66 : 1**.

**Ícone / símbolo (tile quadrado):**
- `assets/lugui-icone.svg` — tile **1:1**: fundo lima `#E9FF81` + quadrado
  arredondado teal com o "l." em lima dentro.

Em páginas HTML auto-contidas (pages.lugui.ai), **inline o SVG oficial** no
markup (o `template.html` já traz o wordmark inline com `fill: currentColor` —
as únicas cores permitidas são teal `#234248` em fundo claro e lima `#E9FF81`
em fundo escuro). Nunca escreva "lugui." como texto estilizado em CSS.

### Wordmark vs ícone — quando usar cada um
- **Wordmark** → o padrão: cabeçalhos, rodapés, capas, qualquer lugar com
  espaço horizontal.
- **Ícone** → onde o espaço é quadrado e pequeno e o nome não cabe: **avatar /
  foto de perfil (WhatsApp, redes), favicon, ícone de app**. O ícone é um
  **tile fechado** (já tem fundo lima próprio) → vale sobre qualquer fundo;
  **não recolorir, não trocar o fundo, não recriar a letra**.

### Cor do wordmark conforme o fundo
| Cor do fundo | Versão do wordmark |
|---|---|
| white, offwhite ou lima | **teal** (`lugui-logo-teal.svg`) — sempre |
| navy, teal ou gold | **lima** (`lugui-logo-lima.svg`) |

### Tamanho
Manter sempre a proporção **2,66 : 1** (nunca esticar para preencher). Dois
usos:

| Uso | Altura | Largura proporcional |
|---|---|---|
| **Identificação** — cabeçalho de documento, topo de apresentação, rodapé, header de site | **máx. 32px** | ≈ 85px |
| **Protagonista** — logo isolado como elemento principal, capa/hero | **96–160px** (default 128px) | ≈ 255–425px |

- No uso protagonista, **não passar de ~35% da largura do layout** — a logo
  domina sem encostar nas bordas, mantendo o respiro editorial.

### Regras absolutas
- ❌ NUNCA escreva "Lugui" em outras fontes nem improvise o logo.
- ❌ NUNCA distorça, rotacione ou altere as proporções.
- ❌ NUNCA aplique sobre fundos que comprometam a legibilidade.
- ✅ SEMPRE use um dos arquivos SVG oficiais acima.

---

## 6. Mockups de produto

O produto da Lugui acontece no WhatsApp. **Para mostrar o produto funcionando,
use sempre:**
- Mockup de celular com a tela de WhatsApp e a conversa da Lugui, ou
- Mockup de desktop com o WhatsApp Web e a conversa.

Nunca use ilustrações abstratas ou dashboards fictícios — o WhatsApp **é** o
produto, e mostrá-lo é mais honesto e mais forte.

**Specs obrigatórias do mockup de celular:**
- Sempre **iPhone** — nunca iPad/tablet (proporção errada passa a mensagem
  errada).
- Proporção: `h = w × 2.16` (ex.: w=2.2" → h=4.75"; w=2.0" → h=4.32").
  ❌ Nunca ~1.35 (ex.: 3.4×4.6 vira iPad).
- Elementos: dynamic island (pill escuro no topo), botões laterais (volume
  esquerdo ×2, power direito), home indicator (pill cinza no rodapé).
- Tela inset: `x+0.055`, `y+0.055`, `w-0.11`, `h-0.11`, com `rectRadius`
  proporcional ao body.

**Cores do WhatsApp no mockup** (são do WhatsApp, não da paleta da marca —
replicam o app real):
- Header: verde `#075E54`, com nome "lugui." e status "online".
- Bubble do bot (esquerda): branco `#FFFFFF`. Bubble do usuário (direita):
  `#DCF8C6`.
- Fundo do chat: `#ECE5DD`.

---

## 7. Instrução para páginas `pages.lugui.ai` (`/lugui-ai:pages:publish`)

1. **Comece de `template.html`** e traga `tokens.css` (inline o `:root`).
2. Fundo **claro** (white/offwhite-50); títulos **navy** em Plus Jakarta Sans
   800; corpo em DM Sans; **um** momento de destaque lima (CTA ou grifo).
3. Logo: SVG oficial inline, canto superior esquerdo, máx. 32px de altura no
   header.
4. Copy em pt-BR seguindo a skill **`voz-e-mensagem`** (carregue-a junto).
5. Página auto-contida (CSS inline, sem dependências externas além do Google
   Fonts). Acessível: tags semânticas, `alt`, foco visível.
6. Depois passe pelas skills `security-checklist` e `code-best-practices`
   antes de publicar.

---

## 8. Checklist visual antes de entregar

- [ ] O fundo é claro (branco ou off-white)? O navy está como acento/contraste, não como base?
- [ ] A lima está como acento (botão, tag, grifo) — não como fundo de página?
- [ ] O gold está pontual, sem dominar?
- [ ] Todas as cores são da paleta oficial (hexas exatos, sem a paleta antiga)?
- [ ] Títulos em Plus Jakarta Sans (700–800) e corpo em DM Sans?
- [ ] O logo é o arquivo original, sem distorção, no canto superior esquerdo?
- [ ] Se mostra o produto: é mockup de iPhone/WhatsApp na proporção `h = w × 2.16`?
