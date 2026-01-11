# Kuato

Easily recall what you discussed with your favorite coding agents, what decisions you made, and where you left off so you can pick up where you left off as easily as asking "where did we leave off on XYZ..."

This simple, fully local session recall skill works on Claude Code & other coding agents that support skills. Instant use with text search, optional local postgres with faster and more accurate search on larger session histories. 

## The Problem

Claude Code forgets everything between sessions. You're deep in a feature, close the tab, and the next day ask "where were we?" only to get a blank stare.

> **Kuato:** What do you want, Mr. Quaid?   
> **Quaid:** The same as you; to remember.   
> **Kuato:** But why?   
> **Quaid:** To be myself again.   
> **Kuato:** You are what you do. A man is defined by his actions, not his memory.   

Kuato gives Claude access to what you *did* - the actions that define your work.

## Two Versions

| Feature | File-Based | PostgreSQL |
|---------|------------|------------|
| **Setup time** | 0 minutes | 5 minutes |
| **Dependencies** | Bun only | Bun + Postgres (via Docker or native) |
| **Search speed** | ~1-5 seconds | <100ms |
| **Full-text search** | Basic matching | Weighted, ranked |
| **Fuzzy matching** | No | Yes (trigram) |
| **API server** | No | Yes |
| **Token tracking** | Yes | Yes + per-model |
| **Best for** | Quick lookups | Daily use |

**Start with file-based.** Upgrade to PostgreSQL when you're running 10+ sessions a day and want faster, smarter search.

## Quick Start: File-Based

Zero setup. Works directly with Claude Code's JSONL files.

```bash
# Clone the repo
git clone https://github.com/alexknowshtml/kuato.git
cd kuato

# Search your sessions
bun run file-based/search.ts --query "email system" --days 7

# Filter by tools used
bun run file-based/search.ts --tools Edit,Bash --limit 10

# Filter by file patterns
bun run file-based/search.ts --file-pattern "components/"
```

Output is JSON with all session metadata:

```json
[
  {
    "id": "abc123-def456",
    "startedAt": "2025-01-15T10:30:00Z",
    "endedAt": "2025-01-15T11:45:00Z",
    "messageCount": 42,
    "toolsUsed": ["Edit", "Bash", "Read"],
    "filesFromToolCalls": ["src/api/email.ts", "src/utils/filter.ts"],
    "userMessages": [
      "Let's build an email filtering system",
      "Yes, use that approach",
      "Commit this and we'll continue tomorrow"
    ],
    "relevance": 23
  }
]
```

## Quick Start: PostgreSQL (Docker)

**Prerequisite:** [Docker Desktop](https://www.docker.com/products/docker-desktop/)

One-command database setup, then sync and search. Docker uses port 5433 to avoid conflicts with native PostgreSQL installations.

```bash
cd kuato/postgres

# Start PostgreSQL (creates database + schema)
bun run db:up

# Install dependencies
bun install

# Sync your sessions
DATABASE_URL="postgres://claude:sessions@localhost:5433/claude_sessions" bun run sync

# Start API server
DATABASE_URL="postgres://claude:sessions@localhost:5433/claude_sessions" bun run serve
```

## Quick Start: PostgreSQL (Native)

Alternative if you prefer not to use Docker.

```bash
# Install PostgreSQL via Homebrew (if not already installed)
brew install postgresql@14

# Start PostgreSQL
brew services start postgresql@14

# Create user and database
psql postgres -c "CREATE USER claude WITH PASSWORD 'sessions';"
psql postgres -c "CREATE DATABASE claude_sessions OWNER claude;"
psql postgres -c "GRANT ALL PRIVILEGES ON DATABASE claude_sessions TO claude;"

# Enable required extension and load schema
psql -d claude_sessions -c "CREATE EXTENSION IF NOT EXISTS pg_trgm;"
psql -U claude -d claude_sessions -f postgres/schema.sql

# Install dependencies and sync
cd kuato/postgres
bun install
DATABASE_URL="postgres://claude:sessions@localhost:5432/claude_sessions" bun run sync

# Start API server
DATABASE_URL="postgres://claude:sessions@localhost:5432/claude_sessions" bun run serve
```

**Note:** Native PostgreSQL uses the default port `5432`, while Docker uses `5433`. Adjust `DATABASE_URL` accordingly.

## Understanding the Two Servers

The PostgreSQL version requires **two separate servers** to be running:

| Server | Port | What it does | Auto-starts? |
|--------|------|--------------|--------------|
| **PostgreSQL** | 5432 (native) or 5433 (Docker) | The database storing your sessions | Yes - via `brew services` or Docker |
| **Kuato API** | 3847 | The REST API that queries the database | No - requires manual start or LaunchAgent |

PostgreSQL stores the data; the Kuato API server (`bun run serve`) provides the REST endpoints that Claude uses to search your sessions.

### Auto-starting the Kuato API (macOS)

To have the API server start automatically on login, install the included LaunchAgent:

```bash
# Copy the plist (edit paths inside if your kuato install location differs)
cp postgres/com.kuato.api.plist ~/Library/LaunchAgents/

# Load it now
launchctl load ~/Library/LaunchAgents/com.kuato.api.plist

# Verify it's running
curl "http://localhost:3847/health"
```

To manage the LaunchAgent:

```bash
# Stop the API server
launchctl unload ~/Library/LaunchAgents/com.kuato.api.plist

# Start it again
launchctl load ~/Library/LaunchAgents/com.kuato.api.plist

# View logs
tail -f /tmp/kuato-api.log
tail -f /tmp/kuato-api.error.log
```

Then query the API:

```bash
# Search
curl "http://localhost:3847/sessions?search=email+filtering&days=7"

# Get single session
curl "http://localhost:3847/sessions/abc123-def456"

# With full transcript
curl "http://localhost:3847/sessions/abc123-def456?with_transcript=true"

# Statistics
curl "http://localhost:3847/sessions/stats?days=30"
```

## Using with Claude Code

The real power is teaching Claude how to search your history. Create a skill file:

```markdown
# Session Search

When the user asks "where did we leave off" or "what did we discuss about X":

1. Search sessions:
   curl "http://localhost:3847/sessions?search=TOPIC&days=14"

2. The results include `user_messages` - these tell the whole story
   (requests, confirmations, decisions)

3. Summarize what happened and offer to continue
```

See `shared/claude-skill.md` for a complete skill template.

## How It Works

### The Key Insight: User Messages

You don't need the full transcript to understand what happened. User messages are the signal:

- "Let's build an email filtering system" → Request
- "Yes, use that approach" → Decision
- "Actually, make it async" → Correction
- "Commit this" → Completion

Combined with `files_touched` and `tools_used`, you can reconstruct the session without loading 50k tokens.

### File-Based Search

Simple substring matching on user messages, tools, and file paths.

```
Query: "septa"
↓
Scan each JSONL file
↓
Check if "septa" appears in:
  - userMessages[]
  - toolsUsed[]
  - filesFromToolCalls[]
↓
Score by match count
↓
Return sorted results
```

**Pros:** Zero setup, no dependencies beyond Bun
**Cons:** Slower on large histories, no stemming ("running" won't match "run")

### PostgreSQL Search

Full-text search with linguistic processing using `tsvector` and `tsquery`.

**Step 1: Text becomes tokens (tsvector)**

```sql
SELECT to_tsvector('english', 'The quick brown foxes are jumping');
-- Result: 'brown':3 'fox':4 'jump':5 'quick':2
```

Notice:
- **Stop words removed:** "The", "are" → gone
- **Stemming:** "foxes" → `fox`, "jumping" → `jump`
- **Positions tracked:** `fox:4` means 4th significant word

**Step 2: Queries match stems**

```sql
-- All of these match "The quick brown fox":
SELECT to_tsvector('english', 'The quick brown fox')
  @@ to_tsquery('english', 'foxes');      -- true (stems to fox)
  @@ to_tsquery('english', 'quick & brown'); -- true (AND)
  @@ to_tsquery('english', 'quick | cat');   -- true (OR)
```

**Step 3: Weights rank importance**

Sessions are indexed with weighted fields:

| Weight | Field | Priority |
|--------|-------|----------|
| A | `summary` | Highest (future use) |
| B | `search_text` | User messages |
| C | `tools_used`, `files_touched` | Tools and paths |
| D | `cwd` | Working directory |

```sql
-- "holiday" in user messages (B) ranks higher than in file path (C)
SELECT ts_rank(search_vector, to_tsquery('holiday')) FROM sessions;
```

**Step 4: GIN index makes it fast**

```sql
CREATE INDEX idx_sessions_search_vector ON sessions USING GIN(search_vector);
```

GIN (Generalized Inverted Index) is like a book index:
- Without index: scan every row → O(n)
- With GIN: lookup "holiday" → [session1, session5, session12] → O(log n)

**Step 5: File paths are tokenized**

Raw paths like `/Users/alex/septa-holiday-bus/App.jsx` become searchable tokens:

```sql
-- We transform: '/path/to/septa-holiday/' → 'path to septa holiday'
regexp_replace(path, '[/\-_.]', ' ', 'g')
```

So searching "septa" finds sessions that touched files in `septa-*` directories.

**Speed comparison:**
- File-based: ~150ms (substring scan)
- PostgreSQL: ~30ms (index lookup)

The gap grows with session count. At 1000+ sessions, PostgreSQL is 10-50x faster.

## API Reference (PostgreSQL)

### GET /sessions

Search sessions with filters.

| Param | Description |
|-------|-------------|
| `search` | Full-text search (weighted: summary > messages > tools) |
| `days` | Last N days only |
| `since` | Sessions after date (YYYY-MM-DD) |
| `until` | Sessions before date |
| `tools` | Filter by tool names (comma-separated) |
| `file_pattern` | Filter by file path pattern |
| `limit` | Max results (default 20, max 100) |

### GET /sessions/:id

Get single session. Add `?with_transcript=true` for full messages.

### GET /sessions/stats

Usage statistics. Add `?days=30` to customize window.

Returns: session count, token totals, breakdown by model.

## Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `CLAUDE_SESSIONS_DIR` | `~/.claude/projects` | Where Claude Code stores sessions |
| `DATABASE_URL` | `postgres://localhost/claude_sessions` | PostgreSQL connection |
| `PORT` | `3847` | API server port |

## Tips

### Scheduled Sync

Run sync every 15 minutes to keep the database current:

```bash
# crontab -e
*/15 * * * * cd /path/to/kuato/postgres && DATABASE_URL="..." bun run sync >> /var/log/kuato-sync.log 2>&1
```

### Search Strategies

**By topic:**
```bash
curl "http://localhost:3847/sessions?search=authentication"
```

**By file:**
```bash
curl "http://localhost:3847/sessions?file_pattern=src/auth"
```

**Recent activity:**
```bash
curl "http://localhost:3847/sessions?days=3&limit=50"
```

**Combined:**
```bash
curl "http://localhost:3847/sessions?search=refactor&tools=Edit&days=7"
```

## Contributing

Issues and PRs welcome. This started as a personal tool - feedback on what's useful appreciated.

## License

MIT
