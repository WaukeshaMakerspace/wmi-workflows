# wmi-workflows

Reusable GitHub Actions workflows for Waukesha Makerspace services.

A service scaffolded from
[wmi-app-template](https://github.com/WaukeshaMakerspace/wmi-app-template)
calls these instead of carrying its own copy, so a CI/CD fix is one pull
request rather than one per service. See **ADR-0006** in
[waukesha-makers-architecture](https://github.com/WaukeshaMakerspace/waukesha-makers-architecture)
for the reasoning.

This repo is public on purpose. Workflow definitions contain no secrets, and a
private host would require configuring the organisation's Actions access policy
before any repository could call them.

## Workflows

| Workflow | Purpose |
|---|---|
| `service-ci.yml` | Typecheck, lint, test, build, with a MySQL service and optional database setup |
| `service-deploy.yml` | test, migrate, deploy-api, deploy-ui, in that order |

### `service-ci.yml`

```yaml
# .github/workflows/ci.yml in a service
name: CI
on:
  push:
    branches: [main]
  pull_request:

jobs:
  ci:
    uses: WaukeshaMakerspace/wmi-workflows/.github/workflows/service-ci.yml@v1
    with:
      db-name: timewright
```

| Input | Required | Default | Notes |
|---|---|---|---|
| `db-name` | yes | | Letters, digits and underscore only |
| `node-version` | no | `22` | |
| `run-database-steps` | no | `true` | Set `false` only in wmi-app-template itself |

### `service-deploy.yml`

```yaml
# .github/workflows/deploy.yml in a service
name: Deploy
on:
  push:
    branches: [main]
  workflow_dispatch:

concurrency:
  group: deploy-timewright
  cancel-in-progress: false

jobs:
  deploy:
    uses: WaukeshaMakerspace/wmi-workflows/.github/workflows/service-deploy.yml@v1
    secrets: inherit
    with:
      app-name: timewright
      db-name: timewright
```

| Input | Required | Default | Notes |
|---|---|---|---|
| `app-name` | yes | | kebab-case. npm scope (`@<app-name>/api`) and Lambda function name (`<app-name>-api`) |
| `db-name` | yes | | Letters, digits and underscore only |
| `node-version` | no | `22` | |
| `aws-region` | no | `us-east-2` | |

The calling repository must provide, through `secrets: inherit`:

| Secret | Purpose |
|---|---|
| `AWS_DEPLOY_ROLE_ARN` | Assumed via GitHub OIDC. No long-lived access keys |
| `MIGRATOR_DATABASE_URL` | The scoped `<db>_migrator` credential, server only |

and must have a **`production` environment** on the repository, which gates
`migrate`, `deploy-api`, and `deploy-ui`.

`concurrency` belongs in the calling workflow, not here, because it is
workflow-level and must cover the whole deploy rather than a single job.

## Versioning

Services pin `@v1`. Fixes ship by moving the `v1` tag, which is the entire
point: a fix reaches every service without touching any of them. Breaking
changes cut `v2` and services migrate deliberately.

Immutable `vX.Y.Z` tags are also published, for pinning during an incident.

## How changes are validated

Be honest about what each layer catches, because the risk here is real: a bad
shared workflow breaks every service's pipeline at once.

1. **`lint.yml` runs actionlint on every push and PR.** Catches YAML and
   expression errors, which is the most common way a shared workflow breaks.
   It does not execute anything.
2. **wmi-app-template is the CI canary.** It is a real npm workspace that calls
   `service-ci.yml` on every push, so a change that breaks CI in practice turns
   the template red.
3. **The first deploying service is the deploy canary.** `service-deploy.yml`
   cannot be meaningfully self-tested: it needs AWS credentials, a provisioned
   stack, and a real database. Nothing here proves it works end to end.

**Move the `v1` tag only after 1 and 2 are green on the new commit.**

A self-test that actually executed `service-ci.yml` would need a fake service
fixture in this repo, which would drift from the real template and give
false confidence. The template is a better canary precisely because it is real.

## Adding a step

Do not fork a workflow to add a step. A reusable workflow cannot have steps
injected by its caller, so a service that needs extra behaviour either:

- gets an explicit input here, if the need is general, or
- adds its own separate job in its own workflow file, if it is genuinely
  service-specific.

If two services want the same extra step, that is the signal to add an input.
