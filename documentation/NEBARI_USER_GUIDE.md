# Nebari User Guide

**Nebari** is the web-based data science platform for the NOS Coastal Modeling Cloud Sandbox. It integrates **JupyterLab**, web **VSCode**, **Marimo** notebooks, **Conda Store**, and **Dask Gateway** inside a managed Kubernetes environment for model post-processing, Python-based data analysis, and AI/ML workflows.

* **Nebari Portal URL:** [https://coastalsandbox.nos.noaa.gov](https://coastalsandbox.nos.noaa.gov) *(requires NOAA network/VPN or an allowlisted IP address)*
* **Login Instructions:** See [Getting Started & Access](./GETTING_STARTED_AND_ACCESS.md) for Keycloak authentication via **NOAA ICAM** or **NOAA Okta**.

---

## 1. Instance Type Selection & Server Lifecycle

After authenticating to Nebari, select a server profile matched to your workload:

| Server Profile | vCPUs / RAM | GPU Hardware | Best Use Case |
| :--- | :--- | :--- | :--- |
| **General Purpose Small** | 2 vCPU / 8 GB RAM | None | Jumping to a Sandbox head node via Okta OPA (`sft ssh`) or lightweight file browsing |
| **General Purpose Medium** | 4 vCPU / 16 GB RAM | None | Standard Python scripting, plotting, and interactive notebook analysis |
| **General Purpose 2XLarge** | 8 vCPU / 32 GB RAM | None | Memory-intensive single-node analysis |
| **T4 GPU XLarge** *(requires `gpu` group)* | 4 vCPU / 16 GB RAM | 1× NVIDIA T4 (16 GB VRAM) | Single-GPU ML model training and inference |
| **T4 GPU 4XLarge** *(requires `gpu` group)* | 16 vCPU / 64 GB RAM | 1× NVIDIA T4 (16 GB VRAM) | GPU workloads requiring higher host CPU/RAM |
| **2 T4 GPU 4XLarge** *(requires `multi-gpu` group)* | 16 vCPU / 64 GB RAM | 2× NVIDIA T4 GPUs | Multi-GPU distributed training |
| **4 T4 GPU 12XLarge** *(requires `multi-gpu` group)* | 48 vCPU / 192 GB RAM | 4× NVIDIA T4 GPUs | Large-scale multi-GPU deep learning |

### Resource & Cost Best Practices
* **Choose the Smallest Appropriate Instance:** If you only need a terminal to SSH into a head node, pick **General Purpose Small**.
* **Offload Heavy Parallelism to Dask:** You do not need a large primary JupyterLab server for big-data processing; **Dask clusters** provision separate worker nodes dynamically.
* **Stop Idle Servers:** Return to the Nebari Home page and click **Stop My Server** when finished with a session so cloud compute resources are released.

---

## 2. Nebari Apps & User Interface

From the Nebari Home page top navigation bar, you can access the bundled services:

* **JupyterHub (`My Server`):** Launches your personal JupyterLab container on your chosen server instance.
* **Conda Store (`Environment Management`):** Build, version-control, and share Python Conda environments across users and groups (`https://coastalsandbox.nos.noaa.gov/conda-store`).
* **Argo Workflows (`Argo Workflows`):** Workflow automation UI (`https://coastalsandbox.nos.noaa.gov/argo/`). *(Note: Available by default in Nebari, but not currently operationally supported in the Sandbox.)*
* **Grafana (`Monitoring`):** Cluster monitoring dashboards (`https://coastalsandbox.nos.noaa.gov/monitoring`). *(Note: Available by default in Nebari, but not currently operationally supported in the Sandbox.)*

---

## 3. Using JupyterLab & Shared Directories

JupyterLab is the primary interactive workspace in Nebari. Your container comes pre-configured with standard CLI tools (`git`, `ssh`, `zip`, `sft` for Okta OPA) and built-in JupyterLab extensions (Dask dashboard, GPU profiler, and Git UI).

### Persistent Storage & Shared Group Directories
* **Home Directory (`/home/<username>`):** Persists across server restarts and sessions.
* **Shared Group Directories (`/shared/`):** Persistent directories shared with members of your assigned Nebari groups. Most users belong to the `analyst` group by default:
  * `/shared/analyst`
  * `/shared/ioos-dmac`

---

## 4. Using Web VSCode

If you prefer a full IDE experience over Jupyter notebooks, launch **VSCode** directly from the JupyterLab Launcher screen by clicking the **VS Code** tile:

* **Persistent Extensions:** Extensions installed in web VSCode are saved in your persistent home directory across sessions.
* **Conda Store Integration:** You can select any Conda Store environment as your Python interpreter or Jupyter notebook kernel inside web VSCode.

---

## 5. Managing Conda Environments with Conda Store

**Conda Store** manages reproducible Python environments across the Sandbox:

1. Open **Environment Management** from the Nebari menu bar (or visit `/conda-store`).
2. Select your personal namespace (`<your.email>`) or a shared group namespace (`analyst`, `global`, `nebari-git`) and click **+ New**.
3. Add packages manually via the UI or paste an `environment.yaml` specification.
4. Once the build status shows **Completed / Active**, the environment appears automatically as a selectable kernel in both JupyterLab and web VSCode.

For detailed UI walkthroughs, see the [Official Conda Store Documentation](https://conda.store/conda-store-ui/tutorials/).

---

## 6. Using GPUs & CUDA in Nebari

To run GPU-accelerated workflows (PyTorch, TensorFlow, CUDA), request membership in the **`gpu`** (1× T4) or **`multi-gpu`** (2×–4× T4) group via **sandbox.helpdesk@noaa.gov**.

### Verifying the Supported CUDA Driver Version
Start a GPU server instance, open a terminal, and check the host NVIDIA driver's maximum supported CUDA version:

```bash
nvidia-smi
```

Look for the `CUDA Version:` field in the top-right header of the output (currently **CUDA 12.8** in Nebari).

### Example CUDA-Enabled `environment.yaml` for Conda Store
When building a GPU environment in Conda Store, pin `cuda-toolkit` and `CONDA_OVERRIDE_CUDA` to match the version reported by `nvidia-smi`:

```yaml
channels:
  - conda-forge
dependencies:
  - pandas
  - boto3
  - numpy
  - tensorflow
  - sagemaker
  - ipykernel
  - protobuf=4.25.3
  - fastparquet
  - pyarrow
  - pytorch
  - cuda-toolkit=12.8.1
variables:
  CONDA_OVERRIDE_CUDA: "12.8"
```

*(See also: [Nebari Official GPU Guide](https://classic.nebari.dev/how-tos/use-gpus/).)*

---

## 7. Using Dask in Nebari

Nebari includes **Dask Gateway** to scale Python post-processing and out-of-core xarray/netCDF model analysis across distributed Kubernetes worker pods without resizing your primary JupyterLab server.

### Connecting to a Dask Cluster
In a notebook or Python script using an environment with `dask-gateway` installed, spin up an on-demand Dask cluster:

```python
from dask_gateway import Gateway

gateway = Gateway()

# View available cluster options (worker cores, memory, environment)
options = gateway.cluster_options()
display(options)  # In JupyterLab, displays an interactive configuration widget

# Launch a cluster and connect a Dask client
cluster = gateway.new_cluster(options)
cluster.scale(4)  # Scale to 4 Dask workers
client = cluster.get_client()
client
```
### Monitoring & Cleanup
* **Dask Dashboard:** Open the Dask extension tab in the left sidebar of JupyterLab (or click the dashboard link printed by `client`) to monitor task streams, worker memory, and progress.
* **Shut Down When Done:** Always close your cluster when finished so worker pods are released:

```python
client.close()
cluster.shutdown()
```

*(For additional examples, see the [Nebari Dask Gateway Tutorial](https://nebari.dev/classic/tutorials/).)*

---

## 8. Using Marimo Notebooks in Nebari

The Sandbox Nebari image comes pre-configured with **Marimo** (a reactive Python notebook environment).

* Launch Marimo directly from the JupyterLab Launcher—**no manual installation is required**, as the JupyterHub extension is already built into the base Sandbox image.
* Refer to the [Marimo Documentation](https://docs.marimo.io/) and [Marimo JupyterHub Guide](https://docs.marimo.io/guides/deploying/jupyterhub/) for usage details.

---

## Next Steps

* **Managing Storage & S3 Transfers:** Proceed to [Storage & Data Management](./STORAGE_AND_DATA_MANAGEMENT.md) to learn how to stage inputs/outputs between S3, NODD, and the Sandbox.
* **Running Models on Head Nodes:** See [Getting Started & Access](./GETTING_STARTED_AND_ACCESS.md) to connect to a head node via Okta OPA (`sft ssh`).

