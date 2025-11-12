# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

VimTea is a Go library that provides a Vim-like text editor component for terminal applications built with [Bubble Tea](https://github.com/charmbracelet/bubbletea). It implements modal editing with normal, insert, visual, and command modes, along with Vim-style key bindings and commands.

## Development Commands

### Running the Example
```bash
cd example && go run main.go
```

### Testing
```bash
# Run all tests
go test -v

# Run specific test
go test -v -run TestName

# Run tests with coverage
go test -v -cover

# Run tests in a specific file
go test -v -run TestName ./file_test.go
```

Note: The clipboard dependency requires X11 headers on Linux. Tests may fail in headless environments.

### Code Quality
```bash
# Format code
go fmt ./...

# Run Go vet
go vet ./...

# Check imports (requires goimports)
goimports -w .
```

### Dependencies
```bash
# Download dependencies
go mod download

# Tidy dependencies
go mod tidy
```

## Architecture

### Core Components

The codebase follows a clean modular architecture with separation of concerns:

1. **Editor Model** (`model.go`, `vimtea.go`)
   - Central `editorModel` struct that implements the `Editor` interface
   - Implements Bubble Tea's `tea.Model` interface (Init, Update, View)
   - Manages editor state: mode, cursor, viewport, key sequences, count prefixes
   - Handles the event loop and message routing to different modes

2. **Buffer System** (`buffer.go`, `wrapped_buffer.go`)
   - `buffer`: Internal implementation with text content stored as `[]string` (lines)
   - `Buffer`: Public interface exposed to external code and custom bindings
   - `wrappedBuffer`: Adapter that wraps `editorModel` to expose Buffer interface
   - Implements undo/redo via snapshot-based state stacks (`undoStack`, `redoStack`)

3. **Key Binding System** (`bindings.go`)
   - `BindingRegistry`: Maps key sequences to commands for each mode
   - Supports multi-key sequences (e.g., "dd", "gg") via prefix detection
   - `KeyBinding`: Public API struct for users to register custom bindings
   - `internalKeyBinding`: Internal representation used by the registry
   - Numeric prefixes (e.g., "5j") are parsed in `handlePrefixKeypress` in model.go

4. **Command System** (`commands.go`)
   - `CommandRegistry`: Maps command names to command functions
   - Commands are Vim-style colon commands (e.g., ":q", ":w")
   - `CommandFn`: Public function signature for custom commands
   - `Command`: Internal function signature that operates on `*editorModel`
   - Contains all built-in command implementations (movement, editing, visual mode, etc.)

5. **View Layer** (`view.go`)
   - Renders the editor using lipgloss for styling
   - Handles line numbers (absolute and relative)
   - Renders visual selections and syntax highlighting
   - Status bar showing mode, cursor position, and messages

6. **Cursor** (`cursor.go`)
   - Simple `Cursor` struct with Row/Col fields
   - `TextRange` for representing selections
   - Helper functions for cursor positioning

7. **Syntax Highlighting** (`highlight.go`)
   - Uses Chroma library for syntax highlighting
   - Theme-based coloring
   - File extension-based language detection

8. **Styles** (`styles.go`)
   - Default lipgloss styles for UI elements
   - Customizable via `With*Style()` options

### Key Architectural Patterns

**Message-Driven Architecture**: The editor follows Bubble Tea's Elm architecture. All state changes happen via messages:
- `tea.KeyMsg`: User input
- `cursorBlinkMsg`: Cursor animation
- `EditorModeMsg`: Mode changes
- `CommandMsg`: Command execution
- `UndoRedoMsg`: Undo/redo operations
- `statusMessageMsg`: Status updates

**Mode-Based Input Handling**: The `Update` function routes keypresses to mode-specific handlers:
- Normal/Visual modes: Use `handlePrefixKeypress()` for key sequences and count prefixes
- Insert mode: Direct character insertion with registry lookup for special keys
- Command mode: Build command buffer with special handling for enter/escape

**Key Sequence Processing**: Multi-key commands (like "dd", "gg", "diw") are handled via:
1. Keys accumulated in `keySequence` slice
2. Timeout mechanism (750ms) resets incomplete sequences
3. `BindingRegistry.IsPrefix()` checks if more input is expected
4. `BindingRegistry.FindExact()` executes complete sequences

**Count Prefix System**: Numeric prefixes (like "5j", "10k") are handled by:
1. Parsing digits before non-digit keys in `handlePrefixKeypress()`
2. Storing count in `countPrefix` field
3. Commands use `withCountPrefix()` helper to repeat operations
4. Prefix reset after command execution

**Buffer Wrapping**: The `wrappedBuffer` pattern allows:
- Internal state (`editorModel`) to remain encapsulated
- Public `Buffer` interface exposed to custom key bindings and commands
- Automatic undo state management on buffer modifications

**Viewport Scrolling**: The editor uses `charmbracelet/bubbles/viewport` for scrolling:
- `ensureCursorVisible()` adjusts viewport offset to keep cursor in view
- Viewport dimensions updated on window resize
- Status bar height accounted for in viewport calculations

### Testing Strategy

- Unit tests for each component (`*_test.go` files)
- Integration tests in `integration_test.go` test cross-component behavior
- Tests use table-driven design where appropriate
- Editor tests verify mode transitions, key handling, and command execution
- Buffer tests verify text operations and undo/redo correctness

## Extension Points

When adding new features:

1. **Custom Key Bindings**: Use `Editor.AddBinding()` with a `KeyBinding` struct. The `Handler` receives a `Buffer` interface.

2. **Custom Commands**: Use `Editor.AddCommand()` with a `CommandFn`. Commands are invoked with `:commandname` in command mode.

3. **New Built-in Bindings**: Add to `registerBindings()` in commands.go and implement the command function.

4. **New Editor Options**: Add field to `options` struct, create `With*()` option function, and handle in `NewEditor()`.

## Important Implementation Details

- **Cursor Constraints**: In normal mode, cursor cannot be at end of line (Vim behavior). This is enforced in `switchMode()`.

- **Line Storage**: Lines stored as `[]string` with newlines removed. `buffer.text()` joins with "\n".

- **Visual Mode**: Selection bounds calculated in `GetSelectionBoundary()` which handles both character-wise and line-wise visual modes.

- **Desired Column**: The `desiredCol` field preserves horizontal position during vertical movement over shorter lines.

- **Clipboard Integration**: Uses `golang.design/x/clipboard` for system clipboard. Watches clipboard in goroutine if supported.

- **Syntax Highlighting**: Language detected from filename. Theme applied during rendering in view layer.

## Common Pitfalls

- Don't forget to call `m.ensureCursorVisible()` after cursor movement to update viewport
- Count prefix must be reset after command execution (use `defer` or explicit reset)
- Mode transitions require cursor position validation (normal mode can't have cursor at end of line)
- Buffer modifications should go through `wrappedBuffer` to ensure undo state is saved
- Key sequences have a 750ms timeout - ensure related commands are registered with proper prefixes
