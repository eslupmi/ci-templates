# ci-templates

Reusable GitHub Actions workflows for IMPulse repositories.

## Usage

```yaml
jobs:
  lint:
    uses: eslupmi/ci-templates/.github/workflows/_lint.yml@develop
```

Templates run in the context of the caller: `actions/checkout`, `github.repository` and `GITHUB_TOKEN` refer to the calling repository. Secrets that are not declared in `workflow_call.secrets` are available only with `secrets: inherit`.

## Templates

| Template | Purpose | Default `runs-on` |
|----------|---------|--------|
| [`_lint.yml`](#_lintyml) | ruff and mypy | `blacksmith-4vcpu-ubuntu-2204` |
| [`_unit_tests.yml`](#_unit_testsyml) | pytest in the caller repository | `blacksmith-4vcpu-ubuntu-2204` |
| [`_build-and-push-image.yml`](#_build-and-push-imageyml) | Build and push Docker image to GHCR | `ubuntu-24.04` |
| [`_autotests.yml`](#_autotestsyml) | Fan out autotests by messenger | — (uses `_external_tests_suite.yml` default) |
| [`_test_suites.yml`](#_test_suitesyml) | Fan out response, system, provider, browser | — (uses `_external_tests_suite.yml` default) |
| [`_external_tests_suite.yml`](#_external_tests_suiteyml) | Run one suite from `eslupmi/tests` | `blacksmith-4vcpu-ubuntu-2204` |
| [`_migration.yml`](#_migrationyml) | Check incidents upgrade/downgrade migration on a server | `blacksmith-4vcpu-ubuntu-2204` |
| [`_deploy.yml`](#_deployyml) | Deploy via docker compose over SSH | `ubuntu-24.04` |
| [`_notify.yml`](#_notifyyml) | Send release notes to Telegram | `ubuntu-24.04` |

### `_lint.yml`

Runs `ruff check .` and `mypy .` on the runner via `setup-uv`. ruff gets `--target-version py<python-version>`. mypy installs `requirements.txt` from the caller repository plus `types-PyYAML`, `types-icalendar`, `types-requests`. Other settings (rules, extra excludes) come from the caller repository config (`pyproject.toml`, `mypy.ini`). If `mypy-plugins` is set, mypy runs with a generated config containing only these plugins and the caller repository config is ignored.

| Input | Type | Default | Description |
|-------|------|---------|-------------|
| `python-version` | string | `3.12` | Python version for uv and ruff `--target-version` |
| `exclude` | string | `[]` | JSON list of paths relative to the repository root. Passed to ruff as `--extend-exclude` and to mypy as `--exclude ^<path>(/\|$)` |
| `ruff-version` | string | `0.17.0` | ruff version |
| `mypy-version` | string | `2.4.0` | mypy version |
| `mypy-plugins` | string | `[]` | JSON list of mypy plugins, e.g. `'["pydantic.mypy"]'` |

```yaml
    with:
      exclude: '["tests", "docs"]'
```

Env:

| Name | Value |
|------|-------|
| `FORCE_COLOR` | `1` |
| `TERM` | `xterm-256color` |
| `PYTHON_VERSION` | `${{ inputs.python-version }}` |
| `EXCLUDE` | `${{ inputs.exclude }}` |

### `_unit_tests.yml`

Copies `examples/impulse.none.yml` to `impulse.yml`, installs `requirements.txt` and `tests.requirements.txt` on the runner via `setup-uv`, runs `pytest tests/`. Timeout: 10 minutes.

| Input | Type | Default | Description |
|-------|------|---------|-------------|
| `python-version` | string | `3.12` | Python version |

Step env:

| Name | Value |
|------|-------|
| `DATA_PATH` | `${{ runner.temp }}/data` |

### `_build-and-push-image.yml`

Builds the image from the caller repository root and pushes it with tags and labels from `docker/metadata-action`.

Inputs: `runs-on` only. Required caller permissions: `contents: read`, `packages: write`.

| Env | Value |
|-----|-------|
| `REGISTRY` | `ghcr.io` |
| `IMAGE_NAME` | `${{ github.repository }}` |

Secrets: `GITHUB_TOKEN`.

### `_autotests.yml`

Calls `_external_tests_suite.yml` for suite `inputs.autotests` and each messenger (`slack`, `mattermost`, `telegram`).

| Input | Type | Default | Description |
|-------|------|---------|-------------|
| `impulse-ref` | string | required | Image tag of `ghcr.io/eslupmi/impulse` |
| `tests-ref` | string | `develop` | Ref of `eslupmi/tests` |
| `autotests` | string | required | `autotests` or `autotests_mocked` |

Secrets: requires `secrets: inherit` (see `_external_tests_suite.yml`).

### `_test_suites.yml`

Calls `_external_tests_suite.yml` for suites `response`, `system`, `provider`, `browser`.

| Input | Type | Default | Description |
|-------|------|---------|-------------|
| `impulse-ref` | string | required | Image tag of `ghcr.io/eslupmi/impulse` |
| `tests-ref` | string | `develop` | Ref of `eslupmi/tests` |

Secrets: requires `secrets: inherit` (see `_external_tests_suite.yml`).

### `_external_tests_suite.yml`

Checks out `eslupmi/tests`, installs its `requirements.txt` with uv (Python 3.12), installs Chromium for Playwright, adds `host.docker.internal` to `/etc/hosts` and runs `python -m autotest.runner`. For `autotests` + `telegram` decodes `TELEGRAM_SESSION_BASE64` into `session.session`.

| Input | Type | Default | Description |
|-------|------|---------|-------------|
| `impulse-ref` | string | required | Image tag of `ghcr.io/eslupmi/impulse` |
| `tests-ref` | string | `develop` | Ref of `eslupmi/tests` |
| `suite` | string | required | Suite name passed to `--suite` |
| `messenger` | string | `""` | Messenger passed to `--messenger` |

Secrets (via `secrets: inherit`):

| Secret | Used as |
|--------|---------|
| `ESLUPMI_TESTS_REPO_TOKEN_READ` | Token to check out `eslupmi/tests` |
| `AUTOTESTS_SLACK_VERIFICATION_TOKEN` | `SLACK_VERIFICATION_TOKEN` |
| `AUTOTESTS_SLACK_BOT_USER_OAUTH_TOKEN` | `SLACK_BOT_USER_OAUTH_TOKEN` |
| `AUTOTESTS_MATTERMOST_ACCESS_TOKEN` | `MATTERMOST_ACCESS_TOKEN` |
| `AUTOTESTS_TELEGRAM_BOT_TOKEN` | `TELEGRAM_BOT_TOKEN` |
| `AUTOTESTS_API_ID` | `API_ID` |
| `AUTOTESTS_API_HASH` | `API_HASH` |
| `AUTOTESTS_TELEGRAM_SESSION_BASE64` | `TELEGRAM_SESSION_BASE64` |

Env:

| Name | Value |
|------|-------|
| `IMPULSE_IMAGE` | `ghcr.io/eslupmi/impulse:${{ inputs.impulse-ref }}` |
| `PYTHONUNBUFFERED` | `1` |
| `MATTERMOST_URL` | `https://mattermost.impulse.bot` |
| `SUITE` | `${{ inputs.suite }}` |
| `MESSENGER` | `${{ inputs.messenger }}` |

### `_migration.yml`

On the target server: backs up `docker-compose.yml` and `<messenger>/data/incidents`, switches the image to `ghcr.io/eslupmi/impulse:<branch>`, starts it (upgrade), downgrades to the latest GitHub release of the caller repository via `python -m main --downgrade`, diffs original vs downgrade (fails on difference), restores the original state and prints original vs upgrade diff. Snapshots are stored in `/tmp/impulse-migration`.

| Input | Type | Default | Description |
|-------|------|---------|-------------|
| `branch` | string | required | Image tag to migrate to |
| `deploy-path` | string | required | Directory with `docker-compose.yml` on the server |

| Secret | Description |
|--------|-------------|
| `HOST` | SSH host |
| `PORT` | SSH port |
| `USER` | SSH user |
| `SSH_KEY` | SSH private key |

Env:

| Name | Value |
|------|-------|
| `MESSENGERS` | `slack mattermost telegram` |
| `YELLOW` | `\033[33m` |
| `RESET` | `\033[0m` |
| `BRANCH` | `${{ inputs.branch }}` |
| `DEPLOY_PATH` | `${{ inputs.deploy-path }}` |
| `LATEST_RELEASE` | Latest release tag of `github.repository` |

### `_deploy.yml`

Over SSH in `deploy-path`: `docker compose pull`, runs `python -m main --check` for `impulse-slack`, `impulse-mattermost`, `impulse-telegram`, then `docker compose up -d`.

| Input | Type | Default | Description |
|-------|------|---------|-------------|
| `deploy-path` | string | required | Directory with `docker-compose.yml` on the server |

| Secret | Description |
|--------|-------------|
| `HOST` | SSH host |
| `PORT` | SSH port |
| `USER` | SSH user |
| `SSH_KEY` | SSH private key |

### `_notify.yml`

Takes the `## <tag>` section from `CHANGELOG.md` of the caller repository, converts it to Telegram MarkdownV2 (truncated to 4096 chars) and sends it with a link to the GitHub release. Intended for tag pushes.

| Secret | Description |
|--------|-------------|
| `RELEASE_TELEGRAM_BOT_TOKEN` | Telegram bot token |
| `RELEASE_TELEGRAM_CHAT_ID` | Target chat ID |

Env:

| Name | Value |
|------|-------|
| `TAG` | `${{ github.ref_name }}` |
| `REPO` | `${{ github.repository }}` |
