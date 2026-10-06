# IsraRecover command line — guide for AI agents (version 1.0.2)

IsraRecover is a free file-recovery program for Windows. Its command line runs the same drive check, scan,
recovery and file doctor as the window, prints exactly one JSON document per command and uses fixed exit codes.
Human page (Hebrew): https://israrecover.syncup.co.il/cli.html · This file: https://israrecover.syncup.co.il/ai.md

## Rules for the agent

- Never write anything to the drive or image you recover from: no saving, no installing, no chkdsk, no format.
  The CLI refuses a destination on the scanned drive (exit 4, `destination_on_source`).
- Ask the user before `recover` (it writes files) and before `doctor --repair`. `--dry-run` is always safe.
- Recommended order: `drives` → `check <source>` → `scan <source> --save <file.rscan>` →
  `recover <file.rscan> --to <folder on another drive> --dry-run` → ask the user → the same without `--dry-run`.
- A drive letter or a physical disk needs an elevated (admin) prompt; the CLI never asks for elevation itself
  (exit 4, `needs_admin`). Image files and saved scans need no admin rights.
- Hardware trouble (drive not detected, clicking, wrong size) is for a recovery lab, not for repeated scans.

## Install (unattended)

From an elevated cmd:

```
curl.exe -L -o IsraRecover-setup.exe https://github.com/bot-sync-up/israrecover/releases/latest/download/IsraRecover-setup.exe
IsraRecover-setup.exe /VERYSILENT /SUPPRESSMSGBOXES /NORESTART
```

It installs `C:\Program Files\IsraRecover\IsraRecover.exe` (exit 0 on success). Without installing, download
`https://github.com/bot-sync-up/israrecover/releases/latest/download/IsraRecover-portable.exe` and run that file directly.
Silent uninstall: `"C:\Program Files\IsraRecover\unins000.exe" /VERYSILENT /SUPPRESSMSGBOXES /NORESTART`.

## How to run it (the exe is a GUI-subsystem program)

| From | How |
|---|---|
| cmd, Git Bash, Python `subprocess`, Node `child_process` | `IsraRecover.exe cli scan E: --json > out.json` — waits; the exit code is right |
| PowerShell | `& $exe cli scan E: --json \| Out-String \| ConvertFrom-Json` — without the pipe PowerShell does not wait |
| PowerShell, Hebrew file names | `[Console]::OutputEncoding = [Text.Encoding]::UTF8` before running |
| An interactive cmd window | `cmd /c IsraRecover.exe cli drives` — otherwise the prompt returns before the output |

stdout is UTF-8 (no BOM) when redirected. With `--json`, stdout holds one JSON document; progress and warnings go
to stderr. Ctrl+C stops a scan or a recovery cleanly: the answer is still written, `cancelled: true`, exit 5.

## Commands (`IsraRecover.exe cli <command> [options]`)

### `cli version [--json]`

The program's version and the CLI's format version.

### `cli help [command] [--json]`

The commands, their options and the exit codes.

### `cli drives [--json]`

The drives the home screen lists: disks, partitions, size, free space, health (SMART) and state. Read-only; no admin needed.

- SMART health needs admin rights on most drives; without them it is reported as not available.

### `cli check <source> [--situation deleted|formatted|noisy|unsure] [--json]`

The drive check of the 'check the drive' screen: open, health, structure, history - each step's result, the recommendation and the scan it plans. Read-only.

- `--situation` <value>: What happened, as the 'what happened?' screen asks it (default unsure). It changes the plan the same way.
- <source> is a drive letter (E:), a physical disk (disk:2 = \\.\PhysicalDrive2) or an image file (IMG, VHD, VHDX, VMDK, E01, ...).
- A drive letter or a disk needs admin rights (error needs_admin, exit 4).

### `cli scan <source> [--mode auto|quick|deep|signatures] [--situation ...] [--save <file.rscan>] [--list <file.json|file.csv>] [--json]`

Scans the source the way the program does and reports what it found. Ctrl+C stops the scan cleanly (exit 5, the files found so far are kept).

- `--mode` <value>: auto (default): the drive check decides, as in the program; quick: the file systems only; deep: every sector, file systems + signatures + lost partitions; signatures: the 'by file type' scan with every type.
- `--situation` <value>: For --mode auto: what happened (deleted, formatted, noisy, unsure).
- `--save` <value>: Write the scan as a .rscan file the program opens (File > open saved scan) and 'cli recover' takes.
- `--list` <value>: Write the full file list: .csv, or .json (any other extension = JSON).
- <source> is a drive letter (E:), a physical disk (disk:2 = \\.\PhysicalDrive2) or an image file (IMG, VHD, VHDX, VMDK, E01, ...).
- --mode auto refuses (exit 4) a drive that should be imaged first, a hardware fault and a locked BitLocker drive; give --mode quick|deep to scan anyway.

### `cli recover <source|file.rscan> --to <folder> [--include <glob>] [--type jpg,docx] [--status intact|partial|overwritten|not-checked|any] [--deleted-only] [--dry-run] [--mode ...] [--json]`

Recovers files with the program's own recovery: same folders, same report (CSV with SHA-256). A source is scanned first (--mode, default auto); a .rscan is used as saved.

- `--to` <value>: The folder to recover into. Refused (exit 4) when it is on the scanned drive or is the image file itself - the same check the program makes.
- `--include` <value>: A glob (* and ?). With a \ or / it is matched against the path (folder\name), else against the name. Repeatable.
- `--type` <value>: File extensions, comma separated (jpg,docx). Repeatable.
- `--status` <value>: intact, partial, overwritten, not-checked or any (default any). Comma separated for several.
- `--deleted-only`: Only files that are not present on the drive any more (deleted, lost, found by signature).
- `--dry-run`: List what would be recovered; write nothing. --to is optional.
- `--mode` <value>: The scan mode when <source> is a drive or an image (see 'cli help scan').
- `--situation` <value>: For --mode auto: what happened (deleted, formatted, noisy, unsure).
- <source> is a drive letter (E:), a physical disk (disk:2 = \\.\PhysicalDrive2) or an image file (IMG, VHD, VHDX, VMDK, E01, ...).
- A file known by name only (its data location is gone) is never recovered; it is left out of the list.

### `cli doctor <file...> [--repair --to <folder>] [--json]`

The file doctor's verdict for each file; with --repair the repairable ones are rebuilt into --to. The original files are never changed.

- `--repair`: Write repaired copies of the repairable files into --to.
- `--to` <value>: The folder for the repaired copies (needed with --repair).

Common option: `--json` — Print exactly one JSON document to stdout (progress and warnings go to stderr).

## Exit codes

| Code | Name | Meaning |
|---|---|---|
| 0 | `ok` | Finished without problems. |
| 1 | `usage` | Unknown command or option, or a missing / bad value. |
| 2 | `not_found` | The source (drive, disk, image, saved scan, file) is not there or cannot be opened. |
| 3 | `nothing` | The command ran and found nothing (no files, or nothing matches the filters). |
| 4 | `refused` | Refused: a write would land on the scanned drive, admin rights are needed, or the drive should be imaged first. |
| 5 | `partial` | Finished with some errors: failed files, stopped by Ctrl+C, or the drive disconnected. |

Error codes (`error.code` when `ok` is false): `usage`, `bad_source`, `source_not_found`, `open_failed`, `needs_admin`, `destination_on_source`, `low_space`, `destination_failed`, `image_first`, `lab`, `bitlocker`, `cancelled`, `failed`.

## The answer

Every answer is one object with fixed camelCase keys. New keys may be added; a renamed or removed key raises
`formatVersion` (now 1, shown by `cli version --json`).

```json
{ "ok": true, "command": "scan", "exitCode": 0, "...": "the command's own fields", "warnings": [] }
{ "ok": false, "command": "scan", "exitCode": 2, "error": { "code": "source_not_found", "message": "No such file: D:\\x.img" }, "warnings": [] }
```

`ok` is false only when there is an `error`. Exit 3 and 5 come with `ok: true`; the exit code says the rest.

## Guarantees

- The CLI never writes to the source drive or image. It writes only to --to, --save and --list.
- Not in the CLI (they write to drives): drive repair, driver switch, device cleanup, card test, imaging. Use the program's window for those.
- It never opens a window and never asks for admin rights by itself: run it from an elevated prompt to read a drive letter or a disk.
