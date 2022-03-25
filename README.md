# Running AlphaFold as a Singularity container on VSC

This workflow closely follows the recipe on how to build and use an AlphaFold Signularity container as described here: https://github.com/hyoo/alphafold_singularity.

The repository referenced above provides definition files to build a Singularity container of DeepMind's AlphaFold v2 (https://github.com/deepmind/alphafold).

The build instructions for a [non-docker setting] by kalininalab have been used (https://github.com/kalininalab/alphafold_non_docker).

## Build the containers

By default, Singularity uses the VSC user's home directory to store its cache. For very small containers that might not be an issue but bigger ones will cause the home direcotry to run out of space. Hence, it is advisable to point Singularity to use a different directory, e.g., somewhere on the user's scratch space:

```
export SINGULARITY_CACHEDIR=$VSC_SCRATCH/singularity-cache
```

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

# Running AlphaFold as a batch job on VSC
Please refer to the [VIB](https://vib.be/) tutorial material created by Jasper Zuallaert (VIB-UGent), with the help of Alexander Botzki (VIB) and Kenneth Hoste (UGent) here:  https://elearning.bits.vib.be/courses/alphafold/
