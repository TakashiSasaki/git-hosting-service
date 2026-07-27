# ADR 0003: Contents API write concurrency

- Status: Accepted
- Date: 2026-07-28
- Repository: `TakashiSasaki/git-hosting-service`

## Context

The initial implementation includes GitHub Contents API-compatible `PUT` and `DELETE` operations that create commits and update existing branches. These writes can race with other Contents API requests and with Git Smart HTTP pushes handled by `git-receive-pack` in a separate process.

The service therefore needs a consistency mechanism that does not depend on an in-process mutex and does not require a custom lock shared between Node.js and `git-receive-pack`.

## Decision

Contents API writes MUST use Git-native optimistic concurrency and an atomic ref compare-and-swap operation.

For every `PUT` or `DELETE` request, the service MUST:

1. resolve the target existing branch;
2. record the branch head commit as `expectedOldCommit`;
3. validate the requested path and, when required, the caller-supplied expected blob SHA;
4. create the new blob, tree, and commit objects without updating the branch ref;
5. atomically update the branch ref only if it still points to `expectedOldCommit`.

The ref update is conceptually equivalent to:

```text
git update-ref refs/heads/<branch> <newCommit> <expectedOldCommit>
```

If the branch no longer points to `expectedOldCommit`, the operation MUST fail as a concurrency conflict and MUST NOT overwrite the newer branch state.

## Interaction with Git Smart HTTP

No repository-wide lock shared with `git-receive-pack` will be introduced in the initial implementation.

A concurrent Git push and a Contents API write compete through Git's native atomic ref-update mechanism:

- whichever operation first updates the branch from the expected old commit succeeds;
- an operation whose expected old commit is stale fails;
- the service MUST NOT automatically rebase, merge, or retry the user's content change against the new branch head.

## Interaction with blob SHA validation

For an existing file update or deletion, the GitHub-compatible request blob `sha` MUST be validated against the blob currently reachable at the requested path from `expectedOldCommit`.

Blob SHA validation and branch-head compare-and-swap solve different problems and both are required:

- blob SHA detects that the requested file version differs from the caller's expectation;
- branch-head CAS detects any concurrent branch change, including changes elsewhere in the tree.

For creating a new file, the target path MUST not already exist at `expectedOldCommit`; no blob SHA is supplied.

## Failure behavior

A stale blob SHA or failed branch-head compare-and-swap MUST produce a documented conflict response. The exact HTTP error envelope and compatibility-level status mapping will be defined with the remaining Contents API request and response contract.

Objects created before a failed ref update may remain unreachable. They MUST NOT be exposed as successful writes and MAY later be reclaimed by normal Git garbage collection.

## Consequences

- Contents API writes remain safe against concurrent Git pushes without a cross-process application lock.
- Writes to different branches do not block each other unnecessarily.
- The implementation must create Git objects before publishing them through an atomic ref update.
- The service must treat the ref update as the transaction commit point.
- Node.js-only queues or mutexes are insufficient as the primary consistency mechanism.

## Remaining related decisions

- exact author and committer policy
- exact conflict HTTP status and response subset
- path, file-size, symlink, and Base64 restrictions
- whether an additional in-process per-branch queue is used only as an optimization, not as the correctness mechanism
