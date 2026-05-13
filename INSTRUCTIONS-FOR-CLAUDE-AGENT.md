# Instructions for the Claude Agent on the New Machine

> **You are reading this on a fresh Windows machine.** The user has just installed Claude Code and is asking you to migrate their previous configuration into place. Follow this document carefully — it is the operational runbook for the migration. The companion `README.md` in this same folder is the human-facing reference. Use it for context when the user asks "why are we doing X."

---

## Your job, in one sentence

Take the files in this repository and restore them to the correct locations on the new machine so the user's Claude Code + VS Code environment behaves identically to the source machine.

---

## Hard rules

1. **Never run destructive commands without confirming with the user.** "Destructive" includes overwriting existing files in `~/.claude/`, removing folders, modifying global config, or running uninstallers.
2. **Stop and ask** if any file you are about to write already exists on the new machine. Diff first; let the user decide.
3. **Read-only inspection is fine** without asking — `Test-Path`, `Get-Content`, `git status`, `claude --version`, etc.
4. **Do not commit secrets back to this repo.** If the user pastes API keys, tokens, or SSH keys during setup, route them into env vars or the appropriate config location — never into a tracked file.
5. **Treat `settings.local.json` as a personal allowlist from a different machine.** Most entries reference Administrator's specific Desktop folders. They are harmless but irrelevant on the new machine. Offer to keep them or trim them — let the user choose.

---

## Pre-flight: confirm context before touching anything

Run these in order. Each is read-only.

```powershell
# 1. Confirm we're on the new machine
hostname
whoami

# 2. Confirm Claude Code is installed and recent
claude --version

# 3. Check what already exists at the destination
Test-Path "$env:USERPROFILE\.claude"
Get-ChildItem "$env:USERPROFILE\.claude" -Force -ErrorAction SilentlyContinue | Select-Object Name

# 4. Confirm prereqs
git --version
node --version
code --version

# 5. Confirm this repo is cloned locally and what's in it
Get-ChildItem . -Force
```

**Decision point:**
- If `~/.claude/` already has content (it will, even on a fresh install — Claude creates state files on first launch), **DO NOT blindly overwrite.** Tell the user what's there and ask before each copy.
- If prereqs are missing, follow `README.md` section 1 to install them first.

---

## Migration steps (in order)

### Step 1 — Prereqs

If `git`, `node`, `code`, or `claude` is missing, install per `README.md` section 1. Stop and verify versions before continuing.

### Step 2 — Git identity

Ask the user for their git email (it was scrubbed from the README for privacy). Then:

```powershell
git config --global user.name  "<their-name-or-username>"
git config --global user.email "<their-email>"
```

### Step 3 — Global npm packages

Per `README.md` section 2. Install each one, confirming the user actually wants each. Skip any they say they no longer need.

### Step 4 — Claude Code global settings

Source files in this repo: `claude/settings.json` and `claude/settings.local.json`.

**Workflow:**

```powershell
# Show the user what's already there (if anything)
$dest = "$env:USERPROFILE\.claude\settings.json"
if (Test-Path $dest) {
    Write-Output "EXISTING settings.json:"
    Get-Content $dest
    Write-Output "---"
    Write-Output "INCOMING settings.json (from repo):"
    Get-Content ".\claude\settings.json"
    # ASK USER: replace, merge, or skip?
} else {
    Copy-Item ".\claude\settings.json" $dest
}
```

Repeat the same flow for `settings.local.json`. For that file specifically, **flag to the user** that the allowlist entries reference paths from the old machine (`Desktop\random`, `Desktop\games`, `Videos\NVIDIA`, `ParkControl`, etc.). Ask whether they want the full file or a trimmed version with only the universally-useful entries.

### Step 5 — Plugins and marketplaces

Source files: `claude/plugins/installed_plugins.json`, `claude/plugins/known_marketplaces.json`.

**Do not just copy these files.** Claude Code regenerates them when you install plugins via slash commands. Instead, launch Claude Code and run:

```
/plugin marketplace add https://github.com/affaan-m/everything-claude-code.git
/plugin marketplace add anthropics/claude-plugins-official
/plugin install ecc@ecc
```

Then confirm with `/plugin` that `ecc@ecc` shows up enabled.

### Step 6 — Rules

Source: `claude/rules/ecc/` (common + typescript subfolders, 15 markdown files total).

The `ecc@ecc` plugin should install rules automatically when activated. **Verify first:**

```powershell
Test-Path "$env:USERPROFILE\.claude\rules\ecc\common\agents.md"
```

If the plugin's automatic install put them there, you're done. If not, copy them manually:

```powershell
$rulesDest = "$env:USERPROFILE\.claude\rules"
New-Item -ItemType Directory -Force -Path $rulesDest | Out-Null
Copy-Item ".\claude\rules\*" $rulesDest -Recurse
```

### Step 7 — Skills

Source: `claude/skills/` (9 skill folders).

Same logic as rules — the plugin should install most of them. Run `/skills` inside Claude Code to verify. For any that are missing:

```powershell
$skillsDest = "$env:USERPROFILE\.claude\skills"
New-Item -ItemType Directory -Force -Path $skillsDest | Out-Null
# Copy ONLY the missing skill folders, not the whole skills/ directory
Copy-Item ".\claude\skills\<skill-name>" $skillsDest -Recurse
```

### Step 8 — Auto-memory (optional)

Source: `claude/projects/C--Users-Administrator/memory/`.

This folder is keyed by the Windows username on the **source** machine. On the new machine, Claude derives a different slug from the new username. So:

```powershell
# Find what slug Claude is using on the new machine
Get-ChildItem "$env:USERPROFILE\.claude\projects" -Directory | Select-Object Name
```

Once you know the new slug (e.g., `C--Users-NewName`), copy the memory folder into it:

```powershell
$newSlug = "<the-slug-you-just-found>"
$memDest = "$env:USERPROFILE\.claude\projects\$newSlug\memory"
New-Item -ItemType Directory -Force -Path $memDest | Out-Null
Copy-Item ".\claude\projects\C--Users-Administrator\memory\*" $memDest
```

**Heads up:** `ecc-plugin-install.md` in the memory folder references the Windows account `Administrator`. If the new username is different, ask the user whether to update that reference or leave it as historical context.

### Step 9 — VS Code

Per `README.md` section 8.

```powershell
# Install extensions
code --install-extension anthropic.claude-code
code --install-extension github.vscode-github-actions
code --install-extension ms-azuretools.vscode-containers
code --install-extension ms-vscode-remote.remote-containers

# Copy settings (diff first if a file exists)
$vsUser = "$env:APPDATA\Code\User"
Copy-Item ".\vscode\settings.json" "$vsUser\settings.json"
Copy-Item ".\vscode\keybindings.json" "$vsUser\keybindings.json"
```

### Step 10 — Verification

Run the checklist in `README.md` section 13. If any item fails, jump back to the corresponding section. Report the final state to the user.

---

## Things you should proactively offer

After the basic migration is done, ask the user about these enhancements (don't do them unprompted):

- Enable Windows long-path support (registry change — needs admin).
- Set up Windows Terminal as the default shell.
- Configure auto-backup of `~/.claude/projects/` so auto-memory survives reinstalls.
- Pin Node version via `.nvmrc` in their active projects.

---

## When things go wrong

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| `/doctor` reports malformed `settings.json` | A merge/edit introduced a syntax error | Read it, find the unmatched brace/comma, fix it. The original repo version is known-good. |
| `/plugin install ecc@ecc` fails | Marketplace not registered or network/auth issue | Re-run `/plugin marketplace add <url>`, then retry install. |
| Skills don't appear in `/skills` | Plugin activation incomplete | Restart Claude Code. If still missing, copy from `claude/skills/` manually. |
| VS Code Claude panel won't open | Extension version mismatch or login expired | `code --list-extensions --show-versions`, update `anthropic.claude-code`, then `/login` inside Claude Code. |
| Auto-memory not loaded | Wrong project slug folder | Re-check the username-derived slug under `~/.claude/projects/` and move the `memory/` folder accordingly. |

---

## When you're done

Report to the user:

1. What was installed / copied / configured.
2. What was **skipped** and why (e.g., "did not overwrite existing settings.json — user kept their version").
3. The output of `/doctor` (should be clean).
4. Anything that needs the user's manual attention (login flows, SSH key setup, secrets they need to recreate).

Keep the report short. The user wants to know the migration is done, not relive every step.
