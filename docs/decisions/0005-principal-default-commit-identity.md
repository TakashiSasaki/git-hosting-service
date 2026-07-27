# ADR 0005: Principal default commit identity

- Status: Accepted
- Date: 2026-07-28
- Repository: `TakashiSasaki/git-hosting-service`

## Context

The initial Contents API creates Git commits. ADR 0004 requires the server-generated committer, and the default author when no explicit author is supplied, to come from the authenticated Principal. Firebase profile claims may be absent or may change independently of the service, so commit generation cannot depend on live Firebase claims for every request.

## Decision

Each Principal MUST have a persistent default Git commit identity stored in SQLite.

The identity consists of at least:

- `commit_name`
- `commit_email`

The service MUST use this stored identity:

- as the committer for every Contents API `PUT` and `DELETE` operation performed by that Principal;
- as the author when the request omits an explicit `author` object.

The stored values MUST NOT be recomputed from the current Firebase token on every request. Later Firebase profile changes therefore do not silently rewrite the Principal's Git identity.

## Initial provisioning

When a Principal is first created, the service SHOULD initialize the stored identity from available Firebase claims:

- use a valid Firebase display name as the initial commit name when available;
- otherwise generate a stable service-defined name derived from the Principal;
- use a verified Firebase email as the initial commit email when available;
- otherwise generate a stable service-local noreply address derived from the Principal ID and service host.

The exact validation limits and the initial-implementation editing API remain separate decisions.

## Consequences

- Commit creation remains possible even when Firebase provides no display name or email.
- Commit identity remains stable across Firebase profile changes.
- A deterministic noreply identity can preserve privacy and guarantee a syntactically valid email address.
- SQLite schema and Principal creation must include the persistent commit identity.
- Any future identity-edit endpoint must update the stored values explicitly rather than relying on Firebase synchronization.

## Remaining related decisions

- whether the authenticated Principal may edit the stored commit identity in the initial implementation;
- exact commit name and email validation rules;
- whether user-supplied author email addresses must be verified;
- exact noreply address format.