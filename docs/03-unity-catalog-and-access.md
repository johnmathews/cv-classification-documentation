# Unity Catalog and external storage access

This is the trickiest part of the setup. Before code can read PDFs from `kagglecvdataset`, four UC objects + one Azure RBAC grant must line up.

## Unity Catalog hierarchy

Unity Catalog (UC) is Databricks' governance layer. Three levels:

```
catalog
└── schema
    └── table / view / function / volume
```

Like database server → database → table in Postgres. Fully-qualified names look like `cv_classification_catalog.dev.cv_bronze`.

In this workspace:
- `cv_classification_catalog` — the project catalog
  - `default` — auto-created empty schema (unused)
  - `dev` — the working schema, holds `cv_bronze`, `cv_silver`, `cv_gold`
  - `information_schema` — auto-generated metadata
- `system` — Databricks-managed (audit logs, billing)
- `samples` — public read-only datasets shared via Delta Sharing

## Default Storage vs External Storage

Tables managed by UC need somewhere to write Delta files. Two backends:

- **Default Storage** — Databricks-managed Azure storage (in the locked `databricks-rg-*` RG). Free Edition uses this by default. Zero config, just works.
- **External Storage** — your own storage account. More setup, but you control encryption keys, region, RBAC.

This catalog uses **Default Storage**: tables live in `abfss://unity-catalog-storage@dbstorage675adlzi5e24w.dfs.core.windows.net/...`. That storage is managed by Databricks.

## Reading from a non-default storage account

The CV PDFs live in `kagglecvdataset` (which is managed outside Databricks). To read them, UC needs to be told that storage account is accessible. Four objects involved:

```
Azure Access Connector  ──>  Storage Credential  ──>  External Location  ──>  Grants
   (managed identity)        (UC reference)          (URL + cred pair)        (READ FILES)
```

### 1. Azure Access Connector (`cv-access-connector`)

A Databricks-specific Azure resource type whose only purpose is to hold a Managed Identity that authenticates to Azure storage. Created in the **`simmons-demo` RG** (your unlocked one, not the locked Databricks-managed one).

### 2. RBAC on the storage account

The connector's managed identity must be granted **Storage Blob Data Contributor** on `kagglecvdataset` via Azure Portal → IAM. Without this, UC can hold the credential but the underlying Azure call fails.

### 3. Storage Credential (`cv_credential`)

A UC object that wraps the Access Connector's resource ID:

```sql
CREATE STORAGE CREDENTIAL cv_credential
WITH AZURE_MANAGED_IDENTITY (
  ACCESS_CONNECTOR_ID = '/subscriptions/.../resourceGroups/simmons-demo/providers/Microsoft.Databricks/accessConnectors/cv-access-connector'
);
```

### 4. External Location (`cv_data`)

Pairs the credential with a specific URL prefix, exposing it to Spark:

```sql
CREATE EXTERNAL LOCATION cv_data
URL 'abfss://kaggle-cv-dataset@kagglecvdataset.dfs.core.windows.net/'
WITH (STORAGE CREDENTIAL cv_credential);
```

### 5. Grants

UC requires explicit `READ FILES` / `WRITE FILES` grants on the external location:

```sql
GRANT READ FILES ON EXTERNAL LOCATION cv_data TO `jonnosgone@gmail.com`;
```

### 6. Test

```sql
LIST 'abfss://kaggle-cv-dataset@kagglecvdataset.dfs.core.windows.net/';
```

## Why Free Edition needs a custom Access Connector

When you create a UC catalog in Free Edition, Databricks auto-creates a Storage Credential called `simmons_demo` (or named after the workspace). This credential is **locked to the Databricks-managed storage path**:

```
PERMISSION_DENIED: The credential 'simmons_demo' is a workspace default credential
that is only allowed to access data in the following paths:
'abfss://unity-catalog-storage@dbstorage675adlzi5e24w.dfs.core.windows.net/7405612721969748'.
```

You **cannot** reuse it to read from your own storage. So Free Edition users must:
1. Create their own Access Connector in Azure
2. Grant it RBAC
3. Create a fresh Storage Credential pointing at it
4. Create the External Location using the new credential

In production / Premium tier, the auto-created credential is more permissive.

## Useful introspection queries

```sql
SHOW EXTERNAL LOCATIONS;
SHOW STORAGE CREDENTIALS;
DESCRIBE STORAGE CREDENTIAL <name>;
DESCRIBE CATALOG EXTENDED cv_classification_catalog;
```

The Access Connector ID format:
```
/subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.Databricks/accessConnectors/<name>
```
