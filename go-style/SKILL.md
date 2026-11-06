---
name: go-style
description: Use when writing, reviewing, refactoring, or documenting Go code. Applies the user's default idiomatic, standard-library-inspired house style to existing and new projects, with optional profiles for GUTS services and go.astrophena.name/base.
---

# Go Style

> Clear is better than clever.

House test: write code so Rob Pike would approve of its simplicity. Prefer the
design that makes the operation obvious, keeps policy at the right boundary, and
needs the fewest concepts.

Write unsurprising Go: clear control flow, small APIs, explicit ownership, and
few dependencies. This is the default house style for every user-owned Go
project, existing or new. The user's task and nearer repository instructions
take precedence; in third-party repositories, preserve local conventions.

Before broad work, read the applicable repository guidance and `go.mod`. Treat
the module's `go` version as the compatibility floor; a newer local toolchain
does not authorize newer APIs or syntax.

## Communicate simply

When this skill is loaded, speak and write in plain, direct English. Take the
useful lesson from grug-brained programming: complexity is dangerous, and simple
systems are easier to trust. Use proper grammar and a professional tone; do not
copy the site's character voice.

Prefer short sentences and common words. State the result first. Explain only
the details that help the reader understand the code, the risk, or the next
step. Keep review notes, documentation, and code comments clear and concrete.

In code comments, never use the words `intentionally` or `deliberately`. Say
what the code does or why the constraint exists.

## Keep the system simple

Minimize the complexity of the whole system, not the line count of one function.
Keep related behavior together, wait for abstractions to emerge, and prefer a
little obvious duplication to callbacks, configuration, or a framework. Keep
refactors small and complete. Measure before optimizing.

Fear complexity and choose boring code. See
[grug-brained programming](https://grugbrain.dev/) for the idea behind this
rule.

Start a new project with the smallest useful module and package structure.
Establish `gofmt`, package documentation, focused tests, and one clear
validation command early. Add layers, directories, fixtures, and machinery only
when real code needs them.

Start concrete and flat. Add a package when it owns a coherent capability, not
to reproduce architectural layers. Prefer `auth` or `billing` to `utils`,
`service`, or `repository`; use `internal` when its import restriction is
useful. Let `main` assemble dependencies and handle process concerns.

Prefer the standard library. Add a dependency when it removes meaningful risk or
complexity, not for a trivial wrapper. Avoid speculative configuration, generic
frameworks, and knobs without a current use.

## Write readable functions

Keep the common path linear. Handle validation, errors, and edge cases early:

```go
func loadUser(id string) (*User, error) {
	if id == "" {
		return nil, errors.New("empty user ID")
	}

	u, err := db.FindUser(id)
	if err != nil {
		return nil, fmt.Errorf("find user %q: %w", id, err)
	}
	return u, nil
}
```

Make code breathe. Separate setup, validation, work, and cleanup with blank
lines when they are distinct phases. Group related local state so its shape is
visible at a glance:

```go
var (
	items []Item
	seen  = make(map[string]bool)
)
```

Write a function or method on one line when its body is one obvious statement
and the complete declaration remains easy to scan after `gofmt`:

```go
func main() { cli.Main(cli.AppFunc(run)) }

func (e errorFS) Open(string) (fs.File, error) { return nil, e.err }
```

Do not compress validation, branching, error handling, or multiple side effects.
Use named results when they clarify the contract or deferred cleanup; avoid
naked returns in non-trivial functions.

## Name and expose deliberately

Use short, lowercase package names and avoid stutter (`http.Server`, not
`http.HTTPServer`). Preserve initialisms such as `ID` and `HTTP`. Omit `Get`
from a simple accessor. Name booleans affirmatively—`enabled`, `found`,
`valid`—and avoid double negatives.

Keep identifiers unexported by default. If a type is used only within its
package, keep the type and its methods lowercase even when it forms a useful
abstraction. Export only what another package genuinely needs to name or call;
do not export implementation details merely for tests.

Use one short receiver name derived from the type consistently across its
methods; avoid `this`, `self`, and descriptive receiver names. Use pointer
receivers for mutation, non-copyable state, or meaningfully large values. Use
value receivers for small value-like types. Do not mix receiver kinds without a
semantic reason.

Make zero values useful where natural. Require a constructor only when it
establishes an invariant or dependency:

```go
type counter struct {
	mu sync.Mutex
	n  int
}

func (c *counter) inc() {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.n++
}

func (c *counter) dec() {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.n--
}
```

## Design APIs and data from use

Keep exported APIs small. Accept an interface when callers benefit from
substitution; otherwise accept the concrete type. Define the smallest useful
interface in the consuming package and usually return a concrete type:

```go
type UserFinder interface {
	FindUser(context.Context, string) (*User, error)
}

type Handler struct {
	users UserFinder
}

func NewHandler(users UserFinder) *Handler { return &Handler{users: users} }
```

Do not invent an interface solely to mock an implementation. Let it emerge from
the operations a consumer needs. Use generics for one algorithm that genuinely
applies to several types, not to build type hierarchies. Start concrete;
generalize only after duplication makes the common operation clear.

For new APIs with optional configuration, prefer functional options to public
configuration bags or proliferating constructors. Keep required inputs as
ordinary parameters, provide useful defaults, and add an option only for a real
use case. Keep option types and functions unexported when construction is
package-local:

```go
type option func(*server)

func withTimeout(d time.Duration) option {
	return func(s *server) { s.timeout = d }
}

func newServer(addr string, opts ...option) *server {
	s := &server{addr: addr, timeout: 30 * time.Second}
	for _, opt := range opts {
		opt(s)
	}
	return s
}
```

Export `Option`, `WithTimeout`, and the constructor only when callers in other
packages genuinely need them.

Keep mutable state out of package globals. Globals should normally be constants,
immutable data, or sentinel errors treated as immutable. Put runtime state and
dependencies in an owned type or pass them explicitly.

Do not expose internal slices or maps when callers could mutate package state.
Return a copy when the package retains ownership, and preserve the API's nil
versus empty semantics:

```go
func (s *store) names() []string { return slices.Clone(s.names) }
```

Preallocate collections when the final or approximate size is already known:

```go
items := make([]Item, 0, len(records))
```

Do not add a counting pass or complicated capacity estimate merely to
preallocate.

## Handle errors, resources, and concurrency

Errors are values. Do not just check errors; handle them gracefully. Create
sentinel or typed errors only when callers need `errors.Is` or `errors.As` to
make a policy decision. Otherwise return an ordinary contextual error. Error
strings normally begin lowercase and end without punctuation. Log an error or
return it—normally not both.

```go
func readConfig(path string) ([]byte, error) {
	data, err := os.ReadFile(path)
	if err != nil {
		return nil, fmt.Errorf("read config %q: %w", path, err)
	}
	return data, nil
}
```

Do not repeat rote error checks when the API can express one shared failure
boundary. For a sequence of related operations, consider a sticky-error writer
or object: record the first error, make later operations no-ops, and report the
error once at the end.

```go
type errWriter struct {
	w   io.Writer
	err error
}

func (ew *errWriter) write(p []byte) {
	if ew.err != nil {
		return
	}
	_, ew.err = ew.w.Write(p)
}

func writeMessage(w io.Writer, header, body []byte) error {
	ew := &errWriter{w: w}
	ew.write(header)
	ew.write(body)
	if ew.err != nil {
		return fmt.Errorf("write message: %w", ew.err)
	}
	return nil
}
```

The same design appears in APIs such as `bufio.Writer` and `bufio.Scanner`:
perform the natural operation repeatedly, then check `Flush` or `Err` once.
Always check the final error. Use explicit per-step checks when partial
progress, different recovery policies, or the exact failing operation matters.

Keep acquisition and cleanup visibly paired. Check cleanup errors when they can
change the result, especially writes:

```go
if err := w.Flush(); err != nil {
	return fmt.Errorf("flush output: %w", err)
}
```

Pass `context.Context` as the first parameter across cancellation or request
boundaries; do not store it in a struct or replace it with `context.Background`
inside the call chain. Every goroutine needs an owner, a stopping condition,
and—when completion matters—a join path:

```go
func watch(ctx context.Context, updates <-chan Update) error {
	for {
		select {
		case <-ctx.Done():
			return ctx.Err()
		case u, ok := <-updates:
			if !ok {
				return nil
			}
			apply(u)
		}
	}
}
```

Use channels for communication and ownership transfer; use a mutex for simple
shared state. Limit concurrency when the downstream resource is bounded. Prefer
synchronous functions when they are clearer; callers can add concurrency more
easily than remove it.

## Test behavior

Test observable behavior at stable boundaries. Use a table when cases share the
same setup and assertion—not by reflex. When cases are independent and order is
irrelevant, prefer `cases := map[string]struct { ... }` and
`for name, tc := range cases`:

```go
func TestParsePort(t *testing.T) {
	cases := map[string]struct {
		in   string
		want int
	}{
		"http":        {in: "80", want: 80},
		"development": {in: "8080", want: 8080},
	}

	for name, tc := range cases {
		t.Run(name, func(t *testing.T) {
			got, err := ParsePort(tc.in)
			if err != nil {
				t.Fatal(err)
			}
			if got != tc.want {
				t.Fatalf("ParsePort(%q) = %d; want %d", tc.in, got, tc.want)
			}
		})
	}
}
```

Use `t.Parallel()` only when the test and its subtests share no mutable state,
environment, working directory, fixture, or golden output target.

For non-trivial text transformations, generated output, or filesystem behavior,
default to a hand-editable `testdata/name.txtar` input paired with a
`testdata/name.golden` result. Keep output deterministic and updates explicit;
review the diff, then rerun without the update flag. Use direct assertions for
small scalar behavior. When the project uses `base`, follow
[references/base-packages.md](references/base-packages.md).

Use `t.Helper()` in assertion helpers, `t.TempDir()` for filesystem tests, and
`testdata` for fixtures. Prefer small fakes to interaction-heavy mocks. Do not
wait for concurrency with arbitrary sleeps; synchronize on the event under test.
Add a regression test for a bug when practical.

## Document and finish

Start package documentation with one short sentence saying what the package
does. Exported declaration comments begin with the name and operation; add
contracts callers need, keeping defaults and details with the API that owns
them. Explain non-obvious reasons and invariants inside functions.

Examples show ordinary use of the public API. Keep deterministic examples
runnable, and use compile-only examples when external I/O or test setup would
obscure the call. When writing or reviewing documentation, read
[references/documentation.md](references/documentation.md) for the agreed
comment and example conventions.

```go
// Parse parses src and reports syntax errors with byte offsets.
func Parse(src []byte) (*Tree, error)
```

Run `gofmt` on changed Go files. While iterating, run the narrowest relevant
test; before handoff, run the repository-prescribed checks, commonly
`go test ./...` and `go vet ./...`. Use `go test -race` when concurrency changes
and the repository supports it. Report checks that could not run.

Use current standard-library features only when the module version permits
them—for example `slices` (Go 1.21), enhanced `http.ServeMux` patterns (Go
1.22), or `testing.B.Loop` (Go 1.24). Confirm exact APIs with `go doc` rather
than memory.

## Project profiles

Load only the relevant profile:

- For the GUTS stack, read
  [references/guts-services.md](references/guts-services.md).
- For imports from `go.astrophena.name/base`, read
  [references/base-packages.md](references/base-packages.md) and inspect the
  version selected by the target repository.

Do not introduce profile-specific technology merely because this skill is
active.

## References

- [Effective Go](https://go.dev/doc/effective_go) and
  [Go Code Review Comments](https://go.dev/wiki/CodeReviewComments) for official
  idioms.
- [spf13's Go skill](https://github.com/spf13/go-skills/blob/main/go/SKILL.md)
  for pragmatic simplicity and modern-library reminders.
- Rob Pike's [Errors are values](https://go.dev/blog/errors-are-values) for
  graceful error handling and sticky-error writers.
