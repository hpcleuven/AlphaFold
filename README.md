# General notes on Apptainer/Singularity containers on VSC

## Apptainer cache directory

By default, [Apptainer](https://apptainer.org) (formerly Singularity) uses the VSC user's home directory to store its cache (default ```$HOME/.apptainer/cache```). For very small containers that might not be an issue but bigger ones will cause the VSC user home direcotry to run out of space. Hence, it is advisable to point Apptainer to use a different directory, e.g., somewhere on the user's scratch space:

```
export APPTAINER_CACHEDIR=$VSC_SCRATCH/apptainer-cache
```

When building a container, or pulling/running an Apptainer container from a Docker/OCI source, a temporary working space is required (default ```/tmp```). The container is constructed in this temporary space before being packaged into an Apptainer SIF image. Similarly, one should create a temporary workspace:

```
export APPTAINER_TMPDIR=$VSC_SCRATCH/apptainer-tmpdir
```

NOTE: If both ```SINGULARITY_``` (legacy) and ```APPTAINER_``` have been defined and have different values, ```APPTAINER_``` will be used. The value of  ```SINGULARITY_``` will be accepted only if ```APPTAINER_``` has not been defined. This holds for all legacy (Singularity) environment variables.

## Mount external directories

One will have to mount external directories in order for them to be available within the Apptainer container.

One can bind multiple directories in a single command with this syntax:

```
apptainer exec --bind /dir1,/dir2:/mnt my_container.sif
```

Using the environment variable instead:

```
export APPTAINER_BIND="/dir1,/dir2:/mnt"
```

All subdirectories and files in /dir1 and /dir2 will be mounted under /mnt - e.g., */mnt/relative-subdir-path/* and */mnt/filename* - and available for use within the container.

Binding directories like so:

```
apptainer exec --bind /dir1 --bind /dir2 my_container.sif
```

will bind */dir1* and */dir2* as additional separate directories within the container.

## AlphaFold database

The database on the VSC machines at KU Leuven is located in:

Tier-1: /scratch/leuven/projects/lp_alphafold

Tier-2: /lustre1/project/res_00002/lp_alphafold

If these locations are not visible within the Apptainer container then they should be mounted.

# Running AlphaFold as an Apptainer container on VSC at KU Leuven -  Tutorial 1

This workflow follows the recipe on how to build and use an AlphaFold Apptainer container as described here: https://github.com/avapirev/alphafold_apptainer.

The repository referenced above provides definition files to build an Apptainer container of DeepMind's AlphaFold v2 (https://github.com/deepmind/alphafold).

The build instructions for a [non-docker setting] by kalininalab have been used (https://github.com/kalininalab/alphafold_non_docker).

The current recipe makes use of an updated Python packages list. The orginal AlphaFold software relies on and has been tested against a specific version-bound package list, some of which packages are too old and/or obsolate. If the user wishes to use the original package list to build AlphaFold then please refer to Tutorial 2 on how to build the container.

## Build the containers

Clone the Git repository and navigate to its location:

```
git https://github.com/avapirev/alphafold_apptainer.git
cd alphafold_apptainer
```

Then build the necessary containers:

```
# build base container
apptainer build --fakeroot base.sif base.def

# build alphafold container
apptainer build --fakeroot alphafold.sif alphafold.def
```

## Run Alphafold

Load the container and source the environment, then start a run, e.g.,:

```
apptainer exec --nv -B <DATA_DIR> alphafold.sif bash
source /opt/miniconda3/etc/profile.d/conda.sh
conda activate alphafold
cd /opt/alphafold/
./run.sh -d <DATA_DIR>  -o <OUTPUT_DIR> -m model_1 -f <SEQUENCE_FILE> -t 2020-05-14
```

*Required Parameters*:

-d <data_dir>     Path to directory of supporting data

-o <output_dir>   Path to a directory that will store the results

-m <model_names>  Names of models to use (a comma separated list)

-f <fasta_path>   Path to a FASTA file containing one sequence

-t <max_template_date> Maximum template release date to consider (ISO-8601 format - i.e. YYYY-MM-DD). Important if folding historical test sets


*Optional Parameters*:

-b <benchmark>    Run multiple JAX model evaluations to obtain a timing that excludes the compilation time, which should be more indicative of the time required for inferencing many proteins (default: 'False')
  
-g <use_gpu>      Enable NVIDIA runtime to run with GPUs (default: True)
  
-a <gpu_devices>  Comma separated list of devices to pass to 'CUDA_VISIBLE_DEVICES' (default: 0)
  
-p <preset>       Choose preset model configuration - no ensembling (full_dbs) or 8 model ensemblings (casp14) (default: 'full_dbs')

# Running AlphaFold as an Apptainer container on VSC at KU Leuven - Tutorial 2

One can also download the official AlphaFold repository on a local computer, build a Docker container, and then create from it an Aptainer/Singularity container which later can be transferred to VSC. In the workflow below we use Singularity for the container conversion step since it is readily available for install on most Linux systems.

Clone locally the official Alphafold repository and build:

Warning: the building and pushing stages can be very heavy on the CPU

```
git clone https://github.com/deepmind/alphafold.git
cd alphafold
sudo docker build -f docker/Dockerfile -t alphafold .
sudo docker run -d -p 5000:5000 --restart=always --name registry registry:2
sudo docker image tag alphafold:latest localhost:5000/alphafold:latest
sudo docker push localhost:5000/alphafold:latest
sudo SINGULARITY_NOHTTPS=true singularity build alphafold.sif docker://localhost:5000/alphafold:latest
```

The above procedure will create a SIF container which can be now transferred to VSC. The container can be then run, e.g., see the available options like so:
```
apptainer exec $CONTAINER_IMAGE_DIR/alphafold.sif python /app/alphafold/run_alphafold.py --help
```

# Running AlphaFold as a batch job on VSC
Please refer to the [VIB](https://vib.be/) tutorial material created by Jasper Zuallaert (VIB-UGent), with the help of Alexander Botzki (VIB) and Kenneth Hoste (UGent) here: https://elearning.bits.vib.be/courses/alphafold/

Below are two examples of Slurm job submission scripts - one only on CPUs and one using both CPUs and GPUs. Please bear in mind that the job parameters here are only an example and may not be suitable for proper Alphafold runs.
  
Submit to CPU nodes on wICE:
```
#!/bin/bash
#SBATCH --cluster=wice                  # cluster name
#SBATCH --job-name="alpha_monomer_cpu"  # job name
#SBATCH --time=48:00:00                 # max job run time hh:mm:ss
#SBATCH --nodes=3                       # number of nodes
#SBATCH --ntasks-per-node=72            # tasks per compute node
#SBATCH --mem-per-cpu=3400M             # memory per cpu
#SBATCH --output=%x-%j.log              # job log
#SBATCH --account=<project_credits>     # compute credits account

module purge

DATABASE_DIR=/lustre1/project/res_00002/lp_alphafold/AlphaFoldDBdir_20221123
INPUT_DIR=$VSC_SCRATCH/alphafold/fastas
OUTPUT_DIR=$VSC_SCRATCH/alphafold/runs_wice_cpu_slurm

export APPTAINER_CACHEDIR=$VSC_SCRATCH/apptainer-cache
export APPTAINER_BIND="$INPUT_DIR,$OUTPUT_DIR,$DATABASE_DIR"

export CONTAINER_IMAGE_DIR=$VSC_SCRATCH/alphafold_docker_to_apptainer_image

# To list input options type:
# apptainer exec $CONTAINER_IMAGE_DIR/alphafold.sif python /app/alphafold/run_alphafold.py --help
# If using GPUs then use the '--nv' flag, i.e. 'apptainer exec --nv ...'

apptainer exec $CONTAINER_IMAGE_DIR/alphafold.sif python /app/alphafold/run_alphafold.py \
 --data_dir=$DATABASE_DIR \
 --uniref90_database_path=$DATABASE_DIR/uniref90/uniref90.fasta \
 --mgnify_database_path=$DATABASE_DIR/mgnify/mgy_clusters_2018_12.fa \
 --bfd_database_path=$DATABASE_DIR/bfd/bfd_metaclust_clu_complete_id30_c90_final_seq.sorted_opt \
 --uniref30_database_path=$DATABASE_DIR/uniref30/UniRef30_2021_03 \
 --pdb70_database_path=$DATABASE_DIR/pdb70/pdb70 \
 --template_mmcif_dir=$DATABASE_DIR/pdb_mmcif/mmcif_files \
 --obsolete_pdbs_path=$DATABASE_DIR/pdb_mmcif/obsolete.dat \
 --model_preset=monomer \
 --max_template_date=2022-1-1 \
 --db_preset=full_dbs \
 --output_dir=$OUTPUT_DIR \
 --fasta_paths=$INPUT_DIR/T1050.fasta \
 --use_gpu_relax=FALSE

# GPUs must be available if '--use_gpu_relax' is enabled
```

Submit to GPU nodes:
```
#!/bin/bash
#SBATCH --cluster=wice                  # cluster name
#SBATCH --job-name="alpha_monomer_gpu"  # job name
#SBATCH --time=48:00:00                 # max job run time hh:mm:ss
#SBATCH --nodes=1                       # number of nodes
#SBATCH --output=stdout.%x.%j           # save stdout to file
#SBATCH --error=stderr.%x.%j            # save stderr to file
#SBATCH --output=%x-%j.log              # job log
#SBATCH --account=<project_credits>     # compute credits account
#SBATCH --partition=gpu                 # partition
#SBATCH --gpus-per-node=2               # number of gpus per node
#SBATCH --ntasks=36                     # = gpus-per-node x 18

module purge

# Unified memory can be used to request more than just the total GPU memory for the JAX step in AlphaFold
# - A100 GPU has 80GB memory
# - GPU total memory (80) * XLA_PYTHON_CLIENT_MEM_FRACTION (4.0)
# - XLA_PYTHON_CLIENT_MEM_FRACTION default = 0.9
# This example script has 620 GB of unified memory
# - 80x2 GB from A100 GPU + 460 GB DDR from motherboard
# In case of "RuntimeError: Resource exhausted: Out of memory", add the following variables to the slurm script
#export SINGULARITYENV_TF_FORCE_UNIFIED_MEMORY=1
#export SINGULARITYENV_XLA_PYTHON_CLIENT_MEM_FRACTION=4.0

DATABASE_DIR=/lustre1/project/res_00002/lp_alphafold/AlphaFoldDBdir_20221123
INPUT_DIR=$VSC_SCRATCH/alphafold/fastas
OUTPUT_DIR=$VSC_SCRATCH/alphafold/runs_wice_gpu_slurm

export APPTAINER_CACHEDIR=$VSC_SCRATCH/apptainer-cache
export APPTAINER_BIND="$INPUT_DIR,$OUTPUT_DIR,$DATABASE_DIR"

export CONTAINER_IMAGE_DIR=$VSC_SCRATCH/alphafold_docker_to_apptainer_image

# To list input options type:
# apptainer exec --nv $CONTAINER_IMAGE_DIR/alphafold.sif python /app/alphafold/run_alphafold.py --help
# If using GPUs then use the '--nv' flag, i.e. 'apptainer exec --nv ...'

apptainer exec --nv $CONTAINER_IMAGE_DIR/alphafold.sif python /app/alphafold/run_alphafold.py \
 --data_dir=$DATABASE_DIR \
 --uniref90_database_path=$DATABASE_DIR/uniref90/uniref90.fasta \
 --mgnify_database_path=$DATABASE_DIR/mgnify/mgy_clusters_2018_12.fa \
 --bfd_database_path=$DATABASE_DIR/bfd/bfd_metaclust_clu_complete_id30_c90_final_seq.sorted_opt \
 --uniref30_database_path=$DATABASE_DIR/uniref30/UniRef30_2021_03 \
 --pdb70_database_path=$DATABASE_DIR/pdb70/pdb70 \
 --template_mmcif_dir=$DATABASE_DIR/pdb_mmcif/mmcif_files \
 --obsolete_pdbs_path=$DATABASE_DIR/pdb_mmcif/obsolete.dat \
 --model_preset=monomer \
 --max_template_date=2022-1-1 \
 --db_preset=full_dbs \
 --output_dir=$OUTPUT_DIR \
 --fasta_paths=$INPUT_DIR/T1050.fasta \
 --use_gpu_relax=TRUE

# GPUs must be available if '--use_gpu_relax' is enabled
```
