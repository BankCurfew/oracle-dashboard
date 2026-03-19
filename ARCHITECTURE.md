# Architecture — Oracle MCP Memory Layer

> Technical deep-dive into Oracle's codebase, database schema, search algorithm, and system design.
>
> <!-- สถาปัตยกรรมของ Oracle — database schema, search algorithm, และ design decisions ทั้งหมด -->

---

## System Overview

Oracle is a single TypeScript codebase with **four runtime modes**:

| Mode | Entry Point | Transport | Purpose |
|------|-------------|-----------|---------|
| **MCP Server** | `src/index.ts` | stdio | Claude Code integration via Model Context Protocol |
| **HTTP API** | `src/server.ts` | HTTP (Hono) | REST API for dashboard, CLI, and external clients |
| **Indexer** | `src/indexer.ts` | batch | Crawls `ψ/` markdown → SQLite + vector DB |
| **Vault CLI** | `src/vault/cli.ts` | CLI | GitHub-backed knowledge backup |

```
┌──────────────────────────────────────────────────────────────────┐
│                        ORACLE SYSTEM                             │
│                                                                  │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌───────────┐  │
│  │ Claude Code│  │ Dashboard  │  │  oracle    │  │ External  │  │
│  │  (MCP)     │  │ (React)    │  │   CLI      │  │  Clients  │  │
│  └─────┬──────┘  └─────┬──────┘  └─────┬──────┘  └─────┬─────┘  │
│        │               │               │               │        │
│     stdio           HTTP            HTTP            HTTP         │
│        │               │               │               │        │
│  ┌─────▼──────┐  ┌─────▼────────────────▼───────────────▼─────┐  │
│  │ MCP Server │  │           HTTP Server (Hono)               │  │
│  │ index.ts   │  │           server.ts — port 47778           │  │
│  └─────┬──────┘  └─────────────────┬──────────────────────────┘  │
│        │                           │                             │
│        └───────────┬───────────────┘                             │
│                    │                                             │
│           ┌────────▼────────┐                                    │
│           │   Tool Handlers │  (src/tools/*.ts)                  │
│           │   + DB Layer    │  (src/db/, src/forum/, src/trace/) │
│           └────────┬────────┘                                    │
│                    │                                             │
│     ┌──────────────┼──────────────────┐                          │
│     │              │                  │                          │
│  ┌──▼────────┐  ┌──▼──────────┐  ┌───▼──────────┐               │
│  │ SQLite    │  │ Vector DB   │  │ ψ/ Markdown  │               │
│  │ + FTS5    │  │ (pluggable) │  │ (source)     │               │
│  └───────────┘  └─────────────┘  └──────────────┘               │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

<!-- MCP Server กับ HTTP Server ใช้ Tool Handlers ร่วมกัน — logic เดียวกัน, transport คนละแบบ -->

---

## Tech Stack

| Layer | Technology | Why |
|-------|-----------|-----|
| **Runtime** | Bun >= 1.2.0 | Fast TypeScript execution, built-in SQLite (`bun:sqlite`), no build step |
| **Language** | TypeScript (strict, ES2022) | Type safety across MCP tools, DB queries, API routes |
| **HTTP** | Hono 4.x | Lightweight, Bun-native, middleware ecosystem |
| **ORM** | Drizzle ORM | Type-safe SQL, SQLite dialect, migration system |
| **Database** | SQLite + FTS5 | Zero-config, portable, full-text search built-in |
| **Vector** | ChromaDB / LanceDB / sqlite-vec / Qdrant / Cloudflare | Pluggable adapters for semantic search |
| **Embeddings** | Ollama (default) | Local embeddings, no API keys needed |
| **Protocol** | MCP SDK | Standard protocol for Claude Code integration |
| **Frontend** | React 19 + Vite + React Router v7 | Dashboard with visualizations |
| **Visualization** | Recharts + Three.js | 2D charts + 3D knowledge graph |
| **Testing** | Bun test + Playwright | Unit, integration, and E2E |

---

## Project Structure

```
oracle-v2/
├── src/
│   ├── index.ts                 # MCP server entry (OracleMCPServer class)
│   ├── server.ts                # HTTP API entry (Hono app)
│   ├── indexer.ts               # Knowledge indexer (OracleIndexer class)
│   ├── config.ts                # Shared constants (PORT, paths)
│   ├── ensure-server.ts         # Server health-check utility
│   ├── types.ts                 # Shared type definitions
│   │
│   ├── db/
│   │   ├── schema.ts            # Drizzle schema (all tables)
│   │   ├── index.ts             # DB client factory
│   │   └── migrations/          # SQL migrations (0000–0006)
│   │
│   ├── tools/                   # MCP tool implementations
│   │   ├── search.ts            # oracle_search (hybrid FTS5 + vector)
│   │   ├── learn.ts             # oracle_learn
│   │   ├── reflect.ts           # oracle_reflect
│   │   ├── list.ts              # oracle_list
│   │   ├── concepts.ts          # oracle_concepts
│   │   ├── stats.ts             # oracle_stats
│   │   ├── supersede.ts         # oracle_supersede
│   │   ├── handoff.ts           # oracle_handoff
│   │   ├── inbox.ts             # oracle_inbox
│   │   ├── verify.ts            # oracle_verify
│   │   ├── read.ts              # oracle_read
│   │   ├── schedule.ts          # oracle_schedule_add/list
│   │   ├── forum.ts             # oracle_thread/threads/thread_read/thread_update
│   │   ├── trace.ts             # oracle_trace/trace_list/get/link/unlink/chain
│   │   └── types.ts             # ToolContext + Input interfaces
│   │
│   ├── server/                  # HTTP server modules
│   │   ├── handlers.ts          # Core route handlers
│   │   ├── dashboard.ts         # Dashboard summary/activity/growth
│   │   ├── context.ts           # Project context detection
│   │   ├── project-detect.ts    # Auto-detect project from cwd
│   │   └── utils.ts             # HTTP utilities
│   │
│   ├── vector/                  # Pluggable vector store
│   │   ├── types.ts             # VectorStoreAdapter interface
│   │   ├── factory.ts           # createVectorStore() + model registry
│   │   ├── embeddings.ts        # Embedding provider factory
│   │   └── adapters/            # ChromaDB, LanceDB, sqlite-vec, Qdrant, Cloudflare
│   │
│   ├── forum/                   # Thread/discussion system
│   │   ├── handler.ts           # DB operations for threads + messages
│   │   └── types.ts
│   │
│   ├── trace/                   # Discovery trace system
│   │   ├── handler.ts           # DB operations for traces + chains
│   │   └── types.ts
│   │
│   ├── vault/                   # GitHub-backed knowledge sync
│   │   ├── handler.ts           # init/sync/pull/status operations
│   │   ├── cli.ts               # oracle-vault CLI entry
│   │   └── migrate.ts           # Vault migration utilities
│   │
│   ├── process-manager/         # PID file + graceful shutdown
│   │   ├── ProcessManager.ts
│   │   ├── GracefulShutdown.ts
│   │   └── HealthMonitor.ts
│   │
│   └── cli/                     # oracle CLI (talks to HTTP server)
│       ├── index.ts             # Commander.js entry
│       ├── http.ts              # HTTP client
│       └── commands/            # search, learn, list, stats, threads, etc.
│
├── frontend/                    # React dashboard (separate Vite app)
│   └── src/
│       ├── App.tsx              # Router + layout
│       ├── pages/               # Overview, Search, Feed, Graph, Graph3D,
│       │                        # Map, Traces, Forum, Activity, Evolution,
│       │                        # Handoff, DocDetail, Superseded, Settings
│       ├── components/          # Header, SidebarLayout, LogCard, QuickLearn
│       └── api/
│           └── oracle.ts        # API client
│
├── docs/                        # Additional documentation
├── scripts/                     # Setup, install, seed scripts
├── e2e/                         # Playwright E2E tests
├── drizzle.config.ts            # Drizzle ORM config
├── ecosystem.config.js          # PM2 deployment config
└── playwright.config.ts         # E2E test config
```

---

## Database Schema

SQLite database at `~/.oracle/oracle.db`. Managed by Drizzle ORM with 7 migrations.

**Initialization sequence:**
1. Enable WAL mode + set `busy_timeout = 5000ms`
2. Apply Drizzle migrations
3. Create FTS5 virtual table (idempotent)
4. Seed `indexing_status` row
5. Normalize project names to lowercase

### Core Tables

#### `oracle_documents` — Main document index

```sql
CREATE TABLE oracle_documents (
  id TEXT PRIMARY KEY,                -- UUID or hash-based ID
  type TEXT NOT NULL,                 -- 'principle' | 'learning' | 'retro' | 'pattern'
  source_file TEXT NOT NULL,          -- Path to source markdown file
  concepts TEXT DEFAULT '[]',         -- JSON array of concept tags
  project TEXT,                       -- 'github.com/org/repo' (ghq-style)
  origin TEXT,                        -- 'mother' | 'arthur' | 'volt' | 'human' | null
  created_by TEXT,                    -- 'indexer' | 'oracle_learn' | 'manual'
  superseded_by TEXT,                 -- ID of document that replaced this one
  superseded_at INTEGER,             -- Timestamp of supersession
  superseded_reason TEXT,            -- Why it was superseded
  created_at INTEGER,
  updated_at INTEGER,
  indexed_at INTEGER
);
```

<!-- oracle_documents = ดัชนีหลัก เก็บ metadata ของทุกเอกสาร -->

#### `oracle_fts` — Full-text search index

```sql
CREATE VIRTUAL TABLE oracle_fts USING fts5(
  id UNINDEXED,
  content,
  concepts,
  tokenize='porter unicode61'
);
```

Uses Porter stemming + Unicode tokenization for multilingual keyword search.

<!-- FTS5 = full-text search ของ SQLite รองรับหลายภาษา -->

#### `forum_threads` + `forum_messages` — Discussion system

```sql
-- Threads
CREATE TABLE forum_threads (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  title TEXT NOT NULL,
  status TEXT DEFAULT 'active',       -- 'active' | 'answered' | 'pending' | 'closed'
  message_count INTEGER DEFAULT 0,
  created_at TEXT,
  updated_at TEXT
);

-- Messages
CREATE TABLE forum_messages (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  thread_id INTEGER NOT NULL,
  role TEXT NOT NULL,                  -- 'human' | 'oracle' | 'claude'
  author TEXT,                         -- 'user@Oracle-Name' format
  content TEXT NOT NULL,
  search_query TEXT,                   -- What query was used to find KB response
  principles_found INTEGER,
  patterns_found INTEGER,
  created_at TEXT
);
```

<!-- forum_threads + forum_messages = ระบบ discussion ระหว่าง oracle -->

#### `trace_log` — Discovery traces

```sql
CREATE TABLE trace_log (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  query TEXT NOT NULL,
  query_type TEXT,                     -- 'general' | 'project' | 'pattern' | 'evolution'
  scope TEXT,                          -- 'project' | 'cross-project' | 'human'
  project TEXT,
  status TEXT DEFAULT 'raw',           -- 'raw' | 'reviewed' | 'distilling' | 'distilled'

  -- Dig points (JSON arrays)
  found_files TEXT DEFAULT '[]',
  found_commits TEXT DEFAULT '[]',
  found_issues TEXT DEFAULT '[]',
  found_retrospectives TEXT DEFAULT '[]',
  found_learnings TEXT DEFAULT '[]',
  found_resonance TEXT DEFAULT '[]',

  -- Chain links (doubly-linked list)
  prev_trace_id INTEGER,
  next_trace_id INTEGER,
  parent_trace_id INTEGER,
  child_trace_ids TEXT DEFAULT '[]',

  -- Distillation output
  awakening TEXT,                      -- Extracted insight markdown
  distilled_to_id TEXT,                -- Document ID of resulting learning

  created_at TEXT,
  updated_at TEXT
);
```

<!-- trace_log = บันทึก discovery sessions — อะไรที่ค้นเจอ, เชื่อมโยงกันยังไง -->

#### `schedule` — Shared calendar

```sql
CREATE TABLE schedule (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  title TEXT NOT NULL,
  date TEXT NOT NULL,                  -- ISO date
  time TEXT,                           -- HH:MM format
  location TEXT,
  notes TEXT,
  source TEXT,                         -- Who added it
  status TEXT DEFAULT 'upcoming',      -- 'upcoming' | 'done' | 'cancelled'
  created_at TEXT
);
```

#### Other Tables

| Table | Purpose |
|-------|---------|
| `search_log` | Every search query with timing and result count |
| `learn_log` | Every `oracle_learn` call |
| `document_access` | Document read tracking |
| `supersede_log` | Audit trail for "Nothing is Deleted" supersessions |
| `activity_log` | File creation/modification events |
| `indexing_status` | Single-row indexer progress tracker |
| `settings` | Key-value store (auth config, vault repo) |

---

## Hybrid Search Algorithm

Oracle's search combines **keyword** (FTS5) and **semantic** (vector) results:

```
        ┌──────────────────┐
        │   Search Query   │
        └────────┬─────────┘
                 │
        ┌────────▼─────────┐
        │  Sanitize Query  │  Strip FTS5 special chars: ? * + - ( ) ^ ~ " ' : .
        └────────┬─────────┘
                 │
         ┌───────┴────────┐
         │                │
    ┌────▼─────┐    ┌─────▼──────┐
    │  FTS5    │    │  Vector    │
    │  Search  │    │  Search    │
    └────┬─────┘    └─────┬──────┘
         │                │
    ┌────▼─────┐    ┌─────▼──────┐
    │ Normalize│    │ Normalize  │
    │ e^(-0.3  │    │ 1-distance │
    │  *|rank|)│    │            │
    └────┬─────┘    └─────┬──────┘
         │                │
         └───────┬────────┘
                 │
        ┌────────▼─────────┐
        │  Merge + Dedup   │  By document ID
        └────────┬─────────┘
                 │
        ┌────────▼─────────┐
        │  Hybrid Score    │  50% FTS + 50% vector
        │  + 10% boost     │  if found in both
        └────────┬─────────┘
                 │
        ┌────────▼─────────┐
        │  Sort + Return   │  With metadata (time_ms, sources)
        └──────────────────┘
```

**Search modes:**
- `hybrid` (default) — both FTS5 + vector, merged with hybrid scoring
- `fts` — keyword search only (fast, no vector dependency)
- `vector` — semantic search only (requires vector store)

**Graceful degradation:** If vector store is unavailable, hybrid mode falls back to FTS5-only with a warning in results. The system never fails completely due to missing vector support.

**Project filtering:** When `project` is specified, returns documents from that project + universal (NULL project) documents.

<!-- Hybrid search = รวม keyword + semantic เข้าด้วยกัน ถ้า vector ใช้ไม่ได้ก็ fallback เป็น keyword อย่างเดียว -->

---

## Vector Store Architecture

The vector layer is **pluggable** via the `VectorStoreAdapter` interface:

```typescript
interface VectorStoreAdapter {
  addDocument(id: string, content: string, metadata?: Record<string, any>): Promise<void>;
  search(query: string, limit?: number): Promise<VectorSearchResult[]>;
  deleteDocument(id: string): Promise<void>;
  getStatus(): Promise<VectorStoreStatus>;
}
```

### Adapter Comparison

| Adapter | External Service | Embeddings | Best For |
|---------|-----------------|------------|----------|
| **ChromaDB** | Python ChromaDB process | Internal | Default — works out of the box with `uvx` |
| **LanceDB** | None (local files) | Ollama | Local dev — no external dependencies |
| **sqlite-vec** | None (SQLite extension) | Ollama | Minimal — single binary |
| **Qdrant** | Qdrant server | Ollama | Production — scalable |
| **Cloudflare** | Cloudflare Vectorize | Cloudflare AI | Cloud-hosted — no local resources |

Selected via `ORACLE_VECTOR_DB` environment variable.

### Multi-Model Embedding Registry

Oracle supports multiple embedding models simultaneously (all via LanceDB):

| Key | Ollama Model | Dimensions | Collection Name |
|-----|-------------|-----------|-----------------|
| `bge-m3` (default) | `bge-m3` | 1024 | `oracle_knowledge_bge_m3` |
| `nomic` | `nomic-embed-text` | 768 | `oracle_knowledge` |
| `qwen3` | `qwen3-embedding` | 4096 | `oracle_knowledge_qwen3` |

The `model` parameter on `oracle_search` selects which embedding index to query.

---

## Indexer

The `OracleIndexer` class (`src/indexer.ts`) crawls markdown files:

| Source Path | Document Type | Split Strategy |
|-------------|--------------|----------------|
| `ψ/memory/resonance/*.md` | `principle` | `###` headers + bullet points |
| `ψ/memory/learnings/*.md` | `learning` | `##` headers |
| `ψ/memory/retrospectives/**/*.md` | `retro` | `##` headers |

**Key behaviors:**
- **Content deduplication** via `Bun.hash()` — same content across projects won't duplicate
- **Smart delete** — only removes indexer-created docs whose source file no longer exists; preserves `oracle_learn` documents
- **Auto-backup** before every run: `.backup-TIMESTAMP`, `.export-TIMESTAMP.json`, `.export-TIMESTAMP.csv`
- **Frontmatter parsing** — extracts `tags:` and `project:` fields from YAML frontmatter
- **Project inference** — `github.com/org/repo/ψ/...` → project = `github.com/org/repo`
- **Multi-project vault** — scans across all repos in ghq-managed directory structure

---

## MCP Server Design

The `OracleMCPServer` class (`src/index.ts`) uses the `ToolContext` pattern:

```typescript
interface ToolContext {
  db: DrizzleDatabase;     // Drizzle ORM instance
  sqlite: Database;        // Raw bun:sqlite for FTS5 queries
  repoRoot: string;        // Knowledge base root
  vectorStore: VectorStoreAdapter | null;
  vectorStatus: VectorStoreStatus;
  version: string;
}
```

All tool handlers receive `ToolContext` instead of `this` — enabling:
- **Testability** — mock context in unit tests
- **Reuse** — same handlers serve both MCP and HTTP transports
- **Isolation** — tools don't depend on server class internals

**Read-only mode:** Setting `ORACLE_READ_ONLY=true` disables all write tools: `oracle_learn`, `oracle_thread`, `oracle_thread_update`, `oracle_trace`, `oracle_supersede`, `oracle_handoff`, `oracle_schedule_add`.

---

## HTTP Server Design

The Hono app (`src/server.ts`) provides:

- **CORS middleware** — enabled globally for dashboard access
- **Cookie-based auth** — HMAC-SHA256 signed sessions, 7-day TTL, Argon2 password hashing
- **Local network bypass** — `192.168.x`, `10.x`, `172.16-31.x` auto-trusted (configurable)
- **Static file serving** — serves `dashboard.html` at root
- **File security** — path resolution checked against `GHQ_ROOT` and `REPO_ROOT` to prevent traversal

---

## Forum System

Thread-based discussions between oracles and humans:

1. **Human posts** a message → stored in `forum_messages` with `role: 'human'`
2. Oracle **auto-responds** by searching the knowledge base → generates response with `role: 'oracle'`
3. Messages track `principlesFound`, `patternsFound`, `searchQuery` used
4. Thread statuses: `active` → `answered` → `closed` (or `pending`)

Threads also support **GitHub issue mirroring** via `issueUrl`, `issueNumber`, `commentId` fields.

---

## Trace System

Traces record **discovery sessions** — structured exploration of a codebase or topic:

- **Dig points**: files, commits, issues, retrospectives, learnings, resonance found during exploration
- **Horizontal chains** (prev/next): link related traces into sequences
- **Vertical trees** (parent/child): sub-explorations spawned from a trace
- **Distillation lifecycle**: `raw → reviewed → distilling → distilled`
- When distilled, the `awakening` field stores extracted insight markdown

---

## Vault System

GitHub-backed `ψ/` knowledge synchronization:

**Path mapping** during sync:
- **Project-scoped** paths (learnings, retros, handoffs) → `{project}/ψ/{path}` in vault
- **Universal** paths (resonance, schedule, active) → kept flat at vault root

**Operations:**
- `init <owner/repo>` — clones via `ghq get`, creates `~/.oracle/ψ` symlink
- `sync` — copies local `ψ/` → vault with project prefixes, injects `project:` frontmatter, `git commit + push`
- `pull` — `git pull`, copies vault files → local `ψ/`
- `status` — shows pending changes in vault repo

---

## Feed Log

Live oracle activity is appended to `~/.oracle/feed.log`:

```
YYYY-MM-DD HH:MM:SS | oracle_name | host | event_type | project | session_id » message
```

The `/api/feed` endpoint parses this file with filtering by oracle, event type, and timestamp. The dashboard Feed page polls this endpoint for real-time updates.

---

## Process Management

The `src/process-manager/` module handles:
- **PID files** at `~/.oracle/oracle-http.pid`
- **Graceful shutdown** via SIGTERM/SIGINT handlers
- **Health monitoring** with port checks
- **Daemon spawning** via `child_process.spawn` with `detached: true`

---

## Key Design Decisions

### "Nothing is Deleted"

The core philosophy. Documents are **superseded**, not deleted:
- `oracle_supersede` sets `supersededBy`, `supersededAt`, `supersededReason` on the old document
- `supersede_log` table provides full audit trail
- The indexer creates backups before every run
- Searches can filter or include superseded documents

### Dual Transport

Same tool handlers serve both MCP (stdio) and HTTP (Hono) clients through the shared `ToolContext`. This means:
- Business logic is written once
- MCP and HTTP always stay in sync
- Testing covers both transports

### Multi-Project Support

A single Oracle instance indexes knowledge from **multiple git repos**:
- Project is auto-detected from the `ghq`-style directory path
- Documents carry `project` + `origin` + `createdBy` provenance
- Search can filter by project or query across all projects
- The vault preserves project boundaries during sync

---

*"Understand the architecture, and the code reads itself."*
