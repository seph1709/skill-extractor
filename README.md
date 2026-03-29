# 📦 skill-extractor

> An [OpenClaw](https://openclaw.ai) skill that exports any installed skill into a clean, shareable ZIP — with credential scrubbing and an auto-generated `STRUCTURE.md` so anyone (or any LLM) can understand and reinstall it.

---

## What It Does

1. **Lists** all installed skills across your OpenClaw skill directories
2. **Copies** the selected skill to a temp staging area (source is never modified)
3. **Scrubs** credential values from `.json`, `.env`, `.yaml`, and `.toml` files — sensitive fields (tokens, secrets, passwords, API keys, etc.) are set to `""`
4. **Generates `STRUCTURE.md`** — a folder tree + per-file descriptions + install instructions, readable by both humans and LLM agents
5. **Zips** everything into a shareable `.zip` file
6. **Cleans up** staging automatically

---

## Install

```bash
clawhub install skill-extractor
```

Or manually — copy the `skill-extractor/` folder into your OpenClaw workspace `skills/` directory.

---

## Usage

Just ask your OpenClaw agent:

> "Export the `github` skill"
> "Use skill-extractor to package `gog`"
> "Zip up the `weather` skill so I can share it"

The agent will walk you through selection (if no skill is named), confirm the output path, and deliver a ready-to-share ZIP to your Desktop.

---

## Output Structure

The generated ZIP contains:

```
<skill-name>/
├── SKILL.md         ← original skill instructions (unchanged)
├── _meta.json       ← registry metadata (ownerId cleared)
├── STRUCTURE.md     ← auto-generated install guide ✨
└── ...              ← any other skill files, credentials scrubbed
```

### STRUCTURE.md preview

```markdown
# Skill: github

## Folder Layout
github/
├── SKILL.md
└── _meta.json

## File Descriptions
| File | Purpose |
|------|---------|
| `SKILL.md` | Main skill instructions. The LLM reads this... |
| `_meta.json` | ClawhHub registry metadata... |

## How to Install
### Option A — ClawhHub
clawhub install github
...
```

---

## Credential Scrubbing

Fields matching these patterns are zeroed out before packaging:

`token` · `secret` · `password` · `api_key` · `apikey` · `auth` · `bearer` · `jwt` · `access_key` · `private_key` · `client_secret` · `webhook` · `passphrase` · `pin` · `otp` · `seed` · `cert` · `credential`

Applies to: `.json` (deep recursive), `.env` (line-based), `.yaml` / `.toml` (line-based).

> The original skill directory is **never modified** — all scrubbing happens on a staging copy.

---

## Requirements

- PowerShell 5+ (Windows) or `pwsh` (macOS/Linux)
- [OpenClaw](https://openclaw.ai) with an agent that can run PowerShell

---

## Publishing Your Own Skills

Once you've exported a skill, you can publish it to ClawhHub:

```bash
clawhub publish ./skill-name --slug skill-name --name "My Skill" --version 1.0.0
```

---

## License

MIT — do whatever you want with it.
