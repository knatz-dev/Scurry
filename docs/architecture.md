# Scurry — Technical Architecture

**Status:** Initial architecture decision. Performance limits and capacity targets must be set from measurements.

Scurry is a self-hosted, multi-account application that combines URL shortening and bookmark management. It is being built for personal use first, with a public MIT-licensed repository and a deployment path for larger shared installations.

## 1. Goals and Non-Goals

### Goals

- Let multiple accounts use one Scurry installation with clear ownership and authorization boundaries.
- Make the basic self-hosted installation straightforward, with no separate database service required.
- Provide a PostgreSQL deployment profile for higher write concurrency and multiple application instances.
- Keep redirect handling fast and independent from optional analytics work.
- Keep the application as one maintainable codebase and process in the default deployment.

### Non-Goals

- Guarantee sub-millisecond end-to-end redirects. Network and proxy latency are outside the application’s control.
- Use microservices, Redis, or a separate message broker in the initial deployment.
- Share an SQLite database file between hosts or across a network filesystem.
- Promise lossless click analytics while using an in-memory event queue.

## 2. Architecture Overview

Scurry is a modular monolith. A reverse proxy may handle TLS and public routing; Scurry serves the management UI, API, and redirect routes from one application. Static assets are built ahead of time and embedded in the Go binary.

The application has four primary modules:

- **Identity:** users, sessions, authentication, and authorization.
- **Short Links:** slugs, domains, destinations, expiry, and redirect behavior.
- **Bookmarks:** saved destinations, user notes, tags, and bookmark-specific state.
- **Analytics:** click ingestion, aggregation, retention, and reporting.

Module boundaries are code boundaries, not separate services. The application should be stateless apart from the selected database and any explicitly configured persistent files.

## 3. Technology Stack

- **Backend:** Go, using the standard HTTP server and Chi for routing and middleware.
- **UI:** server-rendered HTML with HTMX for interactive page updates.
- **CSS:** Tailwind CSS compiled during the build. The generated CSS and other static assets are embedded with Go’s embed package.
- **Persistence:** SQLite as the default profile; PostgreSQL as the larger-installation profile.
- **Reverse proxy:** optional. Caddy, Nginx, or Traefik may provide TLS termination and routing.

Pin a supported Go toolchain version in the build and update it through normal maintenance. The SQLite driver choice must be deliberate: a CGO-based driver requires a compatible C runtime in the final image unless the binary is explicitly and successfully linked otherwise. Do not describe the image as scratch-compatible or statically linked without verifying the produced binary.

## 4. Deployment Profiles

| Profile | Intended use | Constraints |
|---|---|---|
| **SQLite default** | Personal or small shared installation on one host | Database must be on persistent local storage. Run one Scurry instance against the file. WAL permits concurrent readers but only one writer at a time. |
| **PostgreSQL** | Larger shared installation or multiple Scurry application instances | Run one or more stateless application instances behind a proxy or load balancer, connected to a PostgreSQL service. |

SQLite WAL requires processes to share memory on the same host and does not work over a network filesystem. Use PostgreSQL when the deployment needs multiple application hosts or more concurrent writes. citesqlite-wal

Supporting both engines has an ongoing cost. Maintain migrations for each engine, keep shared queries within the supported SQL subset, and verify behavior against both databases in CI. SQLite-specific optimizations belong in SQLite-specific migrations or initialization, not in assumptions shared by both profiles. Moving data between engines requires an explicit export/import or migration procedure; changing a configuration value alone does not convert the database.

## 5. Domain and Data Model

Model bookmarks and short links as distinct resources that can refer to the same destination.

- **Users:** authentication identity and account status.
- **Link targets:** destination URL and shared metadata such as title and description.
- **Short links:** owner, target, slug, optional custom domain, expiry, status, and any access-control settings.
- **Bookmarks:** owner, target, notes, tags, and saved or archived state.
- **Click events or aggregates:** reference a short link and contain only the analytics fields required by the product.

Every user-owned resource must carry an ownership relationship. Queries and service operations must enforce that ownership; possession of a slug or record identifier is not authorization. Custom slugs must be protected by a unique constraint on the relevant domain-and-slug pair, with collision handling at creation time.

The schema must not assume that SQLite-specific types or features are available in PostgreSQL. Avoid redundant indexes that duplicate primary-key indexes. Add indexes to support measured queries, especially per-user listing and per-link time-based analytics.

## 6. Redirect and Analytics Flow

### Redirect request

1. Resolve the requested domain and slug.
2. Look up the active short link and destination.
3. Prepare a click event.
4. Attempt to add the event to a bounded, non-blocking in-process queue.
5. Return the redirect response without waiting for analytics persistence.

When the queue is full, the redirect still succeeds and a dropped-event counter is incremented. The initial analytics policy is **best effort**: a process crash may lose queued events. If the product requires durable click events, replace this policy with a durable ingestion path before claiming lossless analytics.

Use a single batched analytics writer for SQLite. PostgreSQL deployments may batch writes through each application instance, while monitoring database latency and queue depth.

### Bookmark metadata

If Scurry fetches titles, descriptions, icons, or previews from saved URLs, perform that work asynchronously. Treat it as an untrusted outbound fetch: permit only supported HTTP or HTTPS URLs, block loopback, private, link-local, and cloud-metadata addresses, re-check redirect targets, and enforce response-size and timeout limits.

## 7. Database Configuration

For SQLite:

- Enable WAL mode and verify that SQLite accepted the setting.
- Enable foreign-key enforcement on every connection.
- Apply connection-scoped settings through driver connection initialization. In particular, cache size is not a process-wide memory allocation; budget it per connection. citesqlite-pragmas
- Use a busy timeout only as a bounded lock-wait safeguard, not as a concurrency strategy.
- Choose the synchronous setting according to the durability requirement. In WAL mode, NORMAL can lose recently committed transactions after power loss; it is not a guaranteed fixed-factor performance improvement. citesqlite-pragmas
- Use SQLite’s Online Backup API or another SQLite-supported live backup method. Do not back up a live WAL database by copying only the main database file. citesqlite-backup

Avoid WITHOUT ROWID in shared schema assumptions because it is SQLite-specific. Use it only if profiling justifies a SQLite-specific migration and the PostgreSQL schema remains separately correct.

## 8. Security and Privacy

- Require authentication for bookmark and link-management operations.
- Enforce authorization for every user-owned record, including API endpoints and bulk operations.
- Protect cookie-authenticated state-changing requests against CSRF; store password verifiers using a modern password-hashing algorithm.
- Validate redirect destinations as HTTP or HTTPS URLs. Redirects are an intentional product feature, but account creation and link creation need rate limits and abuse controls.
- Trust forwarded host and client-IP headers only from explicitly configured proxies.
- Bound request sizes and stored user-agent and referrer lengths. Define analytics retention and deletion behavior because these fields can contain sensitive information.
- Keep secrets in deployment configuration, not in the image or repository.

## 9. Operations and Deployment

- Apply versioned database migrations. In PostgreSQL mode with multiple application instances, migration execution must be serialized or run as a separate deployment step.
- Provide liveness and readiness endpoints, structured logs, and metrics for request latency, redirect lookups, database errors, analytics queue depth, dropped events, and migration status.
- On shutdown, stop accepting new requests, drain the analytics queue within a configured deadline, and report any events that could not be persisted.
- Run the container as a non-root user. Mount persistent storage for SQLite data and any retained snapshots or uploaded assets.
- Provide a backup and restore procedure for both database profiles, and document how to verify a restored installation.

Redis and a durable message broker are not required in the initial design. Add them only when a measured need—such as shared rate limiting, cross-instance caching, or durable analytics ingestion—justifies the extra service.

## 10. Performance Targets and Validation

Do not claim a fixed 1 ms redirect time, a fixed idle memory range, or a fixed image size without measurements. Define server-side p50, p95, and p99 targets after establishing the expected workload. Measure cold and warm lookups, concurrent reads, link creation, and analytics ingestion on representative hardware.

Optimize the redirect path first. Add an in-memory slug cache only if measurements show a useful improvement, and define cache invalidation for edits, expiry, and deletion. Track database and queue behavior under load; when SQLite’s single-writer limit becomes material, use the PostgreSQL profile.

## 11. Explicit Assumptions and Open Decisions

- The initial deployment serves multiple accounts from one installation.
- SQLite is the zero-setup default; PostgreSQL is the supported larger-installation option.
- Analytics are best effort until a durable ingestion requirement is accepted.
- Numeric capacity and latency targets remain open until a representative workload and deployment size are defined.
- OIDC, public registration policy, team/organization accounts, and data-sharing rules are product decisions to settle before implementing the corresponding features.

## References

- [SQLite Write-Ahead Logging](https://sqlite.org/wal.html)
- [SQLite PRAGMA documentation](https://www.sqlite.org/pragma.html)
- [SQLite Online Backup API](https://sqlite.org/backup.html)
- [Go downloads and release versions](https://go.dev/dl/)

