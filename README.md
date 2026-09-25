# databio/actions

Shared GitHub Actions code for lab repositories. Anything here can be called
from any repo, public or private, in any org (databio, pepkit, refgenie,
bedbase). This repo is public: it must hold no secrets and nothing private.
A called workflow runs inside the calling repo's job and uses the caller's
secrets.

GitHub offers two kinds of shared code, and this repo holds both:

- **Reusable workflows** (`.github/workflows/*.yml` with `on: workflow_call`)
  are whole jobs. Call one from a job:
  `uses: databio/actions/.github/workflows/<name>.yml@master`.
- **Composite actions** (`<name>/action.yml`) are groups of steps that run
  inside the caller's job. Call one from a step:
  `uses: databio/actions/<name>@master`.

## What's here

| Name | Kind | What it does |
|------|------|--------------|
| `notify-doc-hub.yml` | reusable workflow | Tells [databio/doc-hubs](https://github.com/databio/doc-hubs) that a spoke's docs changed, so that site rebuilds |

### notify-doc-hub.yml

A spoke repo adds `.github/workflows/notify-docs.yml`:

```yaml
name: Notify docs

on:
  push:
    branches: [master]
    paths:
      - 'docs/**'
      - '.github/workflows/notify-docs.yml'
  workflow_dispatch:

jobs:
  notify:
    uses: databio/actions/.github/workflows/notify-doc-hub.yml@master
    with:
      site: village              # site slug in doc-hubs sites.toml
      spoke: researcher-profiles # spoke slug in hubs/<site>/spokes.toml
    secrets:
      DOCS_DISPATCH_TOKEN: ${{ secrets.DOCS_DISPATCH_TOKEN }}
```

`DOCS_DISPATCH_TOKEN` is a databio org secret. Repos in other orgs need it set
in their own org. The full spoke checklist (path filters, naming for repos
with several spokes) is in the doc-hubs README, "Add a spoke".

Callers use `@master`, so a change here reaches every caller at once. Keep
changes backward compatible with the `site` / `spoke` inputs, or update
doc-hubs' `deploy.yml` in step.
