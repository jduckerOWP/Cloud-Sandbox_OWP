# Cloudflow Model Integration Template Guide

This step-by-step developer guide walks through integrating a new ocean or coastal model into the **Cloudflow** workflow framework using `Model_Experiment_Template`.

> **Before You Begin:**
> * **Head Node Access & Storage:** Ensure you can connect to your Sandbox head node via Okta Privileged Access ([Getting Started & Access](./GETTING_STARTED_AND_ACCESS.md)) and review filesystem layout rules in [Storage & Data Management](./STORAGE_AND_DATA_MANAGEMENT.md).
> * **Architecture Overview:** Review [Model Porting & Workflow Development](./MODEL_PORTING_AND_WORKFLOW.md) and the [Cloudflow UML Diagram](../cloudflow/docs/UML_ModelWorkflows.png) to see how `Main_Driver`, `Flows`, `JobFactory`, and `[MODEL]_Experiment` classes interact.

---

1. **Log into the head node and create your working directory**
   Once you’ve logged into the head node of Cloud-Sandbox (see [Getting Started & Access](./GETTING_STARTED_AND_ACCESS.md)), navigate to `/save/$USER` (or `/save/ec2-user` if using a shared group account—see [Storage & Data Management](./STORAGE_AND_DATA_MANAGEMENT.md)) and create a new directory based on your affiliated organization (e.g., `mkdir USER_AFFILIATION`) if you're using a shared group account. Change into that newly created directory or your own personal directory `/save/$USER` for the next step.

2. **Clone the latest Cloud-Sandbox repository from GitHub:**
   ```bash
   git clone [https://github.com/ioos/Cloud-Sandbox.git](https://github.com/ioos/Cloud-Sandbox.git)
   ```
3. **Create a job file for your model**
   The keywords in this job file will inform the rest of the workflow.
   Change directory into:
   ```bash
   cd Cloud-Sandbox/job.configs
   ```
Copy `cloud_sandbox_experiment.template` into a new file called `your_model_name.exp`. Inside that new file, specify at minimum:
   - `MODEL`: Model name/class
   - `JOBTYPE`: Model job type (be explicit on the workflow name for the given model class if it is separate from the standard implementation class of `'experiment'`)
   - `APP`: Model application (for a simple job run, use `'basic'`)
   - `EXEC`: Path to the compiled model executable
   - Any executable dependencies as new key-value variables (such as an input file)
   - `MODEL_DIR`: Path to the model run directory on the Cloud-Sandbox

4. **Create an AWS cluster configuration file for your model**
   Change directory into:
   ```bash
   cd Cloud-Sandbox/cluster.configs
   ```
Copy `template.ioos` into a new file called `your_model_name.config` inside your affiliation directory to specify the AWS cloud resource configuration. Edit the following variables (see also [Cluster Configuration & Cost Tagging](./MODEL_PORTING_AND_WORKFLOW.md#5-creating-job--cluster-configuration-files)):
   - `nodeType`: Eligible AWS EC2 node instances are listed in `cloudflow/cluster/AWSCluster.py` under `aws_types` and summarized with core counts (`PPN`), RAM, and EFA specs in [Supported AWS EC2 Compute Instance Types](./GLOSSARY_AND_BEST_PRACTICES.md#2-supported-aws-ec2-compute-instance-types).
   - `nodeCount`: Number of nodes to allocate for the specified AWS instance type. *(Caution: The Cloud-Sandbox AWS account enforces quotas per instance family; benchmark scaling across 1, 2, and 4 nodes first.)*
   - `tags`: Set the `"Name"` and `"Project"` values to reflect your model name and affiliation so administrators can track active runs.
   - `image_id`: The specific compute node AMI matching your head node environment. Defaults point to the NOS Sandbox image; if unsure, contact `sandbox.helpdesk@noaa.gov`.

5. **Build the workflow for your model — `workflow_main.py`**
   Change directory into:
   ```bash
   cd Cloud-Sandbox/cloudflow/workflows
   ```
Inspect `workflow_main.py`.
   - If your head node does not have a pre-configured Cloudflow Conda/Miniforge environment and you need to point to a custom Python path, replace `-S python3 -u` in the first line with your Python executable path (e.g., `#!/usr/bin/env /pathway/to/cloudflow_environment/python`).
   - Follow [Local Python Miniforge3 Installation Instructions](./LOCAL_PYTHON_MINIFORGE3_INSTALLATION_CLOUD_SANDBOX_INSTRUCTIONS.md) if you need to install a compatible Python/Prefect environment on the head node (see also [Prefect v3 Overview](./PREFECTv3.md)).

6. **Build the workflow for your model — `your_model_name_Experiment.py`**
   Create a new Job subclass to parse the keywords from your `.exp` job file (refer to the `[MODEL]_Experiment` hierarchy in the [Cloudflow UML Diagram](../cloudflow/docs/UML_ModelWorkflows.png) and [Implementing a Model Job Class](./MODEL_PORTING_AND_WORKFLOW.md#3-implementing-a-model-job-class)).
   Change directory into:
   ```bash
   cd Cloud-Sandbox/cloudflow/job
   ```
Copy `Model_Experiment_Template.py` to `your_model_name_Experiment.py`. Its default structure mirrors `cloud_sandbox_experiment.template` from Step #3:
   - Rename the `Model_Experiment_Template` Python class to `your_model_name_Experiment`.
   - If your model execution only needs `MODEL_DIR` and `EXEC`, no further changes are required in this file.
   - If your model executable requires extra parameters defined in `your_model_name.exp` in Step #3, parse them inside `parseConfig`:
     - Required variable example (`"IN_FILE": "path/to/inputfile"`): add `self.IN_FILE = cfDict['IN_FILE']` inside `parseConfig`.
     - Optional argument example: add `self.OPTIONAL_ARGUMENT_NAME = cfDict.get('OPTIONAL_ARGUMENT_NAME', "DEFAULT_VALUE")` inside `parseConfig`.

7. **Register your class in `JobFactory.py`**
   Edit `cloudflow/job/JobFactory.py` and add an import statement at the top of the file for the class created in Step #6:
   ```python
   from cloudflow.job.your_model_name_Experiment import your_model_name_Experiment
   ```
Inside `class JobFactory`, update the `job()` factory method:
   - Add an `if/elif` branch matching the `MODEL` and `JOBTYPE` values defined in `your_model_name.exp` in Step #3.
   - Instantiate your new class:
     ```python
     newjob = your_model_name_Experiment(configfile, NPROCS)
     ```

8. **Pass custom job variables in `tasks.py` *(skip if no extra variables were added in Step #3)***
   If your model only uses `MODEL_DIR` and `EXEC`, skip to Step #9. Otherwise, change directory into:
   ```bash
   cd Cloud-Sandbox/cloudflow/workflows
   ```
Edit `tasks.py` inside `template_run` (or the corresponding experiment task function):
   - Add an `elif` block for your `MODEL` / `JOBTYPE` to extract the extra attributes from `job` (e.g., `IN_FILE = job.IN_FILE`) and append `str(IN_FILE)` to the end of the `subprocess.run([...])` argument list passed to `experiment_launcher.sh`.
   - Use the existing `schism`, `dflowfm`, or `ucla-roms` blocks as reference templates.

9. **Update `experiment_launcher.sh`**
   Change directory into:
   ```bash
   cd Cloud-Sandbox/cloudflow/workflows
   ```
Edit `experiment_launcher.sh`:
   - If you completed Step #8, add an `if` block to ingest and `export` the positional argument(s) passed from `tasks.py` (`$7`, `$8`, etc., following the `schism`, `dflowfm`, or `ucla-roms` blocks).
   - Near the bottom of `experiment_launcher.sh`, add an `if/elif` block for your model suite that invokes your model's run script (constructed in Step #10) with `MODEL_DIR`, `EXEC`, and any additional model-specific arguments.

10. **Create your model launcher script — `your_model_run.sh`**
    In `Cloud-Sandbox/cloudflow/workflows`, copy `model_basic_run.sh` to `your_model_run.sh` and customize it to:
    1. Load the exact compiler and MPI Spack modules used to compile your model (`module load ...`; see [Environment Module Mismatches](./MODEL_PORTING_AND_WORKFLOW.md#environment-module-mismatches)).
    2. Export any required runtime or EFA/MPI environment variables.
    3. Execute the model binary with `mpirun` or `mpiexec`.

11. **Verify `experiment_launcher.sh` arguments & `MPIOPTS`**
    Return to `experiment_launcher.sh` and verify that the `if/elif` block from Step #9 matches the exact filename, positional arguments, `JOBTYPE`, and `APP` values defined in `your_model_name.exp`, including any model-specific `MPIOPTS` settings.

12. **Run and monitor your model**
    Change directory into `Cloud-Sandbox/cloudflow` (you must launch workflows from inside the `cloudflow` directory) and submit the workflow in the background (see also [Cloudflow Model Execution Instructions](./CLOUDFLOW_MODEL_EXECUTION_INSTRUCTIONS.md)):
    ```bash
    cd Cloud-Sandbox/cloudflow
    nohup ./workflows/workflow_main.py ../cluster.configs/USER_AFFILIATION/your_model_name.config ../job.configs/your_model_name.exp > your_model_test.out &
    ```
    Monitor progress in real time:
    ```bash
    tail -f your_model_test.out
    ```

    > **Troubleshooting & Safe Termination:** If a run hangs or fails, inspect the first error in `your_model_test.out` and `~/cluster_runtime.log`. **Never** press `Ctrl+C` or `kill` `workflow_main.py` directly—follow [How to Safely Stop a Hung Run](./MODEL_PORTING_AND_WORKFLOW.md#how-to-safely-stop-a-hung-run-preventing-zombie-nodes) and the [Zombie Job Checklist](./ZOMBIE_JOB_CHECKLIST.md) so EC2 compute nodes terminate cleanly.
   
