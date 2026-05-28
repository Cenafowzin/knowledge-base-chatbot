# Projeto 2 — Chatbot com Base de Conhecimento

> Stack: n8n · OpenAI/Gemini · Pinecone (vector DB) · Telegram ou Widget HTML  
> Tempo estimado: 4–5 horas  
> Valor de venda: $500–1.500

---

## O que esse projeto faz

1. Você sobe documentos do cliente (PDFs, textos, FAQs)
2. n8n processa e indexa o conteúdo num banco vetorial (Pinecone)
3. Usuário manda pergunta via Telegram ou widget no site
4. n8n busca os trechos mais relevantes dos documentos
5. IA responde com base **apenas** no conteúdo do cliente — sem inventar nada

Casos de uso que fecham rápido:
- Chatbot de suporte que responde perguntas com base no manual do produto
- Assistente interno que responde dúvidas sobre políticas da empresa
- Bot de FAQ para e-commerce (frete, troca, garantia)

---

## Estrutura do repositório

```
chatbot-base-conhecimento/
├── docker-compose.yml
├── .env.example
├── .env
├── documentos/
│   └── coloque-seus-pdfs-aqui.pdf
├── workflows/
│   ├── indexar-documentos.json   ← roda uma vez para carregar os docs
│   └── chatbot.json              ← roda a cada mensagem do usuário
└── README.md
```

---

## Pré-requisitos

- n8n rodando via Docker (mesmo do Projeto 1 ou nova instância)
- Conta no [Pinecone](https://pinecone.io) — free tier suficiente
- Chave de API OpenAI ou Gemini
- Bot no Telegram (via @BotFather) **ou** usar o widget HTML incluso

---

## Passo 1 — Subir o n8n

### `docker-compose.yml`

```yaml
version: "3.8"

services:
  n8n:
    image: n8nio/n8n:latest
    container_name: chatbot-n8n
    restart: unless-stopped
    ports:
      - "5678:5678"
    environment:
      - N8N_BASIC_AUTH_ACTIVE=true
      - N8N_BASIC_AUTH_USER=${N8N_USER}
      - N8N_BASIC_AUTH_PASSWORD=${N8N_PASSWORD}
      - GENERIC_TIMEZONE=America/Recife
      - WEBHOOK_URL=${WEBHOOK_URL:-http://localhost:5678}
    volumes:
      - n8n_data:/home/node/.n8n
      - ./documentos:/documentos
    env_file:
      - .env

volumes:
  n8n_data:
```

### `.env.example`

```env
N8N_USER=admin
N8N_PASSWORD=troque_isso

WEBHOOK_URL=http://localhost:5678

OPENAI_API_KEY=sk-...
GEMINI_API_KEY=AIza...

PINECONE_API_KEY=...
PINECONE_INDEX=chatbot-conhecimento
PINECONE_ENVIRONMENT=us-east-1

TELEGRAM_BOT_TOKEN=...
NOME_DO_ASSISTENTE=Assistente
EMPRESA=Nome da Empresa
```

---

## Passo 2 — Configurar o Pinecone

1. Crie conta em [pinecone.io](https://pinecone.io)
2. Crie um índice:
   - Name: `chatbot-conhecimento`
   - Dimensions: `1536` (OpenAI) ou `768` (Gemini)
   - Metric: `cosine`
   - Cloud: `AWS` / Region: `us-east-1`
3. Copie a API Key para o `.env`

---

## Passo 3 — Criar bot no Telegram (opcional)

Se for usar Telegram como interface:

1. Abra o Telegram e fale com `@BotFather`
2. `/newbot` → dê um nome → copie o token
3. Cole o token no `.env` como `TELEGRAM_BOT_TOKEN`

---

## Workflow A — Indexar documentos

> Roda **uma única vez** (ou sempre que o cliente atualizar os documentos).  
> Cria os embeddings e salva no Pinecone.

Monte os nós na ordem:

---

### Nó 1 — Manual Trigger

- Tipo: **Manual Trigger**
- Descrição: acionado pelo botão "Execute" no n8n

---

### Nó 2 — Ler arquivos da pasta

- Tipo: **Read Binary Files**
- File Path: `/documentos/*.pdf`
- Ou: **Read Binary File** com caminho fixo para um PDF específico

---

### Nó 3 — Extrair texto do PDF

- Tipo: **HTTP Request** (chamada para extração)
- Alternativa mais simples: nó **Code** com lógica de chunking direto

Use o nó **Code** para simular extração e chunking (em produção, use o nó Extract From File do n8n ou uma API de extração):

```javascript
// Simula texto extraído — em produção, o n8n tem nó nativo "Extract From File"
// que extrai texto de PDFs automaticamente
const nomeArquivo = $input.first().json.fileName ?? 'documento.pdf';

// O texto real virá do nó Extract From File
// Aqui fazemos o chunking do texto extraído
const texto = $input.first().json.text ?? $input.first().json.data ?? '';

// Divide em chunks de ~500 palavras com overlap de 50
const palavras = texto.split(' ');
const CHUNK_SIZE = 500;
const OVERLAP = 50;
const chunks = [];

for (let i = 0; i < palavras.length; i += CHUNK_SIZE - OVERLAP) {
  const chunk = palavras.slice(i, i + CHUNK_SIZE).join(' ');
  if (chunk.trim().length > 50) {
    chunks.push({
      json: {
        texto: chunk,
        fonte: nomeArquivo,
        chunk_index: chunks.length,
        total_palavras: chunk.split(' ').length
      }
    });
  }
}

return chunks;
```

---

### Nó 4 — Gerar embeddings (OpenAI)

- Tipo: **HTTP Request**
- Method: `POST`
- URL: `https://api.openai.com/v1/embeddings`
- Headers:
  - `Authorization`: `Bearer {{ $env.OPENAI_API_KEY }}`
  - `Content-Type`: `application/json`
- Body (JSON):

```json
{
  "model": "text-embedding-3-small",
  "input": "{{ $json.texto }}"
}
```

> `text-embedding-3-small` custa $0.02 por 1M tokens — praticamente gratuito.

#### Alternativa — Gemini Embeddings

```json
{
  "model": "models/text-embedding-004",
  "content": { "parts": [{ "text": "{{ $json.texto }}" }] }
}
```
URL: `https://generativelanguage.googleapis.com/v1beta/models/text-embedding-004:embedContent?key={{ $env.GEMINI_API_KEY }}`

---

### Nó 5 — Montar vetor para o Pinecone

- Tipo: **Code** (JavaScript)

```javascript
const embedding = $input.first().json.data[0].embedding;
// Para Gemini: $input.first().json.embedding.values

const chunkData = $('Chunking').first().json; // ajuste o nome do nó

return [{
  json: {
    vectors: [{
      id: `${chunkData.fonte}-chunk-${chunkData.chunk_index}`,
      values: embedding,
      metadata: {
        texto: chunkData.texto,
        fonte: chunkData.fonte,
        chunk_index: chunkData.chunk_index
      }
    }]
  }
}];
```

---

### Nó 6 — Salvar no Pinecone

- Tipo: **HTTP Request**
- Method: `POST`
- URL: `https://api.pinecone.io/vectors/upsert`
- Headers:
  - `Api-Key`: `{{ $env.PINECONE_API_KEY }}`
  - `Content-Type`: `application/json`
  - `X-Pinecone-API-Version`: `2024-07`
- Body: `{{ $json }}`

> Após executar, verifique no painel do Pinecone se os vetores aparecem no índice.

---

## Workflow B — Chatbot (responde perguntas)

> Roda a cada mensagem recebida do usuário.

---

### Nó 1 — Trigger (escolha um)

#### Opção A — Telegram
- Tipo: **Telegram Trigger**
- Token: `{{ $env.TELEGRAM_BOT_TOKEN }}`
- Updates: `message`

#### Opção B — Webhook (para widget HTML)
- Tipo: **Webhook**
- Method: `POST`
- Path: `chatbot`
- Response Mode: `When Last Node Finishes`

---

### Nó 2 — Extrair pergunta do usuário

- Tipo: **Code** (JavaScript)

```javascript
const input = $input.first().json;

// Detecta se veio do Telegram ou do Webhook
let pergunta, chat_id, session_id;

if (input.message) {
  // Telegram
  pergunta = input.message.text;
  chat_id = input.message.chat.id;
  session_id = String(chat_id);
} else {
  // Webhook / widget HTML
  pergunta = input.body?.pergunta ?? input.pergunta ?? '';
  session_id = input.body?.session_id ?? input.session_id ?? 'default';
  chat_id = null;
}

return [{
  json: { pergunta, chat_id, session_id }
}];
```

---

### Nó 3 — Gerar embedding da pergunta

Mesma configuração do Nó 4 do Workflow A, mas usando `{{ $json.pergunta }}` como input.

---

### Nó 4 — Buscar contexto relevante no Pinecone

- Tipo: **HTTP Request**
- Method: `POST`
- URL: `https://api.pinecone.io/query`
- Headers:
  - `Api-Key`: `{{ $env.PINECONE_API_KEY }}`
  - `Content-Type`: `application/json`
  - `X-Pinecone-API-Version`: `2024-07`
- Body (JSON):

```json
{
  "vector": {{ $json.data[0].embedding }},
  "topK": 4,
  "includeMetadata": true
}
```

---

### Nó 5 — Montar contexto e prompt

- Tipo: **Code** (JavaScript)

```javascript
const matches = $input.first().json.matches ?? [];
const pergunta = $('Extrair pergunta').first().json.pergunta;
const chat_id = $('Extrair pergunta').first().json.chat_id;
const session_id = $('Extrair pergunta').first().json.session_id;

// Monta contexto com os trechos mais relevantes
const contexto = matches
  .filter(m => m.score > 0.7) // ignora matches irrelevantes
  .map((m, i) => `[Trecho ${i + 1} — ${m.metadata.fonte}]\n${m.metadata.texto}`)
  .join('\n\n---\n\n');

const prompt = contexto.length > 0
  ? `Você é o assistente virtual da ${process.env.EMPRESA ?? 'empresa'}. Responda a pergunta do usuário usando APENAS as informações abaixo. Se a informação não estiver nos trechos, diga que não sabe e sugira o cliente entrar em contato diretamente.\n\nINFORMAÇÕES DISPONÍVEIS:\n${contexto}\n\nPERGUNTA DO USUÁRIO: ${pergunta}`
  : `Você é o assistente virtual da ${process.env.EMPRESA ?? 'empresa'}. O usuário perguntou: "${pergunta}". Você não encontrou informações relevantes na base de conhecimento. Diga educadamente que não tem essa informação e sugira contato direto com a equipe.`;

return [{
  json: { prompt, pergunta, chat_id, session_id, teve_contexto: contexto.length > 0 }
}];
```

---

### Nó 6 — Gerar resposta com IA

- Tipo: **HTTP Request** (OpenAI)
- Method: `POST`
- URL: `https://api.openai.com/v1/chat/completions`
- Headers:
  - `Authorization`: `Bearer {{ $env.OPENAI_API_KEY }}`
  - `Content-Type`: `application/json`
- Body (JSON):

```json
{
  "model": "gpt-4o-mini",
  "max_tokens": 500,
  "temperature": 0.3,
  "messages": [
    {
      "role": "system",
      "content": "Você é um assistente prestativo. Responda sempre em português, de forma clara e objetiva. Nunca invente informações."
    },
    {
      "role": "user",
      "content": "{{ $json.prompt }}"
    }
  ]
}
```

---

### Nó 7 — Extrair resposta

- Tipo: **Code** (JavaScript)

```javascript
const resposta = $input.first().json.choices[0].message.content;
const dados = $('Montar contexto').first().json;

return [{
  json: {
    resposta,
    chat_id: dados.chat_id,
    session_id: dados.session_id,
    pergunta: dados.pergunta,
    teve_contexto: dados.teve_contexto
  }
}];
```

---

### Nó 8 — Enviar resposta (escolha um)

#### Opção A — Telegram
- Tipo: **Telegram**
- Operation: `Send Message`
- Chat ID: `{{ $json.chat_id }}`
- Text: `{{ $json.resposta }}`

#### Opção B — Resposta do Webhook
- Tipo: **Respond to Webhook**
- Response Body:

```json
{
  "resposta": "{{ $json.resposta }}",
  "teve_contexto": {{ $json.teve_contexto }}
}
```

---

## Widget HTML para embutir no site

Quando o cliente não quiser usar Telegram, entregue esse widget:

```html
<!DOCTYPE html>
<html lang="pt-BR">
<head>
  <meta charset="UTF-8">
  <title>Assistente Virtual</title>
  <style>
    * { box-sizing: border-box; margin: 0; padding: 0; }
    body { font-family: sans-serif; background: #f5f5f5; display: flex; justify-content: center; align-items: center; min-height: 100vh; }
    .chat-container { width: 420px; height: 600px; background: white; border-radius: 16px; box-shadow: 0 4px 24px rgba(0,0,0,0.12); display: flex; flex-direction: column; overflow: hidden; }
    .chat-header { background: #4f46e5; color: white; padding: 16px 20px; }
    .chat-header h3 { font-size: 16px; font-weight: 500; }
    .chat-header p { font-size: 12px; opacity: 0.8; margin-top: 2px; }
    .chat-messages { flex: 1; overflow-y: auto; padding: 16px; display: flex; flex-direction: column; gap: 12px; }
    .msg { max-width: 80%; padding: 10px 14px; border-radius: 12px; font-size: 14px; line-height: 1.5; }
    .msg.user { background: #4f46e5; color: white; align-self: flex-end; border-bottom-right-radius: 4px; }
    .msg.bot { background: #f1f0ff; color: #1f1f1f; align-self: flex-start; border-bottom-left-radius: 4px; }
    .msg.loading { opacity: 0.6; font-style: italic; }
    .chat-input { display: flex; padding: 12px 16px; border-top: 1px solid #eee; gap: 8px; }
    .chat-input input { flex: 1; padding: 10px 14px; border: 1px solid #ddd; border-radius: 8px; font-size: 14px; outline: none; }
    .chat-input input:focus { border-color: #4f46e5; }
    .chat-input button { background: #4f46e5; color: white; border: none; border-radius: 8px; padding: 10px 16px; cursor: pointer; font-size: 14px; }
    .chat-input button:disabled { opacity: 0.5; cursor: not-allowed; }
  </style>
</head>
<body>
  <div class="chat-container">
    <div class="chat-header">
      <h3>🤖 Assistente Virtual</h3>
      <p>Tire suas dúvidas sobre nossos produtos e serviços</p>
    </div>
    <div class="chat-messages" id="messages">
      <div class="msg bot">Olá! Como posso te ajudar hoje?</div>
    </div>
    <div class="chat-input">
      <input type="text" id="inputMsg" placeholder="Digite sua pergunta..." />
      <button id="sendBtn" onclick="enviarMensagem()">Enviar</button>
    </div>
  </div>

  <script>
    const WEBHOOK_URL = 'COLE_AQUI_A_URL_DO_WEBHOOK_N8N';
    const SESSION_ID = 'session-' + Math.random().toString(36).substr(2, 9);

    const messagesEl = document.getElementById('messages');
    const inputEl = document.getElementById('inputMsg');
    const sendBtn = document.getElementById('sendBtn');

    function adicionarMsg(texto, tipo) {
      const div = document.createElement('div');
      div.className = `msg ${tipo}`;
      div.textContent = texto;
      messagesEl.appendChild(div);
      messagesEl.scrollTop = messagesEl.scrollHeight;
      return div;
    }

    async function enviarMensagem() {
      const texto = inputEl.value.trim();
      if (!texto) return;

      adicionarMsg(texto, 'user');
      inputEl.value = '';
      sendBtn.disabled = true;

      const loading = adicionarMsg('Digitando...', 'bot loading');

      try {
        const res = await fetch(WEBHOOK_URL, {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({ pergunta: texto, session_id: SESSION_ID })
        });
        const data = await res.json();
        loading.remove();
        adicionarMsg(data.resposta ?? 'Desculpe, ocorreu um erro.', 'bot');
      } catch (e) {
        loading.remove();
        adicionarMsg('Erro ao conectar. Tente novamente.', 'bot');
      }

      sendBtn.disabled = false;
      inputEl.focus();
    }

    inputEl.addEventListener('keydown', e => {
      if (e.key === 'Enter') enviarMensagem();
    });
  </script>
</body>
</html>
```

Para embutir no site do cliente via iframe:
```html
<iframe src="https://URL_DO_DEPLOY/form/index.html" width="440" height="620" frameborder="0"></iframe>
```

---

## Testar localmente

```bash
# 1. Rode o Workflow A (indexar documentos) manualmente no n8n

# 2. Teste o chatbot via curl
curl -X POST http://localhost:5678/webhook/chatbot \
  -H "Content-Type: application/json" \
  -d '{
    "pergunta": "Qual é a política de devolução?",
    "session_id": "teste-123"
  }'
```

Verifique:
- [ ] Pinecone mostra vetores indexados no dashboard
- [ ] Resposta usa informações reais do documento
- [ ] Quando pergunta algo fora do doc, bot diz que não sabe
- [ ] Widget HTML funciona no browser

---

## Deploy em produção

Mesmo processo do Projeto 1 — Railway ou Render.

Após o deploy, atualize `WEBHOOK_URL` no `.env` e no `index.html` com a URL pública.

---

## Como apresentar para o cliente (pitch de 2 minutos)

1. Abra o widget no browser
2. Pergunte algo que está no documento: *"Qual o prazo de entrega?"*
3. Mostre a resposta correta vindo do documento
4. Pergunte algo que **não** está: *"Vocês vendem no exterior?"*
5. Mostre o bot dizendo que não tem essa informação e pedindo contato
6. Diga: *"Isso evita que seu cliente fique sem resposta às 2h da manhã"*

---

## Diferenciais para vender no Upwork

- **Respostas apenas com dados do cliente** — sem alucinação, sem invenção
- **Atualização simples** — troca o PDF e roda o indexador de novo
- **Funciona 24/7** sem custo de atendente
- **Integra com Telegram ou site** — dois canais, mesmo workflow

---

## Expansões para cobrar à parte

- Histórico de conversa com memória (banco de dados por session_id) — **+$150**
- Painel admin para ver todas as perguntas feitas — **+$200**
- Suporte a múltiplos idiomas — **+$100**
- Integração com WhatsApp Business — **+$400**
- Handoff para humano quando bot não sabe — **+$200**
