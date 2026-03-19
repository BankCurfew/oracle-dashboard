# Setup Guide — Oracle MCP Memory Layer

> Step-by-step installation from zero to running Oracle.
>
> <!-- คู่มือติดตั้ง Oracle ตั้งแต่เริ่มต้นจนใช้งานได้ -->

---

## Prerequisites

| Requirement | Version | Check | Install |
|------------|---------|-------|---------|
| **Bun** | >= 1.2.0 | `bun --version` | `curl -fsSL https://bun.sh/install \| bash` |
| **Git** | any | `git --version` | Your OS package manager |
| **SQLite** | bundled | — | Included with Bun (`bun:sqlite`) |

### Optional (for vector/semantic search)

| Tool | Purpose | Install |
|------|---------|---------|
| **Ollama** | Local embeddings | [ollama.com](https://ollama.com/) |
| **ChromaDB** | Default vector store | Auto-managed via `uvx` |
| **uv/uvx** | Runs ChromaDB | `curl -LsSf https://astral.sh/uv/install.sh \| bash` |

> Without vector search, Oracle falls back to FTS5 keyword search only — still fully functional.
>
> <!-- ถ้าไม่มี vector search ก็ใช้ได้ — แค่ค้นหาแบบ keyword เท่านั้น -->

---

## Installation

### Method 1: Quick Install Script

```bash
curl -sSL https://raw.githubusercontent.com/Soul-Brews-Studio/oracle-v2/main/scripts/fresh-install.sh | bash
```

This will:
1. Clone to `~/.local/share/oracle-v2`
2. Install dependencies
3. Create seed philosophy files (29 documents)
4. Index seed data
5. Run tests

**After install, skip to [Step 4: Connect to Claude Code](#step-4-connect-to-claude-code).**

---

### Method 2: Manual Install (step by step)

#### Step 1: Clone and install

```bash
git clone https://github.com/Soul-Brews-Studio/oracle-v2.git
cd oracle-v2
bun install
```

<!-- Clone repo แล้ว install dependencies ด้วย bun -->

#### Step 2: Initialize database

```bash
# Option A: Push schema directly (fastest)
bun run db:push

# Option B: Run migrations (recommended for production)
bun run db:migrate
```

This creates `~/.oracle/oracle.db` with all tables + FTS5 index.

<!-- สร้าง database ที่ ~/.oracle/oracle.db -->

#### Step 3: Create your knowledge base

Oracle reads markdown from `ψ/memory/`. Create the directory structure:

```bash
mkdir -p ψ/memory/resonance
mkdir -p ψ/memory/learnings
mkdir -p ψ/memory/retrospectives
mkdir -p ψ/inbox/handoff
```

Add your first principle:

```bash
cat > ψ/memory/resonance/my-principles.md << 'EOF'
# My Principles

### Always verify before acting
Check your assumptions. Test before deploying. Measure before optimizing.

### Write it down
If it's not written, it doesn't exist. Knowledge must be captured to persist.

### Keep it simple
The minimum complexity needed for the current task. No premature abstraction.
EOF
```

Then index:

```bash
bun run index
```

Expected output:
```
Indexing ψ/memory/resonance/my-principles.md...
Indexed 3 documents (3 principles)
```

<!-- สร้างโฟลเดอร์ ψ/memory/ แล้วใส่ไฟล์ markdown ที่เป็น knowledge ของเรา จากนั้น index เข้า database -->

#### Step 4: Connect to Claude Code

**Option A: CLI command**
```bash
claude mcp add oracle-v2 -- bun run /path/to/oracle-v2/src/index.ts
```

**Option B: Edit config file**

Add to `~/.claude.json` (global) or `.mcp.json` (per-project):
```json
{
  "mcpServers": {
    "oracle-v2": {
      "command": "bun",
      "args": ["run", "/path/to/oracle-v2/src/index.ts"],
      "env": {
        "ORACLE_REPO_ROOT": "/path/to/your/project"
      }
    }
  }
}
```

**Option C: Use bunx (no local clone needed)**
```json
{
  "mcpServers": {
    "oracle-v2": {
      "command": "bunx",
      "args": ["--bun", "oracle-v2@github:Soul-Brews-Studio/oracle-v2#main"]
    }
  }
}
```

Restart Claude Code. You should see `oracle_search`, `oracle_learn`, etc. in the available tools.

<!-- เชื่อมต่อกับ Claude Code ผ่าน MCP config -->

#### Step 5: Start the HTTP API

```bash
bun run server
```

Open [http://localhost:47778/api/health](http://localhost:47778/api/health) — you should see `{"status":"ok"}`.

Test search:
```bash
curl "http://localhost:47778/api/search?q=verify+before+acting"
```

<!-- เริ่ม HTTP server แล้วทดสอบ search -->

#### Step 6: Start the Dashboard (optional)

```bash
cd frontend
bun install
bun run dev
```

Open [http://localhost:5173](http://localhost:5173) — the React dashboard.

<!-- เริ่ม dashboard ที่ http://localhost:5173 -->

---

## Vector Search Setup (optional)

Vector search enables **semantic** search — finding documents by meaning, not just keywords.

### Option A: ChromaDB (default)

```bash
# Install uv (provides uvx to run ChromaDB)
curl -LsSf https://astral.sh/uv/install.sh | bash

# Restart the server — it auto-connects to ChromaDB
bun run server
```

### Option B: LanceDB (no external service)

```bash
# Set in .env or export
export ORACLE_VECTOR_DB=lancedb

# Install Ollama for embeddings
# Then pull an embedding model
ollama pull bge-m3

# Re-index with vectors
bun run index
```

### Option C: sqlite-vec (embedded)

```bash
export ORACLE_VECTOR_DB=sqlite-vec
ollama pull bge-m3
bun run index
```

### Option D: Qdrant (remote)

```bash
export ORACLE_VECTOR_DB=qdrant
export QDRANT_URL=http://localhost:6333
ollama pull bge-m3
bun run index
```

---

## Environment Variables

Create a `.env` file in the project root (or export these):

```bash
# Core
ORACLE_PORT=47778                    # HTTP server port
ORACLE_REPO_ROOT=/path/to/project   # Where ψ/ lives (default: cwd)
ORACLE_DATA_DIR=~/.oracle            # Data directory
ORACLE_DB_PATH=~/.oracle/oracle.db   # SQLite database

# Vector search (optional)
ORACLE_VECTOR_DB=chroma              # chroma | lancedb | sqlite-vec | qdrant | cloudflare-vectorize
ORACLE_EMBEDDING_PROVIDER=ollama     # ollama | openai | cloudflare-ai
ORACLE_EMBEDDING_MODEL=bge-m3       # Model name (depends on provider)

# Security (optional)
ORACLE_SESSION_SECRET=your-secret    # Session signing key (random UUID if not set)
ORACLE_READ_ONLY=false               # Disable write MCP tools

# Qdrant (if using qdrant)
QDRANT_URL=http://localhost:6333
QDRANT_API_KEY=

# Cloudflare (if using cloudflare-vectorize)
CLOUDFLARE_ACCOUNT_ID=
CLOUDFLARE_API_TOKEN=
```

<!-- ตั้งค่า environment variables — ส่วนใหญ่มี default ใช้งานได้เลย ไม่ต้องตั้งถ้าไม่ต้องการ -->

---

## Vault Setup (optional)

The vault backs up your `ψ/` knowledge to a private GitHub repo.

```bash
# 1. Create a private repo on GitHub (e.g., my-oracle-vault)

# 2. Initialize vault
oracle-vault init your-username/my-oracle-vault

# 3. Sync knowledge to GitHub
oracle-vault sync

# 4. Pull on another machine
oracle-vault pull
```

The vault uses [ghq](https://github.com/x-motemen/ghq) for path management. Install it if you want cross-machine vault sync.

<!-- Vault = backup ψ/ ไป GitHub repo ส่วนตัว sync ข้ามเครื่องได้ -->

---

## Production Deployment

### Using PM2

```bash
# Install PM2
bun install -g pm2

# Start with ecosystem config
pm2 start ecosystem.config.js

# Or manually
pm2 start src/server.ts --interpreter bun --name oracle-api
```

### Using systemd

```ini
# /etc/systemd/system/oracle-api.service
[Unit]
Description=Oracle API Server
After=network.target

[Service]
Type=simple
User=your-user
WorkingDirectory=/path/to/oracle-v2
ExecStart=/home/your-user/.bun/bin/bun run src/server.ts
Restart=on-failure
Environment=ORACLE_PORT=47778
Environment=ORACLE_DATA_DIR=/home/your-user/.oracle

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl enable oracle-api
sudo systemctl start oracle-api
```

### Build for production

```bash
# Build TypeScript → JavaScript
bun run build

# Build frontend
cd frontend && bun run build

# Serve production
bun run serve
```

---

## Verify Installation

Run these checks after setup:

```bash
# 1. Database exists
ls -la ~/.oracle/oracle.db

# 2. Server responds
curl http://localhost:47778/api/health
# → {"status":"ok"}

# 3. Stats show indexed documents
curl http://localhost:47778/api/stats
# → {"total": N, "by_type": {...}}

# 4. Search works
curl "http://localhost:47778/api/search?q=your+keyword"
# → results array

# 5. MCP tools visible in Claude Code
# Start a Claude Code session and check tools list
```

<!-- ตรวจสอบว่าทุกอย่างทำงานถูกต้อง -->

---

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| `bun: command not found` | Bun not in PATH | `export PATH="$HOME/.bun/bin:$PATH"` |
| Search returns 0 results | DB not indexed | Run `bun run index` then restart server |
| `ENOENT ψ/memory/` | Wrong directory structure | Must be `ψ/memory/` not `memory/` |
| ChromaDB timeout | ChromaDB not installed | Install `uv` or use `ORACLE_VECTOR_DB=lancedb` |
| `Vector search unavailable` | No vector store configured | FTS5 still works; install Ollama + set `ORACLE_VECTOR_DB` for vectors |
| Port 47778 in use | Another instance running | `lsof -i :47778` then `kill <PID>`, or change `ORACLE_PORT` |
| MCP tools not showing | Config not loaded | Restart Claude Code; check `.mcp.json` or `~/.claude.json` |
| `Database is locked` | Multiple writers | Only one server instance should run; check `~/.oracle/oracle-http.pid` |

<!-- ปัญหาที่พบบ่อยและวิธีแก้ -->

---

## Uninstall

```bash
# Remove code
rm -rf /path/to/oracle-v2

# Remove data (WARNING: deletes all knowledge)
rm -rf ~/.oracle

# Remove MCP config from Claude Code
claude mcp remove oracle-v2
```

---

## Next Steps

- [ARCHITECTURE.md](./ARCHITECTURE.md) — understand the codebase
- [CUSTOMIZATION.md](./CUSTOMIZATION.md) — customize for your use case
- [docs/API.md](./docs/API.md) — full API reference

---

*"Start with one principle. Index it. Search for it. That's Oracle."*
