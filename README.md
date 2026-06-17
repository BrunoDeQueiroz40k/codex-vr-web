# Codex VR

Site estático que cataloga avatares de **VRChat** num "codex" com tema de Warhammer 40k.
Ele lê um arquivo `avatars.json` e monta uma galeria de cards com uma sidebar
organizada por **categoria → canal** do Discord.

O `avatars.json` é gerado pelo bot que fica em [`../bot-avatares`](../bot-avatares):
quando alguém posta um link de avatar do VRChat num canal monitorado, o bot extrai
título, imagem e criador (via Open Graph) e grava aqui. Veja o README do bot para detalhes.

## Estrutura

| Arquivo | O que é |
|---|---|
| `index.html` | A aplicação inteira (template + lógica, sobre o runtime `dc`). |
| `support.js` | Runtime `dc` que renderiza o componente (gerado — não editar à mão). |
| `avatars.json` | Os dados dos avatares. Gerado/atualizado pelo bot. |
| `ultra.png` | Ícone usado na interface. |

## Como rodar

O site faz `fetch('./avatars.json')`, e navegadores **bloqueiam `fetch` via `file://`**.
Por isso é preciso servir por HTTP (não basta dar duplo-clique no `index.html`):

```bash
cd codex-vr
python3 -m http.server 8000
```

Depois abra **http://localhost:8000**.

Qualquer servidor estático serve. Alternativas:

```bash
npx serve .        # Node
php -S localhost:8000
```

## Como os avatares aparecem

1. Rode o bot em `../bot-avatares` (ele grava em `codex-vr/avatars.json`).
2. Poste um link de avatar do VRChat num canal/tópico monitorado.
3. Atualize a página — o card aparece, agrupado pela **categoria** e **canal** do Discord.

A sidebar é montada automaticamente a partir dos dados:

- **Categorias** viram as faixas coloridas (ordenadas pela posição no Discord).
- **Canais** ficam aninhados dentro da categoria (prefixo `#`); clicar filtra os avatares.
- Cada categoria pode ser **retraída/expandida** na seta.
- Há busca por nome, criador, canal ou categoria, e ordenação por **recentes** ou **A–Z**.

> Avatares sem `category` (ex.: dados antigos) caem no grupo **"Sem categoria"**.

## Deploy na Vercel (site) + Vercel Blob (dados)

A Vercel hospeda **só o site** (estático). O **bot não roda na Vercel** — ele precisa
de um processo sempre-ligado (sua máquina, um VPS, Railway, Fly.io…). A ponte entre os
dois é o **Vercel Blob**: o bot sobe o `avatars.json` pra lá a cada mudança e o site lê
essa URL pública.

### 1. Criar o Blob Store

1. Vercel → **Storage** → **Create** → **Blob**.
2. Abra o store → copie a variável **`BLOB_READ_WRITE_TOKEN`** (formato `vercel_blob_rw_...`).

### 2. Configurar o bot

1. No `bot-avatares/.env`, preencha `BLOB_READ_WRITE_TOKEN=...`.
2. Rode o bot. No log do `on_ready` aparece algo como:
   ```
   [blob] avatars.json publicado em: https://xxxxxxxx.public.blob.vercel-storage.com/avatars.json
   ```
3. **Copie essa URL.**

### 3. Apontar o site para a URL

Em `index.html`, dentro de `componentDidMount`, cole a URL na constante `DATA_URL`:

```js
const DATA_URL = 'https://xxxxxxxx.public.blob.vercel-storage.com/avatars.json';
```

(Vazio = usa o `./avatars.json` local, útil em dev.)

### 4. Publicar o site

1. Suba o repositório no GitHub (confirme que `bot-avatares/.env` **não** foi commitado).
2. Vercel → **Add New → Project** → importe o repo.
3. **Root Directory:** `codex-vr` · **Framework Preset:** Other · sem build command/output.
4. Deploy.

Pronto: o bot publica o JSON no Blob, e o site lê de lá. Atualizações aparecem em até
~1 min (cache do CDN, ajustável via `blob_cache_max_age` no `config.json`). Não é
preciso republicar o site a cada novo avatar.

> O `DISCORD_TOKEN` e o `BLOB_READ_WRITE_TOKEN` ficam **só no host do bot** (no `.env`),
> nunca na Vercel.

## Observações

- O `avatars.json` é só um array JSON; dá pra editar/popular à mão para testar.
- Os atributos `hint-placeholder-*` no `index.html` servem apenas ao **preview** do
  runtime `dc` (desenham um esqueleto sem dados). Na página servida por HTTP, o que
  vale são os dados reais do `avatars.json`.
