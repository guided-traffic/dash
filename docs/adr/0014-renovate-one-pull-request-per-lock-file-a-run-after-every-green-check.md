# ADR 0014: Renovate: One Pull Request per Lock File, a Run After Every Green Check

## Status

Accepted 2026-11-08 (the owner: the Renovate pull requests should land on `main` and run through
better from now on). Amends [ADR 0006](0006-one-version-of-every-tool.md) D4. **Built** 2026-11-08
in [renovate.json](../../renovate.json), [.github/workflows/renovate.yml](../../.github/workflows/renovate.yml)
and [.github/workflows/release.yml](../../.github/workflows/release.yml). Verified:
`renovate-config-validator --strict` (Renovate 44.222.4) accepts the file; a local dry run
(`--platform=local --dry-run=lookup`) of `main` before this change, with this configuration, put the
open minor and patch updates into `renovate/go-modules` (four modules) and `renovate/frontend-npm`
(Angular, vitest, Playwright) and listed no update of `k8s.io/kube-openapi`. **Not verified:** the run
after a check (D2) has not run on GitHub yet.

## Context

On 2026-11-08 thirteen Renovate pull requests were open, #1 and #32 to #44; on 2026-10-28 the owner
had already brought sixteen of them to `main` by hand (`1548a72`). Verified in the repository and its
runs:

- Release Tooling failed on twelve of them, and on nine of those it was the only failure. The job set
  up Node from `lts/*`; the alias had moved to Node 26, which exits 127 on the runner without
  `libatomic1`. Nothing in the repository had changed.
- Ten of them rewrote `backend/go.sum` or `frontend/package-lock.json`. A merge puts the others in
  conflict; Renovate rebases them on its next run, and it runs once a night (three minutes). So at most
  one update per lock file reached `main` a day.
- The repository does not allow auto-merge and no rule protects `main`, so `platformAutomerge` has no
  effect: Renovate merges a green pull request itself, on its next run. A pull request green at eight
  in the morning waits until the next night.
- On 2026-11-05 GoSec and the end-to-end tests of #39 and #40 ended after 12 to 18 minutes without a
  conclusion on their steps and without a log. **Not verified:** the cause. The checks of four pull
  requests (#1, #38, #39, #40) started within two minutes of each other, each with fifteen jobs on the
  self-hosted runners, and that day's Renovate run took 293 minutes; contention is the likely reason.

## Decision

**D1 — One pull request per lock file for minor, patch and digest updates.** `Go modules` for the
Go modules (`golang.org/x` stays with the Go version group of ADR 0006 D4); `Frontend npm` for
`frontend/package.json`, Angular's minors and patches included; `Release tooling` for the root
`package.json`; `Icon packs` as before. A major stays a pull request of its own and waits for a person;
an Angular major keeps the `Angular` group.

**D2 — Renovate runs again after every green Test and Release run on one of its own branches**, beside
the nightly run: a `workflow_run` trigger for branches `renovate/**` of this repository, a run only on
`success`, one run at a time. A green pull request is merged minutes after its last check, and the run
rebases what that merge put in conflict, whose checks start the next run.

**D3 — `k8s.io/kube-openapi` is never updated by itself.** It moves with the Kubernetes modules, at the
commit their release requires (`go mod tidy`). Its own commits run ahead: #1 (`5bee61f`) needs
structured-merge-diff v7 while `k8s.io/apimachinery` v0.37.2 builds against v6, and the backend does not
compile.

**D4 — Every job with Node.js runs the major of `NODE_VERSION`**, the release jobs included, never an
alias like `lts/*`: the one Node of ADR 0006 D1, moved by Renovate in a pull request that CI tests.

## Consequences

- One failing update holds its whole group. The group's checks name the failing step; the update is
  taken out with a `packageRule` or brought by hand, as #40 was.
- Every finished check on a Renovate branch costs one Renovate run of about three minutes on the
  runners; a check that fails starts none.
- The run after a check holds the GitHub App's token, as the nightly run does. It runs the workflow
  of `main` and checks out `main`, never the pull request's code, and the condition on the head
  repository ignores a fork's branch that is named `renovate/` too; a pull request cannot start code
  of its own with the token.
- A pull request opened in the nightly run is usually on `main` the same morning, not one per night.
- The open pull requests that the change of 2026-11-08 brought to `main` are closed by Renovate itself
  on its next run (`- autoclosed`), #1 too, as D3 disables its update.

## Alternatives considered

- **A ruleset on `main` with required checks and auto-merge allowed**, so GitHub merges the moment the
  checks pass (`platformAutomerge`). Rejected for now: the owner pushes to `main` directly, which a
  ruleset would block without a bypass; it is a repository setting outside the code; and a pull request
  in conflict still waits for Renovate to rebase it.
- **An hourly schedule.** Twenty-four runs a day, nearly all with nothing to do, and still up to an hour
  per step.
- **Separate pull requests rebased whenever behind `main`.** Every merge rebases all the others and
  starts their checks at once: the load of 2026-11-05, and conflicts on the same lock files again.

## References

- [renovate.json](../../renovate.json), [.github/workflows/renovate.yml](../../.github/workflows/renovate.yml),
  [.github/workflows/release.yml](../../.github/workflows/release.yml)
- [docs/developer/ci-and-release.md](../developer/ci-and-release.md)
