# GUTS service profile

Use this profile when a repository already follows GUTS or when designing a new
personal service whose requirements fit it. GUTS means Go, Unix, TypeScript, and
SQLite. It is a preference, not a reason to override the user's chosen stack or
an existing repository's architecture.

## Service shape

- Keep a service in one process unless a real operational boundary requires
  another process.
- Prefer a cohesive service over microservices. A network boundary must justify
  its deployment, failure, observability, latency, and compatibility costs.
- Keep domain packages concrete. Extract an interface where there is a stable
  caller boundary, not for every implementation type.
- Prefer configuration that can be read from flags, environment, or a small
  explicit file. Avoid building a configuration framework.

## Unix and operations

- Prefer Unix sockets, systemd socket activation, ordinary files, and
  inspectable command-line workflows over orchestration infrastructure.
- Make startup, readiness, shutdown, state directories, backups, and restore
  behavior explicit.
- Use structured, context-aware logs at operational boundaries. Preserve request
  or job identifiers across calls when available.
- Keep one obvious local-development command and one repository-prescribed
  validation command.

If the project uses `go.astrophena.name/base`, also read
[base-packages.md](base-packages.md). `base/web/service` and `base/web` normally
own service lifecycle, HTTP serving, systemd notification, static assets, and
the public/admin endpoint split.

## Data

- Use SQLite directly when one durable local database fits the service.
- Make transaction boundaries and migration order visible in code.
- Treat backup and restore as part of the storage design, especially around WAL
  mode and application-consistent snapshots.
- Do not add an ORM, repository layer, or database portability abstraction in
  anticipation of changing databases. Add a narrow boundary only when the domain
  or tests demonstrably benefit from it.

## Web UI

- Render HTML on the server with `templ`.
- Add browser behavior progressively with HTMX.
- Use small TypeScript modules for behavior that is clearer on the client.
- Reach for Preact only when client-side state genuinely benefits from
  components.
- Keep markup, behavior, and the handler that serves them close enough that a
  reader can follow one feature without searching the whole repository.

## RPC

Use Plug when the project already defines typed internal RPC through Plug
schemas or when a new trusted Go-to-Go boundary benefits from schema generation,
compatibility checking, and inspection. Do not add another RPC stack for the
same boundary. Do not introduce RPC where an in-process call, Unix command, or
ordinary HTTP handler is simpler.
