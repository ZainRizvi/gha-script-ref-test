# GitHub Actions Script Reference Behavior Test

When scripts referenced by GitHub Actions workflows are moved to a new location on `main`, do PRs that branched off *before* the move break?

**TL;DR:** With default GitHub Actions behavior, **no** — everything works because GitHub uses a synthetic merge commit. But if your CI explicitly checks out `github.event.pull_request.head.sha` (as PyTorch does), then **yes** — direct `run:` commands referencing moved scripts will break.

## Official GitHub Documentation

The behaviors observed in these experiments are documented by GitHub:

### Merge commit for `pull_request` events

> **GITHUB_SHA**: Last merge commit on the `GITHUB_REF` branch.
> **GITHUB_REF**: PR merge branch `refs/pull/PULL_REQUEST_NUMBER/merge`.

Source: [Events that trigger workflows — `pull_request`](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#pull_request)

### Workflow file version selection

> Each workflow run will use the version of the workflow that is present in the associated commit SHA or Git ref of the event.

Since `GITHUB_SHA` for `pull_request` is the merge commit, the workflow YAML is read from the merge commit — not the PR head.

Source: [About workflows — Triggering process](https://docs.github.com/en/actions/using-workflows/about-workflows#triggering-a-workflow)

### Merge conflicts prevent workflow runs

> Workflows will not run on `pull_request` activity if the pull request has a merge conflict; the merge conflict must be resolved first.

Source: [Events that trigger workflows — `pull_request`](https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows#pull_request)

### Checking out PR HEAD instead of merge commit

> If you want to get the commit ID for the last commit to the head branch of the pull request, use `github.event.pull_request.head.sha` instead.

The `actions/checkout` action [documents this pattern](https://github.com/actions/checkout#checkout-pull-request-head-commit-instead-of-merge-commit) for checking out the PR HEAD:
```yaml
- uses: actions/checkout@v4
  with:
    ref: ${{ github.event.pull_request.head.sha }}
```

## Official GitHub Documentation

GitHub documents that for `pull_request` events, each workflow run uses the version of the workflow file from the merge commit (`refs/pull/N/merge`), while the checked-out code can be overridden to the PR HEAD via `ref: github.event.pull_request.head.sha`. This is the root cause of the mismatch — workflow YAML from the merge commit references moved paths, but the checkout has pre-move code.

- [Workflow triggering process](https://docs.github.com/en/actions/using-workflows/about-workflows#triggering-a-workflow): *"Each workflow run will use the version of the workflow that is present in the associated commit SHA or Git ref of the event."*
- [`pull_request` event](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#pull_request): Documents that `GITHUB_SHA` is the merge commit and `GITHUB_REF` is `refs/pull/N/merge`.
- [actions/checkout — Checkout PR HEAD](https://github.com/actions/checkout#checkout-pull-request-head-commit-instead-of-merge-commit): Documents how to override the default merge commit checkout with `ref: ${{ github.event.pull_request.head.sha }}`.

## Background

This repo was created to systematically test which combinations of workflow type, script invocation method, and script location trigger failures when scripts move on `main` while a PR is still open from before the move.

## Test Matrix (3 Dimensions)

**Dimension 1 — Workflow type:**
- A) Regular workflow (triggered by `pull_request`)
- B) Reusable workflow (called via `uses: ./.github/workflows/...`)

**Dimension 2 — How the script is invoked:**
1. Direct `run:` command in a job step (e.g., `bash scripts/foo.sh`)
2. GitHub Action referenced via relative path (e.g., `uses: ./.github/actions/my-action`)
3. GitHub Action referenced with branch (e.g., `uses: org/repo/.github/actions/my-action@main`)

**Dimension 3 — Where the script lives:**
- X) Inside `.github/` (e.g., `.github/scripts/do-thing.sh`)
- Y) Outside `.github/` (e.g., `ci/do-thing.sh`)

This gives **2 x 3 x 2 = 12 permutations**, each in its own workflow file.

| # | Workflow Type | Invocation Method | Script Location | Workflow File |
|---|---------------|-------------------|-----------------|---------------|
| 1 | Regular | Direct run | .github/scripts/ | test-regular-direct-github-scripts.yml |
| 2 | Regular | Direct run | ci/ | test-regular-direct-ci-scripts.yml |
| 3 | Regular | Relative action | .github/scripts/ | test-regular-relative-action-github-scripts.yml |
| 4 | Regular | Relative action | ci/ | test-regular-relative-action-ci-scripts.yml |
| 5 | Regular | Branch-ref action | .github/scripts/ | test-regular-branch-action-github-scripts.yml |
| 6 | Regular | Branch-ref action | ci/ | test-regular-branch-action-ci-scripts.yml |
| 7 | Reusable | Direct run | .github/scripts/ | caller-reusable-direct-github-scripts.yml |
| 8 | Reusable | Direct run | ci/ | caller-reusable-direct-ci-scripts.yml |
| 9 | Reusable | Relative action | .github/scripts/ | caller-reusable-relative-action-github-scripts.yml |
| 10 | Reusable | Relative action | ci/ | caller-reusable-relative-action-ci-scripts.yml |
| 11 | Reusable | Branch-ref action | .github/scripts/ | caller-reusable-branch-action-github-scripts.yml |
| 12 | Reusable | Branch-ref action | ci/ | caller-reusable-branch-action-ci-scripts.yml |

## Experiment Design

1. Set up all 12 workflow permutations on `main`, verify all green
2. Tag `pre-move` state, add dummy commits for history
3. Move all scripts to subdirectories (`*.sh` -> `moved/*.sh`), update all references, merge to `main`
4. Create PRs from the `pre-move` tag (before the scripts were moved) and observe which workflows pass/fail

## Results

### Experiment 1: Default Checkout (PR #3)

**PR from `pre-move` with no workflow changes.**

| # | Workflow | Result |
|---|----------|--------|
| 1-12 | All 12 permutations | **ALL PASS** |

**Why:** For `pull_request` triggers, GitHub creates a synthetic merge commit (`refs/pull/N/merge`) merging the PR into the base branch. Both the workflow YAML files and `actions/checkout@v4` use this merge commit. Since the PR only changed `README.md`, the merge commit has main's updated workflow files (referencing `moved/` paths) and main's moved scripts. Everything is self-consistent.

### Experiment 2: Non-Conflicting Workflow Modifications (PR #5)

**PR from `pre-move` that adds a comment to the top of every workflow and action file.**

| # | Workflow | Result |
|---|----------|--------|
| 1-12 | All 12 permutations | **ALL PASS** |

**Why:** Git's 3-way merge combines the PR's comment additions (top of file) with main's path changes (bottom of file). The merged result has both — main's updated paths win on the lines they changed, and the PR's additions are preserved.

### Experiment 3: Merge Conflicts (PRs #4, #6, #7)

Three conflict scenarios tested:

| PR | Conflict Location | Workflows Triggered |
|----|-------------------|---------------------|
| #4 | Workflow files (adjacent lines to path changes) | **NONE** |
| #6 | Non-workflow file only (NOTES.md) | **NONE** |
| #7 | Single workflow file (same line as path change) | **NONE** |

**Why:** When GitHub cannot create the merge commit (`refs/pull/N/merge`), `pull_request` workflows simply don't trigger. This is true regardless of whether the conflict is in a workflow file or a data file.

### Experiment 4: New Workflow Files with Old Paths (PR #8)

**PR from `pre-move` that adds 4 brand new workflow files referencing old script paths.**

| New Workflow | Invocation | Result |
|---|---|---|
| PR-Added: Direct + GitHub Scripts | `run: bash .github/scripts/direct-regular.sh` | **FAIL** |
| PR-Added: Direct + CI Scripts | `run: bash ci/direct-regular.sh` | **FAIL** |
| PR-Added: Relative Action + GitHub Scripts | `uses: ./.github/actions/...` | PASS |
| PR-Added: Relative Action + CI Scripts | `uses: ./.github/actions/...` | PASS |

**Why:** New workflow files don't exist on main, so there's nothing to merge — they're included in the merge commit verbatim from the PR branch. The `run:` commands reference old paths, but the checkout (merge commit) has scripts at new paths only. Relative actions pass because the `action.yml` in the merge commit has main's updated paths.

### Experiment 5: Git History Diagnostic (PR #9)

**PR from `pre-move` with a diagnostic workflow that dumps git history + unrelated functional changes to existing workflows.**

The diagnostic confirmed:
```
HEAD = Merge <PR-sha> into <main-sha>   (synthetic merge commit)
GITHUB_REF = refs/pull/9/merge
Parent 1: main (includes move commit)
Parent 2: PR branch (pre-move + changes)

Files at old paths: EMPTY (only 'moved/' subdirectory)
Files at new paths: ALL 6 SCRIPTS PRESENT
```

All 12 existing workflows passed, confirming that functional changes (concurrency blocks, env vars, permissions) added by the PR merge cleanly with main's path updates.

### Experiment 6: PyTorch Head-SHA Checkout Pattern (PR #10)

**PR from `pre-move` that modifies 4 existing workflows to use `ref: ${{ github.event_name == 'pull_request' && github.event.pull_request.head.sha || github.sha }}` in the checkout step — the pattern used by PyTorch.**

| Workflow | Checkout | Result |
|----------|----------|--------|
| Regular + Direct + GitHub Scripts | **head.sha** | **FAIL** |
| Regular + Direct + CI Scripts | **head.sha** | **FAIL** |
| Regular + Relative Action + GitHub Scripts | **head.sha** | PASS |
| Regular + Relative Action + CI Scripts | **head.sha** | PASS |
| All 8 unmodified workflows | merge commit (default) | PASS |

**Why:** The workflow YAML comes from the merge commit (with main's new `moved/` paths), but the checkout gets the PR's HEAD commit (with scripts at old paths). The `run:` command says `bash .github/scripts/moved/direct-regular.sh` but the file is at `.github/scripts/direct-regular.sh` in the checkout.

Diagnostic output confirmed:
```
GITHUB_SHA:         b70d9e60...  (merge commit)
Checked-out commit: d0282dc...   (PR HEAD -- different!)
OLD paths: ALL EXIST
NEW paths: ALL MISSING
```

**Relative actions are immune** because `uses: ./.github/actions/...` reads the `action.yml` from the checked-out directory (PR HEAD), not from the workflow YAML. The action.yml has old paths matching the old file tree.

### Bonus: The Move PR Itself (PR #2)

When testing the move PR (before it was merged to main), 4 of 12 workflows failed — all `@main` branch-ref action variants. The action.yml on `main` still had old paths while the PR checkout had scripts at new paths.

## Key Findings

### 1. Default behavior is safe

With default `actions/checkout@v4` (no `ref:` override), GitHub uses the merge commit for everything. Script moves on `main` are transparently picked up by PRs. Nothing breaks.

### 2. `ref: pull_request.head.sha` breaks direct `run:` commands

When the checkout explicitly uses the PR's HEAD commit:
- **Workflow YAML** still comes from the merge commit (GitHub platform behavior, not configurable)
- **Checked-out code** comes from the PR HEAD (before the move)
- **Direct `run:` commands** reference new paths (from merged workflow YAML) but scripts are at old paths (from checkout) -> **FAIL**

### 3. Relative action references are always safe

`uses: ./.github/actions/...` reads the `action.yml` from the checked-out directory, so it's always consistent with whatever code is checked out, regardless of the `ref:` setting.

### 4. Merge conflicts prevent all CI

Any merge conflict — in workflow files or non-workflow files — prevents GitHub from creating `refs/pull/N/merge`, and no `pull_request` workflows run at all.

### 5. `@main` branch-ref actions break during transitions

`uses: org/repo/.github/actions/action@main` always fetches the action.yml from `main`. If `main` has updated paths but the checkout has old paths (or vice versa during the move PR), these fail.

## PyTorch Case Study

### Why PyTorch uses head-SHA checkout

PyTorch's `checkout-pytorch` action (`/Users/zainr/pytorch/.github/actions/checkout-pytorch/action.yml`) deliberately checks out `pull_request.head.sha` instead of the merge commit. This was introduced by Michael Suo in PR [#71974](https://github.com/pytorch/pytorch/pull/71974) (Feb 2022) for:

- **Reproducibility**: Test the exact code the developer wrote and tested locally
- **Consistency**: Eliminate confusion where some jobs tested merge commit and others tested head commit
- **Predictability**: Re-running CI gives the same commit every time (merge commit changes as main advances)
- **Squash-merge alignment**: PyTorch uses squash-merge, which is better modeled by testing the head commit

This was NOT a security decision — it was an engineering choice for developer experience.

### Blast radius in PyTorch

The head-SHA checkout affects **19 of 26** PR-triggered workflows in pytorch/pytorch:

**Affected (19 workflows):**
- All build workflows (`_linux-build.yml`, `_mac-build.yml`, `_win-build.yml`) via `checkout-pytorch@main`
- All test workflows (`_linux-test.yml`, `_mac-test.yml`, `_win-test.yml`, `_rocm-test.yml`, `_xpu-test.yml`) via `checkout-pytorch@main`
- `_docs.yml`, `_bazel-build-test.yml`, `target_determination.yml` via `checkout-pytorch@main`
- All `lint.yml` jobs (via explicit `ref:` parameter or `checkout-pytorch@main`)
- `lint-autoformat.yml`, `nitpicker.yml`

**Not affected (7 workflows):**
- `check_mergeability_ghstack.yml` — uses default `actions/checkout`
- `auto_request_review.yml` — no checkout at all
- `runner-determinator-validator.yml` — uses default `actions/checkout`
- `_get-changed-files.yml` — API-only, no checkout
- `_runner-determinator.yml` — no checkout, fetched `@main`
- `job-filter.yml` — no checkout
- `llm_td_retrieval.yml` — uses default ref

### Single fix point

Modifying `checkout-pytorch/action.yml` lines 59 and 107 to remove the `ref:` override would fix 16 of 19 affected workflows. The remaining 3 need `ref:` removed from explicit parameters in `lint.yml` and `_link_check.yml`.

### Historical note

The "Which commit is used in CI?" section in PyTorch's `CONTRIBUTING.md` (lines 1352-1384) was originally written by Sam Estep in March 2021, then updated by Zain Rizvi in November 2022 to document the current B-vs-C behavior and the nuance about workflow files coming from the merge commit.

## Answers to the Original Key Questions

1. **Does the script location (inside vs outside `.github/`) matter?**
   No — both `.github/scripts/` and `ci/` behave identically in all scenarios.

2. **Does the invocation method matter?**
   Yes, critically:
   - **Direct `run:`** — Vulnerable to path mismatches (the `run:` line comes from the workflow YAML, which may have different paths than the checkout)
   - **Relative action (`uses: ./.github/actions/...`)** — Safe (action.yml is read from the checkout directory, always consistent with checked-out code)
   - **Branch-ref action (`uses: ...@main`)** — Breaks during transitions (always fetches from `@main` regardless of checkout)

3. **Does the workflow type (regular vs reusable) matter?**
   No — both regular and reusable workflows behave identically for all tested patterns.

4. **Which version of the workflow file does GitHub use?**
   Always the **merge commit** (`refs/pull/N/merge`) for `pull_request` triggers. This is a GitHub platform behavior that cannot be overridden. The `ref:` parameter on `actions/checkout` only controls what code is checked out, not which workflow YAML is executed.
