# ADR 0009: Recent authentication for PAT operations

- Status: Accepted
- Date: 2026-07-28
- Repository: `TakashiSasaki/git-hosting-service`

## Context

ADR 0008 introduces a Firebase-authenticated self-service API for issuing, listing, and revoking personal access tokens. Issuing a PAT creates a new long-lived credential, while listing and revocation are primarily inspection and incident-response operations. The service must decide where step-up authentication is required without making emergency revocation unnecessarily difficult.

## Decision

Recent Firebase authentication MUST be required when issuing a new PAT, but MUST NOT be required merely to list or revoke existing PATs.

### Issuance

`POST /api/v1/personal-access-tokens` MUST require:

- a valid Firebase ID token; and
- evidence that the user's Firebase authentication occurred within the configured recent-authentication window.

The service SHOULD determine recency from the verified token's authentication-time claim rather than from token issuance time. A freshly refreshed ID token does not by itself prove that the user recently reauthenticated.

If the authentication is too old, PAT issuance MUST fail without creating a token or persisting token metadata. The response MUST tell the client that recent authentication is required without exposing authentication internals.

The initial recommended recent-authentication window is 10 minutes. This duration is an operator-configurable security value rather than a permanent public API constant.

### Listing

`GET /api/v1/personal-access-tokens` requires a valid Firebase ID token but does not require recent authentication.

### Revocation

`DELETE /api/v1/personal-access-tokens/{tokenId}` requires a valid Firebase ID token but does not require recent authentication.

Revocation must remain readily available during suspected credential compromise. Requiring a fresh login for revocation could delay containment and is therefore intentionally avoided.

### PAT authentication

A PAT MUST NOT be accepted for any PAT-management endpoint, including issuance, listing, or revocation.

## Consequences

- A stolen but old browser session cannot be used directly to mint a new long-lived Git credential.
- Token inspection and emergency revocation remain available with a normally valid Firebase session.
- The Web UI must be able to trigger Firebase reauthentication and then retry issuance.
- The server must distinguish authentication time from ID-token issuance or refresh time.
- Failure to satisfy the recent-authentication requirement must not partially create a PAT.

## Remaining related details

The following are implementation details rather than additional architectural blockers:

- exact HTTP status and error-envelope code for `RECENT_AUTHENTICATION_REQUIRED`;
- exact configuration key for the recency window;
- provider-specific Web UI reauthentication flows;
- clock-skew tolerance;
- exact expiration and label validation limits.