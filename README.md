# Scurry
Scurry around the web, gather your links, and stash them safely on your own server.
**A self-hosted home for short links and bookmarks.**

Scurry is an early-stage project exploring how to combine URL shortening and bookmark management in one multi-account installation. It is being designed for personal use first, with a path for larger self-hosted deployments.

## Project Status

Scurry is in the architecture and early development stage. The feature list below describes the intended direction; features, installation instructions, and deployment profiles are not yet confirmed as implemented. Do not rely on it for production use yet.

## Planned Capabilities

- Create and manage short links, including custom slugs and optional link settings.
- Save bookmarks with notes, tags, and destination metadata.
- Keep each account’s links and bookmarks isolated.
- Collect click analytics for short links, with a documented retention policy.
- Offer a web interface and, as the API takes shape, support integrations.

The exact MVP scope is still being defined.

## Design

Scurry is planned as a modular Go application with a server-rendered web interface using HTMX. Tailwind CSS will be compiled during the build, and the resulting static assets will be embedded with the application.

The deployment profiles are:

- **SQLite by default** for a simple, single-host installation using persistent local storage.
- **PostgreSQL as an option** for installations that need more write concurrency or multiple application instances.

SQLite WAL supports concurrent readers but only one writer at a time. Scurry’s SQLite profile is intended for a local database file, not a file shared across hosts. PostgreSQL is the planned path for larger multi-account installations. See [architecture.md](architecture.md) for the detailed design and tradeoffs.

Click analytics are planned as best-effort in the initial design. A process crash or a full in-memory queue may lose queued events; Scurry will not claim lossless analytics unless a durable ingestion path is implemented.

## Getting Started

There is no runnable release or verified installation procedure yet. Setup and development commands will be added after the initial application structure and build workflow are in place.

## Roadmap

1. Confirm the initial short-link, bookmark, and account workflows.
2. Build the first end-to-end application slice using SQLite.
3. Validate PostgreSQL support and database migrations.
4. Add analytics, retention, backup, and restore workflows.
5. Publish verified installation and contribution instructions.

## Performance

Scurry is designed to keep redirect handling independent from analytics persistence. Performance targets will be set and measured against a defined workload; no fixed end-to-end latency or memory claims are made at this stage.

## License

The project is intended to use the MIT License. Add the repository’s LICENSE file before public release.

