# Running AlphaFold as a Singularity container on VSC

This workflow closely follows the recipe on how to build and use an AlphaFold Signularity container as described here: https://github.com/hyoo/alphafold_singularity.
The repository referenced above provides definition files to build a singularity container of DeepMind's AlphaFold v2 (https://github.com/deepmind/alphafold).
The build instructions for a [non-docker setting] by kalininalab have been used (https://github.com/kalininalab/alphafold_non_docker).

## Build the containers

By default, Singularity uses the VSC user's home directory to store its cache. For very small containers that might not be an issue but bigger ones will cause the home direcotry to run out of space. Hence, it is advisable to point Singularity to use a different directory, e.g., somewhere on the user's scratch space:

```
export SINGULARITY_CACHEDIR=$VSC_SCRATCH/singularity-cache
```

Then build the necessary containers:

```
# build base container
singularity build --fakeroot base.sif base.def

# build alphafold container
singularity build --fakeroot alphafold.sif alphafold.def
```

## Run Alphafold
```
singularity exec --nv -B <DATA_DIR> alphafold.sif bash
source /opt/miniconda3/etc/profile.d/conda.sh
conda activate alphafold
cd /opt/alphafold/
./run.sh -d <DATA_DIR>  -o <OUTPUT_DIR> -m model_1 -f <SEQUENCE_FILE> -t 2020-05-14
```
