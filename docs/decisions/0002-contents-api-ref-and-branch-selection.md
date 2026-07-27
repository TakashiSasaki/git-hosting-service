# ADR 0002: Contents API ref and branch selection

- Status: Accepted
- Date: 2026-07-28
- Repository: `TakashiSasaki/git-hosting-service`

## Context

The initial implementation includes a documented GitHub Contents API-compatible subset with `GET`, `PUT`, and `DELETE`. The API therefore needs an explicit contract for selecting the Git object or branch that each operation targets.

## Decision

The service MUST distinguish read selection from write selection.

### GET

`GET` MAY specify an existing ref.

- When `ref` is omitted, the Repository's configured default branch is used.
- When `ref` is present, it MAY resolve to an existing branch, tag, or commit object ID.
- A tag or commit selection is read-only.
- An unknown or invalid ref is rejected and MUST NOT silently fall back to the default branch.

### PUT and DELETE

`PUT` and `DELETE` MAY specify an existing branch.

- When `branch` is omitted, the Repository's configured default branch is used.
- When `branch` is present, it MUST identify an existing branch.
- Tags, detached commit object IDs, and other non-branch refs MUST NOT be accepted as write targets.
- The Contents API MUST NOT create a new branch implicitly in the initial implementation.
- Branch creation remains a Git operation performed through Git Smart HTTP or a future dedicated branch API.

### Missing default branch

If the default branch ref does not exist:

- an operation that omits `ref` or `branch` MUST fail explicitly;
- the service MUST NOT silently select another branch;
- the API error contract will be defined with the remaining Contents API response decisions.

## Consequences

- Repository contents can be read from branches, tags, and historical commits.
- REST writes are restricted to existing mutable branches.
- The API remains compatible in concept with GitHub's separation between read `ref` selection and write `branch` selection.
- Ref parsing, ambiguity handling, and validation must use Git's ref-resolution rules without permitting writes through tags or detached commits.

## Remaining related decisions

- exact GET request and response subset
- exact PUT and DELETE request and response subset
- author and committer policy
- optimistic concurrency and ref compare-and-swap
- coordination with concurrent Git Smart HTTP pushes
- path, file-size, symlink, and Base64 restrictions
