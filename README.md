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

# step 8 : assemble genome ! yay

used a nextflow script: 

```
#!/usr/bin/env nextflow

// Define the list of IDs
params.ids = [ '01_hilli', '02_quadrimaculata', '03_stygia', '04_vicina']

// Create a channel from the list of IDs
Channel.from(params.ids)
    .set { id_ch }

// Define the process for running flye
process runFlye {
    cpus 24
    memory '60 GB'
    publishDir("/nesi/nobackup/uow03920/05_blowfly_assembly_march/09_assembly", mode: 'move')
    // Define the input
    input:
    val id from id_ch

    // Define the output
    output:
    file("${id}_flye") into flye_out_ch

    // Define the command to be executed
    script:
    """
    flye --nano-hq /nesi/nobackup/uow03920/05_blowfly_assembly_march/08_filtered/${id}.fastq \
         --out-dir ${id}_flye --threads ${task.cpus} -g 700m
    """
}

// Define what to do with the output (optional)
flye_out_ch.view()
```

and ran it using a slurm script: 

```
#!/bin/bash -e
#SBATCH --account=uow03920
#SBATCH --job-name=nextflow_Flye
#SBATCH --time=48:00:00
#SBATCH --cpus-per-task=48
#SBATCH --mem=180G
#SBATCH --mail-type=ALL
#SBATCH --mail-user=paige.matheson14@gmail.com
#SBATCH --output nextflow_flye_%j.out    # save the output into a file
#SBATCH --error nextflow_flye%j.err     # save the error output into a file

# purge all other modules that may be loaded, and might interfare
module purge

## load tools
module load Flye/2.9.3-gimkl-2022a-Python-3.11.3 
ml Nextflow/22.10.7
### FLYE

nextflow run flye.nf
```


# step 11: quast 

```
pip install quast
```

```
quast.py -t 16 -o /nesi/nobackup/uow03920/05_blowfly_assembly_march/10_post_assembly_QC -l '01_hilli 02_quadrimaculata 03_stygia 04_vicina'  /nesi/nobackup/uow03920/05_blowfly_assembly_march/09_assembly/01_hilli/01_hilli.fasta /nesi/nobackup/uow03920/05_blowfly_assembly_march/09_assembly/02_quadrimaculata/02_quadrimaculata.fasta /nesi/nobackup/uow03920/05_blowfly_assembly_march/09_assembly/03_stygia/03_stygia.fasta /nesi/nobackup/uow03920/05_blowfly_assembly_march/09_assembly/04_vicina/04_vicina.fasta
```

# step 12: busco 

```#!/bin/bash -e
#SBATCH --account=uow03920
#SBATCH --job-name=compleasm
#SBATCH --time=3:00:00
#SBATCH --cpus-per-task=8
#SBATCH --mem=15G
#SBATCH --mail-type=ALL
#SBATCH --mail-user=paige.matheson14@gmail.com
#SBATCH --output compleasm_%j.out    # save the output into a file
#SBATCH --error compleasm_%j.err     # save the error output into a file

# purge all other modules that may be loaded, and might interfare
module purge

## load tools
module load compleasm/0.2.5-gimkl-2022a

### Compleasm
for i in 01_hilli 02_quadrimaculata 03_stygia 04_vicina;
do
compleasm.py run -a /nesi/nobackup/uow03920/05_blowfly_assembly_march/09_assembly/${i}_flye/${i}.fasta -o /nesi/nobackup/uow03920/05_blowfly_assembly_march/10_post_assembly_QC/02_busco/${i}/${i}_compleasm -l diptera_odb10 -t 8 ;
done
```

# step 13: purge haplotigs 
busco revealed that there were quite a few duplicated genes, so we will purge haplotigs to try and remove them

1. First align the fastq files to the fasta sequences to generate .paf files. Make a folder for each sample and do the code in those folders. I was doing it all in one big conglomerate folder and it kept overlapping and creating problems lol. I would recommend completing all of the steps for each sample one at a time rather than doing each step for each sample at the same time (i.e., do all the steps for sample one and then all the steps for sample 2). Needed like 72 hours to run!!!! Ended up timing out quite a few times

```
#!/bin/bash -e

#SBATCH --account=uow03920
#SBATCH --job-name=minimap
#SBATCH --mem=15G
#SBATCH --cpus-per-task=8
#SBATCH --ntasks-per-node=8
#SBATCH --time=72:00:00
#SBATCH --mail-type=ALL
#SBATCH --mail-user=paige.matheson14@gmail.com
#SBATCH --output minimapout_%j.out    # save the output into a file
#SBATCH --error minimap_%j.err     # save the error output into a file

module purge
module load minimap2

#####MINIMAP#####
for i in 01_hilli 02_quadrimaculata 03_stygia 04_vicina; do
minimap2 -c /nesi/nobackup/uow03920/05_blowfly_assembly_march/09_assembly_${i}_flye/${i}.fasta /nesi/nobackup/uow03920/05_blowfly_assembly_march/08_filtered/${i}.fastq > /nesi/nobackup/uow03920/05_blowfly_assembly_march/11_purge_haplotigs/${i}/${i}_alignment.paf
done
```

next, generate statistics for each .paf file. does not need to be done through a slurm script 

```
ml purge_dups
```
```
pbcstat 01_hilli_alignment.paf
```

calculate the cutoffs

```
calcuts PB.stat > cutoffs 2> calcuts_01_hilli.log
```

split the consensus fasta file 

```
split_fa /nesi/nobackup/uow03920/05_blowfly_assembly_march/09_assembly/01_hilli_flye/01_hilli.fasta > con_split_01_hilli.fa
```

generate a self ampping .paf file 

```
minimap2 -xasm5 -DP con_split_01_hilli.fa con_split_01_hilli.fa | gzip -c - > con_split_01_hilli.self.paf.gz
```

purge duplicates 

```
purge_dups -2 -T cutoffs -c PB.base.cov con_split_01_hilli.self.paf.gz > dups_01_hilli.bed 2> purge_dups_01_hilli.log
```

extract sequences!!!

```
get_seqs -e dups_01_hilli.bed 01_hilli.fasta
```

# step ? BUSCO again and quast 

```
#SBATCH --account=uow03920
#SBATCH --job-name=compleasm_purged
#SBATCH --time=3:00:00
#SBATCH --cpus-per-task=8
#SBATCH --mem=15G
#SBATCH --mail-type=ALL
#SBATCH --mail-user=paige.matheson14@gmail.com
#SBATCH --output compleasm_%j.out    # save the output into a file
#SBATCH --error compleasm_%j.err     # save the error output into a file

# purge all other modules that may be loaded, and might interfare
module purge

## load tools
module load compleasm/0.2.5-gimkl-2022a

### Compleasm
for i in 01_hilli 02_quadrimaculata 03_stygia 04_vicina;
do
compleasm.py run -a /nesi/nobackup/uow03920/05_blowfly_assembly_march/11_purge_haplotigs/${i}/${i}_purged.fa -o /nesi/nobackup/uow03920/05_blowfly_assembly_march/12_post_purge_QC/02_busco/${i}/${i}_compleasm -l diptera_odb10 -t 8 ;
done
```

```
quast.py -t 16 -o /nesi/nobackup/uow03920/05_blowfly_assembly_march/12_post_purge_QC -l '01_hilli 02_quadrimaculata 03_stygia 04_vicina'  /nesi/nobackup/uow03920/05_blowfly_assembly_march/11_purge_haplotigs/01_hilli/01_hilli_purged.fa
/nesi/nobackup/uow03920/05_blowfly_assembly_march/11_purge_haplotigs/02_quadrimaculata/02_quadrimaculata_purged.fa
/nesi/nobackup/uow03920/05_blowfly_assembly_march/11_purge_haplotigs/03_stygia/03_stygia_purged.fa
/nesi/nobackup/uow03920/05_blowfly_assembly_march/11_purge_haplotigs/04_vicina/04_vicina_purged.fa
```


# then meeran did the medaka polishing 

# use meryl and merqury to check some other stuff:
need to download meryl AND merqury (https://github.com/marbl/merqury?tab=readme-ov-file#direct-installation <-- see this page for steps on how to install). Importantly - run this line of code to get it to work lol ```export MERQURY=$PWD``` (i.e., set the environment variable as $merqury). idk man it doesn't work without it. 


# first create meryl database: 

```
#!/bin/bash -e

#SBATCH --account=uow03920
#SBATCH --job-name=meryl
#SBATCH --mem=15G
#SBATCH --cpus-per-task=8
#SBATCH --ntasks-per-node=8
#SBATCH --time=2:00:00
#SBATCH --mail-type=ALL
#SBATCH --mail-user=paige.matheson14@gmail.com
#SBATCH --output meryl_%j.out    # save the output into a file
#SBATCH --error meryl_%j.err     # save the error output into a file

cd /nesi/nobackup/uow03920/05_blowfly_assembly_march/16_merqury

module purge
module load Merqury

#meryl
for i in 01_hilli 02_quadrimaculata 03_stygia 04_vicina; do

meryl k=18 count /nesi/nobackup/uow03920/05_blowfly_assembly_march/16_merqury/${i}/consensus.fasta output /nesi/nobackup/uow03920/05_blowfly_assembly_march/16_merqury/${i}/${i}_.meryl;

## Create Histogram
meryl histogram ${i}_k18.meryl > ${i}_k18.hist;

## Replace with space separator
tr '\t' ' ' <${i}_k18.hist > ${i}_k18_s.hist;

done
```

BTW - I created 4 different folders for each species to work within. If you try make it work all within one folder it probably won't work. 

# merqury 

```
#!/bin/bash -e
#SBATCH --account=uow03920
#SBATCH --job-name=merqury
#SBATCH --time=12:00:00
#SBATCH --cpus-per-task=32
#SBATCH --mem=25G
#SBATCH --mail-type=ALL
#SBATCH --mail-user=paige.matheson14@gmail.com
#SBATCH --output merqury_%j.out    # save the output into a file
#SBATCH --error merqury_%j.err     # save the error output into a file

# purge all other modules that may be loaded, and might interfare
module purge
## load module 
module load Merqury/1.3-Miniconda3
module load R
# Need argparse, ggplot2, scales
Rscript -e 'install.packages(c("argparse", "ggplot2", "scales"),repos = "http://cran.us.r-project.org")'

module load SAMtools/1.19-GCC-12.3.0
module load BEDTools/2.30.0-GCC-11.3.0
module load Java/20.0.2
module load IGV/2.16.1

cd /nesi/nobackup/uow03920/05_blowfly_assembly_march/16_merqury/merqury
export MERQURY=$PWD

for i in 01_hilli 02_quadrimaculata 03_stygia 04_vicina;
do
cd /nesi/nobackup/uow03920/05_blowfly_assembly_march/16_merqury/${i}

#meryl count
$MERQURY/merqury.sh ${i}_.meryl ${i}.fasta ${i}_merqury_output
```

# KAT to check quality 
```
#!/bin/bash -e
#SBATCH --account=uow03920
#SBATCH --partition=milan
#SBATCH --job-name=kat_analysis
#SBATCH --time=12:00:00
#SBATCH --cpus-per-task=8
#SBATCH --ntasks-per-node=2
#SBATCH --mem=50G
#SBATCH --mail-type=ALL
#SBATCH --mail-user=paige.matheson14@gmail.com
#SBATCH --output kat_analysis_%j.out    # save the output into a file
#SBATCH --error kat_analysis_%j.err     # save the error output into a file

# purge all other modules that may be loaded, and might interfere
module purge
ml KAT/2.4.2-gimkl-2018b-Python-3.7.3

########## Loop ##############
for i in 01_hilli 02_quadrimaculata 03_stygia 04_vicina;
do
    cd /nesi/nobackup/uow03920/05_blowfly_assembly_march/16_KAT/${i}
    #### Link files
    #assembly
    ln -s /nesi/nobackup/uow03920/05_blowfly_assembly_march/14_medaka_polished/${i}_flye_medaka/consensus.fasta
    ####kat command
    kat comp -t 16 -o ${i}_kat "/nesi/nobackup/uow03920/05_blowfly_assembly_march/06_concatenate/${i}.fastq" consensus.fasta
    cd ../../../
done
```

# Scaffolding
Want to check synteny to see if i can use the high quality C. vicina reference genome to scaffold all my species

If two genomes are syntenic, scaffolding will likely improve contiguity and be accurate.
If they’re not syntenic (due to inversions, translocations, etc.), reference-guided scaffolding can:
- Misplace contigs
- Introduce false joins
- Create chimeric scaffolds that do not exist in nature

Synteny analysis (e.g., dot plots) shows:

- Conserved regions (straight diagonals)
- Inversions (flipped diagonals)
- Duplications, deletions, or rearrangements (off-diagonal or fragmented patterns)

This lets you:
- Understand evolutionary divergence between species
- Avoid blindly trusting scaffolding decisions in divergent regions

Used mumMER

```#!/bin/bash -e
#SBATCH --account=uow03920
#SBATCH --job-name=synteny_check
#SBATCH --time=10:00:00
#SBATCH --cpus-per-task=8
#SBATCH --ntasks-per-node=2
#SBATCH --mem=10G
#SBATCH --mail-type=ALL
#SBATCH --mail-user=paige.matheson14@gmail.com
#SBATCH --output synteny_%j.out    # save the output into a file
#SBATCH --error synteny_%j.err     # save the error output into a file

# purge all other modules that may be loaded, and might interfere
module purge
ml MUMmer  

########## Loop ##############
for i in 01_hilli 02_quadrimaculata 03_stygia 04_vicina; do 
nucmer --prefix=vicina_vs_${i} /nesi/nobackup/uow03920/05_blowfly_assembly_march/14_medaka_polished/04_vicina_flye_medaka/consensus.fasta /nesi/nobackup/uow03920/05_blowfly_assembly_march/14_medaka_polished/${i}_flye_medaka/consensus.fasta
delta-filter -1 vicina_vs_${i}.delta > vicina_vs_${i}.filtered.delta
mummerplot --fat --postscript --layout --prefix=vicina_vs_${i} vicina_vs_${i}.filtered.delta;
done
```

# scaffold using ragtag

```
#!/bin/bash 
#SBATCH --account=uow03920
#SBATCH --job-name=scaffold
#SBATCH --time=10:00:00
#SBATCH --cpus-per-task=8
#SBATCH --ntasks-per-node=2
#SBATCH --mem=30G
#SBATCH --mail-type=ALL
#SBATCH --mail-user=paige.matheson14@gmail.com
#SBATCH --output scaffold_%j.out    # save the output into a file
#SBATCH --error scaffold_%j.err     # save the error output into a file

# purge all other modules that may be loaded, and might interfere
module purge

# Load necessary modules
ml Python/3.11.6-foss-2023a
ml minimap2/2.28-GCC-12.3.0
ml unimap/0.1-GCC-11.3.0
pip install RagTag

#ragtag of stygia which had better synteny with vicina hi c so more straightfoward
ragtag.py scaffold /nesi/nobackup/uow03920/05_blowfly_assembly_march/17_vicina/vicina_hic.fna /nesi/nobackup/uow03920/05_blowfly_assembly_march/14_medaka_polished/03_stygia_flye_medaka/consensus.fasta -o ragtag_stygia

#ragtag of hilli which is a bit more cautious due to less synteny
ragtag.py scaffold --nucmer-params="--maxmatch -c 100 -g 1000" --remove-small /nesi/nobackup/uow03920/05_blowfly_assembly_march/17_vicina/vicina_hic.fna /nesi/nobackup/uow03920/05_blowfly_assembly_march/14_medaka_polished/03_stygia_flye_medaka/consensus.fasta -o ragtag_hilli


#evaluate results
ragtag.py stats ragtag_stygia/ragtag.scaffold.fasta
ragtag.py stats ragtag_hilli/ragtag.scaffold.fasta
```


| Parameter   | What it does                                               | Why it's good                                    |
| ----------- | ---------------------------------------------------------- | ------------------------------------------------ |
| `-maxmatch` | Align all maximal exact matches (most sensitive mode)      | Captures all syntenic regions, even short ones   |
| `-c 100`    | Minimum cluster length = 100 bp                            | Reduces noise from short spurious matches        |
| `-g 1000`   | Max distance between two matches to be in the same cluster | Allows flexibility in aligning regions with gaps |




# QC of all assemblies using QUAST, BUSCO, qualimap, and KAT

# busco (do a loop for this if multiple assemblies) 
```
#!/bin/bash -e
#SBATCH --account=uow03920
#SBATCH --job-name=compleasm_purged
#SBATCH --time=1:00:00
#SBATCH --cpus-per-task=8
#SBATCH --mem=10G
#SBATCH --mail-type=ALL
#SBATCH --mail-user=paige.matheson14@gmail.com
#SBATCH --output compleasm_%j.out    # save the output into a file
#SBATCH --error compleasm_%j.err     # save the error output into a file

# purge all other modules that may be loaded, and might interfare
module purge

## load tools
module load compleasm/0.2.5-gimkl-2022a

### Compleasm
compleasm.py run -a /nesi/nobackup/uow03920/05_blowfly_assembly_march/19_scaffold/02_quadrimaculata/02_quadrimaculata_scaffold.fasta -o /nesi/nobackup/uow03920/05_blowfly_assembly_march/19_scaffold/00_busco/02_quadrimaculata_compleasm -l diptera_odb10 -t 8;
```

# quast 
```
#!/bin/bash -e

#SBATCH --account=uow03920
#SBATCH --job-name=compleasm_purged
#SBATCH --time=00:10:00
#SBATCH --cpus-per-task=8
#SBATCH --mem=10G
#SBATCH --mail-type=ALL
#SBATCH --mail-user=paige.matheson14@gmail.com
#SBATCH --output compleasm_%j.out    # save the output into a file
#SBATCH --error compleasm_%j.err     # save the error output into a file

quast.py -t 16 -o /nesi/nobackup/uow03920/05_blowfly_assembly_march/19_scaffold/00_quast -l '01_hilli, 02_quadrimaculata, 03_stygia, 04_vicina' /nesi/nobackup/uow03920/05_blowfly_assembly_march/19_scaffold/01_hilli/01_hilli_scaffold.fasta /nesi/nobackup/uow03920/05_blowfly_assembly_march/19_scaffold/02_quadrimaculata/02_quadrimaculata_scaffold.fasta /nesi/nobackup/uow03920/05_blowfly_assembly_march/19_scaffold/03_stygia/03_stygia_scaffold.fasta /nesi/nobackup/uow03920/05_blowfly_assembly_march/19_scaffold/04_vicina/04_vicina_scaffold.fasta
```

# kat

```
#!/bin/bash -e
#SBATCH --account=uow03920
#SBATCH --job-name=kat_analysis
#SBATCH --time=12:00:00
#SBATCH --cpus-per-task=8
#SBATCH --ntasks-per-node=2
#SBATCH --mem=50G
#SBATCH --mail-type=ALL
#SBATCH --mail-user=paige.matheson14@gmail.com
#SBATCH --output kat_analysis_%j.out    # save the output into a file
#SBATCH --error kat_analysis_%j.err     # save the error output into a file

# purge all other modules that may be loaded, and might interfere
module purge
ml KAT/2.4.2-gimkl-2018b-Python-3.7.3

########## Loop ##############
for i in 01_hilli 02_quadrimaculata 03_stygia 04_vicina;
do
    cd /nesi/nobackup/uow03920/05_blowfly_assembly_march/19_scaffold/00_kat
    #### Link files
    #assembly
    ln -s /nesi/nobackup/uow03920/05_blowfly_assembly_march/19_scaffold/${i}/${i}_scaffold.fasta
    ####kat command
    kat comp -t 16 -o ${i}_kat "/nesi/nobackup/uow03920/05_blowfly_assembly_march/06_concatenate/${i}.fastq" ${i}_scaffold.fasta
    cd ../../../
done
```

# qualimap 

```
#!/bin/bash -e
#SBATCH --account=uow03920
#SBATCH --job-name=qualimap
#SBATCH --time=01:00:00
#SBATCH --mem=20G
#SBATCH --mail-type=ALL
#SBATCH --mail-user=paige.matheson14@gmail.com
#SBATCH --output qualimap%j.out
#SBATCH --error qualimap%j.err

module purge
module load BWA/0.7.18-GCC-12.3.0

for i in 01_hilli 02_quadrimaculata 03_stygia 04_vicina;
do
    # Index the scaffolded genome for alignment
    bwa index /nesi/nobackup/uow03920/05_blowfly_assembly_march/19_scaffold/${i}/${i}_scaffold.fasta;

    # Align reads to the scaffolded genome and process BAM file
    bwa mem -t 24 /nesi/nobackup/uow03920/05_blowfly_assembly_march/19_scaffold/${i}/${i}_scaffold.fasta \
    samtools view --threads 16 -F 0x4 -b - | \
    samtools fixmate -m --threads 16 - - | \
    samtools sort -m 2g --threads 16 - | \
    samtools markdup --threads 16 -r - /nesi/nobackup/uow03020/05_blowfly_assembly_march/19_scaffold/00_qualimap/${i};

    # Index the BAM file
    samtools index -@ 24 /nesi/nobackup/uow03920/05_blowfly_assembly_march/19_scaffold/00_qualimap/${i}/${i}_sort_sfld.bam -o /nesi/nobackup/uow03920/05_blowfly_assembly_march/19_scaffold/00_qualimap/${i}/${i}_sort_sfld.bam.bai;

    # Perform Qualimap bamqc on the BAM file
    /nesi/nobackup/uow03920/05_blowfly_assembly_march/19_scaffold/00_qualimap/qualimap_v2.3/qualimap bamqc \
    -bam /nesi/nobackup/uow03920/05_blowfly_assembly_march/19_scaffold/00_qualimap/${i}/${i}_sort_sfld.bam \
    -outdir /nesi/nobackup/uow03920/05_blowfly_assembly_march/19_scaffold/00_qualimap \
    -nt 16 --java-mem-size=8G;

    cd ../;
done
```


# at this point it is probably a good idea to download all of the QC files generated to save for results :)

# BlobToolKit for QC and Detecting Contamination in Assembly

BlobToolKit is used here for quality control and to detect potential contamination in the genome assembly. Several files need to be generated for BlobToolKit to function effectively:

Blast output (to identify taxonomic affiliations)
Aligned Nanopore reads (BAM file with .csi index)
Assembly (FASTA format)
BUSCO report (CSV format)

## BLAST the assembly against the NT database

```
#!/bin/bash -e
#SBATCH --account=uow03744
#SBATCH --job-name=blastn
#SBATCH --time=120:00:00
#SBATCH --cpus-per-task=46
#SBATCH --mem=120G
#SBATCH --mail-type=ALL
#SBATCH --mail-user=meeranhussain1996@gmail.com
#SBATCH --output Blastn_%j.out
#SBATCH --error Blastn_%j.err

# Purge any pre-loaded modules
module purge

# Load BLAST modules
module load BLASTDB/2024-07 BLAST/2.13.0-GCC-11.3.0

# Run BLAST for each scaffolded genome
for i in 06 07 08 13 16 19 40; do 
    blastn \
    -query ./Maethio_${i}/ragtag.scaffold.fasta \
    -task megablast \
    -db nt \
    -outfmt '6 qseqid staxids bitscore std sscinames sskingdoms stitle' \
    -culling_limit 10 \
    -num_threads 42 \
    -evalue 1e-3 \
    -out ./Maethio_${i}/Maethio_${i}_megablast.out;
done
```

# blob to check for contamination

Blob uses the older version of busco so need to run the old one. 

`ml busco`
`ml miniprot`

```busco -i /nesi/nobackup/uow03920/05_blowfly_assembly_march/19_scaffold/01_hilli/01_hilli_scaffold.fasta -l diptera_odb10 -o 01_hilli -m genome --out_path ./01_hilli```

Then run blobtools 

```
#!/bin/bash -e
#SBATCH --account=uow03920
#SBATCH --job-name=blob
#SBATCH --time=00:05:00
#SBATCH --cpus-per-task=12
#SBATCH --mem=1G
#SBATCH --mail-type=ALL
#SBATCH --mail-user=paige.matheson14@gmail.com
#SBATCH --output blob_%j.out
#SBATCH --error blob_%j.err

pip install Blob

blobtools create --busco /nesi/nobackup/uow03920/05_blowfly_assembly_march/23_busco_for_blob/01_hilli/01_hilli/short_summary.specific.diptera_odb10.01_hilli.txt --fasta /nesi/nobackup/uow03920/05_blowfly_assembly_march/19_scaffold/01_hilli/01_hilli_scaffold.fasta --cov /nesi/nobackup/uow03920/05_blowfly_assembly_march/20_contamination/01_minimap/01_hilli_sort.bam.csi --hits /nesi/nobackup/uow03920/05_blowfly_assembly_march/20_contamination/01_hilli_megablast.out --replace --taxdump /nesi/nobackup/uow03920/05_blowfly_assembly_march/22_blob/00_taxon 01_hilli
```


























