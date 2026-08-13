# git-fit-actions/db-store

GitHub Actions for caching and restoring the git-fit SQLite database.

## Actions

| Path | Description |
|------|-------------|
| `paths` | Derive the database directory, filename, and full path from a single file path |
| `cache/restore` | Restore the database cache and snapshot baseline checksums before syncing |
| `cache/save` | Detect changes and save the database to cache after syncing |

## Usage

```yaml
env:
  GIT_FIT_DATABASE_PATH: ${{ vars.GIT_FIT_DATABASE_PATH || 'data/db/git-fit.db' }}

steps:
  - uses: git-fit-actions/db-store/paths@v1
    id: db-paths
    with:
      database-path: ${{ env.GIT_FIT_DATABASE_PATH }}
  - uses: git-fit-actions/db-store/cache/restore@v1
    id: restore
    with:
      db-dir: ${{ steps.db-paths.outputs.dir }}
      db-name: ${{ steps.db-paths.outputs.name }}
  - name: Run sync
    run: bundle exec git fit sync --checkpoint
    env:
      GIT_FIT_DATABASE_PATH: ${{ steps.db-paths.outputs.db-path }}
  - uses: git-fit-actions/db-store/cache/save@v1
    id: save
    with:
      db-dir: ${{ steps.db-paths.outputs.dir }}
      db-name: ${{ steps.db-paths.outputs.name }}
  - name: Upload DB artifact
    if: steps.save.outputs.changed == 'true'
    uses: actions/upload-artifact@v4
    with:
      name: git-fit-db
      path: ${{ steps.db-paths.outputs.dir }}
```

The `paths` action is optional: `cache/restore` and `cache/save` accept `db-dir` + `db-name`
directly. `paths` exists so every DB path reference in a workflow (db-store, git-fit,
checkpoint, artifacts, summary) is derived once from a single variable.

## Inputs

### paths

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `database-path` | no | `data/db/git-fit.db` | Full path to the SQLite database file. Use an absolute path or a path relative to `GITHUB_WORKSPACE`. |

### cache/restore

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `db-dir` | no | `data/db` | Path to the database directory. Use an absolute path or a path relative to `GITHUB_WORKSPACE`. Tilde (`~`) is not expanded. |
| `db-name` | no | `git-fit.db` | Database filename within `db-dir`. Must be a plain filename (no path separators); the database file path is `db-dir/db-name`. |
| `git-restore-mode` | no | `none` | Whether/how to restore the database from a git branch when the cache does not provide it: `none` (disabled) / `fallback` (restore only when the database is missing/empty after the cache restore) / `force` (always restore from git, overwriting any cache-restored content). |
| `git-branch` | no | `GitFit/db` | Archive branch to restore the database from. Only used when `git-restore-mode` is not `none`. |

### cache/save

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `db-dir` | no | `data/db` | Path to the database directory. Use an absolute path or a path relative to `GITHUB_WORKSPACE`. Tilde (`~`) is not expanded. |
| `db-name` | no | `git-fit.db` | Database filename within `db-dir`. Must be a plain filename (no path separators); the database file path is `db-dir/db-name`. |
| `save-to-git-branch` | no | `false` | When the database changed and this is a branch name, push the database to that branch. `false` or empty disables the git push. `true` is rejected — pass an explicit branch name. |

## Outputs

### paths

| Output | Description |
|--------|-------------|
| `dir` | Database directory relative to `GITHUB_WORKSPACE` (e.g. `data/db`). |
| `name` | Database filename (e.g. `git-fit.db`). |
| `db-path` | Full database path relative to `GITHUB_WORKSPACE` (`dir/name`). |

### cache/restore

| Output | Description |
|--------|-------------|
| `git-restored-bytes` | Bytes restored from the git branch (0 when none or disabled). |
| `save-to-git-branch` | Branch to push the database back to after sync when the git restore succeeded; `false` otherwise. Wire this into `cache/save`'s `save-to-git-branch` input. |

### cache/save

| Output | Description |
|--------|-------------|
| `changed` | Whether the database changed during sync. |
| `git-save-status` | Result of the optional git branch save: `skipped`, `unchanged`, `pushed`, or `failed`. |

## Step summaries

Each action appends a markdown block to the job step summary (`| Item | Value |`):

- `cache/restore` → `### GitFit db-store restore`: DB path, DB file presence/size,
  restore source (`cache` / `git` / `none (fresh)`), git-restored bytes and
  `save-to-git-branch`.
- `cache/save` → `### GitFit db-store save`: DB path, `Changed` flag, cache save
  status (`saved` / `skipped (unchanged)` / `failed`) and git save status
  (`pushed` / `unchanged` / `skipped` / `failed`).

## Git branch fallback & save-back

The `cache/restore` and `cache/save` actions can use a git archive branch (default `GitFit/db`) as a fallback when the 7-day LRU cache misses or is untrusted.

- **restore — `git-restore-mode: fallback`** (recommended for sync): restores from the git branch only when the database is missing/empty after the cache restore. On success it outputs the branch in `save-to-git-branch`.
- **restore — `git-restore-mode: force`**: always restores from the git branch, overwriting cache-restored content. Git is treated as authoritative. On git failure it warns and keeps the cache-restored database (best effort).
- **save — `save-to-git-branch`**: pushes the changed database back to the branch via a detached worktree (DB + `.SHA256SUMS` + `-wal` + `-shm`, whichever exist). Git save is best-effort: fetch/push failures warn and set `git-save-status=failed` without failing the job. Non-fast-forward pushes are not force-pushed.

Typical cache-miss recovery wiring:

```yaml
- uses: git-fit-actions/db-store/paths@v1
  id: db-paths
  with:
    database-path: ${{ vars.GIT_FIT_DATABASE_PATH }}
- uses: git-fit-actions/db-store/cache/restore@v1
  id: restore
  with:
    db-dir: ${{ steps.db-paths.outputs.dir }}
    db-name: ${{ steps.db-paths.outputs.name }}
    git-restore-mode: fallback
    git-branch: GitFit/db
- uses: git-fit-actions/db-store/cache/save@v1
  with:
    db-dir: ${{ steps.db-paths.outputs.dir }}
    db-name: ${{ steps.db-paths.outputs.name }}
    save-to-git-branch: ${{ steps.restore.outputs.save-to-git-branch }}
```

Precedence: `cache` (primary) → git branch (fallback/force) → fresh DB created by sync. In `fallback` mode the git branch is only refreshed by the save-back when the cache missed; the weekly archive workflow remains the primary git writer.

## Cache key scheme

All keys share a `GitFit-db-v0-` prefix. `cache/restore` never writes a cache:
its exact `key` is a sentinel that always misses, forcing a `restore-keys` prefix match that
selects the most recently created cache.

```
GitFit-db-v0-sentinel                          (restore exact key — sentinel, never written)
GitFit-db-v0-*                                 (restore-keys prefix — most recent wins)
GitFit-db-v0-<run_number>-<run_id>[-<attempt>] (sync save key)
GitFit-db-v0-snapshot                          (unarchive recovery snapshot — external convention)
```

- **sentinel** (`-v0-sentinel`): restore-only exact key, never written; guarantees prefix fallback.
- **snapshot** (`-v0-snapshot`): written by unarchive workflows (git → cache) as a readable recovery snapshot.
- Per-run sync caches are saved as `-v0-<run_number>-<run_id>`.

## Permissions

| Action | Required |
|--------|----------|
| `paths` | none |
| `cache/restore` | `actions: read`; `contents: read` when `git-restore-mode` is not `none` (git fetch/archive) |
| `cache/save` | `actions: write`; `contents: write` when `save-to-git-branch` is set (git push) |

## Baseline contract

`cache/restore` snapshots the database directory contents after restoring from cache and writes a baseline to `$RUNNER_TEMP/git-fit-db-before.SHA256SUMS`. `cache/save` reads that baseline, diffs it against the post-sync state, and reports `changed`. If the baseline is missing (e.g. `save` used without a preceding `restore`), the action treats the database as changed.

## Security

Free-form inputs are validated before any filesystem, cache, or git operation; invalid values fail fast.

**paths:**
- `database-path`: rejected if empty, contains control characters, or starts with `-`.

**cache/restore:**
- `db-dir`: rejected if empty, contains control characters, or starts with `-`.
- `db-name`: rejected if empty, contains control characters, starts with `-`, contains a path separator, or is `.`/`..` — it must be a plain filename.
- `git-restore-mode`: must be `none`, `fallback`, or `force`.
- `git-branch` (when `git-restore-mode` is not `none`): defaults to `GitFit/db` when empty; otherwise rejected if it contains control characters or starts with `-`.

**cache/save:**
- `db-dir`: rejected if empty, contains control characters, or starts with `-`.
- `db-name`: rejected as in `cache/restore`.
- `save-to-git-branch`: `true` is rejected — pass an explicit branch name or `false`/empty; otherwise rejected if it contains control characters or starts with `-`. Branch names are shell-quoted in `git fetch`/`git push`.

## License

MIT
