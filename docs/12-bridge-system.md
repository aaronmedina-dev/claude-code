# Bridge System

> **Key files:** `src/bridge/`, `src/remote/`, `src/hooks/useReplBridge.tsx`

## Overview

The bridge system enables communication between the Claude Code CLI process and external hosts -- the Desktop app (Mac/Windows), the Web app (claude.ai/code), and IDE extensions (VS Code, JetBrains). It's the infrastructure that allows Claude Code to run as a backend while different frontends provide the UI.

## Architecture

```
Desktop App / Web App / IDE Extension
        |
        |  WebSocket / Bridge Protocol
        |
        v
Bridge Layer (src/bridge/)
        |
        v
REPL Bridge (replBridge.ts)
        |
        v
Claude Code Core (REPL.tsx, tools, etc.)
```

## Key Components

### Bridge Main (`bridgeMain.ts`)

The entry point for bridge mode:
- Initializes bridge transport
- Sets up message routing
- Manages session lifecycle
- Handles attachment processing

### REPL Bridge (`replBridge.ts`, `initReplBridge.ts`)

Bidirectional communication channel between the REPL and external hosts:

```typescript
// Bridge provides:
- Send messages to frontend
- Receive messages from frontend
- Handle permission delegation
- Stream tool outputs
- Report session state changes
```

### Bridge API (`bridgeApi.ts`)

HTTP/WebSocket API surface:
- Session creation endpoints
- Message send/receive endpoints
- File upload/download
- Status queries

### Bridge Messaging (`bridgeMessaging.ts`)

Message protocol for bridge communication:
- Structured message format
- Message serialization/deserialization
- Queue management for reliable delivery

### Bridge Config (`bridgeConfig.ts`, `envLessBridgeConfig.ts`)

Configuration for bridge connections:
- Transport selection (WebSocket, polling)
- Endpoint URLs
- Timeouts and retry settings
- Environment-less configuration for headless operation

## Session Management

### Session Runner (`sessionRunner.ts`)

Manages the lifecycle of bridge sessions:

```
1. Frontend requests new session
2. Bridge creates session context
3. CLI process spawned/attached
4. Messages routed bidirectionally
5. Session state tracked (active, idle, completed)
6. Cleanup on disconnect
```

### Session Creation (`createSession.ts`)

Creates new session contexts:
- Assigns session ID
- Sets up working directory
- Initializes tool configuration
- Establishes permission context

### Code Session API (`codeSessionApi.ts`)

API for managing code sessions:
- Create, resume, list sessions
- Session metadata management
- Cross-device session transfer

## Authentication

### JWT Utils (`jwtUtils.ts`)

JSON Web Token handling for secure bridge communication:
- Token generation and validation
- Session authentication
- Token refresh logic

### Trusted Device (`trustedDevice.ts`)

Device trust management:
- Device registration and verification
- Trust token storage
- Cross-session device identity

### Work Secret (`workSecret.ts`)

Shared secret for bridge authentication:
- Generated per-session
- Used to verify bridge messages
- Prevents unauthorized bridge connections

## Remote Bridge

### Remote Bridge Core (`remoteBridgeCore.ts`)

For claude.ai/code web sessions:
- WebSocket connection to Anthropic's servers
- Session state synchronization
- Remote tool execution
- Permission bridging between local and remote

### Session ID Compatibility (`sessionIdCompat.ts`)

Handles session ID format translation:
- `cse_*` format (server endpoints)
- `session_*` format (frontend routing)
- Transparent conversion between formats

## Remote Session Management (`src/remote/`)

### RemoteSessionManager (`RemoteSessionManager.ts`)

Manages connections to remote Claude Code sessions:
- Session discovery and connection
- State synchronization
- Error recovery and reconnection

### SessionsWebSocket (`SessionsWebSocket.ts`)

WebSocket client for remote session communication:
- Persistent connection management
- Message framing
- Heartbeat/keepalive
- Automatic reconnection

### Remote Permission Bridge (`remotePermissionBridge.ts`)

Bridges permission requests between local and remote:
- Remote session needs local file access -> request bridged to local
- Local approval sent back to remote
- Ensures user always sees permission requests

## Bridge UI (`bridgeUI.ts`)

UI integration for bridge mode:
- Status indicators (connected, disconnected, syncing)
- Connection error display
- Session switching UI

## Poll Configuration

### Poll Config (`pollConfig.ts`, `pollConfigDefaults.ts`)

When WebSocket isn't available, falls back to polling:
- Configurable poll intervals
- Backoff strategy for idle sessions
- Default polling parameters

## Inbound Handling

### Inbound Messages (`inboundMessages.ts`)

Processing messages received from frontend:
- Message validation
- Type routing
- Queue integration

### Inbound Attachments (`inboundAttachments.ts`)

Handling file attachments from frontend:
- Image processing
- Document handling
- Size validation
- Storage management

## Bridge Status (`bridgeStatusUtil.ts`)

Utilities for tracking bridge connection status:
- Connection state machine
- Status event emission
- Health monitoring

## Capacity Wake (`capacityWake.ts`)

Handles wake-from-sleep scenarios:
- Detects capacity changes
- Re-establishes connections
- Resynchronizes state

## Flush Gate (`flushGate.ts`)

Ensures all pending messages are delivered:
- Queues messages during connection gaps
- Flushes queue on reconnection
- Prevents message loss

## Debug Utilities

### Bridge Debug (`bridgeDebug.ts`, `debugUtils.ts`)

Debugging tools for bridge connections:
- Message logging
- Connection state inspection
- Timing analysis

## Bridge Pointer (`bridgePointer.ts`)

Manages references to the active bridge instance:
- Singleton pattern for bridge access
- Context-aware bridge selection
- Used by components that need to send bridge messages

## Bridge Permission Callbacks (`bridgePermissionCallbacks.ts`)

Handles permission requests that flow through the bridge:
- Tool permission requests forwarded to frontend
- Frontend approval/denial forwarded back
- Timeout handling for unresponsive frontends

## Bridge Enabled Check (`bridgeEnabled.ts`)

Determines if bridge mode is active:
- Feature flag check (`BRIDGE_MODE`)
- Environment detection
- Configuration validation

## REPL Bridge Hook (`useReplBridge.tsx`)

React hook for REPL-bridge integration:
- Provides bridge state to React components
- Handles bridge message events
- Manages connection lifecycle within React

## How Desktop/Web Apps Use the Bridge

### Desktop App Flow

```
1. Desktop app launches Claude Code CLI as child process
2. Bridge established over stdio/pipe
3. Desktop renders UI, CLI runs agent logic
4. User input -> Desktop -> Bridge -> CLI -> Model -> CLI -> Bridge -> Desktop
5. Tool calls rendered in Desktop's UI
6. Permission requests shown as Desktop dialogs
```

### Web App Flow (claude.ai/code)

```
1. User opens claude.ai/code
2. Frontend connects to Anthropic's bridge relay
3. Relay connects to user's Claude Code Remote instance
4. Messages flow: Frontend <-> Relay <-> Remote CLI
5. Session state synchronized via WebSocket
6. Files accessible via remote bridge
```

### IDE Extension Flow

```
1. VS Code/JetBrains extension starts Claude Code
2. Bridge established over extension host protocol
3. IDE provides file context and diagnostics
4. Claude Code provides AI capabilities
5. Tool results rendered in IDE panels
```
