# Go documentation

## Package comments

Start with one simple, short sentence explaining the package's purpose:

```go
// Package request sends HTTP requests and decodes JSON responses.
```

Usually, that sentence is enough. Add prose when readers need a concept to get
started, such as a file format or a record/replay mode. Use doc.go for a
substantial overview. Keep command documentation as compact help pages with
invocations and essential setup. Prefix shell command examples in doc comments
with `$`.

Use concrete nouns and verbs. Describe what the package does without selling its
simplicity, listing every feature, or narrating the implementation. Internal
package comments should help maintainers.

## Declaration comments

Begin with the declaration's name and operation. Add the facts that change how
callers use it: meaningful errors, ownership, cleanup, concurrency, or
zero-value behavior. Include these where relevant, without treating them as a
checklist.

Keep defaults with their fields or constructors. Link to related contracts
rather than repeating them in package, type, and method comments.

```go
// Make sends an HTTP request and returns its response.
// Response may be [Bytes] for raw data, [IgnoreResponse] to discard the
// body, or another type to decode from JSON.
//
// An unexpected status returns an error wrapping [StatusError].
// If a successful response exceeds [Params.MaxResponseBytes],
// Make returns [ErrResponseTooLarge].
```

## Examples

Show one useful task through the public API, with the inputs, call sequence, and
result a caller needs. Attach the example to the API it teaches. A package does
not need an example for every exported declaration.

```go
func ExampleMake() {
	type user struct{ Name string }

	u, err := request.Make[user](context.Background(), request.Params{
		URL: "https://example.com/user",
	})
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println(u.Name)
}
```

This example compiles without running: it has no Output comment. Use this form
for live services, terminal output, systemd, or files supplied by the caller.
Use checked Output for small, deterministic examples. Keep mocks, fixtures, and
output normalization when that setup is part of the lesson itself.

Handle errors as callers would: return them from application callbacks and use
ordinary reporting at the program boundary. Keep acquisition and cleanup paired;
check cleanup errors when they affect the result.

For a complete application, put one Example function and its supporting
declarations in a file without Test, Fuzz, or Benchmark functions. Go
documentation then displays the whole file. Other examples can share a file when
each is self-contained.

For APIs requiring *testing.T, show a short ordinary Test recipe in the
documentation. It can refer to an existing application or operation when that
context is stated. Keep the recipe focused on using the helper.

Review rendered documentation as well as source: check links, visible helper
declarations, readable output, and introductions whose examples were removed.
Run go test to compile all Example functions and execute those with Output.
