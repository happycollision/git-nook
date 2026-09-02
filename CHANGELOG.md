# Changelog

All notable changes to `git-nook` are documented here.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.5.0] - 2026-09-02

### Added

- `git nook -n <name> git-dir` and `git nook -n <name> home` print the nook's
  inner git-dir and its content directory, respectively — exactly one bare
  absolute path each, so the value composes directly in a subshell
  (`fork "$(git nook -n notes git-dir)"`). `show` already reported both as
  labeled fields; these exist for scripts and for launching external GUI tools
  that want a path rather than a git command. Both are read-only: they never
  materialize, contact a remote, or mutate config. `home` errors instead of
  printing an empty line when no home is recorded, so
  `cd "$(git nook -n x home)"` cannot silently land in `$HOME`.

### Documentation

- New README section, "Diffing and browsing a nook." A nook has always been a
  real local git repository, so any history tool already works on it through
  `run` (`git nook -n notes run difftool HEAD~3`); this documents that, plus
  two ways to get a confident wrong answer:
  - Driving git by path requires **both** `--git-dir` and `--work-tree`. The
    inner repo sets `core.bare false`, so `--git-dir` alone does not error —
    git silently treats the caller's cwd as the work tree and compares the
    wrong files.
  - Some GUIs reject a bare git-dir. GitKraken ignores `GIT_DIR`, does its own
    filesystem discovery, and rejects a nook's `branch.<name>.merge`
    (`refs/nook/<slug>/files`) as an invalid merge ref, since it expects
    `refs/heads/*` — the invisibility design working as intended. For those
    tools, clone the git-dir into a throwaway directory: cloning rewrites the
    tracking config to a conventional `refs/heads/main`, yielding an ordinary
    repo any GUI will open. Read-only; it is a snapshot.

## [0.4.0] - 2026-07-22

### Changed

- **A nook's checkout now lives in the working tree, not under `.git/`.** Each
  nook has a "primary home": a real directory at `<toplevel>/<dir>` in the
  worktree that created or adopted it, recorded in local-only git config
  `nook.<slug>.home` (never pushed to any remote). The inner git repository
  (objects/refs) still lives under `.git/nook/<slug>.git`. Other worktrees
  expose the nook as a symlink to the primary home. This unblocks tools that
  refuse to operate on paths under `.git/` (e.g. `br`/beads), which previously
  could not read a nook's content.
- `git nook init` and `git nook clone` now elect the caller's `<toplevel>/<dir>`
  as the primary home (a real directory), instead of creating a checkout under
  `.git/`.
- `git nook materialize` gained an election / adoption / promotion / peer-symlink
  model:
  - **Election** — a worktree with no recorded home becomes the primary home.
  - **Adoption** — a pre-existing content directory (e.g. a nook created before
    this version) is adopted in place, recording the home without moving or
    clobbering anything.
  - **Promotion** — if the recorded home has vanished, running `materialize` in
    any surviving worktree re-checks-out the content from the inner repository
    and re-records the home there.
  - **Peer** — other worktrees receive a symlink to the primary home.

### Added

- `git nook -n <name> materialize` — materialize a single nook by name (the
  recovery form referenced by the dangling-home guard's hint).
- A dispatch guard that aborts `run` / `remove` / `destroy` with an actionable
  `git nook -n <name> materialize` hint when a nook's primary home is missing,
  before any underlying git/tool command runs. `materialize` and `show` are
  exempt.
- `git nook show` / `git nook list` now report the primary-home / linked state.
- Invariant test coverage: no checkout path or symlink target ever contains a
  `.git` path component; `nook.<slug>.home` never travels to a remote.

### Fixed

- `git nook remove` / `git nook destroy` run from a peer worktree no longer
  orphan the primary home directory in the worktree that owns it.
- Electing through a live symlink is refused, so a nook's home can never be
  recorded at a path resolving under `.git/`.
- The live-symlink refusal for the pre-worktree-home nested-content-dir layout
  now names the concrete `rm` / `mv` / `materialize` recovery steps and points
  at `MIGRATION.md`.
- `materialize` restores an emptied-but-present primary home from the inner
  repository (recovery after `git clean -x`), without ever repopulating a
  non-empty home.
- `populate_checkout_from_head` no longer errors on a legitimately empty
  committed tree.

### Migration

- Existing nooks are migrated by relocating their checkout out of `.git/` into a
  real working-tree directory; see the "worktree-home layout" section of
  `MIGRATION.md`. Migration is manual and human-confirmed, and — for the common
  pre-home case — non-destructive (`git nook materialize` adopts existing
  content in place). Note: because the checkout is now host-visible (hidden only
  via `.git/info/exclude`), it is exposed to `git clean -x`; this is inherent to
  keeping the content readable outside `.git/`, and is recoverable by re-running
  `git nook materialize`.

## [0.3.0] - 2026-07-14

### Added

- Universal nook identity: an immutable UUID and a slug
  `<name>.<id3>.<owner>.<repo_dir>` used for config keys, local paths, and refs.
- `git nook init` (fresh create), `git nook clone` (bootstrap an existing nook,
  with index-based disambiguation), `git nook remove` (local delete with an
  unpushed-work guard), and `git nook destroy` (delete upstream refs + local
  state).
- Per-nook manifest ref (`refs/nook/<slug>/manifest`) and a rebuildable
  collection index (`refs/nook/index`, `git nook reindex`).
- Legacy-layout detection with a `MIGRATION.md` procedure (never auto-migrates).
- Pull/fetch guard that refuses a remote whose manifest UUID differs from the
  local one.

### Changed

- `-n <name>` selector grammar everywhere; the old bare `git nook <name> <verb>`
  passthrough form was removed.
- Removed the `git nook add` command (superseded by `init`/`clone`).

## [0.2.1] - earlier

- Earlier development releases (pre-universal-identity).

[0.5.0]: https://github.com/happycollision/git-nook/releases/tag/v0.5.0
[0.4.0]: https://github.com/happycollision/git-nook/releases/tag/v0.4.0
[0.3.0]: https://github.com/happycollision/git-nook/releases/tag/v0.3.0
[0.2.1]: https://github.com/happycollision/git-nook/releases/tag/v0.2.1
