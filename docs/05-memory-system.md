# Memory System

> **Key files:** `src/memdir/`, `src/services/extractMemories/`, `src/services/SessionMemory/`, `src/utils/memory/`

## Overview

Claude Code has a persistent, file-based memory system that allows it to remember context across conversations. Memories are stored as markdown files with YAML frontmatter in the user's `~/.claude/` directory and project `.claude/` directory. An index file `MEMORY.md` is always loaded into the conversation context.

## Memory Types

Four discrete memory types, each with different purposes:

### 1. User Memories

Information about the user's role, goals, preferences, and knowledge.

```yaml
---
name: user_role
description: User is a senior backend engineer focused on Go microservices
type: user
---
Senior Go developer with 10+ years experience, currently focused on microservice
architecture and observability.
```

**When saved:** When learning details about the user's role, preferences, or expertise.
**How used:** Tailoring responses to the user's skill level and perspective.

### 2. Feedback Memories

Guidance about how to approach work -- corrections and confirmed approaches.

```yaml
---
name: feedback_no_mocks
description: Integration tests must use real database, not mocks
type: feedback
---
Integration tests must hit a real database, not mocks.

**Why:** Prior incident where mock/prod divergence masked a broken migration.

**How to apply:** When writing tests for database-related code, always use the
test database container, never mock the DB layer.
```

**When saved:** When the user corrects an approach OR confirms a non-obvious approach worked.
**How used:** Guides behavior so the user doesn't need to repeat guidance.

### 3. Project Memories

Information about ongoing work, goals, initiatives within the project.

```yaml
---
name: project_merge_freeze
description: Merge freeze begins 2026-03-05 for mobile release cut
type: project
---
Merge freeze begins 2026-03-05 for mobile release cut.

**Why:** Mobile team is cutting a release branch -- all non-critical merges paused.

**How to apply:** Flag any non-critical PR work scheduled after that date.
```

**When saved:** When learning about who is doing what, why, or by when.
**How used:** Understanding broader context behind user requests.

### 4. Reference Memories

Pointers to where information can be found in external systems.

```yaml
---
name: reference_linear_bugs
description: Pipeline bugs tracked in Linear project "INGEST"
type: reference
---
Pipeline bugs are tracked in Linear project "INGEST".
Check this when investigating pipeline-related issues.
```

**When saved:** When learning about resources in external systems.
**How used:** When the user references an external system.

## File Structure

### Memory Storage Locations

```
~/.claude/
  projects/<project-hash>/
    memory/
      MEMORY.md              # Index file (always loaded into context)
      user_role.md           # Individual memory files
      feedback_testing.md
      project_freeze.md
      reference_linear.md
```

### MEMORY.md Index

`MEMORY.md` is an index file, not a memory itself. Each entry is one line under ~150 characters:

```markdown
- [User Role](user_role.md) -- senior Go developer, microservices focus
- [No Mocks in Tests](feedback_testing.md) -- use real DB, not mocks
- [Merge Freeze](project_freeze.md) -- freeze starts 2026-03-05
- [Linear Bugs](reference_linear.md) -- pipeline bugs in INGEST project
```

**Important constraints:**
- Always loaded into conversation context
- Lines after 200 are truncated
- No frontmatter in MEMORY.md itself
- One-line entries only

### Individual Memory Files

Each memory is a separate markdown file with YAML frontmatter:

```yaml
---
name: {{memory name}}
description: {{one-line description used for relevance matching}}
type: {{user | feedback | project | reference}}
---

{{memory content}}
```

## Memory Loading

### `loadMemoryPrompt()` (`memdir.ts`)

Called during system prompt assembly to inject memories:

1. Reads `MEMORY.md` from the project memory directory
2. Loads referenced memory files
3. Formats as a system prompt section
4. Returns the formatted prompt text (or null if no memories)

### Memory Paths (`paths.ts`)

Memory directories are resolved relative to the project:
- `~/.claude/projects/<project-hash>/memory/` -- project-scoped memories
- `~/.claude/memory/` -- global memories (shared across projects)

## Auto-Memory Extraction

### `src/services/extractMemories/`

Claude Code can automatically extract memories from conversations:

**Trigger:** After significant interactions, the system analyzes the conversation for memory-worthy information.

**What gets extracted:**
- User corrections ("don't do X") -> feedback memories
- User confirmations of approach -> feedback memories
- Role/preference revelations -> user memories
- Project context mentions -> project memories
- External system references -> reference memories

**What does NOT get extracted:**
- Code patterns/conventions (derivable from code)
- Git history (use `git log`)
- Debugging solutions (fix is in the code)
- Anything in CLAUDE.md files
- Ephemeral task details

### Extraction Process

1. Conversation analyzed for memory-worthy content
2. Candidate memories generated
3. Deduplication against existing memories
4. New memories written to individual files
5. `MEMORY.md` index updated

## Memory Aging (`memoryAge.ts`)

Memories can become stale over time:

- Project memories decay fastest (work context changes)
- User memories are more durable (personal traits stable)
- Reference memories are semi-stable (systems may change)
- Feedback memories are highly durable (learned preferences)

The aging system helps future conversations evaluate whether a memory is still relevant.

## Memory Scanning (`memoryScan.ts`)

Scans the memory directory for:
- Orphaned memory files (not in MEMORY.md)
- Stale memories that may need updating
- Duplicate or conflicting memories
- Index consistency checks

## Session Memory (`src/services/SessionMemory/`)

Separate from the persistent memory system, session memory tracks:
- Per-session conversation summaries
- Used for faster session memory compaction
- Enables resuming conversations with context

## Team Memory Sync (`src/services/teamMemorySync/`)

In multi-agent swarm scenarios:
- Team memories can be synchronized across agents
- Team-specific memory prompts (`teamMemPrompts.ts`)
- Ensures all team members share common project context

## Agent Memory (`src/tools/AgentTool/agentMemory.ts`)

Sub-agents have their own memory system:

### Three Scopes

| Scope | Path | Purpose |
|-------|------|---------|
| `user` | `~/.claude/agent-memory/` | Cross-project agent knowledge |
| `project` | `.claude/agent-memory/` | Project-specific agent knowledge |
| `local` | `.claude/agent-memory-local/` | Local-only agent knowledge |

### Agent Memory Loading

`loadAgentMemoryPrompt()` reads the agent's MEMORY.md and injects it into the agent's system prompt.

### Snapshot Propagation

`agentMemorySnapshot.ts` supports project-scoped snapshots that propagate to local scope on first agent spawn.

## The `/memory` Command

`src/commands/memory/` provides CLI access to the memory system:

- View all memories
- Search memories
- Edit specific memories
- Clear/reset memories
- Memory management UI

## Memory and the System Prompt

The memory loading flow in the system prompt assembly:

```
getSystemPrompt()
  -> systemPromptSection('memory', () => loadMemoryPrompt())
    -> reads MEMORY.md
    -> reads referenced .md files
    -> formats as prompt section
    -> cached until /clear or /compact
```

Memory is injected as part of the dynamic (per-session) prompt content, after the cache boundary.

## What NOT to Save

The system explicitly excludes:
- Code patterns, conventions, architecture (read the code)
- Git history and recent changes (use `git log`)
- Debugging solutions (fix is in code, commit message has context)
- Anything in CLAUDE.md files
- Ephemeral task details, in-progress work
- PR lists or activity summaries (ask what was surprising instead)

## Memory Verification

Before acting on memories, Claude Code verifies:
- If memory names a file path: check the file exists
- If memory names a function/flag: grep for it
- If user is about to act on recommendation: verify first

"The memory says X exists" is not the same as "X exists now."
