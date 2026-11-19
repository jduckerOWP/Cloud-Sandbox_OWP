# Storage & Data Management

This guide explains the Sandbox head node filesystem layout, Amazon EFS lifecycle rules, AWS S3 staging and output buckets, and accessing public environmental datasets via the **NOAA Open Data Dissemination (NODD)** program.

---

## 1. Head Node Filesystem Layout

When logged into a Sandbox head node, store your code, forcing data, and temporary run files in the designated directories rather than `/home`:

```text
+-----------------------------------------------------------------------------+
|                        SANDBOX HEAD NODE FILESYSTEM                         |
+------------------------------------+----------------------------------------+
| /home/<user> (Ephemeral Root)      | /save/<user> (EFS Persistent Work)     |
| - Root volume with limited space   | - Elastic File System (EFS/NFS)        |
| - DO NOT USE for modeling work     | - Store model code, builds & scripts   |
| - Consider temporary / ephemeral   | - Shell alias: cds                     |
+------------------------------------+----------------------------------------+
| /com/<user> (EFS Shared Data)      | /ptmp (Temporary Workflow Scratch)     |
| - Shared model forcing & outputs   | - Temporary EFS or FSx Lustre volume   |
| - Check shared datasets (e.g. ERA5)| - Active model execution scratch space |
| - Shell alias: cdc                 | - Copy final outputs to /com or S3     |
+------------------------------------+----------------------------------------+
```

| Directory | Storage Type | Shell Alias | Intended Purpose & Rules |
| :--- | :--- | :--- | :--- |
| **`/home/<user>`** | Local Root Volume | — | **Do not use for modeling work.** Limited disk space; treat as ephemeral. |
| **`/save/<user>`** | Amazon EFS | `cds` | Primary persistent workspace for cloning `Cloud-Sandbox`, compiling model executables, and storing run scripts. |
| **`/com/<user>`** | Amazon EFS | `cdc` | Shared directory for model forcing inputs and run outputs. Check existing shared forcing datasets (such as ERA5) before duplicating data, and name new datasets clearly. |
| **`/ptmp`** | EFS or FSx Lustre | — | High-speed temporary scratch space for active model workflows. Copy completed outputs to `/com` or S3 when finished. |

---

## 2. EFS Lifecycle Management & Storage Costs

To optimize cloud storage costs, Amazon EFS applies automated lifecycle policies to inactive files:

1. **Standard EFS (`$0.30/GB`):** Active files accessed within the last 33 days.
2. **EFS Infrequent Access / IA (`$0.01/GB`):** Files unused for **33 days** automatically transition to IA storage. Reading an IA file automatically promotes it back to Standard access with minimal friction.
3. **Glacier Archive Tier (`$0.0004/GB` instant retrieval tier):** Files unused for **100 days** automatically transition to Glacier storage.

> **Warning — Avoiding Glacier Retrieval Failures & Recurring Fees:**  
> * If a workflow tries to read input or forcing files that have not been accessed in over **100 days**, the run may fail or stall while retrieving data from Glacier.  
> * Temporary retrieval calls return files to Glacier afterward, incurring recurring retrieval charges if accessed repeatedly.  
> * Before running a workflow against older datasets, contact **sandbox.helpdesk@noaa.gov** (or attend Sandbox office hours) to migrate required files out of Glacier storage.

### Storage Tier Cost Comparison

| Storage Service / Tier | Approximate Cost | Best Use Case |
| :--- | :--- | :--- |
| **EFS Standard** | `$0.30 / GB` | Active code, builds, and current run directories |
| **EFS Infrequent Access (IA)** | `$0.01 / GB` | Warm datasets accessed every 1–3 months |
| **S3 Standard (First 50 TB)** | `$0.023 / GB` | Long-term storage, dataset sharing, and staging inputs/outputs |
| **S3 Standard (Next 450 TB)** | `$0.022 / GB` | Large multi-year hindcast/reanalysis archives |
| **S3 Standard (Over 500 TB)** | `$0.021 / GB` | Large-scale institutional archives |
| **Glacier Instant Retrieval** | `$0.0004 / GB` | Cold archival backups *(retrieval fees apply)* |

---

## 3. Staging & Exporting Data with AWS S3

Amazon S3 is the primary mechanism for transferring datasets into and out of the Sandbox (`us-east-2` region) and sharing completed model products.

### Coastal Sandbox S3 Buckets

```text
+------------------------------+         Manual AWS CLI sync/cp         +-----------------------+
|      s3://ioos-transfers     | <------------------------------------> |                       |
|  (General Staging Bucket)    |                                        |   Sandbox Head Node   |
+------------------------------+                                        |                       |
                                                                        |  /inputs              |
+------------------------------+         Mounted Read/Staging           |  /outputs_bucket      |
|  s3://ioos-coastalsb-inputs  | -------------------------------------> |  /save/<user>         |
|  (Input Staging Bucket)      |                                        |  /com/<user>          |
+------------------------------+                                        |                       |
                                                                        |                       |
+------------------------------+         Mounted Export Bucket          |                       |
|  s3://ioos-coastalsb-outputs | <------------------------------------- |                       |
|  (PUBLIC Outputs Bucket)     |                                        +-----------------------+
+------------------------------+
```

| S3 Bucket | Head Node Mount Point | Access & Purpose |
| :--- | :--- | :--- |
| **`s3://ioos-transfers`** | *Not mounted* (use `aws s3` CLI) | General staging bucket for transferring files between external systems and the Sandbox. |
| **`s3://ioos-coastalsb-inputs`** | `/inputs` | Staging bucket for incoming model datasets (e.g., UND inputs). |
| **`s3://ioos-coastalsb-outputs`** | `/outputs_bucket` | Outbound bucket for sharing and exporting Sandbox model results. |

> **Critical Rules for S3 Buckets & Mounts:**
> 1. **Never Run Models Directly Against S3 Mounts:** `/inputs` and `/outputs_bucket` are object-storage mounts meant strictly for transferring data in and out. Always copy input files to EFS (`/save` or `/com`) or `/ptmp` before executing a model.
> 2. **Public Visibility Warning:** Anything placed inside `/outputs_bucket` or uploaded to `s3://ioos-coastalsb-outputs` should be assumed **publicly accessible**. Do not place sensitive or restricted files there.
> 3. **Object Storage Behavior:** S3 objects cannot be appended or modified in place—they can only be overwritten.

### AWS CLI Authentication & Transfer Commands

Configure your AWS credentials or sign in via AWS Single Sign-On (SSO) using your configured profile in `~/.aws/config`:

```bash
aws sso login --profile ioos
```

#### Sync a Local Directory to S3 (`aws s3 sync`)
`aws s3 sync` synchronizes new or updated files between a source and destination in region `us-east-2`:

```bash
aws s3 sync /path/to/local/data s3://ioos-transfers --region us-east-2
```

#### Copy Specific Files or Directories (`aws s3 cp`)
```bash
aws s3 cp /path/to/local/file.nc s3://ioos-transfers/project_folder/ --region us-east-2
```

#### Bucket-to-Bucket Transfer
```bash
aws s3 sync s3://ioos-transfers s3://your-destination-bucket --region us-east-2
```

#### Exporting Results via `/outputs_bucket` on the Head Node
To publish completed run outputs to `s3://ioos-coastalsb-outputs`:

```bash
# Copy output files into a descriptive project subdirectory
cp -r /com/$USER/my_model_run/* /outputs_bucket/<ModelName>/<Experiment>/

# Check total disk usage staged in the outputs bucket
du -sh /outputs_bucket/<ModelName>/<Experiment>/
```

To download files from the public outputs bucket to a local machine:

```bash
aws s3 sync s3://ioos-coastalsb-outputs/<ModelName>/<Experiment>/ ./local_outputs/ --region us-east-2
```

---

## 4. Accessing & Publishing Data via NODD

The **NOAA Open Data Dissemination (NODD)** program hosts public NOAA environmental and operational model datasets directly on commercial cloud object storage, eliminating egress bottlenecks.

### Reading Public NODD Datasets in Python
You can query NODD S3 buckets anonymously without AWS credentials using `intake` or `s3fs`:

```python
# Option 1: Open an Intake catalog if a YAML catalog is available
import intake

catalog = intake.open_catalog("s3://<bucket-name>/<catalog>.yml", storage_options={"anon": True})
print(list(catalog))

# Option 2: Browse top-level directories directly with s3fs
import s3fs

fs = s3fs.S3FileSystem(anon=True)
bucket_files = fs.ls("noaa-gestofs-pds")
print("Top-level directories:", bucket_files[:10])
```

### Exporting Datasets to NODD
If your project produces community-value coastal model datasets that you would like hosted publicly on NODD, submit a hosting request to **sandbox.helpdesk@noaa.gov**. The Sandbox team will coordinate with the NODD program to onboard your dataset.

---

## Next Steps

* **Running & Porting Models:** Proceed to [Model Porting & Workflow Architecture](./MODEL_PORTING_AND_WORKFLOW.md) to configure model inputs, templates, and Cloudflow jobs.
* **Best Practices & Instance Limits:** Review [Glossary & Best Practices](./GLOSSARY_AND_BEST_PRACTICES.md) for storage cleanup habits and approved EC2 instance types.



