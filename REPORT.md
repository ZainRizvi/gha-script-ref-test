# GitHub Actions Workflow/Checkout Version Mismatch in PyTorch CI

## The Problem

When scripts or files referenced by GitHub Actions workflow YAML are moved on `main`, in-flight PRs that branched before the move break with `No such file or directory` errors — even though the PR didn't touch any workflow files.

This affects **19 of 26** PR-triggered workflows in `pytorch/pytorch`, including all build, test, lint, and docs jobs.

## How `pull_request` Workflows Normally Work

For `pull_request` events, GitHub creates a synthetic merge commit at `refs/pull/N/merge` — the result of merging the PR branch into the base branch. GitHub then:

1. Reads the workflow YAML from this merge commit
2. Sets `GITHUB_SHA` to this merge commit
3. Sets `GITHUB_REF` to `refs/pull/N/merge`
4. `actions/checkout@v4` (with default settings) checks out this same merge commit

Because the workflow YAML and the checked-out code come from the same commit, everything is self-consistent. If scripts moved on `main`, the merge commit has both the updated workflow references *and* the moved scripts. Nothing breaks.

**We verified this across 12 permutations** (regular/reusable workflows × direct run/relative action/branch-ref action × `.github/scripts`/`ci/` locations). All passed. ([PR #3](https://github.com/ZainRizvi/gha-script-ref-test/pull/3), [PR #5](https://github.com/ZainRizvi/gha-script-ref-test/pull/5), [PR #9](https://github.com/ZainRizvi/gha-script-ref-test/pull/9))

## Root Cause

PyTorch's `checkout-pytorch` composite action overrides the default checkout behavior:

```yaml
# .github/actions/checkout-pytorch/action.yml, lines 59 and 107
ref: ${{ github.event_name == 'pull_request' && github.event.pull_request.head.sha || github.sha }}
```

This checks out the **PR's HEAD commit** instead of the merge commit. But the workflow YAML being executed still comes from the merge commit — this is a GitHub platform behavior that cannot be overridden.

The result is a split:

| Component | Source | Contains |
|-----------|--------|----------|
| Executing workflow YAML | Merge commit (`refs/pull/N/merge`) | Main's updated script paths |
| Checked-out code on disk | PR HEAD (`pull_request.head.sha`) | Pre-move script paths |

When the workflow runs `bash .github/scripts/moved/foo.sh` (the path from the merged workflow YAML), the file doesn't exist on disk (the checkout has it at `.github/scripts/foo.sh`).

**We reproduced this exactly.** Modified 4 existing workflows to use head-sha checkout, opened a PR from before the script move. The direct `run:` workflows failed with `No such file or directory`. ([PR #10](https://github.com/ZainRizvi/gha-script-ref-test/pull/10))

## When It Manifests

The mismatch only causes failures when **all three conditions** are true:

1. A file referenced by a workflow's `run:` command is moved/renamed on `main`
2. An open PR was branched before the move (or hasn't rebased since)
3. The checkout uses `ref: pull_request.head.sha` instead of the default merge commit

It does **not** manifest when:
- Using default `actions/checkout` behavior (merge commit) — everything is self-consistent
- Using relative composite actions (`uses: ./.github/actions/...`) — the `action.yml` is read from the checkout workspace, so it stays consistent with the checked-out code
- The PR has a merge conflict — GitHub can't create the merge commit, so no workflows run at all

### Resolution Hierarchy

Not all references resolve from the same source. We experimentally verified each:

| Reference type | Resolved from | Consistent with checkout? |
|----------------|--------------|--------------------------|
| Root workflow YAML (`on: pull_request`) | Merge commit | No (when using head-sha checkout) |
| Reusable workflows (`uses: ./.github/workflows/...`) | Merge commit | No (same as root — resolved at planning time) |
| Composite actions (`uses: ./.github/actions/...` in steps) | Checkout workspace | **Yes** (resolved during step execution) |
| `run:` commands | Text from executing workflow YAML | No (workflow is from merge commit) |
| Branch-ref actions (`uses: ...@main`) | Always from `main` | Depends on timing |

Evidence: In [PR #10](https://github.com/ZainRizvi/gha-script-ref-test/pull/10), direct `run:` workflows failed but relative action workflows passed — the `action.yml` was read from the checkout (PR HEAD), had old paths, and scripts were at old paths in the checkout. In [PR #11](https://github.com/ZainRizvi/gha-script-ref-test/pull/11), an existing reusable workflow executed `bash .github/scripts/moved/direct-reusable.sh` (main's path), confirming reusable workflows also come from the merge commit.

## Affected Scope in PyTorch

The head-sha checkout propagates through two vectors:

**Vector 1: `checkout-pytorch/action.yml`** (lines 59, 107) — used by:
- `_linux-build.yml`, `_linux-test.yml`, `_linux-test-stable-fa3.yml`
- `_mac-build.yml`, `_mac-test.yml`
- `_win-build.yml`, `_win-test.yml`
- `_rocm-test.yml`, `_xpu-test.yml`
- `_docs.yml`, `_bazel-build-test.yml`
- `target_determination.yml`
- `lint-autoformat.yml`, `nitpicker.yml`

**Vector 2: Explicit `ref:` in `lint.yml`** — passes `ref: head.sha` to 7 `linux_job_v2` calls and uses `checkout-pytorch@main` for 5 inline jobs.

**Not affected** (7 workflows): `check_mergeability_ghstack.yml`, `auto_request_review.yml`, `runner-determinator-validator.yml`, `_get-changed-files.yml`, `_runner-determinator.yml`, `job-filter.yml`, `llm_td_retrieval.yml` — these either use default checkout or don't check out at all.

## Documentation

GitHub documents the individual behaviors but does not warn about the mismatch:

**Workflow version selection** ([source](https://docs.github.com/en/actions/using-workflows/about-workflows#triggering-a-workflow)):
> *"Each workflow run will use the version of the workflow that is present in the associated commit SHA or Git ref of the event."*

**`pull_request` event context** ([source](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#pull_request)):
> *GITHUB_SHA: Last merge commit on the GITHUB_REF branch.*
> *GITHUB_REF: PR merge branch `refs/pull/PULL_REQUEST_NUMBER/merge`.*

Combining these: the workflow YAML for `pull_request` events is read from the merge commit. This is a platform behavior — `actions/checkout`'s `ref:` parameter controls what code is on disk, not which workflow YAML is executed.

**Merge conflicts** ([source](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#pull_request)):
> *"Workflows will not run on `pull_request` activity if the pull request has a merge conflict."*

We verified this across three scenarios: conflicts in workflow files ([PR #4](https://github.com/ZainRizvi/gha-script-ref-test/pull/4)), non-workflow files ([PR #6](https://github.com/ZainRizvi/gha-script-ref-test/pull/6)), and single workflow files ([PR #7](https://github.com/ZainRizvi/gha-script-ref-test/pull/7)). In all cases, zero workflows ran.

## Why PyTorch Uses Head-SHA Checkout

This was a deliberate choice by Michael Suo in [PR #71974](https://github.com/pytorch/pytorch/pull/71974) (Feb 2022), consolidated into `checkout-pytorch` in [PR #74327](https://github.com/pytorch/pytorch/pull/74327) (Mar 2022). The rationale:

- **Reproducibility**: Test the exact code the developer wrote and tested locally, not a synthetic merge that includes whatever landed on `main` since the PR was created
- **Consistency**: Eliminate the situation where some CI jobs tested the merge commit while others tested the head commit
- **Predictability**: Re-running CI always tests the same commit (the merge commit changes as `main` advances)
- **Squash-merge alignment**: PyTorch uses squash-merge, which is better modeled by testing the head commit

This was not a security decision. The tradeoff was acknowledged in the PR: *"The primary disadvantage is that now when re-running workflows, you will not be re-running against a 'rebased' version of the PR."*

The behavior is documented in `CONTRIBUTING.md` (lines 1352-1384).

## Test Repository

All experiments are reproducible at [github.com/ZainRizvi/gha-script-ref-test](https://github.com/ZainRizvi/gha-script-ref-test). Each PR isolates a single variable and has CI results showing pass/fail for all permutations.
