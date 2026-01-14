# covid-scfv-publication

## Table of Contents
- [Purpose](#purpose)
- [Dependencies](#dependencies)
- [Licensing](#licensing)
- [Help](#help)
- [Scope](#scope)
- [Analysis](#analysis)
  - [Steps](#steps)


## Purpose

This repository contains instructions for reproducing the work from an upcoming research manuscript. This study describes the characterization of scFv antibody fragments selected through phage display against viral and control epitopes.

## Dependencies

The first step of the analysis -- scFv extraction from sequencing files -- is based on [Seq2scfv](https://github.com/ngs-ai-org/seq2scfv), published by [Salvy et al. (2024)](https://www.tandfonline.com/doi/full/10.1080/19420862.2024.2408344). This toolkit, to function, required fixes and upgrades which I maintain in [my seq2scfv fork](https://github.com/MikolajKocikowski/seq2scfv-unofficial-updated). This repository was the source for building the Docker image that we will use below.

## Licensing

While this repository is released under the MIT License, a part of this analysis relies on modified [Seq2scfv](https://github.com/ngs-ai-org/seq2scfv) software, which is published under a **Limited License**, restricting its use to **non-profit research purposes**. Users are responsible for complying with the terms of that license.

## Help

Please reach out with any problems running the pipeline - I will try to help and am happy for constructive feedback! For complete descriptions of each step please consult materials above, while keeping in mind the code and instructions presented here are not identical to the original `Seq2scfv`

## Scope

The repository currently contains:
- instructions for scFv extraction
- custom species reference files for IgBlast (description in progress [future link])

What will be progressively added:
- extensive data analysis in `R` 

# Analysis

## Steps...

