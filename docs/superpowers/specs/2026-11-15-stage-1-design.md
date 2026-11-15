# Atrium stage 1: projects, agents and terminals

Status: draft for review, 2026-11-15.

## Goal

Atrium is a cross-platform (Windows and macOS) desktop app for running Claude Code agents and project sessions. It will replace ClaudeDesk, the macOS Swift app, once it matches it. Stage 1 delivers the daily core:

- choose a workspace and prepare it;
- create and list agents, and connect them to the Inbox, routines, widgets and the Memory Vault exactly as ClaudeDesk does;
- add projects with named session shortcuts;
- run Claude Code conversations for agents and projects in embedded terminal panes with splits;
- resume last session's conversations after a relaunch.

Later stages: (2) Inbox and routines (Windows Task Scheduler), (3) Cockpit, (4) mail and calendar plus a signed installer and auto-update.

## Decisions

| Topic | Decision |
|---|---|
| Product name | **Atrium** (repo `poom-h/atrium-desk`) |
| Licence | MIT, matching atrium-memory |
| Compatibility | Writes the same files and the same text as ClaudeDesk 3.8.1, as described in [WORKSPACE-SPEC.md](../../WORKSPACE-SPEC.md). Both apps can open the same workspace on any day. Agent-facing text keeps saying "ClaudeDesk" and markers keep the `claudedesk:` prefix until a planned rename. Format changes are additive only (new files or keys ClaudeDesk ignores). |
| Improvement review | Each stage reviews its part of ClaudeDesk and adopts chosen improvements. |
| Stack | Tauri 2, Rust core, React + TypeScript + Vite UI, pnpm |
| Core approach | Rust owns PTYs (`portable-pty`: ConPTY on Windows, openpty on macOS) and all workspace logic; the UI is a thin view |
| Stage-1 extras | Agent connection steps only. The skill library UI, the agent document editor and the vault viewer are later; you open agent files in your own editor. |

## Repository layout

```
atrium-desk/
├── LICENSE                   MIT
├── package.json              Vite + React + TS, pnpm
├── src/                      UI
├── src-tauri/                Tauri app: window, commands, events, bundling, sidecar config
├── crates/
│   ├── workspace/            workspace format (no Tauri dependency)
│   ├── pty/                  PTY sessions, environment, claude lookup
│   └── frontmatter/          ClaudeDesk front matter dialect
├── resources/
│   ├── VaultTemplate/        copied from ClaudeDesk
│   └── SkillTemplates/       inbox-instructions, widget-instructions, copied from ClaudeDesk
├── scripts/fetch-atrium-memory   downloads the pinned atrium-memory release per OS
├── atrium-memory.version     pinned version (0.1.0)
├── .github/workflows/ci.yml  build and test on macos-latest and windows-latest
└── docs/
```

Rust crates form a Cargo workspace. `src-tauri` depends on the three crates; the crates depend on none of Tauri, so the stage-2 runner can reuse them as a command-line helper.

## Architecture

```
React UI ──commands──▶ src-tauri ──▶ crates/workspace  (files in <workspace>)
   ▲                      │      └─▶ crates/pty        (claude processes)
   └──events/channels─────┘
```

- **Commands:** `get_state`, `set_workspace`, `create_agent`, `rename_agent`, `repair_agent`, `reconnect_agent`, `add_project`, `update_project`, `remove_project`, `open_pane`, `write_pane`, `resize_pane`, `close_pane`, `save_layout`, `diagnostics`.
- **Streams:** each pane has its own Tauri channel carrying output bytes. Input and resize go back through commands.
- **Events:** `agents-changed` (folder watcher on `agent_brain/`), `pane-exited`, `health-changed`.
- **Truth:** the workspace is the truth for agents. Per-machine state lives in Atrium's config folder (`~/Library/Application Support/Atrium/` on macOS, `%APPDATA%\Atrium\` on Windows): `settings.json`, `projects.json`, `layout.json`, `external-agents.json`.
- **Single instance (A2):** a second launch focuses the running window (Tauri single-instance plugin).

## Workspace

### Setup

- First run shows a screen to choose the workspace folder. On macOS, Atrium suggests ClaudeDesk's `workspacePath` (UserDefaults domain `local.claudedesk.app`, read with `defaults read`) when present.
- On every start, Atrium runs `prepare` exactly as WORKSPACE-SPEC section 1.1.
- **Workspace record (W2):** Atrium creates or updates `.claudedesk/workspace.json`:

```json
{
  "formatVersion" : 1,
  "createdBy" : "ClaudeDesk",
  "lastOpenedBy" : { "app" : "Atrium", "version" : "0.1.0", "time" : "2026-11-15T01:30:00Z" }
}
```

  `createdBy` is `Atrium` when Atrium prepared an empty folder, otherwise `ClaudeDesk`. ClaudeDesk ignores the file.
- **Health check (W1):** after preparing, and when watched folders change, Atrium checks: workspace and `.claudedesk/` writable; `MemoryVault/Home.md` present; `routines/routines.md` readable; bundled skills present; every agent readable (below); atrium-memory runs (M2). Results show in a Health panel opened from a sidebar badge (green, amber or red with a count).
- **Cloud-sync warning (W3):** when the workspace path is inside OneDrive, iCloud Drive (`~/Library/Mobile Documents`) or Dropbox, the Health panel shows a warning. It also lists conflict copies it finds in `inbox/`, `routines/` and `.claudedesk/` (names containing ` (1)`, `conflicted copy` or `-<PCNAME>`).

### Finding agents

- An agent is a folder directly in `agent_brain/` whose name doesn't start with `.` and that contains `agent.json`. The folder's real location is the truth; a stale `path` inside `agent.json` is ignored and not rewritten.
- A folder watcher (`notify` crate, debounced 500 ms) rescans on changes (D1).
- **Broken agents (D2):** an agent with unreadable `agent.json`, or missing `.runtime/`, `memory/MEMORY.md`, `CLAUDE.md` or `.mcp.json`, is listed with the reason and a Repair button. Repair only creates missing pieces with their initial contents (WORKSPACE-SPEC 3.3); it never overwrites.
- **Duplicate IDs (D3):** when two folders share a UUID, both are flagged. "Make this a new agent" gives the chosen folder a new UUID, rewrites its `agent.json` and renames its folder suffix to the new UUID; the other keeps the original.
- **ClaudeDesk agents outside the workspace (macOS):** on first run Atrium reads `~/Library/Application Support/ClaudeDesk/agents.json` and records agents whose folders are outside `agent_brain/` in `external-agents.json`; they appear in the list with an "outside workspace" label.

### Creating agents

- Name, purpose, and presets (C3): model (default, or a model ID such as `opus` or `sonnet`) and permission mode (`default`, `acceptEdits`, `plan`). Presets are written as `model` and `permissions.defaultMode` in `.runtime/settings.json` next to `autoMemoryDirectory`; left at default, the file is exactly ClaudeDesk's.
- Everything else as WORKSPACE-SPEC 3.2 and 3.3, byte for byte: folder name, staging folder, file contents, `agent.json` with `createdAt` as seconds since 2001-01-01 and an uppercase UUID. POSIX `0700` on macOS only.
- Then the connection steps run for the new agent.

### Renaming agents (C4)

Rename changes `name` in `agent.json` and `ATRIUM_MEMORY_ACTOR` in the app-written atrium-memory entry. The folder name is unchanged, and so is the `# <name>` heading in `CLAUDE.md` (it may have been edited). Logged as `agent.edited`. Known limitation: ClaudeDesk shows names from its own per-machine registry and keeps the old name until renamed there.

## Connection steps

On every start and after each new agent, for each agent whose folder exists, Atrium runs the four steps in WORKSPACE-SPEC 3.5–3.7:

1. Inbox and routines: `additionalDirectories`, the `## Inbox` and `## Routines` sections, attach `inbox-instructions`.
2. Widgets: `widgets/assets/`, the `## Cockpit widgets` section, attach `widget-instructions`.
3. atrium-memory server entry (every start).
4. atrium-memory settings and instructions (first time only).

Each step skips what is already present. Text written is byte-identical to ClaudeDesk's, with the paths for the current OS.

### Recording what's done

`.claudedesk/connections.json` replaces ClaudeDesk's per-machine flag lists for Atrium:

```json
{
  "agents" : {
    "8E1F…UUID…" : {
      "inbox" : "2026-11-15T01:30:00Z",
      "widgets" : "2026-11-15T01:30:00Z",
      "memory" : "2026-11-15T01:30:00Z"
    }
  }
}
```

A step listed here is not run again in its "first time" form (steps 1, 2 and 4). On macOS, the first time Atrium opens a workspace it seeds this file from ClaudeDesk's `workspaceConnectedAgents.v2`, `.widgets.v1` and `.memory.v1` lists, so steps you undid in ClaudeDesk are not redone. Agents never get access to `.claudedesk/`.

### Connections panel (K1)

Each agent has a Connections panel listing the four steps with done, missing or failed (with the error) and a Reconnect button. Reconnect runs the step in its first-time form even if recorded.

### Minimal JSON edits (K3)

When Atrium changes `.mcp.json` or `.claude/settings.local.json`, it keeps the file's existing key order, indentation and line endings and changes only the values it needs (`serde_json` with `preserve_order`, plus re-indentation matching the file). New files are written as ClaudeDesk writes them (pretty, sorted keys).

### Backups (K4)

Before changing any agent file, Atrium copies the previous version to `.claudedesk/backups/<yyyy-MM-dd>/<agent folder>/<relative path>` (one copy per file per day; first version of the day wins). Backup folders older than 30 days are deleted at start.

### Platform details

- **Vault deny rule:** macOS `Edit(//Users/me/Workspace/MemoryVault/**)`. Windows uses Claude Code's POSIX form: `C:\Users\me\Workspace\MemoryVault` becomes `Edit(//c/Users/me/Workspace/MemoryVault/**)` (drive letter lowercase), per the Claude Code permissions docs.
- **`additionalDirectories` on Windows:** written as native paths (`C:\Users\…`). The docs don't say whether these are accepted; this is verified on Windows before stage 1 ships (see Risks).
- **atrium-memory entry:** recognised as app-written when `args` is `["mcp"]` and `command` ends with `/atrium-memory` or `\atrium-memory.exe` (case-insensitive on Windows). Atrium repairs an app-written entry only when its `command` no longer exists or its `ATRIUM_MEMORY_PATH` is not the current vault. It does not repoint a working ClaudeDesk binary to itself.
- **Skill links:** macOS uses relative symlinks (S2) for agents inside the workspace, for example `../../../../skill_library/inbox-instructions` from `<workspace>/agent_brain/<agent>/.claude/skills/`; links for external agents and the global `~/.claude/skills/` stay absolute. Windows uses directory junctions (S1). "Attached" means a symlink or junction that resolves to the library folder. Dangling links in agent and global skills folders (S3) appear in the Health panel with Remove or Relink.

### atrium-memory checks

- **Version (M1):** Atrium runs the bundled binary with `--version` and each agent's entry `command` with `--version` (cached by path and modification time). An entry pointing at an older version than the bundled one shows a warning on the agent with "Use Atrium's".
- **Health probe (M2):** at start, Atrium runs the bundled binary as an MCP server against the vault with `ATRIUM_MEMORY_ACTOR=atrium-health`, sends `initialize` and `tools/list`, expects `vault_read` in the list, and stops it. A failure (including Gatekeeper or SmartScreen blocking) shows in the Health panel.

If `--version` isn't supported by atrium-memory 0.1.0, M1 is done with a small atrium-memory change before the pinned version is bumped.

## Terminals

### Starting Claude

- Lookup: the `claudeExecutable` setting, then macOS `~/.local/bin/claude`, `/opt/homebrew/bin/claude`, `/usr/local/bin/claude`, or Windows `%USERPROFILE%\.local\bin\claude.exe`, then `PATH`.
- Environment: Atrium's environment minus every variable starting with `CLAUDE`; for agents, plus `CLAUDE_CONFIG_DIR=<agent>/.runtime` and `CLAUDE_CODE_PROJECT_DIR_NAME=agent`; `TERM=xterm-256color`.
- macOS runs `$SHELL -l -c 'exec <claude> <args>'` so your login profile applies. Windows runs `claude.exe` directly through ConPTY.
- Arguments: `--session-id <new lowercase UUID>` for new conversations (plus `-- <prompt>` when a shortcut has one), `--resume <id>` to resume.
- Logs `conversation.started`, `conversation.resumed`, `conversation.ended` to `logs/` with the same details as ClaudeDesk (`pane`, `session`, and `agent` or `project`).
- **Missing claude (T2):** instead of a pane, a guide with the install command for your OS, a Browse button and a Retry button.

### Panes

- xterm.js with the WebGL, fit and web-links addons; font size from settings.
- Title bar: coloured mark (ClaudeDesk's colour formula, from the agent UUID), agent or project name and session name, activity dot, close.
- **Activity dot (T3):** working, waiting for you, or idle, from the newest line of the session's transcript (ClaudeDesk's step logic in WORKSPACE-SPEC 8.4), checked every 2 seconds.
- **Exit (T1):** the pane stays, showing the exit code and the last non-empty output line, with Restart (new session), Resume (same session) and Close.

### Layout

- A split tree per mode (Projects, Agents). Split right or down, drag a title bar onto another pane's edge to move it, drag dividers to resize.
- **Drop targets (L2):** drag an agent or project from the sidebar onto a pane edge to open a conversation there.
- **Keyboard (L1):** split right / down, move focus between panes, close pane, reopen the last closed pane (resumes its session). Ctrl on Windows, ⌘ on macOS.

### Restore

- `layout.json` stores both trees with each pane's kind (`agent` with UUID, or `project` with ID and session shortcut), session ID and title, saved whenever the layout changes.
- **On launch (R1):** panes return closed, each with Resume. A bar offers "N conversations from last time · Resume all · Dismiss".
- **Stale check (R2):** when the session's transcript file doesn't exist, the pane offers "Start new" instead of Resume.

## Projects

- Same model as ClaudeDesk: name, folder, and named session shortcuts with optional prompts; a new project gets "Main session". Stored in Atrium's `projects.json` (ISO 8601 dates). On macOS, imported once from ClaudeDesk's `projects.json`.
- Project sessions use your normal Claude profile (no `CLAUDE_CONFIG_DIR`).
- **Missing folder (P1):** shown greyed with Locate; adding a folder that's already a project selects it.

## App shell

- One window: sidebar (Projects/Agents switch, list, Health badge, Settings) and the pane area. Inbox, Routines, Skills and Cockpit appear greyed with "Coming in stage 2" or "stage 3".
- **Command palette (A1):** Ctrl/⌘+K to open an agent or project, create an agent or project, switch mode, open Health or Settings.
- **Settings:** workspace folder, claude path, terminal font size, theme (follows the system). On macOS, the first run imports `workspacePath`, `claudeExecutable` and `terminalFontSize` from ClaudeDesk.
- **Diagnostics (A3):** Atrium, claude and atrium-memory versions, workspace and config paths, OS; Copy button.

## Errors

- Every file change goes through the `workspace` crate's writer: atomic write, backup first (K4), never overwrite a file it couldn't parse, with ClaudeDesk's error wording for the same cases.
- Problems collect in the Health panel and the Connections panel; at most one toast per action.
- PTY problems show inside the pane (T1, T2) and never crash the app.

## Testing

- **Compatibility fixtures:** a ClaudeDesk 3.8.1 workspace captured from this Mac (a new agent before and after connection, `.mcp.json`, settings, `CLAUDE.md`). Tests assert Atrium produces byte-identical files for the same inputs, and that running the steps on an already connected agent changes nothing.
- **Unit tests:** front matter round trips, folder naming, `agent.json` date encoding, app-written entry detection, deny rule forms for macOS and Windows, junction and relative-link attachment checks, layout serialisation.
- **CI:** GitHub Actions on `macos-latest` and `windows-latest`: `cargo test`, `pnpm test` (Vitest for UI logic), and a PTY smoke test that spawns a shell command, reads its output and resizes.
- **Manual release check:** open one workspace in ClaudeDesk, then Atrium, then ClaudeDesk again; `git diff` of the workspace (initialised for the test) shows no changes after the second ClaudeDesk start.

## Bundling atrium-memory

`scripts/fetch-atrium-memory` downloads the release asset for the pinned version (`atrium-memory.version`) for the target OS and architecture from `poom-h/atrium-memory` releases and verifies its checksum. Tauri bundles it as a sidecar next to the app executable, so its path ends in `/atrium-memory` (macOS) or `\atrium-memory.exe` (Windows). This needs the atrium-memory v0.1.0 release (pending its PR #2 and tag) to publish macOS and Windows assets.

## Out of scope for stage 1

Inbox UI and MCP inbox tools, routines and the runner, Cockpit and widgets, the skill library UI, the agent document editor, the vault viewer, mail and calendar, signing and auto-update. Later-stage ideas recorded from the review: W4, W5, D4, D5, C1, C2, C5, K2, K5, M3, M4, M5, S4, S5, T4, T5, L3–L5, R3–R5, P2–P5, A4, A5.

### Stage-2 decision recorded: Inbox through MCP tools

Inbox files stay as they are, but agents write them through new atrium-memory tools instead of the Write tool: `inbox_send` (validates, fills `from` from `ATRIUM_MEMORY_ACTOR`, plus `session` and `created`, refuses `alert` and `routine`, writes atomically) and `inbox_status`/`inbox_answers`. A deny rule blocks direct edits to `inbox/`. This fixes unchecked YAML, forgeable senders, agents' write access to other items, and half-written files.

## Risks

1. Windows acceptance of `C:\…` paths in `permissions.additionalDirectories` and of an absolute `.exe` command in `.mcp.json` is undocumented; both are tested on Windows early in implementation, and the spec is amended if another form is needed.
2. ConPTY behaviour (resize, Ctrl+C, wide characters) in `portable-pty` needs the CI smoke test plus manual checks.
3. The atrium-memory release must publish Windows and macOS binaries before Atrium can bundle it.
