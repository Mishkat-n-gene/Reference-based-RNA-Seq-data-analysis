# Reference-based RNA-Seq Data Analysis

A complete Galaxy-based workflow for identifying differentially expressed genes and biological pathways from RNA-Seq data, demonstrated on *Drosophila melanogaster*.

**Source:** [Galaxy Training Network Tutorial](https://training.galaxyproject.org/training-material/topics/transcriptomics/tutorials/ref-based/tutorial.html)
**Level:** Introductory | **Time:** ~8 hours | **License:** CC BY 4.0

---

## The Experiment

This tutorial reproduces the analysis from **Brooks et al. (2011)**, which investigated genes regulated by the *Pasilla* splicing factor in *Drosophila* cells. The *Pasilla* gene was knocked down via RNAi and RNA-Seq libraries were prepared from treated and untreated cells.

**7 samples total:**

| Sample | Condition | Library Type |
|---|---|---|
| GSM461176, GSM461177, GSM461178, GSM461182 | Untreated | Mixed SE/PE |
| GSM461179, GSM461180, GSM461181 | PS-depleted | Mixed SE/PE |

> The first 2 samples (GSM461177 + GSM461180) are used for FASTQ processing steps. All 7 are used for differential expression analysis.

---

## Workflow

```
FASTQ files
    |
    v
Quality Control  (Falco, MultiQC, Cutadapt)
    |
    v
Splice-aware Mapping  (STAR + dm6 genome)
    |
    v
Read Counting per Gene  (featureCounts or STAR)
    |
    v
Differential Expression  (DESeq2)
    |
    v
Functional Enrichment  (GO + KEGG)
```

---

## Step-by-Step

### 1. Data Upload

Import paired-end FASTQ files from Zenodo into a Galaxy history and build a paired collection.

```
# Full files (~1.5 GB each)
https://zenodo.org/record/6457007/files/GSM461177_1.fastqsanger
https://zenodo.org/record/6457007/files/GSM461177_2.fastqsanger
https://zenodo.org/record/6457007/files/GSM461180_1.fastqsanger
https://zenodo.org/record/6457007/files/GSM461180_2.fastqsanger

# Subsampled files (~5 MB, for quick testing)
https://zenodo.org/record/6457007/files/GSM461177_1_subsampled.fastqsanger
https://zenodo.org/record/6457007/files/GSM461177_2_subsampled.fastqsanger
```

Verify datatype is `fastqsanger` (not `fastq`).

---

### 2. Quality Control

**Tools:** Falco, MultiQC, Cutadapt

- Run **Falco** on the flattened collection to generate per-sample reports
- Aggregate with **MultiQC** for a side-by-side view
- Trim with **Cutadapt**: quality cutoff = 20, minimum length = 20

**What to look for:**
- Read length: 37 bp for both samples
- Quality drop at read ends, especially in reverse reads of GSM461180
- High duplication rates are expected in RNA-Seq data

**Expected Cutadapt output:**

| Sample | Reads removed (too short) |
|---|---|
| GSM461177_untreat_paired | ~1.4% |
| GSM461180_treat_paired | ~9% |

---

### 3. Mapping

**Tool:** RNA STAR v2.7.11b | **Reference genome:** dm6 (*Drosophila melanogaster*)

Since eukaryotic reads originate from processed mRNAs, a **splice-aware aligner** is required. STAR handles reads that span exon-exon junctions.

**Key parameters:**
- Genome: `Fly (Drosophila melanogaster): dm6 Full`
- Annotation: `Drosophila_melanogaster.BDGP6.32.109_UCSC.gtf.gz`
- Junction length: `36` (read length - 1)
- Enable per-gene read counts: `Yes`
- Compute coverage: `Yes in bedgraph format`

```
https://zenodo.org/record/6457007/files/Drosophila_melanogaster.BDGP6.32.109_UCSC.gtf.gz
```

**Expected mapping rates (from MultiQC):**

| Sample | Uniquely mapped |
|---|---|
| GSM461177_untreat_paired | >83% |
| GSM461180_treat_paired | >79% |

> Rates below 70% should be investigated for contamination.

**Optional QC on BAM files:**

| Check | Tool | What to expect |
|---|---|---|
| Duplicate rate | MarkDuplicates (Picard) | <50% is normal |
| Chromosome distribution | Samtools idxstats | Most reads on chr2, chr3, chrX |
| Gene body coverage | RSeQC Gene Body Coverage | Even 5' to 3' coverage |
| Read distribution | RSeQC Read Distribution | >80% reads on exons |

---

### 4. Counting Reads per Gene

#### Estimating Library Strandness

Before counting, determine whether the library is stranded. This affects how reads overlapping genes on opposite strands are handled.

**5 methods available:**

1. **IGV** - color reads by first-of-pair strand; equal red/blue = unstranded
2. **JBrowse2** - same visual check, integrated in Galaxy
3. **pyGenomeTracks** - compare STAR strand 1 vs strand 2 coverage
4. **STAR gene counts** - the stranding with the highest gene assignment rate matches the library
5. **Infer Experiment (RSeQC)** - compares read orientation to gene models

**Result for this dataset:** Both samples are **unstranded**

```
GSM461177_untreat_paired:
  Fraction explained by "1++,1--,2+-,2-+": 0.4626
  Fraction explained by "1+-,1-+,2++,2--": 0.4360
```

**Strandness settings across tools:**

| Library Type | Infer Experiment | HTSeq-count | featureCounts |
|---|---|---|---|
| Unstranded | undecided | no | 0 |
| Stranded Forward | 1++,1--,2+-,2-+ | yes | 1 |
| Stranded Reverse | 1+-,1-+,2++,2-- | reverse | 2 |

#### Counting with featureCounts

**Tool:** featureCounts v2.1.1

Key settings:
- Strand: `Unstranded`
- Feature type: `exon`
- Gene identifier: `gene_id`
- Paired-end reads counted as 1 fragment
- Minimum mapping quality: `10`

**Expected assignment rate:** ~63% — check with MultiQC featureCounts summary.

> If assignment rate falls below 50%, inspect the BAM in IGV and verify the GTF matches your reference genome version.

---

### 5. Differential Expression Analysis

**Tool:** DESeq2

All 7 samples are used from this step. DESeq2 models count data using a negative binomial distribution and tests for expression differences between treated (PS-depleted) and untreated conditions.

Pipeline:
1. Run DESeq2 on the count matrix
2. Annotate results with gene names and descriptions
3. Filter by adjusted p-value (typically < 0.05)
4. Visualize with heatmaps and volcano plots

---

### 6. Functional Enrichment Analysis

Two analyses are run on the set of differentially expressed genes:

**Gene Ontology (GO):** Over-represented biological processes, molecular functions, and cellular components

**KEGG Pathways:** Affected metabolic and signaling pathways

---

## Tools Summary

| Tool | Version | Step |
|---|---|---|
| Falco | 1.2.4+galaxy0 | Quality control |
| MultiQC | 1.27+galaxy4 | Report aggregation |
| Cutadapt | 5.2+galaxy0 | Trimming |
| RNA STAR | 2.7.11b+galaxy0 | Mapping |
| featureCounts | 2.1.1+galaxy0 | Read counting |
| Infer Experiment (RSeQC) | 5.0.3+galaxy0 | Strandness |
| MarkDuplicates (Picard) | 3.1.1.0 | QC |
| DESeq2 | - | Differential expression |
| JBrowse2 | 3.6.5+galaxy1 | BAM visualization |

---

## Compatible Galaxy Servers

- [UseGalaxy.eu](https://usegalaxy.eu) (recommended)
- [UseGalaxy.org](https://usegalaxy.org) (recommended)
- [UseGalaxy.fr](https://usegalaxy.fr/) (recommended)
- [UseGalaxy.org.au](https://usegalaxy.org.au) (possibly working)

---

## Citation

Brooks et al. (2011). Conservation of an RNA regulatory map between Drosophila and mammals. *Genome Research*. [doi:10.1101/gr.108662.110](https://doi.org/10.1101/gr.108662.110)

Tutorial by Berenice Batut, Mallory Freeberg, Mo Heydarian et al. — [Galaxy Training Network](https://training.galaxyproject.org)
