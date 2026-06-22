# Composição HyperFrames — DJE (vídeo vertical 9:16)

Esta pasta contém a **composição HTML** usada pela **HyperFrames Cloud Rendering API
(HeyGen)** para montar o vídeo vertical (1080x1920) do workflow
`workflow_dje_hyperframes.json`.

> Esta versão segue o **contrato oficial do framework `@hyperframes/core`**:
> metadados no elemento **raiz** (`<html data-composition-*>`), elementos
> temporizados com `class="clip"` + `data-start`, e **uma timeline GSAP
> registrada em `window.__timelines`** (a renderização é frame‑a‑frame /
> *seek‑driven*). Pode ainda precisar de **1 ajuste fino** após o primeiro
> render real (forma exata de `window.__timelines` e da leitura de variáveis
> pelo runtime).

---

## 1) Contrato `@hyperframes/core` (o que a composição implementa)

O elemento **raiz** (`<html>`) declara os metadados por atributos `data-*`:

| Atributo | Valor | Observação |
|----------|-------|------------|
| `data-composition-id` | `dje-vsl` | id da composição (chave em `window.__timelines`). |
| `data-width` / `data-height` | `1080` / `1920` | **OBRIGATÓRIO** — sem isso o render falha com `missing_composition_dimensions`. |
| `data-composition-duration` | `30` (default) | atualizado via JS para a **duração real** (soma das cenas). |
| `data-composition-variables` | **array JSON de definições** | é o **schema** das variáveis, **não** um objeto de valores. |

**`data-composition-variables` é um SCHEMA** no formato
`[{"id","label","type","default"}]`. No render, a HeyGen **sobrescreve os
defaults** com o objeto `variables` (casado por `id`). Como o schema só aceita
**tipos escalares** (`string`/`number`/`color`/`boolean`/`enum`), os dados
complexos (a **lista de cenas**) são passados como uma variável do tipo
**`string`** (`scenesJson`) contendo JSON — que a composição faz `JSON.parse`.

Schema declarado:

```json
[
  { "id": "audioUrl",    "label": "Audio URL",    "type": "string", "default": "" },
  { "id": "ctaHeadline", "label": "CTA headline", "type": "string", "default": "Orçamento grátis" },
  { "id": "ctaPhone",    "label": "CTA phone",    "type": "string", "default": "(11) 5891.6837" },
  { "id": "scenesJson",  "label": "Scenes JSON",  "type": "string", "default": "[]" }
]
```

Demais regras do contrato implementadas:

- Elementos visíveis temporizados têm **`class="clip"`** e **`data-start`** (cada
  cena e cada bloco de legenda; o `<audio>` é `class="clip" data-start="0"`).
- Há **uma** timeline GSAP `gsap.timeline({ paused: true })` com **todas** as
  animações posicionadas no tempo absoluto e **registrada**:
  ```js
  window.__timelines = window.__timelines || {};
  window.__timelines["dje-vsl"] = tl;
  window.__timeline = tl; // alias por segurança
  ```
- GSAP é carregado via **CDN do GSAP** (`cdnjs .../gsap/3.12.5/gsap.min.js`).
- O `<audio>` **toca** (não é mudo) — só temos imagens + 1 áudio de narração.
- Tudo é **determinístico** (sem `Date.now()`); a animação é dirigida pelo
  **seek** que o runtime aplica na timeline registrada.

### `variables` enviado no render (objeto PLANO, casando com os `id`)

```json
{
  "audioUrl": "https://dje-audios.b-cdn.net/audios/narracao_xxx.mp3",
  "ctaHeadline": "Orçamento grátis",
  "ctaPhone": "(11) 5891.6837",
  "scenesJson": "[{\"index\":1,\"imageUrl\":\"https://...\",\"text\":\"...\",\"start\":0,\"duration\":6.8}]"
}
```

> Antes usávamos um objeto aninhado (`cta:{...}`, `scenes:[...]`). Agora o objeto
> `variables` é **plano** (`audioUrl`, `ctaHeadline`, `ctaPhone`, `scenesJson`),
> e `scenesJson` é uma **string** (`JSON.stringify(scenes)`). Cada cena tem o
> formato `{ index, imageUrl, text, start, duration }`.

---

## 2) O que a composição faz visualmente

Para cada cena, na janela `[start, start+duration)`:

- mostra `imageUrl` cobrindo a tela (`background-size: cover`, 1080x1920) com
  **Ken Burns** (zoom/pan suave via GSAP, alternando a direção por índice);
- faz **fade in/out** entre cenas;
- exibe a **legenda** (`text`) animada **palavra a palavra** (estilo CapCut,
  centralizada, fonte bold grande com contorno/sombra, palavra ativa em destaque);
- exibe um **card flutuante de CTA** discreto (`ctaHeadline` + `ctaPhone`), com
  **destaque na última cena**.

Toca **um** elemento `<audio>` com `src = audioUrl`. A duração total da timeline
= soma das `duration` (ou `start+duration` da última cena), e é gravada de volta
em `data-composition-duration`.

---

## 3) Pré-visualizar localmente

Abra o `index.html` em um navegador. Se não houver runtime nem `variables` reais,
ele entra em **modo preview**: usa 1–2 cenas de exemplo e dá `tl.play()` (apenas
para visualização local). No render, o runtime controla a timeline (a composição
detecta o runtime e mantém a timeline pausada para o *seek*).

```bash
cd hyperframes
python3 -m http.server 8080
# abra http://localhost:8080/index.html
```

---

## 4) Empacotar em `.zip` e hospedar no BunnyCDN

A HyperFrames recebe a composição como uma **URL pública de um `.zip`** (campo
`project = { type: "url", url: <zip url> }`). O `index.html` deve estar na **raiz**
do zip (o campo `composition` do render aponta para `"index.html"`).

```bash
cd hyperframes
zip -r composicao-dje.zip index.html
```

> O `index.html` carrega o **GSAP via CDN**. Se o ambiente de render não tiver
> acesso à internet para o CDN, **vendorize** o `gsap.min.js` dentro da pasta e
> inclua-o no zip, trocando o `<script src>` por um caminho local.

### Subir o zip no Bunny Storage (zona `dje-audios`)

```bash
curl -X PUT \
  -H "AccessKey: SUA_STORAGE_ACCESSKEY" \
  -H "Content-Type: application/octet-stream" \
  --data-binary @composicao-dje.zip \
  "https://br.storage.bunnycdn.com/dje-audios/composicao-dje.zip"
```

A **URL pública** (via Pull Zone) fica:

```
https://dje-audios.b-cdn.net/composicao-dje.zip
```

É essa URL que vai no campo `hyperframesZipUrl` do nó **Config** (já
pré-preenchida com esse valor padrão).

---

## 5) Credenciais e campos a preencher (resumo)

No workflow `workflow_dje_hyperframes.json`:

| Onde | O que preencher |
|------|-----------------|
| Credencial **HeyGen API** (`httpHeaderAuth`) | Header **`x-api-key`** = sua chave da HeyGen/HyperFrames. Usada em *Criar Render* e *Consultar Render*. |
| Nó **Upload Audio Bunny** → header **`AccessKey`** | A **AccessKey (senha) da Storage Zone `dje-audios`**. Agora é enviada por **header inline** (não há mais credencial). Substitua `YOUR_BUNNY_STORAGE_ACCESSKEY` pelo valor real. |
| Nó **Config** → `bunnyStorageZone` | `dje-audios` (já preenchido). |
| Nó **Config** → `bunnyStorageHost` | `br.storage.bunnycdn.com` (já preenchido). |
| Nó **Config** → `bunnyPullHost` | `dje-audios.b-cdn.net` (default; **confirme** o host público do Pull Zone dessa zona). |
| Nó **Config** → `hyperframesZipUrl` | `https://dje-audios.b-cdn.net/composicao-dje.zip` (já preenchido). |

### ⚠️ Aviso de segurança (AccessKey)

A **AccessKey do Bunny Storage é secreta**. O nó **Upload Audio Bunny** agora usa
um **header inline** com `AccessKey: YOUR_BUNNY_STORAGE_ACCESSKEY` (igual ao
workflow que já funciona). Cole a chave real **somente** na sua instância do n8n
e **não** a versione/commite. **Se a AccessKey foi exposta/vazou em algum momento,
regenere-a no painel do Bunny** (Storage Zone → FTP & API Access) e atualize o
header.

> O áudio (`narracao.mp3`) é gerado e hospedado automaticamente pelo workflow no
> caminho `audios/<nome>.mp3` da Storage Zone `dje-audios`, e a URL pública
> (`https://dje-audios.b-cdn.net/audios/<nome>.mp3`) é injetada em
> `variables.audioUrl`.

---

## 6) Ajustes pós primeiro render (TODO)

- Confirmar a forma exata que o runtime espera a timeline em `window.__timelines`
  (hoje registramos `window.__timelines["dje-vsl"]` e o alias `window.__timeline`).
- Confirmar como o runtime injeta as `variables` (a composição tenta, nesta ordem:
  API de runtime `window.hyperframes.getVariables()` / `window.getVariables()`,
  defaults do schema `data-composition-variables`, e overrides via
  `window.__variableValues` / `data-variable-values`).
- Ajustar easing/posição das legendas e do CTA conforme o resultado renderizado.
