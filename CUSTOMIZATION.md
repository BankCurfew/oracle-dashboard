# Customization Guide — Oracle MCP Memory Layer

> How to customize Oracle for your project: theming, new pages, API endpoints, branding, and knowledge structure.
>
> <!-- คู่มือ customize Oracle — เปลี่ยน theme, เพิ่มหน้า, แก้ API, เปลี่ยน branding -->

---

## Quick Reference

| I want to... | See section |
|--------------|-------------|
| Change colors and theme | [Dashboard Theming](#dashboard-theming) |
| Add a new dashboard page | [Adding Dashboard Pages](#adding-dashboard-pages) |
| Add a new API endpoint | [Adding API Endpoints](#adding-api-endpoints) |
| Add a new MCP tool | [Adding MCP Tools](#adding-mcp-tools) |
| Change the branding | [Branding](#branding) |
| Customize the knowledge structure | [Knowledge Structure](#knowledge-structure) |
| Change the database schema | [Database Changes](#database-changes) |
| Add a new vector store adapter | [Vector Store Adapters](#vector-store-adapters) |
| Customize authentication | [Authentication](#authentication) |

---

## Dashboard Theming

The React dashboard lives in `frontend/`. It uses CSS variables for theming.

### Changing Colors

Edit `frontend/src/index.css` (or your global stylesheet):

```css
:root {
  /* Primary brand colors */
  --color-primary: #6366f1;        /* Indigo — change to your brand color */
  --color-primary-hover: #4f46e5;
  --color-background: #0f172a;     /* Dark background */
  --color-surface: #1e293b;        /* Card/panel background */
  --color-text: #e2e8f0;           /* Main text */
  --color-text-muted: #94a3b8;     /* Secondary text */
  --color-border: #334155;         /* Borders */
  --color-accent: #22d3ee;         /* Accent/highlight */
}
```

<!-- เปลี่ยนสีใน CSS variables — แก้ไฟล์เดียว ทั้ง dashboard เปลี่ยนตาม -->

### Light Mode

Override the variables in a light mode media query:

```css
@media (prefers-color-scheme: light) {
  :root {
    --color-background: #ffffff;
    --color-surface: #f8fafc;
    --color-text: #1e293b;
    --color-text-muted: #64748b;
    --color-border: #e2e8f0;
  }
}
```

---

## Adding Dashboard Pages

### Step 1: Create the page component

```
frontend/src/pages/MyPage.tsx
```

```tsx
export default function MyPage() {
  const [data, setData] = useState(null);

  useEffect(() => {
    fetch('/api/my-endpoint')
      .then(res => res.json())
      .then(setData);
  }, []);

  return (
    <div>
      <h1>My Custom Page</h1>
      {/* Your content here */}
    </div>
  );
}
```

### Step 2: Add the route

Edit `frontend/src/App.tsx`:

```tsx
import MyPage from './pages/MyPage';

// Inside the Router/Routes:
<Route path="/my-page" element={<MyPage />} />
```

### Step 3: Add to sidebar navigation

Edit `frontend/src/components/SidebarLayout.tsx`:

```tsx
// Add to the navigation items array:
{ path: '/my-page', label: 'My Page', icon: '📊' }
```

### Step 4: Build and test

```bash
cd frontend
bun run dev    # Development with hot reload
bun run build  # Production build
```

<!-- เพิ่มหน้าใหม่: สร้าง component → เพิ่ม route → เพิ่มใน sidebar → build -->

---

## Adding API Endpoints

### Step 1: Add the route handler

Edit `src/server.ts` to add your endpoint:

```typescript
// Add after existing routes
app.get('/api/my-data', async (c) => {
  const db = getDb();

  // Your query logic
  const results = db.select().from(oracleDocuments)
    .where(eq(oracleDocuments.type, 'learning'))
    .limit(10)
    .all();

  return c.json({ results });
});
```

### Step 2: With authentication (if auth is enabled)

```typescript
// Use the auth middleware pattern from existing routes
app.get('/api/my-protected-data', async (c) => {
  // Auth is handled by the global middleware in server.ts
  // Just write your logic
  return c.json({ data: 'protected' });
});
```

### Step 3: For complex handlers, create a separate file

```
src/server/my-handler.ts
```

```typescript
import type { Context } from 'hono';
import { getDb } from '../db';

export async function handleMyData(c: Context) {
  const query = c.req.query('q') || '';
  const limit = parseInt(c.req.query('limit') || '10');

  const db = getDb();
  // Your logic here

  return c.json({ results: [] });
}
```

Then in `src/server.ts`:
```typescript
import { handleMyData } from './server/my-handler';
app.get('/api/my-data', handleMyData);
```

<!-- เพิ่ม API endpoint: เขียน handler → เพิ่ม route ใน server.ts -->

---

## Adding MCP Tools

### Step 1: Define the tool handler

Create `src/tools/my-tool.ts`:

```typescript
import type { ToolContext } from './types';

export interface MyToolInput {
  query: string;
  limit?: number;
}

export async function handleMyTool(
  input: MyToolInput,
  context: ToolContext
) {
  const { db, sqlite, repoRoot } = context;

  // Your tool logic here
  const results = db.select()
    .from(oracleDocuments)
    .limit(input.limit || 10)
    .all();

  return {
    content: [{
      type: 'text' as const,
      text: JSON.stringify(results, null, 2),
    }],
  };
}
```

### Step 2: Register the tool

In `src/index.ts`, add to the `setupTools()` method:

```typescript
this.server.tool(
  'oracle_my_tool',
  'Description of what this tool does',
  {
    query: z.string().describe('Search query'),
    limit: z.number().optional().describe('Max results'),
  },
  async ({ query, limit }) => {
    return handleMyTool({ query, limit }, this.toolContext);
  }
);
```

### Step 3: Export from barrel

Add to `src/tools/index.ts`:
```typescript
export { handleMyTool } from './my-tool';
```

### Step 4: Add matching HTTP endpoint (optional)

In `src/server.ts`:
```typescript
app.get('/api/my-tool', async (c) => {
  const result = await handleMyTool(
    { query: c.req.query('q') || '', limit: 10 },
    toolContext
  );
  return c.json(JSON.parse(result.content[0].text));
});
```

<!-- เพิ่ม MCP tool: เขียน handler → register ใน index.ts → (optional) เพิ่ม HTTP route -->

---

## Branding

### Project Name

1. **package.json** — change `name` and `description`
2. **README.md** — update title, description, repo URLs
3. **src/index.ts** — update server name in MCP initialization:
   ```typescript
   this.server = new Server({ name: 'your-project-name', version: '1.0.0' }, ...);
   ```

### Dashboard Title

Edit `frontend/index.html`:
```html
<title>Your Dashboard Name</title>
```

Edit `frontend/src/components/Header.tsx`:
```tsx
<h1>Your Dashboard Name</h1>
```

### Dashboard Logo

Replace or add a logo in `frontend/src/components/Header.tsx`:
```tsx
<img src="/logo.svg" alt="Logo" className="header-logo" />
```

Place `logo.svg` in `frontend/public/`.

### MCP Server Identity

In `src/index.ts`:
```typescript
this.server = new Server(
  {
    name: 'your-mcp-server',
    version: '1.0.0',
  },
  { capabilities: { tools: {} } }
);
```

<!-- เปลี่ยน branding: แก้ชื่อใน package.json, README, dashboard header, MCP server name -->

---

## Knowledge Structure

### Default Structure

Oracle expects markdown in `ψ/memory/`:

```
ψ/memory/resonance/     → principles (core beliefs, guidelines)
ψ/memory/learnings/     → lessons learned (dated, specific)
ψ/memory/retrospectives/ → session reviews (dated, grouped by month)
```

### Customizing Document Types

The indexer (`src/indexer.ts`) maps directories to types:

```typescript
// In the indexer source:
// resonance/ → 'principle'
// learnings/ → 'learning'
// retrospectives/ → 'retro'
```

To add a new type:

1. **Add the directory mapping** in `src/indexer.ts`:
   ```typescript
   // Add alongside existing mappings
   if (relativePath.includes('playbooks/')) {
     type = 'playbook';
   }
   ```

2. **Create the directory**:
   ```bash
   mkdir -p ψ/memory/playbooks
   ```

3. **Re-index**:
   ```bash
   bun run index
   ```

4. **Filter by type** in search:
   ```bash
   curl "http://localhost:47778/api/search?q=deploy&type=playbook"
   ```

### Customizing Split Strategy

The indexer splits markdown by headers. To change this:

Edit `src/indexer.ts` — look for the `splitDocument()` function:

```typescript
// Default: split by ## headers for learnings
// You can add custom split logic per type
```

### Adding Non-Markdown Sources

To index from other sources (JSON, YAML, databases):

1. Create a custom indexer in `src/scripts/`
2. Insert directly into `oracle_documents` and `oracle_fts` tables
3. Run your indexer alongside the default one

<!-- ปรับโครงสร้าง knowledge: เพิ่ม directory → แก้ mapping ใน indexer → re-index -->

---

## Database Changes

### Adding a New Table

1. **Define the schema** in `src/db/schema.ts`:

```typescript
import { sqliteTable, text, integer } from 'drizzle-orm/sqlite-core';

export const myTable = sqliteTable('my_table', {
  id: integer('id').primaryKey({ autoIncrement: true }),
  name: text('name').notNull(),
  data: text('data'),
  createdAt: text('created_at').default(sql`(datetime('now'))`),
});
```

2. **Generate migration**:
```bash
bun run db:generate
```

3. **Apply migration**:
```bash
bun run db:migrate
```

4. **Use in code**:
```typescript
import { myTable } from '../db/schema';
import { getDb } from '../db';

const db = getDb();
const rows = db.select().from(myTable).all();
```

### Modifying Existing Tables

Follow the same generate → migrate flow. Drizzle ORM handles the diff.

> **Important:** The FTS5 virtual table (`oracle_fts`) is managed via raw SQL, not Drizzle. Changes to FTS5 must be done manually in the migration files.

<!-- เพิ่ม/แก้ database table: แก้ schema.ts → generate migration → migrate -->

---

## Vector Store Adapters

### Adding a New Vector Store

1. **Create the adapter** at `src/vector/adapters/my-store.ts`:

```typescript
import type { VectorStoreAdapter, VectorSearchResult, VectorStoreStatus } from '../types';

export class MyStoreAdapter implements VectorStoreAdapter {
  async addDocument(id: string, content: string, metadata?: Record<string, any>): Promise<void> {
    // Your implementation
  }

  async search(query: string, limit?: number): Promise<VectorSearchResult[]> {
    // Your implementation — return [{id, score, content?}]
  }

  async deleteDocument(id: string): Promise<void> {
    // Your implementation
  }

  async getStatus(): Promise<VectorStoreStatus> {
    return { available: true, type: 'my-store', documentCount: 0 };
  }
}
```

2. **Register in the factory** at `src/vector/factory.ts`:

```typescript
case 'my-store':
  return new MyStoreAdapter(config);
```

3. **Use it**:
```bash
export ORACLE_VECTOR_DB=my-store
bun run server
```

<!-- เพิ่ม vector store ใหม่: implement VectorStoreAdapter interface → register ใน factory -->

---

## Authentication

### Enable Password Protection

Via the Settings page in the dashboard, or via API:

```bash
curl -X POST http://localhost:47778/api/settings \
  -H 'Content-Type: application/json' \
  -d '{"password": "your-password", "authEnabled": true}'
```

### Disable Local Network Bypass

By default, local IPs (192.168.x, 10.x, 172.16-31.x) bypass auth. To disable:

```bash
curl -X POST http://localhost:47778/api/settings \
  -H 'Content-Type: application/json' \
  -d '{"localBypass": false}'
```

### Custom Auth Integration

To integrate with an external auth provider, modify the auth middleware in `src/server.ts`:

```typescript
// Look for the auth middleware section
// Replace cookie validation with your provider's verification
```

---

## Feed Log Customization

### Custom Events

Any process can append to `~/.oracle/feed.log`:

```bash
echo "$(date -u '+%Y-%m-%d %H:%M:%S') | my-tool | $(hostname) | custom_event | my-project | session-123 » Something happened" >> ~/.oracle/feed.log
```

Format: `timestamp | source | host | event_type | project | session_id » message`

### Custom Feed Filters

The `/api/feed` endpoint supports filtering:
- `?oracle=my-tool` — filter by source name
- `?event=custom_event` — filter by event type
- `?since=2026-03-19T00:00:00Z` — filter by timestamp

---

## Deployment Customization

### Custom Port

```bash
ORACLE_PORT=8080 bun run server
```

### Custom Data Directory

```bash
ORACLE_DATA_DIR=/var/lib/oracle bun run server
```

### Multiple Instances

Run separate instances for different knowledge bases:

```bash
# Instance 1: Project A
ORACLE_PORT=47778 ORACLE_REPO_ROOT=/path/to/project-a ORACLE_DB_PATH=/tmp/oracle-a.db bun run server

# Instance 2: Project B
ORACLE_PORT=47779 ORACLE_REPO_ROOT=/path/to/project-b ORACLE_DB_PATH=/tmp/oracle-b.db bun run server
```

### Reverse Proxy (Nginx)

```nginx
server {
    listen 443 ssl;
    server_name oracle.example.com;

    location / {
        proxy_pass http://localhost:47778;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    location /api/ {
        proxy_pass http://localhost:47778/api/;
    }
}
```

---

## Common Customization Patterns

### Adding a Custom Dashboard Widget

1. Create a component in `frontend/src/components/`:
```tsx
// MyWidget.tsx
export function MyWidget({ data }) {
  return (
    <div className="widget">
      <h3>My Widget</h3>
      <p>{data.value}</p>
    </div>
  );
}
```

2. Use it in `frontend/src/pages/Overview.tsx`:
```tsx
import { MyWidget } from '../components/MyWidget';
// Add to the dashboard grid
```

### Custom Search Scoring

Modify `src/tools/search.ts` to adjust hybrid weights:

```typescript
// Default: 50/50 FTS/vector with 10% overlap boost
const FTS_WEIGHT = 0.5;
const VECTOR_WEIGHT = 0.5;
const OVERLAP_BOOST = 1.1;

// Example: favor semantic search
const FTS_WEIGHT = 0.3;
const VECTOR_WEIGHT = 0.7;
const OVERLAP_BOOST = 1.15;
```

### Custom MCP Tool Descriptions

The `____IMPORTANT` meta-tool in `src/index.ts` provides workflow guidance to Claude. Customize it to match your use case:

```typescript
this.server.tool(
  '____IMPORTANT',
  'Your custom workflow guide here...',
  {},
  async () => ({ content: [{ type: 'text', text: 'Your instructions...' }] })
);
```

---

## Tips

- **Start small** — customize one thing at a time, test, then move on
- **Keep the `ToolContext` pattern** — it makes testing and reuse easy
- **Use the existing API client** (`frontend/src/api/oracle.ts`) as reference for new endpoints
- **Re-index after knowledge structure changes** — `bun run index`
- **Check existing tests** — `src/tools/__tests__/` shows how to test tool handlers

---

*"Fork it. Make it yours. The Oracle adapts."*
