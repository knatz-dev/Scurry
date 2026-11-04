# Scurry Agent Instructions

## Project Purpose

Scurry is a self-hosted, multi-account URL shortener and bookmark manager. The first deployment is personal, but one installation must support multiple accounts with strict data ownership and authorization boundaries. The project is intended to be MIT-licensed.

## Architecture

- Keep Scurry as a modular Go monolith. Use the standard HTTP server and the established Chi routing patterns.
- Keep server-rendered HTML and HTMX for the web UI. Compile Tailwind CSS during the build and embed the resulting static assets.
- SQLite is the default database profile for a single host and a persistent local data volume.
- PostgreSQL is the larger-installation profile for greater write concurrency or multiple application instances.
- SQLite WAL allows readers alongside a writer but has one writer at a time. Do not put a WAL database on a network filesystem or share one SQLite file across hosts.
- Persistence changes must preserve both database profiles. Keep shared SQL portable, provide engine-specific migrations where necessary, and do not assume that changing a connection setting migrates data.
- Do not add Redis, a message broker, microservices, or another runtime service without a demonstrated requirement and an explicit architecture decision.
- Do not claim fixed latency, memory, or image-size results without measurements.

## Multi-Account Security

- Every user-owned record must have an explicit ownership relationship.
- Scope every read, update, delete, export, and bulk operation to the authenticated user or authorized account. A record ID or slug is not proof of authorization.
- Preserve authentication, authorization, and CSRF protections when changing UI or API routes.
- Validate redirect destinations as HTTP or HTTPS URLs. Preserve the product’s intended redirect behavior while preventing unsafe schemes.
- Trust forwarded host and client-IP headers only from configured reverse proxies.
- Treat bookmark metadata fetching as an untrusted outbound request. Prevent access to loopback, private, link-local, and cloud-metadata addresses; validate redirect targets; enforce timeouts and response-size limits.
- Keep secrets out of source control and container images. Bound and minimize stored referrer and user-agent data, and preserve the configured retention behavior.
- Public registration, OIDC, teams, and organization-level sharing are product decisions. Do not invent their policy when implementing adjacent work.

## Redirects and Analytics

- Keep the redirect handler short and independent from analytics persistence.
- Analytics events are best effort unless the architecture is explicitly changed. Never claim lossless counts from an in-memory queue.
- Use a bounded, non-blocking analytics queue. If it fills, let the redirect proceed and record a dropped-event metric.
- Use a single batched writer for SQLite. Keep shutdown draining bounded and report events that could not be persisted.
- If durable analytics becomes a requirement, propose a durable ingestion design before changing the loss policy.

## Persistence and Migrations

- Enable SQLite foreign-key enforcement on every connection. Apply SQLite connection-scoped settings through driver connection initialization.
- Treat SQLite-specific features such as PRAGMAs and WITHOUT ROWID as engine-specific. Do not put them in shared assumptions for PostgreSQL.
- Avoid indexes that duplicate a primary key. Add indexes to support actual query patterns and verify them against representative queries.
- Serialize schema migration execution when multiple PostgreSQL application instances may start together.
- When persistence changes, check behavior in both SQLite and PostgreSQL profiles if both are configured in the repository.

## Development Workflow

- Read the relevant code, README, and build configuration before changing a subsystem. Follow existing project conventions and keep changes focused.
- Use idiomatic Go naming and format Go files with gofmt.
- Use the repository’s documented build, test, lint, and asset commands. Do not invent success results; report checks that could not be run.
- For Go changes, use the project’s test suite; where no project-specific command exists, the standard Go package test command is go test ./....
- Update architecture or operational documentation when a change alters a database profile, security boundary, analytics durability, deployment topology, or configuration contract.
- Keep dependencies small and justify new dependencies by the capability they provide.

## Completion Notes

Summarize the behavior changed, the important files affected, and the checks actually performed. Call out database-profile or security implications when relevant.

