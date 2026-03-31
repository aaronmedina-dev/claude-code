# Context Compaction

> **Key files:** `src/services/compact/`, `src/query/`, `src/constants/apiLimits.ts`

## Overview

As conversations grow, they approach the model's context window limit. Claude Code uses a multi-layered compaction system to manage this: continuous microcompaction removes old tool results non-disruptively, auto-compaction triggers full summarization when needed, and manual `/compact` gives users explicit control.

## Three Levels of Compaction

### Level 1: Microcompaction (Continuous, Non-Disruptive)

Runs before every API call. Two variants:

**Cached Microcompaction** (API `cache_edits`):
- Deletes oldest tool results via API-level edits
- Does NOT modify local messages (edits sent separately)
- Preserves prompt cache validity
- Keeps N most recent tool results (configurable via GrowthBook)
- Gate: `tengu_cache_plum_violet` feature flag

**Time-Based Microcompaction**:
- Triggers when gap since last assistant message > threshold (default 60 minutes)
- Cache has expired server-side anyway (1-hour TTL)
- Clears oldest tool results, keeps recent N (default 5)
- Directly mutates message content
- Gate: `tengu_slate_heron` feature flag

### Level 2: Auto-Compaction (Threshold-Based)

Triggers when token usage exceeds a threshold:

```
effectiveContextWindow = modelContextWindow - 20,000 (output reservation)
autoCompactThreshold = effectiveContextWindow - 13,000 (buffer)
```

For a 200K context model: triggers at ~167K tokens.
For a 1M context model: triggers at ~967K tokens.

### Level 3: Manual `/compact` Command

User explicitly runs `/compact [optional instructions]`:
- Can provide custom summarization instructions
- Tries session memory compaction first (cheaper)
- Falls back to full compaction

## Auto-Compaction Logic

### `shouldAutoCompact()` Predicate

```typescript
// Returns false if:
1. autoCompactEnabled === false (user config)
2. DISABLE_COMPACT or DISABLE_AUTO_COMPACT env vars set
3. querySource is blacklisted (session_memory, compact, marble_origami)
4. Reactive-only mode active
5. Context-collapse mode active

// Returns true if:
6. tokenCount >= autoCompactThreshold
```

### Blacklisted Query Sources

Prevents recursion -- these sources can't trigger auto-compact:
- `session_memory`: Forked agent for session memory extraction
- `compact`: The summarization agent itself
- `marble_origami`: Context-collapse agent (has its own management)

### Circuit Breaker

After 3 consecutive auto-compact failures:
- Stops retrying (prevents API call spam)
- Session continues (user can manually `/compact`)
- Resets on successful compaction

## Full Compaction Algorithm

### Pre-Compaction

1. Execute `PreCompact` hooks (customizable per environment)
2. Strip images and documents from messages (not needed for summary)
3. Strip reinjected attachments (skill_discovery, skill_listing)
4. Estimate token count of input

### Summarization

The model receives a specialized prompt asking it to produce a summary with these sections:

1. **Primary Request and Intent** -- what the user wants
2. **Key Technical Concepts** -- technologies, patterns, decisions
3. **Files and Code Sections** -- full snippets for recent work
4. **Errors and Fixes** -- what went wrong and how it was resolved
5. **Problem Solving Efforts** -- approaches tried
6. **All User Messages** -- preserved verbatim
7. **Pending Tasks** -- incomplete work
8. **Current Work** -- what was being done when compact triggered
9. **Optional Next Step** -- with direct quotes from conversation

The model uses `<analysis>` tags for internal drafting (stripped from output) and `<summary>` tags for the final result.

### Prompt-Too-Long Recovery

If the compact request itself exceeds the context limit:
1. Drop oldest API-round groups (using `groupMessagesByApiRound()`)
2. Use 20% fallback if token gap can't be parsed
3. Retry up to 3 times before giving up

### Post-Compaction Reconstruction

After summarization, the system rebuilds context:

1. Clear file read cache and memory paths
2. **Restore recent files**: Up to 5 most recently-read files (5K tokens per file max, 50K total budget)
3. **Restore skills**: Skills used in session (5K per skill, 25K total budget)
4. **Tool deltas**: Announce new/changed tools since last session
5. **MCP instructions**: Re-inject current MCP state
6. **Plan mode**: Preserve mode flags if in plan mode
7. Execute `SessionStart` hooks
8. Create compact boundary marker with metadata

## What's Preserved vs. Lost

### Preserved

| Content | How | Budget |
|---------|-----|--------|
| Conversation summary | Model-generated 9-section summary | ~17K tokens |
| Recent messages | Last N messages after boundary | Configurable |
| Recently-read files | Up to 5 files with edits | 50K tokens total |
| Invoked skills | Full text of used skills | 25K tokens total |
| Tool deltas | New/changed tools | Variable |
| MCP instructions | Updated MCP state | Variable |
| Plan mode state | Mode flags | Minimal |
| Discovered tools | Set of announced tools | Minimal |
| Transcript path | Link to pre-compaction transcript | Reference only |

### Lost

| Content | Why | Mitigation |
|---------|-----|------------|
| Full conversation history | Replaced with summary | Transcript file stored on disk |
| Raw tool outputs | Cleared from context | Recent files re-readable |
| Detailed error messages | Merged into summary | Summary captures key errors |
| Image/document binaries | Stripped before summarization | Noted as `[image]` in summary |
| Old file edits | Not re-listed unless recent | Transcript has full history |
| Intermediate reasoning | Merged into summary sections | Analysis blocks ensure coverage |

## Partial Compaction

Users can compact a specific range of messages:

**Direction `from`**: Summarize messages AFTER a pivot point
- Keeps earlier messages verbatim
- Preserves prompt cache for kept messages

**Direction `up_to`**: Summarize messages BEFORE a pivot point
- Keeps later messages verbatim
- Invalidates prompt cache (summary replaces prefix)

## Session Memory Compaction

A faster alternative tried before full compaction:
- Uses pre-extracted summaries from the session memory extraction service
- No API call needed for summarization
- Better preservation of tool_use/tool_result pairing
- Configurable thresholds via GrowthBook: `tengu_sm_compact_config`
- Falls back to full compaction if unavailable

## Token Limits and Thresholds

| Threshold | Value | Purpose |
|-----------|-------|---------|
| Model Context Window | 200K (default) / 1M (extended) | API hard limit |
| MAX_OUTPUT_TOKENS_FOR_SUMMARY | 20,000 | Reserved for compact summary output |
| AUTOCOMPACT_BUFFER_TOKENS | 13,000 | Buffer before auto-compact triggers |
| WARNING_THRESHOLD_BUFFER_TOKENS | 20,000 | Show "context low" warning in UI |
| ERROR_THRESHOLD_BUFFER_TOKENS | 20,000 | Show "context critical" error |
| MANUAL_COMPACT_BUFFER_TOKENS | 3,000 | Buffer for manual /compact to succeed |
| POST_COMPACT_MAX_FILES_TO_RESTORE | 5 | Max files restored post-compact |
| POST_COMPACT_TOKEN_BUDGET | 50,000 | Total budget for file restoration |
| POST_COMPACT_MAX_TOKENS_PER_FILE | 5,000 | Per-file restoration limit |
| POST_COMPACT_MAX_TOKENS_PER_SKILL | 5,000 | Per-skill restoration limit |
| POST_COMPACT_SKILLS_TOKEN_BUDGET | 25,000 | Total skill restoration budget |

## Warning System

`calculateTokenWarningState()` returns:

```typescript
{
  percentLeft: number,                    // % of effective context remaining
  isAboveWarningThreshold: boolean,       // Show warning UI (~80% used)
  isAboveErrorThreshold: boolean,         // Show critical error
  isAboveAutoCompactThreshold: boolean,   // Auto-compact should fire
  isAtBlockingLimit: boolean              // Block further API calls
}
```

## Configuration

### Environment Variables

```bash
DISABLE_COMPACT=true                      # Disable all compaction
DISABLE_AUTO_COMPACT=true                 # Disable only auto-compaction
CLAUDE_CODE_AUTO_COMPACT_WINDOW=150000    # Override effective context window
CLAUDE_AUTOCOMPACT_PCT_OVERRIDE=85        # Trigger at percentage of window
CLAUDE_CODE_BLOCKING_LIMIT_OVERRIDE=190000 # Override blocking limit
CLAUDE_CODE_DISABLE_1M_CONTEXT=true       # Disable 1M context
CLAUDE_CODE_MAX_CONTEXT_TOKENS=200000     # Cap max context tokens
```

### GrowthBook Feature Flags

| Flag | Purpose |
|------|---------|
| `tengu_cobalt_raccoon` | Reactive-only mode (suppress proactive autocompact) |
| `tengu_prompt_cache_break_detection` | Track cache invalidation from compaction |
| `tengu_compact_cache_prefix` | Share prompt cache with forked summary agent |
| `tengu_sm_compact_config` | Session memory compaction thresholds |
| `tengu_slate_heron` | Time-based microcompact config |
| `tengu_cache_plum_violet` | Cached microcompact enabled |

## State Resets After Compaction

- Microcompact state: reset
- Context collapse state: reset
- Memory file cache: cleared and re-armed
- Classifier approvals: cleared
- System prompt sections: cleared
- Prompt cache baseline: reset

**NOT reset** (intentionally):
- Invoked skill content (must survive for re-injection)
- Tool definitions (schema must remain available)

## Key Source Files

| File | Purpose |
|------|---------|
| `compact.ts` | Main compaction logic and post-compact reconstruction |
| `autoCompact.ts` | Auto-compact trigger logic and thresholds |
| `microCompact.ts` | Cached microcompaction (API cache_edits) |
| `timeBasedMCConfig.ts` | Time-based microcompaction configuration |
| `prompt.ts` | Summarization prompt templates |
| `grouping.ts` | Message grouping by API round |
| `sessionMemoryCompact.ts` | Session memory-based compaction |
| `postCompactCleanup.ts` | State cleanup after compaction |
| `compactWarningState.ts` | Token warning state tracking |
| `compactWarningHook.ts` | Warning UI hook |
| `apiMicrocompact.ts` | API-level microcompaction integration |
