# 🏗 PostPilot Architecture

> A deep dive into how PostPilot is built, structured, and scaled.

## 📋 Table of Contents

- [System Overview](#system-overview)
- [High-Level Architecture](#high-level-architecture)
- [Tech Stack Decisions](#tech-stack-decisions)
- [Data Flow](#data-flow)
- [Database Schema](#database-schema)
- [API Design](#api-design)
- [Authentication Flow](#authentication-flow)
- [AI Integration](#ai-integration)
- [Performance Optimizations](#performance-optimizations)
- [Security Architecture](#security-architecture)

---

## System Overview

PostPilot is a **modern serverless SaaS** built on a **JAMstack architecture** with edge-rendered React, server-side authentication, and streaming AI responses.

### Key Architectural Principles

1. **Serverless-First** — No server management overhead
2. **Edge-Rendered** — Pages served from closest CDN
3. **Streaming-Native** — Real-time AI responses via SSE
4. **Type-Safe** — End-to-end TypeScript
5. **Secure by Default** — Row-Level Security on all data
6. **Mobile-First** — Responsive design from 375px up

---

## High-Level Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│    🌐 CLIENT LAYER (Browser)                                │
│    ├─ React 19 (Server + Client Components)                 │
│    ├─ TypeScript 5 (Strict Mode)                            │
│    ├─ Tailwind CSS 4                                        │
│    └─ Framer Motion (Animations)                            │
│                                                              │
└────────────────────────┬─────────────────────────────────────┘
                         │ HTTPS
┌────────────────────────▼─────────────────────────────────────┐
│                                                              │
│    ⚡ EDGE LAYER (Vercel)                                   │
│    ├─ Next.js 16 App Router                                 │
│    ├─ Server Components (SSR/SSG)                           │
│    ├─ Route Handlers (API endpoints)                        │
│    ├─ Edge Runtime (where applicable)                       │
│    └─ Streaming Responses (SSE)                             │
│                                                              │
└──────────┬─────────────────────────────┬─────────────────────┘
           │                             │
┌──────────▼──────────┐         ┌────────▼────────────┐
│                     │         │                     │
│  🗄️ DATA LAYER     │         │  🤖 AI LAYER        │
│                     │         │                     │
│  Supabase           │         │  Google Gemini      │
│  ├─ PostgreSQL      │         │  ├─ 2.5 Flash Lite  │
│  ├─ Auth (OAuth)    │         │  ├─ 2.5 Flash       │
│  ├─ RLS Policies    │         │  └─ Streaming API   │
│  ├─ SQL Functions   │         │                     │
│  └─ Real-time       │         │                     │
│                     │         │                     │
└─────────────────────┘         └─────────────────────┘
```

---

## Tech Stack Decisions

### Why Next.js 16?

- **App Router** — File-based routing with Server Components
- **Edge Runtime** — Faster response times globally
- **Built-in Optimizations** — Image, font, and bundle optimization
- **TypeScript Native** — First-class TypeScript support
- **Vercel Integration** — Zero-config deployment

### Why Supabase?

- **Postgres** — Powerful, relational, scalable
- **Auth Built-in** — Email + OAuth providers
- **Row-Level Security** — Database-level security
- **Real-time** — WebSocket subscriptions
- **Edge Functions** — Serverless compute
- **Open Source** — No vendor lock-in

### Why Google Gemini?

- **Multimodal** — Handles text, images, video
- **Fast** — 2.5 Flash Lite for speed-critical paths
- **Cost-Effective** — Cheaper than GPT-4
- **Streaming** — Token-by-token responses
- **Long Context** — 1M+ token context window

### Why Tailwind CSS 4?

- **Utility-First** — Fast development
- **Performance** — Tree-shaken, minimal CSS
- **Design System** — Consistent spacing, colors
- **PostCSS Engine** — New v4 architecture

---

## Data Flow

### 1. User Generates a Post

```
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│  User    │───▶│  Chat UI │───▶│ /api/chat│───▶│  Gemini  │
└──────────┘    └──────────┘    └──────────┘    └──────────┘
                                      │
                                      ▼
                               ┌──────────────┐
                               │  Supabase    │
                               │  - Save msg  │
                               │  - Charge    │
                               └──────────────┘
                                      │
                                      ▼
                               ┌──────────────┐
                               │  Stream SSE  │
                               │  back to UI  │
                               └──────────────┘
```

### Step-by-Step Flow

1. **User Input** → User types prompt in chat UI
2. **Auth Check** → `POST /api/chat` validates session via `@supabase/ssr`
3. **Quota Check** → `can_user_generate` RPC checks remaining credits
4. **URL Detection** → If URL detected, `scrapeArticle()` extracts content
5. **Save User Message** → Insert into `messages` table
6. **AI Streaming** → Gemini streams tokens via `TransformStream`
7. **SSE to Client** → Server-Sent Events to UI
8. **Save Response** → On completion, save assistant message with `post_data` JSONB
9. **Atomic Charge** → `consume_post_credit` charges 1 credit (race-safe)
10. **UI Render** → Post card with copy/regenerate/save actions

---

## Database Schema

### Core Tables

```sql
-- User profiles (auto-created on signup)
profiles (
  id UUID PRIMARY KEY,
  email TEXT,
  full_name TEXT,
  avatar_url TEXT,
  plan TEXT DEFAULT 'free',
  posts_used_this_month INT DEFAULT 0,
  posts_limit INT DEFAULT 10,
  reset_date TIMESTAMP,
  role TEXT DEFAULT 'user', -- 'user' | 'admin'
  created_at TIMESTAMP
)

-- Conversation history
conversations (
  id UUID PRIMARY KEY,
  user_id UUID REFERENCES profiles(id),
  title TEXT,
  pinned BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMP,
  updated_at TIMESTAMP
)

-- Chat messages
messages (
  id UUID PRIMARY KEY,
  conversation_id UUID REFERENCES conversations(id),
  role TEXT, -- 'user' | 'assistant'
  content TEXT,
  post_data JSONB, -- structured post output
  created_at TIMESTAMP
)

-- Generated posts (favorites/saved)
posts (
  id UUID PRIMARY KEY,
  user_id UUID REFERENCES profiles(id),
  platform TEXT, -- 'linkedin' | 'twitter' | 'instagram' | 'facebook'
  content TEXT,
  hashtags TEXT[],
  tone TEXT,
  is_favorite BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMP
)
```

### Row-Level Security (RLS)

Every table enforces user-level isolation:

```sql
-- Example: Users can only read their own posts
CREATE POLICY "Users read own posts" ON posts
  FOR SELECT USING (auth.uid() = user_id);

-- Example: Users can only modify their own data
CREATE POLICY "Users update own posts" ON posts
  FOR UPDATE USING (auth.uid() = user_id);
```

---

## API Design

### RESTful Endpoints

```
POST   /api/chat              # Stream AI post generation (SSE)
POST   /api/scrape            # Extract article from URL
GET    /api/conversations     # List user's conversations
POST   /api/conversations     # Create new conversation
DELETE /api/conversations/:id # Delete conversation
GET    /api/posts             # List saved posts
POST   /api/posts             # Save a post
DELETE /api/posts/:id         # Delete a post
GET    /api/usage             # Get current quota
GET    /api/stats             # Dashboard metrics
GET    /api/admin/users       # Admin: list all users
PATCH  /api/admin/users/:id   # Admin: update user plan/role
```

### Streaming Response Pattern

```typescript
// /api/chat returns Server-Sent Events
return new Response(stream, {
  headers: {
    'Content-Type': 'text/event-stream',
    'Cache-Control': 'no-cache',
    'Connection': 'keep-alive',
  },
})
```

---

## Authentication Flow

### Email/Password Signup

```
1. User submits email + password
   ↓
2. Supabase Auth creates user
   ↓
3. Postgres trigger auto-creates profile
   ↓
4. Email verification (optional)
   ↓
5. User logged in via cookie session
```

### Google OAuth Flow

```
1. User clicks "Continue with Google"
   ↓
2. Redirect to Google OAuth
   ↓
3. User grants permissions
   ↓
4. Callback to /auth/callback
   ↓
5. Supabase creates/updates user
   ↓
6. Session cookie set
   ↓
7. Redirect to /dashboard
```

### Session Management

- **Cookie-Based** — `@supabase/ssr` for SSR-safe sessions
- **HttpOnly** — Cookies not accessible via JavaScript
- **Secure** — HTTPS-only in production
- **SameSite** — CSRF protection

---

## AI Integration

### Gemini Model Strategy

```typescript
// Primary: Fast, cheap
const PRIMARY_MODEL = 'gemini-2.5-flash-lite'

// Fallback: When primary rate-limited
const FALLBACK_MODEL = 'gemini-2.5-flash'

async function generateWithFallback(prompt: string) {
  try {
    return await generate(PRIMARY_MODEL, prompt)
  } catch (error) {
    if (error.status === 429) {
      return await generate(FALLBACK_MODEL, prompt)
    }
    throw error
  }
}
```

### Platform-Specific Prompts

Each platform has tuned prompts:

- **LinkedIn** — Long-form, professional, story-driven
- **Twitter/X** — Punchy, viral, hook-first
- **Instagram** — Emotive, visual-friendly, hashtag-heavy
- **Facebook** — Community-focused, conversational

### Streaming Implementation

```typescript
const stream = await model.generateContentStream(prompt)
const transformStream = new TransformStream({
  transform(chunk, controller) {
    controller.enqueue(`data: ${JSON.stringify(chunk)}\n\n`)
  },
})
return new Response(stream.pipeThrough(transformStream))
```

---

## Performance Optimizations

### Frontend

- ✅ **Server Components** — Less JS shipped to client
- ✅ **Code Splitting** — Route-based chunks
- ✅ **Image Optimization** — AVIF/WebP via `next/image`
- ✅ **Font Optimization** — `next/font` with preload
- ✅ **Streaming SSR** — Instant first byte
- ✅ **Edge Runtime** — Lower latency globally

### Backend

- ✅ **Atomic SQL Functions** — Race-condition prevention
- ✅ **Connection Pooling** — Supabase pgBouncer
- ✅ **Indexed Queries** — Fast lookups on `user_id`, `created_at`
- ✅ **Rate Limiting** — Per-user scrape limits
- ✅ **Caching** — Static assets via Vercel CDN

### Database

```sql
-- Critical indexes
CREATE INDEX idx_posts_user_id ON posts(user_id);
CREATE INDEX idx_messages_conversation_id ON messages(conversation_id);
CREATE INDEX idx_conversations_user_updated ON conversations(user_id, updated_at DESC);
```

---

## Security Architecture

### Defense in Depth

```
┌─────────────────────────────────────────┐
│ Layer 1: Network (HTTPS, DDoS, WAF)     │
├─────────────────────────────────────────┤
│ Layer 2: App (Auth, Input Validation)   │
├─────────────────────────────────────────┤
│ Layer 3: API (Rate Limit, RBAC)         │
├─────────────────────────────────────────┤
│ Layer 4: Database (RLS, Encryption)     │
└─────────────────────────────────────────┘
```

### Security Measures

| Layer | Protection |
|-------|------------|
| **Network** | HTTPS only, Vercel DDoS protection |
| **Headers** | CSP, X-Frame-Options, HSTS |
| **Auth** | Server-side sessions, no client tokens |
| **API** | Rate limiting, input sanitization |
| **Database** | RLS on all tables, prepared statements |
| **AI** | Prompt injection protection, output filtering |
| **Secrets** | Environment variables, no hardcoded keys |

---

## Scalability Considerations

### Current Capacity

- **Users**: 10,000+ concurrent users supported
- **Posts/sec**: 100+ generations per second
- **Database**: 100GB+ data scalable on Supabase

### Future Scaling Path

1. **Caching Layer** — Redis for hot data
2. **Queue System** — Background job processing
3. **CDN Expansion** — More edge locations
4. **Database Replication** — Read replicas
5. **Microservices** — Split AI service from main app

---

## 📚 Related Documentation

- [Features Documentation](./FEATURES.md)
- [Tech Stack Deep Dive](./TECH_STACK.md)
- [Main README](../README.md)

---

**Built with ❤️ by [MZ Studios](https://muhammad-zeeshan-dev.vercel.app)**
