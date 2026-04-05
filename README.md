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
  - [Step 5: Scaffolding with Bionano Optical Maps](#step-5-scaffolding-with-bionano-optical-maps)
  - [Step 6: Hi-C Scaffolding](#step-6-hi-c-scaffolding)
  - [Step 7: Assembly Evaluation](#step-7-assembly-evaluation)
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
[Step 1] Data Preparation
    │
    ▼
[Step 2] HiFi Reads Preprocessing (Cutadapt)
    │
    ▼
[Step 3] Genome Profile Analysis (Meryl + GenomeScope2)
    │
    ▼
[Step 4] Assembly with hifiasm (Hi-C phased)
    │
    ▼
[Step 5] Bionano Scaffolding (Bionano Solve)
    │
    ▼
[Step 6] Hi-C Scaffolding (YaHS)
    │
    ▼
[Step 7] Assembly Evaluation (BUSCO + Merqury + gfastats)
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

Model: this file includes a detailed report about the model fitting.
Summary: Includes inferred genome properties (haploid length, heterozygosity %). These estimates are based on the GenomeScope2 model (black line). Good model fit to the data (blue bars) means the estimates are reliable. In this tutorial, the fit is good.
---

### Step 4: Assembly with hifiasm

**Objective:** Assemble HiFi reads into phased contigs using hifiasm, then evaluate
assembly quality using gfastats, BUSCO, and Merqury.

Hifiasm is a fast open-source *de novo* assembler specifically developed for PacBio HiFi
reads. A key advantage is its ability to resolve near-identical sequences such as repeats
and segmental duplications. Hifiasm outputs **GFA files** (Graphical Fragment Assembly),
which represent the assembly graph including sequences, nodes, and edges (overlaps),
preserving more information than plain FASTA files.

---

#### 4a. Hifiasm Assembly Modes

Depending on available data, hifiasm can be run in three modes:

| Mode | Input | Output |
|------|-------|--------|
| **Solo** | HiFi reads only | Primary + alternate assembly (pseudohaplotype) |
| **Hi-C phased** | HiFi + Hi-C reads | Hap1 + Hap2 phased assemblies |
| **Trio** | HiFi (child) + Illumina (both parents) | Maternal + paternal assemblies |

> This tutorial uses **Hi-C phased mode**, as we have both HiFi and Hi-C data.

---

#### 4b. Hi-C Phased Assembly with Hifiasm

**Tool:** `Hifiasm` 

| Parameter | Value |
|-----------|-------|
| Assembly mode | `Standard` |
| Input reads | `HiFi_collection (trimmed)` *(output of Cutadapt)* |
| Options for Hi-C partition | `Specify` |
| Hi-C R1 reads | `Hi-C_dataset_F` |
| Hi-C R2 reads | `Hi-C_dataset_R` |

**➜ Rename outputs:**

| Original Output | Renamed To | Tag |
|----------------|-----------|-----|
| Hi-C hap1 balanced contig graph | `Hap1 contigs graph` | `#hap1` |
| Hi-C hap2 balanced contig graph | `Hap2 contigs graph` | `#hap2` |

---

#### 4c. Convert GFA to FASTA with gfastats

**Tool:** `gfastats` 

The GFA outputs from hifiasm must be converted to FASTA format for downstream steps.
`gfastats` is a VGP-developed tool suite for manipulation and evaluation of FASTA/GFA files.

| Parameter | Value |
|-----------|-------|
| Input GFA file | `Hap1 contigs graph` and `Hap2 contigs graph` |
| Tool mode | `FASTA conversion` |

**➜ Rename outputs:** `Hap1 contigs FASTA` and `Hap2 contigs FASTA`

---

#### 4d. Assembly Statistics with gfastats

**Tool:** `gfastats` 

| Parameter | Value |
|-----------|-------|
| Input file | `Hap1 contigs graph` and `Hap2 contigs graph` |
| Tool mode | `Summary statistics generation` |
| Expected genome size | `11747160` *(from GenomeScope2 summary)* |
| Thousands separator in output | `No` |

**➜ Rename outputs:** `Hap1 stats` and `Hap2 stats`

**Key statistics reported by gfastats:**

| Statistic | Description |
|-----------|-------------|
| No. of contigs | Total number of contigs in the assembly |
| Largest contig | Length of the largest contig |
| Total length | Total number of bases in the assembly |
| Nx | Largest contig length L where contigs ≥ L account for x% of assembly |
| NGx | Contig length accounting for x% of the reference genome length |
| GC content | Percentage of guanine + cytosine bases |

---

#### 4e. Joining and Filtering Stats

**Step 1 — Join hap1 and hap2 statistics:**

**Tool:** `Column join` 

| Parameter | Value |
|-----------|-------|
| Input files | `Hap1 stats` and `Hap2 stats` |
| All other settings | Default |

**➜ Rename output:** `gfastats on hap1 and hap2 (full)`

**Step 2 — Remove scaffold lines (no scaffolds exist at this stage):**

**Tool:** `Search in textfiles` 
| Parameter | Value |
|-----------|-------|
| Input file | `gfastats on hap1 and hap2 (full)` |
| That | `Don't Match` |
| Type of regex | `Basic` |
| Regular Expression | `scaffold` |
| Match type | `case insensitive` |

**➜ Rename output:** `gfastats on hap1 and hap2 contigs`

The output has three columns: statistic name, hap1 value, hap2 value.

**Expected results:**

| Metric | Hap1 | Hap2 |
|--------|------|------|
| No. of contigs | 16 | 17 |
| Total contig length | ~11.3 Mbp | ~12.2 Mbp |

> ⚠️ Your values may differ slightly or be reversed between haplotypes.

---

#### 4f. Assembly Completeness with BUSCO

**Tool:** `BUSCO` 

BUSCO provides a qualitative assessment of assembly completeness by checking for genes
expected to be present exactly once in a complete assembly.

| Parameter | Value |
|-----------|-------|
| Sequences to analyze | `Hap1 contigs FASTA` and `Hap2 contigs FASTA` |
| Lineage data source | `Use cached lineage data` |
| Cached database with lineage | `Busco v5 Lineage Datasets` |
| Mode | `Genome assemblies (DNA)` |
| Use Augustus instead of Metaeuk | `Use Metaeuk` |
| Auto-detect or select lineage? | `Select lineage` |
| Lineage | `Saccharomycetes` |
| Outputs to generate | `short summary text` and `summary image` |

**➜ Rename outputs:** `BUSCO hap1` and `BUSCO hap2`

> **Note:** BUSCO can be inaccurate for taxonomic groups not well represented in OrthoDB.
> Merqury (below) provides a complementary reference-free quality assessment.

---

#### 4g. Reference-Free Quality Assessment with Merqury

**Tool:** `Merqury` 

Merqury assesses assembly quality via k-mer copy number analysis in a reference-free
manner, complementing BUSCO's gene-based approach.

| Parameter | Value |
|-----------|-------|
| Evaluation mode | `Default mode` |
| k-mer counts database | `Merged meryldb` |
| Number of assemblies | `Two assemblies` |
| First genome assembly | `Hap1 contigs FASTA` |
| Second genome assembly | `Hap2 contigs FASTA` |

**Merqury outputs:**

| Output | Description |
|--------|-------------|
| Stats collection | Completeness statistics |
| QV stats collection | Consensus accuracy quality value (Phred-scaled) |
| Plots collection | Assembly CN spectrum plot (`spectra-cn.ln`) |

> The **spectra-cn plot** shows the k-mer copy number distribution across both assemblies,
> helping identify missing, duplicated, or erroneous sequences.

---

#### Expected Quality Summary

| Metric | Hap1 | Hap2 |
|--------|------|------|
| No. of contigs | *(fill in)* | *(fill in)* |
| Total length | *(fill in)* | *(fill in)* |
| BUSCO completeness (%) | *(fill in)* | *(fill in)* |
| Merqury QV score | *(fill in)* | *(fill in)* |
| Merqury completeness (%) | *(fill in)* | *(fill in)* |

---

### Step 5: Scaffolding with Bionano Optical Maps

**Objective:** Use Bionano optical maps to orient, order, and join contigs into scaffolds.

Contigs from hifiasm are assembled into **scaffolds** — contigs interspaced with gaps —
using two technologies: **Bionano optical maps** (this step) and **Hi-C** (Step 6).

> **Note:** Scaffolding is performed on **Hap1** from Hi-C phased hifiasm. The alternate
> assembly is not scaffolded as it is incomplete.

---

#### 5a. Upload Bionano Dataset

| File | URL | Datatype |
|------|-----|----------|
| bionano.cmap | https://zenodo.org/records/5887339/files/bionano.cmap | `cmap` |

1. Go to **Upload Data → Paste/Fetch Data**, paste the URL, set datatype to `cmap`
2. Click **Start**, then **Close**

**➜ Rename dataset:** `Bionano_dataset`

---

#### 5b. Bionano Hybrid Scaffolding

**Tool:** `Bionano Hybrid Scaffold` 

The tool automates scaffolding in five steps:
1. Generate *in silico* maps from the sequence assembly
2. Align maps against Bionano genome maps and resolve conflicts
3. Merge non-conflicting maps into hybrid scaffolds
4. Align sequence maps back to hybrid scaffolds
5. Generate AGP and FASTA output files

| Parameter | Value |
|-----------|-------|
| NGS FASTA | `Hap1 contigs FASTA` |
| BioNano CMAP | `Bionano_dataset` |
| Configuration mode | `VGP mode` |
| Genome maps conflict filter | `Cut contig at conflict` |
| Sequences conflict filter | `Cut contig at conflict` |

---

#### 5c. Concatenate Scaffolded and Unscaffolded Contigs

**Tool:** `Concatenate datasets`

| Parameter | Value |
|-----------|-------|
| Concatenate Dataset | `NGScontigs scaffold NCBI trimmed` |
| Insert Dataset → Select | `NGScontigs not scaffolded trimmed` |

**➜ Rename output:** `Hap1 assembly bionano`

---

#### 5d. Evaluate Bionano Assembly with gfastats

**Tool:** `gfastats` 

| Parameter | Value |
|-----------|-------|
| Input file | `Hap1 assembly bionano` |
| Expected genome size | `11747160` |
| Thousands separator in output | `No` |

| Metric | Before Bionano | After Bionano |
|--------|---------------|--------------|
| No. of sequences | *(fill in)* | *(fill in)* |
| Largest sequence | *(fill in)* | *(fill in)* |
| Total length | *(fill in)* | *(fill in)* |
| N50 | *(fill in)* | *(fill in)* |
| GC content (%) | *(fill in)* | *(fill in)* |

> **What to look for:** N50 should increase and sequence count should decrease,
> indicating improved contiguity.
---

### Step 6: Hi-C Scaffolding

**Objective:** Use Hi-C chromatin contact data to scaffold Bionano assemblies to
chromosome scale.

Hi-C measures contact frequency between genomic loci, which correlates with their
linear distance. This allows ordering and orienting scaffolds into chromosome-level
assemblies. Hi-C reads are mapped **individually** (not as pairs) because insert sizes
in Hi-C can range from 1 bp to hundreds of megabases, violating paired-end distance
assumptions of most aligners.

---

#### 6a. Map Forward Hi-C Reads

**Tool:** `BWA-MEM2` 

| Parameter | Value |
|-----------|-------|
| Reference genome source | `Use a genome from history and build index` |
| Reference sequence | `Hap1 assembly bionano` |
| Single or Paired-end reads | `Single` |
| FASTQ dataset | `Hi-C_dataset_F` |
| Set read groups information? | `Do not set` |
| Analysis mode | `1. Simple Illumina mode` |
| BAM sorting mode | `Sort by read names (QNAME)` |

**➜ Rename output:** `BAM forward`

---

#### 6b. Map Reverse Hi-C Reads

**Tool:** `BWA-MEM2` 

Same parameters as 6a, except:

| Parameter | Value |
|-----------|-------|
| Reference sequence | `Hap1 assembly bionano` |
| FASTQ dataset | `Hi-C_dataset_R` |

**➜ Rename output:** `BAM reverse`

---

#### 6c. Merge Forward and Reverse Alignments

**Tool:** `Filter and merge chimeric reads from Arima Genomics`

| Parameter | Value |
|-----------|-------|
| First set of reads | `BAM forward` |
| Second set of reads | `BAM reverse` |

**➜ Rename output:** `BAM Hi-C reads`

---

#### 6d. Generate Initial Hi-C Contact Map

Generate a pre-scaffolding contact map as a baseline for comparison.

**Tool:** `PretextMap` 

| Parameter | Value |
|-----------|-------|
| Input BAM | `BAM Hi-C reads` |
| Sort by | `Don't sort` |

**➜ Rename output:** `PretextMap output`

**Tool:** `Pretext Snapshot` 

| Parameter | Value |
|-----------|-------|
| Input Pretext map file | `PretextMap output` |
| Output image format | `png` |
| Show grid? | `Yes` |

---

#### 6e. YaHS Scaffolding

**Tool:** `YaHS` 

YaHS uses Hi-C data to linearly orient and order contigs along chromosomes. Unlike
most Hi-C scaffolding tools, YaHS does **not** require the estimated chromosome count
as input.

| Parameter | Value |
|-----------|-------|
| Input contig sequences | `Hap1 assembly bionano` |
| Alignment file of Hi-C reads | `BAM Hi-C reads` |
| Restriction enzyme | `Enter a specific sequence` |
| Restriction enzyme sequence(s) | `CTTAAG` |

**➜ Rename output:** `YaHS Scaffolds FASTA`

---

#### 6f. Re-map Hi-C Reads to YaHS Scaffolds

Repeat the mapping process using the YaHS scaffolds as the new reference to generate
a post-scaffolding contact map.

**Map forward reads — Tool:** `BWA-MEM2` 

| Parameter | Value |
|-----------|-------|
| Reference sequence | `YaHS Scaffolds FASTA` |
| FASTQ dataset | `Hi-C_dataset_F` |
| Single or Paired-end reads | `Single` |
| Set read groups information? | `Do not set` |
| Analysis mode | `1. Simple Illumina mode` |
| BAM sorting mode | `Sort by read names (QNAME)` |

**➜ Rename output:** `BAM forward YaHS`

**Map reverse reads — Tool:** `BWA-MEM2` 

Same parameters as above, except:

| Parameter | Value |
|-----------|-------|
| FASTQ dataset | `Hi-C_dataset_R` |

**➜ Rename output:** `BAM reverse YaHS`

---

#### 6g. Merge YaHS-Mapped Reads

**Tool:** `Filter and merge chimeric reads from Arima Genomics`

| Parameter | Value |
|-----------|-------|
| First set of reads | `BAM forward YaHS` |
| Second set of reads | `BAM reverse YaHS` |

**➜ Rename output:** `BAM Hi-C reads YaHS`

---

#### 6h. Generate Final Hi-C Contact Map

**Tool:** `PretextMap` 

| Parameter | Value |
|-----------|-------|
| Input BAM | `BAM Hi-C reads YaHS` |
| Sort by | `Don't sort` |

**➜ Rename output:** `PretextMap output YaHS`

**Tool:** `Pretext Snapshot` 

| Parameter | Value |
|-----------|-------|
| Input Pretext map file | `PretextMap output YaHS` |
| Output image format | `png` |
| Show grid? | `Yes` |

---

#### 6i. Comparing Contact Maps

Compare the pre- and post-scaffolding contact maps to assess improvement:

| Feature | Before YaHS | After YaHS |
|---------|------------|-----------|
| Contact map image | *(insert PretextSnapshot)* | *(insert PretextSnapshot)* |
| No. of scaffolds | *(fill in)* | *(fill in)* |
| Largest scaffold | *(fill in)* | *(fill in)* |
| N50 | *(fill in)* | *(fill in)* |

> **What to look for:** The post-YaHS contact map should show clear diagonal blocks
> representing chromosomes, with strong intra-chromosomal signal and minimal
> off-diagonal noise.

---

### Step 7: Assembly Evaluation

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
