# Codex Windows Project Recovery

Portable Windows recovery tools and documentation for restoring Codex Desktop local projects after an update.

## Problem

After a Windows Codex Desktop update, the sidebar may show **Projects → No projects** even though:

- the local project directories still exist;
- the SQLite database still contains project rows and root paths;
- restarting Codex does not help.

This is a local catalog/migration consistency failure, not necessarily deletion of the project data.

## Root cause

Codex uses two relevant local layers:

1. **App-server SQLite data** in `%USERPROFILE%\.codex\state_5.sqlite`.
   This contains project records, root paths, and legacy-to-server ID mappings.
2. **Renderer/global state** in `%USERPROFILE%\.codex\.codex-global-state.json`.
   This contains the catalog used by the Projects sidebar, including `local-projects`, `project-order`, and the host-specific app-server mapping.

The failure occurs when an update leaves the SQLite layer intact but leaves the renderer catalog empty or incomplete, while the migration marker already says `projectsMigrated: true`. Codex then trusts the marker and does not rebuild the missing catalog. Starting the application again can persist the empty catalog.

That is why editing the JSON while Codex is open is unsafe: the running application can overwrite the repair from its in-memory state.

## Files

- [tools/Recover-Codex-Projects.ps1](tools/Recover-Codex-Projects.ps1) — portable recovery script.
- [tools/Recover-Codex-Projects.cmd](tools/Recover-Codex-Projects.cmd) — visible Windows launcher that keeps PowerShell open.
- [docs/RECOVERY.md](docs/RECOVERY.md) — detailed architecture, diagnosis, recovery steps, safety rules, and troubleshooting.
- [Issue #1](issues/1) — issue-format recovery guide and root-cause record.

## Quick recovery

1. Exit Codex completely with **File → Exit ChatGPT** or **Ctrl+Q**.
2. Confirm that no `ChatGPT.exe` or `codex.exe` process remains.
3. Keep the two files from `tools/` in the same local folder.
4. Run `Recover-Codex-Projects.cmd`.
5. Read the visible PowerShell output.
6. Start Codex again only after the script reports completion.

The script derives all project IDs, names, paths, and counts from the current installation. It works with any number of local projects and refuses to modify files while Codex is running.

## What the script verifies and changes

- validates SQLite with `PRAGMA quick_check`;
- reads `projects`, `project_roots`, and `project_idempotency_keys`;
- creates a timestamped backup under `%USERPROFILE%\\.codex\\backups`;
- rebuilds the renderer catalog and project ordering;
- re-arms the local-project migration marker;
- writes state atomically;
- verifies the resulting project count and ordering.

It never deletes `state_5.sqlite`, never invents IDs or paths, and never assumes a fixed project count.

## PowerShell 5.1 note

The script deliberately parses SQLite JSON with `ConvertFrom-Json -InputObject` instead of piping JSON through an array expression. Windows PowerShell 5.1 can otherwise collapse a JSON row array into one object with array-valued properties and produce a false error such as **Project has no legacy idempotency key**.

If `sqlite3.exe` is not on PATH, run the PowerShell script with `-Sqlite3Path <full path to sqlite3.exe>`.
