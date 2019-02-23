# deMULTIplex
deMULTIplex is the companion software for our recently-described method for single-cell RNA sequencing sample Multiplexing Using Lipid-Tagged Indices: MULTI-seq (for more information, check out our preprint: https://www.biorxiv.org/content/early/2018/08/08/387241). MULTI-seq is methodologically analogous to the Cell Hashing (Stoeckius et al., 2018, Genome Biology) and Click-Tags (Gehring et al., 2018, bioRxiv) except we utilize lipid- and cholesterol-modified oligonucleotides to rapidly and non-perturbatively label live-cell and nuclear membranes.

## Installation (in R/Rstudio)
devtools::install_github('chris-mcginnis-ucsf/MULTI-seq')

## Dependencies
DoubletFinder requires the following R packages: 
* KernSmooth (2.23-15) 
* reshape2 (1.4.3) 
* rTSNE (0.15) 
* stringdist (0.9.5.1)
* ShortRead (1.40.0)
* NOTE: These package versions were used in the bioRxiv paper, but other versions may work, as well.

# deMULTIplex Overview
## MULTIseq sample barcode FASTQ alignment
R functions found in 'MULTIseq.Align.Suite.R' can be used to convert MULTI-seq sample barcode FASTQ files into a sample barcode UMI count matrix. Think of this pipeline as 'CellRanger' for sample barcode data. This pre-processing pipeline can be split into four distinct steps:
1. Split raw FASTQs into cell barcode, UMI, and sample barcode sequences associated with a user-defined set of cell barcodes
2. Remove reads that do not align with >1 mismatch to any MULTI-seq sample barcode reference sequence, 
3. Remove reads representing duplicated UMIs on a cell-by-cell basis
4. Convert this parsed read table to a sample barcode UMI count matrix. 

![alternativetext](/Figures/MULTIseq_Alignment_2.png)

This sample barcode UMI count matrix can be used as the input for the MULTI-seq sample classification workflow (discussed below), or alternative classification strategies (Seurat, DemuxEM, etc.).

## MULTIseq sample classification workflow
R functions found in 'MULTIseq.Classification.Suite.R' can applied to MULTI-seq sample barcode UMI count matrices to classify cells into sample groups. The MULTI-seq sample classification workflow builds upon concepts borrowed from Cell-Hashing (Stoeckius et al., 2018) and Perturb-seq (Adamson et al., 2016; Dixit et al., 2016) and can be split into five distinct steps:
1. Model the probability density function for each normalized sample barcode UMI distribution using Gaussian-kernal density estimation (as in Perturb-Seq)
2. Define local maxima corresponding to positive cells (highest maxima) and background cells (mode)
3. Define barcode-specific thresholds across an inter-maxima quantile sweep (0.2-0.99), compute the proportion of classified singlets for each quantile
4. Using barcode-specific thresholds generated by the quantile that maximizes the proportion of singlets, classify cells according to the number of barcode thresholds it surpasses (as in Cell Hashing) -- i.e., 0 thresholds = Negative, 1 threshold = Singlet, >1 threshold = Doublet/Multiplet.
5. Remove unclassified cells, repeat steps 1-4 until all cells are classified as either singlets or doublets/multiplets.

![alternativetext](/Figures/MULTIseq_ClassificationWorkflow.png)

## MULTIseq semi-supervised negative-cell reclassification 
In its current form, MULTI-seq barcoding is an imperfect process that produces a small fraction of cells that cannot be classified into sample groups. These negative cells are of two varieties: True and false negatives. True negatives result from cells with poor barcode labeling. In contrast, false negatives result from algorithmic misclassification. Since a single inter-maxima quantile threshold is applied to all barcodes during sample classification, we believe false negatives arise because this thresholding strategy may be sub-optimal for a subset of barcode distributions. Thus, although false negatives have poor *absolute* signal in comparison to high-confidence singlets, we reasoned that false negatives could be ‘rescued’ by computing the *relative* strength of each barcode signal on a cell-by-cell basis.

Broadly, negative cell reclassification uses the initial MULTI-seq sample classification results as 'ground-truth' labels during semi-supervised k-means clustering of negative cells (as in Cell Hashing). This pipeline can be split into seven distinct steps:  
1. Record the total number of thresholds that each negative cell surpasses at each inter-maxima quantile.
2. Compute each negative cell’s classification stability (CS) – i.e., the number of quantiles across which a cell surpasses a single threshold.
3. Subset equal numbers of ‘ground-truth’ cells from the original classification results.
4. Perform semi-supervised k-means clustering on merged data including ‘ground-truth’ and negative cells. Clustering is semi-supervised because one member of each ‘ground-truth’ sample group is used to initialize cluster centers.
5. Compute the rate at which ‘ground-truth’ and negative cell classifications match the k-means results.
6. Iteratively repeat steps 4 and 5 using a different ‘ground-truth’ cell to initialize cluster centers during each iteration. Repeat until all ‘ground-truth’ cells have been used.
7. Compare k-means matching rates between ‘ground-truth’ and negative cells binned according to CS values. Negative cells with CS values resulting in matching rates that approximate ‘ground-truth’ matching rates are reclassified.

![alternativetext](/Figures/MULTIseq_NegativeCellReclassification.png)

# Tutorial: 96-plex HMEC sample multiplexed scRNA-seq
## Step 1: MULTI-seq sample barcode pre-processing and alignment

```R
## Define vectors for reference barcode sequences and cell IDs
bar.ref <- load("/path/to/BClist.Robj")
cellIDs <- load("/path/to/cellIDs.Robj")
```



# Referencens
1. Stoeckius M, Zheng S, Houck-Loomis B, Hao S, Yeung BZ, Smibert P, Satija R. Cell "hashing" with barcoded antibodies enables multiplexing and doublet detection for single cell genomics. 2017. Preprint. bioRxiv doi: 10.1101/237693.
2. Adamson B, Norman TM, Jost M, Cho MY, Nuñez JK, Chen Y, et al. A Multiplexed Single-Cell CRISPR Screening Platform Enables Systematic Dissection of the Unfolded Protein Response. Cell. 2016; 167(7):1867-82.e21.
3. Dixit A, Parnas O, Li B, Chen J, Fulco CP, Jerby-Arnon L, et al. Perturb-Seq: Dissecting Molecular Circuits with Scalable Single-Cell RNA Profiling of Pooled Genetic Screens. Cell. 2016; 167(7):1853-66.e17.
4. Gaublomme JT, Li B, McCabe C, Knecht A, Drokhlyansky E, Van Wittenberghe N, Waldman J. Nuclei multiplexing with barcoded antibodies for single-nucleus genomics. 2018. Preprint. bioRxiv doi: 10.1101/476036.
