# covid-scfv-publication

## Table of Contents
- [Purpose](#purpose)
- [Scope](#scope)
- [Requirements](#requirements)
- [Tools](#tools)
- [Reproducing this analysis](#reproducing-this-analysis)
- [Citing this work](#citing-this-work)
- [Licensing](#licensing)
- [Help](#help)
- [Analysis](#analysis)
  - [Preparation](#preparation)
    - [Connect to the machine](#connect-to-the-machine-optional)
    - [Prepare the environment](#prepare-the-environment-use-sudo-if-non-root)
    - [Obtain raw data](#obtain-the-raw-data-for-analysis-example-with-rclone-from-s3)
    - [Copy IgBlast references](#copy-in-the-igblast-reference-files)
    - [Download Docker image](#download-the-docker-image-private---you-need-to-be-logged--authorized-on-dockerhub)
    - [Run Docker container](#run-a-docker-container)
    - [Copy references inside container](#copy-the-custom-reference-files-to-paths-required-by-igblast-within-the-container)
  - [scFv analysis](#scfv-analysis)
    - [1. Data preprocessing](#1-data-preprocessing---from-fastq-to-fasta)
    - [2. SEGUID and library merging](#2-seguid-library-merging-redundancy-removal-and-focus_libs-creation)
    - [3. V(D)J alignment](#3-vdj-alignment)
    - [4. scFv delimitation](#4-scfv-delimitation-and-characterization)
    - [5. Linker analysis](#5-generating-a-linker-consensus-scoring-linkers-and-full-dataset-overview)
      - [Linker overview](#linker-overview---inspect-visually)
    - [6. scFv flags](#6-scfv-flags)
    - [7. Per-library read counts](#7-recovering-per-library-read-counts)
    - [8. Generate an attrition table](#8-generate-an-attrition-table)
  - [Post-processing](#post-processing)


## Purpose

This repository contains instructions for reproducing the work from an upcoming research manuscript. This study describes the characterization of scFv antibody fragments obtained through phage display selection against viral epitopes followed by Oxford Nanopore long-read sequencing.

## Scope

The repository currently contains:
- code and instructions for scFv extraction
- custom species reference files for IgBlast ([description in progress]())

What will be progressively added:
- extensive data analysis in `R` 
- pipeline automation with Snakemake/Nextflow

## Requirements

The analysis can be very RAM-intensive - see [step 3](#3-vdj-alignment) to estimate your requirements! If the input dataset is large (more than a few samples), using an HPC or a powerful remote VM is recommended. 

Docker is required, but it is not allowed on some HPC systems due to security restrictions. In theory a `Docker image` can be converted for use with `Apptainer` - if you can make it work, I'd love to talk.

This pipeline assumes high-quality, pre-processed FASTQ files from long-read sequencing are provided - see [step 1](#1-data-preprocessing---from-fastq-to-fasta) for details. If you have already converted them to FASTA, you can analyze FASTA files by skipping a few steps.

## Tools

The first step of the analysis -- scFv extraction from sequencing files -- is based on [Seq2scfv](https://github.com/ngs-ai-org/seq2scfv), published by [Salvy et al. (2024)](https://www.tandfonline.com/doi/full/10.1080/19420862.2024.2408344). This toolkit required upgrades and refinements which I maintain in [this seq2scfv fork](https://github.com/MikolajKocikowski/seq2scfv-unofficial-updated). This repository was the source for building the Docker image that we will use below.

## Reproducing this analysis

To reproduce the exact analysis form this study, download the raw data from:
- planned: Zenodo (pending approval of coauthors) 
- currently: DigitalOcean S3 bucket (contact me for access keys)

You might choose to download from `DO` with `Rclone` (elaborated later below), with this configuration:

```ini
name: "stage"
bucket: "data-stage"

type = s3
provider = DigitalOcean
access_key_id = <readonly-key> # contact me
secret_access_key = <readonly-secret> # contact me
region = ams3
endpoint = ams3.digitaloceanspaces.com
ACL = private
```
The analysis was run on Ubuntu 24.04 (LTS) x64.

## Help

Please reach out with any problems running the pipeline -- I will try to help. I'm also happy to gather constructive feedback. For complete descriptions of each step please consult materials linked above, while keeping in mind the code and instructions presented here are not identical to those from original `Seq2scfv`

## Licensing

While this repository is released under the MIT License, a part of the analysis relies on [Seq2scfv](https://github.com/ngs-ai-org/seq2scfv) toolkit, published under a **limited license**, which allows **non-profit research use only**. Users are responsible for complying with the terms of that license.

## Citing this Work

If you use this code, scripts, or data in your research, we’d love to hear from you! Please also cite this work. Once the manuscript is out, we will provide a DOI. For now, you can cite it as:  

> M. Kocikowski, *covid-scfv-publication* (GitHub repository), 2026. Available at: https://github.com/MikolajKocikowski/covid-scfv-publication

# Analysis

## Preparation

### Connect to the machine (optional)

```bash
# connect
ssh root@123.456.78.910

# Tmux saves lives!
tmux
```

### Prepare the environment (use `sudo` if non-root)

Note: when analyzing sequences originating from any custom species (almost all non-human), it is critical to obtain `IgBlast` genetic reference files for `step 3`. They must be complete, appropriately formated, species-specific and available in the paths described below.

```bash
# Docker is necessary - install if not present
apt install docker.io

# prepare the key directories
mkdir -p analysis_folder/raw_data
```

### Obtain the raw data for analysis (example with Rclone import from S3)

```bash
# Rclone is optional - for file transfer
apt install rclone

# configure an Rclone remote for secure transfer
rclone config

# test connection: list directories on remote
rclone lsd stage:data-stage

# copy raw data; checkers stage data and are lighter than transfers
rclone copy stage:data-stage/data/ ~/analysis_folder/ --progress --verbose --transfers 8 --checkers 16

# check the transfer: compares sizes and hashes in the source and destination, reports mismatches
rclone check stage:data-stage/data/ ~/analysis_folder/

# copy the FASTQ files for analysis; if you have FASTA, you will just skip some steps later
mv analysis_folder/*.fastq analysis_folder/raw_data/
```

### Copy in the IgBlast reference files

```bash
# clone the GitHub repo with IgBlast custom reference files - or generate them yourself
git clone https://github.com/MikolajKocikowski/covid-scfv-publication.git

# copy the `for_igblast` dir to `analysis_folder`
cp -r covid-scfv-publication/for_igblast analysis_folder/
```

### Download the docker image (private - you need to be logged AND authorized on DockerHub)

```bash
# authenticate to the DockerHub
docker login

# pull the image
docker pull skogsv/seq2scfv-unofficial-updated:1.1
```

### Run a docker container

Create a container from the docker image, with read/write access to the analysis files.

```bash
docker run -it --name seq2scfv_container \
  -v ~/analysis_folder:/analysis:rw \
  skogsv/seq2scfv-unofficial-updated:1.1 \
  bash
```

We will use the somewhat confusingly named `/igblast/` directory for internal processing, while input and output files are stored in `/analysis/`, which maps to the local `analysis_folder` on disk. This ensures that outputs persist if the container is stopped or terminated, and allows easy access to the results from outside the container for immediate inspection.

### Copy the custom reference files to paths required by IgBlast within the container

```bash
# blast sequence database for germline
mkdir ncbi-igblast-1.20.0/databases
cp -r /analysis/for_igblast/databases/dog ncbi-igblast-1.20.0/databases/

# auxiliary file for CDR3/FWR4 information
cp -r /analysis/for_igblast/optional_file/dog_gl.aux ncbi-igblast-1.20.0/optional_file/

# germline V gene annotation file
cp -r /analysis/for_igblast/internal_data/dog ncbi-igblast-1.20.0/internal_data/
```

## ScFv Analysis

### 1. Data preprocessing - from fastq to fasta

This code assumes `raw_data` contains high-quality FASTQ files, that were appropriately trimmed, filtered, reverse-complented, size-selected etc. if needed and as preferred. The following step converts them to FASTA format as required by the pipeline. If running on a personal computer, consider checking the number of available cores with `nproc` and limiting `parallel` with `-j maxjobs` lower than that. 

```bash
# create a dir for fasta files
mkdir /analysis/1.Preprocessed

# get parallel (or run the code below as a loop instead) 
apt update && apt install -y parallel

# convert fastq to fasta
parallel 'seqkit fq2fa {} -o /analysis/1.Preprocessed/{/.}.fasta' ::: /analysis/raw_data/*.fastq
```

### 2. SEGUID, library merging, redundancy removal and focus_libs creation

Customize the `remove_extension=` and `files=` depending on your files.

```bash
# define input dir
input_dir="/analysis/1.Preprocessed"

# customize the extension to remove from basename (say, sample.fasta vs sample.crazy.program.made.fasta)
remove_extension=".fasta"

# collect the files you want analyzed
files=("${input_dir}"/*.fasta)

# create or clear (if exists) this output file needed for pipeline step 7
> /analysis/focus_libs.txt

# process the samples and populate focus_libs.txt
for sample in "${files[@]}"; do
  basename=$(basename "$sample" "$remove_extension")
  uniq_id.py "$basename" "$sample" || echo "ERROR processing $sample"
  echo "$basename" >> /analysis/focus_libs.txt
done

# combine all correspondence files into one
cat *correspondence.tsv > merged.nt_correspondence.tsv

# remove duplicate sequences from merged SEGUID fasta files using seqkit
seqkit rmdup -n < <(cat *.nt_seguid.fa) > merged.nt_seguid.fa

# organize the outputs into a new directory
mkdir -p /analysis/2.Catalogued
mv *correspondence* *seguid* /analysis/2.Catalogued/
```

### 3. V(D)J alignment

**Important:** this is the longest and most resource-hungry part. 

`vscan_functions.py` runs several analysis iterations, first one being the longest, and taking approximately 1-2 CPU-hours / 1GB of starting data (FASTQ). Hence adjust the threads in the `ig_threads` parameter based on availability. If unsure, leave some CPUs free to avoid crashing the system. Too many threads compared to available RAM might also cause an Out Of Memory (OOM) killing of the process - reduce threads. 

**HOWEVER the RAM use peaks** at the end of the first iteration (which might a few hours into the runtime), when Pandas reads in the entire output. This easily causes an OOM crash with big datasets. As a rule of thumb, aim to have RAM available 2x the size of starting data. In my experience, starting with 115 GB of FASTQs, 96GB RAM was not enough, but 250GB RAM was enough to complete the analysis. 

```bash
# user input
fasta="/analysis/2.Catalogued/merged.nt_seguid.fa"
ig_seqtype="Ig"
ig_threads="14"
organism="dog"
domain_system="imgt"
minlen="150"
escore="0.01"
# path to your germline genes databases
germline_db_V="/igblast/ncbi-igblast-1.20.0/databases/dog/dog_V"
germline_db_D="/igblast/ncbi-igblast-1.20.0/databases/dog/dog_D"
germline_db_J="/igblast/ncbi-igblast-1.20.0/databases/dog/dog_J"
auxiliary_data="/igblast/ncbi-igblast-1.20.0/optional_file/dog_gl.aux"

vscan_functions.py --query $fasta \
      --germline_V $germline_db_V \
      --germline_D $germline_db_D \
      --germline_J $germline_db_J \
      --organism $organism  \
      --ig_seqtype $ig_seqtype  \
      --domain_system $domain_system \
      --auxiliary_data $auxiliary_data \
      --num_threads $ig_threads \
      --minlen $minlen \
      --escore $escore

# organize the output
mkdir /analysis/3.vscan
mv igBLAST.tsv /analysis/3.vscan/

rm ddDNA*tsv ddDNA*fasta # optional cleanup

```

If `ddDNA_igblast_1.tsv` is empty or wasn't created in the output, sth went catastrophically wrong. If `igBLAST.tsv` is missing, you probably had an OOM process kill due to insufficient RAM, investigate kernel messages with `journalctl -k | tail -50` or `dmesg | grep -i kill`. If `ddDNA_splitfasta_1.fasta` is empty and `igBLAST.tsv` only contains a header, you probably misplaced or misformated the IgBlast reference files.

### 4. scFv delimitation and characterization

FYI: `scFv_splitter.py` uses `AntPack`, and AntPack ≥0.3.9 requires a license key. Hence, I pinned `antpack==0.3.8.5` in the current toolkit's Docker image (1.1) for a reproducible and democratic pipeline. Also, expect syntax warnings from this step.

```bash
# user input
hits="/analysis/3.vscan/igBLAST.tsv"
domain_system="imgt"

scFv_splitter.py --igBLAST-hits $hits --domain-type $domain_system 

# organize output
mkdir /analysis/4.scFv_split
mv *fasta *tsv /analysis/4.scFv_split/
```

### 5. Generating a linker consensus, scoring linkers and full dataset overview

If known, provide the aa linker sequence for `--check-linker` to compare it to inferred consensus linker. 
If not known, state 'undefined' and the consensus linker will be determined as reference. Expect syntax warnings from this step.

```bash
# consensus linker and scoring
aalink="/analysis/4.scFv_split/aa_inferred_linkers.fasta"
ntlink="/analysis/4.scFv_split/nt_inferred_linkers.fasta"
scfvs="/analysis/4.scFv_split/in_frame_igBLAST_paired_delim.tsv"
linksample=0.1

cat ${ntlink} | seqkit sample -p $linksample -o nt_inferred_linkers_${linksample}_sampled.fasta
clustalo --use-kimura -i nt_inferred_linkers_${linksample}_sampled.fasta -o nt_inferred_linkers.aln  

cat ${aalink} | seqkit sample -p $linksample -o aa_inferred_linkers_${linksample}_sampled.fasta
clustalo --use-kimura -i aa_inferred_linkers_${linksample}_sampled.fasta -o aa_inferred_linkers.aln  

# If the linker is "undefined", get_logos.py will write the consensus linker to `aa_reference_linker.fasta` used in the next step.
get_logos.py --nt-aln nt_inferred_linkers.aln --aa-aln aa_inferred_linkers.aln --check-linker 'undefined'

linker_scorer.py --scFv $scfvs --linker aa_reference_linker.fasta
```

#### Linker overview - inspect visually

The script produces - at both aa and nt level: a sequence logo of consensus linker (`*_linkers_logo.pdf`), a linker length distribution table (`*_inferred_linkers_length.tsv`) + a histogram (`*_inferred_linkers_length.png`), and a table with the 10 most frequently observed linker sequences (`*_top10_linkers.tsv`). The linker information is used to flag the quality of the scFv hits (but no automatic filtering), and can be used to infer the quality of the library, sequencing, and gene annotation.

```bash
# get linker lengths
echo -e "ID\tlength" > nt_inferred_linkers_lengths.tsv
bioawk -c fastx '{ print $name, length($seq) }' < ${ntlink} >> nt_inferred_linkers_lengths.tsv
echo -e "ID\tlength" > aa_inferred_linkers_lengths.tsv
bioawk -c fastx '{ print $name, length($seq) }' < ${aalink} >> aa_inferred_linkers_lengths.tsv
linklengths.r nt_inferred_linkers_lengths.tsv aa_inferred_linkers_lengths.tsv 

# most frequent linkers
top_linkers.r $scfvs aa_reference_linker.fasta

# organize output 
mkdir /analysis/5.Linkers
mv inferred_vs_provided_linker_evaluation.txt *tsv *fasta *aln *pdf *png /analysis/5.Linkers/
```

### 6. scFv flags

```bash
linkerscored="/analysis/5.Linkers/in_frame_igBLAST_paired_delim_linker_scored.tsv"
pcntlinkID="90"
maxmismatches="2"
maxlinkeroverhang="undefined"
chain_min_aa_len="80"

scfv_flags.py --scFv ${linkerscored} --linker-min-id $pcntlinkID --mismatches $maxmismatches --linker-overhang $maxlinkeroverhang --chain-length $chain_min_aa_len

# organize output
mkdir /analysis/6.Flags
mv in_frame_igBLAST_paired_delim_linker_scored_flags.tsv /analysis/6.Flags/
```

### 7. Recovering per-library read counts

```bash
# recover counts
scFv="/analysis/6.Flags/in_frame_igBLAST_paired_delim_linker_scored_flags.tsv"
corr="/analysis/2.Catalogued/merged.nt_correspondence.tsv"
libs="/analysis/focus_libs.txt"

ntqual_counts.py --scFv $scFv --correspondence $corr --focus-libraries $libs

# organize output
mkdir /analysis/7.Counts                 
mv in_frame_igBLAST_paired_delim_linker_scored_flags_counts.tsv /analysis/7.Counts/
```

If you inspect the output and for a given scFv there are 0 counts across all samples, it means there is a mismatch between sample names in `focus_libs.txt` and the other input files. This is however unlikely with the current automated generation of `focus_libs.txt`.

### 8. Generate an attrition table

The resultant table summarizes sequence counts at each major step of the pipeline, showing attrition from raw reads to fully validated scFvs.

Run this from outside container (`exit` if needed). Create a script by running `nano attrition_table.sh`.
Paste the code privided below. Deploy: `bash attrition_table.sh > attrition_table.tsv`. Print the results nicely formated: `cat attrition_table.tsv | column -t -s $'\t'`.

```bash
#!/bin/bash

# Explicit order
labels=(
  "Merged libraries"
  "De-duplicated reads"
  "Ig domains"
  "Non-paired / multi VH/VL"
  "Out-of-frame scFvs"
  "Non-standard AAs"
  "Validated scFvs"
)

declare -A files=(
  ["Merged libraries"]="analysis_folder/2.Catalogued/merged.nt_correspondence.tsv"
  ["De-duplicated reads"]="analysis_folder/2.Catalogued/merged.nt_seguid.fa"
  ["Ig domains"]="analysis_folder/3.vscan/igBLAST.tsv"
  ["Validated scFvs"]="analysis_folder/4.scFv_split/in_frame_igBLAST_paired_delim.tsv"
  ["Non-standard AAs"]="analysis_folder/4.scFv_split/in_frame_igBLAST_delim_nonstandard_aas.tsv"
  ["Out-of-frame scFvs"]="analysis_folder/4.scFv_split/out_of_frame_igBLAST_paired_delim.tsv"
  ["Non-paired / multi VH/VL"]="analysis_folder/4.scFv_split/igBLAST_non_paired_nondelim.tsv"
)

# Print header
echo -e "Category\tCount"

# Loop over labels in desired order
for label in "${labels[@]}"; do
    file="${files[$label]}"
    if [[ "$file" == *.tsv ]]; then
        count=$(($(wc -l < "$file") - 1))
    elif [[ "$file" == *.fa ]]; then
        count=$(grep -c '^>' "$file")
    else
        count="N/A"
    fi
    echo -e "${label}\t${count}"
done
```
Example output
```
Category                  Count
Merged libraries          1629771
De-duplicated reads       1629731
Ig domains                539480
Non-paired / multi VH/VL  372322
Out-of-frame scFvs        79938
Non-standard AAs          0
Validated scFvs           3641
```

The sequences from following categories are not included in the final results file: 
- `Non-paired / multi VH/VL` -- unpaired sequences, or sequences with more than 1 VH or VL
- `Out-of-frame scFvs` -- sequences with stop codons in the scFv, or incongruent frames between VH and VL
- `Non-standard AAs` -- paired VH-VL sequences presenting non-standard aminoacids

The final output file contains additional quality scores based on flagging settings from steps 5-6, but these are not used to filter any sequence out, only made available to the user.

## Post-processing

Download the key output file (or all) and process it in R. 

```bash
# example download from server - run from the personal computer: 
scp root@123.456.78.901:/root/analysis_folder/7.Counts/in_frame_igBLAST_paired_delim_linker_scored_flags_counts.tsv ~/Downloads/

```