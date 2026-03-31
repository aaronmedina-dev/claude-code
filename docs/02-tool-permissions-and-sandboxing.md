# Tool Permissions & Sandboxing

> **Key files:** `src/types/permissions.ts`, `src/utils/permissions/`, `src/tools/BashTool/`, `src/utils/sandbox/`

## Overview

Claude Code has a multi-layered security model that controls what the AI can do. Every tool call passes through permission checks, and dangerous operations (shell commands, file writes) have additional security layers including macOS sandboxing, destructive command detection, and path validation.

## Permission Modes

Three permission modes control how tool calls are approved:

### 1. Default Mode (`default`)

The standard interactive mode:
- **Read-only tools** (Glob, Grep, FileRead): auto-approved
- **Write tools** (FileEdit, FileWrite, BashTool): require user approval
- User sees a permission dialog and can approve/deny each tool call
- Approvals can be granted per-instance or as rules (e.g., "allow all git commands")

### 2. Plan Mode (`plan`)

Restricted mode where Claude can only read and plan:
- Read-only tools: auto-approved
- Write tools: **blocked** -- Claude must ask to exit plan mode first
- Uses `EnterPlanModeTool` / `ExitPlanModeTool` to transition
- Useful for reviewing what Claude wants to do before allowing execution

### 3. Bypass Mode (`bypassPermissions`)

Full auto-approval with safety guardrails:
- All tool calls auto-approved without user prompts
- **Still enforces:** destructive command warnings, path validation, sandbox
- Gated by: `isBypassPermissionsModeDisabled()` check
- Can be disabled by enterprise MDM policy
- Requires explicit user opt-in (trust dialog acceptance)

## Permission Check Flow

When Claude calls a tool, this sequence runs:

```
1. Tool.call() invoked
   |
2. useCanUseTool() hook checks:
   |-- Is tool read-only? -> Auto-approve
   |-- Is permission mode "plan"? -> Block if write tool
   |-- Is permission mode "bypass"? -> Auto-approve (with safety checks)
   |-- Is tool in user's approved rules? -> Auto-approve
   |
3. If not auto-approved -> Show PermissionRequest UI
   |-- User approves -> Execute
   |-- User denies -> Return denial to model
   |-- User creates rule -> Save rule, execute
   |
4. Tool-specific validation:
   |-- BashTool: sandbox check, path validation, destructive warning
   |-- FileEditTool: path validation, content validation
   |-- FileWriteTool: path validation
   |-- PowerShellTool: security checks
```

## Permission Rules

Users can create persistent permission rules:

```typescript
// Example rule structure
{
  tool: "Bash",
  pattern: "git *",       // Glob pattern matching command
  scope: "project",       // project | global
  action: "allow"         // allow | deny
}
```

Rules are stored in:
- `~/.claude/settings.json` (global rules)
- `.claude/settings.json` (project rules)

## Bash Tool Security Layers

The `BashTool` is the most heavily guarded tool with 7 security layers:

### Layer 1: Mode Validation (`modeValidation.ts`)

Checks if Bash is allowed in current permission mode:
- Plan mode: blocks all Bash execution
- Default mode: requires approval
- Bypass mode: auto-approves

### Layer 2: Path Validation (`pathValidation.ts`)

Ensures commands operate within allowed directories:
- Commands must target the project directory or allowed additional directories
- Prevents `cd /etc && rm -rf *` style attacks
- Validates both explicit paths in commands and working directory

### Layer 3: Read-Only Validation (`readOnlyValidation.ts`)

Detects if a command is read-only or mutating:
- `git status`, `ls`, `cat` -> read-only
- `git commit`, `rm`, `echo >` -> mutating
- Read-only commands get auto-approved in some contexts

### Layer 4: Command Semantics (`commandSemantics.ts`)

Classifies commands by their impact:
- **Safe**: `ls`, `pwd`, `echo`, `cat`, `git log`
- **File-modifying**: `rm`, `mv`, `cp`, `chmod`
- **System-modifying**: `kill`, `shutdown`, `systemctl`
- **Network**: `curl`, `wget`, `ssh`

### Layer 5: Destructive Command Warning (`destructiveCommandWarning.ts`)

Detects dangerous patterns and shows warnings:

```typescript
// Examples of detected destructive patterns:
"git reset --hard"     // Data loss
"git push --force"     // Remote data loss
"rm -rf /"             // System destruction
"git clean -fd"        // Untracked file deletion
"git checkout ."       // Uncommitted change loss
```

The warning is injected into the tool result so the model sees it and can reconsider.

### Layer 6: Git Safety (`bashSecurity.ts`)

Specific protections for git operations:
- Blocks `git push --force` to main/master
- Warns on `--no-verify` (skipping hooks)
- Warns on `--no-gpg-sign` (skipping signing)
- Detects `git rebase -i` (requires interactive input)
- Prevents `git config` modifications

### Layer 7: macOS Seatbelt Sandbox (`shouldUseSandbox.ts`)

On macOS, commands can be wrapped in a Seatbelt sandbox:

```typescript
// Sandbox profile restricts:
// - File system access to project directory only
// - No network access (for some commands)
// - No process spawning outside allowed list
// - No system modification
```

**When sandbox is used:**
- Non-interactive commands in default mode
- Commands that don't need network access
- When `CLAUDE_CODE_DISABLE_SANDBOX` is not set

**When sandbox is skipped:**
- Interactive commands (e.g., `npm login`)
- Commands requiring network (e.g., `npm install`, `git push`)
- When user explicitly disables sandbox
- In bypass permission mode (still has other safety layers)

### Sed Validation (`sedValidation.ts`)

Special validation for `sed` commands:
- Parses sed expressions to determine if they're file-modifying
- `sed -n 'p'` -> read-only (auto-approve)
- `sed -i 's/foo/bar/'` -> file-modifying (requires approval)
- Validates sed syntax to prevent injection

## PowerShell Security

`src/tools/PowerShellTool/` mirrors the Bash security model for Windows:

- `powershellPermissions.ts` -- permission check logic
- `powershellSecurity.ts` -- dangerous command detection
- `modeValidation.ts` -- plan/bypass mode checks
- `destructiveCommandWarning.ts` -- PowerShell-specific destructive patterns
- `pathValidation.ts` -- Windows path validation
- `readOnlyValidation.ts` -- cmdlet classification

## File Operation Permissions

### FileEditTool

- Requires file to have been read first (prevents blind edits)
- `old_string` must be unique in the file (prevents ambiguous edits)
- Path validation ensures edits are within project

### FileWriteTool

- Requires file to have been read first (for existing files)
- New file creation allowed without prior read
- Path validation for project scope

## Permission UI Components

Each tool type has a dedicated permission request component in `src/components/permissions/`:

| Component | Tool | Shows |
|-----------|------|-------|
| `BashPermissionRequest` | Bash | Command, working directory, security analysis |
| `FileEditPermissionRequest` | FileEdit | File path, old/new string diff |
| `FileWritePermissionRequest` | FileWrite | File path, content preview |
| `PowerShellPermissionRequest` | PowerShell | Command, security analysis |
| `WebFetchPermissionRequest` | WebFetch | URL, method |
| `NotebookEditPermissionRequest` | NotebookEdit | Cell changes |
| `SkillPermissionRequest` | Skill | Skill name, arguments |

## Denial Tracking

`src/utils/permissions/denialTracking.ts` tracks when users deny tool calls:
- After repeated denials, the system adjusts behavior
- Denial patterns are used to improve future permission requests
- Auto-mode denial logic prevents infinite permission request loops

## Enterprise/MDM Controls

Enterprise administrators can enforce permission policies via MDM (Mobile Device Management):
- `src/utils/settings/mdm/` handles MDM policy reading
- Can disable bypass mode entirely
- Can enforce minimum permission levels
- Can restrict which tools are available
- Settings from `src/services/remoteManagedSettings/` override user preferences
