# New-PC Agent Runbook — Full Agent-Assisted Migration

> **For the Claude Code agent running on the NEW PC.** This supersedes the
> older `INSTRUCTIONS-FOR-CLAUDE-AGENT.md` (which only covered Claude/VS Code
> config). This runbook covers the FULL migration: apps, files, config, and
> credentials — everything except the OS itself.

## Source machine facts

- Hostname: `DESKTOP-9UPFD00`
- User profile: `C:\Users\3`
- OS: Windows 11 Pro
- Transfer channel: SMB network share over WiFi (see Phase 1)

## Hard rules

1. Never overwrite an existing file on the new PC without diffing and asking.
2. Read-only inspection is always fine without asking.
3. Treat everything under "Secrets" as sensitive — never commit it to any git
   repo, never print full key contents to the chat.
4. Verify each phase before moving to the next. Report what was skipped and why.

---

## Phase 0 — Pre-flight (read-only)

```powershell
hostname; whoami
claude --version
git --version; node --version; code --version
Test-Path "$env:USERPROFILE\.claude"
```

Record the new PC's username — you need it for the memory folder slug later.

---

## Phase 1 — Connect to the source machine's share

The old PC exposes a read-only share. Mount it:

```powershell
net use Z: \\DESKTOP-9UPFD00\Migration /user:DESKTOP-9UPFD00\3
# (prompts for the old PC's password)
Get-ChildItem Z:\
```

If the share is unreachable: confirm both PCs are on the same WiFi, network
discovery is on, and the old PC is awake.

---

## Phase 2 — Install apps

1. Copy the app inventory: `Copy-Item Z:\winget-apps.json $env:TEMP\`
2. Bulk-install: `winget import $env:TEMP\winget-apps.json --accept-package-agreements --accept-source-agreements`
3. Install the apps winget can't (confirm each with the user first):
   - **Git for Windows** — https://git-scm.com/download/win
   - **nvm-windows** — https://github.com/coreybutler/nvm-windows/releases
   - then: `nvm install 24.12.0; nvm use 24.12.0`
   - **Spotify**, **Realtek Audio** drivers — via vendor / Windows Update
4. Global npm packages:
   ```powershell
   npm install -g @angular/cli@21.0.4 appium@3.2.2 backstopjs@6.3.25 uipro-cli@2.2.3
   ```

---

## Phase 3 — Personal files

Copy each folder (use `robocopy /E` for resilience). Sizes from source:
Desktop ~2.3 GB, Downloads ~1.1 GB, others small.

```powershell
robocopy Z:\Desktop   "$env:USERPROFILE\Desktop"   /E
robocopy Z:\Downloads "$env:USERPROFILE\Downloads" /E
robocopy Z:\Documents "$env:USERPROFILE\Documents" /E
robocopy Z:\Pictures  "$env:USERPROFILE\Pictures"  /E
robocopy Z:\Videos    "$env:USERPROFILE\Videos"    /E
robocopy Z:\Music     "$env:USERPROFILE\Music"     /E
robocopy Z:\Favorites "$env:USERPROFILE\Favorites" /E
```

Also copy any code-project folders the user names.

---

## Phase 4 — Claude Code config

Copy these (diff first if they already exist):

| From (Z:) | To |
|-----------|-----|
| `Z:\.claude\settings.json` | `$env:USERPROFILE\.claude\settings.json` |
| `Z:\.claude\settings.local.json` | `$env:USERPROFILE\.claude\settings.local.json` |
| `Z:\.claude\rules\` | `$env:USERPROFILE\.claude\rules\` |
| `Z:\.claude\skills\` | `$env:USERPROFILE\.claude\skills\` |
| `Z:\.claude\.credentials.json` | `$env:USERPROFILE\.claude\.credentials.json` *(see note)* |

**Do NOT copy:** `.claude\cache`, `backups`, `sessions`, `shell-snapshots`,
`file-history`, `telemetry`, `metrics`, `daemon`, `ide`, `paste-cache`,
`history.jsonl`, `*.log` — machine-local state, regenerated automatically.

**Auto-memory:** source memory lives at `Z:\.claude\projects\C--Users-3\memory\`.
On the new PC the project slug differs (it's derived from the new username).
Find the new slug and copy memory into it:
```powershell
Get-ChildItem "$env:USERPROFILE\.claude\projects" -Directory   # find new slug
robocopy "Z:\.claude\projects\C--Users-3\memory" "$env:USERPROFILE\.claude\projects\<NEW-SLUG>\memory" /E
```

**Plugins:** don't copy `plugins\` — re-install via slash commands:
```
/plugin marketplace add https://github.com/affaan-m/everything-claude-code.git
/plugin install ecc@ecc
```

> **Note on `.credentials.json`:** copying it may carry the login over. If
> Claude Code rejects it on the new machine, just delete it and run `/login`.

---

## Phase 5 — Dev tool config & credentials (SENSITIVE)

Copy these. **Never commit any of them to a git repo.**

```powershell
Copy-Item Z:\.gitconfig "$env:USERPROFILE\.gitconfig"
robocopy Z:\.ssh        "$env:USERPROFILE\.ssh"        /E
robocopy Z:\.docker     "$env:USERPROFILE\.docker"     /E
robocopy Z:\.config     "$env:USERPROFILE\.config"     /E
robocopy Z:\.cagent     "$env:USERPROFILE\.cagent"     /E
robocopy Z:\.copilot    "$env:USERPROFILE\.copilot"    /E
robocopy Z:\.expo       "$env:USERPROFILE\.expo"       /E
robocopy Z:\.gateguard  "$env:USERPROFILE\.gateguard"  /E
```

After copying `.ssh`, fix its permissions (SSH refuses world-readable keys):
```powershell
icacls "$env:USERPROFILE\.ssh" /inheritance:r /grant:r "$($env:USERNAME):(F)"
```

---

## Phase 6 — VS Code & Windows Terminal

```powershell
# Extensions
code --install-extension anthropic.claude-code
code --install-extension github.vscode-github-actions
code --install-extension ms-azuretools.vscode-containers
code --install-extension ms-vscode-remote.remote-containers

# Settings (diff first if present) — copied from the source's live VS Code config
$vs = "$env:APPDATA\Code\User"
$src = "Z:\AppData\Roaming\Code\User"
Copy-Item "$src\settings.json"    "$vs\settings.json"
Copy-Item "$src\keybindings.json" "$vs\keybindings.json"
if (Test-Path "$src\snippets") { robocopy "$src\snippets" "$vs\snippets" /E }
```

Windows Terminal settings: copy from
`Z:\AppData\Local\Packages\Microsoft.WindowsTerminal_*\LocalState\settings.json`
to the same relative path on the new PC.

---

## Phase 7 — Things that do NOT transfer by copy (re-auth fresh)

Tell the user to do these manually — they are bound to the old hardware/account:

- **Windows Hello / PIN** — re-create on the new PC.
- **Tailscale** — install, sign in; the new PC gets its own IP. The old
  `100.121.110.58` stays with the old machine.
- **Browser passwords** — copying a browser profile folder does NOT carry
  saved passwords (they're encrypted with a machine-bound DPAPI key). Sign
  into Chrome/Edge with the user's account so sync restores them.
- **Windows Credential Manager** entries — DPAPI-bound; re-enter as needed.
- **GitHub CLI** — `gh auth login`.
- **Claude Code** — `/login` if `.credentials.json` didn't carry over.
- **VS Code Claude `webview/index.js` patch** — per-install; re-apply if used.

---

## Phase 8 — Verify

- [ ] `winget list` shows the migrated apps
- [ ] `node --version` → v24.12.0
- [ ] Personal files present in Desktop / Downloads / Documents
- [ ] Claude Code starts; `/doctor` clean; `/plugin` shows `ecc@ecc`; `/skills` populated
- [ ] `git config --global user.name` / `user.email` correct
- [ ] `ssh -T git@github.com` (if they use SSH) authenticates
- [ ] VS Code opens, Claude panel works
- [ ] Docker Desktop launches

When done, report to the user: what copied, what was skipped and why, and the
list of manual re-auth items from Phase 7 that still need their attention.

Then `net use Z: /delete` to unmount the share.
