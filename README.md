# git-hosting-service

A self-hosted Git hosting service providing Git Smart HTTP, Firebase Authentication, personal access tokens, repository lifecycle management, REST APIs, and an administrative CLI.

This repository is currently in the specification-first phase. The architecture and major behavioral contracts are being fixed before implementation begins.

## Design document

See [DESIGN.md](./DESIGN.md) for the detailed system design, fixed decisions, provisional data model, operational model, API contracts, and remaining blocking decisions.

## Current implementation scope

The initial implementation targets a single directly installed Linux host with:

- Nginx as the only externally exposed HTTP server
- one loopback-only Node.js process for REST APIs and Git authorization
- `fcgiwrap` and `git-http-backend` for Git Smart HTTP data transfer
- Firebase Authentication for browser/API identity
- personal access tokens for Git over HTTP Basic authentication
- SQLite through `better-sqlite3`
- systemd-managed services, sockets, and migrations
- bare repositories stored on the local filesystem

The repository does not yet contain an implementation. Open design questions are listed in the final section of [DESIGN.md](./DESIGN.md).
