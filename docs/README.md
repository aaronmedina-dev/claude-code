# Claude Code Internals - Documentation Index

Detailed documentation of Claude Code's internal architecture, extracted from the leaked source code (2026-03-31).

## Understanding How Claude Code Works

| # | Document | Description |
|---|----------|-------------|
| 1 | [System Prompt Assembly](./01-system-prompt-assembly.md) | How the system prompt is dynamically constructed from 15+ sections |
| 2 | [Tool Permissions & Sandboxing](./02-tool-permissions-and-sandboxing.md) | Permission modes, macOS Seatbelt sandbox, bash security layers |
| 3 | [Agent & Sub-Agent System](./03-agent-sub-agent-system.md) | Agent spawning, built-in agents, lifecycle, swarm coordination |
| 4 | [Context Compaction](./04-context-compaction.md) | Auto-compact, microcompaction, token budgets, summarization |
| 5 | [Memory System](./05-memory-system.md) | Persistent memory types, MEMORY.md, auto-extraction, aging |

## Reverse-Engineering Undocumented Behavior

| # | Document | Description |
|---|----------|-------------|
| 6 | [Feature Flags & Hidden Commands](./06-feature-flags-and-hidden-commands.md) | `bun:bundle` feature gates, GrowthBook A/B tests, hidden slash commands |
| 7 | [Dream Task System](./07-dream-task-system.md) | Background "dreaming" during idle periods |
| 8 | [Permission Bypass Mode](./08-permission-bypass-mode.md) | How bypass mode works and its safeguards |
| 9 | [Undocumented Hacks & Optimizations](./13-undocumented-hacks-and-optimizations.md) | Env vars, hidden commands, cost reduction, CLAUDE.md tricks, agent hacks |

## Architecture Deep Dives

| # | Document | Description |
|---|----------|-------------|
| 10 | [Custom Ink Fork](./09-custom-ink-fork.md) | React reconciler for terminal, Yoga layout, event system |
| 11 | [MCP Client Implementation](./10-mcp-client-implementation.md) | Server discovery, connection lifecycle, tool proxying, OAuth |
| 12 | [Tool Interface Pattern](./11-tool-interface-pattern.md) | Tool anatomy, registration, prompt/UI pattern, skill system |
| 13 | [Bridge System](./12-bridge-system.md) | CLI-to-Desktop/Web communication, REPL bridge, JWT auth |

---

Each document is self-contained and can be read independently. Start with **[System Prompt Assembly](./01-system-prompt-assembly.md)** for the single most valuable piece of information in the leak.
