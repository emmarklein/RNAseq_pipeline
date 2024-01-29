# RNA-seq pipeline

Excited to work with RNA-seq data? 

This repository includes everything you need for genome mapping. Raw FASTQ files serve as input and undergo genome alignment to a reference. We will go through this process step by step using the T2T and hg38 references.

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
This command builds the index from the reference file and stores the genome index files in the directory, T2T_genomeDir. We can also do this with the hg38 reference and store the index files in /GRCh38_genomeDir.

```
module load star
STAR --runThreadN 8 --runMode genomeGenerate --genomeDir /your/path/GRCh38_genomeDir --genomeFastaFiles /your/path/GRCh38.primary_assembly.genome.fa
```

## STAR Alignment

Now, we are ready for alignment! STAR Aligner determines locations in the human genome associated with read data. This alignment strategy is highly accurate and outperforms other aligners in mapping speed. 

The STAR alignment algorithm includes two main steps: 
(1) Seed searching 
(2) Clustering, stitching, and scoring. 

In seed searching, STAR aligns reads with the longest sequence that matches one or more locations on the reference genome. Seeds are different parts of a particular read that are mapped separately to different genomic locations. This alignment method is sequential – STAR continues to search for unmapped sections of each read that matches the reference genome. STAR uses an uncompressed suffix array to search for the longest matches. Separate seeds are combined to create a full read by clustering, stitching, and scoring. 

```
module load star
STAR --runThreadN 8 --genomeDir /your/path/T2T_genomeDir/ --outFileNamePrefix SRR8615934_ --readFilesIn /your/path/SRR8615934_1.fastq /your/path/SRR8615934_2.fastq
STAR --runThreadN 8 --genomeDir /your/path/GRCh38_genomeDir/ --outFileNamePrefix SRR8615934_hg_ --readFilesIn /your/path/SRR8615934_1.fastq /your/path/SRR8615934_2.fastq
```

These commands use the index files (in T2T_genomeDir and GRCh38_genomeDir) and FASTQ files for STAR alignment. The output of STAR aligner is read counts per gene! Specifically, we have a SAM file (Sequence Alignment/Map), which are text files that contain alignment information.

## Filtering (Optional)

Let's filter out the reads that are not meeting our quality threshold (q flag). Also, add -h to keep the header!!

```
module load samtools
samtools view -h -q 30 /your/path/SRR8615934_Aligned.out.sam > SRR8615934_filtered.sam
```
Awesome! We can do a LOT with this information. For now, let's try making wiggle tracks to visualize the data.

## Wiggle Tracks

To visualize our data, we can create wiggle tracks to upload to UCSC genome browser. This part includes a few steps with some file type conversions.

## SAM to BAM

```
module load samtools
samtools view -bS /your/path/SRR8615934_hg_Aligned.out.sam > /your/path/SRR8615934_hg_Aligned.out.bam
samtools view -bS /your/path/SRR8615934_hg_filtered.sam > /your/path/SRR8615934_hg_filtered.bam
```

## BAM to BED

Here is a script (make_bam_to_bed12.sh) to run. Make sure to run bam_to_bed12.sh afterwards to submit the jobs and create /bedfiles directory to store the output.

```
#!/bin/bash
module load bedtools/2.29

echo "#!/bin/bash" > bam_to_bed12.sh

for file in your/path/*.bam
do
        file_name=$(basename $file ".bam")
        echo $file_name
        echo "sbatch -t 4:00:00 --wrap=\"bedtools bamtobed -bed12 -splitD -i ${file} > /your/path/bedfiles/${file_name}.bed\"" >> bam_to_bed12.sh
done 
```

```
mkdir bedfiles
sbatch bam_to_bed12.sh
```

## Normalization

In order to standardize the wiggle signal relative to the number of aligned reads in the dataset, we must count the number of reads in our alignment file with `run_count_reads.sh`. Make sure to `mkdir readcounts`!

```
mkdir readcounts
```

Here's our little script (run_count_reads.sh)!

```
module load samtools

for file in /your/path/*.bam
do
        file_name=$(basename $file ".bam")
        sbatch --wrap="samtools view -c $file > /your/path/readcounts/${file_name}_readcounts.txt"
done
```

```
sbatch run_count_reads.sh
```

## Finally.. Making Wiggle Tracks!

After completion of all above jobs (not just submitting them, let them finish running), we need to make the actual wiggle tracks. 

There are multiple parameters here to specify:

1. Path and name of your sample's Bed12 file
2. The path to the chrNameLength.txt file. This is generated by STAR during genome indexing and is stored in the genomeDir/ directory if you followed all of these instructions.*
3. Header for output files to be displayed in genome browser, often short yet descriptive. I avoid spaces.
4. Color of the wiggle track (I like to color +/- strand runs separate colors for easy visualization)
5. Whether to log10 normalize the data: y for log10 n for no normalization
6. Bin size for the wiggle track (50 nucleotides is our standard size)
7. Number of reads in the alignment (per wiggle track) for RPM standardization found in the readcounts file. If you don't wish to standardize by RPM, put the value 1 here.

Here is a script to make the wiggles! (run_wiggle_script.sh)

```
#!/bin/bash

sbatch --wrap="python3 make_wiggle_tracks_11_7_23.py $file $chrNameLengthfile header color lognormalize bin_size ${read_counts}"

*Additionally, if you have pre-aligned data, Longleaf stores some the STAR index of certain genomes in proj, so you can look at the /proj/seq/data/STAR_genomes_v277/ to see if one relevant to your work is available. Older versions of STAR will not produce the file we need, so stick to more up to date versions for the chrNameLength.txt file.

**If you followed my examples above with stranded data, you could use the following script to automatically generate your `run_wiggle_script.sh` commands (check chrNameLengthfile path and change the color, log10 normalization, bin size as desired):

#!/bin/bash

chrNameLengthfile="/your/path/chrNameLength.txt"
echo "#!/bin/bash" > run_wiggle_script.sh

for file in bedfiles/*.bed
do	
	file_name=$(basename $file ".bed")
	# get the contents of the corresponding file from readcounts/
	read_counts=$(cat readcounts/${file_name}_readcounts.txt)
	# identify strand to assign color
	if [[ $file_name == *+* ]]; then
		echo "sbatch --wrap=\"python3 make_wiggle_tracks_11_7_23.py $file $chrNameLengthfile ${file_name} red n 50 ${read_counts}\"" >> run_wiggle_script.sh
	elif [[ $file_name == *-* ]]; then
		echo "sbatch --wrap=\"python3 make_wiggle_tracks_11_7_23.py $file $chrNameLengthfile ${file_name} blue n 50 ${read_counts}\"" >> run_wiggle_script.sh
	else
		echo $file_name
	fi
done
```

## Version for unstranded data:

```
#!/bin/bash

sbatch --wrap="python3 make_wiggle_tracks_11_7_23.py $file $chrNameLengthfile header color lognormalize bin_size ${read_counts}"

*Additionally, if you have pre-aligned data, Longleaf stores some the STAR index of certain genomes in proj, so you can look at the /proj/seq/data/STAR_genomes_v277/ to see if one relevant to your work is available. Older versions of STAR will not produce the file we need, so stick to more up to date versions for the chrNameLength.txt file.

**If you followed my examples above with stranded data, you could use the following script to automatically generate your `run_wiggle_script.sh` commands (check chrNameLengthfile path and change the color, log10 normalization, bin size as desired):

#!/bin/bash

chrNameLengthfile="/your/path/chrNameLength.txt"
echo "#!/bin/bash" > run_wiggle_script.sh

for file in bedfiles/*.bed
do	
	file_name=$(basename $file ".bed")
	# get the contents of the corresponding file from readcounts/
	read_counts=$(cat readcounts/${file_name}_readcounts.txt)
		echo "sbatch --wrap=\"python3 make_wiggle_tracks_11_7_23.py $file $chrNameLengthfile ${file_name} blue n 50 ${read_counts}\"" >> run_wiggle_script.sh
done
```

And.. of course, you will need the actual Python script to make the wiggles! (make_wiggle_tracks_11_7_23.py)

```
import sys
import math

# Import chrM warning: If you want to study chromosome M, note that if there are reads in the first 0-50 nucleotides those are not displayed,
# and the last 50 nucleotides of reads are also not displayed. This is due to problems with visual cutoffs and chromosome lengths.
# If you need coverage in these regions, please adjust the script accordingly.

# Parameters:
#1. Bed12 file
#2. chrNameLength.txt file generated in the genome build step of STAR
#3. Header for output files to be displayed in genome browser
#4. Color of the wiggle track
#5. Whether to log10 normalize the data: y for log10 n for no normalization
#6. Bin size for the wiggle track
#7. Number of reads in the dataset (per wiggle track) for RPM standardization. If you don't wish to standardize by RPM, put the value 1 here.

# ex: python3 make_wiggle_tracks.py <bed12> <chrNameLength.txt> test-output blue n 50 1

# import variables
bedfile = sys.argv[1]
chrNameLength = sys.argv[2]
header = sys.argv[3]
color = sys.argv[4]
log10 = sys.argv[5]
bin_size = int(sys.argv[6])
reads_in_dataset = int(sys.argv[7])


if reads_in_dataset > 1:
    rpm = reads_in_dataset/1000000
else:
    rpm = 1


wig_dict = {}

with open(chrNameLength, "r") as infile:
    for line in infile:
        cols = line.split("\t")
        chrom = cols[0]
        if "GL" not in chrom and "JH" not in chrom and "KI" not in chrom:
            length = cols[1]
            wig_dict[chrom] = {}

# make the wiggle tracks
with open(bedfile, "r") as infile:
    for line in infile:
        cols = line.split("\t")
        chrom = cols[0]
        if chrom in wig_dict.keys():
            start = cols[1]
            strand = cols[5]
            block_sizes = cols[10].split(",")
            block_starts = cols[11].split(",")

            # find bin for each block of read
            my_covered_bins = []

            for i in range(len(block_sizes)):
                block_size = int(block_sizes[i])
                block_start = int(block_starts[i])
                block_end = block_start + block_size
                current_blockstart_bin = int(((int(start)+block_start)/bin_size))
                current_blockend_bin = int(((int(start)+block_end)/bin_size))

                for j in range(current_blockstart_bin, current_blockend_bin+1):
                    my_covered_bins.append(j)
                
            fractional_coverage = 1/len(my_covered_bins)
            for current_bin in my_covered_bins:
                if current_bin not in wig_dict[chrom]:
                    wig_dict[chrom][current_bin] = fractional_coverage
                else:
                    wig_dict[chrom][current_bin] += fractional_coverage

# make three number code for color
color_dict = {
    "blue":"0,0,255",
    "red":"255,0,0",
    "green":"0,255,0",
    "yellow":"255,255,0",
    "orange":"255,165,0",
    "purple":"128,0,128",
    "pink":"255,192,203",
    "black":"0,0,0",
}

color_code = color_dict[color]

# Need to sort the order of the bins for output
for chrom in wig_dict:
    wig_dict[chrom] = dict(sorted(wig_dict[chrom].items()))


# delete the first and last bin in the chrM wig_dict dictionary
if 0 in wig_dict["chrM"].keys():
    del wig_dict["chrM"][0]

last_bin = max(wig_dict["chrM"])
del wig_dict["chrM"][last_bin]

# write the wiggle track file
with open(header + ".wig", "w") as outfile:
    outfile.write("track type=wiggle_0 visibility=full name=" + header + " description=" + header + " color=" + color_code +  " maxHeightPixels=128:40:11  group=\"user\" graphType=bar priority=100 viewLimits=0:2.8 autoscale=on\n")
    for chrom in wig_dict:
        outfile.write("variableStep chrom=" + chrom + " span=" + str(bin_size) + "\n")

        for bin in wig_dict[chrom]:
            bin_name = bin*50
            if log10 == "y":
                outfile.write(str(bin_name) + "\t" + str(math.log10(wig_dict[chrom][bin])/rpm) + "\n")
            else:
                outfile.write(str(bin_name) + "\t" + str(wig_dict[chrom][bin]/rpm) + "\n")      
```

The .wig files can be uploaded to the [UCSC Genome Browser](https://genome.ucsc.edu/cgi-bin/hgCustom?hgsid=1804009504_57IfGr7IsksWm32NIHRQkaBNJSFc) for visualization. 

## FeatureCounts to find read counts

You can use any annotation file to get specific counts for genes, lncRNA, whatever your heart desires! Make sure you module load subread before running FeatureCounts! 

```
#!/bin/bash
## Example with GRChg38 reference
sbatch -t 4:00:00 --wrap="featureCounts -p --countReadPairs -a gencode.v38.long_noncoding_RNAs.gtf -o /your/path/SRR8615934_hg_Aligned.out.bam.txt /proj/brunklab/users/emma/strands/SRR8615934_hg_Aligned.out.bam"
```

The output should be text files with read count info in the last column :) have fun with that!


## Acknowledgements

Written by Emma Klein. The pipeline was constructed with the help of Quinn Eberhard.



