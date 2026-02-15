# Technical Design Document (TDD)
## Real-Time Collaborative AI Workspace

**Version:** 1.0  
**Last Updated:** February 15, 2026  
**Document Owner:** Engineering Team

---

## 1. Executive Summary

This document outlines the technical architecture for a production-grade Real-Time Collaborative AI Workspace. The system enables developers to collaborate with AI through two modes: shared group chat and split-screen private instances with cross-context fetching capabilities.

**Key Technical Challenges:**
- Real-time synchronization across multiple users (<100ms latency)
- Secure cross-context data retrieval with permission management
- Vector-based semantic search for RAG implementation
- Scalable WebSocket architecture (10k concurrent connections)
- Row-level security to prevent context bleed

---

## 2. Technology Stack

### 2.1 Frontend

**Framework & Core Libraries:**
- **Next.js 14.2+** (App Router with React Server Components)
  - Server-side rendering for initial page load
  - Client components for interactive features
  - API routes for backend integration
  
- **TypeScript 5.3+**
  - Strict mode enabled
  - Path aliases for clean imports
  
- **Styling:**
  - Tailwind CSS 3.4+ (utility-first styling)
  - shadcn/ui components (accessible, customizable)
  - Radix UI primitives (headless components)

**State Management:**
- **Zustand** for complex client state (split-screen layout, user preferences)
- **React Query (TanStack Query)** for server state caching
- **Supabase Realtime** for synchronized state across clients

**AI Integration:**
- **Vercel AI SDK 3.0+**
  - `useChat` hook for streaming responses
  - `streamText` for server-side streaming
  - Support for multiple providers (OpenAI, Anthropic)

**Real-time Communication:**
- **Supabase Realtime** (WebSocket-based)
  - Presence tracking (who's online)
  - Broadcast (typing indicators)
  - Postgres Changes (message sync)

### 2.2 Backend & Database

**Primary Backend:**
- **Supabase** (PostgreSQL 15+ with extensions)
  - Authentication (JWT-based)
  - Database (with Row-Level Security)
  - Realtime (WebSocket server)
  - Storage (file uploads)
  - Edge Functions (serverless compute)

**Database Extensions:**
- **pgvector** (vector similarity search)
- **pg_cron** (scheduled jobs for cleanup)
- **uuid-ossp** (UUID generation)

**API Layer:**
- Next.js API Routes (serverless functions)
- Supabase Edge Functions (Deno runtime) for compute-heavy tasks

### 2.3 AI & ML Services

**Model Aggregation:**
- **OpenRouter** (unified API for multiple providers)
  - OpenAI (GPT-4o, GPT-4o-mini)
  - Anthropic (Claude 3.5 Sonnet, Claude 3 Opus)
  - Google (Gemini 1.5 Pro)
  - Open source models (Llama 3.1 70B)

**Embedding Generation:**
- **OpenAI text-embedding-3-small** (1536 dimensions)
  - Cost-effective ($0.02 per 1M tokens)
  - Fast inference (<100ms for typical queries)

**Vector Search:**
- **pgvector** (PostgreSQL extension)
  - Cosine similarity for semantic search
  - HNSW indexing for performance

### 2.4 Infrastructure & DevOps

**Hosting:**
- **Vercel** (Next.js frontend + API routes)
  - Edge network for low latency
  - Automatic HTTPS and CDN
  - Preview deployments for PRs

**Database Hosting:**
- **Supabase Cloud** (managed PostgreSQL)
  - Automatic backups
  - Connection pooling (PgBouncer)
  - Read replicas for scaling

**Monitoring & Logging:**
- **Vercel Analytics** (Web Vitals, performance)
- **Sentry** (error tracking)
- **LogRocket** (session replay for debugging)
- **Supabase Dashboard** (database metrics)

**CI/CD:**
- **GitHub Actions**
  - Automated testing on PR
  - Deployment to Vercel on merge
  - Database migrations via Supabase CLI

---

## 3. System Architecture

### 3.1 High-Level Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                         CLIENT LAYER                         │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Next.js 14 (App Router)                             │   │
│  │  - React Server Components (initial render)          │   │
│  │  - Client Components (interactivity)                 │   │
│  │  - Zustand (local state)                             │   │
│  │  - React Query (server state cache)                  │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                            │
                            │ HTTPS / WebSocket
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                      APPLICATION LAYER                       │
│  ┌────────────────────┐         ┌──────────────────────┐    │
│  │  Vercel Edge       │         │  Supabase Realtime   │    │
│  │  - API Routes      │◄────────┤  - WebSocket Server  │    │
│  │  - AI Streaming    │         │  - Presence          │    │
│  │  - Auth Middleware │         │  - Broadcast         │    │
│  └────────────────────┘         └──────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
                            │
                            │ SQL / Realtime Protocol
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                        DATA LAYER                            │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Supabase PostgreSQL 15 + pgvector                   │   │
│  │  - Row-Level Security (RLS)                          │   │
│  │  - Vector embeddings (1536 dimensions)               │   │
│  │  - Indexed queries (room_id, user_id, vectors)      │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                            │
                            │ HTTP API
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                      EXTERNAL SERVICES                       │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │  OpenRouter  │  │  OpenAI      │  │  GitHub/Google   │  │
│  │  (AI Models) │  │  (Embeddings)│  │  (OAuth)         │  │
│  └──────────────┘  └──────────────┘  └──────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 Component Architecture

**Frontend Component Hierarchy:**
```
app/
├── (auth)/
│   ├── login/
│   └── signup/
├── (dashboard)/
│   ├── rooms/
│   │   └── [roomId]/
│   │       ├── group-chat/          # Mode 1
│   │       └── split-screen/        # Mode 2
│   └── layout.tsx                   # Shared nav
├── api/
│   ├── chat/                        # AI streaming endpoint
│   ├── embeddings/                  # Vector generation
│   └── context-fetch/               # Cross-user context
└── layout.tsx                       # Root layout
```

---

## 4. Database Schema & ERD

### 4.1 Entity Relationship Diagram

```
┌─────────────────┐
│    profiles     │
├─────────────────┤
│ id (uuid) PK    │───┐
│ email           │   │
│ display_name    │   │
│ avatar_url      │   │
│ created_at      │   │
└─────────────────┘   │
                      │
                      │ 1:N
                      │
┌─────────────────┐   │    ┌──────────────────────┐
│     rooms       │   │    │  room_participants   │
├─────────────────┤   │    ├──────────────────────┤
│ id (uuid) PK    │───┼───►│ room_id (uuid) FK    │
│ name            │   │    │ user_id (uuid) FK    │◄──┘
│ creator_id FK   │───┘    │ role (enum)          │
│ mode (enum)     │        │ joined_at            │
│ created_at      │        │ permissions (jsonb)  │
└─────────────────┘        └──────────────────────┘
        │                           │
        │ 1:N                       │
        │                           │
        ▼                           │
┌─────────────────────────────────┐│
│          messages               ││
├─────────────────────────────────┤│
│ id (uuid) PK                    ││
│ room_id (uuid) FK               │◄┘
│ user_id (uuid) FK               │
│ content (text)                  │
│ is_private (boolean)            │
│ embedding (vector(1536))        │
│ metadata (jsonb)                │
│ created_at                      │
└─────────────────────────────────┘
        │
        │ 1:N
        ▼
┌─────────────────────────────────┐
│         documents               │
├─────────────────────────────────┤
│ id (uuid) PK                    │
│ room_id (uuid) FK               │
│ uploader_id (uuid) FK           │
│ filename                        │
│ content_chunks (text[])         │
│ embeddings (vector(1536)[])     │
│ uploaded_at                     │
└─────────────────────────────────┘
```


### 4.2 Detailed Table Schemas

#### 4.2.1 profiles

```sql
CREATE TABLE profiles (
  id UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
  email TEXT UNIQUE NOT NULL,
  display_name TEXT NOT NULL,
  avatar_url TEXT,
  preferences JSONB DEFAULT '{"theme": "dark", "notifications": true}'::jsonb,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_profiles_email ON profiles(email);

-- RLS Policies
ALTER TABLE profiles ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Users can view all profiles"
  ON profiles FOR SELECT
  USING (true);

CREATE POLICY "Users can update own profile"
  ON profiles FOR UPDATE
  USING (auth.uid() = id);
```

#### 4.2.2 rooms

```sql
CREATE TYPE room_mode AS ENUM ('group_chat', 'split_screen');

CREATE TABLE rooms (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  name TEXT NOT NULL,
  creator_id UUID NOT NULL REFERENCES profiles(id) ON DELETE CASCADE,
  mode room_mode DEFAULT 'group_chat',
  settings JSONB DEFAULT '{
    "default_model": "gpt-4o",
    "max_participants": 10,
    "context_retention_days": 30
  }'::jsonb,
  invite_code TEXT UNIQUE DEFAULT encode(gen_random_bytes(8), 'hex'),
  is_public BOOLEAN DEFAULT false,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_rooms_creator ON rooms(creator_id);
CREATE INDEX idx_rooms_invite_code ON rooms(invite_code);
CREATE INDEX idx_rooms_created_at ON rooms(created_at DESC);

-- RLS Policies
ALTER TABLE rooms ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Users can view rooms they're members of"
  ON rooms FOR SELECT
  USING (
    id IN (
      SELECT room_id FROM room_participants 
      WHERE user_id = auth.uid()
    )
  );

CREATE POLICY "Users can create rooms"
  ON rooms FOR INSERT
  WITH CHECK (auth.uid() = creator_id);

CREATE POLICY "Admins can update their rooms"
  ON rooms FOR UPDATE
  USING (
    id IN (
      SELECT room_id FROM room_participants 
      WHERE user_id = auth.uid() AND role = 'admin'
    )
  );

CREATE POLICY "Admins can delete their rooms"
  ON rooms FOR DELETE
  USING (
    id IN (
      SELECT room_id FROM room_participants 
      WHERE user_id = auth.uid() AND role = 'admin'
    )
  );
```

#### 4.2.3 room_participants

```sql
CREATE TYPE participant_role AS ENUM ('admin', 'member', 'viewer');

CREATE TABLE room_participants (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  room_id UUID NOT NULL REFERENCES rooms(id) ON DELETE CASCADE,
  user_id UUID NOT NULL REFERENCES profiles(id) ON DELETE CASCADE,
  role participant_role DEFAULT 'member',
  permissions JSONB DEFAULT '{
    "can_invite": true,
    "can_upload": true,
    "context_sharing": "ask_each_time"
  }'::jsonb,
  joined_at TIMESTAMPTZ DEFAULT NOW(),
  last_seen_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(room_id, user_id)
);

-- Indexes
CREATE INDEX idx_room_participants_room ON room_participants(room_id);
CREATE INDEX idx_room_participants_user ON room_participants(user_id);
CREATE INDEX idx_room_participants_last_seen ON room_participants(last_seen_at DESC);

-- RLS Policies
ALTER TABLE room_participants ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Users can view participants in their rooms"
  ON room_participants FOR SELECT
  USING (
    room_id IN (
      SELECT room_id FROM room_participants 
      WHERE user_id = auth.uid()
    )
  );

CREATE POLICY "Users can join rooms"
  ON room_participants FOR INSERT
  WITH CHECK (auth.uid() = user_id);

CREATE POLICY "Admins can manage participants"
  ON room_participants FOR ALL
  USING (
    room_id IN (
      SELECT room_id FROM room_participants 
      WHERE user_id = auth.uid() AND role = 'admin'
    )
  );
```

#### 4.2.4 messages

```sql
CREATE TABLE messages (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  room_id UUID NOT NULL REFERENCES rooms(id) ON DELETE CASCADE,
  user_id UUID NOT NULL REFERENCES profiles(id) ON DELETE CASCADE,
  content TEXT NOT NULL,
  is_private BOOLEAN DEFAULT false,
  is_ai_response BOOLEAN DEFAULT false,
  embedding vector(1536),
  metadata JSONB DEFAULT '{
    "model": null,
    "tokens": null,
    "context_sources": []
  }'::jsonb,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  deleted_at TIMESTAMPTZ
);

-- Indexes
CREATE INDEX idx_messages_room ON messages(room_id, created_at DESC);
CREATE INDEX idx_messages_user ON messages(user_id);
CREATE INDEX idx_messages_private ON messages(room_id, user_id) WHERE is_private = true;

-- Vector similarity index (HNSW for performance)
CREATE INDEX idx_messages_embedding ON messages 
  USING hnsw (embedding vector_cosine_ops)
  WITH (m = 16, ef_construction = 64);

-- RLS Policies
ALTER TABLE messages ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Users can view group messages in their rooms"
  ON messages FOR SELECT
  USING (
    is_private = false AND
    room_id IN (
      SELECT room_id FROM room_participants 
      WHERE user_id = auth.uid()
    )
  );

CREATE POLICY "Users can view their own private messages"
  ON messages FOR SELECT
  USING (
    is_private = true AND
    user_id = auth.uid()
  );

CREATE POLICY "Users can insert messages in their rooms"
  ON messages FOR INSERT
  WITH CHECK (
    auth.uid() = user_id AND
    room_id IN (
      SELECT room_id FROM room_participants 
      WHERE user_id = auth.uid()
    )
  );

CREATE POLICY "Users can update own messages within 5 minutes"
  ON messages FOR UPDATE
  USING (
    auth.uid() = user_id AND
    created_at > NOW() - INTERVAL '5 minutes'
  );

CREATE POLICY "Users can soft-delete own messages"
  ON messages FOR UPDATE
  USING (auth.uid() = user_id)
  WITH CHECK (deleted_at IS NOT NULL);
```

#### 4.2.5 documents

```sql
CREATE TABLE documents (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  room_id UUID NOT NULL REFERENCES rooms(id) ON DELETE CASCADE,
  uploader_id UUID NOT NULL REFERENCES profiles(id) ON DELETE CASCADE,
  filename TEXT NOT NULL,
  file_path TEXT NOT NULL,
  file_size INTEGER NOT NULL,
  mime_type TEXT NOT NULL,
  content_chunks TEXT[] NOT NULL,
  embeddings vector(1536)[] NOT NULL,
  metadata JSONB DEFAULT '{
    "page_count": null,
    "language": "en"
  }'::jsonb,
  uploaded_at TIMESTAMPTZ DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_documents_room ON documents(room_id);
CREATE INDEX idx_documents_uploader ON documents(uploader_id);
CREATE INDEX idx_documents_uploaded_at ON documents(uploaded_at DESC);

-- RLS Policies
ALTER TABLE documents ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Users can view documents in their rooms"
  ON documents FOR SELECT
  USING (
    room_id IN (
      SELECT room_id FROM room_participants 
      WHERE user_id = auth.uid()
    )
  );

CREATE POLICY "Members can upload documents"
  ON documents FOR INSERT
  WITH CHECK (
    auth.uid() = uploader_id AND
    room_id IN (
      SELECT room_id FROM room_participants 
      WHERE user_id = auth.uid() AND role IN ('admin', 'member')
    )
  );
```

#### 4.2.6 context_fetch_logs (Audit Trail)

```sql
CREATE TABLE context_fetch_logs (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  room_id UUID NOT NULL REFERENCES rooms(id) ON DELETE CASCADE,
  requester_id UUID NOT NULL REFERENCES profiles(id) ON DELETE CASCADE,
  target_user_id UUID NOT NULL REFERENCES profiles(id) ON DELETE CASCADE,
  permission_status TEXT NOT NULL CHECK (permission_status IN ('granted', 'denied', 'pending')),
  context_retrieved JSONB,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_context_logs_room ON context_fetch_logs(room_id);
CREATE INDEX idx_context_logs_target ON context_fetch_logs(target_user_id, created_at DESC);

-- RLS Policies
ALTER TABLE context_fetch_logs ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Users can view logs where they're involved"
  ON context_fetch_logs FOR SELECT
  USING (
    auth.uid() = requester_id OR 
    auth.uid() = target_user_id
  );
```

---

## 5. Core System Logic

### 5.1 Real-Time Synchronization Architecture

**Technology:** Supabase Realtime (built on Phoenix Channels)

#### 5.1.1 WebSocket Connection Flow

```typescript
// Client-side connection setup
import { createClient } from '@supabase/supabase-js'

const supabase = createClient(SUPABASE_URL, SUPABASE_ANON_KEY, {
  realtime: {
    params: {
      eventsPerSecond: 10 // Rate limiting
    }
  }
})

// Subscribe to room messages
const channel = supabase
  .channel(`room:${roomId}`)
  .on(
    'postgres_changes',
    {
      event: 'INSERT',
      schema: 'public',
      table: 'messages',
      filter: `room_id=eq.${roomId}`
    },
    (payload) => {
      // New message received
      addMessageToUI(payload.new)
    }
  )
  .on('presence', { event: 'sync' }, () => {
    // User presence updated
    const state = channel.presenceState()
    updateOnlineUsers(state)
  })
  .on('broadcast', { event: 'typing' }, ({ payload }) => {
    // Typing indicator
    showTypingIndicator(payload.userId)
  })
  .subscribe()
```

#### 5.1.2 Split-Screen State Synchronization

**Challenge:** Keep 3 independent chat instances in sync without mixing private contexts.

**Solution:**
```typescript
// Each user subscribes to their own private channel
const privateChannel = supabase
  .channel(`room:${roomId}:user:${userId}`)
  .on('postgres_changes', {
    event: 'INSERT',
    schema: 'public',
    table: 'messages',
    filter: `room_id=eq.${roomId} AND user_id=eq.${userId} AND is_private=eq.true`
  }, (payload) => {
    // Only this user's private messages
    addToPrivateChat(payload.new)
  })
  .subscribe()

// Shared channel for split-screen coordination
const coordinationChannel = supabase
  .channel(`room:${roomId}:split-screen`)
  .on('broadcast', { event: 'layout-change' }, ({ payload }) => {
    // User joined/left split screen
    updateLayout(payload.activeUsers)
  })
  .subscribe()
```

### 5.2 Cross-Context Fetching Logic (Killer Feature)

#### 5.2.1 End-to-End Flow

```
┌─────────────┐
│  User A     │
│  "Check     │
│  User B's   │
│  API code"  │
└──────┬──────┘
       │
       │ 1. Send message
       ▼
┌─────────────────────────────────────┐
│  AI Streaming Endpoint              │
│  /api/chat                          │
│                                     │
│  - Detect cross-context request     │
│  - Parse target user (User B)       │
│  - Check permissions                │
└──────┬──────────────────────────────┘
       │
       │ 2. Permission check
       ▼
┌─────────────────────────────────────┐
│  Permission Service                 │
│                                     │
│  Query: room_participants           │
│  WHERE user_id = User B             │
│  AND permissions->context_sharing   │
└──────┬──────────────────────────────┘
       │
       ├─── If "always_allow" ────────┐
       │                              │
       ├─── If "ask_each_time" ───────┤
       │    (Send realtime request    │
       │     to User B, wait for      │
       │     approval)                │
       │                              │
       └─── If "never" ───────────────┤
            (Return error)            │
                                      │
       ┌──────────────────────────────┘
       │ 3. Fetch context
       ▼
┌─────────────────────────────────────┐
│  Context Retrieval Service          │
│  /api/context-fetch                 │
│                                     │
│  - Get User B's recent messages     │
│  - Perform vector similarity search │
│  - Retrieve top 5 relevant chunks   │
└──────┬──────────────────────────────┘
       │
       │ 4. Inject context
       ▼
┌─────────────────────────────────────┐
│  AI Prompt Construction             │
│                                     │
│  System: "You are helping User A.   │
│  Here is relevant context from      │
│  User B's session: [context]"       │
│                                     │
│  User Query: "Check User B's API"   │
└──────┬──────────────────────────────┘
       │
       │ 5. Generate response
       ▼
┌─────────────────────────────────────┐
│  OpenRouter API                     │
│  (GPT-4o / Claude 3.5)              │
│                                     │
│  - Stream response tokens           │
│  - Include source attribution       │
└──────┬──────────────────────────────┘
       │
       │ 6. Stream to client
       ▼
┌─────────────┐
│  User A     │
│  Receives   │
│  answer     │
│  with       │
│  citation   │
└─────────────┘
```

#### 5.2.2 Implementation Code

**API Route: /api/context-fetch**

```typescript
import { createRouteHandlerClient } from '@supabase/auth-helpers-nextjs'
import { cookies } from 'next/headers'
import { NextResponse } from 'next/server'

export async function POST(request: Request) {
  const supabase = createRouteHandlerClient({ cookies })
  const { roomId, targetUserId, query } = await request.json()

  // 1. Verify requester is in the room
  const { data: { user } } = await supabase.auth.getUser()
  if (!user) return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })

  const { data: participant } = await supabase
    .from('room_participants')
    .select('role')
    .eq('room_id', roomId)
    .eq('user_id', user.id)
    .single()

  if (!participant) {
    return NextResponse.json({ error: 'Not a room member' }, { status: 403 })
  }

  // 2. Check target user's permissions
  const { data: targetParticipant } = await supabase
    .from('room_participants')
    .select('permissions')
    .eq('room_id', roomId)
    .eq('user_id', targetUserId)
    .single()

  const contextSharing = targetParticipant?.permissions?.context_sharing || 'ask_each_time'

  if (contextSharing === 'never') {
    return NextResponse.json({ error: 'Permission denied' }, { status: 403 })
  }

  if (contextSharing === 'ask_each_time') {
    // Send realtime request to target user
    const channel = supabase.channel(`user:${targetUserId}:permissions`)
    await channel.send({
      type: 'broadcast',
      event: 'context-request',
      payload: {
        requesterId: user.id,
        requesterName: user.user_metadata.display_name,
        roomId
      }
    })

    // Wait for response (with 30s timeout)
    // Implementation would use a temporary database record or Redis
    // For brevity, assuming approval granted
  }

  // 3. Generate query embedding
  const queryEmbedding = await generateEmbedding(query)

  // 4. Vector similarity search on target user's messages
  const { data: relevantMessages } = await supabase.rpc('match_messages', {
    query_embedding: queryEmbedding,
    match_threshold: 0.7,
    match_count: 5,
    filter_room_id: roomId,
    filter_user_id: targetUserId
  })

  // 5. Log the context fetch
  await supabase.from('context_fetch_logs').insert({
    room_id: roomId,
    requester_id: user.id,
    target_user_id: targetUserId,
    permission_status: 'granted',
    context_retrieved: { message_ids: relevantMessages.map(m => m.id) }
  })

  // 6. Return context
  return NextResponse.json({
    context: relevantMessages.map(m => ({
      content: m.content,
      timestamp: m.created_at
    })),
    source: targetParticipant.user_id
  })
}

// Helper function to generate embeddings
async function generateEmbedding(text: string): Promise<number[]> {
  const response = await fetch('https://api.openai.com/v1/embeddings', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${process.env.OPENAI_API_KEY}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      model: 'text-embedding-3-small',
      input: text
    })
  })

  const data = await response.json()
  return data.data[0].embedding
}
```

**Database Function: match_messages**

```sql
CREATE OR REPLACE FUNCTION match_messages(
  query_embedding vector(1536),
  match_threshold float,
  match_count int,
  filter_room_id uuid,
  filter_user_id uuid
)
RETURNS TABLE (
  id uuid,
  content text,
  created_at timestamptz,
  similarity float
)
LANGUAGE plpgsql
AS $$
BEGIN
  RETURN QUERY
  SELECT
    messages.id,
    messages.content,
    messages.created_at,
    1 - (messages.embedding <=> query_embedding) AS similarity
  FROM messages
  WHERE messages.room_id = filter_room_id
    AND messages.user_id = filter_user_id
    AND messages.is_private = true
    AND messages.embedding IS NOT NULL
    AND 1 - (messages.embedding <=> query_embedding) > match_threshold
  ORDER BY messages.embedding <=> query_embedding
  LIMIT match_count;
END;
$$;
```

### 5.3 AI Streaming with RAG

#### 5.3.1 Chat API Route

**File: /app/api/chat/route.ts**

```typescript
import { OpenAIStream, StreamingTextResponse } from 'ai'
import { createRouteHandlerClient } from '@supabase/auth-helpers-nextjs'
import { cookies } from 'next/headers'

export const runtime = 'edge'

export async function POST(req: Request) {
  const { messages, roomId, isPrivate, targetUserId } = await req.json()
  const supabase = createRouteHandlerClient({ cookies })

  // 1. Authenticate user
  const { data: { user } } = await supabase.auth.getUser()
  if (!user) return new Response('Unauthorized', { status: 401 })

  // 2. Retrieve RAG context from uploaded documents
  const lastUserMessage = messages[messages.length - 1].content
  const queryEmbedding = await generateEmbedding(lastUserMessage)

  const { data: relevantDocs } = await supabase.rpc('match_documents', {
    query_embedding: queryEmbedding,
    match_threshold: 0.75,
    match_count: 5,
    filter_room_id: roomId
  })

  // 3. If cross-context request, fetch target user's context
  let crossContext = null
  if (targetUserId) {
    const contextResponse = await fetch(`${process.env.NEXT_PUBLIC_APP_URL}/api/context-fetch`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ roomId, targetUserId, query: lastUserMessage })
    })
    crossContext = await contextResponse.json()
  }

  // 4. Build system prompt with RAG context
  const systemPrompt = buildSystemPrompt(relevantDocs, crossContext)

  // 5. Call AI model via OpenRouter
  const response = await fetch('https://openrouter.ai/api/v1/chat/completions', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${process.env.OPENROUTER_API_KEY}`,
      'HTTP-Referer': process.env.NEXT_PUBLIC_APP_URL,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      model: 'openai/gpt-4o',
      messages: [
        { role: 'system', content: systemPrompt },
        ...messages
      ],
      stream: true
    })
  })

  // 6. Stream response back to client
  const stream = OpenAIStream(response, {
    async onCompletion(completion) {
      // Save AI response to database
      const embedding = await generateEmbedding(completion)
      
      await supabase.from('messages').insert({
        room_id: roomId,
        user_id: user.id,
        content: completion,
        is_private: isPrivate,
        is_ai_response: true,
        embedding,
        metadata: {
          model: 'gpt-4o',
          context_sources: relevantDocs?.map(d => d.filename) || []
        }
      })
    }
  })

  return new StreamingTextResponse(stream)
}

function buildSystemPrompt(docs: any[], crossContext: any): string {
  let prompt = `You are an AI assistant helping developers collaborate. 

CRITICAL RULES:
1. Answer ONLY based on provided context. If information is not in context, say "I don't have enough information to answer that accurately."
2. Always cite sources when using context (e.g., "According to [filename]...")
3. Be precise and avoid speculation.

`

  if (docs && docs.length > 0) {
    prompt += `\n--- REFERENCE DOCUMENTS ---\n`
    docs.forEach(doc => {
      prompt += `[${doc.filename}]\n${doc.content}\n\n`
    })
  }

  if (crossContext && crossContext.context) {
    prompt += `\n--- CONTEXT FROM OTHER USER ---\n`
    prompt += `The following is from another user's workspace:\n`
    crossContext.context.forEach((msg: any) => {
      prompt += `${msg.content}\n`
    })
    prompt += `\nWhen referencing this context, attribute it clearly (e.g., "Based on the other user's code...")\n`
  }

  return prompt
}
```


### 5.4 Permission Management System

#### 5.4.1 Permission States

```typescript
type ContextSharingPermission = 
  | 'always_allow'    // Auto-approve all requests
  | 'ask_each_time'   // Prompt for each request
  | 'never'           // Block all requests

interface ParticipantPermissions {
  can_invite: boolean
  can_upload: boolean
  context_sharing: ContextSharingPermission
  allowed_users?: string[]  // Whitelist for 'always_allow'
}
```

#### 5.4.2 Real-Time Permission Request Flow

```typescript
// Target user receives permission request
const permissionChannel = supabase
  .channel(`user:${userId}:permissions`)
  .on('broadcast', { event: 'context-request' }, async ({ payload }) => {
    const { requesterId, requesterName, roomId } = payload

    // Show UI modal to user
    const approved = await showPermissionModal({
      message: `${requesterName} wants to access your context to answer a question.`,
      options: ['Allow Once', 'Always Allow', 'Deny']
    })

    // Send response back
    await supabase.from('permission_responses').insert({
      requester_id: requesterId,
      target_user_id: userId,
      room_id: roomId,
      status: approved,
      expires_at: new Date(Date.now() + 30000) // 30s TTL
    })

    // Update permanent permissions if "Always Allow"
    if (approved === 'always_allow') {
      await supabase
        .from('room_participants')
        .update({
          permissions: {
            ...currentPermissions,
            context_sharing: 'always_allow',
            allowed_users: [...(currentPermissions.allowed_users || []), requesterId]
          }
        })
        .eq('user_id', userId)
        .eq('room_id', roomId)
    }
  })
  .subscribe()
```

---

## 6. Risk Mitigation Strategies

### 6.1 Hallucination Control

#### 6.1.1 System Instruction Enforcement

```typescript
const ANTI_HALLUCINATION_PROMPT = `
STRICT RULES:
1. You MUST ONLY use information from the provided context sections.
2. If the context does not contain the answer, respond EXACTLY with:
   "I don't have enough information in the current context to answer that accurately. Could you provide more details or upload relevant documents?"
3. NEVER make up code examples, API endpoints, or technical details not in context.
4. When citing context, use this format: [Source: filename.ext, line X]
5. If you're uncertain, explicitly state your confidence level: "Based on the available context, I believe... (Confidence: Medium)"
`
```

#### 6.1.2 Post-Generation Validation

```typescript
async function validateResponse(
  response: string,
  context: string[]
): Promise<{ isGrounded: boolean; score: number }> {
  // Check if response contains facts not in context
  const responseFacts = extractFactualClaims(response)
  const contextFacts = context.flatMap(extractFactualClaims)

  let groundedCount = 0
  for (const fact of responseFacts) {
    const isGrounded = contextFacts.some(cf => 
      cosineSimilarity(fact.embedding, cf.embedding) > 0.85
    )
    if (isGrounded) groundedCount++
  }

  const score = groundedCount / responseFacts.length
  return {
    isGrounded: score > 0.7,
    score
  }
}
```

#### 6.1.3 User Feedback Loop

```typescript
// After each AI response, show feedback buttons
<div className="flex gap-2 mt-2">
  <button onClick={() => reportHallucination(messageId)}>
    🚫 Inaccurate
  </button>
  <button onClick={() => confirmAccuracy(messageId)}>
    ✅ Accurate
  </button>
</div>

// Track hallucination rate
async function reportHallucination(messageId: string) {
  await supabase.from('message_feedback').insert({
    message_id: messageId,
    feedback_type: 'hallucination',
    reported_at: new Date()
  })

  // If hallucination rate > 10%, trigger alert
  const { count } = await supabase
    .from('message_feedback')
    .select('*', { count: 'exact' })
    .eq('feedback_type', 'hallucination')
    .gte('reported_at', new Date(Date.now() - 24 * 60 * 60 * 1000))

  if (count > HALLUCINATION_THRESHOLD) {
    await sendAlert('High hallucination rate detected')
  }
}
```

### 6.2 Cost Control

#### 6.2.1 Token Usage Limits

```typescript
// Per-user daily limits
const TOKEN_LIMITS = {
  free_tier: 50_000,      // ~$0.50/day
  pro_tier: 500_000,      // ~$5/day
  enterprise: Infinity
}

async function checkTokenLimit(userId: string): Promise<boolean> {
  const { data: usage } = await supabase
    .from('token_usage')
    .select('total_tokens')
    .eq('user_id', userId)
    .gte('date', new Date().toISOString().split('T')[0])
    .single()

  const userTier = await getUserTier(userId)
  return (usage?.total_tokens || 0) < TOKEN_LIMITS[userTier]
}

// Track usage after each request
async function recordTokenUsage(userId: string, tokens: number, cost: number) {
  await supabase.from('token_usage').upsert({
    user_id: userId,
    date: new Date().toISOString().split('T')[0],
    total_tokens: tokens,
    total_cost: cost
  }, {
    onConflict: 'user_id,date',
    ignoreDuplicates: false
  })
}
```

#### 6.2.2 Response Caching

```typescript
import { createHash } from 'crypto'

async function getCachedResponse(
  messages: Message[],
  context: string[]
): Promise<string | null> {
  // Create cache key from messages + context
  const cacheKey = createHash('sha256')
    .update(JSON.stringify({ messages, context }))
    .digest('hex')

  const { data } = await supabase
    .from('response_cache')
    .select('response')
    .eq('cache_key', cacheKey)
    .gte('created_at', new Date(Date.now() - 60 * 60 * 1000)) // 1 hour TTL
    .single()

  return data?.response || null
}

async function cacheResponse(
  messages: Message[],
  context: string[],
  response: string
) {
  const cacheKey = createHash('sha256')
    .update(JSON.stringify({ messages, context }))
    .digest('hex')

  await supabase.from('response_cache').insert({
    cache_key: cacheKey,
    response,
    created_at: new Date()
  })
}
```

#### 6.2.3 Model Selection Strategy

```typescript
// Route queries to cheaper models when appropriate
function selectOptimalModel(query: string, context: string[]): string {
  const queryLength = query.length
  const contextLength = context.join('').length
  const totalTokens = (queryLength + contextLength) / 4 // Rough estimate

  // Simple queries with small context -> use cheaper model
  if (totalTokens < 1000 && !query.includes('complex') && !query.includes('analyze')) {
    return 'openai/gpt-4o-mini' // $0.15/1M tokens
  }

  // Code generation or complex reasoning -> use premium model
  if (query.includes('write code') || query.includes('implement')) {
    return 'anthropic/claude-3.5-sonnet' // Best for code
  }

  // Default to balanced model
  return 'openai/gpt-4o' // $2.50/1M tokens
}
```

### 6.3 Security: Context Bleed Prevention

#### 6.3.1 Multi-Layer Isolation

```typescript
// Layer 1: Database RLS (enforced at PostgreSQL level)
// See section 4.2 for RLS policies

// Layer 2: Application-level checks
async function verifyRoomAccess(userId: string, roomId: string): Promise<boolean> {
  const { data } = await supabase
    .from('room_participants')
    .select('id')
    .eq('user_id', userId)
    .eq('room_id', roomId)
    .single()

  return !!data
}

// Layer 3: Message filtering before AI processing
async function getMessagesForContext(
  roomId: string,
  userId: string,
  isPrivate: boolean
): Promise<Message[]> {
  let query = supabase
    .from('messages')
    .select('*')
    .eq('room_id', roomId)
    .order('created_at', { ascending: false })
    .limit(20)

  if (isPrivate) {
    // Only user's own private messages
    query = query.eq('user_id', userId).eq('is_private', true)
  } else {
    // Only non-private messages
    query = query.eq('is_private', false)
  }

  const { data, error } = await query

  if (error) throw new Error('Failed to fetch messages')
  return data || []
}
```

#### 6.3.2 Sensitive Data Redaction

```typescript
// Automatically redact sensitive patterns before storing/sending
function redactSensitiveData(content: string): string {
  return content
    .replace(/\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b/g, '[EMAIL_REDACTED]')
    .replace(/\b(?:\d{3}-\d{2}-\d{4}|\d{9})\b/g, '[SSN_REDACTED]')
    .replace(/\b(?:sk|pk)_[a-zA-Z0-9]{24,}\b/g, '[API_KEY_REDACTED]')
    .replace(/\b(?:ghp|gho)_[a-zA-Z0-9]{36,}\b/g, '[GITHUB_TOKEN_REDACTED]')
    .replace(/\b(?:AKIA|ASIA)[A-Z0-9]{16}\b/g, '[AWS_KEY_REDACTED]')
}

// Apply before saving to database
async function saveMessage(message: Message) {
  message.content = redactSensitiveData(message.content)
  await supabase.from('messages').insert(message)
}
```

---

## 7. Performance Optimization

### 7.1 Database Query Optimization

#### 7.1.1 Indexing Strategy

```sql
-- Composite indexes for common queries
CREATE INDEX idx_messages_room_created ON messages(room_id, created_at DESC);
CREATE INDEX idx_messages_user_private ON messages(user_id, is_private) WHERE is_private = true;

-- Partial indexes for active rooms
CREATE INDEX idx_active_rooms ON rooms(id) WHERE updated_at > NOW() - INTERVAL '7 days';

-- Vector index tuning (HNSW parameters)
-- m: number of connections per layer (higher = better recall, slower build)
-- ef_construction: size of dynamic candidate list (higher = better quality, slower build)
CREATE INDEX idx_messages_embedding ON messages 
  USING hnsw (embedding vector_cosine_ops)
  WITH (m = 16, ef_construction = 64);
```

#### 7.1.2 Connection Pooling

```typescript
// supabase-client.ts
import { createClient } from '@supabase/supabase-js'

export const supabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
  {
    db: {
      schema: 'public',
    },
    auth: {
      persistSession: true,
      autoRefreshToken: true,
    },
    global: {
      headers: {
        'x-connection-pool': 'enabled'
      }
    }
  }
)

// Use PgBouncer for connection pooling (configured in Supabase dashboard)
// Transaction mode for short-lived connections
// Session mode for long-lived connections (Realtime)
```

### 7.2 Frontend Performance

#### 7.2.1 Code Splitting

```typescript
// app/rooms/[roomId]/split-screen/page.tsx
import dynamic from 'next/dynamic'

// Lazy load heavy components
const SplitScreenLayout = dynamic(
  () => import('@/components/SplitScreenLayout'),
  { ssr: false, loading: () => <LoadingSpinner /> }
)

const CodeEditor = dynamic(
  () => import('@/components/CodeEditor'),
  { ssr: false }
)

export default function SplitScreenPage() {
  return (
    <SplitScreenLayout>
      <CodeEditor />
    </SplitScreenLayout>
  )
}
```

#### 7.2.2 Message Virtualization

```typescript
import { useVirtualizer } from '@tanstack/react-virtual'

function MessageList({ messages }: { messages: Message[] }) {
  const parentRef = useRef<HTMLDivElement>(null)

  const virtualizer = useVirtualizer({
    count: messages.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 80, // Average message height
    overscan: 5 // Render 5 extra items above/below viewport
  })

  return (
    <div ref={parentRef} className="h-full overflow-auto">
      <div style={{ height: `${virtualizer.getTotalSize()}px` }}>
        {virtualizer.getVirtualItems().map((virtualItem) => (
          <div
            key={virtualItem.key}
            style={{
              position: 'absolute',
              top: 0,
              left: 0,
              width: '100%',
              transform: `translateY(${virtualItem.start}px)`
            }}
          >
            <MessageItem message={messages[virtualItem.index]} />
          </div>
        ))}
      </div>
    </div>
  )
}
```

### 7.3 Caching Strategy

```typescript
// React Query configuration
import { QueryClient } from '@tanstack/react-query'

export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 5 * 60 * 1000, // 5 minutes
      cacheTime: 10 * 60 * 1000, // 10 minutes
      refetchOnWindowFocus: false,
      retry: 1
    }
  }
})

// Cache room data aggressively
function useRoom(roomId: string) {
  return useQuery({
    queryKey: ['room', roomId],
    queryFn: () => fetchRoom(roomId),
    staleTime: 30 * 60 * 1000 // 30 minutes (rooms rarely change)
  })
}

// Cache messages with shorter TTL
function useMessages(roomId: string) {
  return useQuery({
    queryKey: ['messages', roomId],
    queryFn: () => fetchMessages(roomId),
    staleTime: 1 * 60 * 1000 // 1 minute
  })
}
```

---

## 8. Deployment Architecture

### 8.1 Infrastructure Diagram

```
┌─────────────────────────────────────────────────────────┐
│                    CLOUDFLARE CDN                        │
│              (Static Assets, DDoS Protection)            │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│                   VERCEL EDGE NETWORK                    │
│  ┌──────────────────────────────────────────────────┐   │
│  │  Next.js App (Multiple Regions)                  │   │
│  │  - us-east-1 (Primary)                           │   │
│  │  - eu-west-1 (Europe)                            │   │
│  │  - ap-southeast-1 (Asia)                         │   │
│  └──────────────────────────────────────────────────┘   │
└────────────────────┬────────────────────────────────────┘
                     │
        ┌────────────┴────────────┐
        │                         │
        ▼                         ▼
┌──────────────────┐    ┌──────────────────────┐
│  SUPABASE        │    │  OPENROUTER          │
│  (us-east-1)     │    │  (AI Gateway)        │
│                  │    │                      │
│  - PostgreSQL    │    │  - OpenAI            │
│  - Realtime      │    │  - Anthropic         │
│  - Auth          │    │  - Google            │
│  - Storage       │    └──────────────────────┘
└──────────────────┘
        │
        │ Replication
        ▼
┌──────────────────┐
│  SUPABASE        │
│  (eu-west-1)     │
│  Read Replica    │
└──────────────────┘
```

### 8.2 Environment Configuration

```bash
# .env.production
NEXT_PUBLIC_SUPABASE_URL=https://your-project.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJhbGc...
SUPABASE_SERVICE_ROLE_KEY=eyJhbGc... # Server-side only

OPENROUTER_API_KEY=sk-or-v1-...
OPENAI_API_KEY=sk-... # For embeddings

NEXT_PUBLIC_APP_URL=https://your-app.vercel.app

# Rate limiting
RATE_LIMIT_REQUESTS_PER_MINUTE=100
RATE_LIMIT_TOKENS_PER_DAY=50000

# Feature flags
ENABLE_SPLIT_SCREEN=true
ENABLE_CROSS_CONTEXT=true
MAX_ROOM_PARTICIPANTS=10
```

### 8.3 CI/CD Pipeline

```yaml
# .github/workflows/deploy.yml
name: Deploy to Production

on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '20'
      - run: npm ci
      - run: npm run lint
      - run: npm run type-check
      - run: npm test

  deploy:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: amondnet/vercel-action@v25
        with:
          vercel-token: ${{ secrets.VERCEL_TOKEN }}
          vercel-org-id: ${{ secrets.VERCEL_ORG_ID }}
          vercel-project-id: ${{ secrets.VERCEL_PROJECT_ID }}
          vercel-args: '--prod'

  migrate-db:
    needs: deploy
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - run: npx supabase db push --db-url ${{ secrets.DATABASE_URL }}
```

---

## 9. Monitoring & Observability

### 9.1 Key Metrics Dashboard

```typescript
// Metrics to track in real-time
interface SystemMetrics {
  // Performance
  avgMessageLatency: number        // Target: <100ms
  p95MessageLatency: number        // Target: <150ms
  avgAIResponseTime: number        // Target: <5s
  
  // Usage
  activeUsers: number
  activeRooms: number
  messagesPerMinute: number
  aiRequestsPerMinute: number
  
  // Errors
  errorRate: number                // Target: <1%
  websocketDisconnects: number
  failedAIRequests: number
  
  // Costs
  totalTokensUsed: number
  estimatedCost: number
  
  // Quality
  hallucinationRate: number        // Target: <5%
  userSatisfactionScore: number    // Target: >4/5
}
```

### 9.2 Alerting Rules

```typescript
// alerts.config.ts
export const alerts = [
  {
    name: 'High Error Rate',
    condition: 'errorRate > 0.05', // 5%
    severity: 'critical',
    action: 'page_oncall'
  },
  {
    name: 'Slow AI Responses',
    condition: 'p95AIResponseTime > 10000', // 10s
    severity: 'warning',
    action: 'slack_notification'
  },
  {
    name: 'Database Connection Pool Exhausted',
    condition: 'dbConnectionsUsed > 90',
    severity: 'critical',
    action: 'page_oncall'
  },
  {
    name: 'High Hallucination Rate',
    condition: 'hallucinationRate > 0.10', // 10%
    severity: 'warning',
    action: 'email_team'
  },
  {
    name: 'Cost Spike',
    condition: 'hourlyTokenCost > 100', // $100/hour
    severity: 'warning',
    action: 'slack_notification'
  }
]
```

---

## 10. Testing Strategy

### 10.1 Unit Tests

```typescript
// __tests__/context-fetch.test.ts
import { checkPermissions, fetchUserContext } from '@/lib/context-fetch'

describe('Context Fetching', () => {
  it('should deny access when permission is "never"', async () => {
    const result = await checkPermissions({
      requesterId: 'user-a',
      targetUserId: 'user-b',
      roomId: 'room-1'
    })
    
    expect(result.allowed).toBe(false)
    expect(result.reason).toBe('Permission denied by user')
  })

  it('should retrieve top 5 relevant messages', async () => {
    const context = await fetchUserContext({
      targetUserId: 'user-b',
      query: 'API endpoint for authentication',
      roomId: 'room-1'
    })
    
    expect(context.messages).toHaveLength(5)
    expect(context.messages[0].similarity).toBeGreaterThan(0.7)
  })
})
```

### 10.2 Integration Tests

```typescript
// __tests__/integration/split-screen.test.ts
import { createClient } from '@supabase/supabase-js'

describe('Split-Screen Mode', () => {
  let supabase: any
  let userA: any
  let userB: any

  beforeAll(async () => {
    supabase = createClient(TEST_SUPABASE_URL, TEST_SUPABASE_KEY)
    userA = await createTestUser('user-a@test.com')
    userB = await createTestUser('user-b@test.com')
  })

  it('should isolate private messages between users', async () => {
    const room = await createTestRoom()
    await joinRoom(userA, room.id)
    await joinRoom(userB, room.id)

    // User A sends private message
    await sendMessage(userA, room.id, 'Private message A', { isPrivate: true })

    // User B should not see it
    const messagesForB = await getMessages(userB, room.id)
    expect(messagesForB).not.toContainEqual(
      expect.objectContaining({ content: 'Private message A' })
    )
  })
})
```

### 10.3 Load Testing

```typescript
// load-test/websocket-stress.ts
import WebSocket from 'ws'

async function stressTestWebSockets() {
  const connections: WebSocket[] = []
  const TARGET_CONNECTIONS = 10000

  for (let i = 0; i < TARGET_CONNECTIONS; i++) {
    const ws = new WebSocket('wss://your-app.supabase.co/realtime/v1')
    
    ws.on('open', () => {
      ws.send(JSON.stringify({
        event: 'phx_join',
        topic: `room:test-${i % 100}`,
        payload: {},
        ref: i
      }))
    })

    connections.push(ws)

    // Stagger connections to avoid overwhelming server
    if (i % 100 === 0) {
      await new Promise(resolve => setTimeout(resolve, 100))
    }
  }

  console.log(`Established ${connections.length} WebSocket connections`)

  // Simulate message traffic
  setInterval(() => {
    const randomWs = connections[Math.floor(Math.random() * connections.length)]
    randomWs.send(JSON.stringify({
      event: 'message',
      payload: { content: 'Test message' }
    }))
  }, 10) // 100 messages/second
}
```

---

## 11. Security Considerations

### 11.1 Authentication Flow

```typescript
// Secure JWT handling
import { jwtVerify } from 'jose'

export async function verifyToken(token: string) {
  try {
    const { payload } = await jwtVerify(
      token,
      new TextEncoder().encode(process.env.SUPABASE_JWT_SECRET)
    )
    return payload
  } catch (error) {
    throw new Error('Invalid token')
  }
}

// Middleware to protect API routes
export async function authMiddleware(req: Request) {
  const token = req.headers.get('Authorization')?.replace('Bearer ', '')
  
  if (!token) {
    return new Response('Unauthorized', { status: 401 })
  }

  const payload = await verifyToken(token)
  
  // Attach user to request
  req.user = payload
  return null // Continue to handler
}
```

### 11.2 Rate Limiting

```typescript
import { Ratelimit } from '@upstash/ratelimit'
import { Redis } from '@upstash/redis'

const ratelimit = new Ratelimit({
  redis: Redis.fromEnv(),
  limiter: Ratelimit.slidingWindow(100, '1 m'), // 100 requests per minute
  analytics: true
})

export async function rateLimitMiddleware(req: Request) {
  const ip = req.headers.get('x-forwarded-for') || 'unknown'
  const { success, limit, remaining } = await ratelimit.limit(ip)

  if (!success) {
    return new Response('Rate limit exceeded', {
      status: 429,
      headers: {
        'X-RateLimit-Limit': limit.toString(),
        'X-RateLimit-Remaining': remaining.toString()
      }
    })
  }

  return null
}
```

### 11.3 Input Validation

```typescript
import { z } from 'zod'

const MessageSchema = z.object({
  content: z.string().min(1).max(10000),
  roomId: z.string().uuid(),
  isPrivate: z.boolean().optional()
})

export async function validateMessage(data: unknown) {
  try {
    return MessageSchema.parse(data)
  } catch (error) {
    throw new Error('Invalid message format')
  }
}
```

---

## 12. Future Enhancements (Post-V1)

### 12.1 Advanced Features Roadmap

1. **Voice/Video Integration**
   - WebRTC for peer-to-peer calls
   - AI transcription of conversations
   - Real-time translation

2. **IDE Integration**
   - VS Code extension
   - JetBrains plugin
   - Direct code sync from editor

3. **Advanced Analytics**
   - Team productivity metrics
   - AI usage patterns
   - Collaboration heatmaps

4. **Custom AI Agents**
   - User-trained models on team codebase
   - Specialized agents (code review, testing, documentation)

5. **Mobile Apps**
   - React Native iOS/Android apps
   - Offline mode with sync

### 12.2 Scalability Improvements

```typescript
// Horizontal scaling with Redis pub/sub
import { Redis } from 'ioredis'

const redis = new Redis(process.env.REDIS_URL)

// Publish message to all server instances
export async function broadcastMessage(roomId: string, message: Message) {
  await redis.publish(`room:${roomId}`, JSON.stringify(message))
}

// Subscribe to messages on each server instance
redis.subscribe('room:*')
redis.on('message', (channel, message) => {
  const roomId = channel.split(':')[1]
  const parsedMessage = JSON.parse(message)
  
  // Broadcast to connected WebSocket clients
  broadcastToRoom(roomId, parsedMessage)
})
```

---

## 13. Appendix

### 13.1 API Reference

**Endpoint:** `POST /api/chat`

Request:
```json
{
  "messages": [
    { "role": "user", "content": "Explain async/await" }
  ],
  "roomId": "uuid",
  "isPrivate": false,
  "targetUserId": "uuid" // Optional, for cross-context
}
```

Response: Server-Sent Events (SSE) stream
```
data: {"token": "Async"}
data: {"token": "/await"}
data: {"token": " is"}
...
data: [DONE]
```

### 13.2 Database Backup Strategy

```sql
-- Automated daily backups (configured in Supabase)
-- Point-in-time recovery: 7-day window
-- Manual backup command:
pg_dump -h db.your-project.supabase.co \
  -U postgres \
  -d postgres \
  -F c \
  -f backup_$(date +%Y%m%d).dump

-- Restore command:
pg_restore -h db.your-project.supabase.co \
  -U postgres \
  -d postgres \
  -c \
  backup_20260215.dump
```

### 13.3 Glossary

- **RLS:** Row-Level Security, PostgreSQL feature for fine-grained access control
- **RAG:** Retrieval-Augmented Generation, technique to ground AI in actual documents
- **Vector Embedding:** Numerical representation of text for semantic similarity
- **HNSW:** Hierarchical Navigable Small World, algorithm for fast vector search
- **WebSocket:** Protocol for bidirectional real-time communication
- **SSE:** Server-Sent Events, protocol for server-to-client streaming
- **JWT:** JSON Web Token, standard for secure authentication

---

**Document Status:** Ready for Implementation  
**Last Updated:** February 15, 2026  
**Next Review:** March 15, 2026  
**Approved By:** Engineering Lead, Security Team
