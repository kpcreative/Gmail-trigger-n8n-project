# AI Email Auto-Reply — n8n Workflow

An n8n workflow that automatically reads incoming Gmail messages, retrieves similar past email-reply pairs from a Supabase vector store, and uses an AI agent (via OpenRouter) to generate a context-aware, professional reply — then sends it and saves the new pair for future reference.

## How It Works

```
Step	n8n Node	Kya karta hai
1	Gmail Trigger	Naya email aaya → workflow start
2	If	Check karta hai ki sender ≠ receiver (self-loop se bachne ke liye)
3	Set content	Email body, subject, sender name nikalta hai
4	HTTP Request (Embedding)	Email ka embedding banata hai (OpenRouter API se)
5	Search (Supabase)	Purane similar emails dhundhta hai vector similarity se
6	Prepare Context	Similar examples + original email combine karta hai
7	Email Agent (AI)	AI se reply generate karta hai
8	Code (JS)	Markdown symbols hatata hai reply se
9	Telegram	Telegram pe notification bhejta hai
10	Gmail (Send & Wait)	Tujhe review ke liye email bhejta hai
11	Gmail (Final Send)	Approved reply original sender ko bhejta hai
12	HTTP Request (Final Embedding)	Final reply ka embedding banata hai
13	Supabase (Create Row)	Email + reply + embedding save karta hai future ke liye
```

## Features

- Polls Gmail every minute for new emails
- Embeds email content using `text-embedding-3-small` via OpenRouter
- Performs semantic similarity search (cosine ≥ 0.8) against past replies stored in Supabase
- Uses an AI agent (GPT / OpenRouter) to generate concise, plain-text replies
- Learns over time — saves every new email/reply pair as a new memory row

## Prerequisites

| Service | Purpose |
|---|---|
| Gmail OAuth2 | Read & send emails |
| OpenRouter | LLM inference + embeddings |
| Supabase | Vector store (`emails_memory` table with `pgvector`) |

## Supabase Setup

Create a table called `emails_memory` with the following columns:

```sql
create table emails_memory (
  id bigserial primary key,
  email_text text,
  ai_reply text,
  embedding vector(1536)
);

-- Enable vector similarity search
create or replace function match_emails(
  query_embedding vector(1536),
  match_count int
)
returns table (
  id bigint,
  email_text text,
  ai_reply text,
  similarity float
)
language sql stable
as $$
  select id, email_text, ai_reply,
         1 - (embedding <=> query_embedding) as similarity
  from emails_memory
  order by embedding <=> query_embedding
  limit match_count;
$$;
```

## Setup Instructions

1. **Import** `My workflow.json` into your n8n instance via **Workflows → Import from file**
2. **Replace placeholders** in the workflow with your real credentials:
   - `YOUR_OPENROUTER_API_KEY` → your OpenRouter API key
   - `YOUR_SUPABASE_PROJECT_ID` → your Supabase project reference ID
   - `YOUR_SUPABASE_SERVICE_ROLE_KEY` → your Supabase service role key
3. **Connect credentials** in n8n for Gmail OAuth2, OpenRouter, and Supabase
4. **Activate** the workflow

## Credentials Needed in n8n

- **Gmail OAuth2 API** — for the Gmail Trigger and reply node
- **OpenRouter account** — for the LLM chat model node
- **Supabase account** — for the `Create a row` node

> **Note:** Never commit real API keys to version control. All secrets have been replaced with placeholder values in the JSON file.

## License

MIT
