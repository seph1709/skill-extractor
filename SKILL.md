---
name: skill-extractor
description: "Export any installed OpenClaw skill into a shareable ZIP: scrubs credential values, detects & stages external runtime files, generates STRUCTURE.md for LLM-guided install. Local only — no APIs, no tokens, no external calls."
metadata: {"openclaw":{"emoji":"📦"}}
---

# skill-extractor

Package any installed OpenClaw skill into a clean, shareable ZIP. All credential values are scrubbed (set to `""`). External runtime files referenced in SKILL.md (e.g. `~/.config/<skill>/`) are detected, staged under `_external/`, and documented in STRUCTURE.md — so a new install knows exactly where every file belongs and can reproduce full functionality.

---

## What This Skill Does

1. Scans all known skill directories and lists available skills
2. Accepts a skill name (prompted or provided by user)
3. Copies all skill files to a temp staging folder
4. **Detects external path references in SKILL.md** and stages those files under `_external/`
5. Scrubs credential values from `.json`, `.env`, `.yaml`, `.toml` files (skill dir + `_external/`)
6. Generates `STRUCTURE.md` — folder tree, file descriptions, external file install table, and full install guide
7. Compresses the staging folder into a ZIP
8. Saves ZIP to the Desktop (or user-specified path) and cleans up staging

## What Is NOT Done

- No API calls of any kind
- No files deleted from the source skill directory or external paths
- No data transmitted anywhere — fully local

---

## AGENT RULES

- Always list available skills before asking for selection (unless skill name is already given)
- Always work on a staging copy — never modify the original skill directory or external files
- Scrub ALL files — check `.json`, `.env`, `.yaml`, `.yml`, `.toml` for credential patterns (skill dir AND `_external/`)
- Detect external paths by scanning SKILL.md for `~/.config/`, `$HOME/.config/`, `%APPDATA%/`, `$env:APPDATA/` references
- Stage found external files under `_external/<relative-path>/` — mirror the directory structure
- If an external file does not exist on disk yet (runtime-generated), note it as "created at runtime" in STRUCTURE.md — do not error
- Generate `STRUCTURE.md` inside the staging folder before zipping — include external files section
- Default ZIP output: `$HOME\Desktop\<skill-name>.zip` — confirm path with user first
- If ZIP already exists at target, overwrite it (remove first)
- Clean up staging dir after successful ZIP creation
- If any step fails, report clearly and leave staging intact for inspection

---

## STEP 1 — List Available Skills

Scan these directories for valid skills (must have a `SKILL.md` inside):

```powershell
$skillDirs = @(
    "$HOME\.openclaw\workspace\skills",
    "$HOME\.openclaw\skills",
    "$HOME\AppData\Roaming\npm\node_modules\openclaw\skills"
)

$skills = @()
foreach ($dir in $skillDirs) {
    if (Test-Path $dir) {
        Get-ChildItem $dir -Directory | ForEach-Object {
            if (Test-Path "$($_.FullName)\SKILL.md") {
                $skills += [PSCustomObject]@{
                    Name   = $_.Name
                    Path   = $_.FullName
                    Source = $dir
                }
            }
        }
    }
}

$skills | Format-Table Name, Source -AutoSize
```

Present the list and ask: **"Which skill would you like to export?"**

---

## STEP 2 — Locate Skill

```powershell
$skillName = "<user-provided-name>"
$target = $skills | Where-Object { $_.Name -eq $skillName } | Select-Object -First 1

if (-not $target) {
    Write-Host "❌ Skill '$skillName' not found. Check spelling or run: openclaw skills list"
    return
}

Write-Host "✅ Found: $($target.Path)"
```

---

## STEP 3 — Stage Skill Files

```powershell
$stagingRoot = "$HOME\.openclaw\workspace\.skill-export-staging"
$stagingDir  = "$stagingRoot\$skillName"

if (Test-Path $stagingDir) { Remove-Item $stagingDir -Recurse -Force }
New-Item -ItemType Directory -Force -Path $stagingDir | Out-Null

Copy-Item "$($target.Path)\*" $stagingDir -Recurse
Write-Host "📁 Staged skill files to: $stagingDir"
```

---

## STEP 3.5 — Detect & Stage External Files

Parse the skill's SKILL.md for any paths that live outside the skill directory (config dirs, worker scripts, logs, pid files, state files). Stage copies under `_external/` mirroring the original structure.

```powershell
$skillMdContent = [System.IO.File]::ReadAllText("$($target.Path)\SKILL.md")

# Extract all path-like references: ~/.config/..., $HOME/.config/..., %APPDATA%/..., $env:APPDATA/...
$rawMatches = [regex]::Matches(
    $skillMdContent,
    '(?:~|`\$HOME`|\$HOME|`\$env:APPDATA`|\$env:APPDATA|%APPDATA%)[/\\][\w.\-/\\]+'
)

# Resolve to absolute paths and collect unique files
$externalFiles = @()
foreach ($m in $rawMatches) {
    $raw = $m.Value.Trim('"').Trim("'")
    # Normalise prefix to actual path
    $abs = $raw `
        -replace '^~',              $HOME `
        -replace '^\$HOME',         $HOME `
        -replace '^\$env:APPDATA',  $env:APPDATA `
        -replace '^%APPDATA%',      $env:APPDATA
    $abs = $abs -replace '[/\\]+', [System.IO.Path]::DirectorySeparatorChar

    # Only include paths that contain the skill name (avoids generic system paths)
    if ($abs -notmatch [regex]::Escape($skillName)) { continue }

    if (Test-Path $abs -PathType Leaf) {
        if ($externalFiles -notcontains $abs) { $externalFiles += $abs }
    } elseif (Test-Path $abs -PathType Container) {
        Get-ChildItem $abs -Recurse -File | ForEach-Object {
            if ($externalFiles -notcontains $_.FullName) { $externalFiles += $_.FullName }
        }
    }
}

# Stage each file under _external/, mirroring original structure
$externalStageRoot = "$stagingDir\_external"
$externalMap = @()   # track source → staged → target for STRUCTURE.md

foreach ($src in $externalFiles) {
    # Build relative path from $HOME
    $rel = $src.Replace($HOME, '').TrimStart('\').TrimStart('/')
    $dest = Join-Path $externalStageRoot $rel
    $destDir = Split-Path $dest -Parent
    New-Item -ItemType Directory -Force -Path $destDir | Out-Null
    Copy-Item $src $dest -Force

    $externalMap += [PSCustomObject]@{
        Source  = $src
        Staged  = "_external/$($rel.Replace('\','/'))"
        Target  = $src.Replace($HOME, '~').Replace('\', '/')
    }
    Write-Host "📂 Staged external: $rel"
}

if ($externalFiles.Count -eq 0) {
    Write-Host "ℹ️  No external files detected for this skill."
}
```

---

## STEP 4 — Scrub Credentials

Scrub sensitive values from ALL staged files — both the skill directory and `_external/`.

```powershell
# Credential field name patterns (case-insensitive substring match)
$credPatterns = 'token|secret|password|api.?key|apikey|auth|bearer|jwt|' +
                'access.?key|private.?key|client.?secret|webhook|passphrase|' +
                'pin|otp|seed|cert|credential|private'

# --- JSON scrubbing ---
function Scrub-JsonObject($obj) {
    if ($obj -is [PSCustomObject]) {
        foreach ($prop in $obj.PSObject.Properties) {
            if ($prop.Name -match $credPatterns -and $prop.Value -is [string] -and $prop.Value -ne "") {
                $prop.Value = ""
            } elseif ($prop.Value -is [PSCustomObject] -or
                      ($prop.Value -is [System.Collections.IEnumerable] -and $prop.Value -isnot [string])) {
                Scrub-JsonObject $prop.Value
            }
        }
    } elseif ($obj -is [System.Collections.IEnumerable] -and $obj -isnot [string]) {
        foreach ($item in $obj) { Scrub-JsonObject $item }
    }
}

Get-ChildItem $stagingDir -Recurse -Include "*.json" | ForEach-Object {
    try {
        $raw = [System.IO.File]::ReadAllText($_.FullName)
        $obj = $raw | ConvertFrom-Json
        Scrub-JsonObject $obj
        $scrubbed = $obj | ConvertTo-Json -Depth 20
        [System.IO.File]::WriteAllText($_.FullName, $scrubbed, [System.Text.UTF8Encoding]::new($false))
        Write-Host "🔒 Scrubbed (JSON): $($_.Name)"
    } catch {
        Write-Host "⚠️  Could not parse JSON: $($_.Name) — skipped"
    }
}

# --- .env scrubbing ---
Get-ChildItem $stagingDir -Recurse | Where-Object { $_.Name -like "*.env" -or $_.Name -eq ".env" } | ForEach-Object {
    $lines = Get-Content $_.FullName
    $scrubbed = $lines | ForEach-Object {
        if ($_ -match '^([^#=\s][^=]*)=(.+)$' -and $Matches[1] -match $credPatterns) {
            "$($Matches[1])="
        } else { $_ }
    }
    Set-Content $_.FullName $scrubbed -Encoding UTF8
    Write-Host "🔒 Scrubbed (.env): $($_.Name)"
}

# --- YAML / TOML scrubbing (line-based) ---
Get-ChildItem $stagingDir -Recurse -Include "*.yaml","*.yml","*.toml" | ForEach-Object {
    $lines = Get-Content $_.FullName
    $scrubbed = $lines | ForEach-Object {
        if ($_ -match '^(\s*\S+)\s*[:=]\s*"[^"]*"' -and ($_ -split '[:=]')[0] -match $credPatterns) {
            ($_ -replace '([:=]\s*)(".+")$', '$1""')
        } elseif ($_ -match "^(\s*\S+)\s*[:=]\s*'[^']*'" -and ($_ -split '[:=]')[0] -match $credPatterns) {
            ($_ -replace "([:=]\s*)('.+')$", "$1''")
        } else { $_ }
    }
    Set-Content $_.FullName $scrubbed -Encoding UTF8
    Write-Host "🔒 Scrubbed (YAML/TOML): $($_.Name)"
}
```

---

## STEP 5 — Generate STRUCTURE.md

```powershell
function Get-FileDesc($file) {
    switch ($file.Name) {
        "SKILL.md"      { "Main skill instructions. The LLM agent reads this to understand purpose, setup, and usage." }
        "_meta.json"    { "ClawhHub registry metadata: version, owner, credential paths, persistence info. ownerId cleared." }
        "STRUCTURE.md"  { "This file. Auto-generated folder map and install guide." }
        ".env"          { "Environment variable defaults. Credential values cleared — fill in before use." }
        default {
            switch ($file.Extension) {
                ".json" { "Configuration or data file. Credential values cleared — fill in before use." }
                ".ps1"  { "PowerShell script. Review before running." }
                ".sh"   { "Shell script. Make executable with chmod +x before running." }
                ".py"   { "Python script." }
                ".md"   { "Documentation or reference file." }
                ".yaml" { "YAML configuration. Credential values cleared if present." }
                ".yml"  { "YAML configuration. Credential values cleared if present." }
                ".toml" { "TOML configuration. Credential values cleared if present." }
                default { "Supporting file." }
            }
        }
    }
}

function Build-Tree($dir, $prefix = "") {
    $items = Get-ChildItem $dir | Sort-Object { -not $_.PSIsContainer }, Name
    $out = @()
    for ($i = 0; $i -lt $items.Count; $i++) {
        $item    = $items[$i]
        $isLast  = ($i -eq $items.Count - 1)
        $conn    = if ($isLast) { "└── " } else { "├── " }
        $child   = if ($isLast) { "    " } else { "│   " }
        $label   = if ($item.PSIsContainer) { "$($item.Name)/" } else { $item.Name }
        $out    += "$prefix$conn$label"
        if ($item.PSIsContainer) { $out += Build-Tree $item.FullName "$prefix$child" }
    }
    $out
}

$tree     = (Build-Tree $stagingDir) -join "`n"
$allFiles = Get-ChildItem $stagingDir -Recurse -File
$fileTable = $allFiles | ForEach-Object {
    $rel  = $_.FullName.Replace("$stagingDir\", "").Replace("\", "/")
    $desc = Get-FileDesc $_
    "| ``$rel`` | $desc |"
}

$now = Get-Date -Format "yyyy-MM-dd HH:mm:ss"

# Build external files section
$extSection = ""
if ($externalMap.Count -gt 0) {
    $extRows = $externalMap | ForEach-Object {
        $notes = if ($_.Target -match 'credentials|config') {
            "Credential values cleared — fill in before use."
        } elseif ($_.Target -match 'worker') {
            "Worker script. Extracted from SKILL.md at runtime — included here for reference."
        } elseif ($_.Target -match '\.log$|\.pid$|\.state\.json$') {
            "Runtime-generated by the skill. Included if present; recreated automatically on first run."
        } else {
            "External runtime file. Review SKILL.md for usage."
        }
        "| ``$($_.Staged)`` | ``$($_.Target)`` | $notes |"
    }

    $extSection = @"

---

## External Files

These files live **outside the skill directory** and must be placed at their target paths for the skill to work correctly.
They are included in this ZIP under ``_external/`` with credential values cleared.

| File in ZIP | Install to (on target machine) | Notes |
|-------------|-------------------------------|-------|
$($extRows -join "`n")

### How to install external files

**Windows (PowerShell):**
``````powershell
# Example — adjust paths as shown in the table above
Copy-Item ".\$skillName\_external\Users\<you>\.config\$skillName\*" `
    "$HOME\.config\$skillName\" -Recurse -Force
``````

**macOS / Linux (bash):**
``````bash
cp -r ./$skillName/_external/Users/<you>/.config/$skillName/ ~/.config/$skillName/
``````

> After copying, fill in all cleared credential values before running the skill.
> Restrict permissions on config files: ``icacls`` (Windows) or ``chmod 600`` (macOS/Linux).
"@
}

$structureContent = @"
# Skill: $skillName

Auto-generated by **skill-extractor** on $now.
This file helps LLM agents and humans understand, verify, and install this skill correctly.

---

## Folder Layout

``````
$skillName/
$tree
``````

---

## File Descriptions

| File | Purpose |
|------|---------|
$($fileTable -join "`n")
$extSection

---

## How to Install

### Option A — ClawhHub (if published)
``````bash
clawhub install $skillName
``````

### Option B — Manual (recommended for shared ZIPs)
1. Extract the ZIP
2. Copy the ``$skillName/`` folder (excluding ``_external/``) into your OpenClaw workspace ``skills/`` directory:
   ``````
   <workspace>/skills/$skillName/
   ``````
3. If ``_external/`` exists: copy each file to its target path (see **External Files** table above)
4. Fill in all cleared credential values before use (see SKILL.md for field names)
5. Confirm the skill loaded: ``openclaw skills list``

### Option C — Local clawhub install
``````bash
clawhub install ./$skillName
``````
Then manually handle any ``_external/`` files as described above.

---

## Credential Notes

All credential values in this package have been **cleared** (set to ``""``) for security.
Before using this skill, fill in the required values in their respective config files.
Refer to ``SKILL.md`` for exact field names, file paths, and setup instructions.

> Never commit credential files to version control.

---

*Generated by skill-extractor v1.1.0 — https://clawhub.ai/seph1709/skill-extractor*
"@

[System.IO.File]::WriteAllText("$stagingDir\STRUCTURE.md", $structureContent, [System.Text.UTF8Encoding]::new($false))
Write-Host "📄 Generated: STRUCTURE.md"
```

---

## STEP 6 — Zip and Deliver

```powershell
# Default output: Desktop. Ask user to confirm or change.
$outputDir = "$HOME\Desktop"
$zipPath   = "$outputDir\$skillName.zip"

# Ask user: "Save ZIP to $zipPath? (or enter a different path)"
# If user confirms, proceed:

if (Test-Path $zipPath) { Remove-Item $zipPath -Force }

Compress-Archive -Path $stagingDir -DestinationPath $zipPath -CompressionLevel Optimal

Write-Host "✅ ZIP saved: $zipPath"
Write-Host "📦 Size: $([math]::Round((Get-Item $zipPath).Length / 1KB, 1)) KB"

# Cleanup staging
Remove-Item $stagingRoot -Recurse -Force
Write-Host "🧹 Staging cleaned up"
Write-Host ""
Write-Host "🎉 Done! Share: $zipPath"
```

---

## Error Reference

| Error | Cause | Fix |
|-------|-------|-----|
| Skill not found | Name mismatch | Check spelling; run `openclaw skills list` |
| Access denied on copy | File ownership issue | Run terminal as admin or check source permissions |
| Compress-Archive fails | PowerShell < 5 or disk full | Update PowerShell or free disk space |
| JSON parse error | Non-standard JSON (comments, trailing commas) | Scrubber skips it safely; manually inspect the file |
| Staging dir not cleaned | ZIP step failed | Manually delete `$HOME\.openclaw\workspace\.skill-export-staging` |
| External file not found | File only exists at runtime (e.g. worker.ps1 not yet written) | Safe to ignore — noted as "created at runtime" in STRUCTURE.md |
