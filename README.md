# Voice RAG Agent

A phone-to-SMS AI sales assistant built as an n8n workflow. A customer calls a
number, speaks a question, and the system transcribes it, retrieves the relevant
product information from a vector database, generates a grounded reply with an
LLM, and texts the answer back. Built as a prototype with a fictional furniture
brand ("Lumen Living") and sample catalogue data.

## What it does

1. **Call in** — Twilio receives the call and hits an n8n webhook, which returns
   TwiML greeting the caller and recording their question.
2. **Transcribe** — the recording is downloaded and sent to Whisper
   (`whisper-large-v3` via Groq) for speech-to-text.
3. **Embed + retrieve** — the transcript is embedded with a sentence-transformers
   model (`all-MiniLM-L6-v2`), then matched against a Supabase/pgvector knowledge
   base via an RPC (`match_knowledge`). Results are reranked with lightweight
   keyword boosting so the right product surfaces.
4. **Generate** — the retrieved context and the question go to an LLM
   (`llama-3.3-70b-versatile` via Groq) with a system prompt that keeps answers
   short, grounded, and SMS-friendly.
5. **Reply** — the answer is sent back to the caller as an SMS via Twilio.

## Architecture

```
 Phone call
    │
    ▼
 Twilio ──▶ n8n Webhook (/voice-incoming) ──▶ TwiML greeting + record
    │
    ▼
 n8n Webhook (/voice-recording)
    │
    ├─▶ Download recording
    ├─▶ Whisper transcription (Groq)
    ├─▶ Embedding (HF all-MiniLM-L6-v2)
    ├─▶ Vector search (Supabase pgvector, match_knowledge) + rerank
    ├─▶ LLM reply (Groq, Llama 3.3 70B)
    └─▶ Twilio SMS back to caller
```

## Stack

- **n8n** — workflow orchestration
- **Twilio** — inbound voice + outbound SMS
- **Groq** — Whisper transcription and Llama 3.3 70B generation
- **Hugging Face Inference** — embeddings (`all-MiniLM-L6-v2`)
- **Supabase (pgvector)** — vector knowledge base with an RPC for similarity search

## Setup

1. Import `voice-rag-agent.json` into n8n (Workflows → Import from File).
2. Copy `.env.example` to `.env` and fill in your own keys. **Never commit `.env`.**
   ```bash
   cp .env.example .env
   ```
3. In n8n, add the credentials the workflow references:
   - A **Twilio** credential (replace the placeholder credential ID on the SMS node).
   - The `GROQ_API_KEY`, `HF_API_KEY`, and `SUPABASE_KEY` values, exposed to n8n as
     environment variables (the HTTP and code nodes read them via
     `{{ $env.GROQ_API_KEY }}` and `process.env.SUPABASE_KEY`).
4. Create a Supabase table with a `vector` column and a `match_knowledge` RPC that
   returns the nearest rows for a query embedding, then load your own catalogue
   content.
5. Point your Twilio number's voice webhook at the `/voice-incoming` endpoint on
   your n8n host.

## Notes

- This is a prototype. The brand, catalogue, and knowledge base are sample data.
- All keys in the workflow are placeholders. Supply your own via `.env` /
  n8n credentials.
- The vector-search node keeps retrieval deterministic and adds keyword boosting
  on top of semantic similarity so exact product mentions rank first.
