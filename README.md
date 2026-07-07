# kovostack-github-workflows

Central CI/CD for Kovospace projects. Holds the **reusable** build → deploy → cleanup
pipeline so every project ships the same way and secrets are configured **once**.

## How it fits together

```
Kovospace (GitHub org)
├── secrets (org-level, set ONCE)      REGISTRY_USER/PASSWORD, VPS_HOST/USER/PASSWORD/PORT
├── kovostack-github-workflows/  ← this repo
│   └── .github/workflows/build-deploy.yml   (on: workflow_call — the real pipeline)
└── kovospace-frontend/, project-b/, ...
    └── .github/workflows/docker-build.yml    (thin caller: uses: + secrets: inherit)
```

- **Secrets live at the org level.** Each project reads them via `secrets: inherit`.
  New project = zero secret setup.
- **Pipeline logic lives here.** Fix a bug / bump an action once, cut a new tag;
  projects pick it up when they bump their `@vX` reference.
- **Per-project differences** (`image_name`, `compose_dir`, the build/deploy/cleanup
  toggles) are passed as `with:` inputs by each caller. `registry` defaults to the
  shared self-hosted registry here — callers only set it to override.

## Versioning (tags)

Callers pin a **tag**, not `@main`, so a project's pipeline never changes under it
unexpectedly:

```yaml
uses: Kovospace/kovostack-github-workflows/.github/workflows/build-deploy.yml@v1
```

Convention (same as official GitHub Actions):

- Cut immutable release tags `v1.0.0`, `v1.1.0`, … on each change.
- Keep a **moving major tag** `v1` that always points at the latest `v1.x.x`, so
  callers on `@v1` get backward-compatible fixes automatically:

  ```bash
  git tag v1.1.0 && git push origin v1.1.0     # immutable release
  git tag -f v1 v1.1.0 && git push -f origin v1 # move the major pointer
  ```

- Breaking change to inputs/secrets → cut `v2` and callers opt in by bumping to `@v2`.
- For maximum reproducibility a caller may pin a commit SHA instead of a tag.

## One-time setup

1. Org **Settings → Secrets and variables → Actions → New _organization_ secret** — add:
   `REGISTRY_USER`, `REGISTRY_PASSWORD`, `VPS_HOST`, `VPS_USER`, `VPS_PASSWORD`, `VPS_PORT`.
   Grant them to *All repositories* (or select).
2. Push this repo and cut the first tags (`v1.0.0` + moving `v1`).
3. Org **Settings → Actions → General**: allow this repo's reusable workflows to be
   called by other repos in the org (Actions must be enabled org-wide).

## Adding a project (the template)

Drop this into `.github/workflows/docker-build.yml`, adjust the two `with:` values:

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
    uses: Kovospace/kovostack-github-workflows/.github/workflows/build-deploy.yml@v1
    with:
      image_name: <project-image-name>
      compose_dir: <path/on/vps/to/compose/dir>
      # registry: defaults to the shared self-hosted registry — override only if needed
      build: ${{ inputs.build }}
      deploy: ${{ inputs.deploy }}
      cleanup: ${{ inputs.cleanup }}
    secrets: inherit
```