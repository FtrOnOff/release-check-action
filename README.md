# FtrIO Release Check Action

GitHub Action that validates a target `appsettings.json` contains every toggle key required by a release. Pair with [export-manifest-action](https://github.com/FtrOnOff/export-manifest-action) to gate deployments on missing config.

Part of the [FtrIO ecosystem](https://github.com/FtrOnOff) — free, open source feature toggles for .NET.

---

## What it does

Downloads a toggle manifest (produced by [export-manifest-action](https://github.com/FtrOnOff/export-manifest-action)) and checks every key in it against a target `appsettings.json`. If any keys are missing, the action fails and the deployment is blocked before it reaches production.

A missing toggle key in production will throw a `ToggleDoesNotExistException` at runtime. This action catches that before it happens.

---

## Quick start

```yaml
- uses: FtrOnOff/release-check-action@v1
  with:
    artifact-name: toggle-manifest
    config-url: ${{ secrets.PRODUCTION_CONFIG_URL }}
```

Downloads the `toggle-manifest` artifact from the build pipeline, fetches the production config, and fails if any keys are missing.

---

## Inputs

| Input | Default | Description |
|---|---|---|
| `manifest` | `toggles.manifest.json` | Path to the manifest JSON on disk. Used when manifest is already available locally. |
| `artifact-name` | *(none)* | Name of the build artifact to download. If set, downloads it automatically. |
| `config` | *(none)* | Path to the target `appsettings.json`. Mutually exclusive with `config-url`. |
| `config-url` | *(none)* | URL to fetch the target `appsettings.json` from. Mutually exclusive with `config`. |
| `config-auth-header` | *(none)* | Authorization header for `config-url` (e.g. `Bearer my-token`). |
| `env-name` | `Production` | Display name for the target environment shown in the report and annotations. |
| `fail-on-missing` | `true` | Fail the workflow if any keys are missing. |
| `warn-only` | `false` | Emit warnings but always exit 0. Overrides `fail-on-missing`. |
| `markdown` | `release-check-report.md` | Path to write a markdown report. Uploaded as an artifact automatically. |
| `version` | *(latest)* | Pin a specific version of FtrIO.onetwo. |

## Outputs

| Output | Description |
|---|---|
| `missing-count` | Number of toggle keys missing from the target config |
| `present-count` | Number of toggle keys present in the target config |
| `passed` | `true` if all keys are present, `false` otherwise |
| `report-path` | Path to the markdown report |

---

## Examples

### With artifact from export-manifest-action and a URL config

```yaml
- uses: FtrOnOff/release-check-action@v1
  with:
    artifact-name: toggle-manifest
    config-url: ${{ secrets.PRODUCTION_CONFIG_URL }}
    env-name: Production
    fail-on-missing: true
```

---

### With a local config file

Useful when the manifest and config are both available in the same pipeline:

```yaml
- uses: FtrOnOff/release-check-action@v1
  with:
    artifact-name: toggle-manifest
    config: ./appsettings.json
    env-name: Production
```

---

### Warn only — never block the deployment

```yaml
- uses: FtrOnOff/release-check-action@v1
  with:
    artifact-name: toggle-manifest
    config-url: ${{ secrets.PRODUCTION_CONFIG_URL }}
    warn-only: true
```

Emits GitHub warning annotations for missing keys but always exits 0. Useful when introducing the check to an existing pipeline without immediately blocking deployments.

---

### With authenticated config endpoint

```yaml
- uses: FtrOnOff/release-check-action@v1
  with:
    artifact-name: toggle-manifest
    config-url: ${{ secrets.PRODUCTION_CONFIG_URL }}
    config-auth-header: 'Bearer ${{ secrets.CONFIG_API_TOKEN }}'
    env-name: Production
```

---

### Use outputs in later steps

```yaml
- uses: FtrOnOff/release-check-action@v1
  id: release-check
  with:
    artifact-name: toggle-manifest
    config-url: ${{ secrets.PRODUCTION_CONFIG_URL }}

- name: Print summary
  run: |
    echo "Present: ${{ steps.release-check.outputs.present-count }}"
    echo "Missing: ${{ steps.release-check.outputs.missing-count }}"
    echo "Passed:  ${{ steps.release-check.outputs.passed }}"
```

---

### Full deployment pipeline

```yaml
name: Deploy to Production

on:
  release:
    types: [published]

jobs:
  release-check:
    runs-on: ubuntu-latest
    steps:
      - name: FtrIO release check
        uses: FtrOnOff/release-check-action@v1
        with:
          artifact-name: toggle-manifest
          config-url: ${{ secrets.PRODUCTION_CONFIG_URL }}
          env-name: Production
          fail-on-missing: true
          markdown: release-check-report.md

  deploy:
    needs: release-check
    runs-on: ubuntu-latest
    steps:
      - name: Deploy
        run: echo "deploying..."
```

The `needs: release-check` ensures the deploy job only runs if the toggle check passes. Missing config in production blocks the deploy automatically.

---

## Pairing with export-manifest-action

This action is the second half of FtrIO's deployment safety pipeline. The first half is [export-manifest-action](https://github.com/FtrOnOff/export-manifest-action), which scans the source code and produces the manifest this action consumes.

```
App CI pipeline                      Deployment pipeline
────────────────────────────────     ────────────────────────────────
export-manifest-action           →   release-check-action
Scans source for [Toggle] keys        Downloads manifest artifact
Uploads toggles.manifest.json         Fetches target appsettings.json
                                      Checks every key is present
                                      Blocks deploy if any are missing
```

**App CI (`build.yml`):**

```yaml
jobs:
  build:
    steps:
      - uses: actions/checkout@v4
      - run: dotnet build
      - run: dotnet test
      - uses: FtrOnOff/export-manifest-action@v1
        with:
          source: ./src
          artifact-name: toggle-manifest
```

**Deployment pipeline (`deploy.yml`):**

```yaml
jobs:
  release-check:
    steps:
      - uses: FtrOnOff/release-check-action@v1
        with:
          artifact-name: toggle-manifest
          config-url: ${{ secrets.PRODUCTION_CONFIG_URL }}
          fail-on-missing: true

  deploy:
    needs: release-check
    steps:
      - name: Deploy
        run: echo "deploying..."
```

---

## What missing keys look like in a deployment

When toggle keys are missing the action emits inline GitHub annotations and fails the job:

```
Warning: FtrIO [Production]: PaymentV2   MISSING   Services/PaymentService.cs:88
Warning: FtrIO [Production]: BetaSearch  MISSING   Controllers/SearchController.cs:23
Error:   FtrIO release-check: 2 toggle key(s) missing from Production config.
         Add the missing keys before deploying.
```

The deploy job is blocked and the release report is uploaded as an artifact for the team to review.

---

## The FtrIO safety net

Combined with the Roslyn analyzer and ftrio-action, release-check-action closes the final gap in the FtrIO safety pipeline:

| Stage | What catches it |
|---|---|
| Write `[Toggle]` without config entry | Roslyn analyzer — compile time |
| Push PR with missing key | [ftrio-action](https://github.com/FtrOnOff/ftrio-action) — CI time |
| Release with key missing from target env | **release-check-action** — deploy time |

---

## The FtrIO ecosystem

| Tool | Purpose |
|---|---|
| [FtrIO](https://github.com/FtrOnOff/FtrIO) | Core library. `[Toggle]` attribute woven into IL at compile time. |
| [FtrIO.Toaster](https://github.com/FtrOnOff/FtrIO.Toaster) | Self-hosted Docker UI for managing toggles live. |
| [FtrIO.onetwo](https://github.com/FtrOnOff/FtrIO.onetwo) | CLI audit tool. This action wraps it. |
| [ftrio-action](https://github.com/FtrOnOff/ftrio-action) | General toggle audit action for CI. |
| [export-manifest-action](https://github.com/FtrOnOff/export-manifest-action) | Export a manifest of required toggle keys from source code. |

Full docs: [ftronoff.github.io/FtrIO](https://ftronoff.github.io/FtrIO)
