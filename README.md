# git-fit-actions/db-store

GitHub Actions for caching and restoring the git-fit SQLite database.

## Actions

| Path | Description |
|------|-------------|
| `cache/restore` | Restore the database cache and snapshot baseline checksums before syncing |
| `cache/save` | Detect changes and save the database to cache after syncing |

## Usage

```yaml
steps:
  - uses: git-fit-actions/db-store/cache/restore@v1
    id: restore
    with:
      database-path: data/db/git-fit.db
  - name: Run sync
    run: bundle exec git fit sync --checkpoint
  - uses: git-fit-actions/db-store/cache/save@v1
    id: save
    with:
      database-path: data/db/git-fit.db
  - name: Upload DB artifact
    if: steps.save.outputs.changed == 'true'
    uses: actions/upload-artifact@v4
    with:
      name: git-fit-db
      path: data/db
```

## Inputs

### cache/restore

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `database-path` | no | `data/db/git-fit.db` | Path to the SQLite database file. Use an absolute path or a path relative to `GITHUB_WORKSPACE`. Tilde (`~`) is not expanded. |
| `git-restore-mode` | no | `none` | Whether/how to restore the database from a git branch when the cache does not provide it: `none` (disabled) / `fallback` (restore only when the database is missing/empty after the cache restore) / `force` (always restore from git, overwriting any cache-restored content). |
| `git-branch` | no | `GitFit/db` | Archive branch to restore the database from. Only used when `git-restore-mode` is not `none`. |

### cache/save

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `database-path` | no | `data/db/git-fit.db` | Path to the SQLite database file. Use an absolute path or a path relative to `GITHUB_WORKSPACE`. Tilde (`~`) is not expanded. |
| `save-to-git-branch` | no | `false` | When the database changed and this is a branch name, push the database to that branch. `false` or empty disables the git push. `true` is rejected — pass an explicit branch name. |

## Outputs

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

## Git branch fallback & save-back

The `cache/restore` and `cache/save` actions can use a git archive branch (default `GitFit/db`) as a fallback when the 7-day LRU cache misses or is untrusted.

- **restore — `git-restore-mode: fallback`** (recommended for sync): restores from the git branch only when the database is missing/empty after the cache restore. On success it outputs the branch in `save-to-git-branch`.
- **restore — `git-restore-mode: force`**: always restores from the git branch, overwriting cache-restored content. Git is treated as authoritative. On git failure it warns and keeps the cache-restored database (best effort).
- **save — `save-to-git-branch`**: pushes the changed database back to the branch via a detached worktree (DB + `.SHA256SUMS` + `-wal` + `-shm`, whichever exist). Git save is best-effort: fetch/push failures warn and set `git-save-status=failed` without failing the job. Non-fast-forward pushes are not force-pushed.

Typical cache-miss recovery wiring:

```yaml
- uses: git-fit-actions/db-store/cache/restore@v1
  id: restore
  with:
    database-path: data/db/git-fit.db
    git-restore-mode: fallback
    git-branch: GitFit/db
- uses: git-fit-actions/db-store/cache/save@v1
  with:
    database-path: data/db/git-fit.db
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
| `cache/restore` | `actions: read`; `contents: read` when `git-restore-mode` is not `none` (git fetch/archive) |
| `cache/save` | `actions: write`; `contents: write` when `save-to-git-branch` is set (git push) |

## Baseline contract

`cache/restore` snapshots the database directory contents after restoring from cache and writes a baseline to `$RUNNER_TEMP/git-fit-db-before.SHA256SUMS`. `cache/save` reads that baseline, diffs it against the post-sync state, and reports `changed`. If the baseline is missing (e.g. `save` used without a preceding `restore`), the action treats the database as changed.

## Security

Free-form inputs are validated before any filesystem, cache, or git operation; invalid values fail fast.

**cache/restore:**
- `database-path`: rejected if empty, contains control characters, or starts with `-`.
- `git-restore-mode`: must be `none`, `fallback`, or `force`.
- `git-branch` (when `git-restore-mode` is not `none`): defaults to `GitFit/db` when empty; otherwise rejected if it contains control characters or starts with `-`.

**cache/save:**
- `database-path`: rejected if empty, contains control characters, or starts with `-`.
- `save-to-git-branch`: `true` is rejected — pass an explicit branch name or `false`/empty. Branch names are shell-quoted in `git fetch`/`git push`.

## License

MIT
