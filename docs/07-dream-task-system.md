# Dream Task System

> **Key files:** `src/tasks/DreamTask/DreamTask.ts`, `src/services/autoDream/`

## Overview

The "dream" system is a background processing feature that runs when Claude Code is idle. Like dreaming during sleep, it performs housekeeping and analysis tasks in the background while the user isn't actively interacting.

## How It Works

### Trigger

Dreams are triggered by the `autoDream` service (`src/services/autoDream/`) when:
- The user is idle (no active conversation)
- Background tasks are not disabled (`DISABLE_BACKGROUND_TASKS` not set)
- Sufficient time has passed since the last dream

### DreamTask (`src/tasks/DreamTask/DreamTask.ts`)

Dreams run as a special task type in the task system. Unlike regular tasks that are user-initiated, dream tasks are system-initiated background processes.

The task lifecycle:
1. Auto-dream service detects idle state
2. Creates a `DreamTask` instance
3. Task runs in the background with minimal resource usage
4. Results are stored for later use
5. Task completes silently without user notification

### What Dreams Do

Based on the auto-dream service, dream tasks can perform:

- **Memory extraction**: Analyze recent conversations for memory-worthy content
- **Session summarization**: Generate summaries of recent work for future context
- **Housekeeping**: Clean up temporary files, update indexes

## Integration with Other Systems

### Background Task System

Dreams use the same task infrastructure as other background tasks:
- `src/tasks/` provides the task framework
- Tasks can be listed via `/tasks` command
- Tasks can be stopped via `TaskStopTool`

### Memory System

Dreams feed into the memory system:
- Auto-extracted memories from conversations
- Session memory summaries
- These are used by the compaction system for faster session memory compaction

### Auto-Dream Service (`src/services/autoDream/`)

The service manages:
- Idle detection timing
- Dream scheduling
- Resource throttling (prevents excessive background work)
- Integration with the session lifecycle

## Configuration

Dreams can be disabled via:
```bash
DISABLE_BACKGROUND_TASKS=1          # Disables all background tasks including dreams
CLAUDE_CODE_DISABLE_AUTO_MEMORY=1   # Disables memory extraction (a key dream activity)
```

## Relevance

The dream system explains why Claude Code sometimes "remembers" things from previous sessions even without explicit memory saves -- the background dream process may have extracted memories automatically.
