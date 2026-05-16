# Google Drive RAG Pipeline

An automated document ingestion and retrieval pipeline that keeps an AI knowledge base in sync with a Google Drive folder. Drop a file in — it gets processed, embedded, and made queryable through a chat interface instantly.

---

## How It Works

Two separate n8n workflows watch the same Google Drive folder:

**Create Flow** — triggers when a new file is added
```
Drive Trigger → Edit Fields → Download → Switch (by mimeType) → Extract → Supabase Vector Store
```

**Update Flow** — triggers when an existing file is modified
```
Drive Trigger → Edit Fields → Delete stale embeddings → Download → Switch → Extract → Supabase Vector Store
```

The delete-before-insert pattern on updates ensures no stale or duplicate vectors accumulate in the knowledge base over time.

---

## Architecture

```
Google Drive (watched folder)
        │
        ├── fileCreated trigger
        │         │
        │    Download file
        │         │
        │    Switch (mimeType)
        │    ├── PDF     → Extract text → Embed → Supabase
        │    └── GDocs   → Extract text → Embed → Supabase
        │
        └── fileUpdated trigger
                  │
            Delete old chunks (by file_id)
                  │
            Download file
                  │
            Switch (mimeType)
            ├── PDF     → Extract text → Embed → Supabase
            └── GDocs   → Extract text → Embed → Supabase

Chat Interface
        │
   OpenAI Agent (gpt-4)
        ├── Postgres Chat Memory (persistent across sessions)
        └── Supabase Vector Store (retrieve-as-tool)
```

---

## Stack

| Component | Tool |
|---|---|
| Automation | n8n |
| Vector Store | Supabase pgvector |
| Embeddings | OpenAI text-embedding-ada-002 |
| Chat Agent | OpenAI GPT (via n8n AI Agent node) |
| Chat Memory | Postgres |
| File Source | Google Drive |

---

## Metadata Stored Per Chunk

Each embedded chunk stores the following metadata, enabling filtered retrieval beyond pure semantic search:

```json
{
  "file_id": "Google Drive file ID",
  "file_name": "document.pdf",
  "file_type": "application/pdf",
  "creator_name": "Ribhu Dwivedi",
  "folder_path": "Rag test",
  "created_at": "2026-05-10T08:16:36.990Z",
  "last_modification": "2026-05-10T07:51:49.000Z",
  "ingested_at": "2026-05-16T10:00:00.000Z"
}
```

---

## Supported File Types

| Format | Handler |
|---|---|
| PDF | ExtractFromFile (pdf mode) |
| Google Docs | ExtractFromFile (text mode) |

Easily extensible to Google Sheets, DOCX, and plain text files via additional Switch branches.

---

## Setup

### Prerequisites
- n8n instance (self-hosted or cloud)
- Supabase project with pgvector enabled
- OpenAI API key
- Google account with Drive API access
- Postgres database (for chat memory)

### Supabase Setup

Run this in your Supabase SQL editor to create the required table and similarity search function:

```sql
-- Enable pgvector
create extension if not exists vector;

-- Create documents table
create table documents (
  id bigserial primary key,
  content text,
  metadata jsonb,
  embedding vector(1536)
);

-- Create similarity search function
create or replace function match_documents (
  query_embedding vector(1536),
  match_count int default 5,
  filter jsonb default '{}'
)
returns table (
  id bigint,
  content text,
  metadata jsonb,
  similarity float
)
language plpgsql
as $$
begin
  return query
  select
    id, content, metadata,
    1 - (documents.embedding <=> query_embedding) as similarity
  from documents
  where metadata @> filter
  order by documents.embedding <=> query_embedding
  limit match_count;
end;
$$;
```

### n8n Setup

1. Import `workflow.json` into your n8n instance
2. Add the following credentials:
   - **Google Drive OAuth2** — for file trigger and download
   - **Supabase API** — URL + service role key
   - **OpenAI API** — for embeddings and chat
   - **Postgres** — for chat memory
3. Update the **folder ID** in both Drive Trigger nodes to your watched folder
4. Activate both workflows

---

## Querying the Knowledge Base

The chat interface is exposed via n8n's built-in chat trigger. The AI agent:
- Searches the vector store semantically using your query
- Uses conversation history from Postgres memory for context
- Responds based only on what's in the knowledge base

Example queries:
- *"What are the rules around penalty areas in golf?"*
- *"Summarize the key points from the uploaded document"*
- *"What did we discuss in the last session?"*

---

## Key Design Decisions

**Why delete-before-insert on updates?**
Vector stores have no native concept of "update." Without explicitly deleting old chunks by `file_id`, every re-ingestion adds duplicate vectors — degrading retrieval quality over time.

**Why retrieve-as-tool instead of direct retrieval?**
Using the vector store as a tool on the agent lets the LLM decide when to search and what to search for, rather than always retrieving at the start of every turn. This is more efficient and produces better responses.

**Why store rich metadata?**
Metadata enables hybrid retrieval — you can filter by file type, creator, or date range in addition to semantic similarity. This makes the system more precise as the knowledge base grows.

---

## Roadmap

- [ ] Slack / WhatsApp interface
- [ ] Source citation in agent responses
- [ ] Google Sheets support
- [ ] Multi-folder watching
- [ ] File deletion handling (remove embeddings when file is deleted from Drive)

---

## Author

**Ribhu Dwivedi**
[LinkedIn](https://linkedin.com/in/) · [GitHub](https://github.com/)
