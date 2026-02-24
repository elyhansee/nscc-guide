# NSCC Guide

## 1. Get an Account

<https://user.nscc.sg/saml/>

![What you should see when you enter in nscc](image.png)

After creating your account you should see the dashboard.

![What you should see when you sucessfully enter in](image-1.png)


## 2. Connect to the Systems

### If you are outside the NTU network

1. Connect to the NTU VPN. To get access to the NTU VPN refer to [this guide](https://www3.ntu.edu.sg/cits2/ras/for_cms_ref/GlobalProtectVPN.pdf).

2. Access the NTU Jump Host. Refer to [this guide](https://entuedu.sharepoint.com/teams/ntuhpcusersgroup2/SitePages/Using-NTU-JumpHost-to-NSCC-ASPIRE-2A.aspx) for detailed instructions. **First-time users** should email <hpcsupport@ntu.edu.sg> to request access.

3. Run the following command from the Jump Host. Replace `<userid>` with your **NSCC (not NTU)** user ID.

**Aspire 2A:**
```bash
ssh <userid>@aspire2antu.nscc.sg
```

**Aspire 2A+:**
```bash
ssh <userid>@aspire2pntu.nscc.sg
```

## 3. Transfer Files / Connect to NSCC in Visual Studio Code

### Between Your Computer and NSCC Systems

**Recommended Tools:** [FileZilla](https://filezilla-project.org/) or [WinSCP](https://winscp.net/eng/index.php) or [Visual Studio Code](https://code.visualstudio.com/) (Recommended)

### Visual Studio Code Setup

1. Press `Ctrl-Shift-P` (or `Cmd-Shift-P` on Mac)

![configured SSH](<Screenshot 2025-12-03 103226.png>)

2. Press "Add New SSH Host" / "Configure SSH Hosts"

3. Add these two for VPN and NSCC Cluster (If you are in NTU please ignore the jump host)

![ntu-jump-hots code and asipre2a code](<Screenshot 2025-12-03 103130.png>)

## 4. Storage Setup

Your Home directory has a strict limit (50GB). Do not run big jobs or build containers here. Always work in your Scratch directory (100TB limit).

```bash
cd scratch
```
## 5. Running heavy Installations
Do NOT run heavy installations or code on the Login Node. The login node (where you first land) is shared by everyone. Running heavy tasks like pip install, compiling code, or building containers there will trigger a Fair Share Violation, terminate your processes, and could get your account blocked.

### Start an Interactive Job
Move from the "Login Node" to a "Compute Node" using an interactive session. This gives you a dedicated sandbox to work in.
```bash
# Replace <YOUR_PROJECT_ID> with your actual project code (e.g., personal-xxx)
qsub -I -P <YOUR_PROJECT_ID> -l select=1:ngpus=1:mem=64gb -l walltime=02:00:00 -q normal
```

## 6. Making Containers

### What is a container?

A container is a standard unit of software that packages up code and all its dependencies, so the application runs quickly and reliably from one computing environment to another.

### The Workflow: Pull, Convert, Run

We rarely build containers from scratch on a cluster. Instead, we **pull** existing ones (like "[Alpine](https://hub.docker.com/_/alpine)") from the internet and run them.

#### 1. Load the Tool

First, we must make the Singularity software available in our environment.

```bash
module load singularity
```

#### 2. Pull and Convert (The Setup)

We download the "[Alpine](https://hub.docker.com/_/alpine)" container from Docker Hub. Singularity automatically **converts** the Docker format into a **SIF** (Singularity Image Format) file.

- **Input:** `docker://alpine:latest` (A web address for the container)
- **Output:** `alpine.sif` (A single file on your hard drive)

```bash
singularity pull alpine.sif docker://alpine:latest
```

#### 3. Execute (The Action)

Now, we run a command **inside** that SIF file. The command echo runs inside the container's isolated environment, not on the host computer. It wakes up the Alpine OS, prints the message, and instantly shuts down.

```bash
singularity exec alpine.sif echo "Hello! Singularity is working."
```
Here is what you should see:
![What you should see](<Screenshot 2025-12-09 155919.png>)

## 7. Running a Real Runtime with Python

### 1. Download the official Python image

- `python:3.9-slim`: This tag tells Docker Hub to give us Python version 3.9. The word "slim" gives us a lightweight version without unnecessary tools, keeping the file size small (~50MB).

```bash
singularity pull python.sif docker://python:3.9-slim
```

### 2. Run a Python command inside the container

- **-c**: This flag stands for **"Command"**. This runs the code inside the quote marks immediately.

```bash
singularity exec python.sif python3 -c "print('Python container success! 3 + 4 =', 3+4)"
```
Here is what you should see:
![What you should see](<Screenshot 2025-12-09 161633.png>)

## 8. Running vLLM
Follow these steps to set up your environment, write a test script, and submit a job to the queue.

### 1. Set Up the Environment
Firstly, create and activate a new environment 
```bash
conda create -n vllm-env python=3.10 -y
conda activate vllm-env
```
### 2. Install vLLM 
Install the vLLM library using pip.
```bash
pip install vllm
```

### 3. Create the Python Script
Create a file named vllm_test.py. This script initializes the engine with a small model (TinyLlama) to verify that the installation is working correctly.

```python
from vllm import LLM,SamplingParams

llm=LLM('TinyLlama/TinyLlama-1.1B-Chat-v1.0', gpu_memory_utilization=0.7) 
params=SamplingParams(max_tokens=128,temperature=0.7)
outputs=llm.generate(['What is cake?'],params)

for o in outputs:
    generated_text = o.outputs[0].text
    print(generated_text)
```
gpu_memory_utilization is explicitly set. This is often necessary on shared nodes to prevent vLLM from attempting to reserve all available VRAM, which can cause crashes if the GPU is shared or if overhead is high.

### 4. Create the Job Script
Create a PBS submission script named submit_job.sh. Ensure you replace the placeholders (bracketed text) with your actual project details and username.

```bash
#!/bin/bash
#PBS -N Cake-test

#PBS -l select=1:ngpus=1:mem=400gb
#PBS -l walltime=04:00:00
#PBS -j oe
#PBS -P personal-<Add your own uid>
#PBS -q normal
cd $PBS_O_WORKDIR

module load miniforge3

source ~/anaconda3/etc/profile.d/conda.sh
conda activate yourenvname

python yourfilename.py
```

### 5.Submit the Job
Submit your job script to the scheduler.

```bash
qsub submit_job.sh
```

### 6.Check Status
Use qstat to check if your job is queued (Q), running (R), or finished. Replace <id> with your username.
```bash
qstat -u <your_username>
```
## 9. Running Jupyter Notebook with Singularity

Running JupyterLab on a compute node allows GPU-accelerated interactive development.

Since compute nodes are not directly accessible from your browser, you must:

Submit a PBS job

Use SSH tunneling

1. Create Jupyter PBS Script (jupyter_job.sh)
```bash
#!/bin/bash
#PBS -N gemma3_unsloth_jupyter
#PBS -l select=1:ncpus=16:ngpus=1:mem=64gb
#PBS -l walltime=24:00:00
#PBS -j oe
#PBS -P <YOUR_PROJECT_ID>

set -euo pipefail
cd "$PBS_O_WORKDIR"

export CUDA_VISIBLE_DEVICES=0

module load singularity

SCRATCH_DIR=/scratch/users/ntu/<YOUR_USER_ID>/masterclass
IMG=$SCRATCH_DIR/pytorch_base.sif
CACHE_DIR_INSIDE=/venv_inside/unsloth_cache

echo "=== Starting Jupyter Lab ==="

singularity exec --nv \
  -B $SCRATCH_DIR/hf_cache:/hf_cache_inside \
  -B $SCRATCH_DIR/venv_dir:/venv_inside \
  -B $PBS_O_WORKDIR:/workspace \
  --env PYTHONNOUSERSITE=1 \
  --env HF_HOME=/hf_cache_inside \
  --env TMPDIR=/venv_inside/tmp \
  --env UNSLOTH_CACHE_DIR=$CACHE_DIR_INSIDE \
  --env TRITON_CACHE_DIR=$CACHE_DIR_INSIDE/triton \
  --env TORCH_HOME=/venv_inside/torch_cache \
  "$IMG" \
  bash -c "
    mkdir -p /venv_inside/tmp
    mkdir -p $CACHE_DIR_INSIDE
    mkdir -p $CACHE_DIR_INSIDE/triton
    mkdir -p /venv_inside/torch_cache

    python3 -m venv --system-site-packages /venv_inside/env && \
    source /venv_inside/env/bin/activate && \
    pip install \
      'unsloth[colab-new] @ https://github.com/unslothai/unsloth/archive/refs/heads/main.zip' \
      lark-parser shapely datasets trl jupyterlab && \
    cd /workspace && \
    jupyter-lab --ip=\$(hostname -i) --port=8888 --no-browser
  "

echo "=== Job Complete ==="
```

Submit:
```bash
qsub jupyter_job.sh
```
2. Connect via SSH Tunnel
Step 1: Find IP and Token
```bash
cat gemma3_unsloth_jupyter.o*
```
Look for:

http://10.x.x.x:8888/lab?token=abc123...

Copy:
IP address (10.x.x.x)
Token
Step 2: Create SSH Tunnel (Local Machine)
ssh -N -f -L 8888:10.x.x.x:8888 <userid>@aspire2antu.nscc.sg

Step 3: Open Browser
http://localhost:8888

Paste the token when prompted.

You are now running JupyterLab on an NSCC GPU compute node.

## References

<https://nsccsg.github.io/>
