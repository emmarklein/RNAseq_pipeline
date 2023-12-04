# RNA-seq pipeline

Excited to work with RNA-seq data? 

This repository includes everything you need for genome mapping. Raw FASTQ files serve as input and undergo genome alignment to a reference. We will go through this process step by step using the T2T and GRCh38 references.

Here is the typical RNA-seq pipeline flow...

<img width="531" alt="Screenshot 2023-12-04 at 3 32 06 PM" src="https://github.com/emmarklein/RNAseq_pipeline/assets/152921397/41d26ea8-7045-4986-8ec6-e24e0dffa237">

https://hbctraining.github.io/Intro-to-rnaseq-hpc-O2/lessons/03_alignment.html

## SRA download
Let’s start with raw FASTQ files (.fastq). These files contain our sequencing information! We can download Sequence Read Archive (SRA) data from: https://www.ncbi.nlm.nih.gov/sra. This site stores raw sequencing data and alignment information.

```
function test() {
  console.log("notice the blank line before this function?");
}
```
