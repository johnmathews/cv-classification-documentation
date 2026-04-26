# Documentation

Notes and decisions for the CV classification case study.

## Contents

1. [Azure storage setup](./01-azure-storage.md) — storage account, container, redundancy, ADLS Gen 2.
2. [Databricks Asset Bundles](./02-databricks-bundles.md) — bundle structure, deploy/run loop, wheels, jobs vs pipelines.
3. [Unity Catalog and external storage access](./03-unity-catalog-and-access.md) — UC hierarchy, access connector chain, why Free Edition needed a custom connector.
4. [Pipeline structure](./04-pipeline-structure.md) — bronze/silver/gold tables and the three task entry points.

## Project layout

```
case-study/
├── case-study.md              # original brief
├── docs/                      # this directory
├── journal/                   # dated decision/progress entries
└── cv_classification/         # the Databricks Asset Bundle
    ├── databricks.yml         # bundle config (targets, variables)
    ├── pyproject.toml         # Python package + entry points
    ├── resources/
    │   └── cv_pipeline.job.yml
    ├── src/
    │   └── cv_classification/
    │       ├── common.py
    │       ├── ingest.py
    │       ├── preprocess.py
    │       └── classify.py
    └── tests/
```

## Key facts to remember

- **Workspace**: `https://adb-7405612721969748.8.azuredatabricks.net` (Azure Databricks Free Edition)
- **Catalog**: `cv_classification_catalog` (managed by Databricks default storage)
- **Schema**: `dev` (case-study scope; `prod` exists but unused)
- **External data**: `abfss://kaggle-cv-dataset@kagglecvdataset.dfs.core.windows.net/`
- **Region**: West Europe (workspace + storage account must match to avoid egress)
