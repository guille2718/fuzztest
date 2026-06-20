# CLAUDE.md

## What this repo is

A **fork of [google/fuzztest](https://github.com/google/fuzztest)**. It carries a
small stack of patches on top of upstream `main` so that FuzzTest / Centipede
build outside Google (Bazel 9, renamed repo, dependency tweaks).

Remotes:
- `origin` = `git@github.com:guille2718/fuzztest.git` — my fork (publish here).
- `upstream` = `https://github.com/google/fuzztest` — pull changes from here.

Upstream is exported from Google's monorepo via Copybara. Occasionally a BUILD
target references a file that isn't present in the open-source layout (see
"Drop broken upstream filegroup" below) — that's an upstream quirk, not a local
mistake.

## Version control: Jujutsu (jj)

This repo is managed with **jj** (colocated with git). `trunk()` is defined as
`main@origin`. The `main` bookmark is my published fork main = **latest upstream
+ my patch stack**, kept as a linear stack of commits on top of upstream `main`.

## Syncing with upstream

Keep the fork current by **rebasing the patch stack onto upstream `main`**
(do *not* merge), then **force-pushing** `main` to `origin`:

1. `jj git fetch --remote upstream`
2. `jj rebase -b @ -d 'main@upstream'`
   - The published patches are immutable; override just for the rebase:
     `--config 'revset-aliases."immutable_heads()"=none()'`
3. Resolve conflicts (see conventions below). jj records conflicts in the
   commits instead of stopping; resolve bottom-up with `jj edit <commit>`.
4. **Validate before pushing:** `bazel test //...` must pass.
5. `jj git push --bookmark main` (this is a force / sideways move of `origin/main`).

Everything is reversible until the push (`jj undo`, `jj op restore`).

## Maintained patches (on top of upstream)

Keep these as **discrete, logically-scoped commits**. Grouped by purpose:

**Build outside Google / Bazel 9**
- *Support Bazel 9.* — empties `.bazelversion` (unpins the version), enables
  `--incompatible_autoload_externally` for `@rules_cc,@rules_python,@rules_shell`,
  switches CC/CXX to `--repo_env`, overrides `rules_go` and `aspect_rules_esbuild`.
- *Use the new version of flatbuffers…* — adds a `git_override` for flatbuffers
  (rules_swift / Bazel 9 compatibility), on top of upstream's registry version.
- *Remove Google-internal buildenv/target:non_prod constraint from centipede/tools/BUILD.*
- *git-ignore bazel-generated files and folders.*
- *Enable the newer way of specifying assertion-mode on libc++.*

**Repo rename `@com_google_fuzztest` → `@fuzztest`**
- *Switch to new `@fuzztest` repo name.* — removes `repo_name = "com_google_fuzztest"`
  from `MODULE.bazel` and renames every `@com_google_fuzztest//` → `@fuzztest//`
  across BUILD files. **Exception: leave `codelab/` alone** — it's a separate
  Bazel module that intentionally keeps `repo_name = "com_google_fuzztest"` to
  demo external consumption under that alias.
- *Use @fuzztest instead of @com_google_fuzztest in setup_configs.sh.*

**Fixes**
- *Fix feature_analyzer to not use the deprecated error_message function in absl::Status.*
- *Add nullability specifier.*
- *Drop broken upstream :generate_cmake_from_bazel filegroup.* — upstream's
  `fuzztest/BUILD` declares an (unused, experimental) filegroup whose `srcs`
  resolves to `fuzztest/cmake/generate_cmake_from_bazel.py`, but that file only
  exists at the top-level `//cmake/` (which isn't a Bazel package). It breaks
  `bazel build //...`, so we remove it.

## Conflict-resolution conventions

- **The `@fuzztest` rename is a pure textual substitution** `@com_google_fuzztest`
  → `@fuzztest`. After a rebase, the old diff only covers references that existed
  when the patch was written — re-apply the substitution to any **new** upstream
  references: `grep -rl com_google_fuzztest .` and rename them. **Never touch
  `codelab/`.**
- **`.bazelversion` must end up empty.** Upstream periodically bumps it (e.g.
  `8.5.0` → `8.7.0`); the Bazel 9 patch unpins it, so resolve modify-vs-delete in
  favor of empty.
- **When a fork patch and upstream fix the same thing, prefer upstream's version**
  — it's maintained against upstream's pinned deps. Let the now-redundant fork
  patch go empty and abandon it. (Example: a local "Fix abseil" patch became
  redundant once upstream renamed the same target; the mocking target is
  `absl::random_mocking_access`.)

## Known gotchas

- `bazel test //...` may print **non-fatal** direct-dependency warnings for
  `rules_cc` and `platforms` — the Bazel 9 overrides pull newer transitive
  versions than upstream's pins. Silence with `--check_direct_dependencies=off`
  or bump the pins in `MODULE.bazel`.

## Preferences

- **Rebase, not merge**, to keep the fork in sync.
- **Always validate with `bazel test //...` before force-pushing.**
- Keep patches as small, single-purpose commits; fold rename fixups into the
  rename commit rather than a catch-all "fixups" commit.
