# 📦 skill-extractor

> An [OpenClaw](https://openclaw.ai) skill that exports any installed skill into a clean, shareable ZIP — including external runtime files — with an auto-generated `STRUCTURE.md` so anyone (or any LLM) can understand and reinstall it correctly.

---

## What It Does

1. **Lists** all installed skills across your OpenClaw skill directories
2. **Stages** the selected skill to a temp folder — originals are never touched
3. **Detects external files** — scans the skill's `SKILL.md` for any file paths referenced outside the skill folder (config files, worker scripts, state files, etc.) and stages them under `_external/`
4. **Generates `STRUCTURE.md`** — folder tree, per-file descriptions, and an external files table that maps each file to its install target path
5. **Zips** everything into a shareable file and delivers it to your Desktop
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

> "Export the `facebook-page` skill"  
> "Use skill-extractor to package `gog`"  
> "Zip up the `weather` skill so I can share it"

The agent will walk you through selection (if no skill is named), confirm the output path, and deliver a ready-to-share ZIP to your Desktop.

---

## Output Structure

```
<skill-name>/
├── SKILL.md            ← skill instructions
├── _meta.json          ← registry metadata
├── STRUCTURE.md        ← auto-generated install guide
├── ...                 ← any other files in the skill folder
└── _external/          ← external runtime files (if any)
    └── .config/
        └── <skill>/
            ├── credentials.json
            └── worker.ps1
```

### What's inside STRUCTURE.md

- **Folder layout** — ASCII tree of the full package
- **File descriptions** — what each file does, described by purpose not type
- **External files table** — maps every file in `_external/` to where it should be placed on the target machine, with install notes
- **Install instructions** — three options: ClawhHub, manual, or local clawhub install

---

## Why External Files Matter

Many skills generate or depend on files that live outside the skill folder — worker scripts, config files, state trackers, and more. Without these, the skill won't work on a fresh machine.

skill-extractor finds every external path referenced in the skill's `SKILL.md`, copies them into the ZIP under `_external/`, and documents exactly where they need to go. The receiver gets a complete, self-documenting package with no guesswork.

---

## Use Cases

- **Sharing a skill** with a teammate — they get everything they need in one ZIP, with a clear install map
- **Backing up a skill** before making changes — full snapshot including all runtime files
- **Moving skills between machines** — no need to manually retrace config paths
- **Auditing a skill** — `STRUCTURE.md` shows every file the skill touches outside its own folder
- **Preparing for ClawhHub publishing** — reviewable snapshot before uploading

---

## License

MIT — do whatever you want with it.
