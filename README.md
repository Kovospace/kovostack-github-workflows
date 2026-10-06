# kovostack-github-workflows

Central CI/CD for Kovospace projects. Holds the **reusable** workflows so every
project ships the same way and secrets are configured **once**.

## Workflows

| Workflow | Purpose | Latest tag | Secrets the caller needs |
| --- | --- | --- | --- |
| [`build-deploy.yml`](.github/workflows/build-deploy.yml)<br>`.github/workflows/build-deploy.yml` | Build a Docker image, push it to the registry, commit its tag into the GitOps repo. | `2.1.0` | **Org:** `REGISTRY_USER`, `REGISTRY_PASSWORD` (always); `GITOPS_DEPLOY_KEY` (only when `deploy: true`) |
| [`android-release.yml`](.github/workflows/android-release.yml)<br>`.github/workflows/android-release.yml` | Build a signed Android APK, tag the repo with the `versionName`, publish a GitHub release with the APK. | `2.2.0` | **Repo** (per app): `KEYSTORE_BASE64`, `KEYSTORE_PROPERTIES_BASE64` |
| [`flyway-release.yml`](.github/workflows/flyway-release.yml)<br>`.github/workflows/flyway-release.yml` | Release a Flyway migrations image: next `x.y.z` version → validate → build & push (no deploy) → tag the repo. | `2.3.0` | **Org:** `REGISTRY_USER`, `REGISTRY_PASSWORD` |
| [`flyway-validate.yml`](.github/workflows/flyway-validate.yml)<br>`.github/workflows/flyway-validate.yml` | Apply the migrations to a throwaway Postgres along the fresh and upgrade paths, and require both to end in the same schema. For pull requests; `flyway-release.yml` runs it too. | `2.3.0` | *none* |

*Latest tag* = the newest tag that changed that workflow. Callers can pin it or any
newer tag. Every tag carries all the workflows, so a newer one changes nothing for
a workflow it didn't touch.

All callers pass `secrets: inherit`, so it doesn't matter whether a secret is set at
the org or repo level. The column shows where it *belongs*: registry and GitOps
credentials are shared, but a signing keystore is one per app.

---

# `build-deploy.yml` — build → push → GitOps

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

## Pinning an init container's tag

The other half of that story: the migrations image is built by its own repository and
never deploys itself, so somebody has to write its tag into the GitOps repo. That is
the *application* that runs it as an init container — it is the side that knows which
schema version it was written against.

Such a caller passes **`init_image_tags`**, one `name=tag` per line:

```yaml
with:
  image_name: new-tab-links-backend
  app_namespace: new-tab-links-backend
  init_image_tags: migrations=0.0.1     # name = the init container's name in the chart
```

The deploy job then writes, next to the application's own version file:

```yaml
# versions/<app_namespace>-init.yaml
initImageTags:
  migrations: "0.0.1"
```

Both files are loaded by the app chart and neither overwrites the other — `imageTag`
and `initImageTags` are different keys, and `initImageTags` is a *map*, which Helm
merges across values files (a list would not). Values are quoted because an unquoted
`1.2` is a YAML float. The key must be the init container's **resolved name** in
`applications/<app>/values.yaml`.

Leave the input empty (the default) and no such file is written or touched.

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
---

# `android-release.yml` — signed Android release

## What it does

You trigger it manually from the Android repository. It only runs from that repo's
default branch, or `release_branch` when set: any other branch or ref type fails
the run. It will:

1. Read `versionName` from the module's gradle file (`app/build.gradle` by default).
2. Fail early if the tag for that version already exists, so the same version can't be released twice.
3. Decode the keystore secrets, build `:app:assembleRelease`, and verify the APK signature with `apksigner`.
4. Rename the APK to `<repo-name>-<version>.apk`.
5. Create and push an annotated tag `v<version>` in the **Android** repository.
6. Create a GitHub release for that tag with the APK attached.

Signing material is deleted from the runner right after the build, before anything is pushed.

No registry and no GitOps repo are involved. The release lives entirely in the
calling repository and uses its built-in `GITHUB_TOKEN`.

## Setup checklist

### 1. Secrets

In the Android repository: **Settings → Secrets and variables → Actions → New repository secret**.
(They can be org secrets too, but a keystore usually belongs to exactly one app.)

| Secret | Content |
| --- | --- |
| `KEYSTORE_BASE64` | Base64 of your `release.jks` |
| `KEYSTORE_PROPERTIES_BASE64` | Base64 of your `keystore.properties` |

Generate the values locally. `-w 0` matters, because each value must be a single line:

```bash
base64 -w 0 release.jks > release.jks.b64
base64 -w 0 keystore.properties > keystore.properties.b64
# macOS: base64 -i release.jks -o release.jks.b64
```

Paste the contents of each `.b64` file as the secret value, then delete the temporary files.

`keystore.properties` should look like this:

```properties
storePassword=****
keyPassword=****
keyAlias=release
storeFile=release.jks
```

> The workflow rewrites `storeFile` to the absolute path of the decoded keystore on the
> runner, so whatever you put there locally is fine.

### 2. Signing config in `app/build.gradle`

The workflow only provides the files; the build has to use them. Groovy DSL:

```groovy
def keystorePropertiesFile = rootProject.file("keystore.properties")
def keystoreProperties = new Properties()
if (keystorePropertiesFile.exists()) {
    keystoreProperties.load(new FileInputStream(keystorePropertiesFile))
}

android {
    defaultConfig {
        versionCode 12
        versionName "1.2.3"   // <- this is what the workflow reads
    }

    signingConfigs {
        release {
            storeFile file(keystoreProperties['storeFile'])
            storePassword keystoreProperties['storePassword']
            keyAlias keystoreProperties['keyAlias']
            keyPassword keystoreProperties['keyPassword']
        }
    }

    buildTypes {
        release {
            signingConfig signingConfigs.release
            minifyEnabled true
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
        }
    }
}
```

Kotlin DSL (`app/build.gradle.kts`) works the same way: point the `gradle_file`
input at it. Both `versionName = "1.2.3"` and `versionName("1.2.3")` are recognised.

### 3. `.gitignore`

Never commit the signing material:

```gitignore
release.jks
keystore.properties
*.b64
```

### 4. Workflow permissions

Tagging and releasing use the built-in `GITHUB_TOKEN`. Check the Android repository's
**Settings → Actions → General → Workflow permissions**. If it is set to
*Read repository contents permission*, the caller's `permissions: contents: write`
(shown below) grants the write access, so keep that block.

### 5. Caller workflow

Drop this into the Android repository as `.github/workflows/release.yml`. Pin the
newest tag from the [table above](#workflows):

```yaml
name: Release

on:
  workflow_dispatch:
    inputs:
      prerelease:
        description: 'Mark the release as a pre-release'
        type: boolean
        default: false

permissions:
  contents: write

jobs:
  release:
    uses: Kovospace/kovostack-github-workflows/.github/workflows/android-release.yml@2.2.0
    with:
      gradle_file: app/build.gradle
      module: app
      prerelease: ${{ inputs.prerelease }}
    secrets: inherit
```

Run it from **Actions → Release → Run workflow** with the default branch selected.

## Inputs

All inputs are optional.

| Input | Default | Description |
| --- | --- | --- |
| `gradle_file` | `app/build.gradle` | File the `versionName` is read from. |
| `module` | `app` | Gradle module to build; also where the APK is looked up. |
| `gradle_task` | `assembleRelease` | Assemble task, run as `:<module>:<task>`. |
| `gradle_extra_args` | *(empty)* | Extra arguments for the gradle invocation, e.g. `-Pfoo=bar`. |
| `apk_name_prefix` | repository name | Base name of the APK: `<prefix>-<version>.apk`. |
| `tag_prefix` | `v` | Tag is `<prefix><version>`. Set to `""` for a bare `1.2.3` tag. |
| `version_override` | *(empty)* | Skip reading the gradle file and use this version instead. |
| `java_version` | `17` | JDK used for the build. |
| `java_distribution` | `temurin` | `actions/setup-java` distribution. |
| `keystore_path` | `release.jks` | Where the decoded keystore is written. |
| `keystore_properties_path` | `keystore.properties` | Where the decoded properties file is written. |
| `apk_search_path` | `<module>/build/outputs/apk` | Directory scanned for the built APK(s). |
| `release_name` | the tag | Release title. |
| `release_notes` | *(empty)* | Release body; used only when `generate_release_notes` is `false`. |
| `generate_release_notes` | `true` | Let GitHub generate notes from commits/PRs. |
| `prerelease` | `false` | Mark the release as a pre-release. |
| `draft` | `false` | Create the release as a draft. |
| `verify_signature` | `true` | Run `apksigner verify --print-certs` on the APK. |
| `upload_build_artifact` | `true` | Also attach the APK as a workflow artifact. |
| `release_branch` | repo's default branch | The only branch that may release. |
| `runs_on` | `ubuntu-latest` | Runner label for the build job. |

## Secrets

| Secret | Required | Description |
| --- | --- | --- |
| `KEYSTORE_BASE64` | yes | Base64 encoded `release.jks`. |
| `KEYSTORE_PROPERTIES_BASE64` | yes | Base64 encoded `keystore.properties`. |

## Outputs

| Output | Description |
| --- | --- |
| `version` | Version that was built. |
| `tag` | Tag that was created. |
| `release_url` | URL of the created release. |

## Product flavors

If the build produces more than one APK (flavors), every APK found that isn't
unsigned is attached to the release. Each is named `<prefix>-<version>-<variant>.apk`,
where `<variant>` is the output directory name. To release a single flavor, point the
task at it: `gradle_task: assembleProdRelease`.

## Troubleshooting

| Symptom | Cause / fix |
| --- | --- |
| `Releases can only be built from the '<branch>' branch` | The workflow was dispatched from another branch. Switch the branch in the *Run workflow* dialog, or set `release_branch`. |
| `Could not read a literal versionName` | The version comes from a variable, `libs.versions.toml` or a properties file. Pass `version_override`, or keep a literal `versionName` in the gradle file. |
| `Tag 'v1.2.3' already exists` | Bump `versionName` and re-run. The workflow refuses to overwrite a released version. |
| `Decoding the signing secrets produced an empty file` | The secret was created with line-wrapped base64. Re-encode with `base64 -w 0`. |
| `No signed APK found` | The release build type has no `signingConfig`, or the APK is written somewhere else. Set `apk_search_path`. |
| `Resource not accessible by integration` when tagging | The caller is missing `permissions: contents: write`, or the repo's workflow permissions are read-only. |

---

# `flyway-release.yml` / `flyway-validate.yml` — Flyway migrations image

For a repository that holds only Flyway migrations (`sql/V<n>__*.sql`) and a
`Dockerfile` `FROM flyway/flyway` that copies them to `/flyway/sql`. Its image runs as
an init container of an application, and **its tag is the schema version**: the
application pins it through `init_image_tags` in `build-deploy.yml`.

## `flyway-validate.yml`

Flyway has no offline check, so this builds the caller's image and applies the
migrations to a throwaway Postgres service container along two paths:

| | what it proves |
|---|---|
| **fresh** | an empty database takes every migration from `V1`, as a new environment does |
| **upgrade** | a database migrated to the newest `x.y.z` tag takes the new migrations on top, as production does |

It then `pg_dump`s both databases and requires them to be identical. Before any of
that, it checks that the migrations directory is append-only since the newest tag:
an edited migration means a checksum mismatch, and the application won't start.

| Input | Default | Description |
| --- | --- | --- |
| `postgres_image` | `postgres:17-alpine` | Must track the real database's **major** version. |
| `migrations_dir` | `sql` | Where the `V<n>__*.sql` files live. |

## `flyway-release.yml`

1. **Version.** The newest `x.y.z` tag with its patch number bumped (`0.0.1` when
   there are none), or the exact `version` input for a minor or major bump. The run
   fails if that tag already exists, or if it was dispatched from a branch other
   than the release branch.
2. **Validate.** Runs `flyway-validate.yml`. `skip_validation: true` skips it, for
   when the check itself is wrong.
3. **Build & push.** Runs `build-deploy.yml` with `image_version: <version>` and
   `deploy: false` → `<registry>/apps/<image_name>:<version>` (plus `latest`).
4. **Tag.** Tags the calling repository `<version>`, *after* the image is pushed,
   so a tag exists only for a version that was really published.

| Input | Default | Description |
| --- | --- | --- |
| `image_name` | — (required) | Image name, without registry or namespace. |
| `version` | *(empty)* | Exact version to publish; empty bumps the patch number. |
| `skip_validation` | `false` | Publish without validating. |
| `postgres_image` | `postgres:17-alpine` | Passed to `flyway-validate.yml`. |
| `migrations_dir` | `sql` | Passed to `flyway-validate.yml`. |
| `registry_namespace` | `apps` | Passed to `build-deploy.yml`. |
| `release_branch` | repo's default branch | The only branch that may release. |

Output: `version`, the version that was published.

The release workflow calls the other two by **full path and tag**
(`Kovospace/kovostack-github-workflows/...@2.3.0`), not `./`: inside a reusable
workflow, a relative path resolves to the *caller's* repository. Bump those refs
in the same commit as the tag that releases a change to them.

## Caller workflows

`.github/workflows/docker-build.yml`:

```yaml
name: Build & push migrations image
on:
  workflow_dispatch:
    inputs:
      version:
        description: 'Override the version to publish (e.g. 0.1.0). Leave empty to bump the patch number of the newest tag.'
        type: string
        required: false
      skip_validation:
        description: 'Skip validating the migrations against a throwaway Postgres.'
        type: boolean
        default: false

# One release at a time - two runs would both read the same newest tag.
concurrency:
  group: release-${{ github.repository }}
  cancel-in-progress: false

jobs:
  release:
    uses: Kovospace/kovostack-github-workflows/.github/workflows/flyway-release.yml@2.3.0
    permissions:
      contents: write   # the release pushes a tag
    with:
      image_name: <project>-migrations
      version: ${{ inputs.version }}
      skip_validation: ${{ inputs.skip_validation }}
    secrets: inherit
```

`.github/workflows/pull-request.yml`:

```yaml
name: Validate migrations (PR)
on:
  pull_request:
    paths: ['sql/**', 'Dockerfile', '.github/workflows/pull-request.yml']
concurrency:
  group: pr-${{ github.event.pull_request.number }}
  cancel-in-progress: true
jobs:
  validate:
    uses: Kovospace/kovostack-github-workflows/.github/workflows/flyway-validate.yml@2.3.0
```
