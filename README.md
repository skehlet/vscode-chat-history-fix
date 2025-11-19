# VS Code Chat History Fix

A utility to repair corrupted chat session indices in VS Code's workspace storage.

> **Problem:** VS Code chat sessions become invisible due to index corruption in `state.vscdb`, despite session data files remaining intact on disk.  
> **Solution:** This tool scans session files and rebuilds the database index to restore visibility of all chat sessions.

## Quick Start

### Step 1: Preview Changes

```bash
python3 fix_chat_session_index_v3.py --dry-run
```

Displays affected workspaces and sessions to be restored without modifying any files.

### Step 2: Apply the Fix

**Close VS Code completely**, then run:

```bash
python3 fix_chat_session_index_v3.py
```

### Step 3: Verify

Restart VS Code and verify sessions appear in the Chat view.

---

## Technical Overview

### Storage Architecture

VS Code's core chat service (not the GitHub Copilot extension) manages regular chat sessions using the following structure:

```
~/.config/Code/User/workspaceStorage/<workspace-id>/
├── state.vscdb                    # SQLite database
│   └── chat.ChatSessionStore.index  # Index of all sessions
└── chatSessions/
    ├── session-1.json             # Your actual chat data
    ├── session-2.json
    └── session-3.json
```

**Session Creation Process:**
1. Full conversation stored as JSON in `chatSessions/`
2. Index entry added to `state.vscdb` with metadata (title, timestamp, location)

**Session Restoration Process:**
- On startup, VS Code reads `chat.ChatSessionStore.index` from `state.vscdb` to determine which sessions to load

### Root Cause

The index in `state.vscdb` can become corrupted or out of sync with actual session files, causing:
- Session data files remain intact on disk
- Index missing entries for existing sessions
- VS Code unable to discover sessions during restoration

**Example scenario:**
- Session files on disk: 13
- Index entries in database: 1
- Sessions visible in UI: 1

### Repair Process

The tool performs the following operations:
1. Scans `chatSessions/` directory for all session JSON files
2. Extracts metadata from each session file
3. Rebuilds `chat.ChatSessionStore.index` in `state.vscdb`
4. Creates timestamped backup before modifications

---

## Available Scripts

### Option 1: Automatic Repair (Recommended)

`fix_chat_session_index_v3.py` - Automatically detects and repairs all affected workspaces.

```bash
# See what would be fixed (safe preview)
python3 fix_chat_session_index_v3.py --dry-run

# Fix everything (asks for confirmation)
python3 fix_chat_session_index_v3.py

# Fix everything automatically (no prompts)
python3 fix_chat_session_index_v3.py --yes
```

### Option 2: Manual Workspace Selection

`fix_chat_session_index_v2.py` - Repair specific workspace by ID.

```bash
# List your workspaces
python3 fix_chat_session_index_v2.py

# Fix a specific workspace
python3 fix_chat_session_index_v2.py <workspace_id>
```

Supports the same options as v3:

```bash
# Preview changes without writing the DB
python3 fix_chat_session_index_v2.py <workspace_id> --dry-run

# Apply the fix without prompts
python3 fix_chat_session_index_v2.py <workspace_id> --yes

# Remove orphaned index entries (default is to keep them)
python3 fix_chat_session_index_v2.py <workspace_id> --remove-orphans
```

---

## Important Considerations

### Prerequisites

- Close VS Code completely before running repair scripts to prevent database locks and conflicts

### Safety Features

- Automatic backup creation before any modifications
- Read-only preview mode via `--dry-run` flag
- Index-only modifications - session data files remain untouched
- Zero data loss risk

### System Requirements

- Python 3.6+
- No external dependencies (uses Python standard library only)
- Cross-platform: Linux, macOS, Windows

---

## Example Output

### Preview Mode (--dry-run)

```
🔍 Scanning VS Code workspaces...
   Found 3 workspace(s) with chat sessions

🔧 Found 1 workspace(s) needing repair:

1. Workspace: 68afb7ebecb251d147a02dcf70c41df7
   Folder: /home/user/my-project
   Sessions on disk: 13
   Sessions in index: 1
   ⚠️  Missing from index: 12

📊 Total issues:
   Sessions to restore: 12

🔧 Repairing workspaces...

   Repairing: 68afb7ebecb251d147a02dcf70c41df7 (/home/user/my-project)
      ✅ Will restore 12 session(s)
         • How to fix TypeScript compilation errors (2024-10-28 22:50)
         • Implement user authentication system (2024-10-06 19:25)
         • Debug React component rendering issue (2024-10-07 09:22)
         • Setup PostgreSQL database connection (2024-10-25 11:03)
         • Write unit tests for API endpoints (2024-10-08 16:50)
         ... and 7 more

🔍 DRY RUN COMPLETE

To apply these changes, run without --dry-run:
   python3 fix_chat_session_index_v3.py
```

### Actual Repair

```
✨ REPAIR COMPLETE
   Workspaces repaired: 1
   Total sessions restored: 12

📝 Next Steps:
   1. Start VS Code
   2. Open the Chat view
   3. Your sessions should now be visible!

💾 Backups were created for all modified databases
```

---

## Troubleshooting

**No workspaces found**
- Verify VS Code Chat has been used previously
- Confirm workspace storage directory exists: `~/.config/Code/User/workspaceStorage/` (Linux/macOS) or `%APPDATA%\Code\User\workspaceStorage\` (Windows)

**Sessions not restored after repair**
- Confirm VS Code was completely closed before running the script
- Reload VS Code window: `Ctrl+Shift+P` → "Reload Window"
- Verify backup file creation was successful
- Check workspace ID matches current project

**Rollback procedure**
- Locate backup: `state.vscdb.backup.<timestamp>`
- Replace current database: `cp state.vscdb.backup.<timestamp> state.vscdb`

---

## Upstream Issue

This is a VS Code core bug, not a GitHub Copilot extension issue. The Copilot extension manages only specialized sessions (Claude Code, Copilot CLI, PR sessions) - regular chat session restoration is handled by VS Code's core chat service.

**Analysis:**
- `chat.ChatSessionStore.index` in `state.vscdb` becomes desynchronized from session files
- Write operations succeed but read/restoration logic fails
- Likely race condition in VS Code's chat service initialization

### Reporting

- Technical details: See `VSCODE_CORE_BUG_REPORT.md`
- File issues: https://github.com/microsoft/vscode/issues

---

## FAQ

**Can sessions be transferred between workspaces?**  
Yes. Session files are standard JSON. Copy files between workspace `chatSessions/` directories, then run the repair script to update the index.

**Folder mode vs workspace file (.code-workspace) storage?**  
Different workspace modes use distinct storage locations. Chat histories exist in both locations but are isolated by workspace context.

**Does this tool delete any data?**  
No. Only the database index is modified. Session data files are read-only operations.

**What are orphaned index entries?**  
Index references to non-existent session files. Retained by default for safety (e.g., temporarily unmounted drives). Use `--remove-orphans` to clean up.

---

## Use Cases

Addresses the following symptoms:
- Chat history disappears after VS Code restart
- Previously visible sessions no longer appear in Chat view
- Session count mismatch between filesystem and UI
- Workspace migration with incomplete session restoration

## Contributing

Bug reports and improvements welcome via issues or pull requests.

## License

MIT

## Support

For issues, provide:
- OS and VS Code version
- Output from `--dry-run` mode
- Complete error messages and stack traces
