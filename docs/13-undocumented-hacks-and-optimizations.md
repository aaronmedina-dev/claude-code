# Undocumented Hacks & Optimizations

> Practical tips extracted from the source code that are not publicly documented. These let you control Claude Code's behavior, reduce costs, and work more efficiently.

## Environment Variable Hacks

### Control Context & Compaction

```bash
# Override the auto-compact threshold (default: context window - 33K)
CLAUDE_CODE_AUTO_COMPACT_WINDOW=150000

# Trigger compaction at a specific % of context used
CLAUDE_AUTOCOMPACT_PCT_OVERRIDE=85

# Disable auto-compaction (keep manual /compact only)
DISABLE_AUTO_COMPACT=1

# Disable ALL compaction
DISABLE_COMPACT=1

# Disable 1M context even if your model supports it
CLAUDE_CODE_DISABLE_1M_CONTEXT=1

# Cap maximum context tokens
CLAUDE_CODE_MAX_CONTEXT_TOKENS=500000
```

### Control Thinking & Output

```bash
# Minimal system prompt mode -- strips most instructions, faster and cheaper
CLAUDE_CODE_SIMPLE=1

# Disable extended thinking
CLAUDE_CODE_DISABLE_THINKING=1

# Disable interleaved thinking
DISABLE_INTERLEAVED_THINKING=1
```

### Control Background Behavior

```bash
# Disable all background tasks (auto-memory extraction, dreams, etc.)
DISABLE_BACKGROUND_TASKS=1

# Disable auto-memory extraction specifically
CLAUDE_CODE_DISABLE_AUTO_MEMORY=1
```

### Unlock Hidden Modes

```bash
# Coordinator mode: Claude orchestrates worker sub-agents instead of doing work itself
CLAUDE_CODE_COORDINATOR_MODE=1

# Fork sub-agents: omit subagent_type to fork full context into a background agent
CLAUDE_CODE_FORK_SUBAGENT=1
```

---

## The Most Powerful Undocumented Hack

**`CLAUDE_CODE_SIMPLE=1`** strips the system prompt down to a minimal version:

- Reduces input tokens significantly (the full system prompt is massive)
- Makes Claude respond faster
- Costs less per turn
- Loses all guardrails, style guidance, and tool usage tips

**Best for:** scripted/automated pipelines where you send structured prompts and don't need hand-holding. Not recommended for interactive use.

---

## CLAUDE.md Optimization Tricks

### How CLAUDE.md Is Actually Injected

The source reveals CLAUDE.md goes into the **dynamic** part of the system prompt, after the global cache boundary. This means:

1. **Keep CLAUDE.md concise** -- every token breaks the per-session prompt cache on change. Shorter = better cache hit rate.
2. **CLAUDE.md is loaded via the memory system**, not a simple file read. It goes through `loadMemoryPrompt()` which includes the full MEMORY.md index. Your project CLAUDE.md and memory files compete for the same attention window.
3. **Your CLAUDE.md is in position 10** out of 15+ sections -- after 9 sections of Anthropic's own instructions. If you want Claude to prioritize your instructions, use strong directive language.

### Steal Anthropic's Internal Prompt Tricks

Anthropic's internal builds include numeric length anchors that reduced output tokens by ~1.2%:

```
Length limits: keep text between tool calls to <=25 words.
Keep final responses to <=100 words unless the task requires more detail.
```

**Add similar instructions to your CLAUDE.md** to control verbosity:

```markdown
# Output Rules
- Keep responses under 100 words unless I ask for detail.
- Keep text between tool calls under 25 words.
- No preamble, no summaries, just do the work.
```

### The Exact Prompt Order Claude Sees

Understanding this lets you write CLAUDE.md that complements rather than repeats what's already there:

1. Identity ("You are Claude Code...")
2. Security guardrails (`CYBER_RISK_INSTRUCTION`)
3. System behavior rules (tool rules, permission modes, hooks)
4. Task execution guidelines (code quality, verification)
5. Safety/reversibility guidance (destructive operation warnings)
6. Tool usage guidance (when to use which tool)
7. Tone & style (emoji policy, markdown format)
8. **--- CACHE BOUNDARY ---**
9. Session-specific tool tips
10. **Your MEMORY.md + CLAUDE.md contents** <-- you are here
11. Environment info (CWD, git status, model, OS)
12. Language preference
13. Output style
14. MCP server instructions
15. Scratchpad guidance
16. Token management hints

Don't repeat what sections 1-7 already cover (e.g., "be concise", "use tools properly"). Instead, add project-specific context that Claude can't derive from the codebase.

---

## Hidden Slash Commands Worth Knowing

| Command | What It Does |
|---------|-------------|
| `/rewind` | Restore code AND conversation to a previous point (full undo) |
| `/compact <instructions>` | Custom compaction -- e.g., `/compact focus on the API changes, forget the CSS work` |
| `/context` | See what's actually in your context window |
| `/stats` | Token usage, costs, timing stats |
| `/cost` | Accumulated session cost |
| `/diff` | See what files changed this session |
| `/export` | Export conversation transcript |
| `/stickers` | Order Claude Code stickers (easter egg -- opens StickerMule) |
| `/rewind` (alias: `/checkpoint`) | Restore to earlier conversation state |

### Custom Compact Instructions

This is one of the most useful undocumented features. When running `/compact`, you can tell the compactor what to prioritize:

```
/compact Preserve all details about the API migration. Forget the CSS discussion.
/compact Keep the database schema decisions. Summarize everything else.
/compact Focus on the test failures and their fixes.
```

The compactor model uses your instructions to shape the summary. Much better than blind compaction for long sessions.

---

## Memory System Tricks

### MEMORY.md Hard Limits

```
Max lines:     200 (everything after is truncated)
Max size:      25KB
Entry length:  ~150 characters per line recommended
Max files:     200 per memory directory
```

If your MEMORY.md is bloated, Claude literally cannot see entries past line 200. **Prune regularly.**

### Memory Staleness Warnings

Memories older than 1 day get a caveat injected into the prompt:

> "This memory is X days old. Verify against current code before asserting as fact."

This makes Claude **second-guess old memories**. If you have important stable memories that shouldn't be doubted, `touch` the file to reset its modification time:

```bash
touch ~/.claude/projects/<project>/memory/important_memory.md
```

### Auto-Memory Extraction Details

The auto-extraction system:
- Runs as a **forked sub-agent** after every N turns
- Has a **5-turn budget** max per extraction run
- Can only write to the memory directory (sandboxed)
- Checks for conflicts -- if you manually wrote to memory this turn, extraction is skipped
- Is throttled by GrowthBook feature flags

**Tip**: Explicitly saying "remember this" is more reliable than hoping auto-extraction catches it, because auto-extraction has throttling and conflict detection that can cause it to skip turns.

---

## Tool Permission Tricks

### The `acceptEdits` Permission Mode

Not widely documented -- besides `default`, `plan`, and `bypassPermissions`, there's an `acceptEdits` mode that auto-approves these specific operations without prompting:

- `mkdir`, `touch`, `rm`, `rmdir`, `mv`, `cp`, `sed`

This is a useful middle ground: you get auto-approved file operations without full bypass mode.

### Bash Commands That Never Prompt

These read-only commands are auto-approved in default permission mode -- no dialog, no delay:

**Git:** `git log`, `git show`, `git diff`, `git status`, `git config --get`, `git branch`, `git remote`

**Files:** `cat`, `head`, `tail`, `less`, `more`, `file`, `strings`, `od`, `xxd`, `hexdump`

**Search:** `grep`, `rg`, `find`, `fd`, `locate`, `which`, `type`

**Docker:** `docker ps`, `docker logs`, `docker inspect`, `docker images`

**Other:** `xargs` (read-only context), `jq`, `pyright`

Knowing this lets you predict when Claude will pause for permission vs. run immediately.

### Per-Command Sandbox Bypass

The `dangerouslyDisableSandbox` parameter on individual Bash tool calls bypasses the macOS Seatbelt sandbox for that specific command. The model knows about this but rarely uses it unless the sandbox is blocking a legitimate operation (e.g., network access from a sandboxed command).

---

## Compaction Optimization

### Microcompaction Runs Silently

Two types happen before every API call without you knowing:

1. **Cached microcompaction**: Deletes oldest tool results via API `cache_edits` (preserves prompt cache)
2. **Time-based microcompaction**: If idle >60 minutes, clears old tool results (cache expired anyway)

**Practical impact**: If you step away for an hour and come back, your oldest tool results have been silently removed. This is why Claude sometimes says "let me re-read that file" after breaks -- the earlier read result was garbage-collected.

### Compaction Circuit Breaker

After 3 consecutive auto-compact failures, the system stops retrying. If you notice compaction isn't working, run `/compact` manually to reset the circuit breaker.

### Partial Compaction

You can compact a specific range by selecting messages in the UI:
- **Direction `from`**: Summarize messages AFTER a point (keeps earlier messages + prompt cache)
- **Direction `up_to`**: Summarize messages BEFORE a point (keeps later messages, breaks cache)

---

## Agent Optimization

### Explore Agent Uses Haiku (Cheapest Model)

The built-in `Explore` agent uses Haiku for external builds -- it's the cheapest and fastest model. It's read-only (Glob, Grep, Bash, Read) and skips CLAUDE.md loading (`omitClaudeMd: true`). Use it liberally for codebase searches without worrying about cost.

### One-Shot Agents Skip Overhead

`Explore` and `Plan` agents are in `ONE_SHOT_BUILTIN_AGENT_TYPES` -- they skip the agentId/SendMessage instructions, saving tokens on every invocation. These agents are designed for single-query, fire-and-forget use.

### Custom Agents via Markdown Files

Create custom agents in `~/.claude/agents/` (global) or `.claude/agents/` (project):

```yaml
---
name: test-runner
description: Run tests and report results
tools: [Bash, Read, Glob]
model: haiku
maxTurns: 10
---
You are a test runner agent. Run the test suite and report failures concisely.
Do not explain the failures in detail -- just list them.
```

This gives you a cheap, specialized agent you can invoke via `Agent(subagent_type: "test-runner")`. Using `model: haiku` makes it very cheap.

### Available Agent Frontmatter Options

The full set of options you can use in custom agent markdown files:

```yaml
---
name: my-agent                    # Required: agent type identifier
description: What this agent does # Required: shown in UI and prompt
tools: [Read, Bash, Glob]        # Optional: tool allowlist (default: all)
disallowedTools: [Write]          # Optional: tool denylist
model: haiku                      # Optional: model override (haiku/sonnet/opus)
effort: high                      # Optional: effort level
maxTurns: 20                      # Optional: turn limit
memory: project                   # Optional: persistent memory scope (user/project/local)
isolation: worktree               # Optional: run in git worktree
background: true                  # Optional: run in background by default
color: red                        # Optional: UI color
skills: skill1, skill2            # Optional: skills to preload
initialPrompt: /run-tests         # Optional: slash command to run on start
permissionMode: plan              # Optional: permission mode override
mcpServers: [slack, github]       # Optional: MCP servers to connect
---
```

---

## Cost Reduction Strategies (from Source Analysis)

1. **Use `CLAUDE_CODE_SIMPLE=1`** for automated/scripted pipelines -- massive prompt reduction
2. **Use Explore agent** for searches -- it runs on Haiku, skips CLAUDE.md
3. **Keep CLAUDE.md short** -- it's injected on every turn, so 1KB of CLAUDE.md = 1KB x every API call
4. **Use custom compact instructions** -- `/compact keep only X` prevents re-doing work after compaction
5. **Create Haiku-based custom agents** for repetitive tasks (linting, testing, searching)
6. **Set `CLAUDE_CODE_MAX_CONTEXT_TOKENS`** to a lower value if you don't need the full context window -- triggers compaction earlier, keeps per-turn costs down
7. **Disable auto-memory** (`CLAUDE_CODE_DISABLE_AUTO_MEMORY=1`) if you don't use it -- saves the background extraction sub-agent cost
8. **Disable background tasks** (`DISABLE_BACKGROUND_TASKS=1`) for short scripted sessions -- eliminates all background agent spawns
