# 📦 skill-extractor

> **Pack any OpenClaw skill into a ZIP anyone can install.**
> It finds every file the skill needs — including ones outside the skill folder — puts them all in one place, and generates a guide that tells the receiver exactly where each file goes.

---

## Use Cases

**Sharing a skill with someone else**
You built a skill. It works on your machine. skill-extractor bundles the skill folder, all its runtime files, and a step-by-step install guide into a single ZIP. Your teammate unzips it, reads the guide, drops the files in the right places — done.

**Backing up before making changes**
Export a full snapshot of a skill — including every external config, worker script, and state file — before you start breaking things.

**Moving a skill to another machine**
No more manually retracing which files live where. The ZIP carries everything and the install guide tells you exactly where to put it.

**Auditing what a skill touches**
The auto-generated `STRUCTURE.md` inside the ZIP lists every file the skill references outside its own folder, with a plain-English description of what each one does.

---

## How It Works

1. You tell it which skill to export
2. It scans the skill's instructions for any external file references and shows you what it found
3. You confirm — nothing is packaged without your approval
4. It bundles everything into a ZIP and saves it to your Desktop

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
> "Package `gog` so I can share it"
> "Zip up `weather` for backup"

---

## What's Inside the ZIP

```
<skill-name>/
├── SKILL.md            ← skill instructions
├── _meta.json          ← registry metadata
├── STRUCTURE.md        ← install guide (auto-generated)
├── ...                 ← any other skill files
└── _external/          ← external runtime files, if any
    └── .config/
        └── <skill>/
            ├── credentials.json
            └── worker.ps1
```

`STRUCTURE.md` tells the receiver:
- What every file does
- Where every external file should be placed on their machine
- How to install (ClawhHub, manual, or local)

> Files are packaged as-is. If a skill has credentials or tokens, they will be in the ZIP. Review before sharing.

---

## License

MIT — do whatever you want with it.
