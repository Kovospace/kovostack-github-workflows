# kovospace-gitops

Central CI/CD for kovospace projects. Holds the **reusable** build → deploy → cleanup
pipeline so every project ships the same way and secrets are configured **once**.

## How it fits together

```
kovospace (GitHub org)
├── secrets (org-level, set ONCE)      REGISTRY_USER/PASSWORD, VPS_HOST/USER/PASSWORD/PORT
├── gitops/  ← this repo
│   └── .github/workflows/build-deploy.yml   (on: workflow_call — the real pipeline)
└── kovospace-frontend/, project-b/, ...
    └── .github/workflows/deploy.yml          (thin caller: uses: + secrets: inherit)
```

- **Secrets live at the org level.** Each project reads them via `secrets: inherit`.
  New project = zero secret setup.
- **Pipeline logic lives here.** Fix a bug / bump an action once; every project
  gets it on its next run (they all point at `@main`).
- **Per-project differences** (`image_name`, `compose_dir`, the build/deploy/cleanup
  toggles) are passed as `with:` inputs by each caller. `registry` defaults to the
  shared self-hosted registry here — callers only set it to override.

## One-time setup

1. Create the GitHub org (e.g. `kovospace`) and move projects into it.
2. Org **Settings → Secrets and variables → Actions → New organization secret** — add:
   `REGISTRY_USER`, `REGISTRY_PASSWORD`, `VPS_HOST`, `VPS_USER`, `VPS_PASSWORD`, `VPS_PORT`.
   Grant them to *All repositories* (or select).
3. Push this repo as `kovospace/gitops`.
4. Org **Settings → Actions → General → "Access"**: allow this repo's workflows to be
   used by other repos in the org (Actions must be permitted to call reusable workflows).

## Adding a project (the template)

Drop this into `.github/workflows/deploy.yml`, adjust the three `with:` values:

```yaml
name: Build and push Docker image
on:
  workflow_dispatch:
    inputs:
      build:   { description: 'Build and push the image', type: boolean, default: true }
      deploy:  { description: 'Deploy to the VPS',          type: boolean, default: true }
      cleanup: { description: 'Clean up registry + host',   type: boolean, default: true }
jobs:
  pipeline:
    uses: kovospace/gitops/.github/workflows/build-deploy.yml@main
    with:
      image_name: <project-image-name>
      compose_dir: <path/on/vps/to/compose/dir>
      # registry: defaults to the shared self-hosted registry — override only if needed
      build: ${{ inputs.build }}
      deploy: ${{ inputs.deploy }}
      cleanup: ${{ inputs.cleanup }}
    secrets: inherit
```

## Pinning

Callers reference `@main` for convenience. For reproducibility you can pin to a
tag or commit SHA (`...build-deploy.yml@v1`) and bump deliberately.