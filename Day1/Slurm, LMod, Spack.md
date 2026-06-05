# HPC Tutorial: Slurm, Lmod (module load), and Spack

A practical reference guide for working on HPC clusters.
![HPC Architecture](https://raw.githubusercontent.com/amrltu/wdlms-2026/refs/heads/main/Day1/hpc-architecture.jpeg)

## Part 1: Slurm — Job Scheduler

Slurm (Simple Linux Utility for Resource Management) is the most common HPC job scheduler. It controls who gets to use which compute nodes and for how long.

---

### 1.1 Core Concepts

| Term | Meaning |
|------|---------|
| **Job** | A unit of work submitted to the scheduler |
| **Partition** | A logical group of nodes (like a queue), e.g. `gpu`, `short`, `long` |
| **Node** | A physical compute machine |
| **Task** | A process; `--ntasks` controls how many MPI ranks run |
| **CPU / core** | `--cpus-per-task` for OpenMP threads per MPI rank |
| **Account** | Billing group your jobs are charged to |

---

### 1.2 Checking the Cluster

```bash
# List all partitions and their status
sinfo

# Show currently running/queued jobs (all users)
squeue

# Show only your own jobs
squeue -u $USER

# Show detailed info about a specific job
scontrol show job <job_id>

# Show available resources on each node
sinfo -N -l

# Check your account and fairshare priority
sshare -u $USER
sacctmgr show user $USER
```

---

### 1.3 Submitting a Batch Job

Create a job script (`job.sh`):

```bash
#!/bin/bash
#SBATCH --job-name=my_job          # Name shown in squeue
#SBATCH --partition=short          # Which partition to use
#SBATCH --nodes=1                  # Number of nodes
#SBATCH --ntasks=4                 # Number of MPI tasks (total)
#SBATCH --cpus-per-task=4          # Threads per task (OpenMP)
#SBATCH --mem=16G                  # Total memory per node
#SBATCH --time=02:00:00            # Wall-clock limit HH:MM:SS
#SBATCH --output=logs/%j_out.txt   # stdout (%j = job ID)
#SBATCH --error=logs/%j_err.txt    # stderr
#SBATCH --mail-type=END,FAIL       # Email on end/fail
#SBATCH --mail-user=you@email.com

# Load modules
module purge
module load intel/2023 impi/2023

# Run your program
srun ./my_program input.dat
```

Submit it:

```bash
sbatch job.sh
```

---

### 1.4 Interactive Jobs

Use interactive jobs for debugging, compiling, or testing.

```bash
# Simple interactive session (1 node, 2 CPUs, 4 GB, 1 hour)
srun --pty --nodes=1 --ntasks=1 --cpus-per-task=2 \
     --mem=4G --time=01:00:00 bash

# With GPU (if available)
srun --pty --nodes=1 --ntasks=1 --gres=gpu:1 \
     --partition=gpu --time=02:00:00 bash

# Using salloc (reserve nodes first, then ssh in or srun)
salloc --nodes=2 --ntasks=8 --time=01:00:00
srun ./my_program   # runs across the allocated nodes
exit                # releases the allocation
```

---

### 1.5 Managing Jobs

```bash
# Cancel a job
scancel <job_id>

# Cancel ALL your jobs
scancel -u $USER

# Hold a pending job (prevents it from starting)
scontrol hold <job_id>

# Release a held job
scontrol release <job_id>

# Modify a queued job (e.g. change time limit)
scontrol update JobId=<job_id> TimeLimit=04:00:00

# Requeue a failed/running job
scontrol requeue <job_id>
```

---

### 1.6 Monitoring & History

```bash
# Real-time resource usage of a running job
sstat -j <job_id> --format=JobID,MaxRSS,MaxVMSize,AveCPU

# Accounting info after job completes
sacct -j <job_id> --format=JobID,JobName,State,Elapsed,MaxRSS,ExitCode

# All your jobs from the last week
sacct -u $USER --starttime=now-7days \
      --format=JobID,JobName,State,Elapsed,CPUTime,MaxRSS

# Job efficiency report (shows how much of allocated resources were used)
seff <job_id>
```

---

### 1.7 Common Slurm Directives Reference

```bash
#SBATCH --partition=gpu            # Partition name
#SBATCH --nodes=2                  # Node count
#SBATCH --ntasks-per-node=8        # Tasks per node (alternative to --ntasks)
#SBATCH --cpus-per-task=4          # OpenMP threads per task
#SBATCH --mem-per-cpu=2G           # Memory per CPU (alternative to --mem)
#SBATCH --gres=gpu:a100:2          # 2x A100 GPUs per node
#SBATCH --constraint=haswell       # Require nodes with specific feature
#SBATCH --exclusive                # Don't share nodes with other jobs
#SBATCH --dependency=afterok:1234  # Wait for job 1234 to succeed first
#SBATCH --array=1-100              # Job array (100 tasks; use $SLURM_ARRAY_TASK_ID)
#SBATCH --requeue                  # Auto-requeue if preempted
#SBATCH --begin=2025-12-01T08:00   # Don't start before this time
```

---

### 1.8 Job Arrays

Job arrays are ideal for parameter sweeps and embarrassingly parallel workloads.

```bash
#!/bin/bash
#SBATCH --job-name=sweep
#SBATCH --array=1-50               # Runs 50 tasks (IDs 1 to 50)
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=2
#SBATCH --mem=4G
#SBATCH --time=01:00:00
#SBATCH --output=logs/sweep_%A_%a.out  # %A = array job ID, %a = task ID

# Use task ID to select input file or parameter
INPUT="inputs/case_${SLURM_ARRAY_TASK_ID}.dat"
./my_program $INPUT
```

Submit and monitor:
```bash
sbatch array_job.sh
squeue -u $USER     # Shows e.g. 12345_[1-50]
scancel 12345_3     # Cancel only task 3
```

---

### 1.9 MPI + OpenMP Hybrid Jobs

```bash
#!/bin/bash
#SBATCH --nodes=4
#SBATCH --ntasks-per-node=2        # 2 MPI ranks per node
#SBATCH --cpus-per-task=8          # 8 OpenMP threads per rank
#SBATCH --mem=32G
#SBATCH --time=04:00:00

export OMP_NUM_THREADS=$SLURM_CPUS_PER_TASK

module load openmpi/4.1.5 gcc/12.2

# srun handles MPI launch; --bind-to socket helps NUMA locality
srun --bind-to socket ./hybrid_program
```

---

## Part 2: Lmod / Environment Modules (`module load`)

Lmod lets you switch between versions of software (compilers, MPI libraries, applications) without conflicts. It manages `PATH`, `LD_LIBRARY_PATH`, `MANPATH`, etc.

---

### 2.1 Basic Commands

```bash
# List available modules
module avail

# Search for a module by keyword
module avail gcc
module spider python

# Show what a module does (what env vars it sets)
module show gcc/12.2

# Load a module
module load gcc/12.2

# Load multiple modules at once
module load gcc/12.2 openmpi/4.1.5 hdf5/1.14.0

# List currently loaded modules
module list

# Unload a specific module
module unload gcc/12.2

# Swap one version for another
module swap gcc/11.3 gcc/12.2

# Unload ALL loaded modules
module purge

# Save current set of modules as a "default" collection
module save

# Restore saved collection
module restore

# Save a named collection
module save my_chimere_env
module restore my_chimere_env
```

---

### 2.2 Module Hierarchy (IMPORTANT)

Most clusters use a hierarchical module system. You can only see MPI-dependent modules (like parallel HDF5) after loading the correct compiler and MPI first.

```bash
# Step 1: Load compiler
module load gcc/12.2

# Step 2: Now MPI modules built with GCC 12 appear
module load openmpi/4.1.5

# Step 3: Now parallel HDF5/NetCDF (built with GCC+OMPI) appear
module load hdf5/1.14.0-parallel

# WRONG: Loading HDF5 before MPI may give serial-only version
```

Always check with `module spider <name>` if a module is not visible:

```bash
module spider hdf5          # Lists all versions
module spider hdf5/1.14.0   # Shows what prerequisites are needed
```

---

### 2.3 Module Files (Writing Your Own)

If admins allow, you can write a personal module file (Lua format for Lmod).

```
~/.modulefiles/myapp/1.0.lua
```

```lua
-- myapp/1.0.lua
help([[
  Loads MyApp v1.0 from ~/software/myapp
]])

whatis("Name: MyApp")
whatis("Version: 1.0")

local home    = os.getenv("HOME")
local prefix  = pathJoin(home, "software/myapp/1.0")

prepend_path("PATH",            pathJoin(prefix, "bin"))
prepend_path("LD_LIBRARY_PATH", pathJoin(prefix, "lib"))
prepend_path("MANPATH",         pathJoin(prefix, "share/man"))
setenv("MYAPP_HOME",            prefix)
```

Activate your personal module directory:

```bash
module use ~/.modulefiles
module load myapp/1.0
```

Add to `~/.bashrc` to make it permanent:

```bash
module use $HOME/.modulefiles
```

---

### 2.4 Module Tips

```bash
# See full description and help text
module help netcdf-c/4.9.2

# Check which module provides a command
module whatis gcc/12.2

# Show what will change before loading (dry run)
module show openmpi/4.1.5

# Find conflicts between modules
module list       # then check for warnings when loading new ones

# Use in job scripts: always start with module purge
module purge
module load gcc/12.2 openmpi/4.1.5 hdf5/1.14.0
```

---

## Part 3: Spack — Package Manager for HPC

Spack installs scientific software with full control over compilers, versions, variants, and dependencies. It is especially powerful when system modules are insufficient.

---

### 3.1 Installation

```bash
# Clone Spack (do this in your home directory)
cd ~
git clone -c feature.manyFiles=true https://github.com/spack/spack.git

# Activate Spack shell support (add to ~/.bashrc)
. ~/spack/share/spack/setup-env.sh

# Verify
spack --version
```

---

### 3.2 Finding and Installing Packages

```bash
# Search for a package
spack list netcdf
spack list %gcc         # packages buildable with GCC

# Show available versions of a package
spack versions hdf5

# Show all variants (build options) for a package
spack info hdf5
spack info netcdf-c

# Basic install
spack install hdf5

# Install specific version
spack install hdf5@1.14.3

# Install with specific compiler
spack install hdf5@1.14.3 %gcc@12.2.0

# Install with variants enabled/disabled
spack install hdf5@1.14.3 %gcc@12.2.0 +mpi ~cxx

# Install multiple packages
spack install openmpi hdf5 netcdf-c netcdf-fortran
```

---

### 3.3 Specs — Spack's Dependency Language

The "spec" is how you describe exactly what you want:

```
spack install <name>@<version>%<compiler>@<compiler_version>+<variant>~<off_variant>^<dependency_spec>
```

Examples:

```bash
# OpenMPI 4.1.5 with GCC 12, PMI support, no CUDA
spack install openmpi@4.1.5 %gcc@12.2.0 +pmi ~cuda

# HDF5 with parallel support, using that OpenMPI
spack install hdf5 %gcc@12.2.0 +mpi ^openmpi@4.1.5

# NetCDF-C linked to the above HDF5
spack install netcdf-c %gcc@12.2.0 ^hdf5+mpi ^openmpi@4.1.5

# Python 3.11 with optimizations
spack install python@3.11.0 %gcc@12.2.0 +optimizations +shared
```

---

### 3.4 Environments (Recommended Workflow)

Spack environments let you group related packages together — like a `conda env` but for compiled HPC software.

```bash
# Create a new environment
spack env create chimere_env

# Activate it
spack env activate chimere_env
# or: spacktivate chimere_env

# Add packages to the environment
spack add gcc@12.2.0
spack add openmpi@4.1.5 %gcc@12.2.0
spack add hdf5@1.14.3 %gcc@12.2.0 +mpi
spack add netcdf-c %gcc@12.2.0 +mpi
spack add netcdf-fortran %gcc@12.2.0

# View what will be installed (concretize = resolve all deps)
spack concretize

# Or force re-concretize
spack concretize -f

# Install everything
spack install

# Deactivate
spack env deactivate

# List all environments
spack env list

# Remove an environment
spack env remove chimere_env
```

#### Using an `spack.yaml` file

```yaml
# spack.yaml — place in your project directory
spack:
  specs:
    - gcc@12.2.0
    - openmpi@4.1.5 %gcc@12.2.0
    - hdf5@1.14.3 %gcc@12.2.0 +mpi
    - netcdf-c %gcc@12.2.0 +mpi
    - netcdf-fortran %gcc@12.2.0
  view: true                        # creates a merged prefix in .spack-env/view/
  concretizer:
    unify: true                     # all specs share one consistent dependency graph
  packages:
    all:
      compiler: [gcc@12.2.0]
      target: [x86_64]
```

```bash
spack env activate .    # activate env defined by spack.yaml in current dir
spack concretize -f
spack install
```

---

### 3.5 Managing Installed Packages

```bash
# List all installed packages
spack find

# Find a specific package
spack find hdf5
spack find -l hdf5    # show hashes
spack find -ldf hdf5  # show full dependency tree

# Show where a package is installed
spack location -i hdf5

# Uninstall a package
spack uninstall hdf5

# Uninstall and remove all dependents
spack uninstall --dependents hdf5

# Garbage-collect unused packages
spack gc

# See the dependency tree of an installed package
spack dependencies hdf5
spack dependents hdf5

# Check for available updates
spack versions --new hdf5
```

---

### 3.6 Compilers in Spack

```bash
# List compilers Spack knows about
spack compilers

# Find and register system compilers automatically
spack compiler find

# Add a custom compiler manually
spack compiler add /path/to/gcc-13/bin

# Show compiler config
spack config get compilers

# Use a specific compiler for install
spack install mypackage %gcc@12.2.0
spack install mypackage %intel@2023.1.0
```

---

### 3.7 Loading Spack Packages

After installation, load packages into your shell:

```bash
# Load into current shell (sets PATH, LD_LIBRARY_PATH, etc.)
spack load hdf5
spack load hdf5@1.14.3 %gcc@12.2.0

# Load by hash (avoids ambiguity)
spack load /abc1234

# Unload
spack unload hdf5

# Load all packages in an environment
spack env activate my_env
spack load --all

# Generate a shell script to source (for use in job scripts)
spack load --sh hdf5 > load_hdf5.sh
source load_hdf5.sh
```

Use in a Slurm job script:

```bash
#!/bin/bash
#SBATCH --job-name=chimere_run
#SBATCH --ntasks=8
#SBATCH --time=04:00:00

# Method 1: Source Spack and load packages
. ~/spack/share/spack/setup-env.sh
spack env activate chimere_env
spack load --all

# Method 2: Source a pre-generated load script
source ~/load_chimere_env.sh

srun ./chimere
```

---

### 3.8 Mirrors and Binary Caches (Speed Up Installs)

```bash
# Add a public binary cache (pre-built binaries, much faster)
spack mirror add E4S https://cache.e4s.io
spack buildcache keys --install --trust

# List configured mirrors
spack mirror list

# Check if binaries are available before building
spack buildcache list hdf5

# Install from binary cache only (don't build from source)
spack install --cache-only hdf5

# Create your own local mirror (useful on air-gapped systems)
spack mirror create -d /path/to/mirror -f packages.txt
```

---

### 3.9 Debugging and Troubleshooting

```bash
# Verbose output during install
spack install -v hdf5

# Extra debug output
spack --debug install hdf5

# See the build log after a failed install
spack log hdf5

# Re-run the build phase only (not fetch/unpack)
spack install --keep-stage hdf5
spack build hdf5     # rebuild without refetching

# Open an interactive build environment (for manual debugging)
spack build-env hdf5 bash

# See the full spec after concretization
spack spec -lI hdf5    # -l = hashes, -I = install status

# Edit the package recipe
spack edit hdf5

# Check config files
spack config get config
spack config get packages
spack config get compilers
```

---

## Part 4: Putting It All Together

### Typical HPC Workflow

```bash
# 1. Log in to HPC cluster
ssh user@hpc.cluster.edu

# 2. Check available software via modules
module avail
module spider gcc

# 3. Load a module stack
module purge
module load gcc/12.2 openmpi/4.1.5

# 4. If needed software is missing, use Spack
. ~/spack/share/spack/setup-env.sh
spack env activate my_project_env
spack install      # installs all packages defined in spack.yaml

# 5. Write a job script
cat > run.sh << 'EOF'
#!/bin/bash
#SBATCH --job-name=my_sim
#SBATCH --nodes=2
#SBATCH --ntasks-per-node=16
#SBATCH --cpus-per-task=2
#SBATCH --mem=64G
#SBATCH --time=08:00:00
#SBATCH --output=logs/%j.out
#SBATCH --error=logs/%j.err

module purge
module load gcc/12.2 openmpi/4.1.5

. ~/spack/share/spack/setup-env.sh
spack env activate my_project_env
spack load --all

export OMP_NUM_THREADS=2
srun ./my_simulation config.nml
EOF

# 6. Create log directory and submit
mkdir -p logs
sbatch run.sh

# 7. Monitor
watch -n 10 squeue -u $USER

# 8. Check results after completion
sacct -j <job_id> --format=JobID,State,Elapsed,MaxRSS
seff <job_id>
```

---

### Quick Reference Card

| Task | Command |
|------|---------|
| Submit job | `sbatch job.sh` |
| Interactive job | `srun --pty --mem=4G --time=01:00:00 bash` |
| List my jobs | `squeue -u $USER` |
| Cancel job | `scancel <job_id>` |
| Job efficiency | `seff <job_id>` |
| Job history | `sacct -j <job_id>` |
| List modules | `module avail` |
| Search module | `module spider <name>` |
| Load module | `module load <name>/<version>` |
| Reset modules | `module purge` |
| Install package | `spack install <pkg>@<ver> %<compiler>` |
| Create env | `spack env create <name>` |
| Activate env | `spack env activate <name>` |
| Load pkg | `spack load <pkg>` |
| Find installed | `spack find <pkg>` |

---

*Last updated: June 2026*
