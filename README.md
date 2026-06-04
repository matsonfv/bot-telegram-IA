<div align="center">

# 🤖 Bot Telegram Multifuncional

**Workflow n8n com IA, geração de documentos PDF, transcrição de áudio e feed de notícias em tempo real**

[![n8n](https://img.shields.io/badge/n8n-Workflow-EA4B71?style=for-the-badge&logo=n8n&logoColor=white)](https://n8n.io/)
[![Telegram](https://img.shields.io/badge/Telegram-Bot-26A5E4?style=for-the-badge&logo=telegram&logoColor=white)](https://core.telegram.org/bots)
[![Google Docs](https://img.shields.io/badge/Google_Docs-API-4285F4?style=for-the-badge&logo=googledocs&logoColor=white)](https://developers.google.com/docs)
[![Google Drive](https://img.shields.io/badge/Google_Drive-API-34A853?style=for-the-badge&logo=googledrive&logoColor=white)](https://developers.google.com/drive)
[![Gemini](https://img.shields.io/badge/Google_Gemini-LLM-8E75B2?style=for-the-badge&logo=google&logoColor=white)](https://ai.google.dev/)
[![Gladia](https://img.shields.io/badge/Gladia-STT-FF6B35?style=for-the-badge)](https://gladia.io/)

</div>

---

## 📋 Índice

- [Sobre o Projeto](#-sobre-o-projeto)
- [Funcionalidades](#-funcionalidades)
- [Arquitetura do Workflow](#-arquitetura-do-workflow)
- [Fluxo por Caso de Uso](#-fluxo-por-caso-de-uso)
- [Integrações e Credenciais](#-integrações-e-credenciais)
- [Como Importar e Configurar](#-como-importar-e-configurar)
- [Gestão de Estado](#-gestão-de-estado)
- [Estrutura dos Nós](#-estrutura-dos-nós)
- [Melhorias Futuras](#-melhorias-futuras)

---

## 💡 Sobre o Projeto

Este projeto é um **workflow n8n** que transforma um bot do Telegram em uma ferramenta multifuncional de produtividade, combinando automações com integrações de IA. Com um único ponto de entrada (o Telegram), o usuário pode gerar documentos PDF personalizados com texto e imagem, consultar notícias em tempo real, transcrever mensagens de voz e conversar com um agente de IA com memória de contexto.

A lógica de roteamento é feita via **Switch com 8 saídas**, gerenciando um sistema de **estado por conversa** através do `$getWorkflowStaticData('global')` — o que permite interações de múltiplos passos sem banco de dados externo.

> **Tipo de projeto:** Automação / Low-code / AI-powered Bot  
> **Plataforma:** [n8n](https://n8n.io/) (self-hosted ou cloud)

---

## ✅ Funcionalidades

| Comando / Gatilho | Funcionalidade |
|---|---|
| `/start` | Exibe menu interativo com todas as opções |
| `/criardoc` | Gera documento com papel timbrado, exporta em PDF e envia |
| `/docimagem` | Gera documento com texto + imagem embutida, exporta PDF |
| `/noticias` | Busca e exibe as 8 últimas notícias do G1 Mundo em tempo real |
| Mensagem de voz 🎙️ | Transcreve automaticamente o áudio via Gladia AI |
| Qualquer texto ✍️ | Responde via AI Agent (Google Gemini) com memória de contexto |

---

## 🏗 Arquitetura do Workflow

```
┌─────────────────────────────────────────────────────────────────┐
│                      TELEGRAM TRIGGER                           │
│             (Webhook — recebe toda mensagem)                    │
└──────────────────────┬──────────────────────────────────────────┘
                       │
              ┌────────▼────────┐
              │ Code: Verificar │  → detecta voice message
              │     Estado      │  → injeta _estado salvo por chatId
              └────────┬────────┘
                       │
              ┌────────▼────────┐
              │     SWITCH      │  8 saídas baseadas em texto ou _estado
              └────────┬────────┘
                       │
       ┌───────────────┼───────────────────────┐
       │               │                       │
  /start          /criardoc              /docimagem
  [Menu]       [Caso 1: PDF]         [Caso 2: PDF+Imagem]
       │               │                       │
  /noticias      voice message          fallback (texto)
  [Caso 3]       [Caso 4: STT]         [AI Agent + Gemini]
```

---

## 🔄 Fluxo por Caso de Uso

### 📝 Caso 1 — `/criardoc` (Documento com Papel Timbrado)

```
/criardoc
    │
    ▼
Code: Salvar Estado ("aguardando_texto_doc" no StaticData)
    │
    ▼
Telegram: Pede texto ao usuário
    │
    ▼  [usuário digita o texto — próxima mensagem]
    │
Code: Limpar Estado + injeta _texto_doc no item
    │
    ├──► Google Drive: Copia template (Google Docs)
    │    └── nome: Documento_DD-MM-YYYY_HH-mm
    │
    ├──► Telegram: "⏳ Gerando o documento..."
    │
    ▼
Google Docs: Substitui {{TEXTO}} pelo texto do usuário
    │
    ▼
HTTP Request: Exporta documento como PDF (Google Docs Export API)
    │
    ├──► Google Drive: Torna arquivo público (anyone reader)
    │        └──► Telegram: Envia link público do Drive
    │
    └──► Telegram: Envia o arquivo PDF diretamente no chat
```

### 🖼️ Caso 2 — `/docimagem` (Documento com Texto + Imagem)

```
/docimagem
    │
    ▼
Code: Coletar dados — salva estado "doc2_texto"
    │
    ▼
Telegram: Pede o texto
    │
    ▼  [usuário envia o texto]
    │
Code: Coletar dados — salva texto, muda estado para "doc2_imagem"
    │
    ▼
Telegram: Pede a imagem
    │
    ▼  [usuário envia foto ou arquivo de imagem]
    │
Code: Processar Foto — extrai file_id, recupera dados do _c2, limpa estado
    │
    ▼
HTTP: getFile (Telegram API) → obtém file_path
    │
    ▼
HTTP: Download da imagem (arquivo binário)
    │
    ├──► Telegram: "⏳ Gerando doc com imagem..."
    │
    ▼
Google Drive: Upload da imagem
    │
    ▼
Google Drive: Torna imagem pública (anyone reader)
    │
    ▼
Google Drive: Copia template DocImagem
    │
    ▼
Google Docs: Substitui {{TEXTO}}
    │
    ▼
HTTP GET: Busca estrutura JSON do documento (Docs API v1)
    │
    ▼
Code: Search IMAGEM — localiza índice do marcador {{IMAGEM}} no JSON
    │
    ▼
HTTP POST: batchUpdate — insere imagem inline no índice encontrado
                         + remove marcador {{IMAGEM}} (deleteContentRange)
    │
    ▼
HTTP: Exporta PDF → Telegram: envia PDF + link público
```

### 📰 Caso 3 — `/noticias` (Feed RSS em Tempo Real)

```
/noticias
    │
    ▼
HTTP GET: https://g1.globo.com/rss/g1/mundo/
    │
    ▼
XML: Parse RSS → extrai array de items
    │
    ▼
Code: Formatar Notícias
    └── Pega até 8 items
    └── Formata: "N. [Título](link)"
    └── Markdown para Telegram
    │
    ▼
Telegram: Envia mensagem com os 8 títulos + links clicáveis
```

### 🎙️ Caso 4 — Transcrição de Áudio (automático)

```
[Usuário envia mensagem de voz]
    │
    ▼
Code: Verificar Estado — detecta campo voice, injeta _tipo="voice"
    │
    ▼
Switch → saída "Transcrever áudio"
    │
    ▼
HTTP: getFile (Telegram API) — obtém file_path do áudio
    │
    ▼
HTTP: Download do arquivo de áudio (binário)
    │
    ▼
Gladia: Upload Audio (POST /v2/upload, multipart/form-data)
    │
    ▼
Gladia: Solicita transcrição (POST /v2/transcription)
    └── language_config: pt, code_switching: false
    │
    ▼
Wait 3 segundos
    │
    ▼
Gladia: Buscar Resultado (GET /v2/transcription/:id)
    │
    ▼
If: status === "done"?
    ├── SIM → Code: Extrair Texto (full_transcript ou utterances)
    │              └──► Telegram: Envia transcrição
    │
    └── NÃO → volta para Wait (polling loop até concluir)
```

### 🤖 Fallback — AI Agent (qualquer mensagem não reconhecida)

```
[Mensagem de texto não reconhecida pelo Switch]
    │
    ▼
AI Agent (n8n LangChain)
    ├── LLM: Google Gemini Chat Model
    ├── Memória: memoryBufferWindow (por chat_id — memória de contexto)
    └── System Prompt: conhece as 4 funcionalidades, responde em pt-BR
    │
    ▼
Telegram: Envia resposta da IA com Markdown
```

---

## 🔌 Integrações e Credenciais

| Serviço | Tipo de Credencial | Uso no Workflow |
|---|---|---|
| **Telegram Bot API** | `telegramApi` | Trigger, envio de mensagens, arquivos e documentos |
| **Google Drive** | `googleDriveOAuth2Api` | Copiar templates, fazer upload, compartilhar arquivos |
| **Google Docs** | `googleDocsOAuth2Api` | Editar documentos, exportar PDF, batchUpdate |
| **Gladia AI** | `httpHeaderAuth` (API Key) | Upload e transcrição de áudios (STT) |
| **Google Gemini** | `googlePalmApi` | LLM do AI Agent conversacional |
| **Telegram (file)** | `httpHeaderAuth` | Download direto de arquivos via HTTP (áudio e imagem) |

---

## 🚀 Como Importar e Configurar

### Pré-requisitos

- [n8n](https://n8n.io/) instalado (self-hosted via Docker ou n8n Cloud)
- Conta no [Telegram](https://core.telegram.org/bots) com um bot criado via [@BotFather](https://t.me/BotFather)
- Conta Google com acesso à Google Drive API e Google Docs API habilitadas
- Conta na [Gladia](https://app.gladia.io/) para transcrição de áudio
- Conta no [Google AI Studio](https://aistudio.google.com/) para a API do Gemini

### Passo a passo

**1. Importar o workflow**
```
n8n → Menu → Import from File → selecione o arquivo bot-telegram-version.json
```

**2. Configurar credenciais**

No painel do n8n, acesse **Credentials** e crie/vincule:
- `telegramApi` → Token do seu bot (obtido via @BotFather)
- `googleDriveOAuth2Api` → OAuth2 do Google Cloud
- `googleDocsOAuth2Api` → OAuth2 do Google Cloud (pode ser a mesma app)
- `httpHeaderAuth` (Gladia) → Header: `x-gladia-key: SUA_API_KEY`
- `googlePalmApi` → Chave da API do Gemini

**3. Preparar templates no Google Docs**

Crie dois documentos modelo no Google Drive:
- **Template `/criardoc`**: documento com papel timbrado contendo o marcador `{{TEXTO}}`
- **Template `/docimagem`**: documento com o marcador `{{TEXTO}}` e o marcador `{{IMAGEM}}` no lugar desejado para a imagem

Substitua os IDs de template no workflow:
- Nó **Copy file** → `fileId`: ID do template `/criardoc`
- Nó **Copy Template** → `fileId`: ID do template `/docimagem`

**4. Atualizar o token do Telegram nas requisições HTTP diretas**

Os nós `HTTP: Get File Path`, `HTTP: Download Audio` e `Get Foto Path` / `Download Foto` usam a URL da Bot API com o token hardcoded. Substitua pelo token do seu bot:
```
https://api.telegram.org/bot<SEU_TOKEN>/getFile
https://api.telegram.org/file/bot<SEU_TOKEN>/...
```

**5. Ativar o workflow**

Ligue o toggle **Active** no canto superior direito. O n8n registrará o webhook automaticamente no Telegram.

---

## 🧠 Gestão de Estado

O bot implementa um sistema de **estado por conversa sem banco de dados**, usando a função nativa do n8n:

```javascript
const workflow = $getWorkflowStaticData('global');

// Salvar estado
workflow[chatId] = 'aguardando_texto_doc';

// Ler estado
const estado = workflow[chatId];

// Limpar estado
delete workflow[chatId];
```

Os estados possíveis por `chatId` são:

| Chave | Valor | Significado |
|---|---|---|
| `chatId` | `aguardando_texto_doc` | Esperando o texto do `/criardoc` |
| `chatId` | `doc2_texto` | Esperando o texto do `/docimagem` |
| `chatId` | `doc2_imagem` | Esperando a imagem do `/docimagem` |
| `chatId + '_c2'` | `{ texto: "..." }` | Dados coletados do Caso 2 em andamento |

---

## 🗂 Estrutura dos Nós

```
Workflow (40 nós)
│
├── 📡 ENTRADA
│   └── Telegram Trigger
│
├── 🔀 ROTEAMENTO
│   ├── Code: Verificar Estado         (detecta voice, injeta _estado)
│   └── Switch                         (8 saídas + fallback para AI Agent)
│
├── 📝 CASO 1 — /criardoc
│   ├── Code: Salvar Estado Doc
│   ├── Pedir Texto do Documento
│   ├── Code: Limpar Estado
│   ├── Copy file                      (Google Drive)
│   ├── Gerando o documento            (feedback ao usuário)
│   ├── Update a document              (Google Docs: replaceAll {{TEXTO}})
│   ├── HTTP Request: Exportar PDF
│   ├── Share file
│   ├── Envia documento PDF
│   └── Enviar Link Público
│
├── 🖼️ CASO 2 — /docimagem
│   ├── Code: Coletar dados            (máquina de estados texto→imagem)
│   ├── Perguntar                      (feedback dinâmico)
│   ├── Code: Processar Foto
│   ├── Get Foto Path
│   ├── Download Foto
│   ├── Gerando doc com imagem         (feedback ao usuário)
│   ├── Upload Foto                    (Google Drive)
│   ├── Foto Pública
│   ├── Copy Template                  (Google Drive)
│   ├── Substituir Marcadores          (Google Docs: replaceAll {{TEXTO}})
│   ├── HTTP Request: GET              (Docs API: busca estrutura JSON)
│   ├── Code: Search IMAGEM            (localiza índice do marcador)
│   ├── HTTP Request: POST             (batchUpdate: insere imagem)
│   ├── HTTP Request: Exportar PDF1
│   ├── Share file1
│   ├── Envia documento PDF1
│   └── Enviar Link Público1
│
├── 📰 CASO 3 — /noticias
│   ├── HTTP: Buscar RSS G1
│   ├── XML: Parse RSS
│   ├── Code: Formatar Notícias
│   └── Enviar Notícias
│
├── 🎙️ CASO 4 — Áudio
│   ├── HTTP: Get File Path
│   ├── HTTP: Download Audio
│   ├── Gladia: Upload Audio
│   ├── Gladia: Transcrição
│   ├── Wait                           (3s polling)
│   ├── Gladia: Buscar Resultado
│   ├── If                             (status === "done"?)
│   ├── Code: Extrair Texto
│   └── Enviar Transcrição
│
└── 🤖 FALLBACK — AI Agent
    ├── Google Gemini Chat Model       (LLM)
    ├── Simple Memory                  (memória por chat_id)
    ├── AI Agent                       (conversationalAgent)
    └── Enviar Resposta IA
```

---

## 🔮 Melhorias Futuras

- [ ] Mover token do bot das URLs hardcoded para variáveis de ambiente do n8n
- [ ] Adicionar suporte a múltiplos idiomas na transcrição (Gladia suporta 99+)
- [ ] Webhook de confirmação de entrega das mensagens
- [ ] Comando `/help` mais detalhado com exemplos visuais
- [ ] Persistência de estado em banco externo (Redis ou PostgreSQL) para maior escalabilidade
- [ ] Suporte a grupos do Telegram (não apenas chats privados)
- [ ] Adicionar outros feeds RSS (tecnologia, esportes, etc.) com seleção interativa
- [ ] Rate limiting por usuário para evitar abuso
- [ ] Logs e monitoramento via n8n Error Workflow

---

## 👨‍💻 Autor

Desenvolvido por **Matson** — estudante de Técnico em Informática para Internet no SENAC/RN e graduado em Ciências e Tecnologia pela UFRN.

[![GitHub](https://img.shields.io/badge/GitHub-matsonfv-181717?style=for-the-badge&logo=github)](https://github.com/matsonfv)

---

<div align="center">
  <sub>Feito com 🤖 n8n + Telegram + Google APIs + Gladia AI</sub>
</div>
