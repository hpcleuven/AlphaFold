# General notes on Apptainer/Singularity containers on VSC

## Apptainer cache directory

By default, [Apptainer](https://apptainer.org) (formerly Singularity) uses the VSC user's home directory to store its cache. For very small containers that might not be an issue but bigger ones will cause the home direcotry to run out of space. Hence, it is advisable to point Apptainer to use a different directory, e.g., somewhere on the user's scratch space:

```
export APPTAINER_CACHEDIR=$VSC_SCRATCH/apptainer-cache
```

NOTE: If both ```SINGULARITY_CACHEDIR``` (legacy) and ```APPTAINER_CACHEDIR``` have been defined and have different values, ```APPTAINER_CACHEDIR``` will be used. The value of  ```SINGULARITY_CACHEDIR``` will be accepted only if ```APPTAINER_CACHEDIR``` has not been defined. This holds for all legacy (Singularity) environment variables.

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

## AlphaFold database

The database on the VSC machines at KU Leuven is located in:

Tier-1: /scratch/leuven/projects/lp_alphafold

Tier-2: /lustre1/project/res_00002/lp_alphafold

If these locations are not visible within the Apptainer container then they should be mounted.

# Running AlphaFold as an Apptainer container on VSC at KU Leuven -  Tutorial 1

This workflow closely follows the recipe on how to build and use an AlphaFold Apptainer container as described here: https://github.com/avapirev/alphafold_apptainer.

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

# Running AlphaFold as a batch job on VSC
Please refer to the [VIB](https://vib.be/) tutorial material created by Jasper Zuallaert (VIB-UGent), with the help of Alexander Botzki (VIB) and Kenneth Hoste (UGent) here: https://elearning.bits.vib.be/courses/alphafold/
