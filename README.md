# Prova Run Github Action

Run a [Prova](https://github.com/prova-rs/prova) acceptance-test suite inside a GitHub Workflow.
The action installs a released `prova` binary (no Rust toolchain, no build from source) and runs
your suite.

## Basic usage

Run the suite declared in `./prova.toml`:

```yaml
- uses: prova-rs/run-action@v1
```

Run a manifest profile:

```yaml
- uses: prova-rs/run-action@v1
  with:
    profile: ci
```

Run specific files or directories (this bypasses the manifest):

```yaml
- uses: prova-rs/run-action@v1
  with:
    paths: "tests/acceptance"
    jobs: "8"
```

## Inputs

| Input | Default | Description |
| --- | --- | --- |
| `version` | `v0.1.0` | The [Prova release](https://github.com/prova-rs/prova/releases) to install |
| `paths` | — | Files/dirs to run. Set this and the manifest is bypassed. |
| `manifest` | `prova.toml` | Path to the suite manifest |
| `profile` | — | Manifest profile to run |
| `jobs` | — | Run up to N units concurrently |
| `format` | `console` | `console` or `json` (JSONL event stream) |
| `working-directory` | `.` | Directory to run `prova` from |
| `args` | — | Extra arguments appended to the invocation |
| `plugins` | — | Ad-hoc plugins, one `name = source` per line, layered over the manifest |
| `cache-plugins` | `true` | Cache fetched git plugins across runs (`false` to disable) |

The action also puts `prova` on `PATH`, so later steps in the same job can call it directly.

## Plugins

Plugins declared in `prova.toml`'s `[plugins]` need nothing here — the action fetches and
**caches** them (`~/.cache/prova/plugins`, keyed on the manifest) so pinned plugins clone once and
reuse across runs. Disable with `cache-plugins: false`.

Add CI-only plugins (or override a manifest one) with `plugins:`, one `name = source` per line —
`source` is a path, a git URL, or an `org/repo@ref` shorthand:

```yaml
- uses: prova-rs/run-action@v1
  with:
    profile: ci
    plugins: |
      loadtest = acme/prova-loadtest@v2
      redis    = github:acme/prova-redis@v1
```

Runners: Linux (x86_64/arm64) and macOS (arm64).

## Example: suite against a CI-provided Postgres

The same suite runs against ephemeral containers on a laptop (via Prova's `docker` module) or
against CI-provided services here — the tests only see connection details through env, set by the
manifest profile.

```yaml
name: acceptance
on: [push, pull_request]

jobs:
  prova:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: secret
          POSTGRES_DB: orders
        ports:
          - 5432:5432
        options: >-
          --health-cmd "pg_isready"
          --health-interval 5s
          --health-timeout 5s
          --health-retries 10
    env:
      DATABASE_URL: postgres://postgres:secret@localhost:5432/orders
    steps:
      - uses: actions/checkout@v4
      - uses: prova-rs/run-action@v1
        with:
          profile: ci
```
