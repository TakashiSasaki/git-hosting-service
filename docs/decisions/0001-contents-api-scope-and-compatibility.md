# ADR 0001: Contents API scope and compatibility

- Status: Accepted
- Date: 2026-07-28
- Repository: `TakashiSasaki/git-hosting-service`

## Context

The initial implementation must expose repository contents through REST in addition to Git Smart HTTP. The design needed to determine both the supported operations and the degree of GitHub API compatibility.

## Decision

The initial implementation MUST include all three core Contents API operations:

- `GET`: retrieve a file or list a directory
- `PUT`: create or replace a file and create a Git commit
- `DELETE`: delete a file and create a Git commit

The public contract MUST be a documented subset of the GitHub Contents API rather than an unrelated custom API.

The intended route family is:

```text
GET    /api/v1/repos/{principalId}/{repositoryName}/contents/{path}
PUT    /api/v1/repos/{principalId}/{repositoryName}/contents/{path}
DELETE /api/v1/repos/{principalId}/{repositoryName}/contents/{path}
```

The subset is expected to use GitHub-style concepts including:

- Base64-encoded file content
- blob SHA for update and delete conflict detection
- commit message
- branch selection, subject to a separate design decision
- GitHub-style file, directory, content, and commit response fields

Compatibility is limited to the documented subset. The service does NOT promise complete drop-in compatibility with all GitHub SDKs, authentication mechanisms, media types, or every behavior of GitHub's REST API.

## Consequences

The initial implementation must also solve:

- commit author and committer identity
- optimistic concurrency
- branch/ref compare-and-swap
- coordination with concurrent Git Smart HTTP pushes
- repository-level serialization or an equivalent consistency mechanism
- validation of content paths, file sizes, symlinks, and Base64 payloads

## Remaining related decisions

- default branch only versus arbitrary branch selection
- exact GET file and directory response subset
- exact PUT and DELETE request/response subset
- author and committer policy
- concurrency and ref-update algorithm
- path, size, and symlink restrictions
