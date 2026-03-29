---
name: skill-extractor
description: "Export any installed OpenClaw skill into a shareable ZIP: scrubs credential values, detects & stages external runtime files, generates STRUCTURE.md for LLM-guided install. Local only — no APIs, no tokens, no external calls."
metadata: {"openclaw":{"emoji":"📦"}}
---

# skill-extractor

Package any installed OpenClaw skill into a clean, shareable ZIP. Credentials are scrubbed. External runtime files referenced in SKILL.md are detected, staged, and documented so a new install knows exactly where every file belongs.

---

## Steps

### 1. List skills
Scan for dirs containing a `SKILL.md` inside:
- `<workspace>/skills/`
- `~/.openclaw/skills/`
- `~/AppData/Roaming/npm/node_modules/openclaw/skills/`

Present the list. Ask which skill to export (skip if already provided).

### 2. Stage skill files
Copy the skill directory to a temp staging dir at `<workspace>/.skill-export-staging/<skill-name>/`. Never touch the originals.

### 3. Detect & stage external files
Read the staged SKILL.md. Extract every path reference matching `~/.config/`, `$HOME/.config/`, `%APPDATA%/`, or `$env:APPDATA/`. Only include paths that contain the skill name (avoids false positives on generic system paths).

For each match: if the path exists on disk, copy it into `_external/` inside the staging dir, mirroring the original directory structure. If it doesn't exist yet (runtime-generated), note it as "created at runtime" — do not error.

Track a map of: staged path → original target path. This is used in STRUCTURE.md.

### 4. Scrub credentials
In all staged files (skill dir + `_external/`), find fields whose **name** matches any of: `token, secret, password, api_key, apikey, auth, bearer, jwt, access_key, private_key, client_secret, webhook, passphrase, pin, otp, seed, cert, credential, private`. Set their **value** to `""`. Apply to `.json`, `.env`, `.yaml`, `.yml`, `.toml`. Skip files that can't be parsed — warn but continue.

### 5. Generate STRUCTURE.md
Write `STRUCTURE.md` into the staging dir containing:

- **Folder layout** — ASCII tree of the staged dir
- **File descriptions table** — relative path + one-line purpose for every file
- **External files table** (if any external files were staged) — three columns: file in ZIP (`_external/...`) | install target path (`~/.config/...`) | notes (credential files: "fill in values"; worker scripts: "extracted from SKILL.md at runtime"; logs/pid/state: "recreated automatically")
- **Install instructions** — Option A (clawhub install), Option B (manual: copy skill dir + handle external files), Option C (local clawhub install). Include a short snippet showing how to copy `_external/` files to their target paths on Windows and macOS/Linux.
- **Credential note** — all values cleared; fill in before use; never commit to version control

### 6. Zip and deliver
Confirm the output path with the user (default: `~/Desktop/<skill-name>.zip`). Remove any existing ZIP at that path, compress the staging dir, report the file size, then delete the staging dir.

---

## Rules

- Never modify the source skill directory or any external files
- Scrub both the skill dir and `_external/` — not just one
- Only stage external files whose path contains the skill name
- If an external file doesn't exist, note it in STRUCTURE.md — don't fail
- If ZIP already exists at target, overwrite it
- If any step fails, leave staging intact and report clearly
- No API calls, no network, no external tools — fully local PowerShell only

---

## Errors

| Problem | Cause | Fix |
|---|---|---|
| Skill not found | Name mismatch | Check spelling; run `openclaw skills list` |
| Access denied | File ownership | Run as admin or check permissions |
| ZIP creation fails | PowerShell < 5 or disk full | Update PowerShell or free space |
| JSON parse error | Non-standard JSON | Scrubber skips it safely; inspect manually |
| Staging not cleaned | ZIP step failed | Delete `<workspace>/.skill-export-staging` manually |
| External file missing | Runtime-generated, not yet created | Safe to skip — document as "created at runtime" |
