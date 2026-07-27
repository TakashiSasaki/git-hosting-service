# ADR 0006: Principal commit identity editing

- Status: Accepted
- Date: 2026-07-28
- Repository: `TakashiSasaki/git-hosting-service`

## Context

ADR 0005 requires each Principal to have a persistent default Git commit identity stored in SQLite. The initial implementation must determine whether authenticated users can edit that identity and, in particular, how much freedom they have to choose the email address used as the server-generated committer identity.

## Decision

The authenticated Principal MAY edit their own stored default commit identity, subject to the following restrictions.

### Commit name

`commit_name` MAY be changed to any value that passes the service's documented commit-name validation rules.

### Commit email

`commit_email` MUST be selected from service-approved identities only:

- a currently verified Firebase email associated with the authenticated Principal; or
- the stable service-generated noreply address assigned to that Principal.

The API MUST NOT accept an arbitrary unverified email address as the Principal's default committer email.

### Effect of changes

- A successful identity update applies only to commits created after the update.
- Existing Git commits are immutable and MUST NOT be rewritten.
- The stored values remain independent of later Firebase profile changes; Firebase is used only to determine whether an email is currently eligible for selection.
- The service MUST continue to record the authenticated Principal independently of Git commit metadata where audit logging is enabled.

## Consequences

- Users can correct or improve the human-readable commit name.
- Users can choose between a verified Firebase email and a privacy-preserving noreply identity.
- The server-generated committer identity cannot be used to impersonate an arbitrary email address.
- The Principal API must expose a read/update contract for the stored commit identity.

## Remaining related decisions

- exact endpoint and request/response shape for reading and updating the identity;
- exact name and email validation limits;
- whether an explicitly supplied Contents API `author.email` must also be verified;
- exact noreply address format.