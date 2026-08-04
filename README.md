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
    with:
      database-path: data/db/git-fit.db
  - name: Run sync
    run: bundle exec git fit sync --checkpoint
  - uses: git-fit-actions/db-store/cache/save@v1
    with:
      database-path: data/db/git-fit.db
  - name: Upload DB artifact
    if: steps.save.outputs.changed == 'true'
    uses: actions/upload-artifact@v4
    with:
      name: git-fit-db
      path: ${{ steps.restore.outputs.db-dir }}
```

## Inputs

| Input | Required | Description |
|-------|----------|-------------|
| `database-path` | no | Path to the SQLite database file. Default: `data/db/git-fit.db`. Use an absolute path or a path relative to `GITHUB_WORKSPACE`. Tilde (`~`) is not expanded. |

## Outputs

| Action | Output | Description |
|--------|--------|-------------|
| `cache/restore` | `db-dir` | Directory containing the database, relative to `GITHUB_WORKSPACE` when possible. |
| `cache/save` | `changed` | Whether the database changed during sync. |

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
| `cache/restore` | `actions: read` |
| `cache/save` | `actions: write` |

## Baseline contract

`cache/restore` snapshots the database directory contents after restoring from cache and writes a baseline to `$RUNNER_TEMP/git-fit-db-before.SHA256SUMS`. `cache/save` reads that baseline, diffs it against the post-sync state, and reports `changed`. If the baseline is missing (e.g. `save` used without a preceding `restore`), the action treats the database as changed.

## Security

`database-path` is checked for emptiness, control characters, and a leading dash before any filesystem or cache operation. Invalid values fail fast.

## License

MIT
