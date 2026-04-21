Reference-based RNA-Seq Data Analysis
A hands-on tutorial from the Galaxy Training Network covering the complete computational workflow for detecting differentially expressed (DE) genes and pathways from RNA-Seq data. The tutorial is based on a real experiment profiling Drosophila melanogaster cells after depletion of the Pasilla regulatory gene.
Source: Galaxy Training Network - Reference-based RNA-Seq
Estimated time: 8 hours
Level: Introductory
License: Creative Commons Attribution 4.0 International

Background
RNA sequencing (RNA-Seq) is a widely used technology for analyzing the cellular transcriptome. One of its most common applications is profiling gene expression by identifying differentially expressed genes or molecular pathways across biological conditions.
This tutorial reproduces an analysis from Brooks et al. (2011), where the authors identified genes and pathways regulated by the Pasilla gene (the Drosophila homologue of mammalian splicing regulators Nova-1 and Nova-2) using RNA-Seq data. The Pasilla gene was depleted via RNA interference (RNAi), and total RNA was isolated from both treated and untreated samples.
Seven samples are used across the tutorial:

4 untreated samples: GSM461176, GSM461177, GSM461178, GSM461182
3 treated (Pasilla-depleted) samples: GSM461179, GSM461180, GSM461181

Two samples (one per condition) are used for the initial FASTQ processing steps; all seven are used for the differential expression analysis.

Learning Objectives
After completing this tutorial, you will be able to:

Evaluate RNA-Seq data quality using Falco and MultiQC reports
Explain the principle of splice-aware mapping of RNA-Seq reads to a eukaryotic reference genome
Select and run a state-of-the-art RNA-Seq mapping tool (STAR)
Assess the quality of mapping results
Determine library strandness from mapping outputs
Count the number of reads per annotated gene using featureCounts or STAR
Understand count normalization prior to sample comparison
Run a differential gene expression analysis with DESeq2
Interpret DESeq2 output to identify, annotate, and visualize DE genes
Perform Gene Ontology (GO) and KEGG pathway enrichment analyses


Prerequisites

Introduction to Galaxy Analyses
Quality Control (slides and hands-on)
Mapping (slides and hands-on)


Data
All input datasets are available on Zenodo at https://zenodo.org/record/6457007.
For the FASTQ processing steps, the following paired-end files are used:
https://zenodo.org/record/6457007/files/GSM461177_1.fastqsanger
https://zenodo.org/record/6457007/files/GSM461177_2.fastqsanger
https://zenodo.org/record/6457007/files/GSM461180_1.fastqsanger
https://zenodo.org/record/6457007/files/GSM461180_2.fastqsanger
Subsampled files (~5 MB each) are also available for a quicker run-through:
https://zenodo.org/record/6457007/files/GSM461177_1_subsampled.fastqsanger
https://zenodo.org/record/6457007/files/GSM461177_2_subsampled.fastqsanger
https://zenodo.org/record/6457007/files/GSM461180_1_subsampled.fastqsanger
https://zenodo.org/record/6457007/files/GSM461180_2_subsampled.fastqsanger
The gene annotation file (GTF):
https://zenodo.org/record/6457007/files/Drosophila_melanogaster.BDGP6.32.109_UCSC.gtf.gz

Workflow Overview
1. Data Upload
Import FASTQ files into a Galaxy history and create a paired-end dataset collection. Verify that the datatype is set to fastqsanger.
2. Quality Control
Sequence quality is assessed and trimmed using three tools:

Falco - generates per-sample quality reports (a drop-in replacement for FastQC)
MultiQC - aggregates reports for easy comparison across samples
Cutadapt - trims low-quality bases and removes reads shorter than 20 bp

Key parameters for Cutadapt: quality cutoff of 20, minimum read length of 20. The tool is run on the paired-end collection to ensure consistent trimming of both mates.
3. Mapping
Reads originating from eukaryotic mRNAs must be mapped using a splice-aware aligner, since the majority of reads span intron-exon boundaries. This tutorial uses STAR (Spliced Transcripts Alignment to a Reference).
Mapping is performed against the Drosophila melanogaster dm6 reference genome, with the Ensembl GTF annotation provided for splice junction guidance. The junction length parameter should be set to (read length - 1), which is 36 for this dataset.
Mapping quality is summarized using MultiQC. Both samples are expected to have over 80% of reads uniquely mapped. The BAM outputs can be inspected visually in IGV or JBrowse2.
Optional quality checks on BAM files include:

MarkDuplicates (Picard) - checks duplicate read levels (up to 50% is considered normal)
Samtools idxstats - examines read distribution across chromosomes
Gene Body Coverage (RSeQC) - checks for 5' or 3' coverage bias
Read Distribution (RSeQC) - confirms that the majority of reads (>80%) map to exonic regions

4. Counting Reads per Annotated Gene
To compare gene expression across conditions, reads are counted per gene using exon-level overlaps.
Estimating Library Strandness
Library strandness must be determined before counting, as it affects how overlapping genes on opposite strands are handled. There are five methods available:

Visual inspection of read strands in IGV (color reads by first-of-pair strand)
Visual inspection in JBrowse2 with pileup strand coloring
Coverage comparison across strands from STAR using pyGenomeTracks
Inspection of STAR gene count output columns across stranded/unstranded scenarios
Infer Experiment (RSeQC) - compares read orientation against reference gene model

For this dataset, both samples produce near-equal fractions of forward and reverse reads, confirming an unstranded library.
Library TypeInfer ExperimentHISAT2HTSeq-countfeatureCountsPE - UnstrandedundecideddefaultnoUnstranded (0)PE - Stranded Forward1++,1--,2+-,2-+Second Strand F/FRyesForward (1)PE - Stranded Reverse1+-,1-+,2++,2--First Strand R/RFreverseReverse (2)
Counting with featureCounts
featureCounts is run with the following key settings:

Strand information: Unstranded
Feature type: exon
Gene identifier: gene_id
Paired-end reads counted as single fragments
Minimum mapping quality: 10

Approximately 63% of reads are expected to be assigned to genes, which is a satisfactory rate. The output is a two-column count table (gene ID and count) compatible with DESeq2, edgeR, and limma-voom.
Alternatively, STAR's built-in gene count output can be used. The STAR output contains four columns (gene ID, unstranded counts, stranded-forward counts, stranded-reverse counts) and requires reformatting to extract the relevant column (column 2 for this unstranded dataset).
5. Differential Gene Expression Analysis
All seven samples are used from this step onwards. The analysis is performed with DESeq2, which:

Normalizes counts across samples
Models count data using a negative binomial distribution
Tests for differential expression between treated and untreated conditions

Results are annotated to add gene names and descriptions, and filtered to extract significantly DE genes (adjusted p-value threshold typically < 0.05). Visualization includes heatmaps and volcano plots.
6. Functional Enrichment Analysis
Two enrichment analyses are performed on the set of DE genes:

Gene Ontology (GO) analysis - identifies over-represented biological processes, molecular functions, and cellular components
KEGG pathway analysis - identifies affected metabolic and signaling pathways


Tools Used
ToolVersionPurposeFalco1.2.4+galaxy0Per-sample quality reportsMultiQC1.27+galaxy4Aggregated quality reportingCutadapt5.2+galaxy0Read trimming and filteringRNA STAR2.7.11b+galaxy0Splice-aware read mappingfeatureCounts2.1.1+galaxy0Read counting per geneMarkDuplicates3.1.1.0Duplicate read assessmentSamtools idxstats2.0.7Chromosome-level read distributionGene Body Coverage5.0.3+galaxy0Coverage uniformity across gene bodiesRead Distribution5.0.3+galaxy0Read distribution across genomic featuresInfer Experiment5.0.3+galaxy0Library strandness estimationDESeq2-Differential expression analysisJBrowse23.6.5+galaxy1Genome browser for BAM inspectionpyGenomeTracks3.9+galaxy0Coverage visualization

Compatible Galaxy Servers

UseGalaxy.eu (recommended)
UseGalaxy.fr (recommended)
UseGalaxy.org / Main (recommended)
UseGalaxy.org.au (possibly working)


Input Histories and Answer Histories
Pre-loaded input and answer histories are available on UseGalaxy.eu for reference. See the tutorial page for direct links.

Citation
Brooks AN, Yang L, Duff MO, et al. Conservation of an RNA regulatory map between Drosophila and mammals. Genome Research, 2011. doi:10.1101/gr.108662.110
Tutorial authors include Berenice Batut, Mallory Freeberg, Mo Heydarian, Anika Erxleben, Pavankumar Videm, Clemens Blank, Maria Doyle, Nicola Soranzo, Peter van Heusden, and Lucille Delisle.

License
Tutorial content is licensed under Creative Commons Attribution 4.0 International. The GTN framework is licensed under MIT.
