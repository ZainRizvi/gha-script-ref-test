# GHA Script Reference Test

Testing GitHub Actions behavior when scripts referenced by workflows move locations on main.

## Test Matrix

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

<!-- verify all workflows pass -->

<!-- dummy commit 1 -->

<!-- dummy commit 2 -->

<!-- dummy commit 3 -->

# test from pre-move state with modified workflows
