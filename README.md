# ci-templates

Reusable GitHub Actions workflows for common CI tasks. Call them from any repo
with `uses: gortcodes/ci-templates/.github/workflows/<name>.yml@main`.

## Available workflows

| Workflow            | Purpose                                                  | Status  |
| ------------------- | -------------------------------------------------------- | ------- |
| `python-ci.yml`     | Install, lint (ruff), and test a Python project          | Ready   |
| `docker-build.yml`  | Build a container image and upload it as an artifact     | Ready   |
| `docker-health.yml` | Start a container from an artifact and probe its health  | Ready   |
| `docker-push.yml`   | Load a container image artifact and push it to a registry | Ready  |
| `lint.yml`          | Generic linting across languages                         | Planned |

## `python-ci.yml`

### Usage

```yaml
# .github/workflows/ci.yml in the calling repo
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  python:
    uses: gortcodes/ci-templates/.github/workflows/python-ci.yml@main
    with:
      python-version: "3.12"
      test-command: "pytest -q"
      lint: true
```

### Inputs

| Name                | Type    | Default      | Description                                          |
| ------------------- | ------- | ------------ | ---------------------------------------------------- |
| `python-version`    | string  | `"3.12"`     | Python version passed to `actions/setup-python`.     |
| `working-directory` | string  | `"."`        | Directory all steps run from.                        |
| `test-command`      | string  | `"pytest -q"`| Shell command used to run tests.                     |
| `lint`              | boolean | `true`       | When true, installs `ruff` and runs `ruff check .`.  |

### Behavior

- **Dependencies:** if `pyproject.toml` exists, runs `pip install .`. Otherwise
  falls back to `pip install -r requirements.txt`. If neither is present, the
  install step is skipped.
- **Lint:** `ruff` is installed into the job at runtime so callers don't need
  to pin it in their own dev dependencies.
- **Caching:** pip wheels are cached via `actions/setup-python` using the
  project's `pyproject.toml` / `requirements.txt` as the cache key.
- **Reporting:** relies on native GitHub Actions step status — failures show up
  in the Checks tab and PR UI.

## Docker workflows

The docker flow is split into three composable workflows so projects can gate a
registry push on their own integration tests. The common shape is:

```
docker-build  →  (health + project integration tests)  →  docker-push
     │                         │                              │
     │                         │                              │
     └── uploads tarball ──────┴── download+load ─────────────┘
```

Image handoff between jobs is via a GitHub Actions artifact (the image is
exported as a tarball by `docker-build.yml` and re-loaded by downstream jobs).
Nothing hits the registry until `docker-push.yml` runs.

**Scope note:** single-arch (`linux/amd64`) only. Multi-platform images don't
round-trip through `docker save` / `docker load` cleanly, so they're out of
scope for this flow.

### End-to-end caller example

```yaml
# .github/workflows/docker.yml in the calling repo
name: Docker

on:
  push:
    branches: [main]
  pull_request:

jobs:
  build:
    uses: gortcodes/ci-templates/.github/workflows/docker-build.yml@main

  health:
    needs: build
    uses: gortcodes/ci-templates/.github/workflows/docker-health.yml@main
    with:
      port: "8080"
      path: "/healthz"

  integration:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/download-artifact@v4
        with:
          name: docker-image
          path: /tmp
      - run: docker load -i /tmp/image.tar
      - run: ./scripts/integration-tests.sh  # project-owned

  push:
    needs: [health, integration]
    if: github.ref == 'refs/heads/main'
    uses: gortcodes/ci-templates/.github/workflows/docker-push.yml@main
    permissions:
      contents: read
      packages: write
```

### `docker-build.yml`

Builds the image, exports it as a tarball, and uploads it as an artifact. No
registry access.

| Input           | Type   | Default           | Description                                   |
| --------------- | ------ | ----------------- | --------------------------------------------- |
| `context`       | string | `"."`             | Build context path.                           |
| `dockerfile`    | string | `"./Dockerfile"`  | Path to the Dockerfile.                       |
| `artifact-name` | string | `"docker-image"`  | Name of the uploaded image tarball artifact.  |

Uses `docker/build-push-action` with `type=gha` layer cache. The artifact is
kept for 1 day (enough to survive a normal CI run; short enough to not clutter
storage).

### `docker-health.yml`

Downloads the image artifact, runs the container, and probes an HTTP endpoint.
Fails the job if the endpoint doesn't return the expected status within the
timeout.

| Input              | Type    | Default          | Description                                       |
| ------------------ | ------- | ---------------- | ------------------------------------------------- |
| `artifact-name`    | string  | `"docker-image"` | Artifact produced by `docker-build.yml`.          |
| `port`             | string  | `"8080"`         | Container port to publish and probe.              |
| `path`             | string  | `"/"`            | URL path to probe.                                |
| `expected-status`  | string  | `"200"`          | Expected HTTP status code.                        |
| `timeout-seconds`  | number  | `30`             | Total time to wait for a healthy response.        |
| `run-args`         | string  | `""`             | Extra args passed to `docker run` (env, cmd, …).  |

On failure, the container's logs are printed to the job output before teardown.

### `docker-push.yml`

Downloads the image artifact, logs in to the registry, re-tags the image using
`docker/metadata-action` defaults (branch, PR ref, git SHA, `latest` on default
branch, semver on tags), and pushes each tag.

| Input           | Type    | Default                              | Description                                           |
| --------------- | ------- | ------------------------------------ | ----------------------------------------------------- |
| `artifact-name` | string  | `"docker-image"`                     | Artifact produced by `docker-build.yml`.              |
| `image-name`    | string  | `ghcr.io/${{ github.repository }}`   | Fully-qualified destination image name (without tag). |
| `registry`      | string  | `"ghcr.io"`                          | Registry hostname used for login.                     |
| `tags`          | string  | `""`                                 | Newline/comma-separated tag override.                 |

Secrets:

| Name                | Required | Default          | Description                       |
| ------------------- | -------- | ---------------- | --------------------------------- |
| `registry-username` | no       | `github.actor`   | Registry username.                |
| `registry-password` | no       | `GITHUB_TOKEN`   | Registry password or token.       |

**GHCR note:** GHCR requires lowercase image names and the calling job must
grant `packages: write` (shown in the end-to-end example above). For other
registries, pass `registry`, `image-name`, and credentials via `secrets:`.

## Versioning

Pin to `@main` for the latest, or to a specific commit SHA / tag for stability.
