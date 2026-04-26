# Azure storage setup

The CV PDFs need to live somewhere Spark can read from. Azure Blob Storage with hierarchical namespace enabled (i.e. ADLS Gen 2) was chosen.

## Hierarchy in Azure Blob Storage

- **Storage account** — top-level resource with a globally unique DNS name (e.g. `kagglecvdataset.blob.core.windows.net`)
- **Container** — a bucket inside the account, holds blobs
- **Blob** — the individual file

For this case study:
- Storage account: **`kagglecvdataset`**
- Resource group: **`simmons-demo`** (NOT `databricks-rg-*` — that's the locked Databricks-managed RG)
- Container: **`kaggle-cv-dataset`**
- Blobs: ~2,488 PDF files from the Kaggle resume dataset

## Blob vs File vs Disk storage

| | **Blob (Object)** | **File** | **Disk (Block)** |
|---|---|---|---|
| Hierarchy | Flat (containers + blobs) | Real filesystem | Raw VHD |
| Access | HTTPS REST / SDK | SMB / NFS (mountable drive) | Attached to VM |
| Use for | Programmatic, big data | Lift-and-shift apps, shared drives | VM disks |
| Cost | Cheapest | Higher | Per-VM |

Spark reads via SDK, so **Blob** is the right choice.

## ADLS Gen 2 (hierarchical namespace)

When the storage account is created, **Hierarchical namespace** must be enabled. This upgrades Blob Storage to ADLS Gen 2:
- Real directories instead of pretend ones (paths like `/folder/sub/file.pdf` are first-class)
- Faster listing of large prefixes
- Atomic rename
- Required for the modern `abfss://` Spark connector to work efficiently
- Always enable for analytics workloads — the price is the same

## Redundancy choice: LRS

**Locally-redundant storage** (LRS) was selected — three copies in one datacenter.

Rationale: the dataset is reproducible from Kaggle. Paying for geo-redundancy on disposable data is wasted money. Reserve GRS/GZRS for data you can't recreate.

| Tier | Copies | Use when |
|---|---|---|
| **LRS** | 3 in one DC | Reproducible / non-critical (this project) |
| **ZRS** | 3 across 3 DCs in one region | High availability |
| **GRS** | LRS + async replica in second region | Backup of irreplaceable data |
| **GZRS** | ZRS + geo replica | Critical data |

## Region: West Europe

Must match the Databricks workspace region. Cross-region reads cost egress and add latency. West Europe was chosen for both.

## Block size during upload

For tiny files (PDFs are 20–25 KB each), the upload **block size** setting is irrelevant. Block size controls how a *single large file* is split for parallel upload. If the file is smaller than the block size, it uploads as one block — no padding, no waste.

Block size only matters when uploading multi-GB files.

## Why a separate resource group

The case study began with a Databricks Free Edition workspace, which auto-creates a managed resource group called `databricks-rg-simmons-demo-j7up5wwkemohe`. That RG has a system **deny assignment** preventing anyone (even owners) from adding resources there.

Trying to put the storage account in the locked RG produced:
```
The access is denied because of the deny assignment with name
'System deny assignment created by Azure Databricks ...'
```

So `kagglecvdataset` was created in the unrelated `simmons-demo` RG. Rule of thumb: any RG named `databricks-rg-*` is off-limits.

## Uploading the dataset

The Azure portal upload UI works for the 2,488 small PDFs but is slow and flaky. For larger uploads, prefer `azcopy`:

```
azcopy copy "./Resume/" "https://kagglecvdataset.blob.core.windows.net/kaggle-cv-dataset/" --recursive
```

Filter out macOS junk (`find . -name ".DS_Store" -delete`) before uploading.
