# IOOS Coastal Modeling Cloud Sandbox — User Documentation

Welcome to the user documentation for the **NOS Coastal Modeling Cloud Sandbox** (`ioos/Cloud-Sandbox`). The Sandbox is a collaborative cloud-based High-Performance Computing (HPC) environment for NOAA modelers, IOOS Regional Associations, and academic partners to develop, test, validate, and transition coastal and ocean models from research to operations (R2O).

> **Need Help or Account Access?**  
> Contact the Sandbox Helpdesk at **sandbox.helpdesk@noaa.gov**.

---

## User Guides (`documentation/`)

New to the Sandbox? Start with **Getting Started & Access**, then explore the guides below based on your workflow:

| Guide | Description |
| :--- | :--- |
| **[Getting Started & Access](./GETTING_STARTED_AND_ACCESS.md)** | Account onboarding, IP allowlisting, logging into **Nebari** via Keycloak (NOAA ICAM & NOAA Okta), and connecting to head nodes via **Okta Privileged Access (OPA)** (`sft`). |
| **[Nebari User Guide](./NEBARI_USER_GUIDE.md)** | Instance sizing, **JupyterLab**, web **VSCode**, **Marimo** notebooks, managing shared environments with **Conda Store**, GPU/CUDA configuration, and **Dask** workflows. |
| **[Storage & Data Management](./STORAGE_AND_DATA_MANAGEMENT.md)** | Head node directory rules (`/home`, `/save`, `/com`, `/ptmp`), EFS lifecycle tiers (32-day IA / 100-day Glacier), S3 staging & output buckets, and **NODD** data access. |
| **[Model Porting & Workflow Architecture](./MODEL_PORTING_AND_WORKFLOW.md)** | Porting models from Slurm/WCOSS to **Cloudflow**, tracing model entry points (`fcstrun.sh` / `mpiexec`), namelist templating, and Prefect v3 execution flows. |
| **[Glossary & Best Practices](./GLOSSARY_AND_BEST_PRACTICES.md)** | Sandbox Do's and Don'ts (including safe job termination), approved EFA-enabled EC2 instance types, scaling tips, and a cloud/HPC terminology cheat sheet. |

---

## Core Technical & Operational Guides (Repository Root)

For system administrators, workflow developers, and engineers executing or deploying Cloudflow jobs directly on the head node, refer to the core technical guides in the repository:

* **Workflow Orchestration & Prefect**
  * [Cloudflow System Overview](../CLOUDFLOW.md) — Architectural overview of the Cloudflow Python framework.
  * [Prefect v3 Setup & Troubleshooting Guide](../PREFECTv3.md) — Running the local Prefect v3 server, SSH tunneling to the dashboard UI (`http://localhost:4200/dashboard`), and managing flow runs.
  * [Zombie Job Checklist](../cloudflow/ZOMBIE_JOB_CHECKLIST.md) — Operational protocols, `I_MPI` flags, `mpi4py` deadlock hooks, and Dask fast-fail settings to prevent hung EC2 instances.
* **Model Execution & Integration**
  * [Cloudflow Model Execution Instructions](../cloudflow/CLOUDFLOW_MODEL_EXECUTION_INSTRUCTIONS.md) — Step-by-step instructions for staging and running supported ocean models on the cloud (e.g., SCHISM, ROMS, FVCOM, ADCIRC).
  * [Python Model Execution Instructions](../cloudflow/CLOUDFLOW_PYTHON_MODEL_EXECUTION_INSTRUCTIONS.md) — Running serial, MPI, and Dask Python workflows on the cloud.
  * [Model Integration Template](../CLOUD_SANDBOX_MODEL_INTEGRATION_TEMPLATE.md) — Step by step instructions for integrating a new model into Cloudflow.
* **Environment & Infrastructure**
  * [Local Python Miniforge3 Installation](../LOCAL_PYTHON_MINIFORGE3_INSTALLATION_CLOUD_SANDBOX_INSTRUCTIONS.md) — Setting up an isolated Miniforge3 environment under `/save/$USER` on the head node and constructing a Cloudflow compatible Python environment.
  * [Infrastructure Deployment Guide](../DEPLOYMENT.md) — Provisioning and hosting a Cloud Sandbox environment on AWS using Terraform.

---

## External Resources

* **Nebari Web Portal:** [https://coastalsandbox.nos.noaa.gov](https://coastalsandbox.nos.noaa.gov) *(requires NOAA network/VPN or allowlisted IP)*
* **IOOS Cloud Sandbox Website:** [https://coastalsandbox.ioos.us](https://coastalsandbox.ioos.us)
* **Nebari Official Tutorials:** [https://nebari.dev/classic/tutorials/](https://nebari.dev/classic/tutorials/)
* **Marimo Documentation:** [https://docs.marimo.io/](https://docs.marimo.io/)
