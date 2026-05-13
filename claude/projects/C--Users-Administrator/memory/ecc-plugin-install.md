---
name: ecc-plugin-install
description: "User has the ECC (everything-claude-code) plugin installed globally via the plugin marketplace path, plus common+typescript rules"
metadata: 
  node_type: memory
  type: project
  originSessionId: 666fdec7-1fd3-41ec-a05f-1bd51864c002
---

User installed `ecc@ecc` (https://github.com/affaan-m/everything-claude-code) globally on 2026-05-12:
- Marketplace registered at `~/.claude/plugins/marketplaces/ecc/`
- Plugin enabled in `~/.claude/settings.json` under `enabledPlugins`
- Rules manually copied: `~/.claude/rules/ecc/common` and `~/.claude/rules/ecc/typescript`
- Repo also cloned at `~/.claude/ecc-source/` (older clone, predates the plugin install)

**Why:** User wanted skills/commands/agents accessible from every Claude Code surface (VS Code, terminal, etc.) under the Windows account `Administrator`.

**How to apply:**
- Do NOT also run `install.ps1 --profile full` / `install.sh` / `npx ecc-install` — the README is explicit that stacking the plugin install on top of the manual installer creates duplicate skills and hooks. Reset path is `node ~/.claude/ecc-source/scripts/uninstall.js --dry-run` if duplication appears.
- If user adds another language, copy `~/.claude/plugins/marketplaces/ecc/rules/<lang>` into `~/.claude/rules/ecc/`. Don't copy from `~/.claude/ecc-source/` — that's the older clone.
- ECC commands like `/tdd`, `/plan`, `/code-review`, `/e2e`, `/learn`, `/skill-create` are available globally; suggest them when the task fits.
- TypeScript was the chosen language pack — frame language-specific suggestions accordingly unless user signals otherwise.
