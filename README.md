# kovostack-github-workflows

Central CI/CD for Kovospace projects. Holds the **reusable** build → deploy
pipeline so every project ships the same way and secrets are configured **once**.

Deployment is **GitOps**: CI never touches the cluster. It builds the image,
pushes it to the registry, and commits the new tag into the infra repo — the
cluster's GitOps controller rolls it out from there.

## How it fits together

```
Kovospace (GitHub org)
├── secrets (org-level, set ONCE)      REGISTRY_USER/PASSWORD, GITOPS_DEPLOY_KEY
├── kovostack-github-workflows/  ← this repo
│   └── .github/workflows/build-deploy.yml   (on: workflow_call — the real pipeline)
├── kovospace-frontend/, project-b/, ...
│   └── .github/workflows/docker-build.yml    (thin caller: uses: + secrets: inherit)
└── kovostack-infra-gitops/
    └── versions/<app_namespace>.yaml         (imageTag: sha-abc1234)  ← CI writes this
```

- **Secrets live at the org level.** Each project reads them via `secrets: inherit`.
  New project = zero secret setup.
- **Pipeline logic lives here.** Fix a bug / bump an action once, cut a new tag;
  projects pick it up when they bump their `@vX` reference.
- **Per-project differences** (`image_name`, `app_namespace`, the build/deploy
  toggles) are passed as `with:` inputs by each caller. `registry`,
  `registry_namespace`, `gitops_repo` and `gitops_branch` default to the shared
  values — callers only set them to override.
- **Branch name doesn't matter.** Build and deploy run on the calling repo's
  *default* branch, whether that is `main`, `master` or anything else — the guard
  compares against `github.event.repository.default_branch`. Pass
  `release_branch: <name>` only to release from a non-default branch.
- **Images live under `apps/`** in the registry:
  `<registry>/apps/<image_name>:<tag>`. Override `registry_namespace` for images
  that aren't applications (e.g. `infra`, `base`).

## The deploy step

1. The build job tags the image `sha-<7-char commit sha>` (plus `latest`) and
   pushes it to `<registry>/apps/<image_name>`.
2. The deploy job clones `Kovospace/kovostack-infra-gitops` over SSH (deploy key),
   writes

   ```yaml
   imageTag: sha-abc1234
   ```

   to `versions/<app_namespace>.yaml`, then commits and pushes it.
3. Pushes are retried with `--rebase` up to 5× — several projects write to that
   repo concurrently.

Deploy-only run (uncheck **build**): re-pins `versions/<app_namespace>.yaml` to the
tag of the currently checked-out commit — that's how you roll back or forward to an
image that was already built.

Old image tags are deliberately **not** pruned from the registry any more: a GitOps
rollback must still be able to pull them.

## Image tags

By default the image is tagged `sha-<7-char commit sha>` plus `latest`, and the GitOps
repo is pinned to that sha tag. That is right for an application: the commit is the
only version it has.

Some artifacts carry a version of their own that consumers pin explicitly — a
database migration image, whose tag *is* the schema version the application expects.
Those callers pass **`image_version`**:

```yaml
with:
  image_name: new-tab-links-migrations
  image_version: 0.0.1        # -> <registry>/apps/new-tab-links-migrations:0.0.1
  deploy: false
```

The tag is then used verbatim instead of the commit-derived one. Leave it empty and
nothing changes for existing callers.

**Build-only callers** (`deploy: false`) may also omit `app_namespace`: it selects a
file in the GitOps repo, and an artifact that is not deployed as its own workload has
no such file. The deploy job refuses to run without it.

## Versioning (tags)

Callers pin a **tag**, not `@main`, so a project's pipeline never changes under it
unexpectedly:

```yaml
uses: Kovospace/kovostack-github-workflows/.github/workflows/build-deploy.yml@v2
```

Convention (same as official GitHub Actions):

- Cut immutable release tags `v2.0.0`, `v2.1.0`, … on each change.
- Keep a **moving major tag** `v2` that always points at the latest `v2.x.x`, so
  callers on `@v2` get backward-compatible fixes automatically:

  ```bash
  git tag v2.0.0 && git push origin v2.0.0      # immutable release
  git tag -f v2 v2.0.0 && git push -f origin v2 # move the major pointer
  ```

- Breaking change to inputs/secrets → cut the next major, callers opt in by bumping
  their `@vX`. The Kubernetes switch is exactly that: `v1` callers pass `compose_dir`
  + `cleanup` and deploy over SSH, `v2` callers pass `app_namespace`.
- For maximum reproducibility a caller may pin a commit SHA instead of a tag.

## One-time setup

1. GitOps repo (`kovostack-infra-gitops`) → **Settings → Deploy keys → Add deploy
   key**: paste the public key and tick **Allow write access**.
2. Org **Settings → Secrets and variables → Actions → New _organization_ secret** — add:
   `REGISTRY_USER`, `REGISTRY_PASSWORD`, and `GITOPS_DEPLOY_KEY` (the *private* half
   of that deploy key, full PEM including the BEGIN/END lines).
   Grant them to *All repositories* (or select).
3. Push this repo and cut the tags (`v2.0.0` + moving `v2`).
4. Org **Settings → Actions → General**: allow this repo's reusable workflows to be
   called by other repos in the org (Actions must be enabled org-wide).

## Adding a project (the template)

Drop this into `.github/workflows/docker-build.yml`, adjust the two `with:` values:

```yaml
name: Build and push Docker image
on:
  workflow_dispatch:
    inputs:
      build:  { description: 'Build and push the image', type: boolean, default: true }
      deploy: { description: 'Commit the tag to GitOps', type: boolean, default: true }
jobs:
  pipeline:
    uses: Kovospace/kovostack-github-workflows/.github/workflows/build-deploy.yml@v2
    with:
      image_name: <project-image-name>     # -> <registry>/apps/<project-image-name>
      app_namespace: <k8s-app-namespace>   # -> versions/<k8s-app-namespace>.yaml
      # registry / registry_namespace / gitops_repo / gitops_branch default to the
      # shared values; the pipeline runs on this repo's default branch (main,
      # master, …) — set release_branch: <name> only to release from another one
      build: ${{ inputs.build }}
      deploy: ${{ inputs.deploy }}
    secrets: inherit
```