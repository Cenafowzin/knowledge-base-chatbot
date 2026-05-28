# Knowledge Base Chatbot — n8n + RAG + OpenAI

> An AI-powered chatbot that answers questions using **only** your company's documents — no hallucinations, no made-up answers.

![n8n](https://img.shields.io/badge/n8n-self--hosted-orange?logo=n8n)
![OpenAI](https://img.shields.io/badge/OpenAI-GPT--4o--mini-green?logo=openai)
![Pinecone](https://img.shields.io/badge/Pinecone-Vector%20DB-blue)
![Docker](https://img.shields.io/badge/Docker-ready-2496ED?logo=docker)
![License](https://img.shields.io/badge/license-MIT-lightgrey)

---

## What It Does

This project implements a **Retrieval-Augmented Generation (RAG)** pipeline that turns your static documents into a live, intelligent support agent:

1. **Ingest your documents** — PDFs, FAQs, manuals, SOPs, product sheets
2. **Index content** into a vector database (Pinecone) for semantic search
3. **User asks a question** via the embeddable chat widget or Telegram
4. **System retrieves** the most relevant document excerpts using similarity search
5. **AI answers** using *only* the retrieved content — never from general internet knowledge

The result: a chatbot that is **accurate, on-brand, and fully auditable**.

---

## Use Cases

| Industry | Example Application |
|---|---|
| E-commerce | FAQ bot for shipping, returns, warranty, sizing |
| SaaS / Tech | Product documentation assistant |
| Healthcare | Patient FAQ respecting only official clinic documents |
| Legal / Compliance | Internal policy Q&A for employees |
| HR & Onboarding | Employee handbook assistant |
| Real Estate | Property listings & buyer FAQ bot |
| Education | Course content assistant for students |

---

## Tech Stack

| Component | Technology | Notes |
|---|---|---|
| Automation Engine | [n8n](https://n8n.io) (self-hosted) | Visual workflow builder |
| AI Chat Model | OpenAI GPT-4o-mini | Fast, cost-effective |
| Embeddings | OpenAI text-embedding-3-small | 1536 dimensions |
| Vector Database | [Pinecone](https://pinecone.io) | Free tier supports this project |
| Chat Interface | HTML Widget (embeddable) | Iframe or floating button |
| Messaging Channel | Telegram Bot (optional) | Same backend, zero extra cost |
| Deployment | Docker / Railway / Render | One-command setup |

---

## Architecture

```
┌──────────────────────────────────────────────────────────────────────────┐
│                         WORKFLOW 1 — Document Indexer                    │
│                           (run once, re-run on updates)                  │
│                                                                          │
│  [Manual Trigger]                                                        │
│       │                                                                  │
│       ▼                                                                  │
│  [Read /documents]  →  PDF, TXT, MD files                                │
│       │                                                                  │
│       ▼                                                                  │
│  [Chunk Text]       →  500-char chunks, 80-char overlap                  │
│       │                                                                  │
│       ▼                                                                  │
│  [OpenAI Embeddings]  →  text-embedding-3-small (1536-dim vector)        │
│       │                                                                  │
│       ▼                                                                  │
│  [Build Vector]     →  { id, values, metadata: { text, fileName } }      │
│       │                                                                  │
│       ▼                                                                  │
│  [Pinecone Upsert]  →  Stored in "knowledge-base" namespace              │
└──────────────────────────────────────────────────────────────────────────┘

                              ▼ at query time ▼

┌──────────────────────────────────────────────────────────────────────────┐
│                         WORKFLOW 2 — Chatbot (RAG)                       │
│                           (always-on webhook endpoint)                   │
│                                                                          │
│  User types question                                                     │
│       │                                                                  │
│       ▼                                                                  │
│  [Webhook POST /chat]  ←────────  HTML Widget  or  Telegram              │
│       │                                                                  │
│       ▼                                                                  │
│  [Extract Question]   →  { question, session_id }                        │
│       │                                                                  │
│       ▼                                                                  │
│  [Embed Question]     →  OpenAI converts question to vector              │
│       │                                                                  │
│       ▼                                                                  │
│  [Query Pinecone]     →  Top-5 most relevant chunks                      │
│       │                                                                  │
│       ▼                                                                  │
│  [Build RAG Prompt]   →  System: "Answer using ONLY this context…"       │
│       │                                                                  │
│       ▼                                                                  │
│  [GPT-4o-mini]        →  Grounded, accurate answer                       │
│       │                                                                  │
│       ▼                                                                  │
│  [Respond to Webhook] →  { "answer": "…" }  →  Widget / Telegram        │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## Prerequisites

- [Docker](https://docs.docker.com/get-docker/) installed locally (or a cloud account)
- [Pinecone account](https://pinecone.io) — free tier is sufficient
- [OpenAI API key](https://platform.openai.com/api-keys)
- A domain or tunnel (e.g., [ngrok](https://ngrok.com)) for the webhook URL
- *(Optional)* A Telegram bot token from [@BotFather](https://t.me/BotFather)

---

## Quick Start

### 1. Clone the repository

```bash
git clone https://github.com/your-username/knowledge-base-chatbot.git
cd knowledge-base-chatbot
```

### 2. Configure environment variables

```bash
cp .env.example .env
```

Open `.env` and fill in your credentials:

```env
N8N_USER=admin
N8N_PASSWORD=your_secure_password

OPENAI_API_KEY=sk-...
PINECONE_API_KEY=...
PINECONE_INDEX=knowledge-chatbot
PINECONE_ENVIRONMENT=us-east-1

WEBHOOK_URL=https://your-domain.com   # or ngrok URL
```

### 3. Start n8n with Docker

```bash
docker compose up -d
```

n8n will be available at `http://localhost:5678`.

### 4. Import the workflows

1. Log in to n8n at `http://localhost:5678`
2. Go to **Workflows → Import from file**
3. Import `workflows/index-documents.json`
4. Import `workflows/chatbot.json`

### 5. Create a Pinecone index

In your Pinecone dashboard, create a new index:
- **Name:** `knowledge-chatbot`
- **Dimensions:** `1536`
- **Metric:** `cosine`

### 6. Add your documents

Drop your PDF, TXT, or Markdown files into the `documents/` folder.

### 7. Run the indexer

In n8n, open the **"Knowledge Base — Index Documents"** workflow and click **Execute Workflow**. Watch the logs — each chunk will be upserted to Pinecone.

### 8. Activate the chatbot

Open the **"Knowledge Base — Chatbot (RAG)"** workflow and toggle it **Active**. Your webhook endpoint is now live at:

```
POST https://your-domain.com/webhook/chat
```

### 9. Configure the widget

Edit `widget/index.html` and replace the placeholder:

```javascript
const WEBHOOK_URL = 'https://your-domain.com/webhook/chat';
```

Embed the widget on your site using an `<iframe>`:

```html
<iframe
  src="https://your-domain.com/widget/index.html"
  width="420"
  height="640"
  style="border: none; border-radius: 20px;"
  title="AI Assistant"
></iframe>
```

---

## Configuration Reference

| Variable | Required | Description |
|---|---|---|
| `N8N_USER` | Yes | n8n admin username |
| `N8N_PASSWORD` | Yes | n8n admin password (use a strong one) |
| `WEBHOOK_URL` | Yes | Public URL where n8n is reachable |
| `OPENAI_API_KEY` | Yes | OpenAI secret key (starts with `sk-`) |
| `GEMINI_API_KEY` | No | Alternative to OpenAI (Google Gemini) |
| `PINECONE_API_KEY` | Yes | Pinecone API key |
| `PINECONE_INDEX` | Yes | Name of your Pinecone index |
| `PINECONE_ENVIRONMENT` | Yes | Pinecone environment (e.g. `us-east-1`) |
| `TELEGRAM_BOT_TOKEN` | No | Required only for Telegram channel |
| `ASSISTANT_NAME` | No | Display name of the bot (default: `Assistant`) |
| `COMPANY_NAME` | No | Used in system prompts and UI |

---

## Project Structure

```
knowledge-base-chatbot/
├── docker-compose.yml          # n8n container definition
├── .env.example                # Environment variable template
├── documents/                  # Drop your PDFs and text files here
│   └── .gitkeep
├── workflows/
│   ├── index-documents.json    # Workflow 1: document ingestion
│   └── chatbot.json            # Workflow 2: RAG chatbot
└── widget/
    └── index.html              # Embeddable chat UI
```

---

## Deployment

### Option A — Railway (recommended, free tier available)

1. Push your repository to GitHub
2. Go to [railway.app](https://railway.app) and click **New Project → Deploy from GitHub**
3. Select your repo; Railway auto-detects `docker-compose.yml`
4. Add all environment variables in the Railway dashboard
5. Railway provides a public HTTPS URL — use it as `WEBHOOK_URL`

### Option B — Render

1. Go to [render.com](https://render.com) and create a **New Web Service**
2. Connect your GitHub repo
3. Set **Docker** as the environment
4. Add environment variables under **Environment**
5. Deploy — Render provides a `.onrender.com` public URL

### Option C — VPS (DigitalOcean, Linode, Hetzner)

```bash
# On your server
git clone https://github.com/your-username/knowledge-base-chatbot.git
cd knowledge-base-chatbot
cp .env.example .env
nano .env          # fill in your values
docker compose up -d
```

Use [Caddy](https://caddyserver.com) or Nginx to set up HTTPS and reverse-proxy port 5678.

---

## Updating the Knowledge Base

Updating the chatbot's knowledge is as simple as:

1. Add, remove, or replace files in the `documents/` folder
2. Re-run the **"Index Documents"** workflow in n8n
3. Done — the chatbot immediately uses the updated information

No code changes required.

---

## Cost Estimate

| Service | Usage | Estimated Monthly Cost |
|---|---|---|
| n8n (self-hosted on Railway) | Up to 5k executions/mo | $0 (free tier) |
| OpenAI Embeddings | 100 document chunks | ~$0.002 (one-time indexing) |
| OpenAI GPT-4o-mini | 1,000 chat questions | ~$0.15 |
| Pinecone | Up to 100k vectors | $0 (free tier) |
| **Total** | **Typical small business** | **< $1/month** |

---

## Add-on Services

Need more power? These optional enhancements are available:

| Feature | Description | Price |
|---|---|---|
| **Conversation Memory** | Bot remembers previous messages within a session | +$150 |
| **Admin Dashboard** | View chat history, top questions, unanswered queries | +$200 |
| **Multi-Language Support** | Auto-detect user language, respond in kind | +$100 |
| **WhatsApp Business** | Same chatbot, delivered via WhatsApp | +$400 |
| **Human Handoff** | Escalate to live agent when confidence is low | +$200 |
| **Analytics & Reporting** | Weekly email report of chatbot performance | +$150 |
| **Custom AI Persona** | Branded name, tone, avatar, fallback messages | +$75 |

---

## Why This Solution Beats Traditional Chatbots

| Feature | Rule-based Chatbot | This Solution |
|---|---|---|
| Setup time | Days / weeks | Hours |
| Handles unexpected questions | No | Yes |
| Answers are grounded in facts | Sometimes | Always |
| Easy to update | Requires developer | Drop a PDF file |
| Scales with your content | No | Yes (vector search) |
| Works in multiple languages | Requires rewrites | Yes (auto-detect) |
| Cost per query | Low | Very low (~$0.00015) |

**Grounded answers** — the AI is constrained to answer using only your documents. If the answer is not in the knowledge base, it says so honestly rather than inventing information.

**Easy updates** — your marketing team can update the bot's knowledge without touching code. Add a new product? Drop the PDF in the folder and re-run the indexer.

**Dual-channel** — the exact same n8n backend powers both the embeddable web widget and Telegram. Adding more channels (WhatsApp, Slack, SMS) is straightforward.

---

## Security Notes

- The n8n dashboard is protected by HTTP Basic Auth (username + password in `.env`)
- API keys are never exposed to the frontend widget
- Pinecone queries are server-side only
- CORS headers on the webhook allow cross-origin requests from your widget host only (customizable)
- All traffic should be served over HTTPS in production

---

## License

MIT — use freely in commercial and personal projects. Attribution appreciated but not required.

---

## About the Author

Built with care by a freelance automation engineer specializing in **AI-powered business workflows** using n8n, OpenAI, and modern cloud infrastructure.

Available for custom projects on [Upwork](https://www.upwork.com). Typical delivery time: **3–5 business days**.

> "The goal is not just to answer questions — it is to give your customers instant, accurate answers at 3 AM, in any language, without a support team on standby."
