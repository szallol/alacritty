# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Alacritty is a fast, cross-platform, OpenGL terminal emulator written in Rust. The project is at beta level and focuses on performance through GPU acceleration and a clean architecture that delegates features like tabs and splits to external tools (window managers, tmux).

## Build and Test Commands

### Building

```bash
# Standard release build
cargo build --release

# Build for Wayland only (Linux/BSD)
cargo build --release --no-default-features --features=wayland

# Build for X11 only (Linux/BSD)
cargo build --release --no-default-features --features=x11

# macOS app bundle
make app

# macOS universal binary (x86 + ARM)
rustup target add x86_64-apple-darwin aarch64-apple-darwin
make app-universal
```

The release binary is placed at `target/release/alacritty`.

### Testing

```bash
# Run all tests
cargo test

# Test alacritty_terminal crate without default features
cargo test -p alacritty_terminal --no-default-features

# Run ref tests (requires --ref-test flag on patched binary)
# Binary records terminal state to files in working directory
# Copy to ./tests/ref/NEW_TEST_NAME/ and edit ./tests/ref.rs
```

### Code Quality

```bash
# Format code
cargo fmt

# Run linter
rustup component add clippy
cargo clippy --all-targets
```

### Rust Version

Minimum supported Rust version (MSRV) is specified in workspace `Cargo.toml` (currently 1.85.0). Must always build with MSRV and bumping should be avoided.

## Codebase Architecture

### Workspace Structure

The repository is a Cargo workspace with four crates:

- **`alacritty/`** - Main terminal emulator application with UI, rendering, and event handling
- **`alacritty_terminal/`** - Core terminal emulation library (PTY, VTE parser, grid, term state)
- **`alacritty_config/`** - Configuration parsing utilities
- **`alacritty_config_derive/`** - Procedural macros for config

### Main Application (`alacritty/`)

The main binary orchestrates window management, rendering, and user interaction:

- **`main.rs`** - Entry point. Creates event loop, initializes logging, loads config, spawns IPC socket (Unix), and starts the processor
- **`event.rs`** - The `Processor` struct is the central event handler implementing `ApplicationHandler`. Manages multiple windows, dispatches winit events, handles config reloading
- **`window_context.rs`** - Per-window state including `Display`, `Terminal`, PTY notifier, message buffer, search state. Each window has its own terminal and rendering context
- **`display/`** - Rendering layer coordinating OpenGL rendering, managing the `Window`, and compositing terminal content
  - `mod.rs` - `Display` struct owns the window, renderer, size calculations
  - `window.rs` - Winit window wrapper with platform-specific handling
  - `content.rs` - Determines what to render (cells, cursor, selection, hints)
  - `hint.rs` - URL/path hint detection and rendering
- **`renderer/`** - OpenGL rendering implementation using shaders
  - `mod.rs` - Main renderer coordinating text and rect rendering
  - `text/` - Glyph atlas, text rendering shaders
  - `rects.rs` - Rendering filled rectangles (selection, cursor, etc)
- **`input/`** - Keyboard and mouse input handling, mapping to terminal actions
- **`config/`** - Configuration loading and structures (TOML/YAML). Monitors config files for changes
- **`clipboard.rs`** - Cross-platform clipboard integration
- **`ipc.rs`** - Unix socket IPC for `alacritty msg` subcommand

### Terminal Library (`alacritty_terminal/`)

Platform-agnostic terminal emulation core:

- **`lib.rs`** - Exports `Term`, `Grid`, and other public APIs
- **`term/`** - The `Term<T>` struct is the main terminal state machine
  - `mod.rs` - Processes VTE sequences, maintains grid state, handles scrollback
  - `cell.rs` - Terminal cell representation (character, foreground/background, flags)
  - `search.rs` - Regex search over terminal content
- **`grid/`** - The `Grid<T>` data structure for terminal cell storage
  - `mod.rs` - 2D grid with efficient scrolling (circular buffer)
  - `storage.rs` - Underlying storage with viewport and scrollback
  - `resize.rs` - Grid resizing logic preserving content
  - `row.rs` - Grid row representation
- **`event_loop.rs`** - PTY I/O event loop running in separate thread. Reads from PTY, feeds VTE parser, writes input to PTY
- **`tty/`** - PTY spawning and management
  - `unix.rs` - Unix PTY implementation (fork, exec, signal handling)
  - `windows/` - Windows ConPTY implementation
- **`vte`** - Re-exported VTE parser crate (ANSI/VT100 escape sequence parsing)
- **`selection.rs`** - Terminal selection state and semantic selection
- **`vi_mode.rs`** - Vi-mode cursor and motions
- **`index.rs`** - Grid coordinate types (`Point`, `Line`, `Column`)

### Key Architectural Patterns

**Event Flow:**
1. Winit events arrive at `Processor` (event.rs)
2. `Processor` dispatches to appropriate `WindowContext`
3. Input goes through `input/` module to generate actions
4. Actions modify `Term` state or send bytes via `Notifier` to PTY event loop
5. PTY event loop reads output, feeds VTE parser, updates `Term` grid
6. `Term` notifies UI via `EventListener` trait
7. UI marks window dirty and renders on next frame

**Threading:**
- Main thread: UI event loop, rendering, input processing
- PTY thread: One per terminal, handles I/O event loop (event_loop.rs)
- Config monitor thread: Watches config files for changes

**Rendering Pipeline:**
1. `Display` queries `Term` for dirty state
2. `content.rs` builds `RenderableContent` from grid, cursor, selection
3. Renderer uploads glyphs to atlas and renders via OpenGL shaders
4. Damage tracking minimizes redraws

## Configuration

- Config files: TOML (preferred) or YAML (legacy)
- Locations: `$XDG_CONFIG_HOME/alacritty/alacritty.toml`, `$HOME/.config/alacritty/alacritty.toml`, etc.
- Hot-reload: Config monitor detects changes and reloads
- CLI options override config file settings

## Platform-Specific Notes

**macOS:**
- Requires locale setup (macos/locale.rs)
- Uses Cocoa APIs for some native integrations
- Universal binaries support x86_64 + aarch64
- MACOSX_DEPLOYMENT_TARGET set to 10.12 in Makefile

**Windows:**
- Uses ConPTY backend for PTY
- Requires Windows 10 1809+ for ConPTY support
- Special drop order needed to avoid deadlocks (see main.rs comments)

**Linux/BSD:**
- Supports both X11 and Wayland via winit features
- Can be built with only one backend via `--no-default-features`

## Style Guidelines

- Follow rustfmt configuration in `rustfmt.toml`
- All comments must be fully punctuated with trailing periods
- Adhere to Rust API guidelines: https://rust-lang.github.io/api-guidelines
- Code comments should explain "why" not "what"

## Documentation

- Man pages in `extra/man/*.scd` (scdoc format)
- Changes to `config.rs` must be documented in man pages
- User-facing changes go in `CHANGELOG.md`
- `alacritty_terminal` changes go in `alacritty_terminal/CHANGELOG.md`
- Follow keepachangelog.com format for changelog entries

## Performance Considerations

- Use vtebench (https://github.com/alacritty/vtebench) for throughput testing
- Test on latest stable Rust for benchmarks
- Latency testing with typometer on X11/Windows/macOS
- Ref tests ensure no visual regressions

## Common Patterns

**Adding a config option:**
1. Add field to appropriate config module in `alacritty/src/config/`
2. Add serde derivation with defaults
3. Document in relevant man page (extra/man/alacritty.5.scd or alacritty-bindings.5.scd)
4. Update CHANGELOG.md

**Modifying terminal behavior:**
1. Changes likely in `alacritty_terminal/src/term/mod.rs`
2. May need to handle new VTE sequences
3. Add tests - either Rust `#[test]` or ref tests
4. Ensure ref test fails on unpatched version

**Adding a keybinding action:**
1. Add variant to `Action` enum in `alacritty/src/config/bindings.rs`
2. Implement handling in `impl ActionContext` in `alacritty/src/input/mod.rs`
3. Document in man page extra/man/alacritty-bindings.5.scd

## Release Process

- Versions: Development uses `-dev` suffix (e.g., `0.17.0-dev`)
- Release candidates: `-rcX` suffix
- Releases made on separate `v0.X` branches, not master
- Master always tracks next development version
- `alacritty_terminal` versions stay synchronized with main crate
