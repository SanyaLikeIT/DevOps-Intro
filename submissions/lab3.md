# Lab 3: A PR-gated pipeline for QuickNotes

## Chosen path and current status

This submission uses GitHub Actions because the repository and pull requests are
hosted on GitHub, so the checks and branch rules are available in the same place.
The baseline pipeline, deliberate failure, recovery on GitHub, and required
status checks have been verified. Task 2 and the performance bonus are still
in progress.

- [Course draft PR](https://github.com/inno-devops-labs/DevOps-Intro/pull/1603)
- [Fork validation PR](https://github.com/SanyaLikeIT/DevOps-Intro/pull/2)

## Task 1: Baseline gate and evidence

The workflow runs three independent jobs on `ubuntu-24.04` with Go `1.24`:
`go vet ./...`, `go test -race -count=1 ./...`, and golangci-lint `v2.5.0`.
All checks run against `app/`. Actions are pinned to full commit SHAs, and
`GITHUB_TOKEN` has only `contents: read` permission. Both setup-go caching and
the lint action's cache were disabled for the baseline; setup-go caching is now
enabled for the next measurement.

| Evidence | Result |
|---|---|
| [Initial run 35305034562](https://github.com/SanyaLikeIT/DevOps-Intro/actions/runs/35305034562) | `vet`, `test`, and `lint` succeeded at commit `9e87967794177539666b6d51de5cbd38f8137f6d`. |
| Deliberate failure commit | `11475036c6b3aa2145a16a66bca12fff83881944` changed the expected health note count from one to two. |
| [Failed run 35305763786](https://github.com/SanyaLikeIT/DevOps-Intro/actions/runs/35305763786) | `test` failed; `vet` and `lint` succeeded. |
| Recovery commit | `e8f856422d2a6912414234ef42c03a7c624c8a2e` restores the original assertion. |
| Recovery validation | Local `go vet ./...` and `go test -race -count=1 ./...` passed using Go 1.24.13. [Recovery run 35306448411](https://github.com/SanyaLikeIT/DevOps-Intro/actions/runs/35306448411) passed all three jobs at commit `43ac23b6129c065beb2e65f7bcff2980f6f78f60`. |

The active ruleset on the fork's `main` requires `vet`, `test`, and `lint` from
GitHub Actions and requires the branch to be up to date. These settings were
confirmed through the public branch-rules API on September 18, 2026;
the relevant response is saved in [required-checks.json](evidence/lab3/required-checks.json).

![Required status checks and strict update policy](evidence/lab3/required-checks.png)

![Failed test marked Required on the fork PR](evidence/lab3/failed-required-check.png)

The second screenshot shows the failed required test and two successful required
checks. It also shows a Draft PR, which independently prevents merging; the
disabled merge button alone is therefore not proof that CI caused the block.
The active required-check rule and failed check establish that normal merging
requires the test to pass. No merge or bypass was attempted.

### Task 1 design questions

**a) Why pin the runner?** `ubuntu-24.04` avoids an automatic migration to a
different Ubuntu release when `ubuntu-latest` changes. Such a migration can
change compilers, system libraries, package availability, and command behavior.
The numbered runner image still receives updates, so this is an OS-release pin,
not a fully immutable environment.

**b) Why separate the jobs?** Independent jobs can run concurrently and report
separate results. A failed test does not hide the vet or lint result. In a single
job, steps normally run sequentially and a failure skips subsequent steps,
reducing diagnostic feedback and potentially increasing elapsed time.

**c) What attack does SHA pinning prevent?** During the March 14-15, 2025
`tj-actions/changed-files` compromise (CVE-2025-30066), attackers redirected
existing version tags to malicious code that exposed secrets in workflow logs.
A reviewed, full commit SHA prevents a moved tag from silently replacing the
selected action code. It does not make an already malicious commit safe or pin
external resources downloaded by the action. Sources:
[GitHub advisory](https://github.com/advisories/GHSA-mrrh-fwg8-r2c3) and
[GitHub action security guidance](https://docs.github.com/en/actions/reference/security/secure-use).

**d) What does `permissions` do?** It controls the permissions granted to the
workflow's `GITHUB_TOKEN`. Granting only `contents: read` applies least privilege:
these checks need repository access but do not need permission to modify code,
publish releases, or administer pull requests. Limiting token permissions reduces
the impact of a compromised action. See the
[GitHub security guidance](https://docs.github.com/en/actions/reference/security/secure-use).

**e) GitLab theory: stage, job, and dependencies.** This submission uses GitHub
Actions, but in GitLab a job defines executable work and a stage groups jobs.
Jobs in a stage can run concurrently, while stages normally run in order.
`dependencies` selects which earlier jobs' artifacts a job downloads; it does
not define stage ordering or a scheduling dependency graph. `dependencies: []`
disables those artifact downloads. See the
[GitLab YAML reference](https://docs.gitlab.com/ci/yaml/#dependencies).

## Task 2: Timing and optimizations in progress

Two successful baseline runs took **36 seconds** and **32 seconds**, giving a
**34-second median**. Each measurement spans run creation to the last job
completion and includes initial queue time. Both runs used identical application
code and workflow settings; the recovery head also added documentation. The
deliberately failed run is excluded. This two-run sample is smaller than the
recommended three to five runs: authenticated reruns were unavailable, so these
results are preliminary rather than a stable performance estimate.

[Baseline timing evidence](evidence/lab3/baseline-timings.json) records the run
URLs, revisions, timestamps, job results, and per-step timings returned by the
GitHub API. Job times overlap and must not be added to calculate wall-clock time.
The interval before a job starts combines queueing and provisioning; the API does
not isolate runner provisioning time.

| Scenario | Wall-clock |
|---|---:|
| Baseline: no cache, single Go version, no path filter | 34 s (median of two successful runs: 36 s, 32 s) |
| With cache | 30 s (first population run); warm-cache measurement pending. |
| With cache and matrix | TODO: Implement and measure. |

QuickNotes currently has no third-party module dependencies: `app/go.mod` has
no `require` block and there is no `app/go.sum`. Module-download caching therefore
has no dependency downloads to accelerate; build-cache effects must be measured.

### Cache implementation

The three jobs now use setup-go caching for the Go module and build caches.
The dependency-path input hashes `app/go.mod` and `app/go.sum`; the existing
`go.mod` supplies a deterministic input even though `go.sum` is absent. If
third-party dependencies are introduced later, their checksums will also
participate in the key. The pinned action includes the operating system,
architecture, Go version, and dependency-file hash in its cache key. Go itself
validates cached compilation results against source and build inputs.

The jobs share a cache key for the same platform, toolchain, and dependency
inputs. Concurrent cache saves can race: the first successful save supplies the
entry, so a later run may still compile packages missing from that entry. The
linter action's separate cache remains disabled to isolate this change. A first
run may only populate the cache; no cache hit or speed improvement is claimed
until a subsequent run confirms it. See the pinned setup-go
[cache implementation](https://github.com/actions/setup-go/blob/d35c59abb061a4a6fb18e82ac0862c26744d6ab5/src/cache-restore.ts)
and [cache directories](https://github.com/actions/setup-go/blob/d35c59abb061a4a6fb18e82ac0862c26744d6ab5/src/package-managers.ts).

### First cache population run

[Run 35306668705](https://github.com/SanyaLikeIT/DevOps-Intro/actions/runs/35306668705)
passed all three jobs at commit `20eccd7f283b2f24683d748bff77d37fae04ff1d`.
It took **30 seconds**, measured from run creation to final job completion.
The [API evidence](evidence/lab3/cache-cold-run.json) includes all job and step
timestamps plus the cache inventory after the run.

GitHub reports cache ID `7832179942`, scoped to `refs/pull/2/merge`, with
20,777,241 bytes stored. Its key identifies Ubuntu 24, Go 1.24.13, and the
current dependency-input hash. The entry was created at
`2026-09-18T04:22:14.943621Z`, after all three Go setup steps completed.
This establishes cache population, not a warm-cache hit. The difference from
the 34-second baseline median cannot be attributed to restored cache data;
runner variation and the small sample also affect the result.

The next documentation-only push keeps the application, workflow, and dependency
inputs unchanged. Path filtering is not enabled yet, so that push triggers a
comparable run on the same PR with the existing cache available. Record its
wall-clock time and verify cache access before adding the version matrix.

TODO: Verify cache restoration on GitHub, implement the Go 1.23/1.24 matrix,
the aggregate gate, updated required checks, and docs-only path filtering.
Answer questions f-h using the final implementation.

## Performance bonus in progress

TODO: Preserve actual step timing data, apply and measure at least three
additional optimizations, provide a before/after table, and write the required
four-to-six-sentence bottleneck analysis.
