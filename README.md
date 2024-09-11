# Bioinformatics_Diploma_NGS_Course_EBI_Pipeline
Sure, I’ve revised the `README.md` file to ensure clarity and consistency. Here’s an improved version:

```markdown
# EBI Pipeline

This repository provides a comprehensive pipeline for processing sequencing data using various bioinformatics tools. Follow the steps below to successfully download, process, and visualize your sequencing data.

## Prerequisites

Ensure you have the following tools installed:
- `sratoolkit`
- `SeqPrep`
- `fastqc`
- `trimmomatic`
- `seqkit`
- `infernal` (for CM search)
- `conda` (for managing environments and installing packages)

## Setup

### 1. Download and Extract sratoolkit

```bash
conda activate NGS
wget https://ftp-trace.ncbi.nlm.nih.gov/sra/sdk/3.1.1/sratoolkit.3.1.1-ubuntu64.tar.gz
tar -xzf sratoolkit.3.1.1-ubuntu64.tar.gz
```

### 2. Download SeqPrep

```bash
wget https://github.com/jstjohn/SeqPrep/archive/refs/heads/master.zip -O SeqPrep-master.zip
unzip SeqPrep-master.zip
cd SeqPrep-master
make
```

## Pipeline Steps

### 1. Download the Sample Using Prefetch

```bash
sratoolkit.3.1.1-ubuntu64/bin/prefetch SRR5903765
```

### 2. Split Sample into FASTQ Files Using Fasterq-Dump

```bash
sratoolkit.3.1.1-ubuntu64/bin/fasterq-dump SRR5903765 --split-files
```

### 3. Merge Reads Using SeqPrep

```bash
SeqPrep -f SRR5903765_1.fastq -r SRR5903765_2.fastq \
    -1 unmerged_subset_SRR5903765_1.fastq -2 unmerged_subset_SRR5903765_2.fastq \
    -A GATCGGAAGAGCACACG -B AGATCGGAAGAGCGTCGT -s subset_SRR5903765_merged.fastq.gz
```

### 4. Quality Check Before Trimming Using FastQC

```bash
fastqc subset_SRR5903765_merged.fastq.gz
```

### 5. Trim Reads Using Trimmomatic

```bash
# Note: Download the adaptor_file
conda install -c bioconda trimmomatic
trimmomatic SE -phred33 subset_SRR5903765_merged.fastq.gz trimmomatics_output.fq.gz \
    ILLUMINACLIP:adaptor.fa:2:30:10 LEADING:3 TRAILING:3 SLIDINGWINDOW:4:15 MINLEN:36
```

### 6. Quality Check After Trimming Using FastQC

```bash
fastqc trimmomatics_output.fq.gz
```

### 7. Convert FASTQ to FASTA Using SeqKit

```bash
seqkit fq2fa trimmomatics_output.fq.gz > fa_SRR5903765_trimmomatic_output.fasta
```

### 8. Download Reference Databases

```bash
mkdir ribosomes
wget "ftp://ftp.ebi.ac.uk/pub/databases/metagenomics/pipeline-5.0/ref-dbs/rfam_models/other_models/*.cm" -P ribosomes
```

### 9. Perform CM Search

```bash
mkdir samples_765_VS_ribosome
for i in ribosomes/*.cm; do
  cmsearch --tblout samples_765_VS_ribosome/${i#ribosomes/}.tbl $i fa_SRR5903765_trimmomatic_output.fasta
done
```

### 10. Create BED File

```bash
# Create a BED file from CM search results
for tbl_file in samples_765_VS_ribosome/*.tbl; do
  awk '/^[^#]/ {if ($8 > $9) {print $1 "\t" $9-1 "\t" $8} else {print $1 "\t" $8-1 "\t" $9}}' "$tbl_file" >> SRR5903765_combined.bed
done
```

### 11. Mask and Extract FASTA Sequences

```bash
# Mask sequences using maskfasta
bedtools maskfasta -fi fa_SRR5903765_trimmomatic_output.fasta -bed SRR5903765_combined.bed -fo SRR5903765_maskfasta.fasta

# Extract sequences using getfasta
bedtools getfasta -fi fa_SRR5903765_trimmomatic_output.fasta -bed SRR5903765_combined.bed -fo SRR5903765_getfasta.fasta
```

### 12. Download SILVA Databases

```bash
# Download SILVA databases
wget ftp://ftp.arb-silva.de/release_138/Exports/SILVA_138_SSURef_NR99_tax_silva.fasta
wget ftp://ftp.arb-silva.de/release_138/Exports/SILVA_138_LSURef_NR99_tax_silva.fasta
```

### 13. Map Sequences Using `mapseq`

```bash
# Map sequences to LSU database
mapseq SRR5903765_getfasta.fasta SILVA_138_LSURef_NR99_tax_silva.fasta slv_lsu_filtered2.txt > SRR5903765_lsu_mapseq_output.txt

# Map sequences to SSU database
mapseq SRR5903765_getfasta.fasta SILVA_138_SSURef_NR99_tax_silva.fasta slv_ssu_filtered2.txt > SRR5903765_ssu_mapseq_output.txt
```

### 14. Generate OTU Files

```bash
# Generate OTU file from SSU mapping results
mapseq -otucounts SRR5903765_ssu_mapseq_output.txt > SRR5903765_ssu.otu
awk 'NR>1 {gsub(/;/, "\t", $3); print $4 "\t" $3}' SRR5903765_ssu.otu > SRR5903765_ssu.txt

# Generate OTU file from LSU mapping results
mapseq -otucounts SRR5903765_lsu_mapseq_output.txt > SRR5903765_lsu.otu
awk 'NR>1 {gsub(/;/, "\t", $3); print $4 "\t" $3}' SRR5903765_lsu.otu > SRR5903765_lsu.txt
```

### 15. Visualize Data

```bash
# Create Krona charts
ktImportText SRR5903765_ssu.txt -o SRR5903765_ssu_output.html
ktImportText SRR5903765_lsu.txt -o SRR5903765_lsu_output.html
```
```

This revised version includes improved formatting and consistency, making the steps clearer and more organized. If you have any additional details or specific instructions to include, let me know!
