---
name: gopls-refactor
description: Use when making semantic Go refactors where AST and type information are safer than textual replacement.
---

# gopls Refactoring

Use `gopls` for refactors that depend on Go syntax or types. For narrow,
obviously textual edits, Perl, Python, or `sed` may be simpler.

Keep the program buildable while refactoring. Work in small steps, inspect the
diff, format, and run focused tests after each step.

## Rename

Prefer `gopls rename` for symbols:

```sh
gopls rename path/file.go:line:column NewName
```

or with a zero-based byte offset:

```sh
gopls rename path/file.go:#123 NewName
```

Use semantic rename instead of global text replacement when references may be
ambiguous. It checks type information, including shadowing and interface
satisfaction.

Renames can still affect behavior that Go's type system cannot see, such as
reflection, templates, or string-based names. Review and test the result.

## Code actions

`gopls` exposes inline, extract, rewrite, and organize-imports refactors as code
actions.

Preview a refactor first:

```sh
gopls codeaction -exec -kind=refactor.inline.call -diff path/file.go:#123-#456
```

Apply it with `-write`:

```sh
gopls codeaction -exec -kind=refactor.inline.call -write path/file.go:#123-#456
```

Useful kinds include:

- `refactor.inline.call`
- `refactor.inline.variable`
- `refactor.extract.function`
- `refactor.extract.method`
- `refactor.extract.variable`
- `refactor.extract.toNewFile`
- `refactor.rewrite.removeUnusedParam`
- `source.organizeImports`

Prefer Rename and Inline over Extract when possible. Extract is less rigorous,
and Extract/Inline may lose or move comments. Always inspect their diffs.

Do not refactor generated files.

## Verify

After a refactor:

```sh
gofmt -w <changed files>
go test <affected packages>
```

Use `rg` to check that obsolete names or references are gone.

The `gopls` command-line interface is experimental. If a command or flag differs
from this skill, run:

```sh
gopls help
gopls help <command>
```

and follow the installed version.
