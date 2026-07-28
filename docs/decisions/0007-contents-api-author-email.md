# ADR 0007: Contents API explicit author email

- Status: Accepted
- Date: 2026-07-28
- Repository: `TakashiSasaki/git-hosting-service`

## Context

ADR 0004 permits a Contents API `PUT` or `DELETE` request to provide an explicit `author` object while requiring the server to generate the committer from the authenticated Principal. The service must determine whether an explicitly supplied author email is restricted to verified identities or is treated as ordinary Git attribution metadata.

## Decision

An explicitly supplied `author.email` MAY be any syntactically valid email address that passes the service's documented validation rules.

The service MUST treat the explicit author identity as attribution metadata only.

- It does not prove that the named person authenticated to the service.
- It does not prove ownership or verification of the supplied email address.
- It MUST NOT replace or alter the server-generated committer identity.
- The authenticated Principal remains attributable through the committer identity and, where enabled, the audit log.

The Principal's stored default committer email remains restricted by ADR 0006 to either a verified Firebase email or the Principal-specific service-generated noreply address.

## Consequences

- The API can represent original authors, delegated commits, imported attribution, and content supplied by another person.
- The API remains conceptually compatible with Git and the documented GitHub Contents API subset.
- Consumers MUST NOT treat `author.email` as verified identity data.
- API documentation and response models must distinguish author attribution from authenticated actor identity.

## Implementation guidance

Exact length limits, Unicode handling, control-character rejection, and email syntax validation are non-blocking implementation details and should be documented as conservative service limits.
