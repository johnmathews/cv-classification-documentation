# 2026-04-26 — initial setup

First working session. Goal: stand up the infrastructure end-to-end so we can iterate on real pipeline logic next session.

## What was built

### Azure
- Storage account `kagglecvdataset` in RG `simmons-demo` (West Europe, LRS, ADLS Gen 2)
- Container `kaggle-cv-dataset` with the Kaggle resume PDFs uploaded
- Access Connector `cv-access-connector` in `simmons-demo` RG
- Storage Blob Data Contributor role on `kagglecvdataset` granted to the connector's managed identity

### Databricks
- Workspace already existed; re-authed against the correct host (`adb-7405612721969748.8.azuredatabricks.net`)
- Catalog `cv_classification_catalog` (managed by Databricks Default Storage)
- Schema `cv_classification_catalog.dev`
- Storage Credential `cv_credential` referencing the Access Connector
- External Location `cv_data` covering `abfss://kaggle-cv-dataset@kagglecvdataset.dfs.core.windows.net/`
- Asset Bundle `cv_classification` deployed and run successfully

### Code
- DAB scaffold via `databricks bundle init` (default-python template, no notebooks, no DLT, no personal schemas)
- Sample taxi code deleted; replaced with three modules: `ingest.py`, `preprocess.py`, `classify.py` (all stubs)
- Job `cv_pipeline` with three chained tasks
- Pure-python pytest test for `common.parse_args`
- Documentation in `docs/`

## Decisions

- **DABs over notebooks.** Modular code beats notebook for code review.
- **Jobs over DLT.** LLM step doesn't fit DLT's declarative model. Dataset is small.
- **Schema-as-environment** (`dev` / `prod`). Cheap, fine for solo project.
- **LRS redundancy.** Data is reproducible from Kaggle.
- **Wheel-based deploy.** No notebook syncing, no remote-edit drift.

## Things that broke and why

- **DNS error during `bundle init`.** The default profile in `~/.databrickscfg` pointed at a stale workspace URL. Re-auth fixed it.
- **Terraform GPG key expired.** Databricks CLI's embedded Terraform downloader fails because HashiCorp's signing key expired. Workaround: install Terraform via `brew tap hashicorp/tap && brew install hashicorp/tap/terraform`, then `export DATABRICKS_TF_EXEC_PATH=$(which terraform)` and `DATABRICKS_TF_VERSION=1.14.9`.
- **Catalog creation rejected blank storage location.** Free Edition's "Default Storage" path didn't auto-trigger; had to use the existing Databricks-managed external location.
- **Workspace default credential refused custom storage.** The auto-created `simmons_demo` storage credential is path-locked to `unity-catalog-storage@dbstorage...`. Created our own access connector + credential to read from `kagglecvdataset`.

## What's next

- Implement real `ingest.py` reading PDFs from the External Location
- Implement real `preprocess.py` extracting text (likely PyMuPDF)
- Implement real `classify.py` using an LLM for experience-bracket prediction
- Add Spark-backed integration tests (uses Databricks Connect serverless)
- Write the report deliverable (25% of effort per the brief)
