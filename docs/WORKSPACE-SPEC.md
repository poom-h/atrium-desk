# Workspace format

This document describes exactly what ClaudeDesk 3.8.1 (the macOS Swift app, `poom-h/ClaudeDesk`) reads and writes, so that this app and ClaudeDesk can open the same workspace. Everything here comes from the ClaudeDesk source; file names in brackets, such as [Workspace.swift], name the source file each section follows.

Where this document says "the app", it means any program that manages the workspace. Where a behaviour is specific to the Swift app (macOS paths, UserDefaults, launchd), it is marked **macOS-only** and the new app needs its own equivalent.

Conventions:

- Paths are written with `/`. `<workspace>` is the workspace folder the user chose.
- "Pretty JSON" means 2-space indented JSON with sorted keys, as written by `JSONSerialization`/`JSONEncoder` with `.prettyPrinted, .sortedKeys`. Unless noted, slashes are not escaped (`.withoutEscapingSlashes`).
- "ISO 8601" means `2026-11-15T08:30:00Z` (UTC, no fractional seconds), the default output of `ISO8601DateFormatter`.
- "Local stamp" means local time without a zone, `yyyy-MM-dd'T'HH:mm`, for example `2026-11-15T08:30`.
- All text files are UTF-8. Writes are atomic (write a temporary file, then rename) unless noted.

## 1. Workspace layout

[Workspace.swift]

```
<workspace>/
├── agent_brain/            agents, one folder each (section 3)
├── MemoryVault/            the Memory Vault (section 4)
├── skill_library/          shared skills (section 5)
├── inbox/                  items waiting for the user (section 6)
│   └── archive/            resolved items
├── routines/
│   └── routines.md         routine definitions (section 7)
├── logs/                   activity log, YYYY-MM.jsonl (section 9)
└── .claudedesk/            app and runner records agents must not touch
    ├── queue/              routine run requests
    ├── runs/               run reports, live runs, manual run requests
    ├── usage/              plan limits and session costs
    ├── approved.json       routine approvals
    ├── state.json          runner state
    ├── cockpit.json        Cockpit layout
    ├── runner.lock
    └── runner.log          (macOS: launchd stdout/stderr)
```

Folder names are fixed: `agent_brain`, `MemoryVault`, `skill_library`, `inbox`, `logs`, `routines`, `.claudedesk`.

### 1.1 Preparing a workspace

On setting a workspace, and again every time workspace services start, the app runs `prepare`. It only creates what is missing; existing folders and files are never changed.

1. The workspace folder must already exist and be a folder, or preparation fails with "The workspace folder is unavailable: <path>".
2. Create, when missing: `agent_brain/`, `skill_library/`, `inbox/`, `inbox/archive/`, `logs/`, `routines/`, `.claudedesk/queue/`, `.claudedesk/runs/`.
3. When `routines/routines.md` is missing, write the routines template (section 7.1).
4. For each bundled skill template folder (names not starting with `.`), copy it to `skill_library/<name>/` only when that folder is missing. The bundled skills are `inbox-instructions` and `widget-instructions` (section 5.4). Later edits in the library are therefore kept.
5. When `MemoryVault/` is missing, build it in a staging folder `<workspace>/.creating-vault-<UUID>`, then move it into place:
   - copy the bundled vault template (`Home.md`, `Maps/About this vault.md`, `Notes/Second-Brain-Framework.md`, `.tools/vault_check.py`);
   - in every `.md` file, replace `{{VAULT_PATH}}` with the vault's final absolute path;
   - create `Inbox/`, `Notes/`, `Maps/`, `Assets/`, `Archive/`;
   - on any failure, delete the staging folder.

An existing `MemoryVault/` is never modified by preparation.

### 1.2 Migration from the agent host folder

Before workspaces existed, ClaudeDesk stored only an agent host folder (setting `agentHostPath`). When `workspacePath` is empty and `agentHostPath` is set, and the user has not declined (`workspaceMigrationDeclined`), the app suggests a workspace: the host folder's parent if the host folder is named `agent_brain`, otherwise the host folder itself. While no workspace is set, new agents are created in `agentHostPath`.

## 2. Per-machine state outside the workspace

These are **not** in the workspace, so a second app on the same machine (or the same workspace on another machine) does not see them. All are **macOS-only** locations.

| What | Where | Format |
|---|---|---|
| Agent registry | `~/Library/Application Support/ClaudeDesk/agents.json` | Pretty JSON array of agent profiles (section 3.1) |
| Projects | `~/Library/Application Support/ClaudeDesk/projects.json` | Pretty JSON array of projects (section 2.1) |
| Mail and calendar cache | `~/Library/Application Support/ClaudeDesk/mail-cache.json` | Stage 4; not specified here |
| Settings | UserDefaults domain `local.claudedesk.app` | Section 11 |
| Secrets (Google, Jev API key) | macOS Keychain | Stage 4; not specified here |
| Routine runner schedule | `~/Library/LaunchAgents/local.claudedesk.runner.plist` | Section 7.7 |

**Important for compatibility:** the agent registry is per machine. The workspace itself holds enough to rebuild it, since every agent folder contains its own `agent.json` with the same fields. A new app should discover agents by scanning `<workspace>/agent_brain/*/agent.json`, and treat the registry as a cache. Agents that ClaudeDesk created in an old agent host folder outside the workspace are only listed in the registry.

### 2.1 Projects

[Models.swift, Store.swift] A project is a code folder where the user runs ordinary Claude sessions with their normal Claude profile.

```json
[
  {
    "id" : "2F0C4C1E-6A43-4F5B-9A0D-6C1B2E3F4A5B",
    "name" : "my-repo",
    "path" : "/Users/me/Code/my-repo",
    "sessions" : [
      { "id" : "…UUID…", "name" : "Main session", "prompt" : "", "lastOpened" : 785000000.0 }
    ]
  }
]
```

- `id`, session `id`: UUIDs, uppercase (Swift's default `UUID` encoding).
- `sessions`: named launch shortcuts. A new project gets one, `"Main session"` with an empty prompt. `lastOpened` is optional.
- `agentID`: optional; present only on the synthetic projects built for agents, which are not saved.
- **Dates use Swift's default `Date` encoding: seconds since 2001-01-01 00:00:00 UTC as a number**, not ISO 8601. The same applies to `agent.json` and `agents.json`.
- Adding a project whose path is already listed selects the existing one instead.

## 3. Agents (`agent_brain/`)

[AgentEnvironment.swift, AgentStore.swift]

### 3.1 Agent profile (`agent.json`)

```json
{"id":"8E1F…UUID…","name":"Atlas","purpose":"Keeps the vault tidy.","path":"/Users/me/Workspace/agent_brain/atlas-8e1f…","createdAt":785000000.0}
```

| Field | Type | Notes |
|---|---|---|
| `id` | UUID string (uppercase) | |
| `name` | string | Display name, trimmed, non-empty |
| `purpose` | string | Free text; becomes the second paragraph of `CLAUDE.md` |
| `path` | string | Absolute path of the agent folder |
| `createdAt` | number | Seconds since 2001-01-01 UTC |

`agent.json` inside the agent folder is written once at creation with a plain `JSONEncoder` (compact, properties in declaration order, slashes escaped as `\/`). The registry `agents.json` holds the same objects as pretty JSON. Derived values: the **configuration directory** is `<path>/.runtime`; the **agent folder name** is the last path component of `path`.

### 3.2 Creating an agent

1. Folder name: the name lowercased, split on every non-alphanumeric character, empty parts dropped, joined with `-`, cut to 48 characters, then `-` and the new UUID in lowercase. Example: `Atlas Research` → `atlas-research-8e1f2c3d-…`.
2. Build in a staging folder `<agent_brain>/.creating-<UUID>` (uppercase UUID), then move to the final name. On failure delete the staging folder.
3. Create folders: `memory/`, `files/`, `.runtime/`, `.claude/skills/`, `.claude/agents/`.
4. Write the files in 3.3.
5. Set the folder's POSIX permissions to `0700` (**macOS-only**; the new app needs a Windows equivalent or none).
6. Append the profile to the registry and save it.
7. Connect the agent to the Inbox, widgets and Memory Vault (sections 3.5 to 3.7).

### 3.3 Agent folder contents

```
<agent folder>/
├── agent.json
├── CLAUDE.md                     instructions
├── memory/
│   └── MEMORY.md                 agent memory
├── files/                        the agent's working files
├── .mcp.json                     MCP servers
├── .claude/
│   ├── settings.local.json       local overrides
│   ├── skills/                   attached skills (symlinks into skill_library/)
│   └── agents/
├── .runtime/                     private Claude profile (CLAUDE_CONFIG_DIR)
│   ├── settings.json
│   └── projects/agent/<session-id>.jsonl   transcripts, written by Claude Code
├── widgets/                      Cockpit widgets (section 10)
│   └── assets/
└── skills-proposed/<name>/SKILL.md          written by the agent (section 5.3)
```

Initial file contents:

`CLAUDE.md` (with `<name>` and `<purpose>` filled in):

```markdown
# <name>

<purpose>

## Working environment
This directory is your dedicated working environment. Keep your work in files/ unless the user directs otherwise.
Do not access or change sibling agents' environments unless the user explicitly requests it.

## Persistent memory
Read memory/MEMORY.md at the beginning of each conversation. Maintain concise durable notes there and put detailed topic notes in memory/.
Save useful preferences, decisions, and ongoing task context when appropriate. Do not save credentials or secrets in memory.
Memory is shared across this agent's conversations; read the latest file before editing and preserve other sessions' updates.

@memory/MEMORY.md

```

`memory/MEMORY.md`:

```markdown
# Memory

No memories recorded yet.
```

`.mcp.json`:

```json
{
  "mcpServers": {}
}
```

`.runtime/settings.json` (pretty JSON **with slashes escaped** as `\/`, since this write omits `.withoutEscapingSlashes`; the path is the agent's final folder, not the staging folder):

```json
{
  "autoMemoryDirectory" : "/Users/me/Workspace/agent_brain/atlas-8e1f…/memory"
}
```

`.claude/settings.local.json`: `{}` followed by a newline.

### 3.4 Agent documents and editing rules

The app's editor exposes five documents:

| Document | File | Validation on save |
|---|---|---|
| Instructions | `CLAUDE.md` | none |
| Memory | `memory/MEMORY.md` | none |
| MCP servers | `.mcp.json` | JSON object with a `mcpServers` object whose values are all objects |
| Settings | `.runtime/settings.json` | JSON object |
| Local overrides | `.claude/settings.local.json` | JSON object |

Rules:

- A document that is a symbolic link, or whose path passes through one, is not opened in the app ("This document uses a symbolic link…"). The resolved path must stay inside the agent folder.
- Saving is refused when the file on disk differs from the text the editor loaded (optimistic concurrency): "This file changed outside the editor…".
- Approved memory items are appended to `memory/MEMORY.md` as `\n## <heading>\n\n<text>\n`, keeping all existing text. The heading is `Approved from the Inbox, <item created value>`.

### 3.5 Inbox and routines connection

[AgentEnvironment.connectInbox] Run once per agent (tracked by the setting `workspaceConnectedAgents.v2`, a list of agent UUIDs). Each step is skipped if already done; nothing else in the files changes.

1. `.claude/settings.local.json`: append the absolute paths of `<workspace>/inbox` and `<workspace>/routines` to `permissions.additionalDirectories` (creating the objects as needed) unless already present. The file is rewritten as pretty JSON. A file that is not a JSON object stops the connection with an error.
2. `CLAUDE.md`: append these sections when their marker is absent (a newline is added first if the file doesn't end with one):

```markdown

## Inbox
<!-- claudedesk:inbox -->
Send the user non-urgent requests, questions, reports, and proposed skills through the ClaudeDesk Inbox at `<workspace>/inbox`, following the `inbox-instructions` skill.

```

```markdown

## Routines
<!-- claudedesk:routines -->
Scheduled routines for every agent are defined in `<workspace>/routines/routines.md`. Read the `inbox-instructions` skill before adding or changing one; new or riskier routines wait for the user's approval.

```

3. Attach the library skill `inbox-instructions` to the agent (section 5.2).

### 3.6 Widgets connection

[AgentEnvironment.connectWidgets] Run once per agent (setting `workspaceConnectedAgents.widgets.v1`).

1. Create `widgets/assets/`.
2. Append, when the marker is absent:

```markdown

## Cockpit widgets
<!-- claudedesk:widgets -->
You design your own widgets on the user's ClaudeDesk Cockpit: one Markdown file per widget in `<agent folder>/widgets`, with images in its `assets/` folder. Follow the `widget-instructions` skill, and keep your widgets current when your work changes what they show.

```

3. Attach the library skill `widget-instructions`.

### 3.7 Memory Vault connection (atrium-memory)

[MemoryTool.swift] atrium-memory is the Memory Vault MCP server (`poom-h/atrium-memory`), bundled inside the app. ClaudeDesk 3.8.1 bundles version `0.1.0` (`Resources/atrium-memory.version`) at `ClaudeDesk.app/Contents/MacOS/atrium-memory`.

Constants:

| Name | Value |
|---|---|
| Server name | `atrium-memory` |
| Tool permission | `mcp__atrium-memory` |
| CLAUDE.md marker | `<!-- claudedesk:memory-tool -->` |
| Vault edit rule | `Edit(/` + vault absolute path + `/**)`; on macOS the path starts with `/`, giving `Edit(//Users/me/Workspace/MemoryVault/**)` |

**Server entry** in the agent's `.mcp.json` under `mcpServers`:

```json
"atrium-memory" : {
  "args" : [ "mcp" ],
  "command" : "/Applications/ClaudeDesk.app/Contents/MacOS/atrium-memory",
  "env" : {
    "ATRIUM_MEMORY_ACTOR" : "Atlas",
    "ATRIUM_MEMORY_PATH" : "/Users/me/Workspace/MemoryVault"
  }
}
```

`ATRIUM_MEMORY_ACTOR` is the agent's display name. For the user's own projects the app offers the same entry, with actor `project`, as JSON to paste (it does not write project files).

**App-written entry:** an entry whose `args` is exactly `["mcp"]` and whose `command` ends with `/atrium-memory`. Only app-written entries are ever changed. An entry the user edited so it no longer matches is left alone. (On Windows the new app's binary is `atrium-memory.exe` and paths use `\`; the test needs a Windows form. See open questions.)

**Connecting** runs on every start of workspace services, for every agent whose folder exists, when the bundled executable is present:

1. If the app-written entry points at a different vault (`env.ATRIUM_MEMORY_PATH`), migrate first:
   - in `.claude/settings.local.json`, replace the old vault edit rule in `permissions.deny` with the new one in the same position (or just remove the old one if the new one is already there); nothing is added if the user removed the old rule;
   - in `CLAUDE.md`, replace the exact old instructions section with the new one; a section the user edited is left alone.
   Migration runs before the entry is rewritten, so a failed migration is retried next start.
2. Server entry: add it when missing. When the app-written entry's `command` or `ATRIUM_MEMORY_PATH` differs from the running app and current vault, update just those two values and keep every other key. Write `.mcp.json` as pretty JSON. Errors: file not a JSON object; `mcpServers` not an object.
3. Only the first time for each agent (setting `workspaceConnectedAgents.memory.v1`):
   - `.claude/settings.local.json`: add `atrium-memory` to `enabledMcpjsonServers`, `mcp__atrium-memory` to `permissions.allow`, and the vault edit rule to `permissions.deny`, each only if missing. Rewritten as pretty JSON only when something changed. Errors when the file, `permissions`, or a list has the wrong type.
   - `CLAUDE.md`: append the instructions section below when the marker is absent. An unreadable `CLAUDE.md` is never overwritten.

Instructions section (with `<vault>` the vault's absolute path):

```markdown

## Memory Vault tools
<!-- claudedesk:memory-tool -->
The Memory Vault at `<vault>` is reached through the `atrium-memory` tools, which are built into ClaudeDesk. Use them instead of reading or editing vault files directly; direct edits in the vault are blocked.
- Find notes: start from Home and follow the maps with `vault_read` (it returns each note's links), or use `vault_search`.
- Change notes: `vault_read` first and keep its `version`. Then use `vault_edit` for one passage, `vault_append` to add to the end, or `vault_write` to create a note (no version) or replace a whole note (with the version).
- On `stale_version`, the note changed since you read it: read it again, merge your change, and retry. On `exists`, read the note and update it instead.
- `vault_list` lists a folder's notes, and `vault_backlinks` shows which notes link to one.
- The vault checker still runs with `python3 <vault>/.tools/vault_check.py`.

```

Because migration matches this section exactly, the new app must produce the same text byte for byte for the same vault path (or migration of sections written by the other app will be skipped).

An agent counts as **connected** when its `.mcp.json` `mcpServers` has an `atrium-memory` key, app-written or not.

### 3.8 Starting an agent conversation

[AgentStore.launchAgent, AgentEnvironment.environment]

- Working directory: the agent folder. The folder and `.runtime/` must exist.
- Environment: the app's environment with every variable whose name starts with `CLAUDE` removed, then `CLAUDE_CONFIG_DIR=<agent>/.runtime` and `CLAUDE_CODE_PROJECT_DIR_NAME=agent` added. Removing inherited `CLAUDE*` variables stops panes from becoming nested children of a Claude session that launched the app.
- Panes start through a login shell so the user's profile settings apply (**macOS** detail of the embedded terminal).
- A new conversation gets `--session-id <new lowercase UUID>` (plus `-- <prompt>` when there is one). Resuming uses `--resume <session id>`.
- Because of `CLAUDE_CODE_PROJECT_DIR_NAME=agent`, an agent's transcripts are `<agent>/.runtime/projects/agent/<session-id>.jsonl`. The Cockpit reads these for the latest step and "last active".
- Project sessions use the normal profile; their transcripts are under `~/.claude/projects/<encoded path>/<session-id>.jsonl`.
- Claude executable: the `claudeExecutable` setting (`~` expanded) if set, otherwise the first executable of `~/.local/bin/claude`, `/opt/homebrew/bin/claude`, `/usr/local/bin/claude`.

### 3.9 Finding an agent by reference

[RoutineRunner.swift, `matching`] Inbox `from`, routine `agent`, manual-run `agent` and Cockpit settings refer to agents loosely. Lookup, case-insensitive:

1. exact match on the UUID or on the agent folder name;
2. otherwise, a unique agent whose display name equals the reference, or whose folder name starts with `<reference>-` (so `atlas` finds `atlas-8e1f…`). No match, or more than one, means no agent.

## 4. Memory Vault (`MemoryVault/`)

The note format is defined by atrium-memory's [`SPEC.md`](https://github.com/poom-h/atrium-memory/blob/main/SPEC.md): notes are `.md` files; dot-folders are not notes; a note ID is its `/`-separated relative path (NFC); the version is the SHA-256 of the file bytes; links use Obsidian `[[Target]]` syntax; atrium-memory keeps `audit.jsonl` and `write.lock` in `<vault>/.atrium-memory/`.

What ClaudeDesk adds on top:

- **Layout created:** `Home.md`, `Inbox/`, `Notes/`, `Maps/`, `Assets/`, `Archive/`, `.tools/vault_check.py` (section 1.1).
- **Direct edits are blocked** for connected agents through the deny rule (section 3.7). Agents change notes only through atrium-memory.
- **Vault checker:** the app runs `python3 <vault>/.tools/vault_check.py` (via `/usr/bin/env`) at Cockpit start and every 120 seconds, and shows the text after `Result: ` on the last line that starts with it.
- **Graph (Cockpit):** nodes are `Home.md` (kind home), `.md` files under `Maps/` (map) and `Notes/` (note), skipping hidden files and files whose resolved path leaves those folders. A node's title is its first `# ` heading, otherwise the file name. Edges are `[[target` links outside fenced code (```` ``` ```` or `~~~`), matched by the last path part of the target, case-insensitive, `.md` optional, one undirected edge per pair. (This simpler matching differs from atrium-memory's link rules; it is only used for drawing.)
- **Read activity:** the Cockpit tails `<vault>/.atrium-memory/audit.jsonl` to highlight notes agents read (`vault_read`) or found (`vault_search`), and also tails agent transcripts changed in the last 15 minutes for vault paths.

## 5. Skill library (`skill_library/`)

[SkillLibrary.swift]

### 5.1 Skills

- A skill is a folder `skill_library/<name>/` containing `SKILL.md`. Folders starting with `.` or without `SKILL.md` are ignored. Skills are listed sorted by name.
- Valid names match `^[a-z0-9][a-z0-9-]{0,63}$`.
- `SKILL.md` starts with front matter; the app reads `description` from it (section 12).
- New skill template:

```markdown
---
name: <name>
description: <description, or "Describe when Claude should use this skill.">
---

# <name>

Write the steps Claude should follow when this skill applies.

```

The description is quoted by the front matter rules when needed. Saving is refused when `SKILL.md` changed on disk since it was loaded.

### 5.2 Attaching skills

A skill is attached to a target by a **symbolic link** named `<skill name>` in the target's skills folder, pointing at the absolute path of `skill_library/<name>`.

| Target | Skills folder |
|---|---|
| Global (normal Claude sessions) | `~/.claude/skills/` |
| Agent | `<agent>/.claude/skills/` |
| Project | `<project>/.claude/skills/` |

- **Attached:** the entry is a symlink that resolves to the library folder. **Detached:** no entry. **Conflict:** anything else (a real folder, or a link elsewhere); the app refuses to attach or detach.
- Attaching to a project also adds `.claude/skills/<name>` to the project's `.gitignore` unless that line or `/.claude/skills/<name>` is already there.
- Detaching removes only a link that points at the library; real folders are never deleted.
- Target IDs used in logs: `global`, `agent:<UUID>`, `project:<UUID>`.

On Windows, symlinks need Developer Mode or elevation; this is an open question for the new app (directory junctions are the usual alternative).

### 5.3 Proposed skills

An agent writes a proposal to `<agent>/skills-proposed/<name>/SKILL.md` and files a `skill` inbox item. Approving it:

1. refuses invalid names, symlinks (the `skills-proposed` folder, the proposal folder, or any file inside), a missing `SKILL.md`, and a name already in the library;
2. moves the proposal folder to `skill_library/<name>`;
3. attaches it to the proposing agent.

### 5.4 Bundled skills

The app ships two skills, copied into the library when missing (section 1.1). Their text is the agent-facing description of the Inbox, routines and widgets, and is the authoritative reference for what agents write:

- `inbox-instructions`: inbox item format, kinds, callbacks, proposing skills, routines.
- `widget-instructions`: agent widget file format and blocks.

The new app must ship these same two skills (renamed product text aside) so agents connected by either app behave the same.

## 6. Inbox (`inbox/`)

[Inbox.swift, InboxStore.swift, inbox-instructions skill]

### 6.1 Items

Each waiting item is one `.md` file directly in `inbox/` (names starting with `.` are ignored). Images an item uses may sit beside it in subfolders such as `inbox/assets/`.

File names:

- written by agents: `<YYYY-MM-DD>-<agent folder name>-<short-slug>.md`;
- written by the app or runner: `<yyyy-MM-dd-HHmmss>-claudedesk-<slug>.md`, adding `-2`, `-3`, … when taken. Slugs are `routine-<name>` and `alert-<routine>`.

Example:

```markdown
---
kind: request
from: atlas-8e1f2c3d-…
via: conversation
session: 6d1c…-session-id
title: Archive 5 duplicate AWS notes?
created: 2026-10-23T09:15
status: pending
choices: [Archive all, Keep them]
on_approve:
  - resume: "The user chose: {choice} {reply}. Continue."
on_reject:
  - resume: "The user declined: {reason}"
---

Explain what you want and why…
```

Properties:

| Key | Meaning |
|---|---|
| `kind` | `request`, `message`, `skill`, `memory` (agents); `routine`, `alert` (app only). `draft` is reserved for a later version |
| `from` | The sending agent's folder name (section 3.9 lookup); `claudedesk` for items from the app or runner |
| `via` | `conversation` (default), `routine:<name>`, `runner`, or a manual run kind (`shortcut`, `capture`, `widget`, `ask`) |
| `session` | The conversation's session ID, or the run ID for background runs |
| `title` | One line; defaults to the file name without `.md` |
| `created` | Local stamp |
| `status` | `pending` while waiting |
| `topic` | Optional label such as `news`; Cockpit widgets show the latest item per topic |
| `skill` | Required for `skill` items: the proposed skill name |
| `routine`, `fingerprint` | Required for `routine` items (section 7.4); `routine` defaults to the name in `via: routine:<name>` |
| `agent` | Set on app-written `routine` items |
| `choices` | Inline or block list of choice labels |
| `on_approve`, `on_reject` | Block lists of callbacks: `- action: "argument"` or `- action` |

`memory` items carry the text to append in the first fenced block opened by a line ```` ```memory ```` and closed by a line ```` ``` ````.

### 6.2 Validation

An item is shown with a problem (and never acted on) when, in this order: it has no front matter; `kind` is empty; `kind` is `draft`; `kind` is unknown; `from` is empty; a `skill` item lacks `skill`; a `routine` item lacks `routine` or `fingerprint`; a `memory` item has no memory block.

Sorting: `alert` items first, then newest first by (`created`, file name).

### 6.3 Callbacks

Known actions: `resume`, `new-conversation`, `run-routine`, `none`. Anything else is shown as unsupported and never run. The user can switch off any callback before deciding.

| Action | Effect |
|---|---|
| `resume: "<message>"` | Open the sending agent with `--resume <session>` and, when the message is not empty, `-- <message>` |
| `new-conversation: "<prompt>"` | Start a new conversation with the sending agent and the prompt |
| `run-routine: "<name>"` | Queue the named routine (or the item's own `routine` when empty) unless already queued or running (section 7.6) |
| `none` | Nothing |

Placeholders in arguments: `{reason}` and `{reply}` become the user's note, `{choice}` the chosen label (empty otherwise).

### 6.4 Resolving an item

Decisions: approve → status `approved`, decision `approve`; reject → `rejected`, `reject`; choice → `answered`, `choice: <label>`; reply → `answered`, `reply`; done → `done`, `done`.

1. Refuse if the file's text differs from what was shown.
2. Built-in action by kind:
   - `skill` + approve: import the proposal (section 5.3);
   - `routine` + approve or reject: record the decision (section 7.4);
   - `memory` + approve: append to the sender's `memory/MEMORY.md` (section 3.4).
3. Set front matter keys `status`, `resolved` (ISO 8601), `decision`, and, when the user wrote a note, `reason` (for reject) or `reply` (otherwise). Existing single-line keys are replaced in place; new keys go before the closing `---`.
4. Write the result to `inbox/archive/<same file name>` (or `<name>-2.md`, `-3`, … when taken) and delete the original.
5. Move relative images the body uses (`![alt](path)`, not URLs or absolute paths, and staying inside `inbox/`) to the same relative path under `archive/`; copy instead when another waiting item uses the same image; skip when the destination exists.
6. Run the enabled, known callbacks for the decision: `on_approve` for approve, choice and reply; `on_reject` for reject; none for done. For each, append to the archived file, creating the heading once:

```markdown

## Callbacks
- resume: reopened conversation 6d1c…
```

### 6.5 Agents and the archive

Agents only add new files to `inbox/` and read `archive/`. They never edit or delete other items.

## 7. Routines

[Routines.swift, RoutineRunner.swift, Runner/main.swift]

### 7.1 `routines/routines.md`

Each routine is a `## <name>` section outside fenced code: property lines (`- key: value`) after optional blank lines, then the prompt (everything after the first non-property line, trimmed). Text before the first `##` is ignored. Template written on first preparation:

````markdown
# Routines

Scheduled work for ClaudeDesk agents. Each routine is a `##` section: property lines, then the prompt.
The background runner checks this file every 5 minutes while your Mac is awake.

Properties: `agent` (required), `schedule` (required: `daily 08:30`, `weekdays 08:30`, `mondays 09:00`,
`every 6h`, `every 30m`, `hourly`), `skill`, `max_turns` (default 30), `timeout` (default 15m),
`enabled` (yes/no), `tools` (`web-fetch`), `output`.

New routines, a different agent, running more often than hourly, higher limits, or extra tools wait for
approval in the ClaudeDesk Inbox. Prompt and time-of-day edits apply right away.

Example (remove the backticks to use it):

```markdown
## heartbeat
- agent: atlas
- schedule: daily 08:30
- skill: heartbeat

Run your heartbeat routine for today.
```

````

Properties (keys case-insensitive, values unquoted by the front matter rules):

| Key | Rule |
|---|---|
| `agent` | Required. Agent reference (section 3.9) |
| `schedule` | Required. See 7.2 |
| `skill` | Library skill to use |
| `max_turns` | 1–200, default 30 |
| `timeout` | `<n>s`, `<n>m` or `<n>h`, between 1m and 4h, default 15m |
| `enabled` | `no`, `false`, `off` disable; anything else enables |
| `tools` | Comma list; the only known tool is `web-fetch` |
| `output` | Free text: where to write results |

A routine is invalid (with a combined problem message) when its name fails the skill-name rule, a name repeats, a property is unknown or out of range, `agent` or `schedule` is missing, or it has neither prompt nor skill.

### 7.2 Schedules

Case-insensitive phrases: `daily HH:MM`, `weekdays HH:MM` (Monday–Friday), `<day> HH:MM` or `<day>s HH:MM` with a full English day name (`monday`, `mondays`…), `every <n>m`, `every <n>h`, `hourly` (= every 60m). Times use the local time zone.

- **Minimum interval** (used for approvals): daily and weekdays 1440 min, weekly 10080 min, every n = n.
- **Due:** an interval schedule is due when it has never run or `now - lastRun >= interval`. A clock schedule is due when its most recent scheduled time at or before now (looking back up to 8 days) is later than both the last run and the approval time. A missed time therefore runs once, not once per miss.

### 7.3 Fingerprint and approvals

The **fingerprint** is the fields that need approval, as a string:

```
<agent lowercased>|<minimum interval minutes>|<max turns>|<timeout seconds>|<tools sorted, comma-joined>
```

Example: `atlas|1440|30|900|`.

A routine needs approval (reasons in brackets) when it has no grant (`new routine`), or compared with its grant:

- agent differs, case-insensitive (`runs as a different agent (<agent>)`);
- interval under 60 min and under the granted interval (`runs more often than hourly`);
- max turns above 30 and above the granted value (`turn limit raised to <n>`);
- timeout above 15m and above the granted value (`time limit raised to <n>m`);
- tools not in the grant (`adds tools: <list>`).

Prompt and time-of-day changes never need approval.

`.claudedesk/approved.json` (pretty JSON, ISO 8601 dates):

```json
{
  "grants" : {
    "heartbeat" : {
      "agent" : "atlas",
      "approvedAt" : "2026-11-01T01:30:00Z",
      "maxTurns" : 30,
      "minimumIntervalMinutes" : 1440,
      "timeoutSeconds" : 900,
      "tools" : [ ]
    }
  },
  "rejected" : {
    "news-hourly" : "ai-scout|30|30|900|web-fetch"
  }
}
```

### 7.4 Approval inbox items

When a valid, enabled routine needs approval, is not rejected at its current fingerprint, and no request was filed yet for that fingerprint, the runner files an item and sets the routine's `requestedFingerprint`:

````markdown
---
kind: routine
from: claudedesk
via: runner
title: "Approve routine “heartbeat” for Atlas?"
created: 2026-11-15T08:30
status: pending
routine: heartbeat
fingerprint: "atlas|1440|30|900|"
agent: atlas
---

The runner paused this routine until you approve it.

Why: new routine.

Approving covers the routine exactly as written below. Later edits to the prompt or time of day apply without asking; the changes above need approval again.

```markdown
## heartbeat
- agent: atlas
- schedule: daily 08:30
- skill: heartbeat

Run your heartbeat routine for today.
```
````

Deciding: approve re-reads `routines.md`, fails if the routine is gone or its fingerprint changed, then stores a grant from the current routine (approved now) and clears any rejection. Reject stores `rejected[name] = fingerprint`.

### 7.5 Runner state and records

`.claudedesk/state.json` (pretty JSON, ISO 8601):

```json
{
  "capAlertDay" : "2026-11-15",
  "day" : "2026-11-15",
  "lastPass" : "2026-11-15T01:35:00Z",
  "routines" : {
    "heartbeat" : {
      "lastOutcome" : "ok",
      "lastRun" : "2026-11-15T01:30:04Z",
      "lastRunID" : "0b6f…",
      "requestedFingerprint" : "atlas|1440|30|900|"
    }
  },
  "runsToday" : 3
}
```

`lastOutcome` values: `ok`, `failed`, `timed-out`, `turn-limit`, `stopped`, `invalid`. `day` is the local date; `runsToday` resets when it changes.

`.claudedesk/queue/<UUID>.json` (a run request):

```json
{ "note" : "Inbox item title", "requestedBy" : "user", "routine" : "heartbeat", "time" : "2026-11-15T01:30:00Z" }
```

`requestedBy` is `user` or `inbox`.

`.claudedesk/runs/`:

| File | Content |
|---|---|
| `<runID>.json` | The final `result` line of Claude's stream-json output |
| `<runID>.stderr.txt` | stderr, when not empty |
| `<runID>.live.json` | Live run (below), updated while running |
| `<runID>.stop` | Empty marker: the user pressed Stop |
| `<runID>.request.json` | A manual run request (section 8.3), deleted when picked up |

Live run:

```json
{
  "agentID" : "8E1F…UUID…",
  "finished" : "2026-11-15T01:31:10Z",
  "kind" : "routine",
  "pid" : 41234,
  "result" : "Done: …",
  "runID" : "0b6f…",
  "started" : "2026-11-15T01:30:04Z",
  "status" : "ok",
  "steps" : [ "Reading memory/MEMORY.md", "Using atrium-memory" ],
  "title" : "heartbeat"
}
```

`status`: `running`, `ok`, `failed`, `timed-out`, `turn-limit`, `stopped`. `steps` keeps the last 8 plain-language steps (section 8.4). The Cockpit shows runs still running plus those finished in the last 5 minutes.

### 7.6 A runner pass

One pass, safe to repeat:

1. Take an exclusive non-blocking lock on `.claudedesk/runner.lock` (`flock`); exit if another pass holds it.
2. Parse `routines.md`, load approvals and state; reset the day counter on a new day.
3. Read and delete every queued request (one request per routine is kept). Log `routine.skipped` (`unknown-routine`) for routines not in the file.
4. For each routine, saving state after each:
   - invalid: on the first pass it is invalid, set `lastOutcome` to `invalid`, file an alert and log `routine.invalid`; skip;
   - disabled: skip (log `routine.skipped` / `disabled` if it was requested);
   - needs approval: file a request once per fingerprint (7.4), log `routine.paused`; skip;
   - due when requested or by schedule (7.2);
   - at 20 runs today: file one alert per day ("Daily routine limit reached (20 runs)"), log `routine.cap-reached`, notify; skip;
   - otherwise run it, count it, and record `lastRun`, `lastOutcome`, `lastRunID`.
5. Record `lastPass`.

**Busy** (for queueing): a routine is `queued` while a request for it is in the queue, and `running` while a live run with kind `routine`, that title and status `running` exists whose process is alive (or, with no PID, started under 120 seconds ago). Queueing a busy routine does nothing.

### 7.7 Running Claude in the background

Each background run (routine or manual):

- Run ID: new lowercase UUID; it is also the Claude session ID.
- Working directory: the agent folder; environment as in 3.8 plus `PATH` prefixed with `~/.local/bin:/opt/homebrew/bin:/usr/local/bin:`. stdin is empty.
- Arguments:

```
-p --output-format stream-json --verbose --session-id <runID> --max-turns <n>
--permission-mode default --permission-prompts none
--tools Read,Write,Edit,Glob,Grep,Bash,WebSearch,Skill[,WebFetch]
--allowedTools Read Glob Grep WebSearch Skill [WebFetch] <edit rules…> [checker rules…] [mcp__atrium-memory]
[--disallowedTools <vault edit rule>]
-- <prompt>
```

  - `WebFetch` is added to both lists when the routine has `tools: web-fetch`.
  - Edit rules: `Edit(/<dir>/**)` for the agent folder and each absolute `permissions.additionalDirectories` entry of `.claude/settings.local.json`, except the vault when the agent is connected to atrium-memory.
  - Checker rules: for each of those directories that contains `.tools/vault_check.py`, `Bash(python3 <checker>)` and `Bash(python3 <checker> *)`.
  - Connected agents also get `mcp__atrium-memory`, and the vault edit rule as a disallowed tool.
- Timeout: terminate at the routine's timeout, kill 10 seconds later.
- Outcome: `stopped` if the stop marker exists (then delete it); `timed-out`; `turn-limit` if the result `subtype` is `error_max_turns`; `failed` if the exit status is not 0, the result has `is_error: true`, or there was no result; otherwise `ok`.
- Failures other than stopped file an alert for routines and post a notification. A successful run that created inbox items (items whose `session` is the run ID) notifies "<agent> sent N inbox items".

Routine prompt, joined with blank lines:

```
This is a scheduled ClaudeDesk routine run, not a conversation. Nobody is watching, so you can't ask questions. File anything that needs the user through the Inbox, then finish what you can.

Routine: <name>. Run ID: <runID>. In inbox items you write, use `session: <runID>` and `via: routine:<name>`.

[The user asked for this run now[, noting: <note>].]

[Answers to this routine's earlier inbox requests:
- “<title>”: <decision or status>[. Reply: <reply>][. Reason: <reason>]]

[Use the `<skill>` skill.]

[Write the results to: <output>]

[<prompt>]
```

"Earlier answers" are archived items with `via: routine:<name>`, `from` other than `claudedesk`, and `resolved` after the routine's last run.

Alerts:

```markdown
---
kind: alert
from: claudedesk
via: routine:heartbeat
title: "Routine “heartbeat” failed"
created: 2026-11-15T08:31
status: pending
session: 0b6f…
routine: heartbeat
---

Claude exited with an error (status 1).

Run output: `<workspace>/.claudedesk/runs/0b6f….json`
```

Alert titles: `Routine “<name>” can't run: fix routines.md`, `Daily routine limit reached (20 runs)`, `Routine “<name>” failed: no agent “<agent>”`, `Routine “<name>” failed`, `Routine “<name>” stopped: <timed-out|turn-limit>`.

**Scheduling (macOS-only):** the runner is the helper `claudedesk-runner` inside the app bundle. "Run routines in the background" writes `~/Library/LaunchAgents/local.claudedesk.runner.plist` with `ProgramArguments` `[<runner>, --workspace, <workspace>]`, `StartInterval` 300, `RunAtLoad` true, `ProcessType` Background, and stdout/stderr to `.claudedesk/runner.log`, then `launchctl bootstrap gui/<uid>`. "Run now" queues a request and runs `launchctl kickstart gui/<uid>/local.claudedesk.runner`. The runner is on when the plist exists. Runner modes: no flag (one pass), `--run-request <file>`, `--statusline`. Without `--workspace` it reads `workspacePath` from the app's settings.

**Two apps, one workspace:** both apps' runners must not run the same workspace at once. The pass lock prevents overlapping passes on one machine only if both use `flock` on the same file; the new app's runner must take the same lock on `.claudedesk/runner.lock`.

## 8. Cockpit

[CockpitData.swift, CockpitStore.swift]

### 8.1 Layout (`.claudedesk/cockpit.json`)

Pretty JSON, ISO 8601 dates. Missing or unreadable means the standard layout.

```json
{
  "shortcuts" : [
    { "agent" : "ai-scout", "id" : "…UUID…", "instruction" : "", "skill" : "ai-tech-news", "title" : "News now", "webFetch" : true }
  ],
  "theme" : "space",
  "widgetScale" : 1,
  "widgets" : [
    { "id" : "…UUID…", "kind" : "today", "settings" : { }, "side" : 0, "size" : "wide" },
    { "collapsed" : true, "id" : "…UUID…", "kind" : "agent", "settings" : { "agent" : "ai-scout-1f2e…", "title" : "News", "widget" : "news" }, "side" : 1, "size" : "small" }
  ]
}
```

| Field | Values |
|---|---|
| `widgets[].kind` | `today`, `activity`, `inbox` (titled "Desk"), `routines`, `tokens`, `limits`, `shortcuts`, `agents`, `vault`, `agent`, `mail`, `schedule`; `news` and `capture` are retired but still decode |
| `widgets[].size` | `small` (1×1), `medium` (2×1), `wide` (4×1), `large` (2×2) |
| `widgets[].side` | `0` left, `1` right; absent means the kind's default: left for `today`, `inbox`, `routines`, `limits`, `tokens`, `schedule`, right for the rest |
| `widgets[].settings` | String map; agent widgets use `agent` (agent folder name), `widget` (file name without `.md`), `title` |
| `widgets[].x`, `y` | Retired free-placement position; read once by the side-column migration, then removed |
| `widgets[].collapsed` | Optional boolean |
| `theme` | Cockpit theme; default `space` |
| `widgetScale` | Optional, clamped to 0.75–1.5; offered steps 0.75, 0.85, 1, 1.15, 1.3, 1.5 |
| `shortcuts[]` | Skill shortcuts: `title`, `skill`, `agent` (reference), `instruction`, `webFetch` |

Standard layout, in order: today wide, activity large, inbox medium, routines medium, limits small, tokens small, agents medium, vault small, shortcuts small.

One-time layout migrations ClaudeDesk applies (the new app should apply the same, or rely on ClaudeDesk having done so):

- `cockpitMailWidgets.v1`: insert `mail` (small, right) and `schedule` (small, left) at the top when missing.
- Side columns: when any widget is `capture` or has `x` without `side`, drop `capture` widgets, give positioned widgets the side their `x` was on (split at half the screen width), sort by side then `y` (unpositioned widgets keep their order after positioned ones), and clear `x`/`y`.
- News: each `news` widget becomes an `agent` widget with settings `agent: <ai-scout folder>`, `widget: news`, `title: News`, or is removed when there is no agent matching `ai-scout`.

### 8.2 Plan limits and session cost (status line)

The app installs a Claude Code status line once into `~/.claude/settings.json` and each agent's `.runtime/settings.json` (tracked by setting `statusLineProfiles`, a list of settings file paths), only where no `statusLine` key exists:

```json
"statusLine" : {
  "command" : "'/Applications/ClaudeDesk.app/Contents/MacOS/claudedesk-runner' --statusline --workspace '/Users/me/Workspace'",
  "padding" : 0,
  "type" : "command"
}
```

Each status line update writes:

- `.claudedesk/usage/latest.json` when the input has `rate_limits.five_hour` or `rate_limits.seven_day`:

```json
{ "captured" : "2026-11-15T01:30:00Z", "fiveHour" : { "resetsAt" : "2026-11-15T05:00:00Z", "used" : 42 }, "sevenDay" : { "resetsAt" : "2026-11-20T00:00:00Z", "used" : 18 } }
```

- `.claudedesk/usage/sessions/<session_id>.json` when the session ID matches `^[A-Za-z0-9-]+$` and `cost.total_cost_usd` is present:

```json
{ "captured" : "2026-11-15T01:30:00Z", "costUSD" : 0.4213, "model" : "Opus", "session" : "6d1c…" }
```

and prints `<model> · ctx N% · 5h N% · 7d N%` (parts omitted when unknown).

### 8.3 Manual runs (shortcuts, widget buttons, Ask)

The Cockpit starts a background run right away (clicking is the approval: no Inbox approval and no daily cap) by writing `.claudedesk/runs/<runID>.request.json` and launching the runner with `--workspace <workspace> --run-request <file>`:

```json
{ "agent" : "ai-scout-1f2e…", "kind" : "shortcut", "request" : "", "runID" : "9a7c…", "skill" : "ai-tech-news", "title" : "News now", "webFetch" : true }
```

`kind`: `shortcut`, `capture`, `widget`, `ask`. The run uses the routine profile in 7.7 with routine name `quick-capture` (capture), `ask` (ask), the skill name, or `shortcut`; defaults of 30 turns and 15 minutes. Log events use the `run.` prefix instead of `routine.`. Prompt:

```
<opening>

Run: <title>. Run ID: <runID>. In inbox items you write, use `session: <runID>` and `via: <kind>`.

[Use the `<skill>` skill.]

[<request>, prefixed with "The user's request:\n" for capture or "The user's question:\n" for ask]
```

Opening for `ask`: "The user asked you this from the ClaudeDesk Cockpit. Your final message is shown to them as the answer, rendered as Markdown, so make it the answer itself: direct and concise, no summary of what you did. You can't ask questions back; if something needs their decision, say so in the answer. Read your memory and the Memory Vault as needed."

Otherwise: "This is a ClaudeDesk background run the user started from the Cockpit, not a conversation. Nobody is watching, so you can't ask questions. File anything that needs the user through the Inbox, then finish what you can. End with a short summary of what you did."

Stop: write the empty `<runID>.stop` marker, then send SIGTERM to the run's PID.

### 8.4 Activity and steps

Plain-language steps come from the last `tool_use` block of each `assistant` line in stream-json or a transcript: `Reading a/b`, `Writing a/b`, `Editing a/b` (last two path parts), `Searching files`, `Searching the web: <query>`, `Opening <host>`, `Running <first line, 48 chars>…`, `Using skill <name>`, `Delegating to a subagent`, `Using <server>` for `mcp__<server>__…`, `Using <tool>`; a text-only message is `Writing a reply`.

### 8.5 Token usage

Counted from transcripts (`*.jsonl`) under `~/.claude/projects/` (source "ClaudeDesk projects" when the session ID was started by the app according to the activity log, otherwise "Other Claude Code") and each agent's `.runtime/projects/` (source: the agent's name). Per line with `message.usage`, de-duplicated by `message.id`: tokens = `input_tokens + output_tokens + cache_creation_input_tokens + cache_read_input_tokens`. Costs add session costs (8.2) and the `total_cost_usd` of run reports in `.claudedesk/runs/*.json`. Windows: today, 7 days, 30 days.

## 9. Activity log (`logs/`)

[ActivityLog.swift] One JSON object per line, appended to `logs/<yyyy-MM>.jsonl` (month of the event, local calendar). Keys sorted, no pretty printing, slashes not escaped. Entries hold pointers, never content. Written only by the app and the runner.

```json
{"actor":"runner","details":{"agent":"8E1F…","cost":"0.0412","denials":"0","items":"","kind":"routine","output":"runs/0b6f….json","run":"0b6f…","seconds":"66","turns":"7"},"outcome":"ok","subject":"heartbeat","time":"2026-11-15T01:31:10Z","type":"routine.finished"}
```

| Key | Meaning |
|---|---|
| `time` | ISO 8601 |
| `type` | Event type (below) |
| `actor` | `user`, `app`, `runner`, `inbox`, an agent folder name, or `unknown` |
| `subject` | What it is about: a name, path or item file name |
| `outcome` | `ok` by default; otherwise an outcome such as `failed`, `problem`, `needs-approval` |
| `details` | String-to-string map |

Event types written by ClaudeDesk 3.8.1:

- workspace: `workspace.set`, `workspace.migrated`
- agents: `agent.created`, `agent.edited`, `agent.inbox-connected`, `agent.widgets-connected`, `agent.memory-connected` (details `change`: `added`, `repaired`, `unchanged`), `agent.memory-appended`
- conversations: `conversation.started`, `conversation.resumed`, `conversation.ended` (details include `pane`, `session`, and `project` or `agent`)
- inbox: `inbox.created`, `inbox.unreadable`, `inbox.approved`, `inbox.rejected`, `inbox.answered`, `inbox.done`, `inbox.callback`
- skills: `skill.created`, `skill.edited`, `skill.attached`, `skill.detached`, `skill.approved`
- routines: `routines.edited`, `routine.queued`, `routine.queue-skipped`, `routine.approved`, `routine.rejected`, `routine.paused`, `routine.invalid`, `routine.skipped`, `routine.cap-reached`, `routine.started`, `routine.finished`, `routine.failed`, `routine.timed-out`
- manual runs: `run.requested`, `run.stop-requested`, `run.started`, `run.finished`, `run.failed`, `run.timed-out`
- runner: `runner.enabled`, `runner.disabled`
- Cockpit: `cockpit.side-columns`, `cockpit.news-migrated`, `widget.checked`, `statusline.installed`, `statusline.skipped`
- mail (stage 4): `google.connected`, `google.disconnected`, `mac-mail.connected`, `mail.refreshed`

The Cockpit's routine history is rebuilt from `routine.started` (details `run`, `requested`: `schedule`, `user` or `inbox`) and the finish events (`seconds`, `cost`, `items`), over the last 7 days. `inbox.created` is logged the first time the app sees an item (setting `loggedInboxItems` holds the IDs of items still waiting that were already logged).

## 10. Agent widgets (`<agent>/widgets/`)

[AgentWidgets.swift, widget-instructions skill] Each widget is one `.md` file in the agent's `widgets/` folder (names starting with `.` ignored), with images in `widgets/assets/`. The widget's ID is `<agent folder name>/<file name without .md>`.

```markdown
---
title: News
icon: newspaper
width: medium
updated: 2026-10-24T08:00
---
![Top story](assets/hero.jpg)

### Headline
- Second story

<!-- full -->
## Full digest
…
```

- `title`: defaults to the file name with `-` as spaces, capitalised.
- `icon`: an SF Symbol name (**macOS**); unknown names fall back to `square.grid.2x2`. The new app needs a mapping or its own icon set (open question).
- `width`: `small` (270), `medium` (360), `wide` (460), or a number clamped to 220–640; default 270.
- Everything before `<!-- full -->` is the card; everything after shows only in focus mode.
- Images: `https://` URLs or paths relative to the widget file; `http://`, `file://` and absolute paths don't load. HTML is not rendered.
- Fenced blocks with `key: value` lines: `button` (`label`, `run`, `icon`, `skill`, `web`, `note`), `input` (`placeholder`, `label`, `run` with `{text}`), `stat` (`label`, `value`, `note`), `gauge` (`label`, `value`, `max` default 100, `note`), `chart` (`title`, `type` `bar`|`line`, `unit`, then `label: number` points).
- **Checklist ticks:** the app rewrites only the `[ ]`/`[x]` of the ticked line (by file line number), after checking that the line is still a checklist item; otherwise it refuses ("The widget changed before the tick was saved").
- **Buttons and inputs** start a manual run of kind `widget`, titled `<widget title> · <label>`, with this request:

```
The user clicked a control on your Cockpit widget “<title>” (file: `<path>`).
<instruction>

When the result belongs on the widget, update that file (see the `widget-instructions` skill). Keep the file valid; the Cockpit reloads it within seconds.
```

Widgets appear in the Cockpit's "+ WIDGET" menu until placed; placing one adds a `kind: agent` entry to `cockpit.json`.

## 11. Settings (macOS UserDefaults, domain `local.claudedesk.app`)

Not part of the workspace, listed so the new app can import them and keep its own equivalents.

| Key | Type | Meaning |
|---|---|---|
| `workspacePath` | string | The workspace folder |
| `agentHostPath` | string | Legacy agent folder (section 1.2) |
| `workspaceMigrationDeclined` | bool | |
| `claudeExecutable` | string | Override path to `claude` |
| `appMode` | string | `cockpit` (default) or `studio` |
| `deskTheme` | string | App theme; default `space` |
| `terminalFontSize` | number | Default 13 |
| `cockpitReduceMotion` | bool | |
| `cockpitGraphMode` | string | Vault graph style; default `flat` |
| `studioSidebarShown`, `studioSpaceTheme.v1` | | Studio view state |
| `workspaceConnectedAgents.v2` | [UUID] | Agents connected to Inbox and routines (3.5) |
| `workspaceConnectedAgents.widgets.v1` | [UUID] | Agents connected to widgets (3.6) |
| `workspaceConnectedAgents.memory.v1` | [UUID] | Agents given atrium-memory settings and instructions (3.7) |
| `statusLineProfiles` | [path] | Settings files the status line was installed in (8.2) |
| `loggedInboxItems` | [file name] | Inbox items already logged (9) |
| `cockpitMailWidgets.v1` | bool | Mail/Schedule widgets added once (8.1) |
| `mailSource`, `macMailConnected`, `googleAccounts`, `jevModel` | | Mail and calendar (stage 4) |

The UUIDs in the `workspaceConnectedAgents.*` lists are uppercase.

**Compatibility note:** these "done once" flags live outside the workspace. Every connection step is designed to be idempotent through its marker or by checking for existing values, so a second app that re-runs the steps for an agent ClaudeDesk already connected changes nothing, with one exception: if the user deliberately removed something the app added (a deny rule, a CLAUDE.md section), a second app running the first-time step would add it back. The new app should therefore treat an agent as already connected when its files show the connection (for example the marker in `CLAUDE.md`), not only when its own flag list says so.

## 12. Front matter rules

[Frontmatter.swift] Inbox items, skills and widgets share a small front matter dialect:

- The block starts at the very first line `---` and ends at the next line `---`. CRLF is read as LF.
- `key: value` lines at column 0. Empty lines and lines starting with `#` are skipped.
- `key:` with no value starts a block list; following lines `- item` (indented or not) are its items. An item `name: value` whose name matches `^[A-Za-z][A-Za-z0-9_-]*$` (and doesn't start with a quote) is a pair (used for callbacks); otherwise it is text.
- `[a, b, "c, d"]` is an inline list; commas inside quotes don't split.
- Values in `"…"` unescape `\n`, `\t`, `\"`, `\\`; values in `'…'` unescape `''`.
- Empty values read as absent.
- **Writing a value:** written plain when it matches `^[A-Za-z0-9_./@+-][A-Za-z0-9 _./@+:-]*$`, has no trailing space and no `": "`; otherwise in double quotes with `\`, `"` and newlines escaped.
- **Setting keys** in an existing block replaces a key's line (and any block list under it) in place; new keys are appended before the closing `---`. A file without front matter gets a new block at the top.

## 13. App-written markers

| Marker | Where | Section |
|---|---|---|
| `<!-- claudedesk:inbox -->` | agent `CLAUDE.md`, "## Inbox" | 3.5 |
| `<!-- claudedesk:routines -->` | agent `CLAUDE.md`, "## Routines" | 3.5 |
| `<!-- claudedesk:widgets -->` | agent `CLAUDE.md`, "## Cockpit widgets" | 3.6 |
| `<!-- claudedesk:memory-tool -->` | agent `CLAUDE.md`, "## Memory Vault tools" | 3.7 |
| `<!-- full -->` | widget files: card/focus split | 10 |
| ```` ```memory ```` | memory inbox items: text to append | 6.1 |
| `## Callbacks` | archived inbox items: callback results | 6.4 |
| `from: claudedesk` | inbox items written by the app or runner | 6.1 |
| `{{VAULT_PATH}}` | vault template placeholder, replaced at creation | 1.1 |

The new app keeps these exact markers, including the `claudedesk:` prefix, so that both apps recognise each other's work.

## 14. Open questions for the cross-platform app

1. **Windows paths in permission rules.** The vault edit rule is built as `Edit(/` + path + `/**)`. The correct Claude Code rule form for a Windows path (`C:\Users\…`) needs checking before stage 1 ships.
2. **App-written atrium-memory entry on Windows.** The test "`command` ends with `/atrium-memory`" must also accept `\atrium-memory.exe`, and the new app's bundled path must be recognised as app-written by both apps.
3. **Skill attachment on Windows.** Symlinks need Developer Mode or admin rights; junctions work for folders without either. ClaudeDesk's "attached" test resolves a symlink; a junction must also count.
4. **Folder permissions.** ClaudeDesk sets `0700` on agent folders; decide the Windows equivalent (likely none).
5. **Agent registry.** Discover agents from `agent_brain/*/agent.json` and keep a registry only as a cache, or keep a per-machine registry like ClaudeDesk.
6. **Dates in `agent.json`.** Swift writes seconds since 2001-01-01. The new app must read and write that form for these files (and projects) to stay compatible.
7. **Widget icons.** `icon` is an SF Symbol name; the new app needs an icon set with a mapping for common names.
8. **Status line and runner paths** point at ClaudeDesk's bundle. If the user switches apps, the settings written into profiles keep pointing at the old app's runner.
9. **Product text.** Agent-facing text says "ClaudeDesk" (instructions sections, prompts, bundled skills). Changing it in the new app changes the exact text that migrations match (3.7); decide whether the new app keeps the current wording.
