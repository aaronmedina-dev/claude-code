# Custom Ink Fork (React Terminal Renderer)

> **Key files:** `src/ink/`

## Overview

Claude Code's entire UI is rendered via a heavily customized fork of [Ink](https://github.com/vadimdemedes/ink), a React renderer for command-line interfaces. This fork goes far beyond the original Ink, adding a custom event system, Yoga-based flexbox layout, text selection, bidirectional text, hyperlinks, search highlighting, and more.

## Architecture

### Rendering Pipeline

```
React Components (Box, Text, Button, ScrollBox, ...)
        |
        v
React Reconciler (reconciler.ts)
        |
        v
Virtual Terminal DOM (dom.ts)
        |
        v
Layout Engine (layout/ - Yoga flexbox)
        |
        v
Render to Output Buffer (render-node-to-output.ts)
        |
        v
Render to Screen (render-to-screen.ts)
        |
        v
Terminal (stdout)
```

### React Reconciler (`reconciler.ts`)

A custom React reconciler (using `react-reconciler`) that targets the virtual terminal DOM:

- Creates/updates/removes virtual DOM nodes
- Handles text content insertion
- Manages component mounting/unmounting lifecycle
- Connects React's state management to terminal rendering

This is the same pattern used by React Native (targets mobile views) and React Three Fiber (targets WebGL) -- but targeting terminal output.

### Virtual Terminal DOM (`dom.ts`)

A tree of virtual nodes representing the terminal layout:

- **Element nodes**: Correspond to `<Box>`, `<Text>`, etc.
- **Text nodes**: Raw text content
- Each node has: style properties, children, parent reference, Yoga layout node
- Supports: dimensions, padding, margin, borders, flex properties

### Layout Engine (`layout/`)

Uses **Yoga** (Facebook's flexbox implementation) for terminal layout:

- `yoga.ts` -- Yoga layout engine bindings
- `engine.ts` -- Layout computation and measurement
- `node.ts` -- Layout node management
- `geometry.ts` -- Position and size calculations

Supports CSS flexbox properties adapted for terminal:
- `flexDirection`, `flexWrap`, `flexGrow`, `flexShrink`
- `alignItems`, `justifyContent`, `alignSelf`
- `width`, `height`, `minWidth`, `maxWidth`
- `padding`, `margin`
- `position: absolute`
- `overflow: hidden`

### Output Rendering (`render-node-to-output.ts`)

Walks the virtual DOM tree and generates an output buffer:

1. Traverses nodes depth-first
2. Applies styles (colors, bold, underline, etc.)
3. Handles text wrapping (`wrap-text.ts`)
4. Measures text width (`measure-text.ts`, `stringWidth.ts`)
5. Renders borders (`render-border.ts`)
6. Produces a 2D character grid

### Screen Output (`render-to-screen.ts`)

Converts the output buffer to ANSI escape sequences for the terminal:
- Diff-based updates (only redraws changed regions)
- Terminal cursor management
- Alternate screen support
- Screen clearing strategies

## Components

### Core Components (`src/ink/components/`)

| Component | File | Description |
|-----------|------|-------------|
| `Box` | `Box.tsx` | Flexbox container (like `<div>`) |
| `Text` | `Text.tsx` | Styled text content (like `<span>`) |
| `Button` | `Button.tsx` | Clickable button with focus support |
| `ScrollBox` | `ScrollBox.tsx` | Scrollable container with viewport |
| `Link` | `Link.tsx` | Clickable hyperlink (OSC 8 support) |
| `Spacer` | `Spacer.tsx` | Flexible space filler |
| `Newline` | `Newline.tsx` | Line break |
| `RawAnsi` | `RawAnsi.tsx` | Raw ANSI escape sequence output |
| `AlternateScreen` | `AlternateScreen.tsx` | Alternate terminal screen (fullscreen UI) |
| `NoSelect` | `NoSelect.tsx` | Content excluded from text selection |
| `ErrorOverview` | `ErrorOverview.tsx` | Error display component |

### Context Providers

| Provider | File | Purpose |
|----------|------|---------|
| `AppContext` | `AppContext.ts` | Application-level context |
| `StdinContext` | `StdinContext.ts` | Standard input access |
| `TerminalSizeContext` | `TerminalSizeContext.tsx` | Terminal dimensions |
| `TerminalFocusContext` | `TerminalFocusContext.tsx` | Terminal focus state |
| `ClockContext` | `ClockContext.tsx` | Clock/timer context |
| `CursorDeclarationContext` | `CursorDeclarationContext.ts` | Cursor position management |

## Event System (`src/ink/events/`)

A full DOM-like event system for terminal interactions:

### Event Types

| Event Type | File | Description |
|------------|------|-------------|
| `InputEvent` | `input-event.ts` | Raw keyboard input |
| `KeyboardEvent` | `keyboard-event.ts` | Processed keyboard events |
| `ClickEvent` | `click-event.ts` | Mouse click events |
| `FocusEvent` | `focus-event.ts` | Focus/blur on elements |
| `TerminalFocusEvent` | `terminal-focus-event.ts` | Terminal window focus changes |
| `TerminalEvent` | `terminal-event.ts` | Terminal resize, etc. |

### Event Dispatcher (`dispatcher.ts`)

Routes events through the virtual DOM tree:
- Capture phase (top-down)
- Target phase
- Bubble phase (bottom-up)
- `stopPropagation()` and `preventDefault()` support

### Event Emitter (`emitter.ts`)

Base event emitter for terminal I/O events:
- Keyboard input processing
- Mouse event handling
- Terminal state change events

## Hooks (`src/ink/hooks/`)

React hooks for terminal interaction:

| Hook | File | Purpose |
|------|------|---------|
| `useInput` | `use-input.ts` | Handle keyboard input |
| `useStdin` | `use-stdin.ts` | Access raw stdin |
| `useTerminalViewport` | `use-terminal-viewport.ts` | Viewport dimensions and scroll |
| `useSelection` | `use-selection.ts` | Text selection state |
| `useAnimationFrame` | `use-animation-frame.ts` | Animation timing |
| `useInterval` | `use-interval.ts` | Periodic updates |
| `useDeclaredCursor` | `use-declared-cursor.ts` | Cursor position declaration |
| `useTerminalTitle` | `use-terminal-title.ts` | Set terminal title |
| `useTerminalFocus` | `use-terminal-focus.ts` | Terminal focus state |
| `useSearchHighlight` | `use-search-highlight.ts` | Search result highlighting |
| `useTabStatus` | `use-tab-status.ts` | Tab/window status |

## Terminal I/O (`src/ink/termio/`)

Low-level terminal escape sequence parsing and generation:

| Module | File | Purpose |
|--------|------|---------|
| `parser.ts` | `parser.ts` | Parse incoming terminal byte streams |
| `tokenize.ts` | `tokenize.ts` | Tokenize ANSI sequences |
| `ansi.ts` | `ansi.ts` | ANSI escape code handling |
| `csi.ts` | `csi.ts` | Control Sequence Introducer parsing (cursor, erase) |
| `sgr.ts` | `sgr.ts` | Select Graphic Rendition (colors, bold, underline) |
| `osc.ts` | `osc.ts` | Operating System Command (hyperlinks, titles) |
| `esc.ts` | `esc.ts` | Simple escape sequences |
| `dec.ts` | `dec.ts` | DEC private mode sequences |
| `types.ts` | `types.ts` | Type definitions for parsed tokens |

## Special Features

### Text Selection (`selection.ts`)

Full text selection support in the terminal:
- Click-and-drag selection
- Shift+arrow key selection
- Copy-to-clipboard integration
- Selection highlighting

### Bidirectional Text (`bidi.ts`)

Right-to-left (RTL) text support:
- Detects RTL characters
- Applies Unicode BiDi algorithm
- Handles mixed LTR/RTL content

### Hyperlinks (`supports-hyperlinks.ts`, `Link.tsx`)

OSC 8 hyperlink support:
- Detects terminal hyperlink support
- Renders clickable links in supported terminals
- Falls back to plain text in unsupported terminals

### Search Highlighting (`searchHighlight.ts`)

In-terminal search with match highlighting:
- Regex-based search
- Match count display
- Navigate between matches (n/N keys)
- Highlights rendered inline in terminal output

### Focus Management (`focus.ts`)

Tab-based focus navigation between interactive elements:
- Focus ring tracking
- Tab/Shift+Tab navigation
- Auto-focus on mount
- Focus context for nested components

### Hit Testing (`hit-test.ts`)

Maps terminal coordinates to virtual DOM elements:
- Used for mouse click routing
- Determines which element is at a given position
- Walks the layout tree to find the target

## Rendering Optimizations

### Output Optimizer (`optimizer.ts`)

Reduces the number of ANSI escape sequences:
- Merges adjacent cells with same styling
- Skips unchanged regions
- Batches cursor movements

### Line Width Cache (`line-width-cache.ts`)

Caches line width calculations:
- Avoids recalculating Unicode character widths
- Handles wide characters (CJK, emoji)
- Invalidates on content change

### Node Cache (`node-cache.ts`)

Caches virtual DOM node computations:
- Layout results
- Style calculations
- Reduces per-frame computation

### Log Update (`log-update.ts`)

Efficient terminal output updating:
- Only writes changed lines
- Uses cursor positioning to skip unchanged regions
- Handles terminal resize gracefully

## Key Differences from Upstream Ink

This fork differs substantially from the original Ink library:

1. **Custom event system** -- Ink has basic input handling; this has full DOM-like events
2. **Yoga layout** -- Original Ink uses a simpler layout; this uses full Yoga flexbox
3. **Text selection** -- Not in original Ink
4. **Mouse support** -- Click events, hit testing
5. **Hyperlinks** -- OSC 8 support
6. **BiDi text** -- Right-to-left text support
7. **Search highlighting** -- Built-in search
8. **Terminal I/O parser** -- Full ANSI/CSI/SGR/OSC/DEC parsing
9. **ScrollBox** -- Scrollable containers with viewport management
10. **Focus system** -- Tab navigation between interactive elements
