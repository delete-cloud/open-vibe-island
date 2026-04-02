# open-vibe-island

> 我不想在自己的电脑上运行一个闭源、付费的软件来监视我所有的生产过程。<br>
> 所以我 build 了这个开源的版本。<br>
>
> To all vibe coders: 我们自己构建自己的产品。

`open-vibe-island` is an open-source macOS notch and top-bar companion for AI coding agents. It monitors local agent sessions, surfaces permission requests and questions, and helps you jump back into the right terminal or editor context without breaking flow.

GitHub: <https://github.com/Octane0411/open-vibe-island>

## What It Does

The app is being built as a native Swift control surface for local agent workflows on macOS. The current direction is simple:

- stay local-first
- keep the terminal workflow intact
- make approvals, questions, and session switching visible
- bring you back to the right terminal context fast

## Status

Initial native scaffolding is in place. The repository currently contains a buildable macOS Swift package with:

- `VibeIslandCore` for shared event and session state logic
- `VibeIslandApp` for the SwiftUI and AppKit shell
- `VibeIslandHooks` for Codex hook ingestion over stdin/stdout
- a local Unix-socket bridge between the app and external hook processes
- core tests for session state transitions

The repository name is now `open-vibe-island`, while the Swift modules still use the `VibeIsland` prefix for the moment.

## Product Direction

- Native macOS app built with SwiftUI and AppKit where needed
- Local-first communication over Unix sockets or equivalent IPC
- Support multiple coding agents over time, starting with one narrow integration
- Focus on interaction, not just passive monitoring

## Initial Milestones

1. `v0.1` Single-agent MVP with real Codex hook monitoring and overlay UI
2. `v0.2` Approval flow hardening, terminal jump, and install automation
3. `v0.3` Terminal jump, multi-session state, and external display behavior
4. `v0.4` Multi-agent adapters and install/setup automation

## Getting Started

```bash
swift test
swift build
open Package.swift
```

Open the package in Xcode to run the macOS app target. The app starts a local bridge and waits for Codex hook events. Use `Restart Demo` in the UI if you want the older mock timeline back.

The control center also shows live Codex hook install status from `~/.codex`, and can install or uninstall the managed hook entries directly if it can locate a local `VibeIslandHooks` executable.

## Codex Hook MVP

Enable the official Codex hook feature flag once:

```toml
[features]
codex_hooks = true
```

Build the helper once:

```bash
swift build -c release --product VibeIslandHooks
```

Then let the setup tool install or remove the managed Codex hook entries:

```bash
swift run VibeIslandSetup install --hooks-binary "$(pwd)/.build/release/VibeIslandHooks"
swift run VibeIslandSetup status --hooks-binary "$(pwd)/.build/release/VibeIslandHooks"
swift run VibeIslandSetup uninstall
```

The installer:

- enables `[features].codex_hooks = true` if needed
- merges Vibe Island hook handlers into `~/.codex/hooks.json` without deleting unrelated hooks
- writes a small manifest so uninstall can remove only what Vibe Island added
- creates timestamped backups before rewriting `config.toml` or `hooks.json`

If you want to manage the files yourself, a minimal `~/.codex/hooks.json` shape looks like:

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "startup|resume",
        "hooks": [
          {
            "type": "command",
            "command": "/Users/you/path/to/open-vibe-island/.build/release/VibeIslandHooks"
          }
        ]
      }
    ],
    "UserPromptSubmit": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "/Users/you/path/to/open-vibe-island/.build/release/VibeIslandHooks"
          }
        ]
      }
    ],
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "/Users/you/path/to/open-vibe-island/.build/release/VibeIslandHooks"
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "/Users/you/path/to/open-vibe-island/.build/release/VibeIslandHooks"
          }
        ]
      }
    ],
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "/Users/you/path/to/open-vibe-island/.build/release/VibeIslandHooks"
          }
        ]
      }
    ]
  }
}
```

The helper reads the Codex hook payload from `stdin`, forwards it to the app bridge over a Unix socket in `/tmp`, and only writes JSON to `stdout` when the island explicitly denies a `PreToolUse` Bash command. If the app or bridge is unavailable, the hook fails open and Codex keeps running unchanged.

## Jump Back

Codex hook ingestion captures terminal hints from the hook process environment, such as `TERM_PROGRAM`, `ITERM_SESSION_ID`, and Ghostty-specific variables. The island uses those hints to power a best-effort `Jump` action:

- store terminal-specific locators such as iTerm session id, Ghostty terminal id, and Terminal tty when available
- focus the matching iTerm session, Ghostty terminal, or Terminal tab before falling back
- reopen the recorded working directory in that terminal as the final fallback
- keep the existing CLI workflow unchanged even when exact pane restoration is not yet available

## Repository Layout

- `Package.swift` Swift package entry point for the app and shared core module
- `Sources/VibeIslandCore` Shared models, events, mock scenario, session state reducer, wire protocol, hook models, installer logic, and bridge server
- `Sources/VibeIslandHooks` Hook executable for Codex
- `Sources/VibeIslandSetup` Installer CLI for Codex feature and hook setup
- `Sources/VibeIslandApp` SwiftUI app shell, menu bar entry, and overlay panel controller
- `Tests/VibeIslandCoreTests` Core logic tests
- `docs/product.md` Product scope, MVP boundary, and roadmap
- `docs/architecture.md` System shape, event flow, and engineering decisions

## Principles

- Keep the app local-first. No server dependency for core behavior.
- Build narrow slices end to end before adding more integrations.
- Prefer native platform APIs over cross-platform abstractions.
- Treat hooks, IPC, and focus-switching behavior as first-class engineering concerns.
- Keep the terminal entrypoint unchanged for users. The app should attach to Codex, not replace it.

## Next Step

Polish the Codex hook adapter, harden install flow, and keep improving terminal jump behavior.
