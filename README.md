# covid-scfv-publication

## Table of Contents
- [Purpose](#purpose)
- [Scope](#scope)
- [Requirements](#requirements)
- [Tools](#tools)
- [Reproducing this analysis](#reproducing-this-analysis)
- [Citing this Work](#citing-this-work)
- [Licensing](#licensing)
- [Help](#help)
- [Analysis](#analysis)
  - [Steps](#steps)

## Purpose

This repository contains instructions for reproducing the work from an upcoming research manuscript. This study describes the characterization of scFv antibody fragments selected through phage display against viral and control epitopes.

## Scope

The repository currently contains:
- code and instructions for scFv extraction
- custom species reference files for IgBlast (description in progress: [future link]())

What will be progressively added:
- extensive data analysis in `R` 
- pipeline automation with Snakemake/Nextflow (for now please run manually and verify output)

## Requirements

The script can be very RAM-intensive - see [step 3](#3-vdj-alignment) to estimate your requirements! If the input dataset is large (more than a few samples), using an HPC or a powerful remote VM is recommended. Note that Docker is required, but it is not allowed on some HPC systems due to security restrictions. In theory a `Docker image` can be converted for use with `Apptainer` - if you can make it work, I'd love to talk.

## Tools

The first step of the analysis -- scFv extraction from sequencing files -- is based on [Seq2scfv](https://github.com/ngs-ai-org/seq2scfv), published by [Salvy et al. (2024)](https://www.tandfonline.com/doi/full/10.1080/19420862.2024.2408344). This toolkit required fixes and upgrades which I maintain in [my seq2scfv fork](https://github.com/MikolajKocikowski/seq2scfv-unofficial-updated). This repository was the source for building the Docker image that we will use below.

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

We welcome feedback, suggestions, or contributions — feel free to open an issue or pull request!

# Analysis

## Steps...

