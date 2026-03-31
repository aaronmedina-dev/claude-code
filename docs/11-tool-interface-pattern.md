# Tool Interface Pattern

> **Key files:** `src/Tool.ts`, `src/tools.ts`, `src/tools/*/`, `src/skills/`

## Overview

Every capability Claude Code has -- reading files, running commands, searching code, spawning agents -- is implemented as a "tool." Tools follow a consistent interface pattern with a standard directory structure. Understanding this pattern is the key to understanding how Claude Code extends its capabilities.

## The Tool Interface (`src/Tool.ts`)

The core `Tool` type defines what every tool must provide:

```typescript
// Simplified from the actual interface
type Tool = {
  name: string                          // Unique identifier (e.g., "Bash", "Read")
  description: string                   // Human-readable description
  inputSchema: ToolInputJSONSchema      // JSON Schema for tool parameters

  isReadOnly(): boolean                 // Does this tool only read data?
  needsPermissions(): boolean           // Does this require user approval?
  validateInput(input): ValidationResult // Validate input before execution

  call(input, context): AsyncGenerator   // Execute the tool (async generator for streaming)

  // Optional
  isEnabled?(): boolean                 // Is this tool currently available?
  getPermissionDescription?(): string   // Description for permission dialog
  renderToolUse?(input): React.Element  // Custom UI for tool use display
  renderToolResult?(result): React.Element // Custom UI for result display
}
```

### Key Types

```typescript
type ToolInputJSONSchema = {
  type: 'object'
  properties?: { [x: string]: unknown }
}

type Tools = Map<string, Tool>  // Tool registry

type ToolProgressData =
  | BashProgress        // Shell command progress
  | AgentToolProgress   // Sub-agent progress
  | MCPProgress         // MCP tool progress
  | WebSearchProgress   // Web search progress
  | REPLToolProgress    // REPL progress
  | SkillToolProgress   // Skill execution progress
  | HookProgress        // Hook execution progress
  | TaskOutputProgress  // Task output progress
```

## Standard Directory Structure

Each tool follows a consistent pattern:

```
src/tools/
  MyTool/
    MyTool.ts (or .tsx)   # Tool implementation
    prompt.ts              # Tool description for system prompt
    constants.ts           # Tool name and config constants
    UI.tsx                 # React component for rendering in terminal
    utils.ts               # (optional) Helper utilities
    types.ts               # (optional) Type definitions
```

### Example: Simple Tool (GlobTool)

**`GlobTool/prompt.ts`** -- Tool description:
```typescript
export const GLOB_TOOL_NAME = 'Glob'
export const getGlobToolPrompt = () => `
- Fast file pattern matching tool that works with any codebase size
- Supports glob patterns like "**/*.js" or "src/**/*.ts"
- Returns matching file paths sorted by modification time
...
`
```

**`GlobTool/GlobTool.ts`** -- Implementation:
```typescript
export const GlobTool: Tool = {
  name: GLOB_TOOL_NAME,
  description: getGlobToolPrompt(),
  inputSchema: {
    type: 'object',
    properties: {
      pattern: { type: 'string', description: 'Glob pattern to match' },
      path: { type: 'string', description: 'Directory to search in' }
    },
    required: ['pattern']
  },

  isReadOnly: () => true,
  needsPermissions: () => false,

  async *call(input, context) {
    const matches = await glob(input.pattern, { cwd: input.path })
    yield { type: 'result', content: matches.join('\n') }
  }
}
```

**`GlobTool/UI.tsx`** -- Terminal rendering:
```typescript
export function GlobToolUI({ input, result }) {
  return (
    <Box flexDirection="column">
      <Text>Pattern: {input.pattern}</Text>
      <Text>{result.content}</Text>
    </Box>
  )
}
```

### Example: Complex Tool (BashTool)

The BashTool has the most files due to its security complexity:

```
BashTool/
  BashTool.tsx              # Main implementation
  prompt.ts                 # System prompt description
  toolName.ts               # Tool name constant
  commandSemantics.ts       # Command classification (safe/dangerous)
  bashCommandHelpers.ts     # Command parsing utilities
  bashSecurity.ts           # Git safety checks
  bashPermissions.ts        # Permission check logic
  shouldUseSandbox.ts       # macOS Seatbelt sandbox decision
  destructiveCommandWarning.ts  # Dangerous command detection
  pathValidation.ts         # Directory scope validation
  readOnlyValidation.ts     # Read-only command detection
  modeValidation.ts         # Permission mode checks
  sedValidation.ts          # Sed command validation
  commentLabel.ts           # Tool call labeling
  utils.ts                  # Helper utilities
  BashToolResultMessage.tsx # Result display component
  UI.tsx                    # Terminal UI component
```

## Tool Registration (`src/tools.ts`)

All tools are assembled in `src/tools.ts`:

```typescript
export function getTools(options): Tools {
  const tools = new Map<string, Tool>()

  // Built-in tools
  tools.set(BashTool.name, BashTool)
  tools.set(FileReadTool.name, FileReadTool)
  tools.set(FileEditTool.name, FileEditTool)
  tools.set(FileWriteTool.name, FileWriteTool)
  tools.set(GlobTool.name, GlobTool)
  tools.set(GrepTool.name, GrepTool)
  tools.set(AgentTool.name, AgentTool)
  // ... more tools

  // MCP tools (dynamically added from connected servers)
  for (const [name, tool] of mcpTools) {
    tools.set(name, tool)
  }

  return tools
}
```

## Prompt Description Integration

Tool descriptions from `prompt.ts` files are assembled into the system prompt. The model sees:

```
You have access to a set of tools you can use to answer the user's question.

<tool name="Bash">
Executes a given bash command and returns its output.
...
</tool>

<tool name="Read">
Reads a file from the local filesystem.
...
</tool>
```

The prompt descriptions are critical -- they tell the model when and how to use each tool.

## Permission Integration

The permission system checks each tool call:

```typescript
// In useCanUseTool.tsx
function canUseTool(tool: Tool, input: any): PermissionResult {
  if (tool.isReadOnly()) return { allowed: true }

  if (permissionMode === 'plan') {
    return { allowed: false, reason: 'Plan mode: write tools blocked' }
  }

  if (permissionMode === 'bypassPermissions') {
    return { allowed: true }  // Plus safety checks
  }

  // Check user-defined rules
  const rule = findMatchingRule(tool.name, input)
  if (rule) return { allowed: rule.action === 'allow' }

  // Needs user approval
  return { allowed: false, needsApproval: true }
}
```

## UI Rendering Pattern

Each tool's `UI.tsx` provides two rendering modes:

1. **Tool Use Display**: Shows what the tool was called with (before execution)
2. **Tool Result Display**: Shows the result (after execution)

```typescript
// UI.tsx pattern
export function MyToolUI({ input, result, isStreaming }) {
  if (!result) {
    // Tool use display (before result)
    return <Text>Running: {input.command}</Text>
  }

  // Tool result display
  return (
    <Box flexDirection="column">
      <Text color="green">Result:</Text>
      <Text>{result.content}</Text>
    </Box>
  )
}
```

## The Skill System

Skills extend the tool pattern with markdown-defined prompt templates.

### Skill Definition

Skills live in `src/skills/bundled/` as markdown files:

```yaml
---
name: commit
description: Create a git commit with the staged changes
user_invocable: true
trigger: "when the user asks to commit"
---
# Instructions for committing

When creating a commit:
1. Run git status to see changes
2. Stage relevant files
3. Write a descriptive commit message
...
```

### Skill Registration (`bundledSkills.ts`)

```typescript
export const bundledSkills = [
  { name: 'commit', file: 'commit.md' },
  { name: 'review-pr', file: 'review-pr.md' },
  { name: 'simplify', file: 'simplify.md' },
  // ...
]
```

### User-Defined Skills

Users can create custom skills:
- `~/.claude/skills/*.md` -- global skills
- `.claude/skills/*.md` -- project skills
- Loaded by `loadSkillsDir.ts`

### Skill Execution

Skills are invoked via the `SkillTool`:
1. User types `/skill-name` or model calls `Skill({ skill: "name" })`
2. `SkillTool` loads the skill's markdown content
3. Content is injected as context for the model
4. Model follows the skill's instructions

### SkillTool (`src/tools/SkillTool/SkillTool.ts`)

```typescript
export const SkillTool: Tool = {
  name: SKILL_TOOL_NAME,
  inputSchema: {
    type: 'object',
    properties: {
      skill: { type: 'string', description: 'Skill name' },
      args: { type: 'string', description: 'Optional arguments' }
    },
    required: ['skill']
  },

  async *call(input) {
    const skill = findSkill(input.skill)
    const content = loadSkillContent(skill)
    yield { type: 'result', content }
  }
}
```

## Writing a Custom Tool

Based on the patterns observed, a custom tool would need:

1. **Define the tool** implementing the `Tool` interface
2. **Create a prompt** describing when/how to use it
3. **Register it** in `tools.ts` (or as an MCP tool)
4. **Create UI** for terminal display (optional)
5. **Add permissions** if the tool modifies state

### Minimal Custom Tool Example

```typescript
// MyTool/MyTool.ts
export const MyTool: Tool = {
  name: 'MyTool',
  description: 'Does something useful',
  inputSchema: {
    type: 'object',
    properties: {
      query: { type: 'string' }
    },
    required: ['query']
  },
  isReadOnly: () => true,
  needsPermissions: () => false,
  validateInput: (input) => ({ valid: true }),
  async *call(input) {
    const result = await doSomething(input.query)
    yield { type: 'result', content: result }
  }
}

// MyTool/prompt.ts
export const MY_TOOL_NAME = 'MyTool'
export const getMyToolPrompt = () => `
Does something useful with the user's query.
Use this when the user asks for...
`
```

In practice, **MCP servers are the intended extension mechanism** -- they let you add tools without modifying Claude Code's source.

## Tool Categories

| Category | Tools | Characteristics |
|----------|-------|-----------------|
| **File I/O** | Read, Write, Edit, Glob, Grep | Core file operations |
| **Execution** | Bash, PowerShell | Shell command execution (heavy security) |
| **Agent** | Agent, SendMessage, TeamCreate/Delete | Multi-agent orchestration |
| **Task** | TaskCreate/Get/List/Stop/Update/Output | Background task management |
| **Planning** | EnterPlanMode, ExitPlanMode | Planning workflow |
| **Isolation** | EnterWorktree, ExitWorktree | Git worktree isolation |
| **MCP** | MCPTool, ReadMcpResource, ListMcpResources, McpAuth | MCP server integration |
| **Web** | WebFetch, WebSearch | Internet access |
| **Notebook** | NotebookEdit | Jupyter notebook editing |
| **Meta** | ToolSearch, Config, Skill, Brief | Tool discovery and configuration |
| **Legacy** | TodoWrite, SyntheticOutput, Sleep | Older/utility tools |
