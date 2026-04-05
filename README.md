# Vertebrate-genome-assembly-using-HiFi-Bionano-and-Hi-C
De novo vertebrate genome assembly using PacBio HiFi, Bionano optical maps, and Hi-C data following the VGP-Galaxy pipeline — includes k-mer profiling, hifiasm assembly, purging, and chromosome-level scaffolding.

> A step-by-step implementation of the [VGP Galaxy Tutorial](https://training.galaxyproject.org/training-material/topics/assembly/tutorials/vgp_genome_assembly/tutorial.html) for high-quality vertebrate genome assembly.

---

## 📋 Table of Contents

- [Overview](#overview)
- [Background](#background)
- [Requirements](#requirements)
- [Dataset](#dataset)
- [Pipeline Overview](#pipeline-overview)
- [Step-by-Step Workflow](#step-by-step-workflow)
  - [Step 1: Data Preparation](#step-1-data-preparation)
  - [Step 2: HiFi Reads Preprocessing](#step-2-hifi-reads-preprocessing)
  - [Step 3: Genome Profile Analysis](#step-3-genome-profile-analysis)
  - [Step 4: Assembly with hifiasm](#step-4-assembly-with-hifiasm)
  - [Step 5: Purging False Duplications](#step-5-purging-false-duplications)
  - [Step 6: Scaffolding with Bionano](#step-6-scaffolding-with-bionano)
  - [Step 7: Hi-C Scaffolding](#step-7-hi-c-scaffolding)
  - [Step 8: Assembly Evaluation](#step-8-assembly-evaluation)
- [Tools Used](#tools-used)
- [Key Terminology](#key-terminology)
- [Results](#results)
- [References](#references)
- [Acknowledgements](#acknowledgements)

---

## Overview

This repository documents the completion of the **Vertebrate Genome Project (VGP) genome assembly tutorial** from the Galaxy Training Network (GTN). The goal is to produce a high-quality, near-error-free, chromosome-level, haplotype-phased genome assembly using a combination of:

- **PacBio HiFi** long reads (high accuracy, 10–25 kbp)
- **Bionano** optical maps (for scaffolding)
- **Hi-C** chromatin conformation data (for chromosome-level scaffolding)

**Tutorial source:** [GTN VGP Assembly Tutorial](https://training.galaxyproject.org/training-material/topics/assembly/tutorials/vgp_genome_assembly/tutorial.html)  
**Platform:** [Galaxy Project](https://usegalaxy.org) (UseGalaxy.org)  
**Estimated time:** ~5 hours  
**Difficulty level:** Intermediate

---

## Background

Advances in third-generation sequencing (TGS), particularly PacBio HiFi technology, have transformed *de novo* genome assembly. HiFi reads combine:
- Long read lengths (~10–25 kbp)
- High accuracy (>Q20, ≥99%)
- No amplification bias

The **Vertebrate Genomes Project (VGP)**, launched by the Genome 10K (G10K) consortium, aims to generate reference-quality genome assemblies for every vertebrate species. This tutorial follows the VGP assembly pipeline, addressing two major challenges in vertebrate genome assembly:

1. **Repetitive content** — transposable elements and tandem repeats that complicate reconstruction
2. **Heterozygosity** — haplotype phasing required for diploid genomes

---

## Requirements

### Prerequisites

Before starting, you should be familiar with:
- [Introduction to Galaxy Analyses](https://training.galaxyproject.org/training-material/topics/introduction)
- Basic concepts of genome assembly and sequencing
- [Quality Control in sequencing data](https://training.galaxyproject.org/training-material/topics/sequence-analysis/tutorials/quality-control/tutorial.html)

### Galaxy Account

A Galaxy account is required. This tutorial was run on:
- **Instance:** [UseGalaxy.org](https://usegalaxy.org/)

### Recommended Sequencing Coverage

| Data Type | Minimum Coverage |
|-----------|-----------------|
| PacBio HiFi | 30× (up to 60× for repetitive regions) |
| Hi-C | ~60× |

---

## Dataset

All datasets used in this tutorial are publicly available on Zenodo:

📦 **Zenodo:** [https://zenodo.org/record/5887339](https://zenodo.org/record/5887339)

| File Type | Description |
|-----------|-------------|
| `.fasta` | Reference/genome FASTA files |
| `.fastq.gz` | Compressed HiFi read files |
| Bionano `.cmap` | Optical map data |
| Hi-C `.fastq.gz` | Hi-C paired-end reads |

---

## Pipeline Overview

The VGP pipeline is modular, consisting of up to 10 workflows. This tutorial follows **Analysis Trajectory D** (HiFi + Hi-C + BioNano):

```
Input Data
    │
    ▼
[WF1] K-mer Profiling (Meryl)
    │
    ▼
[WF3/4] Contig Assembly (hifiasm — HiFi-only or Hi-C phased)
    │
    ▼
[WF6] Purging False Duplications (purge_dups)  ← Optional
    │
    ▼
[WF7] Bionano Scaffolding (Bionano Solve)
    │
    ▼
[WF8] Hi-C Scaffolding (YaHS)
    │
    ▼
[WF9] Decontamination
    │
    ▼
Final Chromosome-Level Assembly
```

| Input Combination | Analysis Trajectory |
|-------------------|-------------------|
| HiFi only | A |
| HiFi + Hi-C | B |
| HiFi + BioNano | C |
| **HiFi + Hi-C + BioNano** | **D (this tutorial)** |
| HiFi + parental data | E |
| HiFi + parental + Hi-C | F |
| HiFi + parental + BioNano | G |
| HiFi + parental + Hi-C + BioNano | H |

---

## Step-by-Step Workflow

### Step 1: Data Preparation

**Objective:** Upload and organize all input data in Galaxy before running the pipeline.

1. Log into [UseGalaxy.org](https://usegalaxy.org/)
2. Create a new History and name it (e.g., `VGP Assembly`)
   
#### Datasets

All files are publicly available on Zenodo. Upload the following directly into Galaxy using **Upload Data → Paste/Fetch Data**:

**HiFi Reads (FASTA)**

| File | Zenodo URL |
|------|-----------|
| HiFi_synthetic_50x_01.fasta | https://zenodo.org/record/6098306/files/HiFi_synthetic_50x_01.fasta |
| HiFi_synthetic_50x_02.fasta | https://zenodo.org/record/6098306/files/HiFi_synthetic_50x_02.fasta |
| HiFi_synthetic_50x_03.fasta | https://zenodo.org/record/6098306/files/HiFi_synthetic_50x_03.fasta |

> Set datatype to `fasta` for all three files.

**Hi-C Reads (FASTQ)**

| File | Zenodo URL | Rename To | Description |
|------|-----------|-----------|-------------|
| SRR7126301_1.fastq.gz | https://zenodo.org/record/5550653/files/SRR7126301_1.fastq.gz | `Hi-C_dataset_F` | Forward reads |
| SRR7126301_2.fastq.gz | https://zenodo.org/record/5550653/files/SRR7126301_2.fastq.gz | `Hi-C_dataset_R` | Reverse reads |

> Set datatype to `fastqsanger.gz` for both Hi-C files.

#### Organizing Data in Galaxy

1. Upload all five files via **Upload Data → Paste/Fetch Data**, pasting the Zenodo URLs above.
2. After upload, **rename** the Hi-C files:
   - `SRR7126301_1.fastq.gz` → `Hi-C_dataset_F` (forward reads)
   - `SRR7126301_2.fastq.gz` → `Hi-C_dataset_R` (reverse reads)
3. Create a **Dataset Collection** from the three HiFi FASTA files:
   - Select all three `HiFi_synthetic_50x_*.fasta` files
   - Click **For all selected → Build Dataset List**
   - Name the collection: `HiFi_collection`
4. The Hi-C files (`Hi-C_dataset_F` and `Hi-C_dataset_R`) remain as individual datasets — they are used separately in the Hi-C scaffolding step.

#### Final Data Organization Summary

| Dataset | Type | Format | Used In |
|---------|------|--------|---------|
| `HiFi_collection` | Dataset Collection (list of 3) | `fasta` | Steps 2, 3, 4, 5 |
| `Hi-C_dataset_F` | Single dataset | `fastqsanger.gz` | Step 7 (Hi-C scaffolding) |
| `Hi-C_dataset_R` | Single dataset | `fastqsanger.gz` | Step 7 (Hi-C scaffolding) |

---

### Step 2: HiFi Reads Preprocessing
 
**Tool:** `Cutadapt`
**Objective:** Remove PacBio HiFi adapter sequences from reads. Reads that contain adapters are discarded entirely, as their presence indicates chimeric or incomplete reads that would degrade assembly quality.
 
#### Input
 
| Parameter | Value |
|-----------|-------|
| Single-end or Paired-end reads? | `Single-end` |
| FASTQ/A file | `HiFi_collection` (Dataset Collection) |
 
#### Read 1 Options — 5' or 3' (Anywhere) Adapters
 
Two custom adapters are provided. Using the **Anywhere** mode allows Cutadapt to detect and trim these sequences whether they appear at the 5' or 3' end of a read.
 
| # | Adapter Name | Adapter Sequence |
|---|-------------|-----------------|
| 1 | First adapter | `ATCTCTCTCAACAACAACAACGGAGGAGGAGGAAAAGAGAGAGAT` |
| 2 | Second adapter | `ATCTCTCTCTTTTCCTCCTCCTCCGTTGTTGTTGTTGAGAGAGAT` |
 
> These are the standard PacBio HiFi SMRTbell adapter sequences. The two adapters are reverse complements of each other.
 
#### Adapter Options
 
| Parameter | Value |
|-----------|-------|
| Maximum error rate | `0.1` (10% mismatches allowed during adapter matching) |
| Minimum overlap length | `35` bp |
| Look for adapters in the reverse complement | `Yes` |
 
#### Filter Options
 
| Parameter | Value |
|-----------|-------|
| Discard Trimmed Reads | `Yes` |
 
> **Why discard instead of trim?** In HiFi sequencing, the presence of an adapter mid-read indicates the read is a chimera (two molecules ligated together). Such reads are unreliable and should be removed entirely rather than trimmed, as even the retained portion may be erroneous.
 
#### Output
 
- A filtered collection of adapter-free HiFi reads, ready for k-mer profiling and assembly.
- Reads containing either adapter sequence are completely discarded.
 
---
### Step 3: Genome Profile Analysis

**Objective:** Estimate genome size, heterozygosity, and repeat content from k-mer
frequencies in raw HiFi reads before assembly.

Before assembly, it is useful to collect metrics on genome properties such as expected
genome size. Modern approaches use **k-mer frequency analysis** instead of experimental
methods like DNA flow cytometry. This provides information on genomic complexity and
data quality.

The workflow has three stages:
1. Count k-mers per file with **Meryl** (parallelized over the collection)
2. Merge per-file counts into one database with **Meryl**
3. Fit a model to the merged histogram with **GenomeScope2**

---

#### 3a. K-mer Counting with Meryl (Count)

**Tool:** `Meryl` 

Meryl decomposes reads into k-length substrings and counts each k-mer's frequency.
Since HiFi data is stored as a collection, a separate `count` job runs for each FASTA
file automatically, parallelizing the work.

| Parameter | Value |
|-----------|-------|
| Operation type selector | `Count operations` |
| Count operations | `Count: count the occurrences of canonical k-mers` |
| Input sequences | `HiFi_collection (trim)` *(Dataset Collection)* |
| k-mer size selector | `Set a k-mer size` |
| k-mer size | `31` |

> **Why k = 31?** Long enough that most k-mers are non-repetitive, short enough to
> tolerate sequencing errors. For genomes > 10 Gb or highly repetitive genomes, use
> a larger k.

**➜ Rename output:** `meryldb`

---

#### 3b. Merging K-mer Databases with Meryl (Union-Sum)

**Tool:** `Meryl` 

Merges per-file k-mer databases into a single database by summing counts across all files.

| Parameter | Value |
|-----------|-------|
| Operation type selector | `Operations on sets of k-mers` |
| Operations on sets of k-mers | `Union-sum: return k-mers that occur in any input, set the count to the sum of the counts` |
| Input meryldb | `meryldb` *(collection from previous step)* |

**➜ Rename output:** `Merged meryldb`

---

#### 3c. Generating the K-mer Histogram with Meryl

**Tool:** `Meryl` 

Generates a k-mer frequency histogram from the merged database for input into GenomeScope2.

| Parameter | Value |
|-----------|-------|
| Operation type selector | `Generate histogram dataset` |
| Input meryldb | `Merged meryldb` |

**➜ Rename output:** `meryldb histogram`

---

#### 3d. Genome Profiling with GenomeScope2

**Tool:** `GenomeScope` 

GenomeScope2 fits a mixture of negative binomial distributions to the k-mer histogram
using nonlinear least-squares optimization, estimating genome size, heterozygosity, and
repeat content.

| Parameter | Value |
|-----------|-------|
| Input histogram file | `meryldb histogram` |
| Ploidy | `2` |
| k-mer length | `31` |
| Output options → Summary | Checked |
| Advanced options → Create testing.tsv | `Yes` |

---

#### GenomeScope2 Outputs

**Plots:**

| Output | Description |
|--------|-------------|
| Linear plot | k-mer frequency vs. coverage |
| Log plot | Log-scale version of the linear plot |
| Transformed linear plot | Frequency × coverage vs. coverage (amplifies higher-order peaks) |
| Transformed log plot | Log-scale version of the transformed plot |

---

### Step 4: Assembly with hifiasm

**Tool:** `hifiasm`  
**Objective:** Assemble contigs from HiFi reads (with optional Hi-C phasing).

#### Assembly Modes

| Mode | Input | Output |
|------|-------|--------|
| HiFi-only | HiFi reads | Primary + Alternate assembly |
| Hi-C phased | HiFi + Hi-C reads | Hap1 + Hap2 assemblies |
| Pseudohaplotype | HiFi reads | Primary + Alternate (with purging) |

#### Recommended Settings (Hi-C phased mode)

| Parameter | Value |
|-----------|-------|
| Input HiFi reads | Preprocessed `.fastq.gz` |
| Hi-C R1 reads | Hi-C forward reads |
| Hi-C R2 reads | Hi-C reverse reads |
| Assembly mode | Hi-C phased |

**Outputs (GFA format → converted to FASTA):**
- `hap1.p_ctg.fa` — Haplotype 1 contigs
- `hap2.p_ctg.fa` — Haplotype 2 contigs

#### Assembly QC

**Tool:** `Merqury`  
Evaluates assembly completeness and accuracy using k-mers.

**Tool:** `BUSCO`  
Checks completeness against conserved single-copy orthologs.

**Tool:** `gfastats`  
Reports assembly statistics (N50, L50, contig count, total size).

---

### Step 5: Purging False Duplications

**Tool:** `purge_dups`  
**Objective:** Remove haplotypic duplications and overlaps from the primary assembly.

> **When to purge:** Inspect k-mer multiplicity plots from Merqury. If the primary assembly shows a bimodal distribution with a second peak at 2× coverage, purging is recommended.

**Steps:**
1. Align HiFi reads back to the primary assembly (`minimap2`)
2. Calculate read depth (`purge_dups: calcuts`)
3. Split assemblies at coverage gaps (`split_fa`)
4. Self-align contigs to detect overlaps (`minimap2 -x asm5`)
5. Purge duplications (`purge_dups`)

**Outputs:**
- `purged.fa` — Purged primary assembly
- `hap.fa` — Haplotigs moved to alternate

---

### Step 6: Scaffolding with Bionano

**Tool:** `Bionano Solve`  
**Objective:** Use optical maps to scaffold contigs into larger sequences.

**How it works:** Bionano optical maps provide long-range physical distance information (~1 Mb) by labeling specific enzyme recognition sites in high-molecular-weight DNA.

| Parameter | Value |
|-----------|-------|
| Assembly FASTA | Purged primary assembly |
| Bionano CMAP | Optical map file |
| Enzyme | DLE-1 (or appropriate enzyme) |

**Outputs:**
- Hybrid scaffolds (FASTA)
- Scaffolding statistics
- Conflict reports

**QC:** Compare N50, scaffold count, and total size before and after Bionano scaffolding.

---

### Step 7: Hi-C Scaffolding

**Tool:** `YaHS`  
**Objective:** Use Hi-C chromatin contact data to order and orient scaffolds into chromosome-level assemblies.

#### Pre-processing Hi-C Data

1. **Trim adapters** using `Trim Galore`
2. **Map reads** to assembly using `BWA-MEM2`:
   - Map R1 and R2 independently
   - Filter for high-quality alignments
3. **Filter and sort** BAM files
4. **Mark duplicates** using `Picard MarkDuplicates`

#### YaHS Scaffolding

| Parameter | Value |
|-----------|-------|
| Input assembly | Bionano-scaffolded FASTA |
| Hi-C BAM | Filtered, deduplicated BAM |
| Restriction enzyme | Arima (or HindIII/DpnII) |

#### Generate Hi-C Contact Maps

**Tool:** `PretextMap` → `PretextSnapshot`  
Visualize genome-wide Hi-C contact maps to assess scaffolding quality.

**Good scaffolding signs:**
- Clear diagonal blocks (chromosomes)
- Strong intra-chromosomal signal
- Minimal off-diagonal noise

---

### Step 8: Assembly Evaluation

**Objective:** Comprehensively assess the quality of the final assembly.

| Tool | Metric |
|------|--------|
| `gfastats` | N50, L50, contig/scaffold count, total length |
| `BUSCO` | Gene space completeness (%) |
| `Merqury` | k-mer completeness, QV score |
| `PretextMap` | Hi-C contact map visualization |
| `BlobTools` | Contamination screening |

#### Key Quality Metrics

| Metric | Description | Good Assembly Target |
|--------|-------------|---------------------|
| N50 | Length at which 50% of assembly is in contigs ≥ this length | Chromosome-scale |
| BUSCO completeness | % conserved genes found intact | > 95% |
| QV (Quality Value) | Phred-scaled error rate estimate | > Q40 |
| k-mer completeness | % of k-mers from reads found in assembly | > 95% |

---

## Tools Used

| Tool | Version | Purpose |
|------|---------|---------|
| [Cutadapt](https://cutadapt.readthedocs.io/) | Latest | Adapter trimming |
| [Meryl](https://github.com/marbl/meryl) | Latest | K-mer counting |
| [GenomeScope2](https://github.com/tbenavi1/genomescope2.0) | Latest | Genome profiling |
| [hifiasm](https://github.com/chhylp123/hifiasm) | Latest | HiFi contig assembly |
| [purge_dups](https://github.com/dfguan/purge_dups) | Latest | Duplicate purging |
| [Bionano Solve](https://bionano.com/software-downloads/) | Latest | Optical map scaffolding |
| [YaHS](https://github.com/c-zhou/yahs) | Latest | Hi-C scaffolding |
| [BWA-MEM2](https://github.com/bwa-mem2/bwa-mem2) | Latest | Hi-C read mapping |
| [BUSCO](https://busco.ezlab.org/) | Latest | Assembly completeness |
| [Merqury](https://github.com/marbl/merqury) | Latest | Reference-free QC |
| [PretextMap](https://github.com/wtsi-hpag/PretextMap) | Latest | Hi-C contact maps |
| [gfastats](https://github.com/vgl-hub/gfastats) | Latest | Assembly statistics |

---

## Key Terminology

| Term | Definition |
|------|-----------|
| **Contig** | A contiguous, gapless sequence assembled from reads |
| **Scaffold** | One or more contigs joined by gap (N) sequences using additional data |
| **N50** | The length at which 50% of the total assembly is contained in contigs of this size or larger |
| **Haplotype** | A set of DNA variations inherited together from one parent |
| **Heterozygosity** | Having two different alleles at a locus |
| **Pseudohaplotype assembly** | An assembly with long phased blocks separated by unresolved regions |
| **Primary assembly** | The more complete haplotype representation (includes homo- and heterozygous regions) |
| **Alternate assembly** | Contains alternate alleles not in the primary assembly |
| **Haplotypic duplication** | False duplication where both alleles of a heterozygous region are in the same assembly |
| **Purging** | Removal of false duplications and low-coverage regions from assembly |
| **HiFi reads** | PacBio high-fidelity reads: long (10–25 kbp) and highly accurate (>Q20) |
| **K-mer** | A nucleotide subsequence of length k |
| **BUSCO** | Benchmarking Universal Single-Copy Orthologs — a gene completeness metric |
| **T2T** | Telomere-to-Telomere — a complete, gap-free chromosome assembly |

---

## Results

> ⚠️ *Fill in this section after completing the tutorial with your actual results.*

### Genome Profile (GenomeScope2)

| Metric | Value |
|--------|-------|
| Estimated genome size | ___ Mb |
| Heterozygosity | ___% |
| Repeat content | ___% |
| Ploidy | ___ |

### Assembly Statistics

| Stage | Contigs/Scaffolds | N50 | Total Size | BUSCO (%) |
|-------|------------------|-----|-----------|-----------|
| hifiasm (raw) | | | | |
| After purge_dups | | | | |
| After Bionano scaffolding | | | | |
| After Hi-C scaffolding | | | | |

### Hi-C Contact Map

> *(Insert PretextSnapshot image here)*

---

## References

1. Rhie, A. et al. (2021). Towards complete and error-free genome assemblies of all vertebrate species. *Nature*, 592, 737–746.
2. Cheng, H. et al. (2021). Haplotype-resolved de novo assembly using phased assembly graphs with hifiasm. *Nature Methods*, 18, 170–175.
3. Guan, D. et al. (2020). Identifying and removing haplotypic duplication in primary genome assemblies. *Bioinformatics*, 36(9), 2896–2898.
4. Zhou, C. et al. (2023). YaHS: yet another Hi-C scaffolding tool. *Bioinformatics*, 39(1).
5. Wenger, A.M. et al. (2019). Accurate circular consensus long-read sequencing improves variant detection and assembly of a human genome. *Nature Biotechnology*, 37, 1155–1162.
6. Rhie, A. et al. (2020). Merqury: reference-free quality, completeness, and phasing assessment for genome assemblies. *Genome Biology*, 21, 245.

---

## Acknowledgements

This tutorial was developed by the **Galaxy Training Network (GTN)** and the **Vertebrate Genomes Project (VGP)** team. Original tutorial authors include Alex Ostrovsky, Cristóbal Gallardo, Anna Syme, Linelle Abueg, Brandon Pickett, Giulio Formenti, Marcella Sozzoni, and Anton Nekrutenko.

Tutorial license: [Creative Commons Attribution 4.0 International](http://creativecommons.org/licenses/by/4.0/)

---

*This repository was created as part of a bioinformatics assignment to reproduce and document the VGP genome assembly pipeline using the Galaxy platform.*
