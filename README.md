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
gunzip -c SRR8615934_1.fastq.gz > /your/path/SRR8615934_1.fastq
gunzip -c SRR8615934_2.fastq.gz > /your/path/SRR8615934_2.fastq
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

## Filtering

Let's filter out the reads that are not meeting our quality threshold.

```
module load samtools
samtools view -q 30 /your/path/SRR8615934_Aligned.out.sam > SRR8615934_filtered.sam
```
Awesome! We can do a LOT with this information. For now, let's try making wiggle tracks to visualize the data.

## Wiggle Tracks


