# Local runtime retention

`.local` and `.runtime` are OpenCraft-owned local runtime roots. The maintenance
command below only considers immediate child directories whose names end in an
8-digit date (for example `main-headless-20261115`). It uses
the newest mtime anywhere in the directory tree as the activity signal and
removes a directory only when it is at least seven days old and no process
currently references it through `/proc` `cwd`, `root`, `exe`, `maps`, or open
file descriptors. Any inability to inspect `/proc` is treated as unknown and
keeps the directory. Directories containing credential-like files (`.env`,
keys, certificates, credentials, tokens) or database files (`.db`, SQLite,
`-wal`, `-shm`) are also kept and the reason is printed.
Symbolic links are not followed: a linked child is not a candidate, and links
inside a candidate do not cause their targets to be inspected or deleted.

```sh
node scripts/maintenance/prune-local-runtime.mjs       # report only (default)
node scripts/maintenance/prune-local-runtime.mjs --apply
```

The apply mode appends one JSON record per removal to
`.local/maintenance/local-runtime-retention.jsonl`, including the path and byte
count. A daily systemd timer or cron entry may run the default command and run
`--apply` according to the host owner's change policy; keep the command scoped
to this checkout. `--now=2026-11-17T00:00:00Z` is available for deterministic
audits and tests.

Run `npm run test:local-runtime` for the regression tests in
`tests/maintenance/prune-local-runtime.test.mjs`. Separate cases cover deep
descendant mtimes, credential and database protection, references and read
failures for every `/proc` slot, dry runs, deletion records, symbolic links,
candidate scope, the seven-day boundary, and CLI mode selection. The upstream
contract CI job runs the same command.

Long-lived state is intentionally excluded: `.local/steward` (steward state and
work), `.local/personal-relay` (relay deployment and releases),
`.runtime/server-state` (databases/models), `.runtime/mason` (agent identities
and projects), `.runtime/managed-web-state` and managed-daemon directories
(deployed releases), and `.local/team-credentials` (credentials). These are
counted during inventory but require an owner-specific backup, migration, or
state-retention decision before deletion.
