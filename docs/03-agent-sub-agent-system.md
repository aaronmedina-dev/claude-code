# Agent & Sub-Agent System

> **Key files:** `src/tools/AgentTool/`, `src/utils/swarm/`, `src/tasks/`, `src/coordinator/`

## Overview

Claude Code implements a sophisticated multi-agent architecture where the main conversation can spawn sub-agents (child conversations) to perform work in parallel or with specialized capabilities. The system supports built-in agents, user-defined agents, fork-based agents, and full team/swarm coordination.

## Agent Spawning

Agents are spawned via the `AgentTool` (`src/tools/AgentTool/AgentTool.tsx`). A typical invocation:

```typescript
Agent({
  description: "Search for API endpoints",
  prompt: "Find all REST API endpoints in this codebase",
  subagent_type: "Explore",      // Built-in or custom agent type
  isolation: "worktree",         // Optional: run in git worktree
  run_in_background: true,       // Optional: async execution
  model: "haiku"                 // Optional: model override
})
```

### Spawn Flow

1. Resolve agent definition (built-in, custom, or fork)
2. Validate against permission context
3. Create agent ID (unique async task identifier)
4. Assemble tool pool via `resolveAgentTools()`
5. Generate system prompt via agent's `getSystemPrompt()`
6. Register as `LocalAgentTask` background task
7. Start `runAsyncAgentLifecycle()` in background

## Built-In Agent Types

All built-in agents are defined in `src/tools/AgentTool/builtInAgents.ts` and `src/tools/AgentTool/built-in/`:

| Agent Type | File | Purpose | Tools | Model |
|------------|------|---------|-------|-------|
| **general-purpose** | `generalPurposeAgent.ts` | Research, code search, multi-step tasks | All (`*`) | Default |
| **Explore** | `exploreAgent.ts` | Fast codebase exploration | Glob, Grep, Bash, Read | Haiku |
| **Plan** | `planAgent.ts` | Architecture planning, design | Glob, Grep, Bash, Read | Inherit |
| **verification** | `verificationAgent.ts` | Code testing and validation | Most + Browser MCP | Inherit |
| **Claude Code Guide** | `claudeCodeGuideAgent.ts` | Help with Claude Code itself | All | Default |
| **statusline-setup** | `statuslineSetup.ts` | Configure status line | Read, Edit | Default |

### Explore Agent (Detail)

The Explore agent is optimized for speed:
- Uses Haiku model (cheapest, fastest) for external builds
- Read-only tools only (Glob, Grep, Bash, Read)
- `omitClaudeMd: true` -- skips loading CLAUDE.md to save tokens
- Three thoroughness levels: "quick", "medium", "very thorough"
- `ONE_SHOT_BUILTIN_AGENT_TYPES` -- skips agentId/SendMessage trailers to save tokens

### Plan Agent

Design-focused agent:
- Read-only tools (cannot modify files)
- `omitClaudeMd: true` for token efficiency
- Returns step-by-step plans, identifies critical files
- Considers architectural trade-offs

### Verification Agent (Ant-only A/B test)

Adversarial testing agent:
- Gated by `VERIFICATION_AGENT` feature flag + `tengu_hive_evidence` GrowthBook flag
- Runs in background
- Tests code changes independently
- Has access to Browser MCP tools for UI testing

## Tool Resolution

`resolveAgentTools()` in `agentToolUtils.ts` determines what tools each agent gets:

### Three configuration methods:

1. **Wildcard `['*']`**: Agent receives all available tools (minus disallowed)
2. **Allowlist**: Specific tool names enumerated in agent definition
3. **Denylist**: All tools except those in `disallowedTools`

### Filtering rules:

- All MCP tools (`mcp__*`) are always allowed through
- `ALL_AGENT_DISALLOWED_TOOLS` (from `constants/tools.ts`) blocks certain tools for all agents
- Custom agents additionally blocked from `CUSTOM_AGENT_DISALLOWED_TOOLS`
- Async agents restricted to `ASYNC_AGENT_ALLOWED_TOOLS` (Bash, Read, Write, Edit, File tools, MCP)

## Agent Lifecycle

### Run Phase (`runAgent.ts`)

The agent runs as an async generator:

```
1. Build messages (history + prompt + context)
2. API loop: stream messages to Claude API
3. Tool execution: agent uses tools, results streamed back
4. Check turn limit (maxTurns)
5. On completion: signal cache eviction, return result
```

### Resume Phase (`resumeAgent.ts`)

Agents can be resumed after stopping:

1. Reload transcript from disk (`~/.claude/tasks/<agentId>/`)
2. Reconstruct conversation state
3. For fork resume: re-thread parent's rendered system prompt
4. Register new async task
5. Continue with fresh prompt via `SendMessageTool`

### Memory (`agentMemory.ts`)

Agents have persistent memory across sessions:
- **Three scopes**: `user` (`~/.claude/agent-memory/`), `project` (`.claude/agent-memory/`), `local` (`.claude/agent-memory-local/`)
- `loadAgentMemoryPrompt()` reads MEMORY.md and injects into agent's system prompt
- Snapshot support (`agentMemorySnapshot.ts`): project-scoped snapshots propagate to local scope

## Inter-Agent Communication

### SendMessage Tool (`SendMessageTool.ts`)

Routes messages between agents:
- **By name**: Send to a specific teammate
- **Broadcast**: `to: "*"` sends to all team members
- **Remote**: UDS socket or bridge session routing

Message types:
- **Plain text**: Requires `summary` (5-10 words for UI preview)
- **Structured JSON**: `shutdown_request`, `shutdown_response`, `plan_approval_response`

### Routing Logic

```
1. Target is running in-process subagent?
   -> Queue message, deliver at next tool round
2. Target is stopped subagent?
   -> Auto-resume via resumeAgentBackground()
3. No active task?
   -> Attempt resume from disk transcript
4. Fallback:
   -> Write to team mailbox for async delivery
```

### Mailbox System (`swarm/teamHelpers.ts`)

For async team coordination:
- Write to `~/.claude/teams/<team>/mailboxes/<member>/<timestamp>.json`
- Poll-based delivery (teammates periodically check mailbox)
- Used for cross-process team members (tmux panes, etc.)

## Multi-Agent Swarm System

### Team Structure

Teams are configured in `~/.claude/teams/<team-name>/config.json`:

```json
{
  "teamName": "my-team",
  "members": [
    {
      "agentId": "agent-123",
      "name": "researcher",
      "agentType": "general-purpose",
      "model": "sonnet",
      "color": "blue",
      "cwd": "/path/to/project",
      "worktreePath": "/path/to/worktree",
      "backendType": "in-process"
    }
  ],
  "leadAgentId": "lead-456"
}
```

### Leader/Worker Architecture

**Leader (Team Lead)**:
- Central coordinator (`TEAM_LEAD_NAME = "team-lead"`)
- Spawns/kills teammates, approves plans, handles shutdown
- Can be human user or coordinator agent

**Worker Types**:
1. **In-process teammates**: Same Node process, `AsyncLocalStorage` isolation
2. **Pane-backed teammates**: Separate tmux/iTerm2 panes, subprocess execution
3. **Remote teammates**: Claude Code Remote environments (ant-only)

### Team Coordination

- `SendMessage` for direct messaging
- Mailbox for async delivery to pane-backed teammates
- `<task-notification>` XML blocks for task status updates
- Shutdown coordination via structured messages
- Plan approval workflow for plan-mode-required agents

## Worktree Isolation

Setting `isolation: "worktree"` gives an agent an isolated copy of the repo:

```
1. git worktree add /tmp/worktree-<id> HEAD
2. Agent cwd overridden to worktree path
3. Agent gets notice about path translation
4. Changes stay isolated in worktree
5. Cleanup: automatic if no changes; path returned if changes made
```

The worktree notice injected into the agent's context:

```
You've inherited conversation context from parent working in /path/original.
You are operating in isolated worktree at /path/worktree.
Paths in context refer to parent's directory; translate them to your worktree root.
Re-read files before editing if parent may have modified them.
```

## Fork Subagent

When `FORK_SUBAGENT` feature is enabled and `subagent_type` is omitted:

- Creates an **implicit fork** inheriting parent's full context + system prompt
- Background async execution with notification-based interaction
- Parent's system prompt threaded byte-exact for cache parity

### Fork Safety Rules

Fork children get boilerplate constraints:
1. Must NOT spawn sub-agents; execute directly
2. Must not converse or suggest next steps
3. Use tools directly, report once at end
4. If modifying files, commit + report hash
5. Emit no text between tool calls
6. Response format: "Scope: ... Result: ... Key files: ... Files changed: ..."

Recursive forking is blocked -- `isInForkChild()` checks for fork boilerplate tag.

## User-Defined Agents

### Agent Definition Sources

Agents are loaded from multiple locations (`loadAgentsDir.ts`):

1. **Built-in**: Hardcoded in code
2. **Plugin**: From connected MCP servers
3. **User**: `~/.claude/agents/*.md`
4. **Project**: `.claude/agents/*.md`
5. **Managed**: From admin/policy settings
6. **CLI flags**: JSON via `--agent` or env

### Agent File Format

```yaml
---
name: my-agent
description: What this agent does
tools: [Read, Bash, Glob]           # Optional allowlist
disallowedTools: [Write]            # Optional denylist
model: sonnet                       # Optional model override
effort: high                        # Optional effort level
maxTurns: 20                        # Optional turn limit
memory: project                     # Optional persistent memory scope
isolation: worktree                 # Optional isolation mode
background: true                    # Optional background default
color: red                          # Optional UI color
skills: skill1, skill2             # Optional skills to preload
initialPrompt: /run-tests          # Optional slash command injection
permissionMode: plan               # Optional permission mode
mcpServers: [slack, github]        # Optional MCP servers
---
System prompt content here.
You are a specialized agent that...
```

Files are parsed from YAML frontmatter + markdown content, validated via Zod schema.

## Coordinator Mode

Enabled via `CLAUDE_CODE_COORDINATOR_MODE=1` environment variable.

`src/coordinator/coordinatorMode.ts` provides a specialized prompt where Claude acts as an orchestrator:

### Coordinator Behavior

- Never executes tasks directly -- delegates to "worker" subagents
- Spawns workers via `Agent(subagent_type: "worker")`
- Receives `<task-notification>` XML updates as workers progress
- Synthesizes findings and crafts detailed implementation specs

### Coordinator Phases

1. **Research**: Workers explore codebase in parallel
2. **Synthesis**: Coordinator reads findings, crafts implementation spec
3. **Implementation**: Workers implement per spec, commit
4. **Verification**: Workers test changes

### Key Guidance

- Synthesis is the coordinator's most important job
- "Never delegate understanding" -- must read and comprehend worker findings
- Include file paths, line numbers, exact changes in specs
- Launch independent workers simultaneously (multiple tool calls in one message)
- Use `SendMessage` to continue existing workers when context overlaps
