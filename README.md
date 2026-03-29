# 📦 skill-extractor

> **Pack any OpenClaw skill into a ZIP anyone can install.**
> It finds every file the skill needs — including ones outside the skill folder — puts them all in one place, and generates a guide that tells the receiver exactly where each file goes.

---

## Why not just use `clawhub install` or copy the folder manually?

Because most skills are more than just their skill folder.

A skill that runs a background listener, talks to an API, or tracks state will have files living elsewhere on the machine — a config file in a user directory, a worker script, a credentials file. `clawhub install` and a manual folder copy only get you the skill folder. Those external files don't come along, and there's no record of where they're supposed to go.

The result: the skill installs fine but doesn't work. The receiver has to dig through the SKILL.md, figure out what files are missing, guess the right paths, and set everything up from scratch.

skill-extractor solves this by scanning the skill's own instructions for every external file it references, bundling them into the ZIP alongside the skill folder, and generating a plain-English guide that maps each file to its exact install location. The receiver gets a complete, self-documenting package — no guesswork.

---

## The Use Case

You built a skill that monitors a Facebook Page inbox and forwards messages to a Telegram channel. It works perfectly on your machine. A teammate wants it.

You run skill-extractor. It finds:
- The skill folder (`SKILL.md`, `_meta.json`)
- The credentials file sitting in your config directory
- The worker script that runs in the background
- The state file that tracks which messages have been forwarded

It shows you the full list, warns you that real values will be included, and asks for confirmation. You approve. Everything goes into a ZIP with a `STRUCTURE.md` that tells your teammate: *this file is the worker script — put it here. This is the credentials file — put it here and fill in your own values.*

Your teammate unzips, reads the guide, places the files, fills in their credentials — skill works on the first try.

Without skill-extractor, they'd get the skill folder, hit errors, and spend time asking you what's missing and where it goes.

---

## Install

```bash
clawhub install skill-extractor
```

Or manually — copy the `skill-extractor/` folder into your OpenClaw workspace `skills/` directory.

---

## Usage

Ask your OpenClaw agent:

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

> Files are packaged as-is. If a skill has credentials or tokens, they will be in the ZIP. Review before sharing.

---

## License

MIT — do whatever you want with it.
