# Databricks Asset Bundles (DABs)

DABs are Databricks' "code-as-config" project format — a YAML-defined unit that bundles code, resources (jobs, pipelines, schemas), and per-environment targets. Released GA in 2024.

Think Terraform-meets-`docker-compose`, scoped to Databricks.

## Why we chose DABs over notebooks

The case study brief asks for "clean, modular code." Options:

| Approach | Pros | Cons |
|---|---|---|
| **DABs + Python files** (chosen) | Modular, testable, git-friendly, CI-friendly | Slight learning curve |
| Databricks Connect | Good for iteration | Doesn't show as a deliverable artifact |
| Notebooks | Native, easy | Hard to test, hard to git-diff, anti-pattern for production |

For a code review, modular `.py` + tests + bundle config beats a notebook every time.

## Project shape

```
cv_classification/
├── databricks.yml              # bundle config (targets, variables)
├── pyproject.toml              # Python deps + entry points
├── resources/
│   └── cv_pipeline.job.yml     # job definition
├── src/
│   └── cv_classification/      # Python package, builds into a wheel
└── tests/
```

## The wheel

A `.whl` is Python's standard binary package format — a zip file with the package code, metadata, and entry points.

When `bundle deploy` runs, it executes `uv build --wheel` (per the `artifacts` block in `databricks.yml`), producing `dist/cv_classification-0.0.1-py3-none-any.whl`. The wheel is uploaded to the workspace, and tasks of type `python_wheel_task` install it on the cluster and call its declared entry points.

Inspect with: `unzip -l dist/cv_classification-*.whl`

## Core commands

```
databricks bundle validate              # lint YAML, no network changes
databricks bundle deploy -t dev         # build wheel, upload, create job
databricks bundle run -t dev <job>      # execute the job
databricks bundle destroy -t dev        # remove deployed resources
```

## Jobs vs Pipelines

DABs can deploy two kinds of executable resources, and they're often confused:

**Jobs** — imperative DAG of arbitrary tasks (Python wheels, notebooks, SQL, etc.). What we use.

**Pipelines (DLT / Lakeflow Declarative Pipelines)** — declarative ETL. You write `@dlt.table` functions; Databricks figures out dependencies, incremental updates, schema evolution, quality checks.

For our pipeline (ingest → preprocess → classify with an LLM step), **Jobs** is the right tool. DLT would shine for pure Delta-to-Delta transformations but adds complexity for the LLM step and isn't justified for a 4-hour case study.

## Environment separation

Our `databricks.yml` defines two targets:

- `dev` — writes to schema `cv_classification_catalog.dev`, prefixes job names with `[dev <user>]`
- `prod` — writes to `cv_classification_catalog.prod`, no prefix, requires explicit deploy

This is **schema-as-environment**: cheap and good enough for a solo project. Real production setups use:
1. Separate workspaces (strongest)
2. Separate catalogs
3. Separate schemas in one catalog (what we do)

## Stable build environment caveats

Two annoyances that bit us during setup:

1. **HashiCorp Terraform GPG key expired**: the embedded Terraform downloader inside the Databricks CLI verifies binaries with HashiCorp's signing key. When that key expires (it has, multiple times), `bundle deploy` fails. Workaround:
   ```sh
   brew tap hashicorp/tap && brew install hashicorp/tap/terraform
   export DATABRICKS_TF_EXEC_PATH=$(which terraform)
   export DATABRICKS_TF_VERSION=1.14.9   # or whatever local version
   ```
   These exports must be in `~/.zshrc` to persist across shell sessions.

2. **DNS resolution of stale workspace URLs**: if you re-auth against a new workspace, the old `[DEFAULT]` profile still points at the old (potentially deleted) host, and `bundle init` calls UC APIs against it during template instantiation, failing with "no such host". Either edit `~/.databrickscfg` or use `--profile <name>`.

## The deploy/run/destroy mental model

- `deploy` is `git push` + `docker build/push` for Databricks
- `run` actually starts the work (uses cluster time, costs money)
- `destroy` removes the workspace-side state (jobs, files), but **never** local code, catalogs, schemas, or tables — only what the bundle itself owns

## Tables persist after `destroy`

Important to remember: even if a job wrote to a table, the table is **not** a bundle resource (we didn't declare it in `resources/`). It survives `destroy`. To wipe data:

```sql
DROP SCHEMA cv_classification_catalog.dev CASCADE;
```
