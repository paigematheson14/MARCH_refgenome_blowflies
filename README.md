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
```

# step 3: convert bam files to FASTQ files

We used SAMtools because it is the best for insect genomes. Other tools are highly optimised for humans etc. SAMtools is also the fastest.

```
#!/bin/bash -e

#SBATCH --account=uow03920
#SBATCH --job-name=fastq_0406
#SBATCH --mem=15G
#SBATCH --cpus-per-task=8
#SBATCH --ntasks-per-node=8
#SBATCH --time=1:00:00
#SBATCH --mail-type=ALL
#SBATCH --mail-user=paige.matheson14@gmail.com
#SBATCH --output fastq_0406__%j.out    # save the output into a file
#SBATCH --error fastq_0406__%j.err     # save the error output into a file

module purge
module load SAMtools

cd /nesi/nobackup/uow03920/05_blowfly_assembly_march/04_demux/0406/0406

#fast q#

samtools fastq d58d8b92237ca34d5af928b49daee629f810f013_3_stygia.bam > /nesi/nobackup/uow03920/05_blowfly_assembly_march/05_fastq/0406_stygia.fastq
samtools fastq d58d8b92237ca34d5af928b49daee629f810f013_4_hilli.bam > /nesi/nobackup/uow03920/05_blowfly_assembly_march/05_fastq/0406_hilli.fastq
```

```
#!/bin/bash -e

#SBATCH --account=uow03920
#SBATCH --job-name=fastq_0408
#SBATCH --mem=15G
#SBATCH --cpus-per-task=8
#SBATCH --ntasks-per-node=8
#SBATCH --time=1:00:00
#SBATCH --mail-type=ALL
#SBATCH --mail-user=paige.matheson14@gmail.com
#SBATCH --output fastq_0408__%j.out    # save the output into a file
#SBATCH --error fastq_0408__%j.err     # save the error output into a file

module purge
module load SAMtools

cd /nesi/nobackup/uow03920/05_blowfly_assembly_march/04_demux/0408/0408

#fastq#

samtools fastq 48305005d92ac294badde036140df6e8ce951184_1_quadrimaculata.bam > /nesi/nobackup/uow03920/05_blowfly_assembly_march/05_fastq/0408_quadrimaculata.fastq
samtools fastq 48305005d92ac294badde036140df6e8ce951184_2_vicina.bam > /nesi/nobackup/uow03920/05_blowfly_assembly_march/05_fastq/0408_vicina.fastq
samtools fastq 48305005d92ac294badde036140df6e8ce951184_3_stygia.bam > /nesi/nobackup/uow03920/05_blowfly_assembly_march/05_fastq/0408_stygia.fastq
samtools fastq 48305005d92ac294badde036140df6e8ce951184_4_hilli.bam > /nesi/nobackup/uow03920/05_blowfly_assembly_march/05_fastq/0408_hilli.fastq
```
```
#!/bin/bash -e

#SBATCH --account=uow03920
#SBATCH --job-name=fastq_0410
#SBATCH --mem=15G
#SBATCH --cpus-per-task=8
#SBATCH --ntasks-per-node=8
#SBATCH --time=1:00:00
#SBATCH --mail-type=ALL
#SBATCH --mail-user=paige.matheson14@gmail.com
#SBATCH --output fastq_0410__%j.out    # save the output into a file
#SBATCH --error fastq_0410__%j.err     # save the error output into a file

module purge
module load SAMtools

cd /nesi/nobackup/uow03920/05_blowfly_assembly_march/04_demux/0410/0410

#fastq#

samtools fastq 06b13970143c3e9538f5935f0d28273385d449bc_1_quadrimaculata.bam > /nesi/nobackup/uow03920/05_blowfly_assembly_march/05_fastq/0410_quadrimaculata.fastq
samtools fastq 06b13970143c3e9538f5935f0d28273385d449bc_2_vicina.bam > /nesi/nobackup/uow03920/05_blowfly_assembly_march/05_fastq/0410_vicina.fastq
samtools fastq 06b13970143c3e9538f5935f0d28273385d449bc_3_stygia.bam > /nesi/nobackup/uow03920/05_blowfly_assembly_march/05_fastq/0410_stygia.fastq
samtools fastq 06b13970143c3e9538f5935f0d28273385d449bc_4_hilli.bam > /nesi/nobackup/uow03920/05_blowfly_assembly_march/05_fastq/0410_hilli.fastq
```

# step 4: merge the libraries together so that we are only working off one file

cat sample1.fastq  sample1a.fastq > MO_01_cat.fastq
cat sample2.fastq  sample2a.fastq > MO_02_cat.fastq
cat sample3.fastq  sample3a.fastq > MO_03_cat.fastq
cat sample4.fastq  sample4a.fastq > MO_04_cat.fastq

# step 5: QC prior to filtering using Nanoplot

```
#!/bin/bash -e

#SBATCH --account=uow03920
#SBATCH --job-name=nanoplot
#SBATCH --mem=15G
#SBATCH --cpus-per-task=8
#SBATCH --ntasks-per-node=8
#SBATCH --time=3:00:00
#SBATCH --mail-type=ALL
#SBATCH --mail-user=paige.matheson14@gmail.com
#SBATCH --output nanoplot_%j.out    # save the output into a file
#SBATCH --error nanoplot_%j.err     # save the error output into a file

module purge
module load NanoPlot

#####NANOPLOT#####

for i in 01_hilli 02_quadrimaculata 03_stygia 04_vicina; do
  NanoPlot --verbose -t 8 --fastq /nesi/nobackup/uow03920/05_blowfly_assembly_march/06_concatenate/${i}.fastq -o /nesi/nobackup/uow03920/05_blowfly_assembly_march/07_pre_assembly_QC/01_NanoPlot/${i}
done
```

# step 6: filtering using CHOPPER
we decided to filter reads that were less than 1000 bases long and with quality scores lower than 10

```
#!/bin/bash -e

#SBATCH --account=uow03920
#SBATCH --job-name=chopper
#SBATCH --mem=40G
#SBATCH --cpus-per-task=8
#SBATCH --ntasks-per-node=8
#SBATCH --time=12:00:00
#SBATCH --mail-type=ALL
#SBATCH --mail-user=paige.matheson14@gmail.com
#SBATCH --output chopper_%j.out    # save the output into a file
#SBATCH --error chopper_%j.err     # save the error output into a file

module purge
module load chopper

#####CHOPPER#####
for i in 01_hilli 02_quadrimaculata 03_stygia 04_vicina; do
chopper --threads 8 -q 10 -l 1000 < /nesi/nobackup/uow03920/05_blowfly_assembly_march/06_concatenate/${i}.fastq > /nesi/nobackup/uow03920/05_blowfly_assembly_march/08_filtered/${i}.fastq ;
done
```

# step 7: post filtering QC using nanoplot
samples improved a lot - hilli and stygia look great. vicina looks a little average. quadrimaculata still looks ehhhhhhhhhhh. 


```
#!/bin/bash -e

#SBATCH --account=uow03920
#SBATCH --job-name=nanoplot_post_filter
#SBATCH --mem=15G
#SBATCH --cpus-per-task=8
#SBATCH --ntasks-per-node=8
#SBATCH --time=3:00:00
#SBATCH --mail-type=ALL
#SBATCH --mail-user=paige.matheson14@gmail.com
#SBATCH --output nanoplot_postfilter__%j.out    # save the output into a file
#SBATCH --error nanoplot_postfilter__%j.err     # save the error output into a file

module purge
module load NanoPlot

#####NANOPLOT#####

for i in 01_hilli 02_quadrimaculata 03_stygia 04_vicina; do
  NanoPlot --verbose -t 8 --fastq /nesi/nobackup/uow03920/05_blowfly_assembly_march/08_filtered/${i}.fastq -o /nesi/nobackup/uow03920/05_blowfly_assembly_march/07_pre_assembly_QC/02_NanoPlot_post_filter/${i}
done
```













