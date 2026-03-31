# Feature Flags & Hidden Commands

> **Key files:** `src/commands.ts`, `src/entrypoints/cli.tsx`, `src/services/analytics/growthbook.ts`

## Overview

Claude Code uses two feature flag systems: compile-time `bun:bundle` feature gates for dead code elimination (DCE), and runtime GrowthBook feature flags for A/B testing. Additionally, many slash commands are hidden from the help menu or gated behind internal-only checks.

## Compile-Time Feature Flags (`bun:bundle`)

The `feature()` API from `bun:bundle` enables conditional compilation. When a feature is disabled, the Bun bundler eliminates the gated code entirely from the output binary.

### All Known Feature Flags

| Flag | What It Gates | Internal Only? |
|------|---------------|----------------|
| `COORDINATOR_MODE` | Multi-agent coordinator system (`src/coordinator/coordinatorMode.ts`) | Yes |
| `KAIROS` | Assistant/Kairos mode -- persistent AI assistant mode | Yes |
| `KAIROS_BRIEF` | Brief command without full Kairos | Yes |
| `VOICE_MODE` | Voice input/output mode (`src/voice/`, `src/context/voice.tsx`) | Yes |
| `BRIDGE_MODE` | Desktop/Web bridge communication (`src/bridge/`) | Yes |
| `DAEMON` | Remote control server mode | Yes |
| `PROACTIVE` | Proactive/autonomous work mode | Yes |
| `ABLATION_BASELINE` | A/B testing ablation baseline (disables all optimizations) | Yes |
| `DUMP_SYSTEM_PROMPT` | `--dump-system-prompt` CLI flag for prompt extraction | Yes |
| `FORK_SUBAGENT` | Fork-based subagent spawning | Varies |
| `BUILTIN_EXPLORE_PLAN_AGENTS` | Built-in Explore and Plan agent types | Varies |
| `VERIFICATION_AGENT` | Verification agent for code review | Yes |
| `CACHED_MICROCOMPACT` | API-level cached microcompaction | Varies |
| `TOKEN_BUDGET` | User-specified token budget support | Varies |
| `EXPERIMENTAL_SKILL_SEARCH` | Skill discovery tool | Yes |
| `NATIVE_CLIENT_ATTESTATION` | Bun HTTP attestation support | Yes |

### How DCE Works

```typescript
import { feature } from 'bun:bundle';

// At build time, feature('VOICE_MODE') resolves to true or false
// If false, the entire block (including the require) is eliminated
const VoiceProvider = feature('VOICE_MODE')
  ? require('../context/voice.js').VoiceProvider
  : ({ children }) => children;
```

This means the **external/public npm build** has all internal features stripped out. The leaked source shows them because it's the pre-compilation TypeScript.

### Ablation Baseline

The `ABLATION_BASELINE` flag is particularly interesting -- when enabled, it disables all optimizations to measure their impact:

```typescript
if (feature('ABLATION_BASELINE') && process.env.CLAUDE_CODE_ABLATION_BASELINE) {
  for (const k of [
    'CLAUDE_CODE_SIMPLE',
    'CLAUDE_CODE_DISABLE_THINKING',
    'DISABLE_INTERLEAVED_THINKING',
    'DISABLE_COMPACT',
    'DISABLE_AUTO_COMPACT',
    'CLAUDE_CODE_DISABLE_AUTO_MEMORY',
    'CLAUDE_CODE_DISABLE_BACKGROUND_TASKS'
  ]) {
    process.env[k] ??= '1';
  }
}
```

## GrowthBook Feature Flags (Runtime A/B Tests)

GrowthBook is used for runtime feature flags and A/B testing. Flags are loaded at startup and periodically refreshed.

### Known GrowthBook Flags

Extracted from `logEvent()` calls and `getFeatureValue_CACHED_MAY_BE_STALE()` usage:

| Flag Name | Purpose |
|-----------|---------|
| `tengu_attribution_header` | Attribution header killswitch |
| `tengu_hive_evidence` | Verification agent availability |
| `tengu_amber_stoat` | Explore/Plan agent availability |
| `tengu_cobalt_raccoon` | Reactive-only compaction mode |
| `tengu_prompt_cache_break_detection` | Track cache invalidation |
| `tengu_compact_cache_prefix` | Share cache with compact agent |
| `tengu_sm_compact_config` | Session memory compaction config |
| `tengu_slate_heron` | Time-based microcompact config |
| `tengu_cache_plum_violet` | Cached microcompact enabled |
| `tengu_ant_model_override` | Internal model override |
| `tengu_cicada_nap_ms` | Background refresh throttle |
| `tengu_miraculo_the_bard` | Unknown (gated feature) |

### Naming Pattern

All GrowthBook flags follow the pattern `tengu_<adjective>_<noun>` (e.g., `tengu_amber_stoat`). "Tengu" appears to be Claude Code's internal codename.

### Telemetry Events

All analytics events also use the `tengu_` prefix:
- `tengu_startup_telemetry`
- `tengu_managed_settings_loaded`
- `tengu_compact`
- `tengu_compact_failed`
- `tengu_concurrent_sessions`
- `tengu_agent_flag`
- `tengu_agent_memory_loaded`
- `tengu_timer`
- `tengu_mcp_channel_flags`
- `tengu_structured_output_enabled`
- And many more...

## Hidden Slash Commands

The full command list in `src/commands.ts` reveals many commands not shown in `/help`:

### Internal/Debug Commands

| Command | File | What It Does |
|---------|------|-------------|
| `/bughunter` | `src/commands/bughunter/` | Automated bug hunting mode |
| `/good-claude` | `src/commands/good-claude/` | Positive reinforcement / reward command |
| `/stickers` | `src/commands/stickers/` | Sticker/emoji display (fun feature) |
| `/thinkback` | `src/commands/thinkback/` | Replay thinking/reasoning trace |
| `/thinkback-play` | `src/commands/thinkback-play/` | Animated thinking playback |
| `/teleport` | `src/commands/teleport/` | Session teleportation across machines |
| `/mock-limits` | `src/commands/mock-limits/` | Simulate rate limits for testing |
| `/ant-trace` | `src/commands/ant-trace/` | Internal tracing/debugging |
| `/debug-tool-call` | `src/commands/debug-tool-call/` | Debug individual tool calls |
| `/ctx_viz` | `src/commands/ctx_viz/` | Context visualization |
| `/break-cache` | `src/commands/break-cache/` | Force prompt cache invalidation |
| `/heapdump` | `src/commands/heapdump/` | Generate heap dump for debugging |
| `/passes` | `src/commands/passes/` | Pass/referral management |
| `/rewind` | `src/commands/rewind/` | Rewind conversation to earlier state |
| `/reset-limits` | `src/commands/reset-limits/` | Reset rate limit counters |
| `/backfill-sessions` | `src/commands/backfill-sessions/` | Backfill session data |
| `/btw` | `src/commands/btw/` | "By the way" -- inject side context |
| `/extra-usage` | `src/commands/extra-usage/` | Extra usage/quota management |

### Conditionally Available Commands

| Command | Condition | Purpose |
|---------|-----------|---------|
| `/bridge` | `BRIDGE_MODE` feature | Desktop/Web bridge control |
| `/voice` | `VOICE_MODE` feature | Voice input mode |
| `/mobile` | Always available | Mobile session connection |
| `/agents-platform` | `USER_TYPE === 'ant'` | Internal agent platform |
| `/remote-setup` | Remote capable | Remote session setup |
| `/remote-env` | Remote capable | Remote environment config |

### Standard (Visible) Commands

These are the user-facing commands shown in `/help`:
`/add-dir`, `/agents`, `/clear`, `/color`, `/compact`, `/config`, `/context`, `/copy`, `/cost`, `/desktop`, `/diff`, `/doctor`, `/effort`, `/env`, `/exit`, `/export`, `/fast`, `/feedback`, `/files`, `/help`, `/hooks`, `/ide`, `/init`, `/install-github-app`, `/install-slack-app`, `/issue`, `/keybindings`, `/login`, `/logout`, `/mcp`, `/memory`, `/model`, `/onboarding`, `/output-style`, `/permissions`, `/plan`, `/plugin`, `/pr_comments`, `/release-notes`, `/rename`, `/resume`, `/review`, `/sandbox-toggle`, `/session`, `/share`, `/skills`, `/stats`, `/status`, `/summary`, `/tag`, `/tasks`, `/terminalSetup`, `/theme`, `/upgrade`, `/usage`, `/vim`

## The `USER_TYPE` Gate

Many features check `process.env.USER_TYPE === 'ant'`:

```typescript
// Only available in internal Anthropic builds
const agentsPlatform = process.env.USER_TYPE === 'ant'
  ? require('./commands/agents-platform/index.js').default
  : null
```

Since `USER_TYPE` is set at build time and compared to a string literal, the Bun bundler's DCE eliminates the entire block in external builds (where `USER_TYPE` is `'external'`).

This means:
- **Internal builds** (`USER_TYPE === 'ant'`): All features available
- **External builds** (`USER_TYPE === 'external'`): Internal features stripped out

## Environment Variable Feature Toggles

Beyond feature flags, many behaviors are controlled by environment variables:

```bash
CLAUDE_CODE_SIMPLE=1              # Minimal system prompt
CLAUDE_CODE_DISABLE_THINKING=1    # Disable extended thinking
DISABLE_INTERLEAVED_THINKING=1    # Disable interleaved thinking
DISABLE_COMPACT=1                 # Disable all compaction
DISABLE_AUTO_COMPACT=1            # Disable auto-compaction only
CLAUDE_CODE_DISABLE_AUTO_MEMORY=1 # Disable auto-memory extraction
DISABLE_BACKGROUND_TASKS=1        # Disable background tasks
CLAUDE_CODE_COORDINATOR_MODE=1    # Enable coordinator mode
CLAUDE_CODE_FORK_SUBAGENT=1       # Enable fork subagents
CLAUDE_CODE_REMOTE=true           # Remote session mode
CLAUDE_CODE_ABLATION_BASELINE=1   # Ablation baseline (disable all optimizations)
```
