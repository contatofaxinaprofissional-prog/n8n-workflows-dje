# DJE - Vídeos Verticais Facebook Ads

Workflow do [n8n](https://n8n.io/) que gera **vídeos de marketing verticais (1080x1920)** prontos para anúncios no Facebook Ads da empresa **DJE Construção e Reformas**. A partir de um JSON de briefing, o workflow escreve o roteiro de conversão, gera narração com voz neural, cria as imagens de cada cena, monta os clipes com legendas no estilo CapCut, junta tudo com trilha e logo, e publica o resultado no Google Drive — registrando o link no Google Sheets.

---

## Visão geral do pipeline

O fluxo executa as etapas abaixo, nesta ordem:

1. **Formulário** — entrada do usuário com:
   - Campo único **"Dados JSON"** (cola o JSON gerado pelo workflow anterior).
   - Dropdown **Voz**.
   - Dropdown **Estilo Emocional**.
   - Campo opcional **Música de Fundo (URL)**.
2. **Parse JSON Entrada** — interpreta e normaliza o JSON recebido.
3. **Edit Fields1** — prepara/ajusta os campos para as próximas etapas.
4. **Agente Diretor** — OpenAI `gpt-5-mini`. Gera **no mínimo 14 clips** com um roteiro de conversão.
5. **Parser Roteiro** — converte a saída do agente em uma estrutura processável.
6. **Item Lists (split)** — divide o roteiro em itens individuais (uma cena por item).
7. **Execução em paralelo:**
   - **Azure TTS** — narração com a voz `pt-BR-MacerioMultilingualNeural` usando **SSML emocional**.
   - **Geração de imagens RunWare** — modelo `google:4@3`, com **polling assíncrono e retry**.
8. **Baixar Foto** — faz o download das imagens geradas.
9. **Merge** — combina os ramos de áudio e imagem.
10. **Preparar Arquivos** — FFmpeg: gera **1 clip `.mp4` por cena**, com **zoom alternado** e **legendas estilo CapCut palavra-a-palavra via ASS**.
11. **Preparar Paths** — FFmpeg: **concatena com crossfade**, **sobrepõe o logo DJE + CTA** e **mixa a música de fundo opcional em volume baixo**.
12. **Upload Google Drive** — envia o vídeo final.
13. **Atualizar Google Sheets** — registra o resultado/link na planilha.

---

## Como importar

1. Abra o **n8n**.
2. Acesse o **menu** (canto superior).
3. Selecione **Import from File**.
4. Selecione o arquivo `workflow_dje_facebook_ads.json`.

---

## Pré-requisitos no servidor n8n

- **FFmpeg >= 4.4** — necessário para o filtro `amix normalize=0`.
- **libass + fontes DejaVu ou Liberation** instaladas — sem elas a legenda cai no *fallback* `drawtext`.
- **wget** disponível no sistema.
- **n8n com execução de Code nodes habilitada**.

---

## Credenciais a configurar

> Todas as credenciais estão como **placeholders** no arquivo e **precisam ser preenchidas pelo usuário** dentro da instância n8n.

- **Azure Speech key** — substituir o header `Ocp-Apim-Subscription-Key` (atualmente `YOUR_AZURE_KEY`).
- **RunWare API** — via `httpHeaderAuth`.
- **OpenAI** — credencial da API.
- **Google Drive** — OAuth2.
- **Google Sheets** — OAuth2.

---

## Campos do formulário

| Campo | Obrigatório | Descrição |
|-------|-------------|-----------|
| **Dados JSON** | Sim | Cola o JSON gerado pelo workflow anterior. |
| **Voz** | Sim | Dropdown de vozes (veja abaixo). |
| **Estilo Emocional** | Sim | Dropdown de estilos (veja abaixo). |
| **Música de Fundo (URL)** | Não | URL de um `.mp3` direto (opcional). |

**Voz** (dropdown):
- `pt-BR-MacerioMultilingualNeural` *(padrão)*
- `pt-BR-ThalitaMultilingualNeural`
- `pt-BR-AntonioNeural`
- `pt-BR-FranciscaNeural`

**Estilo Emocional** (dropdown):
- `friendly` *(padrão)*
- `hopeful`
- `calm`
- `excited`
- `serious`

---

## Observações de segurança

- **NÃO commitar chaves reais** no repositório.
- **Rotacione imediatamente** qualquer chave que tenha sido exposta.
- Os **placeholders devem ser substituídos apenas dentro da instância n8n**, nunca no arquivo versionado.

---

## Formato do JSON de entrada

O campo **Dados JSON** aceita um **objeto** ou um **array** (nesse caso usa o **primeiro elemento**). O parser tolera blocos markdown do tipo:

```json
{
  "servico": "Reforma de apartamento",
  "keyword": "reforma residencial",
  "plataforma": "Facebook Ads",
  "numClips": 14,
  "duracaoClip": 5,
  "observacoes": "Tom acolhedor e profissional",
  "empresa": "DJE Construção e Reformas",
  "persona": "Proprietários que querem reformar o lar",
  "jtbd": { "...": "..." },
  "pesquisa": { "...": "..." },
  "estrategia": { "...": "..." }
}
```

Campos esperados:

- `servico`
- `keyword`
- `plataforma`
- `numClips` — **mínimo forçado de 14** (valores menores são elevados para 14).
- `duracaoClip`
- `observacoes`
- `empresa`
- `persona`
- Objetos `jtbd`, `pesquisa` e `estrategia`.

> Mesmo que o JSON venha embrulhado em um bloco ```` ```json ... ``` ````, o workflow consegue extrair o conteúdo automaticamente.
