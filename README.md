# Codex Windows Project Recovery

Portable recovery tools and documentation for restoring Codex Desktop local projects after an update.

## What this solves

After a Windows Codex Desktop update, the sidebar can show **Projects → No projects** even though the local project database and project folders are still intact.

The recovery script rebuilds the renderer/project catalog from the local SQLite database and re-arms the local-project migration. It derives all project IDs, names, paths, and counts from the current installation. It does not contain user-specific project data or a hard-coded project count.

## Files

- [tools/Recover-Codex-Projects.ps1](tools/Recover-Codex-Projects.ps1) — portable PowerShell recovery script.
- [tools/Recover-Codex-Projects.cmd](tools/Recover-Codex-Projects.cmd) — visible Windows launcher that keeps PowerShell open so the result can be read.
- [Issue #1](../../issues/1) — detailed diagnosis, recovery procedure, verification checklist, and PowerShell 5.1 compatibility notes.

## Quick use

1. Exit Codex completely with **File → Exit ChatGPT** or **Ctrl+Q**.
2. Confirm that no `ChatGPT.exe` or `codex.exe` process remains.
3. Keep both files from `tools/` in the same local folder.
4. Run `Recover-Codex-Projects.cmd`.
5. Read the PowerShell output. The script creates a timestamped backup before changing state.
6. Start Codex again only after the script reports completion.

The script refuses to modify files while Codex is running.

## Safety

- It validates SQLite with `PRAGMA quick_check`.
- It reads `projects`, `project_roots`, and `project_idempotency_keys`.
- It writes timestamped backups under `%USERPROFILE%\\.codex\\backups`.
- It never deletes `state_5.sqlite`.
- It never invents project IDs or paths.
- It works with any number of projects.

If `sqlite3.exe` is not on PATH, run the PowerShell script with `-Sqlite3Path <full path to sqlite3.exe>`.
