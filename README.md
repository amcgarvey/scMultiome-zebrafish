# scMultiome-zebrafish
Most tutorials and resources for analysis of single cell omics data are oriented towards human and mouse data. Here are some notes on scATAC+RNAseq analysis methods with specific adaptions for zebrafish.

For now mostly aimed at analysis of 10X multiome data [https://www.10xgenomics.com/products/single-cell-multiome-atac-plus-gene-expression](https://www.10xgenomics.com/products/single-cell-multiome-atac-plus-gene-expression). 

## Resources


## Making a reference 

* [Creating a zebrafish (danrer11) Reference Package with cellranger-arc mkref](notebooks/cellranger-arc-mkref.md)

## Alignment, calling cells, QC
[In progress] Pipeline for fastqc, fastq files generation and mapping + cell calling with cellranger


## Feature selection
Feature selection for scATAC-seq data is challenging in most species due to a lack of annotations of non-coding species and the uneven coverage in single cells.
* Default method of feature selection is peak calling on aggregated pseudobulk from all cells in samples, or an iterative clustering approach
* Alternative feature selection that uses the single cell data without prior clustering/aggregation is [ScregSeg](https://github.com/BIMSBbioinfo/scregseg)

## Clustering, dimensionality reduction, cell type calling and annotation
* seurat v4: specific adaptions for zrbrafish such as finding mitochondrial features, annotation the ATAC modality [in this notebook](https://github.com/amcgarvey/scMultiome-zebrafish/tree/main/notebooks/Seuratv4_zebrafish_notebook.ipynb)
* archR
* deeplearning based methods

## Sample integration
* Single modality datasets: can be done simply with batch correction methods such as CCA from seurat, SCTransform (regress batches) etc
* Using joint modality data: as yet main published tools don't do this

