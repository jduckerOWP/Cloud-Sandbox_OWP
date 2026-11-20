# Glossary & Best Practices

This reference guide compiles operational Do's and Don'ts, cost-management rules, approved AWS EC2 instance types, and a quick-reference glossary of cloud and High-Performance Computing (HPC) terms used across the **NOS Coastal Modeling Cloud Sandbox**.

---

## 1. Sandbox Operational Do's and Don'ts

### Compute & Cluster Management
* **DO** benchmark your model across 1, 2, and 4 nodes to find the optimal scaling point before launching multi-day hindcasts or large production runs.
* **DO** stop your Nebari server (**Stop My Server** on the Nebari Home page) and shut down any active Dask clusters (`cluster.shutdown()`) when finished with interactive sessions.
* **DO** select the smallest Nebari instance size appropriate for your task (e.g., use **General Purpose Small** if you only need a terminal to SSH into a head node).
* **DO** tag your cluster configuration files (`cluster.configs/*.ioos`) with clear `Name` and `Project` keys so administrators can track active runs.
* **DO** Execute a `nohup` command when submitting a cloudflow job (`nohup ./worflows/workflow_main.py ../cluster.configs/Experiments/schism.ioos ../job.configs/MODEL_EXPERIMENTS/schism.experiment > cloudflow_test.out &`). This will eliminate the risk of a zombie job incurring if you get kicked off the server or yourself exiting out of the job, which is further described in the next bullet point.
* **DON'T** press `Ctrl+C` or `kill` the Python `workflow_main.py` driver process during an active Cloudflow run. Doing so skips `AWSCluster.terminate()` and leaves expensive EC2 worker instances running ("zombie" nodes). Instead, terminate your `mpiexec` or `fcstrun.sh` process so Cloudflow catches the exit code and tears down the cluster cleanly.
* **DON'T** launch large multi-node production runs without notifying **sandbox.helpdesk@noaa.gov** in advance so the team can verify AWS EC2 capacity reservations.

### Storage & Data Hygiene
* **DO** store code, build scripts, and executables in `/save/$USER` (`cds`), forcing and shared model outputs in `/com/$USER` (`cdc`), and active run scratch files in `/ptmp`.
* **DO** check `/com` for existing shared forcing datasets (such as ERA5) before downloading duplicate copies, and use clear, descriptive folder names for new datasets.
* **DO** monitor files older than **33 days** (EFS Infrequent Access tier) and **100 days** (Glacier archive tier). Contact **sandbox.helpdesk@noaa.gov** before running workflows against files older than 100 days to avoid Glacier retrieval delays and recurring retrieval charges.
* **DON'T** store modeling code, datasets, or outputs in `/home/$USER`—it resides on a small root volume with limited disk capacity.
* **DON'T** execute models directly against S3 mount points (`/inputs` or `/outputs_bucket`). Always copy inputs to EFS (`/save` or `/com`) or `/ptmp` before running a model.
* **DON'T** upload sensitive, proprietary, or restricted files to `/outputs_bucket` or any of the `s3://ioos-*` buckets because files in the outputs bucket are publicly accessible.

---

## 2. Supported AWS EC2 Compute Instance Types

Cloudflow supports the EC2 instance types registered in `cloudflow/cluster/AWSCluster.py` (`aws_types{}`). Specify the exact `nodeType` string in your `cluster.configs/*.ioos` file.

> **Important Note on EFA & Core Counts (`PPN`):**
> * For tightly coupled multi-node MPI ocean models, choose **HPC-optimized** (`hpc6a`, `hpc6id`, `hpc7a`, `hpc8a`) or **EFA-enabled** network-optimized (`c5n`, `c6i.12xlarge`, `c7i.48xlarge`, `r7iz.32xlarge`) instances.
> * Most entries map `PPN` to **physical CPU cores** (hyperthreading disabled for MPI performance). The `x2idn` and `r7i` entries at the bottom are mapped to **vCPUs** in `AWSCluster.py`.

### HPC-Optimized Instances (Recommended for Multi-Node MPI Models)

| Instance Type (`nodeType`) | Architecture / Processor | Cores (`PPN`) | RAM | Network / EFA | Typical Use Case |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **`hpc6a.48xlarge`** | AMD EPYC (3rd Gen) | 96 | 384 GB | 100 Gbps EFA | High-throughput, cost-effective MPI ocean modeling (ROMS, FVCOM, SCHISM, ADCIRC) |
| **`hpc6id.32xlarge`** | Intel Xeon Scalable (3rd Gen) | 64 | 1024 GB | 200 Gbps EFA + NVMe | Memory-bound workloads requiring 1 TB RAM per node or fast local NVMe scratch |
| **`hpc7a.48xlarge`** | AMD EPYC (4th Gen) | 96 | 768 GB | 300 Gbps EFA | High-bandwidth, higher-memory 4th Gen AMD runs |
| **`hpc7a.96xlarge`** | AMD EPYC (4th Gen) | 192 | 768 GB | 300 Gbps EFA | Large-domain, high-resolution coupled models requiring maximum core density |
| **`hpc8a.96xlarge`** | AMD EPYC (5th Gen) | 192 | 768 GB | 300 Gbps EFA | Next-generation high-density, compute-intensive multi-node MPI simulations |

### Compute-Optimized Instances (`c5`, `c5a`, `c5n`, `c6i`, `c7i`)

| Instance Type (`nodeType`) | Architecture / Processor | Cores (`PPN`) | RAM | Network / EFA | Typical Use Case |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **`c5.large`** | Intel Xeon Scalable (1st/2nd Gen) | 1 | 4 GB | Up to 10 Gbps | Lightweight single-core testing and workflow debugging |
| **`c5.xlarge`** | Intel Xeon Scalable (1st/2nd Gen) | 2 | 8 GB | Up to 10 Gbps | Small pre/post-processing scripts and single-node checks |
| **`c5.2xlarge`** | Intel Xeon Scalable (1st/2nd Gen) | 4 | 16 GB | Up to 10 Gbps | Small test domains and compilation/utility tasks |
| **`c5.4xlarge`** | Intel Xeon Scalable (1st/2nd Gen) | 8 | 32 GB | Up to 10 Gbps | Single-node development runs |
| **`c5.9xlarge`** | Intel Xeon Scalable (1st/2nd Gen) | 18 | 72 GB | 10 Gbps | Single-node mid-sized test runs |
| **`c5.12xlarge`** | Intel Xeon Scalable (2nd Gen) | 24 | 96 GB | 12 Gbps | Single-node compute runs without EFA requirement |
| **`c5.18xlarge`** | Intel Xeon Scalable (1st/2nd Gen) | 36 | 144 GB | 25 Gbps | Single-node full-socket Intel compute runs |
| **`c5.24xlarge`** | Intel Xeon Scalable (2nd Gen) | 48 | 192 GB | 25 Gbps | Large single-node Intel compute runs |
| **`c5.metal`** | Intel Xeon Scalable (2nd Gen) | 36 | 192 GB | 25 Gbps | Bare-metal workloads requiring direct hardware access |
| **`c5a.large`** | AMD EPYC (2nd Gen) | 1 | 4 GB | Up to 10 Gbps | Cost-optimized lightweight single-core testing |
| **`c5a.xlarge`** | AMD EPYC (2nd Gen) | 2 | 8 GB | Up to 10 Gbps | Cost-optimized small pre/post-processing tasks |
| **`c5a.2xlarge`** | AMD EPYC (2nd Gen) | 4 | 16 GB | Up to 10 Gbps | Cost-optimized small test domains |
| **`c5a.4xlarge`** | AMD EPYC (2nd Gen) | 8 | 32 GB | Up to 10 Gbps | Cost-optimized single-node development runs |
| **`c5a.24xlarge`** | AMD EPYC (2nd Gen) | 48 | 192 GB | 20 Gbps | Cost-optimized large single-node AMD compute runs |
| **`c5n.large`** | Intel Xeon Scalable (Cascade Lake) | 1 | 5.25 GB | Up to 25 Gbps | Network-intensive small tasks and S3 data transfers |
| **`c5n.xlarge`** | Intel Xeon Scalable (Cascade Lake) | 2 | 10.5 GB | Up to 25 Gbps | Network-intensive small preprocessing tasks |
| **`c5n.2xlarge`** | Intel Xeon Scalable (Cascade Lake) | 4 | 21 GB | Up to 25 Gbps | Network-intensive small domain tests |
| **`c5n.4xlarge`** | Intel Xeon Scalable (Cascade Lake) | 8 | 42 GB | Up to 25 Gbps | Network-intensive single-node runs |
| **`c5n.9xlarge`** | Intel Xeon Scalable (Cascade Lake) | 18 | 96 GB | 50 Gbps EFA | Smaller-scale EFA-enabled MPI benchmarks |
| **`c5n.18xlarge`** | Intel Xeon Scalable (Cascade Lake) | 36 | 192 GB | 100 Gbps EFA | Legacy EFA workflows and smaller-scale multi-node MPI runs |
| **`c5n.24xlarge`** | Intel Xeon Scalable (Cascade Lake) | 48 | 192 GB | 100 Gbps EFA | Legacy high-core-count Intel EFA workflows |
| **`c5n.metal`** | Intel Xeon Scalable (Cascade Lake) | 36 | 192 GB | 100 Gbps EFA | Bare-metal EFA workloads requiring direct hardware access |
| **`c6i.12xlarge`** | Intel Xeon Scalable (3rd Gen) | 24 | 96 GB | 18.75 Gbps (EFA) | Mid-sized 3rd Gen Intel compute and EFA testing |
| **`c7i.12xlarge`** | Intel Xeon Scalable (4th Gen) | 24 | 96 GB | 18.75 Gbps | Mid-sized 4th Gen Intel single-node compute runs |
| **`c7i.48xlarge`** | Intel Xeon Scalable (4th Gen) | 96 | 192 GB | 50 Gbps EFA | High-core-count 4th Gen Intel EFA-enabled MPI runs |

### Memory-Optimized Instances (`r7iz`, `r7i`, `x2idn`)

| Instance Type (`nodeType`) | Architecture / Processor | Cores / `PPN` | RAM | Network / EFA | Typical Use Case |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **`r7iz.32xlarge`** | Intel Xeon Scalable (4th Gen High-Freq) | 64 *(cores)* | 1024 GB | 50 Gbps EFA | High-frequency, 1 TB memory-intensive EFA MPI simulations |
| **`r7i.xlarge`** | Intel Xeon Scalable (4th Gen) | 2 *(vCPUs)* | 32 GB | Up to 12.5 Gbps | Lightweight high-memory preprocessing and post-processing |
| **`r7i.8xlarge`** | Intel Xeon Scalable (4th Gen) | 16 *(vCPUs)* | 256 GB | 12.5 Gbps | Memory-heavy post-processing, regridding, and single-node runs |
| **`x2idn.24xlarge`** | Intel Xeon Scalable (3rd Gen) | 96 *(vCPUs)* | 1536 GB | 75 Gbps | Ultra-high-memory (1.5 TB RAM) grid generation or analysis |
| **`x2idn.32xlarge`** | Intel Xeon Scalable (3rd Gen) | 128 *(vCPUs)* | 2048 GB | 100 Gbps EFA | Ultra-high-memory (2 TB RAM) large-domain simulations and processing |
---

## 3. Cloud & HPC Terminology Glossary

| Term / Acronym | Definition |
| :--- | :--- |
| **AMI (Amazon Machine Image)** | A pre-configured virtual machine image containing the OS, Spack compilers, MPI libraries, and drivers used to boot Sandbox EC2 compute nodes. |
| **Cloudflow** | The open-source Python workflow orchestration framework developed for the Sandbox to provision on-demand AWS HPC clusters, configure model namelists, execute MPI jobs, and terminate compute nodes automatically. |
| **Conda Store** | The environment management service integrated into Nebari that builds, versions, and shares reproducible Conda Python environments across JupyterLab and web VSCode. |
| **Dask / Dask Gateway** | A parallel computing library and Kubernetes service in Nebari that dynamically spawns distributed worker pods for out-of-core xarray, NetCDF, and DataFrame post-processing. |
| **EC2 (Elastic Compute Cloud)** | On-demand virtual servers in AWS used both for persistent Sandbox head nodes and ephemeral multi-node MPI compute clusters. |
| **EFA (Elastic Fabric Adapter)** | A specialized AWS network interface for EC2 instances that enables low-latency, OS-bypass MPI communication between compute nodes—essential for scaling tightly coupled ocean models. |
| **EFS (Elastic File System)** | Shared network-attached storage (NFS) mounted across head nodes and compute nodes at `/save` and `/com`, with automated 32-day Infrequent Access and 100-day Glacier lifecycle policies. |
| **FSx for Lustre** | High-performance parallel file system optionally mounted at `/ptmp` for I/O-intensive model execution scratch space. |
| **Head Node** | A persistent Linux EC2 login server accessed via Okta Privileged Access (`sft ssh`) where users compile models, manage files in `/save` and `/com`, and launch Cloudflow jobs. |
| **ICAM** | NOAA's Identity, Credential, and Access Management system used by internal `@noaa.gov` staff (with CAC authentication) to log into Nebari. |
| **Keycloak** | The open-source identity broker front-ending Nebari that routes authentication to either **NOAA ICAM** or **NOAA Okta**. |
| **Marimo** | A reactive Python notebook environment pre-installed in the Sandbox Nebari image alongside JupyterLab and web VSCode. |
| **MPI (Message Passing Interface)** | The parallel communication standard (`mpirun` / `mpiexec`) used by compiled Fortran/C ocean models to exchange grid boundary data across CPU cores and EC2 nodes. |
| **Nebari** | The open-source, Kubernetes-hosted data science platform providing web access to JupyterLab, VSCode, Marimo, Conda Store, and Dask Gateway at `https://coastalsandbox.nos.noaa.gov`. |
| **NODD** | **NOAA Open Data Dissemination**—a program that hosts public NOAA operational and environmental datasets on cloud object storage (such as AWS S3) for direct, credential-free access. |
| **OPA (Okta Privileged Access)** | Zero-trust SSH access management (`sft` CLI) used to authenticate and connect from a terminal to Sandbox head nodes without managing static SSH private keys. |
| **Placement Group** | An AWS EC2 configuration (`ioos_cloud_sandbox_Terraform_Placement_Group`) that places compute nodes physically close together inside an Availability Zone to minimize network latency. |
| **Prefect v3** | The Python workflow orchestration engine underlying Cloudflow that tracks task dependencies, retries, logs, and guaranteed cluster cleanup (`AWSCluster.terminate()`). |
| **R2O (Research to Operations)** | The process of transitioning experimental coastal models and workflows into validated, reliable operational forecasting systems. |
| **S3 (Simple Storage Service)** | Scalable AWS object storage used for staging inputs (`s3://ioos-coastalsb-inputs`), transferring files (`s3://ioos-transfers`), and sharing public outputs (`s3://ioos-coastalsb-outputs`). |
| **Spack** | The HPC package manager used on Sandbox head nodes and compute AMIs to build and load versioned compilers and libraries via `module load`. |
| **Zombie Nodes** | Orphaned EC2 compute instances left running after an improperly aborted workflow (e.g., killing `workflow_main.py` with `Ctrl+C`), resulting in unnecessary cloud charges. |

---

## 4. Quick Support & Contact Reference

* **Sandbox Helpdesk & Onboarding:** `sandbox.helpdesk@noaa.gov`
* **Nebari Web Portal:** `https://coastalsandbox.nos.noaa.gov`
* **IOOS Cloud Sandbox Portal:** `https://coastalsandbox.ioos.us`
* **Repository Issues & Contributions:** `https://github.com/ioos/Cloud-Sandbox`
