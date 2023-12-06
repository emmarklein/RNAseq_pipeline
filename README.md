# RNA-seq pipeline

Excited to work with RNA-seq data? 

This repository includes everything you need for genome mapping. Raw FASTQ files serve as input and undergo genome alignment to a reference. We will go through this process step by step using the T2T reference.

<img width="531" alt="Screenshot 2023-12-04 at 3 32 06 PM" src="https://github.com/emmarklein/RNAseq_pipeline/assets/152921397/41d26ea8-7045-4986-8ec6-e24e0dffa237">

https://hbctraining.github.io/Intro-to-rnaseq-hpc-O2/lessons/03_alignment.html

## SRA download and splitting
Let’s start with raw FASTQ files (.fastq). FASTQ files contain our sequencing information. We can download Sequence Read Archive (SRA) data from: https://www.ncbi.nlm.nih.gov/sra. This site stores raw sequencing data and alignment information. I will be using SRR8615934 as example data!

```
module load sratoolkit
prefetch --max-size 1000000000000 --force all -q SRR8615934
```

The output should be the complete .SRA file from the site above. Next, we need to split the SRA file because it includes paired-end reads. Let’s continue using sratoolkit! 

```
module load sratoolkit
fastq-dump --split-files --gzip -O /your/path/SRR8615934 /your/path/SRR8615934/SRR8615934.sra
```

## Decompressing fastq.gz

Both reads (SRR8670768_1.fastq.gz and SRR8670768_2.fastq.gz) need to be decompressed! We can use gunzip for this step.

```
gunzip -c /your/path/SRR8615934_1.fastq.gz > /your/path/SRR8615934_1.fastq
gunzip -c /your/path/SRR8615934_2.fastq.gz > /your/path/SRR8615934_2.fastq
```

Now, we finally have our unzipped raw fastq files!

## Building a STAR Index

Before we map the reads, we must build a genome index. Let's use the T2T reference with the reference genome FASTA file (GCF_009914755.1_T2T-CHM13v2.0_genomic.fna).

```
module load star
STAR --runThreadN 8 --runMode genomeGenerate --genomeDir /your/path/T2T_genomeDir --genomeFastaFiles /your/path/GCF_009914755.1_T2T-CHM13v2.0_genomic.fna
```
This command builds the index from the reference file and stores the genome index files in the directory, T2T_genomeDir.

## STAR Alignment

Now, we are ready for alignment! STAR Aligner determines locations in the human genome associated with read data. This alignment strategy is highly accurate and outperforms other aligners in mapping speed. 

The STAR alignment algorithm includes two main steps: 
(1) Seed searching 
(2) Clustering, stitching, and scoring. 

In seed searching, STAR aligns reads with the longest sequence that matches one or more locations on the reference genome. Seeds are different parts of a particular read that are mapped separately to different genomic locations. This alignment method is sequential – STAR continues to search for unmapped sections of each read that matches the reference genome. STAR uses an uncompressed suffix array to search for the longest matches. Separate seeds are combined to create a full read by clustering, stitching, and scoring. 

```
module load star
STAR --runThreadN 8 --genomeDir /your/path/T2T_genomeDir/ --outFileNamePrefix SRR8615934_ --readFilesIn /your/path/SRR8615934_1.fastq /your/path/SRR8615934_2.fastq
```

This command uses the index files (in T2T_genomeDir) and FASTQ files for STAR alignment. The output of STAR aligner is read counts per gene! Specifically, we have a SAM file (Sequence Alignment/Map), which are text files that contain alignment information.

## Filtering (Optional)

Let's filter out the reads that are not meeting our quality threshold.

```
module load samtools
samtools view -q 30 /your/path/SRR8615934_Aligned.out.sam > SRR8615934_filtered.sam
```
Awesome! We can do a LOT with this information. For now, let's try making wiggle tracks to visualize the data.

## Wiggle Tracks

To visualize our data, we can create wiggle tracks to upload to UCSC genome browser. Let's run bigsam_to_wig_mm10_wcigar4.pl on both filtered and unfiltered SAM files. This perl script takes in the parameters below to build wiggle tracks. 

Parameters:
1. SAM file
2. chrNameLength.txt file generated in the genome build step of STAR
3. Header for output files to be displayed in genome browser
4. Color of the wiggle track
5. Whether the data is paired-end
6. Whether to log10 normalize the data
7. Bin size for the wiggle track

```
module load perl/5.18.2

sbatch -t 48:00:00 --wrap="perl /your/path/bigsam_to_wig_mm10_wcigar4.pl /your/path/SRR8615934_Aligned.out.sam /your/path/T2T_genomeDir/chrNameLength.txt SRR8615934_unfiltered red y y 50"
sbatch -t 48:00:00 --wrap="perl /your/path/bigsam_to_wig_mm10_wcigar4.pl /your/path/SRR8615934_filtered.sam /your/path/emma/T2T_genomeDir/chrNameLength.txt SRR8615934_filtered red y y 50"
```
The output is filtered and unfiltered .wig files. Before uploading them to UCSC, it is important to update the chromosome names. The script below (change_chromosome_names.py) changes chromosome names for T2T-CHM13 v2.0 to match the traditional chromosome names.

```
# changes chromosome names for T2T-CHM13 v2.0 to match the traditional chromosome names
# input: T2T-CHM13 v2.0 based wiggle track
# output: T2T-CHM13 v2.0 based wiggle track with chromosome names changed

# usage: python change_chromosome_names.py <input_file>

import sys
import os
import re

# open file from input parameter
input_file = open(sys.argv[1], "r")

# open output file
output_file_name = sys.argv[1].strip(".wig") + "_final.wig"
output_file = open(output_file_name, "w")


# create chromosome translation dictionary
chromosome_translation = {
    "NC_060925.1" : "chr1",
    "NC_060926.1" : "chr2",
    "NC_060927.1" : "chr3",
    "NC_060928.1" : "chr4",
    "NC_060929.1" : "chr5",
    "NC_060930.1" : "chr6",
    "NC_060931.1" : "chr7",
    "NC_060932.1" : "chr8",
    "NC_060933.1" : "chr9",
    "NC_060934.1" : "chr10",
    "NC_060935.1" : "chr11",
    "NC_060936.1" : "chr12",
    "NC_060937.1" : "chr13",
    "NC_060938.1" : "chr14",
    "NC_060939.1" : "chr15",
    "NC_060940.1" : "chr16",
    "NC_060941.1" : "chr17",
    "NC_060942.1" : "chr18",
    "NC_060943.1" : "chr19",
    "NC_060944.1" : "chr20",
    "NC_060945.1" : "chr21",
    "NC_060946.1" : "chr22",
    "NC_060947.1" : "chrX",
    "NC_060948.1" : "chrY"
    #, "NC_060949.1" : "chrM"
}

# read each line of the input file
for line in input_file:
    # if the line starts with "variableStep", write it to the output file
    if line.startswith("variableStep"):
        components = line.split(" ")
        chromosome = components[1].split("=")[1]
        new_chrom = chromosome_translation[chromosome]
        new_line = "variableStep chrom=" + new_chrom + " span=" + components[2].split("=")[1]
        output_file.write(new_line)

    else:
        output_file.write(line)
    


input_file.close()
output_file.close()
```

Let's update the chromosome names to generate the final .wig files! Run the commands below with the path to your wiggle tracks from the previous step.

```
sbatch --wrap="python3 change_chromosome_names.py /your/path/SRR8615934_filtered.wig"
sbatch --wrap="python3 change_chromosome_names.py /your/path/SRR8615934_unfiltered.wig"
```


