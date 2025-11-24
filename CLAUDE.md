# CLAUDE.md - AI Assistant Guide for ArcaneChat TUI

> **Last Updated:** 2025-11-24
> **Version:** Based on arcanechat-tui v2.22+ (commit: 9618940)

This document provides comprehensive guidance for AI assistants working on the ArcaneChat TUI codebase. It covers architecture, conventions, workflows, and key concepts.

---

## Table of Contents

1. [Project Overview](#project-overview)
2. [Architecture & Design Patterns](#architecture--design-patterns)
3. [Codebase Structure](#codebase-structure)
4. [Development Setup](#development-setup)
5. [Coding Conventions](#coding-conventions)
6. [Key Concepts & Patterns](#key-concepts--patterns)
7. [Dependencies](#dependencies)
8. [Testing Guidelines](#testing-guidelines)
9. [Git Workflow](#git-workflow)
10. [Common Tasks](#common-tasks)
11. [Important Files Reference](#important-files-reference)
12. [Gotchas & Important Notes](#gotchas--important-notes)

---

## Project Overview

**ArcaneChat TUI** is a lightweight ncurses-based Delta Chat client for the command line, built with Python and the urwid library.

### Key Facts
- **Language:** Python 3.8+
- **License:** GNU General Public License v3 (GPLv3)
- **UI Framework:** urwid (ncurses-based)
- **Chat Protocol:** Delta Chat (via deltachat2 RPC client)
- **Lines of Code:** ~972 lines across 15 Python modules
- **Package:** arcanechat-tui (published to PyPI)
- **Entry Points:** `arcanechat-tui` or `arcanechat` commands

### Project Status
Currently in **Beta** (Development Status 4). The core functionality works (creating accounts, sending/receiving messages, read receipts), but many advanced features are not yet implemented (see README for feature checklist).

### Recent Changes
- Updated to deltachat-rpc-server 2.22.0
- Removed usage of deprecated `chat.is_protection_broken` API
- Project renamed from "curseddelta" to "arcanechat"

---

## Architecture & Design Patterns

### Primary Architecture Pattern: Event-Driven with Observer Pattern

The application uses an **event-driven architecture** with the observer pattern for loose coupling between components.

```
┌─────────────────────────────────────────────────────────┐
│              DeltaChat Core (RPC Server)                 │
└────────────────────┬────────────────────────────────────┘
                     │ Events (INCOMING_MSG, CHAT_MODIFIED, etc.)
                     ↓
┌─────────────────────────────────────────────────────────┐
│              EventCenter (Signal Translator)             │
│  - Converts DeltaChat events to urwid signals            │
│  - Signals: CHATLIST_CHANGED, MESSAGES_CHANGED, etc.     │
└────────────────────┬────────────────────────────────────┘
                     │ urwid signals
                     ↓
┌─────────────────────────────────────────────────────────┐
│                    Application                           │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │ ChatList     │  │ Conversation │  │  Composer    │  │
│  │ Widget       │  │   Widget     │  │   Widget     │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
└─────────────────────────────────────────────────────────┘
```

### Threading Model
- **Main Thread:** Runs urwid event loop (`loop.run()`)
- **Background Daemon Thread:** Runs DeltaChat RPC client (`client.run_forever()`)
  - Processes DeltaChat events asynchronously
  - Triggers UI updates via urwid signals

### Widget Composition Hierarchy

```
Frame (application.frame)
└── AttrMap (themed background)
    └── Columns (main_columns) [3-column layout]
        ├── Container(ChatListWidget) [weight: 1]
        ├── Separator [width: 1]
        └── Pile (right_side) [weight: 4]
            ├── CardsWidget (expandable) [shows one card at a time]
            │   ├── "welcome" → WelcomeWidget
            │   └── "conversation" → Container(ConversationWidget)
            └── Container(ComposerWidget) [packed height]
```

### Design Patterns Used

1. **Observer Pattern:** Components subscribe to signals and get notified of changes
2. **Container Pattern:** `Container` widget wraps widgets to intercept key events
3. **Lazy Loading Pattern:** `LazyListWalker` creates widgets on-demand with LRU caching
4. **Card/Stack Pattern:** `CardsWidget` shows one widget at a time (welcome vs conversation)
5. **MVC-like Separation:**
   - **Model:** DeltaChat RPC client
   - **View:** Urwid widgets
   - **Controller:** Application class coordinating events

---

## Codebase Structure

```
arcanechat-tui/
├── arcanechat_tui/          # Main source package (972 LOC)
│   ├── __init__.py          # Package init (APP_NAME = "ArcaneChat")
│   ├── __main__.py          # Module execution entry point
│   ├── main.py              # Main entry point (64 LOC)
│   ├── cli.py               # CLI argument parser (149 LOC)
│   ├── application.py       # Main UI coordinator (165 LOC)
│   ├── eventcenter.py       # Event dispatcher (45 LOC)
│   ├── chatlist.py          # Chat list widget (104 LOC)
│   ├── conversation.py      # Message display widget (127 LOC)
│   ├── composer.py          # Message composer widget (86 LOC)
│   ├── cards_widget.py      # Stack layout widget (35 LOC)
│   ├── container.py         # Key event wrapper (31 LOC)
│   ├── welcome_widget.py    # Welcome screen (10 LOC)
│   ├── lazylistwaker.py     # Lazy list loading (33 LOC)
│   ├── logger.py            # Logging config (29 LOC)
│   ├── util.py              # Utility functions (86 LOC)
│   └── _version.py          # Auto-generated by setuptools_scm (gitignored)
│
├── .github/workflows/       # CI/CD configuration
│   └── python-ci.yml        # GitHub Actions workflow
│
├── docs/                    # Documentation
│   └── user-guide.md        # User documentation
│
├── screenshots/             # Screenshot assets for README
├── pyproject.toml           # Project configuration (PEP 518)
├── pylama.ini               # Linting configuration
├── README.md                # Project readme
├── LICENSE                  # GPLv3 license text
└── .gitignore               # Git ignore patterns
```

### Module Responsibilities

| Module | Purpose | Key Classes/Functions |
|--------|---------|----------------------|
| `main.py` | Entry point, RPC setup, theme/keymap config | `main()` |
| `application.py` | Main UI coordinator, signal wiring | `Application` |
| `cli.py` | CLI argument parsing, subcommands | `Cli` |
| `eventcenter.py` | DeltaChat event → urwid signal translation | `EventCenter` |
| `chatlist.py` | Chat list display | `ChatListWidget`, `ChatListItem` |
| `conversation.py` | Message display with read receipts | `ConversationWidget`, `MessageItem` |
| `composer.py` | Message composition area | `ComposerWidget`, `ReadlineEdit2` |
| `cards_widget.py` | Stack widget (shows one at a time) | `CardsWidget` |
| `container.py` | Key event interception wrapper | `Container` |
| `welcome_widget.py` | Welcome screen placeholder | `WelcomeWidget` |
| `lazylistwaker.py` | Lazy widget loading with caching | `LazyListWalker` |
| `logger.py` | Logging configuration | `create_logger()` |
| `util.py` | Utility functions | Various helpers |

---

## Development Setup

### Prerequisites
- Python 3.8 or higher
- `deltachat-rpc-server` binary in PATH (installed via pip or system package)

### Installation for Development

```bash
# Clone the repository
git clone https://github.com/ArcaneChat/arcanechat-tui.git
cd arcanechat-tui

# Install in development mode with dev dependencies
pip install -e '.[dev]'

# Or with full optional dependencies
pip install -e '.[full,dev]'
```

### Development Tools Installed
- **black** - Code formatter (line-length: 100)
- **mypy** - Type checker (ignore_missing_imports: True)
- **isort** - Import sorter (black profile)
- **pylint** - Linter
- **pylama** - Meta-linter (runs all linters)
- **pytest** - Testing framework
- **deltachat-rpc-server** - DeltaChat RPC server

### Running the Application

```bash
# After installation
arcanechat-tui

# Or using the alias
arcanechat

# Or as a module
python -m arcanechat_tui
```

### CLI Subcommands

```bash
# Initialize/configure account
arcanechat-tui init

# Import account from backup
arcanechat-tui import /path/to/backup.tar

# Get/set account configuration
arcanechat-tui config [key] [value]
```

---

## Coding Conventions

### Code Style

**Formatter:** Black (line-length: 100)
```toml
[tool.black]
line-length = 100
```

**Import Sorting:** isort with black profile
```toml
[tool.isort]
profile = "black"
```

### Type Hints
- Use type hints for function signatures
- Import from `typing` module: `Optional`, `Tuple`, `Any`, etc.
- mypy configured with `ignore_missing_imports = True`

**Example:**
```python
def get_subtitle(rpc: Rpc, accid: int, chat: Any) -> str:
    # implementation
```

### Docstrings
- Module-level docstrings required (single line in triple quotes)
- Function/class docstrings are optional (C0116 ignored in pylama)
- When present, use descriptive docstrings

**Example:**
```python
"""Event center"""  # Module docstring

class EventCenter:
    """Event center dispatching Delta Chat core events"""  # Class docstring
```

### Linting Rules

**Pylama Configuration (pylama.ini):**
```ini
[pylama]
linters=mccabe,pyflakes,pylint,isort,mypy
ignore=C0116,R0902,R0913,W1203,W0718,R0903
skip=.*,build/*,tests/*,*/flycheck_*,*/_version.py
```

**Ignored Rules:**
- `C0116` - Missing function/method/class docstring
- `R0902` - Too many instance attributes
- `R0913` - Too many arguments
- `W1203` - Use lazy % formatting in logging functions
- `W0718` - Catching too general exception
- `R0903` - Too few public methods

### Naming Conventions

- **Modules:** lowercase with underscores (`event_center.py`)
- **Classes:** PascalCase (`ChatListWidget`, `EventCenter`)
- **Functions/Methods:** lowercase with underscores (`get_subtitle`, `process_core_event`)
- **Constants:** UPPERCASE with underscores (`CHATLIST_CHANGED`, `APP_NAME`)
- **Private/Internal:** Leading underscore (`_chatlist_keypress`, `_print_title`)

### Import Organization

Use isort with black profile (automatically groups imports):
1. Standard library imports
2. Third-party imports (urwid, deltachat2)
3. Local application imports (relative imports with `.`)

**Example:**
```python
import sys
from threading import Thread
from typing import Optional, Tuple

import urwid
from deltachat2 import Client

from .cards_widget import CardsWidget
from .chatlist import CHAT_SELECTED, ChatListWidget
```

### Error Handling
- Broad exception catching is allowed (W0718 ignored)
- Prefer explicit error handling at boundaries
- Use try/except for runtime errors (e.g., `Path.expanduser()` in `util.py`)

---

## Key Concepts & Patterns

### 1. Signal-Based Communication

**Defining Signals:**
```python
# eventcenter.py
CHATLIST_CHANGED = "chatlist_changed"
MESSAGES_CHANGED = "msgs_changed"

class EventCenter:
    signals = [CHATLIST_CHANGED, MESSAGES_CHANGED]

    def __init__(self) -> None:
        urwid.register_signal(self.__class__, self.signals)
```

**Emitting Signals:**
```python
urwid.emit_signal(self, CHATLIST_CHANGED, client, accid)
```

**Connecting to Signals:**
```python
urwid.connect_signal(eventcenter, CHATLIST_CHANGED, self.chatlist_changed)
```

### 2. Lazy Loading with Caching

The `LazyListWalker` pattern creates widgets on-demand and caches them:

```python
from functools import lru_cache

class LazyListWalker(urwid.ListWalker):
    def __init__(self, length: int, factory: Callable[[int], urwid.Widget], cache_size: int = 1000):
        self.length = length
        self._factory = lru_cache(maxsize=cache_size)(factory)
```

**Usage:**
```python
self.walker = LazyListWalker(len(chats), self._make_chat_item)
```

**Clearing Cache:**
```python
self.walker._factory.cache_clear()
```

### 3. Container Pattern for Key Interception

The `Container` widget wraps other widgets to intercept key events:

```python
chatlist_cont = Container(self.chatlist, self._chatlist_keypress)

def _chatlist_keypress(self, _size: list, key: str) -> Optional[str]:
    if key in ("right", "tab"):
        # Custom handling - give focus to composer
        self.main_columns.focus_position = 2
        return None  # Event consumed
    return key  # Pass through to wrapped widget
```

### 4. Theme Configuration

Themes are defined as dictionaries with urwid palette entries:

```python
theme = {
    "background": ("white", "g11"),
    "status": ("white", "g23"),
    "selected": ("black", "light blue"),
    # ... more entries
}

# Applied to MainLoop
loop = urwid.MainLoop(
    widget,
    [(key, *value) for key, value in theme.items()],
)
```

### 5. Keymap Configuration

Keymaps are simple dictionaries:

```python
keymap = {
    "quit": "q",
    "send": "enter",
    "newline": "meta enter",
    "next-chat": "meta up",
    "prev-chat": "meta down",
}
```

### 6. Account Management

Accounts are managed through the DeltaChat RPC API:

```python
# Get or create account
accid = get_or_create_account(rpc, "user@example.com")

# Check if configured
if rpc.is_configured(accid):
    addr = rpc.get_config(accid, "configured_addr")
```

### 7. Configuration Directories

Using `appdirs` for cross-platform config paths:

```python
import appdirs

config_dir = appdirs.user_config_dir("ArcaneChat")
# Linux: ~/.config/ArcaneChat/
# macOS: ~/Library/Application Support/ArcaneChat/
# Windows: %APPDATA%\ArcaneChat\
```

---

## Dependencies

### Core Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| `deltachat2` | >=0.8.0 | DeltaChat RPC client library |
| `urwid` | >=2.6.10 | Console UI framework (ncurses wrapper) |
| `urwid-readline` | >=0.14 | Readline-style editing for urwid |
| `appdirs` | >=1.4.4 | Cross-platform config directory paths |

### Optional Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| `deltachat-rpc-server` | >=2.22.0 | DeltaChat core RPC server (installable via `[full]`) |

### Development Dependencies

| Package | Purpose |
|---------|---------|
| `black` | Code formatting |
| `mypy` | Static type checking |
| `isort` | Import sorting |
| `pylint` | Code linting |
| `pylama` | Meta-linter wrapper |
| `pytest` | Testing framework |

### Important Version Notes
- **Python:** Requires 3.8+ for typing features
- **deltachat-rpc-server:** Recently updated to 2.22.0+ (removed deprecated APIs)
- **urwid:** Requires 2.6.10+ for specific features

---

## Testing Guidelines

### Current State
⚠️ **Testing infrastructure is configured but tests are not yet implemented.**

### Test Configuration
- Framework: pytest
- Dev dependency: `pytest` in `[project.optional-dependencies]`
- CI: Test step exists but is commented out in `.github/workflows/python-ci.yml`

### Running Tests (when implemented)

```bash
# Run all tests
pytest

# Run with verbose output
pytest -v

# Run specific test file
pytest tests/test_eventcenter.py
```

### Test File Structure (recommended)

```
arcanechat-tui/
├── arcanechat_tui/
└── tests/
    ├── __init__.py
    ├── test_eventcenter.py
    ├── test_util.py
    └── test_widgets.py
```

### Writing Tests (guidelines for future)

**Unit Tests:**
- Test individual functions in `util.py`
- Test signal emission in `EventCenter`
- Test widget behavior in isolation

**Integration Tests:**
- Test widget interactions
- Test signal flow through application
- Mock DeltaChat RPC client

**Example Test Structure:**
```python
import pytest
from arcanechat_tui.util import shorten_text

def test_shorten_text():
    assert shorten_text("Hello World", 5) == "Hell…"
    assert shorten_text("Hi", 10) == "Hi"
```

---

## Git Workflow

### Branching Strategy
- **main/master:** Stable releases
- Feature branches for development
- Tag releases with `v*.*.*` format

### Commit Message Guidelines
- Use clear, descriptive commit messages
- Reference issue numbers when applicable
- Follow conventional commits format (recommended)

**Examples:**
```
update to core 2.22: remove usage of deprecated chat.is_protection_broken
improve installation instructions in README
fix: handle missing config values gracefully
feat: add support for file attachments
```

### Version Management

**Versioning:** Uses `setuptools_scm` for automatic versioning from git tags

```bash
# Create a new version
git tag v1.2.3
git push origin v1.2.3
```

**Version File:** Auto-generated at `arcanechat_tui/_version.py` (gitignored)

### Pull Request Process

1. Create feature branch
2. Make changes
3. Run linters: `black . && pylama`
4. Ensure CI passes
5. Submit PR to main/master branch

### CI/CD Pipeline

**Triggered On:**
- Push to main/master branches
- Pull requests to main/master
- Tag push matching `v*.*.*`

**Test Job (runs on Python 3.8 and 3.12):**
```bash
pip install '.[dev]'
black --check .
pylama
# pytest (commented out - not yet implemented)
```

**Deploy Job (runs on version tags):**
- Builds package
- Publishes to PyPI (on version tags only)
- Uses `secrets.PYPI_TOKEN` for authentication

---

## Common Tasks

### Adding a New Widget

1. **Create widget file** in `arcanechat_tui/`
2. **Inherit from urwid widget class**
3. **Define signals if needed**
4. **Implement widget logic**
5. **Import and integrate** in `application.py`

**Example:**
```python
# mywidget.py
import urwid

CUSTOM_SIGNAL = "custom_signal"

class MyWidget(urwid.Filler):
    signals = [CUSTOM_SIGNAL]

    def __init__(self):
        urwid.register_signal(self.__class__, self.signals)
        super().__init__(urwid.Text("Hello"))
```

### Adding a New Event Handler

1. **Define signal constant** in appropriate module
2. **Register signal** in `__init__`
3. **Emit signal** when event occurs
4. **Connect handler** in application setup

**Example:**
```python
# In eventcenter.py - add new signal
NEW_EVENT = "new_event"

class EventCenter:
    signals = [CHATLIST_CHANGED, MESSAGES_CHANGED, NEW_EVENT]

    def process_core_event(self, client, accid, event):
        if event.kind == EventType.NEW_TYPE:
            urwid.emit_signal(self, NEW_EVENT, client, accid)

# In application.py - connect handler
urwid.connect_signal(eventcenter, NEW_EVENT, self.handle_new_event)
```

### Adding a CLI Subcommand

1. **Edit `cli.py`**
2. **Add subparser** in `Cli.__init__`
3. **Add handler method**
4. **Call handler** in `Cli.run`

**Example:**
```python
# In Cli.__init__
export_parser = subparsers.add_parser("export", help="Export account backup")
export_parser.add_argument("path", help="Backup file path")

# Add handler method
def export_cmd(self, args):
    # Implementation
    pass

# In Cli.run
elif args.command == "export":
    self.export_cmd(args)
```

### Modifying the Theme

Edit the `theme` dictionary in `main.py`:

```python
theme = {
    "new_style": ("foreground_color", "background_color"),
    # Example:
    "highlight": ("yellow", "dark blue"),
}
```

Available colors: See urwid documentation for color names and 256-color codes (e.g., `g11`, `#f00`)

### Adding Logging

```python
from .logger import create_logger

logger = create_logger(__name__)

logger.debug("Debug message")
logger.info("Info message")
logger.warning("Warning message")
logger.error("Error message")
```

**Log Location:** `~/.config/ArcaneChat/logs/log.txt`

---

## Important Files Reference

### Configuration Files

| File | Purpose |
|------|---------|
| `pyproject.toml` | Project metadata, dependencies, build config, tool settings |
| `pylama.ini` | Linting configuration (ignored rules, linters) |
| `.gitignore` | Git ignore patterns |
| `.github/workflows/python-ci.yml` | CI/CD pipeline configuration |

### Entry Points

| File | Purpose |
|------|---------|
| `main.py` | Main entry point, RPC setup, theme/keymap |
| `__main__.py` | Enables `python -m arcanechat_tui` |
| `cli.py` | CLI argument parsing and subcommands |

### UI Components

| File | Widget(s) |
|------|-----------|
| `application.py` | `Application` - main coordinator |
| `chatlist.py` | `ChatListWidget`, `ChatListItem` |
| `conversation.py` | `ConversationWidget`, `MessageItem`, `DayMarker` |
| `composer.py` | `ComposerWidget`, `ReadlineEdit2` |
| `cards_widget.py` | `CardsWidget` - stack layout |
| `container.py` | `Container` - key interceptor |
| `welcome_widget.py` | `WelcomeWidget` - placeholder |

### Utilities

| File | Purpose |
|------|---------|
| `util.py` | Helper functions (text, paths, account management) |
| `logger.py` | Logging configuration |
| `eventcenter.py` | Event translation layer |
| `lazylistwaker.py` | Lazy loading with caching |

---

## Gotchas & Important Notes

### 1. Auto-Generated Files
- **`_version.py`** is auto-generated by setuptools_scm - DO NOT EDIT
- Version is derived from git tags
- File is gitignored

### 2. Threading Considerations
- UI updates must happen on main thread (urwid event loop)
- DeltaChat events come from background thread
- Signals automatically marshal to main thread

### 3. Widget Focus Management
- Focus controlled via `focus_position` attributes
- Container hierarchy: `main_columns` → `right_side`
- Focus positions are zero-indexed

**Example:**
```python
# Give focus to composer
self.main_columns.focus_position = 2  # Right side
self.right_side.focus_position = 1     # Composer (not conversation)
```

### 4. Lazy List Walker Cache
- Always call `cache_clear()` when data changes
- Cache size default: 1000 items
- Factory function must create new widget instance each time

```python
def refresh_list(self):
    self.walker._factory.cache_clear()
    self.walker._modified()
```

### 5. DeltaChat API Changes
- Recently removed `chat.is_protection_broken` (deprecated)
- Always check deltachat2 documentation for API changes
- Minimum required version: deltachat2 >= 0.8.0, deltachat-rpc-server >= 2.22.0

### 6. Key Event Handling
- Returning `None` from keypress handler **consumes** the event
- Returning the key **passes through** to wrapped widget
- Container pattern allows pre/post processing

### 7. Signal Naming
- Use UPPERCASE constants for signal names
- Define in same module as emitter
- Register all signals in `signals` class attribute

### 8. Account Selection
- `accid = 0` means no account or unspecified
- Always check `rpc.is_configured(accid)` before use
- Selected account stored in `Application.accid`

### 9. Configuration Paths
- Config dir varies by platform (via appdirs)
- Accounts stored in `{config_dir}/accounts/`
- Logs stored in `{config_dir}/logs/`

### 10. Terminal Color Support
- Application requires 256-color terminal
- Set via `loop.screen.set_terminal_properties(colors=256)`
- Test theme changes in actual terminal (not all emulators support all colors)

### 11. Read Receipts
- Messages auto-marked as noticed when displayed
- Read status: `✓✓` (read), `✓` (delivered), `→` (pending), `!` (failed)
- Handled in `ConversationWidget.set_chat()`

### 12. Contact Requests
- Auto-accepted when sending first message
- Handled in `ComposerWidget.send_message()`
- Check `chat.is_contact_request` before accepting

### 13. Import Patching
- `ReadlineEdit2` patches `urwid_readline.ReadlineEdit`
- Required for custom keymap support
- Defined in `composer.py`

### 14. Logging Configuration
- Default level from environment or config
- Uses `RotatingFileHandler` (3 backups, 2MB each)
- Format includes timestamp, level, module, message

### 15. Testing in Development
- Currently no tests exist (infrastructure ready)
- Before adding features, consider adding tests
- Mock DeltaChat RPC client for unit tests

---

## Quick Reference: Key Files to Modify for Common Changes

| Task | Files to Modify |
|------|----------------|
| Add new UI widget | `arcanechat_tui/newwidget.py`, `application.py` |
| Add DeltaChat event handler | `eventcenter.py`, `application.py` |
| Add CLI subcommand | `cli.py` |
| Change theme colors | `main.py` (theme dict) |
| Add keyboard shortcut | `main.py` (keymap dict), `application.py` |
| Add utility function | `util.py` |
| Change logging behavior | `logger.py` |
| Update dependencies | `pyproject.toml` |
| Change linting rules | `pylama.ini` |
| Modify CI/CD | `.github/workflows/python-ci.yml` |

---

## Additional Resources

- **User Guide:** `docs/user-guide.md`
- **README:** `README.md` (installation, features)
- **Delta Chat RPC Docs:** https://py.delta.chat/
- **Urwid Documentation:** http://urwid.org/
- **Project Repository:** https://github.com/ArcaneChat/arcanechat-tui
- **Issue Tracker:** https://github.com/ArcaneChat/arcanechat-tui/issues

---

## Version History

| Date | Version | Changes |
|------|---------|---------|
| 2025-11-24 | Initial | Created comprehensive CLAUDE.md based on v2.22+ codebase |

---

**End of CLAUDE.md**
