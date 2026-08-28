# Cloudflow Python Execution Instructions

A guide for staging, configuring, and executing Python workflows (Serial/Basic, MPI-enabled, and Dask-scalable) on an `ioos/Cloud-Sandbox` head node using Cloudflow and Prefect.

---

## 1. Prerequisites & Environment Setup

Stage your Python scripts to S3 and set up your workspace on the shared EFS volume (`/save`) before executing Python jobs on the head node.

### Stage Python Scripts to S3
From your local machine:
```bash
aws s3 cp /pathway/to/your/Python/script.py s3://ioos-transfers/script.py --recursive
```

### Workspace Preparation
On the Cloud-Sandbox head node:
```bash
cd /save
mkdir -p jason.ducker && cd jason.ducker

# Pull staged scripts from S3
aws s3 cp s3://ioos-transfers/script.py ./script.py --recursive

# Clone the repository
git clone [https://github.com/ioos/Cloud-Sandbox.git](https://github.com/ioos/Cloud-Sandbox.git)
cd Cloud-Sandbox/cloudflow/
```

> **Note:** If you do not have a compatible Python environment installed, follow the **Python Miniforge3 Installation** guide on the main [Cloud-Sandbox Repository](https://github.com/ioos/Cloud-Sandbox.git).

---

## 2. Prefect Workflow Server Setup

All Cloudflow execution modes require an active local Prefect server. **Python Miniforge3 Installation** will install the proper `cloudflow` environment to start a Prefect server if one is already not available. 

```bash
# Check if Prefect server is available
if curl -s --connect-timeout 2 [http://127.0.0.1:4200/api/health](http://127.0.0.1:4200/api/health) > /dev/null; then
    echo "Prefect server is RUNNING and available."
else
    echo "No active Prefect server detected."
fi
```

If no server is detected, start and configure it:
```bash
/save/jason.ducker/miniforge3/envs/cloudflow/bin/prefect server start --background
/save/jason.ducker/miniforge3/envs/cloudflow/bin/prefect config set PREFECT_API_URL=[http://127.0.0.1:4200/api](http://127.0.0.1:4200/api)
```

---

## 3. Python Execution Modes

Cloudflow supports three primary Python execution modes on Cloud-Sandbox:

---

### Mode A: Basic Python Execution (Serial / Concurrent)

Single-node execution utilizing one EC2 instance for serial execution or node-level concurrency (e.g., `multiprocessing`).

#### Step 1: Bind Python Cloudflow Executable
```bash
sed -i 's|#!/usr/bin/env -S python3 -u|#!/usr/bin/env /save/jason.ducker/miniforge3/envs/cloudflow/bin/python|' workflows/workflow_main.py
```

#### Step 2: Configure Job Specification
Edit `../job.configs/MODEL_EXPERIMENTS/python.experiment`:
```json
{
  "MODEL"   : "PYTHON",
  "JOBTYPE" : "python_experiment",
  "APP"     : "basic",
  "EXEC"    : "/save/jason.ducker/miniforge3/envs/special_env/bin/python",
  "SCRIPT"  : "./workflows/python_examples.py"
}
```
* **`MODEL`**: The Cloud-Sandbox model class (`PYTHON`).
* **`JOBTYPE`**: Job category for the model class (`python_experiment`).
* **`APP`**: Application execution mode (`basic` for a single-node Python run).
* **`EXEC`**: Absolute pathway to the specific Python executable containing required script dependencies. This can be a separate environment from the Cloudflow launch environment[cite: 4].
* **`SCRIPT`**: Path to your target Python script harbored on the head node's EFS volume.

#### Step 3: Configure Cluster Infrastructure
Edit `../cluster.configs/Experiments/python.ioos` (**Note:** `nodeCount` must strictly be `1` for Basic mode):
```json
{
  "platform"        : "AWS",
  "region"          : "us-east-2",
  "nodeType"        : "c5.4xlarge",
  "nodeCount"       : 1,
  "vm_retry_delay"  : 300,
  "vm_max_retries"  : 5,
  "tags"            : [
                        { "Key": "Name", "Value": "Python" },
                        { "Key": "Project", "Value": "Model_Experiment" }
                      ],
  "image_id"        : "ami-0cbf4e2a2768a602b",
  "key_name"        : "rsa_sandbox_key",
  "sg_ids"          : ["sg-05c044182398b2a27", "sg-0e5148638d9196f69", "sg-06ca6bc5d4b377dad"],
  "subnet_id"       : "subnet-00075cfbfcbc8f2cf",
  "placement_group" : "ioos_cloud_sandbox_Terraform_Placement_Group",
  "table_name"      : "IOOS-Sandbox-Compute-Nodes"
}
```
* **`platform`**: Target cloud provider (`AWS`).
* **`region`**: AWS region hosting Sandbox resources (`us-east-2`).
* **`nodeType`**: Requested AWS EC2 instance type (e.g., `c5.4xlarge`).
* **`nodeCount`**: Number of EC2 compute nodes allocated (**Note:** Basic mode allows strictly `1` node).
* **`vm_retry_delay`**: Delay in seconds between resource allocation queries (`300`).
* **`vm_max_retries`**: Maximum query retries for AWS resource requests (`5`).
* **`tags`**: Resource tags assigned for Prefect job ID tracking.
* **`image_id`**: AMI ID matching the head node and mounted EFS volume (`ami-0cbf4e2a2768a602b`).
* **`key_name`**: SSH key pair name configured for the AWS account (`rsa_sandbox_key`).
* **`sg_ids`**: Security group IDs for network traffic control.
* **`subnet_id`**: AWS account subnet identifier.
* **`placement_group`**: Target data center placement group.
* **`table_name`**: AWS DynamoDB table tracking job specifications (`IOOS-Sandbox-Compute-Nodes`).

#### Step 4: Submit and Monitor Job
```bash
nohup ./workflows/workflow_main.py ../cluster.configs/Experiments/python.ioos ../job.configs/MODEL_EXPERIMENTS/python.experiment > cloudflow_test.out &
tail -f cloudflow_test.out
```

---

### Mode B: MPI-Enabled Python Execution (`mpi4py`)

Multi-node parallel Python execution linked via `mpiexec` across host instances.

#### Step 1: Environment Build Script (`mpi4py` + `netCDF4`)
MPI Python environments must be compiled against Spack HPC modules available on the head node. Below is an example build script to compile and link Intel MPI compilers and netCDF4 libraries to your Python environment:

```bash
#!/bin/bash
set -e

echo "=== Loading Spack HPC Modules ==="
module load intel-oneapi-compilers/2023.1.0-gcc-11.2.1-3rbcwfi
module load esmf/8.5.0-intel-2021.9.0-5sphkv4

export MPICC=$(which mpiicc)
export MPICXX=$(which mpiicpc)
export NETCDF4_DIR=$(nc-config --prefix)
export HDF5_DIR=$(dirname $(dirname $(which h5dump)))
export LDFLAGS="-L$(nc-config --libdir) -Wl,-rpath,$(nc-config --libdir)"
export CFLAGS="-I$(nc-config --includedir)"

echo "=== Building Environment ==="
./miniforge3/bin/mamba env create -f ocs_mesh_cloud_sandbox.yml -y

ENV_NAME=$(grep -E '^name:' ocs_mesh_cloud_sandbox.yml | awk '{print $2}')
ENV_PATH="./miniforge3/envs/$ENV_NAME"
ENV_PIP="$ENV_PATH/bin/pip"
ENV_LIB="$ENV_PATH/lib"
ENV_SP="$ENV_PATH/lib/python3.13/site-packages"
ENV_BIN="$ENV_PATH/bin"

# Purge conflicting binaries and pre-packaged mpi4py
rm -f $ENV_LIB/libnetcdf.so* $ENV_LIB/libhdf5*
rm -rf $ENV_SP/mpi4py* $ENV_SP/*mpi4py*

echo "=== Building Parallel Extensions ==="
$ENV_PIP install mpi4py --no-binary mpi4py --no-build-isolation --no-cache-dir
$ENV_PIP install h5py --no-binary h5py --no-build-isolation --no-cache-dir
$ENV_PIP install cftime --no-cache-dir

git clone [https://github.com/Unidata/netcdf4-python.git](https://github.com/Unidata/netcdf4-python.git)
cd netcdf4-python && git checkout v1.7.4rel
rm -f src/netCDF4/_netCDF4.c
export CFLAGS="-I$(nc-config --includedir) -DPyMPI_HAVE_MPI_Session=0"
../$ENV_PIP install . --no-binary netcdf4 --no-build-isolation --no-deps --no-cache-dir
cd .. && rm -rf netcdf4-python

$ENV_BIN/python -c "import mpi4py, netCDF4; print('MPI environment ready!')"
```

#### Step 2: Configure Job & Cluster Specifications
Edit `../job.configs/MODEL_EXPERIMENTS/python_mpi.experiment`:
```json
{
  "MODEL"   : "PYTHON",
  "JOBTYPE" : "python_experiment",
  "APP"     : "mpi",
  "EXEC"    : "/save/jason_ducker/miniforge3/envs/ocsmesh/bin/python",
  "SCRIPT"  : "/save/jason_ducker/GSOC_Group/test_mpi.py"
}
```
* **`MODEL`**: Model framework class (`PYTHON`).
* **`JOBTYPE`**: Job type (`python_experiment`).
* **`APP`**: Application execution mode (`mpi` for parallel MPI execution).
* **`EXEC`**: Absolute pathway to the compiled MPI-enabled Python environment (e.g., `ocsmesh`).
* **`SCRIPT`**: Absolute pathway to the target parallel Python script hosted on EFS.

Edit `../cluster.configs/Experiments/python.ioos` (Set `nodeCount` to desired scale, e.g., `2`)[cite: 4]:
```json
{
  "platform"        : "AWS",
  "region"          : "us-east-2",
  "nodeType"        : "c5.4xlarge",
  "nodeCount"       : 2,
  "vm_retry_delay"  : 300,
  "vm_max_retries"  : 5,
  "image_id"        : "ami-0cbf4e2a2768a602b",
  "key_name"        : "rsa_sandbox_key",
  "sg_ids"          : ["sg-05c044182398b2a27", "sg-0e5148638d9196f69", "sg-06ca6bc5d4b377dad"],
  "subnet_id"       : "subnet-00075cfbfcbc8f2cf",
  "placement_group" : "ioos_cloud_sandbox_Terraform_Placement_Group",
  "table_name"      : "IOOS-Sandbox-Compute-Nodes"
}
```
* **`platform`**: Target cloud provider (`AWS`).
* **`region`**: AWS region hosting Sandbox resources (`us-east-2`).
* **`nodeType`**: Requested AWS EC2 instance type (e.g., `c5.4xlarge`).
* **`nodeCount`**: Allocated compute nodes (`2` or more). Because `mpiexec` manages multi-node IP distribution, MPI Python jobs scale seamlessly across multiple instances.
* * **`vm_retry_delay`**: Delay in seconds between resource allocation queries (`300`).
* **`vm_max_retries`**: Maximum query retries for AWS resource requests (`5`).
* **`tags`**: Resource tags assigned for Prefect job ID tracking.
* **`image_id`**: AMI ID matching the head node and mounted EFS volume (`ami-0cbf4e2a2768a602b`).
* **`key_name`**: SSH key pair name configured for the AWS account (`rsa_sandbox_key`).
* **`sg_ids`**: Security group IDs for network traffic control.
* **`subnet_id`**: AWS account subnet identifier.
* **`placement_group`**: Target data center placement group.
* **`table_name`**: AWS DynamoDB table tracking job specifications (`IOOS-Sandbox-Compute-Nodes`).

#### Step 3: Configure Shell Launcher Script
Edit `workflows/python_mpi_basic_run.sh` to load required runtime modules:
```bash
module load intel-oneapi-compilers/2023.1.0-gcc-11.2.1-3rbcwfi
module load esmf/8.5.0-intel-2021.9.0-5sphkv4
```

#### Step 4: Submit and Monitor Job
```bash
nohup ./workflows/workflow_main.py ../cluster.configs/Experiments/python.ioos ../job.configs/MODEL_EXPERIMENTS/python_mpi.experiment > cloudflow_test.out &
tail -f cloudflow_test.out
```

---

### Mode C: Dask Distributed Execution

Scalable task or data parallelism across AWS workers via Dask Distributed.

> **Important Requirement:** Dask requires a single unified Conda environment containing both Cloudflow dependencies and script-level dependencies across head and worker nodes.

#### Step 1: Create Unified Dask Environment
Create `dask_miniforge3_installation.yml` that includes your dask script Python module dependencies as needed starting with the template yaml file below:
```yaml
name: cloudflow_dask
channels:
  - conda-forge
dependencies:
  - pip
  - setuptools
  - wheel
  - boto3==1.40.22
  - botocore==1.40.22
  - dask>=2026.1.0
  - distributed
  - matplotlib==3.10.6
  - netCDF4==1.7.2
  - numpy==2.3.2
  - pillow>=12.2.0
  - pyproj==3.7.2
  - pip:
    - "fastapi>=0.111.0,<0.116.0"
    - prefect==3.6.29
    - haikunator
```

Build environment and bind main workflow script:
```bash
mamba env create -f dask_miniforge3_installation.yml -y
sed -i 's|#!/usr/bin/env -S python3 -u|#!/usr/bin/env /save/jason.ducker/miniforge3/envs/cloudflow_dask/bin/python|' workflows/workflow_main.py
```

#### Step 2: Configure Script Logging
Inside your script (`./workflows/python_examples.py`), redirect logging to Dask worker output:
```python
import logging
log = logging.getLogger("distributed.worker")
log.info("Executing Dask task step...")
```

---

#### Method 1: Data Parallelism (`client.map`)
Use when applying a single function across an iterable dataset across workers.

1. **Job Configuration** (`../job.configs/MODEL_EXPERIMENTS/python_dask_data_parallelism.experiment`):
   ```json
   {
     "MODEL"                        : "PYTHON",
     "JOBTYPE"                      : "python_experiment",
     "APP"                          : "dask_data_parallelism_example",
     "SCRIPT"                       : "./workflows/python_examples.py",
     "ARG1"                         : "./dask_output"
   }
   ```
* **`MODEL`**: Model framework class (`PYTHON`).
* **`JOBTYPE`**: Job type (`python_experiment`).
* **`APP`**: Python application mode for dask data parallelism.
* **`SCRIPT`**: Absolute pathway to the dask Python script hosted on EFS. This must be located within the `Cloud-Sandbox/cloudflow/workflows` directory for proper dask implementation on cloudflow.
* **`ARG1`**: Processing parameter (e.g., target year `2001`).
* **`ARG2`**: Target output directory path on EFS.

2.  **Configure Cluster Infrastructure**
Edit `../cluster.configs/Experiments/python.ioos`:
```json
{
  "platform"        : "AWS",
  "region"          : "us-east-2",
  "nodeType"        : "c5.4xlarge",
  "nodeCount"       : 2,
  "vm_retry_delay"  : 300,
  "vm_max_retries"  : 5,
  "tags"            : [
                        { "Key": "Name", "Value": "Python" },
                        { "Key": "Project", "Value": "Model_Experiment" }
                      ],
  "image_id"        : "ami-0cbf4e2a2768a602b",
  "key_name"        : "rsa_sandbox_key",
  "sg_ids"          : ["sg-05c044182398b2a27", "sg-0e5148638d9196f69", "sg-06ca6bc5d4b377dad"],
  "subnet_id"       : "subnet-00075cfbfcbc8f2cf",
  "placement_group" : "ioos_cloud_sandbox_Terraform_Placement_Group",
  "table_name"      : "IOOS-Sandbox-Compute-Nodes"
}
```
* **`platform`**: Target cloud provider (`AWS`).
* **`region`**: AWS region hosting Sandbox resources (`us-east-2`).
* **`nodeType`**: Requested AWS EC2 instance type (e.g., `c5.4xlarge`).
* **`nodeCount`**: Number of EC2 compute nodes allocated for dask job.
* **`vm_retry_delay`**: Delay in seconds between resource allocation queries (`300`).
* **`vm_max_retries`**: Maximum query retries for AWS resource requests (`5`).
* **`tags`**: Resource tags assigned for Prefect job ID tracking.
* **`image_id`**: AMI ID matching the head node and mounted EFS volume (`ami-0cbf4e2a2768a602b`).
* **`key_name`**: SSH key pair name configured for the AWS account (`rsa_sandbox_key`).
* **`sg_ids`**: Security group IDs for network traffic control.
* **`subnet_id`**: AWS account subnet identifier.
* **`placement_group`**: Target data center placement group.
* **`table_name`**: AWS DynamoDB table tracking job specifications (`IOOS-Sandbox-Compute-Nodes`).
  
3. **Task Integration** (`workflows/tasks.py`)
Modify the code accordingly within the `python_dask_experiment_run` function to conform with your specific Python dask data parallelism implementation. The code block below serves as a guide for proper implementation of the dask data parallelism method on cloudflow:
   ```python
   elif(job.APP == 'dask_data_parallelism_example'):
       client.upload_file(job.SCRIPT)
       from python_examples import dask_data_parallelism_example

       file_list = [f"sensor_{str(i).zfill(3)}" for i in range(0, 21)]
       output_dir = os.path.abspath(job.ARG1)
       os.makedirs(output_dir, exist_ok=True)

       futures = client.map(dask_data_parallelism_example, file_list, output_root=output_dir)
       results = client.gather(futures)
       for r in results:
           print(r)
   ```
4. **Execution**:
   ```bash
   nohup ./workflows/workflow_main.py ../cluster.configs/Experiments/python.ioos ../job.configs/MODEL_EXPERIMENTS/python_dask_data_parallelism.experiment > cloudflow_test.out &
   tail -f cloudflow_test.out
   ```

---

#### Method 2: Task Parallelism (`client.submit`)
Use when submitting dedicated computations to the Dask cluster.

1. **Job Configuration** (`../job.configs/MODEL_EXPERIMENTS/python_dask_task_parallelism.experiment`):
   ```json
   {
     "MODEL"                        : "PYTHON",
     "JOBTYPE"                      : "python_experiment",
     "dask_task_parallelism_example": "basic",
     "SCRIPT"                       : "./workflows/python_examples.py",
     "ARG1"                         : "2001",
     "ARG2"                         : "./dask_output"
   }
   ```
* **`MODEL`**: Model framework class (`PYTHON`).
* **`JOBTYPE`**: Job type (`python_experiment`).
* **`APP`**: Python application mode for dask task parallelism.
* **`SCRIPT`**: Absolute pathway to the dask Python script hosted on EFS. This must be located within the `Cloud-Sandbox/cloudflow/workflows` directory for proper dask implementation on cloudflow.
* **`ARG1`**: Processing parameter (e.g., target year `2001`).
* **`ARG2`**: Target output directory path on EFS.

2.  **Configure Cluster Infrastructure**
Edit `../cluster.configs/Experiments/python.ioos`:
```json
{
  "platform"        : "AWS",
  "region"          : "us-east-2",
  "nodeType"        : "c5.4xlarge",
  "nodeCount"       : 2,
  "vm_retry_delay"  : 300,
  "vm_max_retries"  : 5,
  "tags"            : [
                        { "Key": "Name", "Value": "Python" },
                        { "Key": "Project", "Value": "Model_Experiment" }
                      ],
  "image_id"        : "ami-0cbf4e2a2768a602b",
  "key_name"        : "rsa_sandbox_key",
  "sg_ids"          : ["sg-05c044182398b2a27", "sg-0e5148638d9196f69", "sg-06ca6bc5d4b377dad"],
  "subnet_id"       : "subnet-00075cfbfcbc8f2cf",
  "placement_group" : "ioos_cloud_sandbox_Terraform_Placement_Group",
  "table_name"      : "IOOS-Sandbox-Compute-Nodes"
}
```
* **`platform`**: Target cloud provider (`AWS`).
* **`region`**: AWS region hosting Sandbox resources (`us-east-2`).
* **`nodeType`**: Requested AWS EC2 instance type (e.g., `c5.4xlarge`).
* **`nodeCount`**: Number of EC2 compute nodes allocated for dask job.
* **`vm_retry_delay`**: Delay in seconds between resource allocation queries (`300`).
* **`vm_max_retries`**: Maximum query retries for AWS resource requests (`5`).
* **`tags`**: Resource tags assigned for Prefect job ID tracking.
* **`image_id`**: AMI ID matching the head node and mounted EFS volume (`ami-0cbf4e2a2768a602b`).
* **`key_name`**: SSH key pair name configured for the AWS account (`rsa_sandbox_key`).
* **`sg_ids`**: Security group IDs for network traffic control.
* **`subnet_id`**: AWS account subnet identifier.
* **`placement_group`**: Target data center placement group.
* **`table_name`**: AWS DynamoDB table tracking job specifications (`IOOS-Sandbox-Compute-Nodes`).
  
3. **Task Integration** (`workflows/tasks.py`):
Modify the code accordingly within the `python_dask_experiment_run` function to conform with your specific Python dask task parallelism implementation. The code block below serves as a guide for proper implementation of the dask task parallelism method on cloudflow:
   ```python
   elif(job.APP == 'dask_task_parallelism_example'):
       client.upload_file(job.SCRIPT)
       from python_examples import dask_task_parallelism_example

       log.info("Running Python Dask task parallelism example")
       futures = client.submit(dask_task_parallelism_example, int(job.ARG1))
       results = client.gather(futures)

       output_dir = os.path.abspath(job.ARG2)
       os.makedirs(output_dir, exist_ok=True)

       ds_to_save = results.to_dataset(name='AORC_partial_data_gap')
       output_path = os.path.join(output_dir, f"aorc_gap_mask_{job.ARG1}.nc")
       ds_to_save.to_netcdf(output_path)
       log.info(f"Saved AORC masked year {job.ARG1} to {output_path}")
   ```
4. **Execution**:
   ```bash
   nohup ./workflows/workflow_main.py ../cluster.configs/Experiments/python.ioos ../job.configs/MODEL_EXPERIMENTS/python_dask_task_parallelism.experiment > cloudflow_test.out &
   tail -f cloudflow_test.out
   ```

---

## 4. Integrating Custom Arguments into Cloudflow

Follow this procedure to pass custom arguments (e.g., `ARG1`, `ARG2`) to your Python scripts for any of the modes (Basic, MPI, Dask).

### Step 1: Add Arguments to Job Config
```bash
vi ../job.configs/MODEL_EXPERIMENTS/python.experiment
```
```json
{
  "MODEL"   : "PYTHON",
  "JOBTYPE" : "python_experiment",
  "APP"     : "use_case",
  "EXEC"    : "/save/jason.ducker/miniforge3/envs/special_env/bin/python",
  "SCRIPT"  : "./workflows/python_examples.py",
  "ARG1"    : "5",
  "ARG2"    : "John_Doe"
}
```

### Step 2: Register Arguments in Job Class
```bash
vi job/PYTHON_Experiment.py
```
```python
def parseConfig(self, cfDict):
    self.MODEL = cfDict['MODEL']
    self.jobtype = cfDict['JOBTYPE']
    self.APP = cfDict.get('APP', "basic")
    # Register custom argument inputs
    self.ARG1 = cfDict.get('ARG1', None)
    self.ARG2 = cfDict.get('ARG2', None)
```

> **Note:** For Dask jobs, steps 1 & 2 complete the integration of your arguments to leverage for your dask job. For Basic and MPI modes, proceed through steps 3–6 below.

### Step 3: Handle App Mode in Workflow Runner
```bash
vi workflows/tasks.py
```
```python
elif(JOBTYPE=='python_experiment' and APP=='use_case'):
    SCRIPT = job.SCRIPT
    ARG1 = job.ARG1
    ARG2 = job.ARG2
    try:
        result = subprocess.run([runscript, str(JOBTYPE), str(APP), str(NPROCS), str(PPN), HOSTS, str(MODEL_DIR), str(EXEC), str(SCRIPT), str(ARG1), str(ARG2)], universal_newlines=True, stderr=subprocess.STDOUT)
        if result.returncode != 0:
            log.exception(f'PYTHON use case script execution failed ... result: {result.returncode}')
            raise Exception('PYTHON use case script execution failed')
    except Exception as e:
        log.exception('Exception during subprocess.run :' + str(e))
        raise Exception('Exception during subprocess.run')
    log.info('PYTHON use case script execution finished successfully')
```

### Step 4: Parse Environment Variables in Launcher Header
```bash
vi workflows/experiment_launcher.sh
```
```bash
if [[ "$JOBTYPE" == "python_experiment" && "$APP" == "use_case" ]]; then
  export SCRIPT=$8
  export ARG1=$9
  export ARG2="${10}" 
fi
```

### Step 5: Dispatch Script Execution in Launcher Body
```bash
vi workflows/experiment_launcher.sh
```
```bash
elif [[ "$JOBTYPE" == "python_experiment" && "$APP" == "use_case" ]]; then
    export JOBDIR=$PWD/workflows
    cd "$JOBDIR" || exit 1
    RUNSCRIPT="./python_use_case_run.sh $SCRIPT $EXEC $ARG1 $ARG2"
    $RUNSCRIPT 
    result=$?
```

### Step 6: Update Runner Execution Shell Script
```bash
cp workflows/python_basic_run.sh workflows/python_use_case_run.sh
vi workflows/python_use_case_run.sh
```
```bash
export SCRIPT=$1
export EXEC=$2
export ARG1=$3
export ARG2=$4

ssh ec2-user@$HOSTS "cd $CLOUDFLOW_DIR && env && pwd && $EXEC -m py_compile $SCRIPT $ARG1 $ARG2 && $EXEC -u $SCRIPT $ARG1 $ARG2"
```
