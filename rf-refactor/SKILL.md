---
name: rf-refactor
description: Use when refactoring Go programs with rsc.io/rf and preserving a replayable rsc.io/rf/git-generate recipe, including semantic renames, declaration moves, and typed rewrites.
---

# Reproducible Go Refactoring

Use `rf` to express Go refactors as commands, and retain the complete shell
recipe so `git-generate` can reproduce the change from its Git base.

## Resolve tools first

At the start, look for **both** `rf` and `git-generate` in `PATH` with
`command -v`. Resolve them independently: use each installed command when
available; otherwise run its package directly with `go run ...@latest`.
Do not install the tools or add them to the target module as dependencies.

These Bash helpers preserve that choice and forward arguments correctly:

```bash
run_rf() {
    if command -v rf >/dev/null 2>&1; then
        command rf "$@"
    else
        go run rsc.io/rf@latest "$@"
    fi
}

run_git_generate() {
    if command -v git-generate >/dev/null 2>&1; then
        command git-generate "$@"
    else
        go run rsc.io/rf/git-generate@latest "$@"
    fi
}
```

Include the `run_rf` definition in saved recipes that use it: `git-generate`
starts a fresh Bash process, which does not inherit ordinary shell functions.
Use repository-relative paths in recipes. Record the Go version and the tool
build/module versions used; installed binaries and `@latest` can change on
later replay. Keep the requested `@latest` fallback unless the user asks for
version pinning.

## Plan and preview

Inspect repository guidance, `go.mod`, the affected declarations and callers,
and `git status --short`. Identify the base commit for replay and preserve
existing user edits. Run `rf` from the directory of the package being
refactored, which may differ from the Git or module root.

Pass the refactoring script as **one argument**, with one command per line.
Use single quotes or a quoted heredoc to keep shell expansion out of the
script. Preview with `-diff` before applying the same script without it:

```bash
run_rf -diff 'mv OldName NewName'
run_rf 'mv OldName NewName'
```

Prefer semantic addresses such as `Type.Field` or `Func.local` to line
numbers when possible. Renaming a nested item repeats its enclosing address,
for example `mv Client.Send Client.Deliver`. Moving declarations to another
file or package uses `mv`; typed expression rewrites use `ex`. Consult the
[rf documentation](https://pkg.go.dev/rsc.io/rf) for the chosen operation and
address syntax rather than guessing support. Some documented operations are
unimplemented, including function inlining.

Each command must leave the program type-correct. Split large refactors into
valid steps. For code with platform-specific files, consider `-allplat` and
keep the same flags in the saved recipe; cover relevant custom build tags
separately. Review effects beyond type checking, including interface method
names, reflection, string-based lookups, comments, and generated code.

## Keep a complete recipe

Save every command needed to recreate the final diff, including supporting
edits, formatting, and relevant validation. Avoid interactive steps, local
absolute paths, or dependencies on unrecorded temporary files. If a change
cannot be expressed by `rf`, include a deterministic supporting edit in the
recipe and verify it together with the refactor.

`git-generate` expects a commit message containing `[git-generate]` on a line
by itself, followed by the Bash recipe. Execution starts at the Git root with
`bash -e`. The recipe ends at the end of the message or a `Change-Id:` line;
put explanatory prose before the marker.

For example, a complete message for a package in `internal/client`:

```text
client: rename Send to Deliver

[git-generate]
run_rf() {
    if command -v rf >/dev/null 2>&1; then
        command rf "$@"
    else
        go run rsc.io/rf@latest "$@"
    fi
}
cd internal/client
run_rf 'mv Client.Send Client.Deliver'
go test .
```

When committing is in scope, retain this block in the refactor commit's
message. Otherwise, leave the refactored files and a durable message draft
containing the recipe, and report the base commit. The recipe must reproduce
the whole refactor diff, including any edits made after the initial `rf` run.

## Verify replay

`git-generate` modifies the working tree and index. Prefer a disposable
checkout or worktree for replay verification, especially when the user's
working tree contains other changes. Keep a draft message file outside that
checkout so resetting files cannot remove it.

- `run_git_generate -f /absolute/path/to/message.txt` reads a draft message
  and applies its recipe on top of **HEAD**. Use a checkout at the recorded
  base commit to verify a refactor before committing it. This mode also
  restores tracked files to HEAD before running the recipe.
- `run_git_generate` reads the **HEAD commit message**, restores files to
  **HEAD^**, then runs the recipe. Use a checkout at the refactor commit;
  this requires a parent commit. It does not create or amend a commit.
- `run_git_generate -conflict` uses the pending conflicting commit's recipe
  during a rebase or cherry-pick and applies it on top of HEAD. Use it only
  as part of the requested conflict resolution; review the generated result
  before continuing. It cannot be combined with `-f`.

Inspect both `git diff` and `git diff --cached`: the tool stages tracked
changes and newly created files. After default replay, compare
the complete result to the refactor commit; after `-f` replay, compare it to
the intended patch. Include added and deleted files in the comparison.

Run the affected package tests and required repository checks. `rf` formats
and type-checks its edits, but behavioral tests and diff review still matter.
If replay differs, update the recipe and repeat verification from the same
base before claiming the refactor is reproducible.

See the [git-generate documentation](https://pkg.go.dev/rsc.io/rf/git-generate)
and [implementation](https://github.com/rsc/rf/blob/main/git-generate/main.go)
for reset, staging, and conflict behavior in the version being used.
