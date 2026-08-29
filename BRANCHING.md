# Branching Workflow

This repository uses `main` as the stable branch and `develop` as the integration branch for the POS system.

## Branch Roles

| Branch | Purpose | Merge source |
|--------|---------|--------------|
| `main` | Stable, production-ready code | `develop` through a pull request |
| `develop` | Integration branch containing approved specifications and completed implementation work | `feature/*` through pull requests |
| `feature/*` | Short-lived branch for one implementation area or change | Created from the latest `develop` |

## Branch Flow

```text
main
  └── develop
        ├── feature/pos-foundation
        ├── feature/pos-checkout
        ├── feature/pos-inventory
        ├── feature/pos-refunds
        └── feature/pos-reports
```

The normal promotion path is:

```text
feature/* → develop → main
```

Do not merge implementation branches directly into `main`.

## Initial Branch Rename

If the current integration branch is named `001-retail-pos-system`, rename it to `develop` locally and publish the new remote branch:

```bash
git branch -m 001-retail-pos-system develop
git push -u origin develop
```

After confirming that `origin/develop` exists and points to the expected commit, remove the old remote branch if it is no longer needed:

```bash
git push origin --delete 001-retail-pos-system
```

The rename does not change commit history.

## Starting Feature Work

Always start a feature branch from the latest `develop`:

```bash
git switch develop
git pull --ff-only origin develop
git switch -c feature/pos-foundation
git push -u origin feature/pos-foundation
```

Use one branch for one coherent implementation area. Recommended names for this project include:

- `feature/pos-foundation`
- `feature/pos-checkout`
- `feature/pos-payments`
- `feature/pos-refunds`
- `feature/pos-catalog`
- `feature/pos-inventory`
- `feature/pos-discounts`
- `feature/pos-customers`
- `feature/pos-reports`
- `feature/pos-admin`

## Updating a Feature Branch

Before opening or updating a pull request, bring the latest integration changes into the feature branch:

```bash
git switch feature/pos-checkout
git fetch origin
git merge origin/develop
git push
```

Use a rebase instead of merge only when the team agrees on that history policy and the branch has not already been shared widely.

## Pull Requests

### Feature branch to `develop`

Open a pull request using this direction:

```text
feature/pos-checkout → develop
```

The pull request should include:

- The user story or task IDs implemented
- Tests added before implementation where required by the POS constitution
- Offline behavior and sync implications
- Money, inventory, audit, and authorization impact
- Validation results and any known gaps

### `develop` to `main`

Open a release pull request only after the intended implementation is integrated and validated:

```text
develop → main
```

Before merging, verify the blocking checks required by the project constitution:

- Unit, integration, and contract tests
- Money-invariant tests
- Offline and synchronization tests
- Hardware-fake checkout tests
- Latency budget checks
- Dependency and secret scanning
- Database migration up/down verification
- Quickstart scenario validation

## Recommended Branch Protection

### `main`

- Require pull requests
- Require at least one review
- Require all CI checks to pass
- Disable direct pushes
- Disable force pushes

### `develop`

- Require pull requests
- Require CI checks to pass
- Require review for shared integration changes
- Disable force pushes

## Daily Commands

Check the current branch and working tree:

```bash
git branch --show-current
git status --short
```

Create a new feature branch:

```bash
git switch develop
git pull --ff-only origin develop
git switch -c feature/<short-name>
```

Publish a feature branch:

```bash
git push -u origin feature/<short-name>
```

Return to integration work:

```bash
git switch develop
git pull --ff-only origin develop
```

## Rules of Thumb

1. Keep `main` releasable.
2. Keep `develop` integratable.
3. Keep feature branches focused and short-lived.
4. Never create a feature branch from stale `main`; branch from the latest `develop`.
5. Do not use force-push on `main` or `develop`.
6. Keep specification, planning, and implementation changes traceable through commits and pull requests.
