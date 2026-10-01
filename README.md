# ci-workflows

The shared GitHub Actions pipeline for the apps on `sun`. One reusable workflow,
`.github/workflows/build-deploy.yml`, called by each app repo:

```yaml
name: Build and deploy
on:
  push:
    branches: [main]
    tags: ['v*']
  workflow_dispatch:

jobs:
  build:
    uses: vongaisberg/ci-workflows/.github/workflows/build-deploy.yml@main
    with:
      image: <name>          # Zot repo + apps/<name>/ in sun-gitops
    secrets: inherit
```

It builds the image on the self-hosted runner, pushes it to Zot as OCI, and
commits the new SHA tag into `apps/<name>/values.yaml` in
[sun-gitops](https://github.com/vongaisberg/sun-gitops), which Argo CD then
reconciles. See that repo's README for the pipeline end to end.

**Public on purpose.** `crm` and `sun-gitops` are private; a reusable workflow in
a private repo needs per-caller access grants, while a public one is callable
from anywhere. Nothing secret lives here — the registry address is a private
RFC1918 address and every credential arrives from the caller via
`secrets: inherit`.

**Callers track `@main`.** That is the point: a fix here reaches every app at
once. The cost is that a bad commit here breaks every pipeline at once, so
callers can pin a SHA instead if that ever matters.

## What a caller still owns

- **Its own runner.** `vongaisberg` is a personal account, so GitHub has no
  org-level runner groups: a runner registered to one repo is invisible to every
  other. The `ci-runner` VM runs one systemd unit per repo, each labelled `sun`.
- **Its own secrets:** `ZOT_USERNAME`, `ZOT_PASSWORD`, `GITOPS_DEPLOY_KEY` (a
  deploy key with *write* access to sun-gitops; generate one per app).
- **Its own Dockerfile** — including the BuildKit cache mounts that keep builds
  incremental. A cache mount is a flag on a `RUN` line, so no shared workflow can
  supply it. Without them barbeleuchtung took 22m09s per build; with them, a
  one-line change recompiles one crate.

## Inputs

| Input | Default | Purpose |
|---|---|---|
| `image` | — | Zot repository name, and the `apps/<image>/` directory in sun-gitops |
| `values_file` | `apps/<image>/values.yaml` | Which file's `image:` line gets the tag. `unishort` passes `apps/unishort/dev.yaml` on its `dev` branch |
| `runner_label` | `sun` | Second label of the runner to build on |
