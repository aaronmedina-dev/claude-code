# Permission Bypass Mode

> **Key files:** `src/utils/permissions/permissionSetup.ts`, `src/types/permissions.ts`, `src/migrations/migrateBypassPermissionsAcceptedToSettings.ts`

## Overview

Bypass mode (`bypassPermissions`) auto-approves all tool calls without user prompts. It's the "trust Claude fully" mode -- but it still enforces safety guardrails. Understanding how it works reveals the trust boundary between "user approval" and "hard safety limits."

## How to Enable

Bypass mode requires explicit opt-in:

1. User runs Claude Code with `--dangerously-skip-permissions` flag or equivalent setting
2. A **trust dialog** appears explaining the risks
3. User must accept the trust dialog
4. Setting is persisted in config

The trust dialog acceptance is tracked via `checkHasTrustDialogAccepted()` in `src/utils/config.ts`.

## What Bypass Mode Does

### Auto-Approved (No User Prompt)

- All tool calls execute without permission dialogs
- File reads, writes, edits -- all auto-approved
- Bash/PowerShell commands -- auto-approved
- Agent spawning -- auto-approved
- MCP tool calls -- auto-approved
- Web fetch/search -- auto-approved

### Still Enforced (Even in Bypass)

These safety layers run regardless of permission mode:

| Safety Layer | What It Does |
|-------------|--------------|
| Destructive command warnings | `git push --force`, `rm -rf /` still flagged |
| Path validation | Commands must target project directory |
| Git safety | Force-push to main/master blocked |
| Sandbox (macOS) | Seatbelt sandbox still applies where configured |
| Cyber risk instruction | Security guardrails in system prompt unchanged |
| Tool input validation | Schema validation still runs |

The key insight: bypass mode removes the **user confirmation step** but does NOT remove **programmatic safety checks**. The model still gets warnings about dangerous operations injected into its context.

## Enterprise Disable

Enterprise administrators can disable bypass mode entirely:

```typescript
function isBypassPermissionsModeDisabled(): boolean {
  // Checks MDM policy and remote managed settings
  // Returns true if enterprise has blocked bypass mode
}
```

When disabled:
- `--dangerously-skip-permissions` flag is rejected
- Settings migration removes any existing bypass acceptance
- `createDisabledBypassPermissionsContext()` creates a context that blocks bypass

Controlled via:
- MDM (Mobile Device Management) policies (`src/utils/settings/mdm/`)
- Remote managed settings (`src/services/remoteManagedSettings/`)

## Migration History

`migrateBypassPermissionsAcceptedToSettings.ts` shows that bypass acceptance was originally stored in a different location and was migrated to the settings system. This migration also gave enterprise admins the ability to revoke bypass acceptance remotely.

## Implementation Flow

```
User invokes tool call
  |
  v
useCanUseTool() hook
  |
  +-- permissionMode === 'bypassPermissions'?
  |     |
  |     +-- isBypassPermissionsModeDisabled()? -> Block, fall back to default
  |     |
  |     +-- Tool-specific safety checks still run:
  |           - BashTool: destructiveCommandWarning, pathValidation, gitSafety
  |           - FileEdit: path validation
  |           - FileWrite: path validation
  |           |
  |           +-- Safety check passes -> Execute immediately (no UI prompt)
  |           +-- Safety check fails -> Warning injected into result
  |
  +-- permissionMode === 'default'? -> Show PermissionRequest UI
  +-- permissionMode === 'plan'? -> Block if write tool
```

## Security Implications

Bypass mode is designed for:
- **CI/CD pipelines** where no human is present to approve
- **Trusted automation** where Claude operates on known-safe codebases
- **Power users** who want maximum speed and accept the risk

The remaining safety layers ensure that even in bypass mode:
- Claude can't accidentally destroy the git repo (force-push guards)
- Claude can't escape the project directory (path validation)
- Claude gets warned about destructive operations (warning injection)
- Enterprise policies are respected (MDM override)

The trust boundary is clear: **user approval is optional, programmatic safety is not**.
