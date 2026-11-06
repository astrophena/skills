# `go.astrophena.name/base` package guide

The `base` module contains personal building blocks used across Go projects.
Reuse them when they make the call site smaller and preserve existing behavior.
Do not force them into code where the standard library is clearer.

Before use:

1. Check the version in the target repository's `go.mod`.
2. Read `go doc go.astrophena.name/base/<package>` or inspect that version's
   source.
3. Search nearby code for the repository's established setup and error policy.

## Common choices

| Need                          | Package            | Guidance                                                                                                                                                                                                                              |
| ----------------------------- | ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Command entry point and flags | `base/cli`         | Implement `cli.App`, or use `cli.AppFunc` for a small command. Call `cli.Main`; use `cli.Run` with `cli.Env` in tests. Reuse a command's `doc.go` through `cli.SetDocComment` when the package already follows that pattern.          |
| Command tests                 | `base/cli/clitest` | Use table-driven `clitest.Case` values instead of rebuilding stdin, stdout, stderr, arguments, and environment plumbing.                                                                                                              |
| Structured logging            | `base/logger`      | Pass `context.Context` through the call graph and use `logger.Debug`, `Info`, `Warn`, or `Error` with `slog.Attr` values. Install a logger with `logger.Put`; do not introduce a parallel global logger.                              |
| HTTP server                   | `base/web`         | Use `web.Server` for mux serving, static assets, middleware, CSP, CSRF protection, trusted proxies, debugging, and systemd notification. Use `RespondError`, `RespondJSON`, and `RespondJSONError` so handlers share response policy. |
| Service lifecycle             | `base/web/service` | Implement `PublicService`, `AdminService`, and/or `cli.App`, then call `service.Run`. Add `StatefulService`, `IsActiveChecker`, or `ShutdownHook` only when the service needs those lifecycle features.                               |
| Typed HTTP/JSON client        | `base/request`     | Use `request.Make[T]` with `request.Params`. Select `request.Bytes` or `request.IgnoreResponse` for those response shapes. Supply the caller's context and a custom client when transport policy differs.                             |
| Typed context values          | `base/ctxkey`      | Declare a package-level `ctxkey.Key[T]` with a useful default. Use it for request-scoped values, not as general dependency injection or mutable global state.                                                                         |
| Synchronization               | `base/syncx`       | Use `Protected[T]`, `Lazy[T]`, `Map[K,V]`, or `LimitedWaitGroup` when the type directly matches the concurrency requirement. Prefer an ordinary mutex or `sync.OnceValue` when that is simpler.                                       |
| systemd integration           | `base/systemd`     | Use `Socket` for named socket activation and `Notify`/`Watchdog` for readiness and watchdog state. Prefer `web.Server` or `web/service` when they already own the lifecycle.                                                          |
| File replacement with backups | `base/safefile`    | Use `WriteFile` to retain timestamped backups when replacing a file. The destination can briefly be absent during replacement.                                                                                                        |
| Human-facing values           | `base/humanfmt`    | Use `Bytes`, `DateTime`, and `RelativeTime` for UI or CLI display, not protocol or persisted representations.                                                                                                                         |
| Build identity                | `base/version`     | Use `Version`, `CmdName`, and `UserAgent` instead of duplicating `debug.ReadBuildInfo` parsing.                                                                                                                                       |

## Testing and fixtures

| Need                                        | Package         | Guidance                                                                                                                                                                  |
| ------------------------------------------- | --------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Diffs, golden files, JSON, and HTTP clients | `base/testutil` | Prefer `AssertEqual`, `Run`, `RunGolden`, `BuildTxtar`, `ExtractTxtar`, `UnmarshalJSON`, and `MockHTTPClient` to equivalent local plumbing. Keep golden updates explicit. |
| HTTP record/replay                          | `base/rr`       | Use for stable external HTTP fixtures when a real service is unsuitable. Scrub secrets and unstable fields before committing recordings.                                  |
| Text fixture trees                          | `base/txtar`    | Use for small, hand-editable trees that should diff well. It is not a binary archive or a file-mode-preserving format.                                                    |
| SSE                                         | `base/web/sse`  | Use `Streamer` for simple process-local broadcasts. Check its buffering and slow-client semantics before using it for durable or lossless delivery.                       |

### Txtar and golden tests

In repositories using `base`, default to a hand-editable `.txtar` input and a
same-stem `.golden` result for non-trivial text or filesystem behavior. This
keeps setup, expected output, and failure diffs reviewable:

```go
var update = flag.Bool("update", false, "update golden files in testdata")

func TestTransform(t *testing.T) {
	t.Parallel()

	testutil.RunGolden(t, "testdata/*.txtar", func(t *testing.T, match string) []byte {
		ar, err := txtar.ParseFile(match)
		if err != nil {
			t.Fatal(err)
		}

		dir := t.TempDir()
		testutil.ExtractTxtar(t, ar, dir)
		if err := transform(dir); err != nil {
			t.Fatal(err)
		}

		return testutil.BuildTxtar(t, dir)
	}, *update)
}
```

Return rendered output, logs, or another stable byte representation instead of
`BuildTxtar` when the result is not a filesystem tree. Use `testutil.Run` when a
fixture does not need golden output, and direct assertions for small scalar
behavior. Golden data must be deterministic: normalize timestamps, paths,
ordering, IDs, and secrets before comparison.

Updating goldens is an explicit review action. Run a focused test with
`-update`, inspect the `.golden` diff, and then rerun without `-update`; never
update expected output merely to make a failure disappear.

## Specialized packages

- `base/cli/progressbar` provides a simple non-animated progress display for
  interactive commands.
- `base/devtools/deploy/cdc` defines deployd's content-defined chunking
  contract. Use it only for code participating in that protocol; it is not a
  general file chunking convenience package.
- Packages under `base/devtools` are generally repository tools invoked through
  `go tool`, not libraries for application code.

## Project conventions around `base`

These are personal defaults for every existing and new project that uses `base`,
modeled on `go.astrophena.name/tools`. Nearer explicit repository instructions
still win.

- Keep command wiring small, commonly
  `func main() { cli.Main(cli.AppFunc(run)) }`, and exercise commands through
  `cli.Run`/`clitest` with an explicit `cli.Env` rather than spawning the
  binary.
- Let `service.Run` and `web.Server` own service lifecycle when possible rather
  than duplicating signal, listener, readiness, and shutdown code.
- When using `base/web/service`, preserve its documented reverse-proxy boundary:
  administrative endpoints trust the proxy, while public endpoints remain
  responsible for their own authentication.
- Use the context-aware logger already installed by `cli` or the service runner.
  Preserve request context so attributes and cancellation travel together.
- Prefer the existing response and request helpers to per-handler JSON and error
  conventions, but inspect their sentinel errors and trust behavior before
  changing an authorization boundary.
- Put a `base/web` sentinel first when adding response detail, for example
  `fmt.Errorf("%w: token is missing", web.ErrUnauthorized)`. For an internal
  response that must retain diagnostic text, use
  `fmt.Errorf("%w: load account: %v", web.ErrInternalServerError, err)` unless
  the repository deliberately exposes another wrapped cause.
- Pass explicit `slog.Attr` values to `base/logger`, such as
  `logger.Info(ctx, "renaming", slog.String("from", old),
  slog.String("to", next))`.
- If no package fits naturally, write the small concrete code locally. Add to
  `base` only after the behavior is useful across real repositories and its API
  boundary is understood.
