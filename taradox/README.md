# taradox's YugabyteDB

This branch, `taradox`, is what taradox builds and runs: upstream's stable tag plus our fix branches, regenerated from the manifest below at each sync, never merged into by hand. These notes are for the agents who work in this repository.

## Branches
- `master` mirrors yugabyte/yugabyte-db `master`, fast-forward only. Nothing of ours ever lands on it.
- `fix/<name>`: one fix or feature, cut from `master`. Each one becomes an upstream pull request and stays open on its own; a layered one is cut from the branch below it.
- `backport/<stable>/fix/<name>`: the same change on the stable line we run.
- `taradox`: the stable tag, then each backport in manifest order, then this `taradox/` folder.

## Manifest (release 1, on 2026.1.2.0-b137)
1. `backport/2026.1.2/fix/notifications-poller-timeout-retryable` (#31484)
2. `backport/2026.1.2/fix/cdc-state-write-off-io-thread` (upstream 753607d5149 #33853, then #12173)

## Where the rest lives
- The work, its order and the defects' evidence: the taradox repo, `workspace/taradox-hosted/lanes/database.md` and `production.sheet.md` (T16–T28).
- Building: the taradox repo's `database/builder/` (run only in a container, never on the machine).
- `turn1-notes.md` and `turn2-notes.md` beside this file: the database lane's reading of this source, with file:line citations, and the build, test and QUORUM commands.
