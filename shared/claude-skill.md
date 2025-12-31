# Kuato - Session Memory Skill

*"Open your mind."*

Use this skill when the user asks about previous sessions, what was discussed, or wants to resume work.

## Trigger Phrases

- "where did we leave off"
- "what did we discuss about X"
- "find the session where we"
- "when did we work on"
- "resume from where we were"
- "what were we doing with"

## Workflow

### Step 1: Search for Relevant Sessions

**If using PostgreSQL API:**
```bash
curl -s "http://localhost:3847/sessions?search=TOPIC&days=14&limit=5"
```

**If using file-based search:**
```bash
bun run /path/to/kuato/file-based/search.ts --query "TOPIC" --days 14
```

Also try file pattern if it's a code topic:
```bash
curl -s "http://localhost:3847/sessions?file_pattern=relevant-path&days=14"
```

### Step 2: Extract Context from Results

**CRITICAL: The search results already contain `user_messages` - use this FIRST.**

The `user_messages` array contains ALL user inputs from the session. This is usually enough to understand what happened without loading the full transcript.

Key patterns to look for in user_messages:
- Requests: "Let's build X", "I need to"
- Confirmations: "Yes", "Do it", "That works"
- Corrections: "Actually", "No wait", "Instead"
- Completions: "Commit this", "Ship it", "Done"

Cross-reference with:
- `files_touched` - What code was modified
- `tools_used` - What operations were performed
- `ended_at` - When the session ended

### Step 3: Get Full Transcript (if needed)

Only fetch if user_messages isn't sufficient:

**PostgreSQL:**
```bash
curl -s "http://localhost:3847/sessions/SESSION_ID?with_transcript=true"
```

**File-based:**
Read the file at `transcriptPath` from search results.

### Step 4: Summarize What Happened

From user_messages, identify:
1. What the user was trying to accomplish
2. What decisions were made (look for confirmations)
3. What work was completed
4. What the final state was

Keep summaries to 3-5 bullet points.

### Step 5: Present and Offer Options

Present a clear summary, then offer relevant next actions:

**Example response:**

> **Email Filtering System (Jan 15)**
>
> - Built Gmail filter system with rule matching
> - Implemented archive, label, and delete actions
> - Created 3 filter rules for newsletters
> - Tested on 50 emails, 94% accuracy
> - Paused at: "Need to add exception handling"
>
> Would you like to:
> - Continue where we left off
> - See the full session transcript
> - Search for related sessions

## Example Queries

**Find recent work on a feature:**
```bash
curl "http://localhost:3847/sessions?search=authentication&days=7"
```

**Find sessions that modified specific files:**
```bash
curl "http://localhost:3847/sessions?file_pattern=src/auth"
```

**Find sessions where specific tools were used:**
```bash
curl "http://localhost:3847/sessions?tools=Edit,Bash&days=14"
```

**Combine filters:**
```bash
curl "http://localhost:3847/sessions?search=refactor&tools=Edit&file_pattern=components"
```

## Response Format

Search results include:

| Field | Description |
|-------|-------------|
| `id` | Session UUID |
| `startedAt` | Start timestamp |
| `endedAt` | End timestamp |
| `messageCount` | Total messages in session |
| `userMessages` | Array of user inputs |
| `toolsUsed` | Array of tool names |
| `filesFromToolCalls` | Array of file paths |
| `modelsUsed` | Array of model names |
| `inputTokens` | Total input tokens |
| `outputTokens` | Total output tokens |
| `relevance` | Search relevance score |

## Tips

1. **User messages are gold** - They tell the story without needing transcripts
2. **File patterns work well** - Code work usually touches specific paths
3. **Combine search + filters** - Narrow down large result sets
4. **Recent sessions first** - Use `days` param to limit scope
5. **Cross-reference files** - `files_touched` reveals what was actually modified
