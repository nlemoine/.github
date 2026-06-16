# nlemoine/.github

Shared GitHub configuration and reusable workflows for my PHP repositories.

## `php-qa.yml` — reusable PHP quality-assurance pipeline

A single reusable workflow (`workflow_call`) that resolves the PHP version
matrix from `composer.json`, then runs coding standards, static analysis, a test
matrix, optional mutation testing, and an aggregate status check. It serves both
plain PHP libraries (php-cs-fixer / ecs) and WordPress projects (phpcs / WPCS).
It is DB-free: WordPress integration tests are out of scope.

The PHP version matrix is derived from the consuming repo's `composer.json`
`require.php` constraint (via
[`WyriHaximus/github-action-composer-php-versions-in-range`](https://github.com/WyriHaximus/github-action-composer-php-versions-in-range)),
so bumping supported PHP versions is a `composer.json` edit, not a CI edit.

### Usage

Add a thin caller to each repository at `.github/workflows/qa.yml`:

```yaml
name: QA

on:
  push:
    branches: [main]
  pull_request:

permissions:
  contents: read

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  qa:
    uses: nlemoine/.github/.github/workflows/php-qa.yml@<full-commit-sha>
    with:
      cs-tool: php-cs-fixer
    secrets: inherit
```

PHP versions are resolved from `composer.json` — no `php-versions` input needed.
Pin `@<full-commit-sha>` to a commit of this repository. Dependabot
(`github-actions` ecosystem) bumps the SHA in each consuming repo.

The caller must grant `permissions: contents: read`. The workflow's jobs request
`contents: read`, and a caller that grants less (e.g. `permissions: {}`) fails at
startup, because a called workflow cannot request more than the caller allows.

For a WordPress project:

```yaml
  qa:
    uses: nlemoine/.github/.github/workflows/php-qa.yml@<full-commit-sha>
    with:
      cs-tool: phpcs          # phpcs + WPCS ruleset (project owns phpcs.xml.dist)
      coverage: false
    secrets: inherit
```

### Inputs

| Input | Default | Description |
| --- | --- | --- |
| `php-versions` | `''` (resolve from composer.json) | Optional JSON array override for the test matrix. Element `[0]` is the floor: it also gets the `lowest` dependency leg and coverage. |
| `cs-tool` | `php-cs-fixer` | Coding-standards tool: `phpcs`, `php-cs-fixer`, or `ecs`. |
| `cs-php-version` | `''` (resolved floor) | PHP version used for coding standards. |
| `static-tool` | `auto` | `auto` (pick by config-file presence), `phpstan`, `psalm`, or `none`. |
| `static-php-version` | `''` (resolved floor) | PHP version used for static analysis. |
| `rector` | `auto` | `auto` (run when `rector.php` exists), `true`, or `false`. |
| `composer-dependency-versions` | `highest` | `locked`, `lowest`, or `highest` for non-test jobs. |
| `composer-options` | `--prefer-dist` | Extra options for composer install/update. |
| `coverage` | `true` | Collect coverage on the floor leg and upload to Codecov. |
| `type-coverage` | `false` | Run `pest --type-coverage` on the coverage leg (Pest only). |
| `type-coverage-min` | `100` | Minimum type-coverage percentage. |
| `run-infection` | `false` | Run Infection (advisory; not part of the gate). |
| `extra-cs-args` | `''` | Extra arguments for the coding-standards command. |
| `extra-stan-args` | `''` | Extra arguments for the static-analysis command. |

### Secrets

| Secret | Required | Description |
| --- | --- | --- |
| `CODECOV_TOKEN` | no | Codecov upload token. Pass via `secrets: inherit`. Without it the upload is skipped (it never fails the build). |

### Conventions

- **PHP versions** come from `composer.json` by default (stable releases in the
  `require.php` range). Pass `php-versions` only to override. The floor is the
  lowest resolved version and is where coding standards, static analysis, the
  `lowest` dependency leg, and coverage run.
- **Tests** run `composer test` on every matrix leg, except the floor +
  `highest` leg, which runs the detected runner (`pest` if `vendor/bin/pest`
  exists, else `phpunit`) with clover coverage.
- **Static analysis** with `auto` runs PHPStan if a `phpstan.neon*` file exists,
  Psalm if a `psalm.xml*` file exists, and is skipped otherwise. The project
  owns its own tool config (including `szepeviktor/phpstan-wordpress` + stubs for
  WordPress).
- **`QA OK`** is the single aggregate job. Point branch protection at it; the
  matrix and job roster can then change here without touching branch protection.

### Required status check

Set the required status check in branch protection to **`QA OK`**. A
`failure` or `cancelled` in coding standards, static analysis, or tests fails
it; intentionally skipped jobs (e.g. `static-tool: none`) do not.

## `release-please.yml` — reusable release workflow

Runs [release-please](https://github.com/googleapis/release-please) on push to
the default branch to maintain the release PR and cut releases. Add a caller at
`.github/workflows/release.yml`:

```yaml
name: Release

on:
  push:
    branches: [main]

permissions:
  contents: write
  pull-requests: write

concurrency:
  group: release-please-${{ github.ref }}
  cancel-in-progress: false

jobs:
  release:
    uses: nlemoine/.github/.github/workflows/release-please.yml@<full-commit-sha>
```

The repo keeps owning `release-please-config.json` and
`.release-please-manifest.json` (override the paths with the `config-file` /
`manifest-file` inputs if needed). The caller must grant `contents: write` and
`pull-requests: write`. By default it uses `GITHUB_TOKEN`; pass a PAT via
`secrets: { token: ${{ secrets.RELEASE_TOKEN }} }` if you need release events to
trigger downstream workflows.
