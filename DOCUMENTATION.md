# CogniBook (AI Chatbot) Technical Documentation

CogniBook is an open-source NotebookLM alternative for students. It allows grouping PDFs into workspaces and features auto-generated mindmaps, spaced-repetition flashcards, two-host podcast summaries, and RAG chat with cited passages highlighted inside the source PDF.

## 🏗️ Architecture Overview

The application is built on modern React and Node.js technologies with a strong emphasis on real-time features and secure data access.

### Core Stack
- **Framework**: Next.js 16 (App Router)
- **Database**: PostgreSQL (via InsForge SDK) with `pgvector` for embeddings
- **Authentication**: Better Auth with email + password sign-in
- **Styling**: Tailwind CSS 4 + shadcn/ui components
- **PDF Processing**: `pdfjs-dist` (server-side extraction)
- **AI Models**: Integration via OpenRouter for Embeddings and Chat Completions

### Data Flow & Security
1. **Authentication:** Better Auth handles sign-in and generates a same-origin cookie. `/api/backend-token` signs an HS256 bridge JWT.
2. **Database Access:** The Postgres client receives the `edgeFunctionToken`. Row Level Security (RLS) policies in Postgres read the `sub` claim via `requesting_user_id()` ensuring users only access their own data.
3. **RAG Pipeline:**
   - **Upload:** PDF is uploaded (`/api/documents/upload`), saved to storage, and parsed.
   - **Chunking:** `pdfjs-dist` chunks pages into ~800 character segments.
   - **Embedding:** Chunks are embedded using `client.ai.embeddings.create` (batches of 16) and stored in the `document_chunks` table with `vector(1536)` and an `HNSW` index.
   - **Retrieval & Chat:** User queries `/api/chat` (streaming NDJSON). The query is embedded, and `match_document_chunks` RPC fetches the most relevant chunks. The completion is streamed back with citations.

---

## 🗄️ Database Schema

The database schema (initialized via `migrations/db_init.sql`) defines the following core tables with strict RLS policies:

### `workspaces`
Groups related documents and features.
- `id` (UUID), `user_id` (Text), `name` (Text)
- Stores cached LLM artifacts: `mindmap_markdown`, `audio_url`, `audio_script`

### `documents`
Tracks uploaded PDFs.
- `id` (UUID), `user_id` (Text), `workspace_id` (UUID)
- `file_name`, `file_size`, `mime_type`
- `storage_bucket` (default: 'pdf-documents'), `storage_key`, `status`
- `summary`, `suggested_questions`

### `document_chunks`
Stores text segments and vector embeddings for RAG.
- `id` (UUID), `document_id` (UUID), `user_id` (Text)
- `chunk_index`, `content`, `page_number`
- `embedding` (vector 1536) indexed with HNSW for high recall vector similarity search.

### `chat_sessions` & `chat_messages`
Manages user conversations and shareable chat states.
- **Sessions:** `workspace_id`, `title`, `document_ids`, `share_token` (for shared link access)
- **Messages:** `role`, `content`, `citations` (JSON), `sort_order`

### `document_flashcards`
Manages spaced-repetition (SM-2 lite) flashcards.
- `question`, `answer`, `due_at`, `ease`, `interval_days`, `reps`

---

## 🎧 Audio Overview Feature
The Audio tab generates a two-host podcast summary of the PDFs in a workspace.

1. **Script Generation:** `lib/ai/audio-script.ts` calls the chat completion model with a producer-style prompt (adapted from open-notebooklm) to create a conversational script between two hosts (Sarah and Mike).
2. **TTS Synthesis:** `lib/audio/tts.ts` converts the script to speech using OpenAI TTS (`gpt-4o-mini-tts`) with distinct voices for each host.
3. **Storage:** The audio frames are concatenated and stored in the `audio-overviews` bucket (public read).

---

## 📂 Key File Structure

- **`app/`**: Next.js App Router structure containing routes for `api`, `auth`, `chat`, `documents`, `workspaces`, and `share`.
- **`components/`**: React UI components (e.g., `chat-shell.tsx`, `pdf-viewer.tsx`, `mindmap-view.tsx`, `flashcards-modal.tsx`).
- **`lib/`**: Core application logic.
  - `backend.ts`: Initializes the authenticated Postgres client.
  - `auth.ts` / `auth-client.ts`: Better Auth configuration.
  - `ai/`, `rag/`, `pdf/`, `audio/`: Specialized modules for feature pipelines.
- **`migrations/`**: Contains `db_init.sql` for Postgres initialization, schema, RLS, and RPC functions.

## 🚀 Setup & Deployment
To run the project locally, ensure you have a Postgres project with AI capabilities and pgvector installed.
Run `npm run setup` to trigger `auth:migrate` and create storage buckets via `@insforge/cli`. Start the development server with `npm run dev`.
