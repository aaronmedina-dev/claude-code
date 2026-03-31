# System Prompt Assembly

> **Key file:** `src/constants/prompts.ts` (914 lines) -- the single most valuable file in the leak.

## Overview

Claude Code's behavior is defined by a dynamically-assembled system prompt. The prompt is not a static string -- it's constructed at runtime from 15+ sections, each conditionally included based on the current model, enabled tools, feature flags, user settings, and environment.

## Entry Point

The main function is `getSystemPrompt()` in `src/constants/prompts.ts`. It returns a `string[]` (array of prompt sections) that gets joined before being sent to the API.

## Prompt Structure (In Order)

The system prompt has a **two-part structure** separated by a cache boundary marker:

### Part 1: Static Content (Globally Cacheable)

These sections are identical across users/sessions and can be cached at the API level:

| Order | Section | Function | Description |
|-------|---------|----------|-------------|
| 1 | Identity + Security | `getSimpleIntroSection()` | "You are Claude Code..." + `CYBER_RISK_INSTRUCTION` |
| 2 | System Behavior | `getSimpleSystemSection()` | Tool rules, permission modes, hooks, auto-compression |
| 3 | Task Execution | `getSimpleDoingTasksSection()` | Code quality, verification, security, output styles |
| 4 | Safety Guidance | `getActionsSection()` | Reversibility warnings, destructive operation checks |
| 5 | Tool Usage | `getUsingYourToolsSection()` | When to use dedicated tools vs. Bash, parallelization |
| 6 | Tone & Style | `getSimpleToneAndStyleSection()` | Emoji policy, markdown format, no colon before tools |

### Cache Boundary

```typescript
...(shouldUseGlobalCacheScope() ? [SYSTEM_PROMPT_DYNAMIC_BOUNDARY] : []),
```

The boundary marker `__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__` separates globally-cacheable content from per-session content. This lets the API cache the static prefix across all users.

### Part 2: Dynamic Content (Per-Session)

| Order | Section | Function | Description |
|-------|---------|----------|-------------|
| 7 | Session Guidance | `getSessionSpecificGuidanceSection()` | Conditional tool-specific tips based on enabled tools |
| 8 | Memory | `loadMemoryPrompt()` | Contents of user's `MEMORY.md` + memory files |
| 9 | Environment | `computeSimpleEnvInfo()` | CWD, git status, platform, shell, OS, model, cutoff date |
| 10 | Language | `getLanguageSection()` | User's preferred language (if set) |
| 11 | Output Style | `getOutputStyleSection()` | Active output style instructions |
| 12 | MCP Instructions | `getMcpInstructionsSection()` | Connected MCP server instructions |
| 13 | Scratchpad | `getScratchpadInstructions()` | Temp file directory guidance (if enabled) |
| 14 | Result Clearing | `getFunctionResultClearingSection()` | Token management hints for cached results |
| 15 | Token Budget | (feature-gated) | User-specified token targets |

## Key Files

| File | Lines | Purpose |
|------|-------|---------|
| `src/constants/prompts.ts` | 914 | Main prompt assembly orchestrator |
| `src/constants/system.ts` | 96 | System prefix selection + attribution headers |
| `src/constants/systemPromptSections.ts` | 69 | Section caching mechanism |
| `src/constants/cyberRiskInstruction.ts` | 25 | Security guardrail text (owned by Safeguards team) |
| `src/constants/outputStyles.ts` | 217 | Output style configuration and loading |

## System Prefix Selection

`getCLISyspromptPrefix()` in `system.ts` returns one of three prefixes based on execution context:

```typescript
const DEFAULT_PREFIX = `You are Claude Code, Anthropic's official CLI for Claude.`
const AGENT_SDK_CLAUDE_CODE_PRESET_PREFIX = `You are Claude Code, Anthropic's official CLI for Claude, running within the Claude Agent SDK.`
const AGENT_SDK_PREFIX = `You are a Claude agent, built on Anthropic's Claude Agent SDK.`
```

Selection logic:
- Vertex API provider -> always `DEFAULT_PREFIX`
- Agent SDK with Claude Code preset -> `AGENT_SDK_CLAUDE_CODE_PRESET_PREFIX`
- Agent SDK without preset -> `AGENT_SDK_PREFIX`
- Everything else -> `DEFAULT_PREFIX`

## Dynamic Content Injection

### Environment Info (`computeSimpleEnvInfo`)

Injects runtime context the model needs:

```
- Primary working directory: /path/to/project
  - Is a git repository: true
- Platform: darwin
- Shell: zsh
- OS Version: Darwin 25.3.0
- Model: Claude Opus 4.6 (1M context). Model ID: claude-opus-4-6[1m]
- Knowledge cutoff: May 2025
```

Knowledge cutoff dates are hardcoded per model:
- Sonnet 4.6: August 2025
- Opus 4.6/4.5: May 2025
- Haiku 4: February 2025
- Opus/Sonnet 4: January 2025

### Tool-Specific Guidance (`getSessionSpecificGuidanceSection`)

The prompt includes different instructions depending on which tools are available. For example:
- If `FileReadTool` is available: guidance on reading files before editing
- If `AgentTool` is available: when to use subagents vs. direct tools
- If `GlobTool` + `GrepTool` available: prefer these over bash `find`/`grep`
- If `SkillTool` available: how to invoke skills

### CLAUDE.md Injection

The contents of `CLAUDE.md` files are loaded and injected into the system prompt. The hierarchy:
1. Global `~/.claude/CLAUDE.md`
2. Project `.claude/CLAUDE.md` (and `CLAUDE.md` at project root)
3. Includes from `@include` directives

These are loaded by the memory/config system and injected as part of the dynamic content.

### MCP Server Instructions

When MCP servers are connected, their instructions are formatted as:

```
# MCP Server Instructions
## server-name
<server instructions here>
```

Only servers with `type: "connected"` that have non-empty instructions are included.

## Caching Strategy

### Section-Level Caching

Each dynamic section uses the caching infrastructure from `systemPromptSections.ts`:

```typescript
// Cached: computed once, reused until /clear or /compact
systemPromptSection('memory', () => loadMemoryPrompt())

// Uncached: recomputed every turn (breaks prompt cache)
DANGEROUS_uncachedSystemPromptSection(
  'mcp_instructions',
  () => getMcpInstructionsSection(mcpClients),
  'MCP servers connect/disconnect between turns'
)
```

Only MCP instructions use `DANGEROUS_uncachedSystemPromptSection` because MCP servers can connect/disconnect between turns.

### Global Cache Boundary

The `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` marker enables:
- **Static prefix**: `scope: 'global'` -- cached across all users/orgs
- **Dynamic content**: default session scope -- cached per-session

This optimization is controlled by `shouldUseGlobalCacheScope()`.

## Feature-Gated Sections

### Proactive/Autonomous Mode

When `PROACTIVE` or `KAIROS` features are enabled, a completely different prompt path is used:
- Simplified autonomous agent prompt
- Uses `<tick>` prompts to keep the model active
- `SLEEP_TOOL_NAME` for pacing between actions
- Terminal focus detection for collaboration level

### Verification Agent (Ant-only A/B test)

When `VERIFICATION_AGENT` feature + `tengu_hive_evidence` GrowthBook flag are active:
- Non-trivial changes require a verification subagent review
- Adds a "contract" section to the system prompt

### Token Budget

When `TOKEN_BUDGET` feature is active:
- Model instructed to respect user-specified token targets
- Shows output token count each turn
- System auto-continues if model stops early

### Numeric Length Anchors (Ant-only)

Internal Anthropic builds include:
```
Length limits: keep text between tool calls to <=25 words.
Keep final responses to <=100 words unless the task requires more detail.
```
Research showed ~1.2% output token reduction with quantitative anchors.

## Ant-Only vs. External Builds

The codebase has many conditional blocks gated by `process.env.USER_TYPE === 'ant'`:

**Ant-only additions:**
- Comment minimization guidance
- False claims mitigation
- Bug reporting instructions for Claude Code issues
- Detailed output efficiency prose
- Numeric length anchors

**External (3P) additions:**
- More concise output efficiency instructions
- Token-awareness hints

Since `USER_TYPE` is set at build time and `'ant'` comparisons are inlined, the Bun bundler's dead code elimination (DCE) removes ant-only code from external builds entirely.

## Output Style System

Built-in output styles in `outputStyles.ts`:

| Style | Description |
|-------|-------------|
| `default` | No special instructions |
| `Explanatory` | Adds educational insights with star icons |
| `Learning` | Interactive mode requesting user contributions |

Styles are loaded from multiple sources with priority: `plugin < user < project < managed`.

The active style is retrieved via `getOutputStyleConfig()` and its instructions are injected into the system prompt.

## Undercover Mode

When `isUndercover()` returns true:
- All model names and IDs are stripped from the prompt
- Prevents internal model info from leaking into public commits/PRs
- Only relevant for internal Anthropic builds

## Integration Points

The system prompt connects to:

| System | How |
|--------|-----|
| Agent tool | `enhanceSystemPromptWithEnvDetails()` for sub-agent prompts |
| Skills | `getSkillToolCommands()` for skill descriptions |
| GrowthBook | `getFeatureValue_CACHED_MAY_BE_STALE()` for feature flags |
| Permissions | `isScratchpadEnabled()` for scratchpad section |
| Memory | `loadMemoryPrompt()` for MEMORY.md contents |
| Compact service | `getCachedMCConfig()` for microcompact config |
| Worktree | `getCurrentWorktreeSession()` for worktree notice |
| Settings | `getInitialSettings()` for language, output style |
