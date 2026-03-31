# MCP Client Implementation

> **Key files:** `src/services/mcp/`, `src/tools/MCPTool/`, `src/utils/mcp/`, `src/entrypoints/mcp.ts`

## Overview

Claude Code implements a full Model Context Protocol (MCP) client, allowing it to connect to external MCP servers that provide additional tools, resources, and capabilities. The implementation supports multiple transport types, OAuth authentication, configuration scopes, and dynamic server discovery.

## Transport Types

The MCP client supports five transport types (`src/services/mcp/types.ts`):

| Transport | Enum | Description |
|-----------|------|-------------|
| **stdio** | `'stdio'` | Standard I/O (spawns subprocess, communicates via stdin/stdout) |
| **SSE** | `'sse'` | Server-Sent Events (HTTP long-polling) |
| **HTTP** | `'http'` | Standard HTTP request/response |
| **WebSocket** | `'ws'` | WebSocket bidirectional communication |
| **SDK** | `'sdk'` | Direct SDK integration |

Additionally, `'sse-ide'` is a variant for IDE-hosted MCP servers.

### Transport Configuration

```typescript
// stdio transport
{
  type: 'stdio',
  command: 'node',
  args: ['./server.js'],
  env: { API_KEY: '...' }
}

// HTTP/SSE transport
{
  type: 'http',
  url: 'https://mcp-server.example.com/api'
}

// WebSocket transport
{
  type: 'ws',
  url: 'wss://mcp-server.example.com/ws'
}
```

## Configuration Scopes

MCP servers can be configured at multiple levels (`ConfigScope`):

| Scope | Location | Priority |
|-------|----------|----------|
| `local` | `.claude/settings.local.json` | Highest |
| `project` | `.claude/settings.json` | High |
| `user` | `~/.claude/settings.json` | Medium |
| `enterprise` | MDM/managed settings | Low |
| `dynamic` | Runtime-added servers | Variable |
| `claudeai` | Claude.ai hosted servers | Variable |
| `managed` | Remote managed settings | Low |

Higher-priority scopes override lower ones for the same server name.

## Server Connection Lifecycle

### Discovery

1. **Config loading**: Read MCP server configs from all scopes
2. **Official registry**: Prefetch official MCP server URLs (`prefetchOfficialMcpUrls()`)
3. **Plugin servers**: Discover MCP servers provided by plugins
4. **CLI flags**: `--mcp-config` flag can add servers at startup

### Connection

```
1. Parse server config (type, url/command, args, env)
2. Create MCP Client instance (@modelcontextprotocol/sdk)
3. Establish transport connection
4. Perform capability negotiation
5. Discover server tools, resources, prompts
6. Register tools in Claude Code's tool registry
7. Status: "connected"
```

### Reconnection

Servers can disconnect and reconnect during a session:
- Connection state tracked per-server
- MCP instructions in system prompt use `DANGEROUS_uncachedSystemPromptSection` to stay current
- `isMcpInstructionsDeltaEnabled()` controls whether deltas or full re-injection is used

## Tool Proxying

### MCPTool (`src/tools/MCPTool/MCPTool.ts`)

MCP tools are proxied as native Claude Code tools:

```
Claude calls mcp__servername__toolname({ ... })
    |
    v
MCPTool routes to correct MCP server connection
    |
    v
MCP Client sends tool call to server
    |
    v
Server executes and returns result
    |
    v
MCPTool formats result for Claude
```

Tool naming convention: `mcp__<serverName>__<toolName>`

### Tool Schema

MCP tool schemas are converted to Claude Code's `ToolInputJSONSchema` format:
- JSON Schema properties preserved
- Required fields mapped
- Descriptions included in system prompt

### Collapse Classification (`classifyForCollapse.ts`)

MCP tool results are classified for UI collapse behavior:
- Large results auto-collapsed in the UI
- Small results shown inline
- Configurable per-tool

## Resource Handling

### ReadMcpResourceTool (`src/tools/ReadMcpResourceTool/`)

Reads resources exposed by MCP servers:
- URI-based resource access
- Content type negotiation
- Binary and text resource support

### ListMcpResourcesTool (`src/tools/ListMcpResourcesTool/`)

Lists available resources from MCP servers:
- Server-filtered listing
- Resource metadata (name, description, URI, MIME type)
- Used for discovery before reading

## OAuth Authentication

### McpAuthTool (`src/tools/McpAuthTool/McpAuthTool.ts`)

MCP servers can require OAuth authentication:

1. Server returns auth challenge
2. `McpAuthTool` initiates OAuth flow
3. User directed to browser for authorization
4. Callback received with auth code
5. Token exchanged and stored
6. Server connection re-established with credentials

OAuth configuration from `src/constants/oauth.ts` provides:
- Client IDs
- Auth/token endpoints
- Redirect URIs
- Scope definitions

## MCP Server as Entrypoint

`src/entrypoints/mcp.ts` allows Claude Code itself to act as an MCP server:

```bash
claude --mcp-server  # Start as MCP server
```

This mode:
- Exposes Claude Code's tools as MCP tools to other clients
- Enables embedding Claude Code in MCP-aware applications
- Uses stdio transport by default

## Server Instructions

MCP servers can provide instructions that get injected into the system prompt:

```typescript
// In system prompt assembly (prompts.ts):
getMcpInstructionsSection(mcpClients)

// Output format:
// # MCP Server Instructions
// ## server-name
// <server instructions here>
```

Only connected servers with non-empty instructions are included.

## WebSocket Transport (`src/utils/mcpWebSocketTransport.ts`)

Custom WebSocket transport implementation:
- Bidirectional message passing
- Automatic reconnection
- Heartbeat/keepalive
- Error handling and retry

## Claude-in-Chrome MCP Server

`src/utils/claudeInChrome/mcpServer.ts` implements a special MCP server for the Chrome extension:
- Launched via `--claude-in-chrome-mcp` flag
- Provides browser context tools to Claude Code
- Enables web interaction capabilities

## Configuration Commands

### `/mcp` Command (`src/commands/mcp/`)

Manage MCP servers:
- List connected/configured servers
- Add new servers
- Remove servers
- Restart failed connections
- View server capabilities

## Key Implementation Details

### Connection State

```typescript
type MCPServerConnection =
  | { type: 'connected', client: Client, capabilities: ServerCapabilities, ... }
  | { type: 'connecting', ... }
  | { type: 'disconnected', error?: string, ... }
  | { type: 'failed', error: string, ... }
```

### Server Approval

`src/services/mcpServerApproval.tsx` handles user approval for project-scoped MCP servers:
- First-time connections require approval
- Approval persisted per-project
- Enterprise policies can pre-approve servers

### Plugin Integration

MCP servers can be provided by plugins:
- `src/utils/plugins/mcpPluginIntegration.ts` bridges plugin MCP servers
- `src/utils/plugins/mcpbHandler.ts` handles MCP-over-bridge protocol
- Plugin servers get the same tool proxying as standalone servers

### Date/Time Parsing

`src/utils/mcp/dateTimeParser.ts` provides date/time parsing utilities for MCP tools that work with temporal data.
