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

```
GitFit-db-v0-cold                              (restore key)
GitFit-db-v0-*                                 (restore-keys prefix)
GitFit-db-v0-<run_number>-<run_id>[-<attempt>] (save key)
```

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
