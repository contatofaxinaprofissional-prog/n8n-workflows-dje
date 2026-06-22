# Composição HyperFrames — DJE (vídeo vertical 9:16)

Esta pasta contém a **composição HTML** usada pela **HyperFrames Cloud Rendering API
(HeyGen)** para montar o vídeo vertical (1080x1920) do workflow
`workflow_dje_hyperframes.json`.

> **Primeira versão.** Não tivemos acesso à documentação oficial do framework de
> composição da HyperFrames (o site da HeyGen retornou **403**). Por isso a
> `index.html` foi escrita de forma **autossuficiente e defensiva** (CSS/JS puro,
> sem CDN, relógio único de tempo seek-driven). Após o **primeiro render real**,
> ajuste os pontos marcados nos comentários do `index.html` (principalmente o nome
> exato do hook de tempo/seek exposto pelo renderer).

---

## 1) O que a composição faz

- Renderiza um vídeo **vertical 1080x1920 (9:16)**.
- Lê os valores padrão do atributo `data-composition-variables` do elemento raiz
  `#composition`. **A HeyGen sobrescreve** esses valores com o objeto `variables`
  enviado no corpo do render.
- Para cada cena, na janela `[start, start+duration)`:
  - mostra `imageUrl` cobrindo a tela (`object-fit: cover`) com **Ken Burns**
    (zoom/pan suave, alternando a direção por índice);
  - exibe a **legenda** (`text`) animada **palavra a palavra** (estilo CapCut,
    centralizada na parte inferior, fonte bold grande com contorno/sombra);
  - exibe um **card flutuante de CTA** discreto (`cta.headline` + `cta.phone`),
    com **destaque na última cena**.
- Toca **um** elemento `<audio>` com `src = audioUrl` desde o início.
- Toda a animação é dirigida por um **relógio único** (`getTime()`), que prefere um
  tempo de *seek* global (caso a HyperFrames exponha um) e cai para o `currentTime`
  do `<audio>`. **Não** usa `Date.now()` / wall-clock — para ser determinista.

### Schema das `variables` (use exatamente estes nomes)

```json
{
  "audioUrl": "https://.../narracao.mp3",
  "cta": { "headline": "Orçamento grátis", "phone": "(11) 5891.6837" },
  "scenes": [
    { "index": 1, "imageUrl": "https://...", "text": "...narração do clipe...", "start": 0, "duration": 6.8 }
  ]
}
```

---

## 2) Pré-visualizar localmente

Basta abrir o `index.html` em um navegador. Ele já traz 1–2 cenas de exemplo no
atributo `data-composition-variables` para o preview standalone.

```bash
# opção simples: servir a pasta (evita restrições de file://)
cd hyperframes
python3 -m http.server 8080
# abra http://localhost:8080/index.html
```

---

## 3) Empacotar em `.zip` e hospedar no BunnyCDN

A HyperFrames recebe a composição como uma **URL pública de um `.zip`** (campo
`project = { type: "url", url: <zip url> }`). O `index.html` deve estar na **raiz**
do zip (o campo `composition` do render aponta para `"index.html"`).

### 3.1) Gerar o zip (index.html na raiz)

```bash
cd hyperframes
# inclua TODOS os assets locais que a composição usar (fontes/libs vendorizadas).
# NÃO inclua este README no pacote de render (opcional).
zip -r composicao-dje.zip index.html
# (se adicionar libs/fonts locais no futuro: zip -r composicao-dje.zip index.html assets/ libs/)
```

> Importante: **não** dependa de CDN externa dentro do `index.html` no momento do
> render. Se precisar de alguma lib (ex.: GSAP), **vendorize** o arquivo dentro da
> pasta e inclua-o no zip.

### 3.2) Subir o zip no Bunny Storage

Você pode subir pelo painel do Bunny (Storage > sua Storage Zone > pasta) ou via
API de Storage (HTTP `PUT`). Exemplo com `curl` (ajuste a região do host de
storage e a sua AccessKey de Storage):

```bash
curl -X PUT \
  -H "AccessKey: SUA_STORAGE_PASSWORD" \
  -H "Content-Type: application/octet-stream" \
  --data-binary @composicao-dje.zip \
  "https://br.storage.bunnycdn.com/SUA_STORAGE_ZONE/composicao-dje.zip"
```

A **URL pública** (via Pull Zone) ficará assim:

```
https://SEU_PULL_ZONE.b-cdn.net/composicao-dje.zip
```

É **essa URL pública** que você cola no campo `hyperframesZipUrl` do nó **Config**
do workflow.

---

## 4) Credenciais e campos a preencher (resumo)

No workflow `workflow_dje_hyperframes.json`:

| Onde | O que preencher |
|------|-----------------|
| Credencial **HeyGen API** (`httpHeaderAuth`) | Header **`x-api-key`** = sua chave da HeyGen/HyperFrames. Usada em *Criar Render* e *Consultar Render*. |
| Credencial **BunnyCDN Storage** (`httpHeaderAuth`) | Header **`AccessKey`** = a **senha (password) da sua Storage Zone** do Bunny. Usada em *Upload Audio Bunny*. |
| Nó **Config** → `bunnyStorageZone` | Nome da sua Storage Zone do Bunny. |
| Nó **Config** → `bunnyStorageHost` | Endpoint de storage da sua região (ex.: `br.storage.bunnycdn.com`, `storage.bunnycdn.com`, `ny.storage.bunnycdn.com`...). |
| Nó **Config** → `bunnyPullHost` | Host público do seu Pull Zone (ex.: `vaidosas-videos.b-cdn.net`). |
| Nó **Config** → `hyperframesZipUrl` | URL pública (Pull Zone) do `.zip` da composição já hospedado no Bunny. |

> O áudio (`narracao.mp3`) é gerado e hospedado automaticamente pelo workflow no
> caminho `audios/<nome>.mp3` da sua Storage Zone, e a URL pública
> (`https://{bunnyPullHost}/audios/<nome>.mp3`) é injetada em `variables.audioUrl`.

---

## 5) Ajustes pós primeiro render (TODO)

- Confirmar o **hook de tempo/seek** real da HyperFrames. Hoje `index.html` aceita
  `window.setRenderTime(t)`, `window.seek(t)`, `window.__setRenderTime(t)` e
  `window.HF_seek(t)`. Mantenha apenas o correto após validar.
- Ajustar easing/posição das legendas e do CTA conforme o resultado renderizado.
- Confirmar se a HyperFrames injeta `variables` via `window.COMPOSITION_VARIABLES`
  ou substituindo o atributo `data-composition-variables` (a composição suporta
  ambos).
