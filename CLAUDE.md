# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**claude-rescue** is a zero-dependency Node.js TUI (Terminal User Interface) tool that monitors running Claude Code sessions in real-time, detects anomalies (stuck/retrying/errored sessions), and enables one-keypress takeover to rescue them by spawning a fresh Claude session in the same project directory.

**Core design principle**: Read-only monitoring with explicit takeover action. The tool never modifies or kills running sessions.

**Target environment**: Cross-platform (macOS, Windows, Linux) with identical behavior.

## Architecture

### Single-file structure

All code lives in `sessions.js` (~1000 lines). No external dependencies — only Node.js built-ins (`fs`, `os`, `path`, `child_process`, `readline`, `tty`).

### Key subsystems

1. **Session discovery** (`loadSessions`, `getSessionsDir`)
   - Reads `~/.claude/sessions/*.json` (or `$CLAUDE_CONFIG_DIR/sessions/`)
   - Each file represents one running Claude Code session (PID-based)
   - Files are created/updated by Claude Code itself, removed on exit

2. **Liveness detection** (`isAlive`)
   - Uses `process.kill(pid, 0)` — probes existence without actually signaling
   - Returns false if process doesn't exist, true if alive or EPERM (exists but no permission)

3. **Anomaly detection** (`detectAbnormal`, `annotateSessions`)
   - Reads transcript tail from `~/.claude/projects/<encoded-cwd>/<sessionId>.jsonl`
   - Detects three states:
     - `retrying`: last substantive record is `system/api_error` (API auto-retry in progress)
     - `error`: last 3 substantive records contain an assistant message with `isApiErrorMessage: true`
     - `slow`: process alive but transcript hasn't updated in 5+ minutes
   - Uses mtime/size-based caching (`_abnormalCache`) for performance
   - Skips metadata records (`META_TYPES`: mode, permission-mode, file-history-snapshot, attachment, ai-title, last-prompt, custom-title, agent-name)

4. **TUI rendering** (`runTUI`, `drawScreen`)
   - Full-screen interface using ANSI escape codes and raw terminal mode
   - Auto-refresh every 2 seconds (toggle with `a`, manual refresh with `g`)
   - Color coding: red = error, yellow = retrying/slow, green = healthy
   - Navigation: `↑`/`↓` or `j`/`k`, `PgUp`/`PgDn`, `Home`/`End`
   - Filtering (`/`), sorting (`s`: recent / name / cwd), detail view (`Enter`)

5. **Takeover wizard** (`openTerminalAndRun`, `buildResumeCommand`)
   - Press `o` to enter 3-step wizard: confirm takeover → skip permissions? → name?
   - Opens new terminal (Terminal.app on macOS via osascript, PowerShell on Windows, generic on Linux)
   - Spawns fresh Claude session with instruction: "sessionId为xxx的任务卡住了，你帮我看看任务进度现在到哪里了，下一步我该做什么，请你继续执行任务。"
   - Copies command to clipboard as backup
   - **Does NOT use `--resume`** — starts a clean context to avoid hitting the same rate-limit/timeout loop

6. **Non-interactive modes** (`printPlain`)
   - `--list`: plain-text table (Chinese labels)
   - `--json`: JSON output with English field names for scripting

## Development Commands

### Run the tool

```bash
node sessions.js              # Interactive TUI
node sessions.js --list       # Plain-text table
node sessions.js --json       # JSON output
node sessions.js --dir <path> # Use custom sessions directory
```

### Install globally

```bash
npm install -g .
claude-rescue
```

### Test changes

No automated tests. Manual testing procedure:

1. Start multiple Claude Code sessions in different projects
2. Run `node sessions.js` to verify they appear
3. Test navigation (`j`/`k`, `/` filter, `s` sort)
4. Test detail view (`Enter`)
5. Test takeover (`o`) — verify terminal opens with correct command
6. Test anomaly detection:
   - Let a session hit API error/timeout (red status)
   - Check transcript parsing logic in `detectAbnormalUncached`

### Debugging

- Set `DEBUG=1` environment variable if adding debug logging (currently none exists)
- Check `_abnormalCache` state if transcript detection is wrong
- Verify `getSessionsDir()` returns correct path on each OS

## Code Conventions

- **Chinese UI**: All user-facing strings (labels, prompts, help text) are Simplified Chinese; internal code/comments are mixed Chinese/English
- **No dependencies**: Do not add any `require()` statements beyond Node.js built-ins
- **Cross-platform paths**: Use `path.join()` and `os.homedir()`, never hardcode `/` or `\`
- **ANSI color codes**: Use `\x1b[...m` directly (no chalk/colors lib) — see `C` object for color constants
- **Error handling**: Catch file I/O errors silently (sessions can disappear mid-read); show user-facing errors in Chinese

## Important Constraints

1. **Session files are volatile**: A session can exit and delete its file at any moment. Always use try-catch around `fs.readFileSync` / `fs.statSync`.

2. **Transcript location heuristic**: The tool guesses transcript path as `projects/<cwd-with-slashes-replaced-by-dashes>/<sessionId>.jsonl`, then falls back to scanning all project subdirs. This heuristic can break if Claude Code changes its encoding scheme.

3. **PID reuse**: `isAlive(pid)` can give false positives if the OS reuses the PID quickly. Rare but possible.

4. **Terminal compatibility**: Windows needs VT-capable terminal (Windows Terminal, Win10+ conhost). Very old consoles should use `--list` / `--json`.

5. **Rename/delete operations not supported**: The README explains why — session files are owned by running Claude processes, which immediately overwrite external edits. Use `/rename` inside Claude instead.

## Key Functions to Modify

- **Add new anomaly detection**: Edit `detectAbnormalUncached` and add new `kind` to `abnormalLabel`
- **Change takeover instruction**: Edit `resumePrompt` function (line ~269)
- **Adjust slow threshold**: Change `SLOW_MS` constant (default 5 minutes)
- **Add new keyboard shortcuts**: Edit `runTUI` event handlers (around line 700+)
- **Change color scheme**: Edit `C` object constants (around line 400)

## Testing Anomaly Detection

To verify transcript parsing:

```javascript
// Add temporary debug output in detectAbnormalUncached:
console.error('Essential records:', essential.length, 'Last:', last);
```

Then run tool while a session is retrying/errored and check stderr.

## Cross-platform Testing

- **macOS**: Test `osascript` terminal opening and AppleScript quoting
- **Windows**: Test PowerShell opening and single-quote escaping (`''` for `'`)
- **Linux**: Generic terminal fallback (`x-terminal-emulator -e`)

## Notes

- The tool is a **viewer**, not a controller. It never kills, pauses, or modifies running sessions.
- Takeover is **opt-in per session** — user must press `o` and confirm.
- The fresh Claude session gets a Chinese instruction because the tool's primary audience is Chinese-speaking developers (per README acknowledgements of LINUX DO community).
