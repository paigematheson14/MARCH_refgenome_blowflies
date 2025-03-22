# MARCH_refgenome_blowflies
Protocol I follow to assemble the reference genomes of four calliphora blowfly species


# Genome assembly of four blowfly species
This repository details the steps I took to create genome assemblies for Calliphora stygia, C. hilli, C. quadrimaculata, and C. vicina. The data is ONT and was sequenced at Bragata. Three libraries were made, resulting in three datasets (R0406, R0408, and R0410). Basecalling is done individually on each library. Files I recieved are in POD5 format. The first step is to basecall using a high accuracy model. 

# STEP 1: Basecalling using Dorado
I used DORADO (https://github.com/nanoporetech/dorado). You first need to create a sample file (in .csv format) that contains metadata for your samples like so:

**run R0406**

| Kit            | Experiment ID | Flow Cell ID | Barcode   | Alias      |
|----------------|---------------|--------------|-----------|------------|
| SQK-NBD114-96  | R0406         | PAY72012     | barcode63 | 3_stygia   |
| SQK-NBD114-96  | R0406         | PAY72012     | barcode64 | 4_hilli    |

**run R0408**

| Kit            | Experiment ID | Flow Cell ID | Barcode   | Alias               |
|----------------|---------------|--------------|-----------|----------------------|
| SQK-NBD114-96  | R0408         | PAY72012     | barcode61 | 1_quadrimaculata     |
| SQK-NBD114-96  | R0408         | PAY72012     | barcode62 | 2_vicina             |
| SQK-NBD114-96  | R0408         | PAY72012     | barcode63 | 3_stygia             |
| SQK-NBD114-96  | R0408         | PAY72012     | barcode64 | 4_hilli              |

**run R0410**

| Kit            | Experiment ID | Flow Cell ID | Barcode   | Alias               |
|----------------|---------------|--------------|-----------|----------------------|
| SQK-NBD114-96  | R0410         | PAY78695     | barcode61 | 1_quadrimaculata     |
| SQK-NBD114-96  | R0410         | PAY78695     | barcode62 | 2_vicina             |
| SQK-NBD114-96  | R0410         | PAY78695     | barcode63 | 3_stygia             |
| SQK-NBD114-96  | R0410         | PAY78695     | barcode64 | 4_hilli              |


NOTE: you will need the kit name and experiment ID to match what is in your data. If you didn't recieve this information from your sequencer, it is possible to retrieve this information from the POD5 files using the POD5 tool (https://github.com/nanoporetech/pod5-file-format).

The CSV file needs to be in the same folder as the POD5 files. If you have two libraries to basecall, then you need to have two folders and run each library seperately. Here is the slurm script for basecalling (BTW you only need the '#SBATCH --gpus-per-node=A100:1' line for the basecalling part, so can remove from subsequent slurms)

```
#!/bin/bash -e

#SBATCH --account=uow03920
#SBATCH --job-name=dorado
#SBATCH --gpus-per-node=A100:1
#SBATCH --mem=40G
#SBATCH --cpus-per-task=8
#SBATCH --ntasks-per-node=8
#SBATCH --time=124:00:00
#SBATCH --mail-type=ALL
#SBATCH --mail-user=paige.matheson14@gmail.com
#SBATCH --output basecallout_%j.out    # save the output into a file
#SBATCH --error basecallerr_%j.err     # save the error output into a file

module purge
module load Dorado/0.9.1

##############DORADO#####################
dorado basecaller sup /nesi/nobackup/uow03920/BLOWFLY_ASSEMBLY_DATA/02_basecalling/run_0410/pod5 --recursive --device 'cuda:all' --kit-name SQK-NBD114-96 --sample-sheet 0410.csv > /nesi/nobackup/uow03920/BLOWFLY_ASSEMBLY_DATA/03_BAM/0410_sup_calls_rsumd.bam
```

# step 2: demultiplexing using DEMUX
I used the DORADO demux tool (https://github.com/nanoporetech/dorado), running all three libraries seperately. I tried to run this a couple of times using a slurm script but it failed twice so I just ran it directly on Nesi and it worked fine - took about two hours per library. The output is four BAM files (i.e. one BAM file per sample).

```
#!/bin/bash -e

#SBATCH --account=uow03920
#SBATCH --job-name=demux 
#SBATCH --mem=30G
#SBATCH --cpus-per-task=8
#SBATCH --ntasks-per-node=8
#SBATCH --time=5:00:00
#SBATCH --mail-type=ALL
#SBATCH --mail-user=paige.matheson14@gmail.com
#SBATCH --output demux_0406__%j.out    # save the output into a file
#SBATCH --error demux_0406__%j.err     # save the error output into a file

module purge
module load Dorado/0.9.1

cd /nesi/nobackup/uow03920/05_blowfly_assembly_march/03_BAM       

#demux#
dorado demux --output-dir nesi/nobackup/uow03920/05_blowfly_assembly_march/04_demux/0406 --no-classify 0406_sup_calls_rsumd.bam





