# Model Porting & Workflow Architecture

This guide explains how **Cloudflow** orchestrates coastal ocean models on AWS and provides a practical walkthrough for porting models from traditional on-premises HPC schedulers (such as Slurm, LSF, or NOAA's WCOSS) into the Cloud Sandbox.

* **Running an Existing Model:** See [Cloudflow Model Execution Instructions](./CLOUDFLOW_MODEL_EXECUTION_INSTRUCTIONS.md) or [Python Model Execution Instructions](./CLOUDFLOW_PYTHON_MODEL_EXECUTION_INSTRUCTIONS.md).
* **New Model Boilerplate:** See [Model Integration Template](./CLOUD_SANDBOX_MODEL_INTEGRATION_TEMPLATE.md).
* **Prefect v3 Server Setup:** See [Prefect v3 Setup & Troubleshooting Guide](./PREFECTv3.md).

---

## 1. Cloud HPC vs. Traditional On-Premises Schedulers

Traditional HPC centers rely on shared static clusters managed by workload schedulers (`Slurm`, `PBS`, `LSF`, or WCOSS `J`-jobs) where jobs wait in a queue for allocated nodes.

**The Cloud Sandbox does not use Slurm or a shared job queue.** Instead, Cloudflow uses **Prefect v3** and `boto3` to provision a dedicated, ephemeral AWS EC2 compute cluster with Elastic Fabric Adapter (EFA) networking on demand for each workflow, executes the model directly via `mpirun` or `mpiexec`, and destroys the compute nodes immediately upon completion.

| Concept | Traditional On-Premises HPC | IOOS Cloud Sandbox (Cloudflow) |
| :--- | :--- | :--- |
| **Compute Provisioning** | Shared pre-built cluster; wait in batch queue | On-demand EC2 + EFA cluster spun up per workflow |
| **Job Submission** | `sbatch` / `qsub` / `bsub` wrapper scripts | Python driver (`workflow_main.py`) + JSON configs |
| **Parallel Launch** | `srun` / scheduler-injected MPI ranks | Direct `mpirun` / `mpiexec` with explicit `-hosts`, `-np`, and `-ppn` |
| **Resource Sizing** | Requested inside batch script directives (`#SBATCH`) | Defined in `cluster.configs/*.ioos` (`nodeType`, `nodeCount`) |
| **Cluster Teardown** | Nodes return to shared queue automatically | `AWSCluster.terminate()` destroys EC2 instances at flow end |

---

## 2. Cloudflow Directory & Code Architecture

Cloudflow is structured as an object-oriented Python package inside `cloudflow/`:

```text
cloudflow/
├── cluster/     # Abstract Cluster base class & AWSCluster.py (EC2/EFA provisioning)
├── job/         # Abstract Job base class, JobFactory.py, & model-specific Job classes
│   └── templates/ # Model input namelist templates (e.g., ocean.in, fort.15)
├── utils/       # Helpers: getTiling(), ndate(), sedoceanin() template parser
├── workflows/   # Prefect v3 drivers (workflow_main.py), flows.py, tasks.py, cluster_tasks.py
└── scripts/     # Supporting BASH launcher and utility scripts
```

*(Reference Diagram: See `cloudflow/docs/UML_ModelWorkflows.png` for the class inheritance and task dependency layout.)*

### Implemented `Job` Classes (`cloudflow/job/`)
Each job configuration file specifies a `"JOBTYPE"` string, which `JobFactory.py` maps to a Python `Job` subclass:

* **`ADCIRC_Experiment.py`** — ADCIRC basic model run
* **`DFLOWFM_Experiment.py`** — DFlowFM basic model run
* **`FVCOM_Experiment.py`** — FVCOM basic model run
* **`PYTHON_Experiment.py`** — Python basic application run
* **`ROMS_Experiment.py`** — ROMS basic model run
* **`SCHISM_Experiment.py`** — SCHISM basic model run
* **`WRF_Hydro_Experiment.py`** — WRF Hydro basic model run
* **`Model_Experiment_Template.py`** — Model template for integrating new model class
* **`ROMSForecast.py`** — NOSOFS ROMS and LiveOcean single-day forecasts
* **`ROMSHindcast.py`** — LiveOcean multi-day hindcasts
* **`FVCOMForecast.py`** — NOSOFS FVCOM forecasts
* **`ADCIRCForecast.py`** — ADCIRC hurricane test cases
* **`ADCIRCReanalysis.py`** — CORA ADCIRC reanalysis and UND/OCS runs
* **`UFS_RegressionTest.py`** — UFS-Coastal application runs
* **`Plotting.py`** — Post-processing and Dask-based visualization workflows
* **`*_Experiment.py` / `Model_Experiment_Template.py`** — Basic model experiment implementations and starter template

*(Note: When adding a new `Job` subclass, register its `"JOBTYPE"` mapping inside `cloudflow/job/JobFactory.py`.)*

---

## 3. End-to-End 5-Stage Prefect v3 Execution Trace

Moving forward from the Cloudflow entry point (`workflow_main.py`), every standard forecast workflow executes five Prefect tasks:

```text
  [1. cluster_init()]     -->        [2. job_init()]                -->                 [3. AWSCluster.start()]
         |                                |                                                     |
  Reads *.ioos JSON                     Parses job JSON &                                   Launches EC2 + EFA
  & validates EFA                       renders templates                                   nodes via boto3
                                                                                                |
                                                                                                v
  [6. AWSCluster.terminate()] <-- [5. Run Model (jobs/schism_basic_run.sh)]     <-- [4. Launcher (experiment_launcher.sh)]
         |                                  |                                                   |
  Destroys EC2 nodes              Executes mpirun / mpiexec                                Prepares Bash environment
  (runs even on failure)          across cluster host IPs                                  & MPI linkage to host IPs
```

### Stage 1: Initialize Cluster Object (`cluster_tasks.py -> cluster_init()`)
* Uses `ClusterFactory.py` to instantiate `AWSCluster.py` from your cluster configuration JSON (e.g., `cluster.configs/template.ioos`).
* Looks up `nodeType` in `AWSCluster.py` (`aws_types{}`) to verify EFA support and calculate processors per node (`PPN`) and total cores (`NPROCS`).

### Stage 2: Initialize Job & Render Templates (`tasks.py -> job_init()`)
* Uses `JobFactory.py` to instantiate the target `Job` class based on `"JOBTYPE"`.
* Calls `self.parseConfig()` to load run dates, executable paths, and forcing/output directories.
* Calls `self.make_config()` to populate model input templates via `sedoceanin(template, outfile, settings)`.

### Stage 3: Start On-Demand Cluster (`AWSCluster.py -> start()`)
* Uses `boto3` to launch the requested EC2 instances inside the Sandbox placement group and security groups, returning private host IP addresses for MPI.
* **Capacity Reservations:** If you plan a large multi-node run, coordinate with **sandbox.helpdesk@noaa.gov** in advance so the team can request an AWS Capacity Reservation.

### Stage 4: Prepare Launcher Environment (`workflows/experiment_launcher.sh`)
* Prepares the Bash runtime environment on the cluster, loads required Spack compiler/MPI modules, and links the dynamic EC2 host IP list (`MPIOPTS`, `NPROCS`, `PPN`) for MPI communication.

### Stage 5: Execute Model Job (`jobs/schism_basic_run.sh`)
* Bridges the Cloudflow launcher to the model's native run script and executes `mpirun` or `mpiexec` across the provisioned cluster nodes.
* **Hindcasts / Reanalyses vs. Forecasts:** Single forecast runs execute once for a single date. Hindcast/reanalysis workflows loop across a configured start and end date (for example, stepping day-by-day through a multi-year reanalysis) while reusing the same active cluster before teardown. Nowcast + forecast pipelines repeat Stages 2, 4, and 5 sequentially on the same cluster.

### Stage 6: Destroy Compute Nodes (`AWSCluster.py -> terminate()`)
* Terminates all EC2 worker instances at the end of the flow.
* This task is configured to execute **even if Stages 4 or 5 fail**, preventing orphaned ("zombie") EC2 instances.

---

## 4. Porting a Model: Working Backward from the Entry Point

When onboarding a new model from an external HPC system, the easiest strategy is to **work backward** from the exact `mpirun` or `mpiexec` command that launches the compiled binary, and strip away scheduler-specific wrappers.

### Example A: Stripping WCOSS `J`-Jobs (NOSOFS)
1. **Operational Entry Point:** On NOAA's WCOSS, `jobs/JNOS_OFS_NOWCST_FCST` submits a batch job that calls `ush/nos_ofs_nowcast_forecast.sh`:
   ```bash
   # Original operational script
   mpiexec -n ${TOTAL_TASKS} $EXECnos/${RUN}_roms_mpi ./${RUN}_${OCEAN_MODEL}_forecast.in >> ${MODEL_LOG_FORECAST}
   ```

2. **Sandbox Adaptation:** Replace the WCOSS `J`-job wrapper with a lightweight `jobs/fcstrun.sh` script, and parameterize `MPIEXEC` and `MPIOPTS` so Cloudflow injects the dynamic EC2 host list and core counts at runtime:
   ```bash
   # Sandbox-adapted script
   ${MPIEXEC} ${MPIOPTS} $EXECnos/${RUN}_roms_mpi ./${RUN}_${OCEAN_MODEL}_forecast.in >> ${MODEL_LOG_FORECAST}
   ```

### Example B: Replacing Slurm `srun` Templates (CORA ADCIRC)
1. **Original Slurm Setup:** External scripts (such as `run_storms.sh`) often substitute paths into a `submit.slurm.template` file and submit it via `sbatch`, calling `srun $PHOME/<ModelName> <WriterCores>`.

2. **Sandbox Adaptation:** Set `$Job_Manager` to `"cloudflow"`, create a `submit.cloudflow.template` modeled after `jobs/fcstrun.sh`, update the `module load` statements to match the Sandbox Spack environment, and replace `srun` with direct `mpirun`:
   ```bash
   mpirun $MPIOPTS $PHOME/<ModelName> <WriterCores>
   ```

### Known Architectural Discontinuities
When inspecting existing workflows in the repository, keep two legacy patterns in mind:
* **LiveOcean Hindcasts & ADCIRC CORA:** These workflows bypass `jobs/fcstrun.sh` and use dedicated multi-step launchers (for example, `workflows/lohindcast_main.py` -> `flows.py:hindcast_flow()` -> `tasks.py:hindcast_run_multi()` -> `workflows/fcst_launcher.sh` -> `LiveOcean/jobs/fcstrun.sh`).
* **Hurricane Florence Test Script:** `cloudflow/workflows/fcstrun_adcirc_cluster.sh` was built for a specific RENCI Hurricane Florence test case and is not a generic ADCIRC/CORA launcher.

---

## 5. Namelist Templating & Configuration Files

### Creating Model Input Templates (`cloudflow/job/templates/`)
Start from a working model namelist or control file (such as ROMS `ocean.in` or ADCIRC `fort.15`) and replace run-specific values—dates, file paths, time steps, and `NPROCS`—with double-underscore placeholders (`__VARIABLE__`):

```python
settings = {
    "__ProjectName__": self.ProjectName,
    "__ProjectHome__": self.ProjectHome,
    "__ADCIRCHome__": self.ADCIRCHome,
    "__NPROCS__": self.NPROCS,
}
```

At runtime, `sedoceanin(template, outfile, settings)` substitutes each placeholder to generate the active input file for the run. *(Note: Some older `Job` classes hardcode `self.TEMPLPATH = f"{curdir}/templates"`; verify your class's template path when testing new templates.)*

### Cluster Configuration & Cost Tagging (`cluster.configs/*.ioos`)
Always include descriptive `Name` and `Project` tags in your cluster JSON configuration so administrators can identify active clusters and track costs:

```json
{
  "platform": "AWS",
  "region": "us-east-2",
  "nodeType": "hpc6a.48xlarge",
  "nodeCount": 1,
  "tags": [
    { "Key": "Name", "Value": "MODEL_XXX" },
    { "Key": "Project", "Value": "AFFILIATION" }
  ],
  "image_id": "ami-082e2e4e7ef1c6d77",
  "key_name": "rsa_sandbox_key",
  "sg_ids": ["sg-05c044182398b2a27", "sg-0e5148638d9196f69", "sg-06ca6bc5d4b377dad"],
  "subnet_id": "subnet-00075cfbfcbc8f2cf",
  "placement_group": "ioos_cloud_sandbox_Terraform_Placement_Group",
  "vm_retry_delay": 300,
  "vm_max_retries": 5
}
```

* **Handling AWS Capacity Limits (`vm_retry_delay` / `vm_max_retries`):** If AWS temporarily lacks on-demand capacity for your requested `nodeType`, `AWSCluster.py` automatically retries with exponential backoff (defaulting to `5` retries starting at `300` seconds). Note that mid-run hindcast failures do not auto-restart unless retry logic is explicitly added to the Prefect flow.
* **Benchmarking Node Counts:** Every model has a scaling ceiling where adding more cores yields minimal runtime gains—or slows execution due to MPI communication overhead—while multiplying EC2 costs. Run short benchmark tests across 1, 2, and 4 nodes to find the optimal cost-to-runtime ratio.

---

## 6. Workflow Troubleshooting & Safe Termination

### Environment Module Mismatches
Models must be executed with the exact same compiler and MPI modules used to compile them. Load those modules explicitly inside your Cloudflow launcher script (`experiment_launcher.sh` or `fcstrun.sh`), and prepend `module purge` if you encounter conflicts:

```text
ERROR: Unable to locate a modulefile for 'intel-oneapi-compilers/2023.1.0-gcc-11.2.1-3rbcwfi'
```

Run `module avail` on the head node to inspect currently installed Spack module versions.

### Head Node vs. Compute Node AMI Drift (Re-Imaging Required)
Compute worker nodes are spun up from a static AWS Machine Image (`image_id` in your `cluster.configs/*.config` file), whereas only shared EFS directories (such as `/save`, `/com`, and `/nos`—see [Storage & Data Management](./STORAGE_AND_DATA_MANAGEMENT.md)) are mounted across both the head node and worker nodes.

- **When a Cluster AMI Rebuild Is Required:** If you run a Cloudflow job and encounter unexpected runtime errors after modifying the head node **outside** of shared EFS storage—such as installing or updating system RPM packages, modifying `/etc`, or changing local user configuration files (e.g., `~/.bashrc`) that worker nodes depend on during MPI execution—the worker nodes will not see those changes. You must request a cluster AMI rebuild:
  1. Contact the Sandbox Help Desk (`sandbox.helpdesk@noaa.gov`), as re-imaging requires elevated AWS administrator privileges.
  2. Coordinate a brief shutdown of the head node so it can be re-imaged into a new compute node AMI.
  3. Update the `image_id` field in your `cluster.configs/*.config` file to reference the new AMI ID.
- **When Re-Imaging Is *Not* Required:** Changes contained entirely within shared EFS volumes (e.g., `/save/$USER`), or head-node-only `~/.bashrc` customizations (such as shell aliases or prompt settings) that compute worker nodes do not rely on during job execution, do not require an AMI rebuild.
  
### Reading Logs & Inspecting Worker Nodes
* **Find the First Error:** Check your Cloudflow log (`./workflow_main.py >& run.log &`), `~/cluster_runtime.log`, and the model's native log in `/com` or `/ptmp` (e.g., `cbofs.t00z.<date>.forecast.log`, `mirror.out`, or `lofcst.log`). The root cause is almost always the **first** error in the log rather than the final traceback.
* **SSH into a Running Worker Node:** Private worker IPs are printed in the workflow log (`"host": "10.0.0.177"`). You can SSH directly from the head node (`ssh 10.0.0.177`) to inspect CPU/I/O progress if a model appears hung.

### How to Safely Stop a Hung Run (Preventing Zombie Nodes)

> **CRITICAL — NEVER use `Ctrl+C` or `kill` on the Python workflow process:**  
> Killing `workflow_main.py` directly prevents Prefect from reaching Stage 6 (`AWSCluster.terminate()`), leaving expensive EC2 worker nodes running indefinitely.

1. Check for your active `mpiexec` process on the head node (verify ownership so you do not terminate another user's run):
   ```bash
   ps -ef | grep mpiexec
   ```

2. Terminate **your** `mpiexec` or `fcstrun.sh` process (`kill <PID>`).
3. Cloudflow will catch the non-zero exit code, propagate the exception up the Prefect stack, and automatically destroy the EC2 compute cluster.
4. If a run crashes unexpectedly and you suspect compute nodes are still active, note your cluster's `Name` tag, instance type, and node count, and email **sandbox.helpdesk@noaa.gov** immediately so administrators can verify and terminate them in the AWS Console. *(See also: [Zombie Job Checklist](./ZOMBIE_JOB_CHECKLIST.md).)*

---

## Next Steps

* **Approved EC2 Instance Types & Glossary:** Proceed to [Glossary & Best Practices](./GLOSSARY_AND_BEST_PRACTICES.md) for the complete EFA instance reference table, Sandbox Do's and Don'ts, and cloud/HPC terminology cheat sheet.
   
