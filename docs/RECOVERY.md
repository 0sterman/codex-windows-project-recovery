# Codex Windows local Projects recovery

## Scope

This is a general recovery procedure for Windows Codex Desktop installations where the sidebar shows **Projects → No projects** after an update, while the local project folders and the Codex SQLite database still exist.

The procedure is installation-independent. It does not assume a particular project count, project name, username, or folder layout.

## What actually failed

Codex keeps local-project information in more than one layer.

### 1. App-server SQLite data

The local database is normally:

```text
%USERPROFILE%\.codex\state_5.sqlite
```

The relevant tables are:

- `projects` — project IDs, names, ordering, and timestamps;
- `project_roots` — one or more filesystem roots for each project;
- `project_idempotency_keys` — the legacy/local project key mapped to the app-server project ID.

This layer can remain healthy even when the sidebar is empty.

### 2. Renderer/global project catalog

The desktop UI also reads a catalog from:

```text
%USERPROFILE%\.codex\.codex-global-state.json
```

The important fields are:

- `local-projects`;
- `project-order`;
- `app-server-project-id-by-legacy-project-id-by-host`.

The sidebar uses this catalog to decide what to display.

### 3. Migration state

The same global state contains an app-server project migration marker. It records whether the local-project migration has already run, including fields such as:

```json
{
  "projectsMigrated": true,
  "threadAssignmentsMigrated": false
}
```

## Failure mode after an update

The observed failure is an inconsistent migration state:

1. The SQLite database still has projects, root paths, and ID mappings.
2. The renderer catalog is empty or incomplete.
3. The migration marker says `projectsMigrated: true`.
4. Codex trusts the marker and does not rebuild the missing catalog.
5. Starting Codex again persists the empty catalog, so a simple restart does not repair the sidebar.

This is a high-confidence diagnosis for this failure pattern. It is not evidence that project directories were deleted.

## Why editing the JSON while Codex is open does not work

Codex keeps the state in memory and writes it back during startup/shutdown or state updates. If the application is running while the JSON is edited, it can overwrite the repair with its in-memory empty catalog.

Therefore:

> Codex must be fully closed before the recovery script reads or writes the state.

Closing only the visible window may leave `ChatGPT.exe` or `codex.exe` running. The script checks for both process names and refuses to modify state while they are present.

## The supplied recovery files

- [Recover-Codex-Projects.ps1](../tools/Recover-Codex-Projects.ps1)
- [Recover-Codex-Projects.cmd](../tools/Recover-Codex-Projects.cmd)

The PowerShell script is generic:

- it derives `CodexHome` from `%USERPROFILE%\.codex`;
- it reads the current installation's SQLite rows;
- it never hard-codes a project count;
- it never hard-codes project names or paths;
- it creates a timestamped backup;
- it validates SQLite before writing;
- it re-arms the migration marker;
- it verifies the rebuilt catalog before returning success.

The `.cmd` file is only a visible launcher. It keeps PowerShell open so that the result and any error can be read.

## Safe recovery procedure

### Step 1 — Exit Codex

Use **File → Exit ChatGPT** or **Ctrl+Q**.

Wait until no `ChatGPT.exe` or `codex.exe` process remains. Do not run the script while Codex is open.

### Step 2 — Keep the files together

Place these two files in the same local folder:

```text
Recover-Codex-Projects.ps1
Recover-Codex-Projects.cmd
```

### Step 3 — Run the visible launcher

Double-click:

```text
Recover-Codex-Projects.cmd
```

The script will stop before writing if:

- Codex is still running;
- the state file is missing;
- `state_5.sqlite` is missing;
- `sqlite3.exe` cannot be found;
- SQLite validation fails;
- the database does not contain a complete project catalog;
- a project has no ID mapping or root path;
- the migration marker cannot be identified.

### Step 4 — Read the result

A successful result includes:

- `Status: completed`;
- `SQLiteQuickCheck: ok`;
- a positive `RestoredProjects` count;
- a matching `RestoredOrder` count;
- a timestamped backup path;
- a message telling you to start Codex again.

The count is expected to vary between installations.

### Step 5 — Start Codex

Only after the script reports completion, launch Codex again. The Projects sidebar should be rebuilt automatically.

## What the script changes

The script performs these operations in order:

1. checks that Codex is not running;
2. validates the SQLite database with `PRAGMA quick_check`;
3. reads all project, root, and idempotency rows;
4. maps each app-server project ID to its legacy/local ID;
5. builds `local-projects` from the database;
6. builds `project-order` from the database ordering;
7. rebuilds the host-specific app-server ID mapping;
8. removes the known completed local-project migration IDs;
9. sets the host migration marker to `projectsMigrated: false`;
10. writes timestamped backups;
11. writes the repaired state atomically;
12. parses the resulting state again and reports the actual restored counts.

It does not delete the SQLite database, project directories, threads, or unrelated global state.

## Windows PowerShell 5.1 compatibility

Windows PowerShell 5.1 can mishandle JSON arrays when they are passed through the pipeline and wrapped in an array expression. This can turn a list of row objects into one object whose properties contain arrays.

Avoid:

```powershell
$projects = @($json | ConvertFrom-Json)
```

Use direct input:

```powershell
$projects = ConvertFrom-Json -InputObject $json
$roots    = ConvertFrom-Json -InputObject $rootJson
$keys     = ConvertFrom-Json -InputObject $keyJson

if ($projects -isnot [array]) { $projects = @($projects) }
if ($roots -isnot [array]) { $roots = @($roots) }
if ($keys -isnot [array]) { $keys = @($keys) }
```

Without this correction, a valid database can produce a false error such as:

```text
Project has no legacy idempotency key
```

## Manual diagnostic queries

If the script stops before writing, these read-only queries help distinguish the failure types:

```sql
PRAGMA quick_check;

SELECT id, name, position, created_at_ms, updated_at_ms
FROM projects
ORDER BY position, id;

SELECT project_id, position, path
FROM project_roots
ORDER BY project_id, position;

SELECT key, project_id
FROM project_idempotency_keys
ORDER BY created_at_ms;
```

Interpretation:

- `quick_check = ok` and projects exist: the database is probably intact;
- projects exist but roots are missing: the database catalog is incomplete and the generic repair should stop;
- projects and roots exist but idempotency keys are missing: legacy-to-server mapping cannot be reconstructed safely;
- no projects exist: this procedure cannot recover data that is absent from the database and a separate backup/source must be investigated.

## Verification checklist

Recovery is complete only when all of these are true:

- SQLite reports `ok`;
- the restored project count is non-zero;
- `project-order` and `local-projects` contain the same number of entries;
- every project has at least one root path;
- a timestamped backup exists;
- Codex was closed during the write;
- the state remains populated after Codex starts;
- the Projects section is visible again.

## What not to do

- Do not delete `state_5.sqlite`.
- Do not manually create replacement projects before checking the database.
- Do not edit the state file while Codex is running.
- Do not assume a successful file write means the UI is repaired.
- Do not invent project IDs or root paths.
- Do not use a fixed project count in a recovery script.

## Limitations

This procedure repairs a local catalog that is still present in SQLite. It cannot restore projects that have been deleted from the database and are absent from all backups. It also does not repair unrelated cloud/ChatGPT projects or account synchronization issues.
