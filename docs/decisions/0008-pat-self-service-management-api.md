# ADR 0008: Personal access token self-service management API

- Status: Accepted
- Date: 2026-07-28
- Repository: `TakashiSasaki/git-hosting-service`

## Context

Git Smart HTTP authentication uses personal access tokens (PATs). The initial implementation must provide a practical mechanism for each Principal to issue, inspect, and revoke their own credentials without requiring routine intervention by the host administrator.

## Decision

The initial implementation MUST provide a Firebase-authenticated self-service REST API through which an authenticated Principal manages only their own PATs.

The initial route family is:

```text
POST   /api/v1/personal-access-tokens
GET    /api/v1/personal-access-tokens
DELETE /api/v1/personal-access-tokens/{tokenId}
```

### Authentication and ownership

- These endpoints MUST require a valid Firebase ID token.
- A PAT MUST NOT be accepted as authentication for PAT-management endpoints.
- A Principal MAY list and revoke only PATs whose `principal_id` is their own.
- A token ID belonging to another Principal and a nonexistent token ID MUST be indistinguishable to the caller.

### Issuance

- `POST` creates a new PAT for the authenticated Principal.
- A human-readable label MUST be supplied.
- Expiration is optional; omission means no expiration under the already accepted PAT lifetime policy.
- The complete plaintext PAT MUST be returned exactly once in the successful issuance response.
- The service MUST NOT persist or log the plaintext PAT.

### Listing

`GET` MUST return metadata only. It MAY include at least:

- `tokenId`
- `label`
- `createdAtMs`
- `expiresAtMs`
- `revokedAtMs`
- `lastUsedAtMs`

The response MUST NOT include the plaintext PAT, the stored hash, or any recoverable secret material.

### Revocation

- `DELETE` irreversibly revokes the selected PAT.
- Revocation sets `revoked_at_ms`; the database row is retained.
- A revoked PAT MUST NOT be reactivated.
- The Principal must issue a new PAT when replacement access is required.

### Administrative operations

The initial implementation does not require a general administrator REST API for managing every Principal's tokens. Emergency inspection or revocation MAY be provided by the administrative CLI and MUST be auditable if audit logging is enabled.

## Consequences

- Normal PAT lifecycle operations do not require server-administrator intervention.
- Firebase authentication remains the higher-trust control plane for creating and revoking Git credentials.
- Compromise of a PAT alone does not permit creation of additional PATs.
- The API and Web UI must prominently state that a newly issued plaintext PAT cannot be retrieved again.

## Remaining related decisions

- whether PAT issuance requires recent Firebase reauthentication;
- whether revocation also requires recent reauthentication;
- exact expiration input representation and bounds;
- exact label validation limits;
- exact list and issuance response shapes;
- last-used timestamp update semantics and write-throttling.
