# exordos_ci

Shared GitHub Actions workflows for Exordos element repositories.

Workflow | What it does
--- | ---
[`element_build.yml`](.github/workflows/element_build.yml) | Builds the element on a self-hosted runner and pushes it to a repository
[`element_realm_test.yml`](.github/workflows/element_realm_test.yml) | Gets a realm, installs the pushed version on it, checks it, deletes the realm
[`docs.yml`](.github/workflows/docs.yml) | Builds the mkdocs site and deploys it to GitHub Pages
[`markdownlint.yml`](.github/workflows/markdownlint.yml) | Lints the Markdown files a push changed

## Usage

A typical element repository's `.github/workflows/build.yml`:

```yaml
name: build

on:
  push:
    branches-ignore:
      - master
    tags:
      - '*'

jobs:
  Build:
    if: ${{ github.actor != 'dependabot[bot]' }}
    uses: exordos/exordos_ci/.github/workflows/element_build.yml@master
    with:
      push-target: exordos_repo
    secrets:
      PUSH_CFG: ${{ secrets.PUSH_EXORDOS_CFG }}
  Test:
    needs: Build
    uses: exordos/exordos_ci/.github/workflows/element_realm_test.yml@master
    with:
      elements: empty
      version: ${{ needs.Build.outputs.version }}
    secrets: inherit
```

## element_build.yml

Runs on a self-hosted runner that already has packer, KVM and the exordos
CLI: checks out the repository with its full history, takes the version from
`exordos get-version`, runs `exordos build` and `exordos push -f`.

Input | Default | Description
--- | --- | ---
`runs-on` | `["self-hosted", "vm"]` | Runner labels, as JSON
`push-target` | `local` | Target of the push configuration (`local`, `exordos_repo`, ...)
`element-dir` | | Directory to push the artifacts from (`exordos push -e`)
`latest` | `true` | Push a stable version as the latest one too
`build-args` | | Extra arguments to `exordos build`
`build-timeout` | `60` | Minutes the build and push may take

Secret | Description
--- | ---
`PUSH_CFG` | The push configuration (`exordos.push.yaml`), base64-encoded

Output | Description
--- | ---
`version` | The version that was built and pushed

## element_realm_test.yml

Runs on a GitHub-hosted runner: gets a realm from exordos.com, waits until the
pushed version reaches the realm's catalog, installs the elements in order,
runs the check script and deletes the realm, whatever the outcome.

Input | Default | Description
--- | --- | ---
`elements` | required | Space-separated elements to install, in order
`version` | required | Version to install for every element
`dependencies` | | Space-separated elements to install first, at their latest version
`overbook` | `false` | Overbook the realm's hypervisor (cores and RAM ratio 10)
`prepare-script` | | Script in the calling repository to run before the elements are installed
`check-script` | | Script in the calling repository to run once the elements are ACTIVE
`ssh` | `false` | Put a key on the realm; the scripts get `SSH_KEY`, `SSH_HOST`, `SSH_PORT`
`element-timeout` | `540` | Seconds to wait for each element to become ACTIVE
`ci-ref` | `master` | Ref of exordos_ci whose scripts to use; keep it equal to the ref the workflow is called at

The prepare and check scripts run from the calling repository with the CLI
pointed at the realm, and see `REALM_DOMAIN`, `REALM_CORE_URL` and
`ADMIN_PASSWORD`.

Secrets, usually passed with `secrets: inherit`: `TEST_PROJECT_ID`,
`TEST_USERNAME`, `TEST_USER_PASSWORD`.

The helper scripts it uses live in [`.github/scripts`](.github/scripts).

## docs.yml

Runs `tox -e docs-deploy` (`mkdocs gh-deploy`) to push the site to the
`gh-pages` branch, then deploys that branch to GitHub Pages. The caller has
to grant the permissions:

```yaml
name: docs

on:
  push:
    branches:
      - master
    paths:
      - 'docs/**'
      - mkdocs.yml
  workflow_dispatch:

jobs:
  Docs:
    uses: exordos/exordos_ci/.github/workflows/docs.yml@master
    permissions:
      contents: write
      pages: write
      id-token: write
```

Input | Default | Description
--- | --- | ---
`python-version` | `3.12` | Python to run tox with
`tox-env` | `docs-deploy` | tox environment that pushes the site to `gh-pages`

## markdownlint.yml

Lints the Markdown files changed by the push with markdownlint-cli2, using the
calling repository's `.markdownlint.yaml`.

```yaml
name: markdownlint

on:
  push:
    paths:
      - 'docs/**'
      - mkdocs.yml

jobs:
  Lint:
    uses: exordos/exordos_ci/.github/workflows/markdownlint.yml@master
```

Input | Default | Description
--- | --- | ---
`files` | `**/*.md` | Files to lint when changed, as changed-files patterns
