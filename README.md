# Claude Code + VS Code — New Machine Migration Guide

This README captures the full configuration of your current Windows setup so you can reproduce (or improve on) it on a new computer. Follow the sections in order.

**Source machine snapshot taken:** 2026-05-13
**User:** <your-email>
**OS profile:** Windows 11 Home, user `Administrator`
**Shell:** PowerShell (primary), Bash via Git for Windows

---

## 1. Prerequisites — install these first

Install in this order on the new machine. Use the same versions or newer.

| Tool | Source machine version | Where to get it |
|------|------------------------|-----------------|
| Git for Windows | 2.52.0.windows.1 | https://git-scm.com/download/win |
| nvm-windows (nvm4w) | 1.2.2 | https://github.com/coreybutler/nvm-windows/releases |
| Node.js (via nvm) | 24.12.0 | `nvm install 24.12.0 && nvm use 24.12.0` |
| VS Code | 1.111.0 (x64) | https://code.visualstudio.com/ |
| Claude Code CLI | 2.1.140 | See section 3 |

After installing nvm-windows, open a **new** terminal as Administrator and run:

```powershell
nvm install 24.12.0
nvm use 24.12.0
node --version   # should print v24.12.0
```

Configure git identity (same as source):

```powershell
git config --global user.name  "Administrator"
git config --global user.email "<your-email>"
```

---

## 2. Global npm packages

Install the same global tooling that was on the old box:

```powershell
npm install -g @angular/cli@21.0.4
npm install -g appium@3.2.2
npm install -g backstopjs@6.3.25
npm install -g uipro-cli@2.2.3
# npm 11.6.2 and corepack 0.34.5 ship with Node 24.12.0 — no action needed
```

---

## 3. Install Claude Code

```powershell
# Official installer (Anthropic)
irm https://claude.ai/install.ps1 | iex
```

This installs `claude.exe` to `C:\Users\<you>\.local\bin\claude.exe`. Confirm:

```powershell
claude --version    # expect 2.1.140 or newer
```

---

## 4. Claude Code — global configuration

### 4a. `~/.claude/settings.json`

> **NOTE on the source file:** the live file on the old machine had a stray extra `}` at the end (malformed JSON, flagged by `/doctor`). The version below is the **corrected** one — use this verbatim on the new machine.

Create `C:\Users\<you>\.claude\settings.json` with exactly:

```json
{
  "enabledPlugins": {
    "ecc@ecc": true
  },
  "extraKnownMarketplaces": {
    "ecc": {
      "source": {
        "source": "git",
        "url": "https://github.com/affaan-m/everything-claude-code.git"
      }
    }
  },
  "autoUpdatesChannel": "latest",
  "skipDangerousModePermissionPrompt": true,
  "theme": "dark",
  "effortLevel": "max"
}
```

### 4b. `~/.claude/settings.local.json` (permission allowlist)

This file is the per-machine permissions allowlist that suppresses confirmation prompts for common read-only PowerShell / Bash commands. The full source file is included in the `claude-config-backup/` folder you should ship alongside this README (see section 9). On the new machine, copy it to `C:\Users\<you>\.claude\settings.local.json` as-is — it is safe to reuse because every entry is read-only.

Highlights of what it allows without prompting:
- `Test-Path`, `Get-ChildItem`, `Get-Process`, `Get-Service`, `Get-CimInstance`, `Get-ScheduledTask`, `Join-Path`, `Get-PSDrive`
- Various size/inventory queries against `Desktop`, `Downloads`, `Videos`, `Documents`
- `WebFetch(domain:github.com)`
- `Bash(where git *)`, `Bash(git --version)`, `Bash(git clone *)`, `Bash(where claude *)`, `Bash(where node *)`, `Bash(where code *)`

---

## 5. Claude Code — plugins

### Installed plugin

| Plugin | Version | Marketplace |
|--------|---------|-------------|
| `ecc@ecc` | 2.0.0-rc.1 | `ecc` (custom git marketplace) |

### Marketplaces registered

| Name | Source | URL / repo |
|------|--------|------------|
| `claude-plugins-official` | github | `anthropics/claude-plugins-official` |
| `ecc` | git | `https://github.com/affaan-m/everything-claude-code.git` |

### Install on new machine

After Claude Code is installed and `settings.json` is in place, launch Claude Code once so it reads `enabledPlugins`. Then run:

```
/plugin marketplace add https://github.com/affaan-m/everything-claude-code.git
/plugin install ecc@ecc
/plugin marketplace add anthropics/claude-plugins-official
```

The `ecc` plugin pulls in agents, slash commands, and rules (see next section). Do **NOT** also run `install.ps1` from the ecc repo — it will duplicate the install.

---

## 6. Claude Code — rules (global personal rulebook)

The rules directory `C:\Users\<you>\.claude\rules\ecc\` contains the ECC personal coding standards. The `ecc` plugin installs this automatically when activated, but if you want to seed it manually, the file list is:

**`rules/ecc/common/`** (10 files)
- `agents.md`            — agent orchestration & parallel task patterns
- `code-review.md`       — review checklist, severity levels, agent mapping
- `coding-style.md`      — immutability, KISS/DRY/YAGNI, naming, file size limits
- `development-workflow.md` — research → plan → TDD → review → commit
- `git-workflow.md`      — commit message format, PR workflow
- `hooks.md`             — PreToolUse / PostToolUse / Stop hooks, TodoWrite tips
- `patterns.md`          — skeleton projects, repository pattern, API envelope
- `performance.md`       — model selection (Haiku/Sonnet/Opus), context window mgmt
- `security.md`          — mandatory pre-commit checklist, secret management
- `testing.md`           — 80% coverage rule, TDD workflow, AAA test pattern

**`rules/ecc/typescript/`** (5 files)
- `coding-style.md`, `hooks.md`, `patterns.md`, `security.md`, `testing.md`

These get auto-loaded into Claude Code's context for every session via the ECC plugin.

---

## 7. Claude Code — skills

Skills currently installed under `C:\Users\<you>\.claude\skills\`:

```
accessibility
find-skills
learned/                            (empty — your own learned skills go here)
nextjs-app-router-patterns
owasp-security
react-performance-optimization
security-requirement-extraction
security-review
ui-ux-pro-max
web-design-guidelines
```

Most of these come from the ECC plugin and the official marketplace. After installing the plugins in section 5, run `/skills` inside Claude Code to verify they all show up. If any are missing, you can also copy the `skills/` folder from the backup bundle.

---

## 8. VS Code configuration

### 8a. Extensions installed

Install all four on the new machine:

```powershell
code --install-extension anthropic.claude-code
code --install-extension github.vscode-github-actions
code --install-extension ms-azuretools.vscode-containers
code --install-extension ms-vscode-remote.remote-containers
```

### 8b. User `settings.json`

Path: `%APPDATA%\Code\User\settings.json`

```json
{
    "claudeCode.preferredLocation": "panel",
    "claudeCode.allowDangerouslySkipPermissions": true,
    "claudeCode.useTerminal": true,
    "terminal.integrated.mouseWheelScrollSensitivity": 3
}
```

### 8c. User `keybindings.json`

Path: `%APPDATA%\Code\User\keybindings.json`

```json
[
    {
        "key": "shift+enter",
        "command": "workbench.action.terminal.sendSequence",
        "args": {
            "text": "\r"
        },
        "when": "terminalFocus"
    }
]
```

(This rebinds `Shift+Enter` in the integrated terminal to send `ESC` + `Return` — the standard escape-sequence Claude Code uses for newline insertion.)

---

## 9. What to copy from the old machine (the "bundle")

Create a folder `claude-config-backup/` on a USB / OneDrive / network share with the following structure, then copy it across:

```
claude-config-backup/
├── claude/
│   ├── settings.json              ← use the CORRECTED version from section 4a
│   ├── settings.local.json        ← copy from C:\Users\Administrator\.claude\settings.local.json
│   ├── plugins/
│   │   ├── installed_plugins.json
│   │   └── known_marketplaces.json
│   ├── rules/                     ← entire C:\Users\Administrator\.claude\rules\ tree
│   ├── skills/                    ← entire C:\Users\Administrator\.claude\skills\ tree
│   └── projects/                  ← OPTIONAL — your auto-memory lives here
│       └── C--Users-Administrator\memory\
│           └── MEMORY.md + ecc-plugin-install.md
└── vscode/
    ├── settings.json              ← from %APPDATA%\Code\User\
    ├── keybindings.json
    └── snippets/                  ← from %APPDATA%\Code\User\snippets\ (if any)
```

**Skip these (do NOT copy):**
- `~/.claude/cache/`, `~/.claude/backups/`, `~/.claude/sessions/`, `~/.claude/shell-snapshots/`, `~/.claude/file-history/`, `~/.claude/telemetry/`, `~/.claude/metrics/`, `~/.claude/debug/`, `~/.claude/history.jsonl`, `~/.claude/cost-tracker.log`, `~/.claude/bash-commands.log` — all machine-local state, regenerated automatically.
- `~/.claude/plugins/cache/` — Claude redownloads when you run `/plugin install`.
- `~/.claude/plugins/marketplaces/` — re-cloned when you add the marketplace.
- `~/.claude.json` — contains per-project state, machine IDs, and auth tokens. Don't transplant it; let Claude Code recreate it via login.
- VS Code `workspaceStorage/`, `globalStorage/`, `History/` — per-machine.

### Auto-memory (optional but recommended)

Your auto-memory currently has one entry: `ecc-plugin-install.md`. If you want the new machine to remember it, copy:

```
C:\Users\Administrator\.claude\projects\C--Users-Administrator\memory\
```

…to the **same relative location** on the new machine (replace `Administrator` with the new username in the folder name — Claude Code uses the folder name as a key, so adjust the slug accordingly).

---

## 10. Step-by-step on the new machine

1. **Install prereqs** (section 1) — Git, nvm, Node 24.12.0, VS Code.
2. **Set git identity** (section 1).
3. **Install global npm packages** (section 2).
4. **Install Claude Code CLI** (section 3), then run `claude` once and complete login.
5. **Drop in `claude/` config files** from the backup bundle into `C:\Users\<you>\.claude\`.
   - Use the CORRECTED `settings.json` from section 4a, not the old broken one.
6. **Start Claude Code**, then add marketplaces and install the plugin (section 5).
7. **Install VS Code extensions** (section 8a) and copy VS Code `settings.json` + `keybindings.json` (sections 8b/8c).
8. **Authenticate Claude Code** (`/login` inside Claude or the browser flow on first launch).
9. **Run `/doctor`** inside Claude Code — should report no issues.
10. **Smoke test:** open a project, run `/plugin` to verify `ecc@ecc` is enabled, run `/skills` to verify skills are listed.

---

## 11. Login & authentication

- **Anthropic / Claude Code login:** triggered automatically on first `claude` run, or via `/login`. Use the same account (<your-email>).
- **GitHub auth:** if you use the `gh` CLI, install GitHub CLI and run `gh auth login`. Tokens are not portable — re-auth on the new machine.
- **SSH keys:** if you use SSH for git, copy `C:\Users\Administrator\.ssh\` (treat as sensitive — encrypt during transfer).

---

## 12. Improvements worth making on the new machine

Things you can do better while you're setting up fresh:

- **Fix the malformed `settings.json`** — already done in this README's section 4a, but worth noting: the old file had an extra `}` at the end. The version here is clean.
- **Consider Windows Terminal** if you don't already use it — better tab + profile UX than the default PowerShell window.
- **Pin Node version with `.nvmrc`** in projects so future you doesn't fight version drift.
- **Enable Windows Developer Mode** (`Settings → Privacy & security → For developers`) — lets you create symlinks without admin and speeds up some tooling.
- **Long path support:** enable `LongPathsEnabled` in the registry if you'll be working with deep `node_modules` trees. PowerShell admin:
  ```powershell
  New-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\FileSystem" -Name "LongPathsEnabled" -Value 1 -PropertyType DWORD -Force
  ```
- **Back up `~/.claude/projects/`** to a cloud folder so your auto-memory survives across reinstalls automatically.

---

## 13. Verification checklist

After migration, confirm each of these in order. If any fails, jump back to the matching section.

- [ ] `git --version` → 2.52.x or newer (section 1)
- [ ] `node --version` → v24.12.0 (section 1)
- [ ] `npm ls -g --depth=0` shows angular, appium, backstopjs, uipro-cli (section 2)
- [ ] `claude --version` → 2.1.140 or newer (section 3)
- [ ] `code --list-extensions` shows all 4 extensions (section 8a)
- [ ] Claude Code starts without errors; `/doctor` reports no issues (section 4)
- [ ] `/plugin` shows `ecc@ecc` as enabled (section 5)
- [ ] `/skills` lists the expected skills (section 7)
- [ ] In VS Code, Claude Code panel opens via the extension and `Shift+Enter` in terminal works (section 8)
- [ ] Test a tiny git clone + commit to confirm git identity is set correctly

---

## Appendix A — file location quick reference

| Purpose | Path |
|---------|------|
| Claude global settings | `C:\Users\<you>\.claude\settings.json` |
| Claude permission allowlist | `C:\Users\<you>\.claude\settings.local.json` |
| Claude plugins state | `C:\Users\<you>\.claude\plugins\` |
| Claude rules | `C:\Users\<you>\.claude\rules\` |
| Claude skills | `C:\Users\<you>\.claude\skills\` |
| Claude auto-memory | `C:\Users\<you>\.claude\projects\<project-slug>\memory\` |
| Claude CLI binary | `C:\Users\<you>\.local\bin\claude.exe` |
| VS Code settings | `%APPDATA%\Code\User\settings.json` |
| VS Code keybindings | `%APPDATA%\Code\User\keybindings.json` |
| Git global config | `C:\Users\<you>\.gitconfig` |
| nvm-windows | `C:\nvm4w\` |
| Node (active) | `C:\nvm4w\nodejs\node.exe` |

---

That's the whole setup. Hand this README + the `claude-config-backup/` folder to the new machine and you're done.
