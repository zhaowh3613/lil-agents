# CLAUDE.md

## Project Overview

**lil agents** is a macOS dock application that provides animated AI companion characters (Bruce and Jazz) that walk above the user's dock. Characters open a themed popover terminal for chatting with AI CLI tools (Claude Code, OpenAI Codex, GitHub Copilot).

- **Platform:** macOS 14.0+ (Sonoma)
- **Language:** Swift 5.9+ (SwiftUI + AppKit)
- **Build system:** Xcode (no SPM/CocoaPods — pure Xcode project)
- **License:** MIT

## Repository Structure

```
lil-agents/
├── lil-agents.xcodeproj/       # Xcode project configuration
├── LilAgents/                   # All source code
│   ├── LilAgentsApp.swift       # App entry point & AppDelegate
│   ├── LilAgentsController.swift# Main controller, animation loop, dock geometry
│   ├── AgentSession.swift       # Protocol for AI provider sessions
│   ├── ClaudeSession.swift      # Claude Code CLI integration
│   ├── CodexSession.swift       # OpenAI Codex CLI integration
│   ├── CopilotSession.swift     # GitHub Copilot CLI integration
│   ├── ShellEnvironment.swift   # Shell PATH/env resolution
│   ├── TerminalView.swift       # Terminal UI (NSTextView output + NSTextField input)
│   ├── WalkerCharacter.swift    # Character animation, popover, bubbles, sounds
│   ├── CharacterContentView.swift # Character view with alpha-channel hit testing
│   ├── PopoverTheme.swift       # Theme system (Peach, Midnight, Cloud, Moss)
│   ├── Info.plist               # App config (LSUIElement, Sparkle settings)
│   ├── LilAgents.entitlements   # Security entitlements
│   ├── Assets.xcassets/         # App icons and image assets
│   ├── Sounds/                  # Completion ping sound effects (mp3/m4a)
│   └── walk-*.mov               # Character animation videos (HEVC)
├── appcast.xml                  # Sparkle auto-update feed
├── README.md
└── LICENSE
```

## Building & Running

```bash
open lil-agents.xcodeproj   # Then press ⌘R to build and run
```

There is no command-line build setup. All building, signing, and running is done through Xcode.

## Architecture

### Core Abstractions

1. **`AgentSession` protocol** — Defines the interface all AI providers implement:
   - `start()`, `send(message:)`, `terminate()`
   - Callbacks: `onText`, `onError`, `onToolUse`, `onToolResult`, `onSessionReady`, `onTurnComplete`, `onProcessExit`
   - Properties: `isRunning`, `isBusy`, `history`

2. **Provider implementations** (`ClaudeSession`, `CodexSession`, `CopilotSession`):
   - Each launches its CLI tool as a `Process` with NDJSON streaming output
   - Line-buffered parsing of JSON events
   - Binary path caching for performance

3. **`AgentProvider` enum** — Provider selection with `createSession()` factory method, persisted in UserDefaults.

4. **`LilAgentsController`** — Application controller:
   - Manages two `WalkerCharacter` instances (Bruce & Jazz)
   - `CVDisplayLink`-based animation loop at display refresh rate
   - Dock geometry from `com.apple.dock` UserDefaults
   - Multi-screen support with pinning

5. **`WalkerCharacter`** — Largest file (~770 lines). Handles:
   - `AVQueuePlayer` video playback for walk animation
   - Walking state machine with acceleration/deceleration
   - Popover window with terminal
   - Thinking/completion bubbles
   - Sound effects

6. **`TerminalView`** — Programmatic NSView hierarchy (no XIB):
   - Markdown rendering for agent output
   - Streaming text accumulation
   - Input history

7. **`PopoverTheme`** — Four presets (Peach, Midnight, Cloud, Moss) controlling colors, fonts, sizes, and corner radii. Persisted in UserDefaults.

### Message Flow

```
User types in TerminalView input field
  → WalkerCharacter.onSendMessage
    → AgentSession.send(message:)
      → CLI Process (Claude/Codex/Copilot)
        → NDJSON streaming response
          → Session callbacks (onText, onToolUse, onToolResult, onTurnComplete)
            → TerminalView updates + WalkerCharacter bubble/sound feedback
```

## Code Conventions

- **No storyboards/XIBs** — All UI is built programmatically
- **`// MARK: -` sections** used extensively for code organization
- **Guard statements** preferred for early returns
- **`DispatchQueue.main.async`** for all UI updates from background callbacks
- **Weak `self`** in closures to prevent retain cycles
- **`NSColor` with RGB values** for custom theme colors
- **`UserDefaults`** for persistence (theme, provider, onboarding state)
- **`Process` (Foundation)** for launching CLI tools — not shell commands

## External Dependencies

- **Sparkle** (v2+) — Auto-update framework. Configured in Info.plist with EdDSA signing.

No other external dependencies. No package managers.

## Testing

No automated test suite exists. Testing is manual via Xcode's Run (⌘R).

## Release Process

1. Build and archive in Xcode
2. Sign and notarize the app
3. Create zip: `ditto -c -k --keepParent LilAgents.app LilAgents-vX.Y.zip`
4. Sign for Sparkle: `sign_update LilAgents-vX.Y.zip`
5. Update `appcast.xml` with new version entry
6. Upload to GitHub Releases

## Key Notes for AI Assistants

- The app runs as a menu bar agent (`LSUIElement: true`) — it has no main window or dock icon.
- App sandbox is disabled (`com.apple.security.app-sandbox: false`) to allow launching CLI tools.
- Adding a new AI provider means implementing the `AgentSession` protocol and adding a case to `AgentProvider`.
- `WalkerCharacter.swift` is the most complex file; changes there should be careful about the animation state machine.
- All character assets (videos, sounds) are bundled directly in the `LilAgents/` directory.
