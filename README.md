# databio/actions

Shared GitHub Actions code for lab repositories. Anything here can be called
from any repo, public or private, in any org. This repo is public: it must hold no secrets and nothing private.
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
| `ruff.yml` | reusable workflow | Lints a Python repo with ruff (`ruff check` and `ruff format --check`), using one org-wide ruff version |
| `codon-notify` | composite action | Sends a job's success or failure to Discord through [Codon](https://codon.databio.org) |

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

### ruff.yml

The one lint job for every databio Python repo. A repo's lint workflow is just:

```yaml
name: Lint

on: [push, pull_request]

jobs:
  lint:
    name: Lint
    uses: databio/actions/.github/workflows/ruff.yml@master
```

It runs `ruff check .` and `ruff format --check .` from the repo root, so the
rules come from the repo's own `ruff.toml` or `pyproject.toml`. Ruff is
installed with `astral-sh/ruff-action` (a prebuilt binary), so there is no
Python setup or pip install, and the job takes a few seconds.

**Ruff version.** The default version is set once, in `ruff.yml`. Bump that
default to move every repo to a new ruff at once. A repo that must stay on a
different version overrides it:

```yaml
  lint:
    uses: databio/actions/.github/workflows/ruff.yml@master
    with:
      version: "0.17.0"
```

**Skip tests when lint fails.** Inside the same workflow file, make the test
job wait on lint, so a lint failure does not also spend minutes on tests:

```yaml
  test:
    needs: lint
```

### codon-notify

Add it as the last step of a job. It sends one Discord message with the
status, your message, the repo name and a link to the run:

```yaml
    steps:
      # ... the job's real steps ...
      - name: Notify Codon
        if: always()
        uses: databio/actions/codon-notify@master
        with:
          api-key: ${{ secrets.CODON_API_KEY }}
          url: ${{ secrets.CODON_NOTIFY_URL }}
          status: ${{ job.status }}
          message: BEDbase metadata snapshot
          details: https://example.org/snapshot.json   # optional
```

Pass `status: ${{ job.status }}` yourself: a composite action cannot read the
job's status on its own. `CODON_API_KEY` is the calling repo's Codon caller
key. `CODON_NOTIFY_URL` is a databio org secret holding the endpoint, so the
path stays out of public code. If Codon is down, the step prints a
warning and the job keeps its real result.

Use `if: always()` for rare jobs, such as monthly or quarterly archives, so a
success message tells you the job ran. For daily jobs use `if: failure()`, so
you only hear about it when something breaks.
