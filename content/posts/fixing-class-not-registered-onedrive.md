---
title: "Fixing 'Class Not Registered' in Claude Code on OneDrive"
date: 2026-07-09
draft: false
tags: ["windows", "claude-code", "onedrive", "debugging", "devtools"]
summary: "Why Git Bash's mkdir fails on OneDrive folders in Claude Code, and the one-line fix that also unlocks editing .xlsx and .docx files in place."
ShowToc: true
TocOpen: false
---

The same `mkdir` works in your terminal but fails in Claude Code with a cryptic COM error. The cause isn't Claude Code, and the cure is to stop routing shell commands through Git Bash.

```text
● Bash(mkdir "C:\Users\...\OneDrive - Contoso\Desktop\test")
  ⎿  Error: Exit code 1
     Class not registered
```

## The symptom: one command, two outcomes

You open your own terminal, run `mkdir` against a folder inside OneDrive, and it works. You ask Claude Code to run the exact same command, and it fails with **"Class not registered."**

That asymmetry is the whole mystery, and it points straight at the answer. It looks like a Claude Code bug, but it isn't — it's two *different programs* both named `mkdir`, only one of which knows how to deal with a OneDrive folder.

- Your terminal (Command Prompt or PowerShell) uses the **native Windows** `mkdir`.
- Claude Code's Bash tool shells out to **Git Bash**, whose `mkdir` comes from MSYS2 — a POSIX emulation layer.

Same three letters, two implementations. Native Windows handles the OneDrive folder correctly. The POSIX one walks straight into a wall.

## Root cause: OneDrive folders aren't quite folders

With Files On-Demand switched on, OneDrive doesn't store your synced folders as ordinary directories. It stores them as **reparse points** — cloud placeholders that Windows resolves on the fly through a sync-provider **COM object** the first time a program touches them.

Native Windows tools know how to trigger that resolution. Git Bash's MSYS2 `mkdir`, running as a child process spawned by Claude Code, ends up in a context where that COM class can't be instantiated. Windows returns error `0x80040154` — `REGDB_E_CLASSNOTREG` — which surfaces as the maddeningly vague **"Class not registered."**

## Why it actually matters: your Office files

You might shrug at a failed directory create. Don't. The reason this is worth fixing is that **real edits to `.xlsx` and `.docx` files run through the shell.**

Those are binary formats. When Claude changes a cell, a formula, or document structure, it writes a small Python script (openpyxl, python-docx) and *runs* it — and running it goes through the Bash tool. If Bash can't operate inside your OneDrive folder, that entire workflow is dead in the water. Fix the shell and you unlock editing Office documents in place, right where they already sync.

## The fix: switch Claude Code onto the native PowerShell tool

Claude Code can spawn PowerShell directly instead of Git Bash. PowerShell is a native Windows shell, so its file operations resolve OneDrive placeholders the same way your own terminal does — the "Class not registered" wall simply disappears.

The switch is a single environment flag: `CLAUDE_CODE_USE_POWERSHELL_TOOL=1`. You need Claude Code v2.1.84 or newer (check with `claude doctor`).

### Step 1 — Close Claude Code

Settings and environment changes are only read on a fresh start.

### Step 2 — Check whether a settings file already exists

```powershell
Test-Path "$env:USERPROFILE\.claude\settings.json"
```

`False` → create it (Step 3a). `True` → merge into it (Step 3b) so you don't clobber existing settings.

### Step 3a — File doesn't exist: create it

```powershell
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.claude" | Out-Null
@'
{
  "env": {
    "CLAUDE_CODE_USE_POWERSHELL_TOOL": "1"
  }
}
'@ | Set-Content -Encoding utf8 "$env:USERPROFILE\.claude\settings.json"
```

### Step 3b — File exists: merge the key in

Open it with `notepad "$env:USERPROFILE\.claude\settings.json"` and add the flag inside an `env` block. If an `"env"` section already exists, just add the line inside it (mind the comma):

```json
{
  "env": {
    "CLAUDE_CODE_USE_POWERSHELL_TOOL": "1"
  }
}
```

### Step 4 — Validate the JSON

A stray comma makes Claude Code silently ignore the file.

```powershell
Get-Content "$env:USERPROFILE\.claude\settings.json" -Raw | ConvertFrom-Json
```

If it prints your settings back with no red error, you're good.

### Step 5 — Restart and confirm

Start a fresh session and run the command that used to fail:

```text
mkdir "C:\Users\...\OneDrive - Contoso\Desktop\test2"
```

No "Class not registered" means the switch took. Editing an existing `.xlsx` in that folder should now work end to end.

### Alternative: a system-wide flag

Prefer an environment variable over a settings file? Set it once, then open a **brand-new** terminal so it's inherited:

```powershell
[Environment]::SetEnvironmentVariable("CLAUDE_CODE_USE_POWERSHELL_TOOL", "1", "User")
```

## Belt and suspenders

The PowerShell switch is the fix. Two habits keep OneDrive from biting again:

- **Hydrate the folder.** Right-click your working folder in Explorer → *"Always keep on this device."* Cloud-only placeholders are the most fragile case.
- **Or sidestep OneDrive entirely.** Have Claude work in a plain local folder like `C:\dev\` and only copy finished files into OneDrive. No reparse points, no COM, no surprises.

One caveat: the PowerShell tool is a preview feature. It doesn't cover a couple of edge modes (auto-mode, Windows sandboxing) — none of which affect editing Office files on OneDrive.

## TL;DR

- "Class not registered" = Git Bash's `mkdir` can't resolve a OneDrive reparse point (COM error `0x80040154`).
- Your terminal works because it uses native Windows `mkdir`, not Git Bash.
- Set `CLAUDE_CODE_USE_POWERSHELL_TOOL=1` so Claude Code uses PowerShell — a native shell that resolves OneDrive folders correctly.
- Bonus: this is also what makes in-place `.xlsx` / `.docx` editing on OneDrive work, since those edits run through the shell.
