# General notes on Singularity containers on VSC

## Singularity cache directory

By default, [Singularity](https://sylabs.io) uses the VSC user's home directory to store its cache. For very small containers that might not be an issue but bigger ones will cause the home direcotry to run out of space. Hence, it is advisable to point Singularity to use a different directory, e.g., somewhere on the user's scratch space:

```
export SINGULARITY_CACHEDIR=$VSC_SCRATCH/singularity-cache
```

## Mount external directories

One will have to mount external directories in order for them to be available within the  Singularity container.

One can bind multiple directories in a single command with this syntax:

```
singularity exec --bind /dir1,/dir2:/mnt my_container.sif
```

Using the environment variable instead:

```
export SINGULARITY_BIND="/dir1,/dir2:/mnt"
```

All subdirectories and files in /dir1 and /dir2 will be mounted under /mnt - e.g., */mnt/relative-subdir-path/* and */mnt/filename* - and available for use within the container.

## AlphaFold database

The database on the VSC machines at KU Leuven is located in:

Tier-1: /scratch/leuven/projects/lp_alphafold

Tier-2: /lustre1/project/res_00002/lp_alphafold

If these locations are not visible within the Singularity container then they should be mounted.

# Running AlphaFold as a Singularity container on VSC at KU Leuven -  Tutorial 1

This workflow closely follows the recipe on how to build and use an AlphaFold Signularity container as described here: https://github.com/hyoo/alphafold_singularity.

The repository referenced above provides definition files to build a Singularity container of DeepMind's AlphaFold v2 (https://github.com/deepmind/alphafold).

The build instructions for a [non-docker setting] by kalininalab have been used (https://github.com/kalininalab/alphafold_non_docker).

## Build the containers

Clone the Git repository and navigate to its location:

```
git clone https://github.com/hyoo/alphafold_singularity.git
cd alphafold_singularity
```

Then build the necessary containers:

```
# build base container
singularity build --fakeroot base.sif base.def

# build alphafold container
singularity build --fakeroot alphafold.sif alphafold.def
```

## Run Alphafold

Load the container and source the environment, then start a run, e.g.,:

```
singularity exec --nv -B <DATA_DIR> alphafold.sif bash
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

# Running AlphaFold as a Singularity container on VSC at KU Leuven -  Tutorial 2

This tutorial is based on the [AlphaFold tutorial at HPRC](https://hprc.tamu.edu/wiki/SW:AlphaFold). The Docker container file used to build the Singularity Image File (SIF) can be found here - [catgumag/alphafold](https://hub.docker.com/r/catgumag/alphafold). The respective [GitHub repository](https://github.com/dialvarezs/alphafold) provides the Python interface to run AlphaFold via Singularity on an HPC systems.

Pull and build the AlphaFold container:

```
singularity build AlphaFold2.2.0.sif docker://catgumag/alphafold
```

This will create a Singularity container AlphaFold2.2.0.sif

Pull the wrapper Python script from GitHub:

```
git clone https://github.com/dialvarezs/alphafold.git
```

The wrapper Python script used to run the container is:

```
/path-to-the-git-repository/alphafold/run_alphafold.py
```

Python>=3.8 is required in order to run AlphaFold. Switch to an EasyBuild toolchain with suitable Python version and load the module, e.g.,:

```
module load Python
```

Check the help of the Python script for the available options with:

python /path-to-the-git-repository/alphafold/run_alphafold.py --helpfull

Mount the location of the Python wrapper for example and list its contents like so:
```
singularity exec --bind /path-to-the-git-repository/alphafold:/mnt /path-to-singularity-image-file/AlphaFold2.2.0.sif ls /mnt
```

The Python wrapper script can be used with the container like this:

```
singularity exec --bind /path-to-the-git-repository/alphafold:/mnt /path-to-singularity-image-file/AlphaFold2.2.0.sif python /mnt/run_alphafold.py --helpfull
```

Run the container with, e.g.,:

```
singularity exec --bind /path-to-the-git-repository/alphafold:/mnt /path-to-singularity-image-file/AlphaFold2.2.0.sif python /mnt/run_alphafold.py [OPTIONS]
```

# Running AlphaFold as a batch job on VSC
Please refer to the [VIB](https://vib.be/) tutorial material created by Jasper Zuallaert (VIB-UGent), with the help of Alexander Botzki (VIB) and Kenneth Hoste (UGent) here:  https://elearning.bits.vib.be/courses/alphafold/
