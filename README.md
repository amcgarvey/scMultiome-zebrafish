# scMultiome-zebrafish
Most tutorials and resources for analysis of single cell omics data are oriented towards human and mouse data. Here are some notes on scATAC+RNAseq analysis methods with specific adaptions for zebrafish.

For now mostly aimed at analysis of 10X multiome data [https://www.10xgenomics.com/products/single-cell-multiome-atac-plus-gene-expression](https://www.10xgenomics.com/products/single-cell-multiome-atac-plus-gene-expression). 

## Resources


## Making a reference 

* [Creating a zebrafish (danrer11) Reference Package with cellranger-arc mkref](notebooks/cellranger-arc-mkref.md)

## Alignment, calling cells, QC

* Running cellranger-count, discussion on how it deals with nuclei RNA-seq data (including intronic reads)

## Feature selection
Feature selection for scATAC-seq data is challenging in most species due to a lack of annotations of non-coding species and the uneven coverage in single cells.
* Default method of feature selection is peak calling on aggregated pseudobulk from all cells in samples, or an iterative clustering approach
* Alternative feature selection that uses the single cell data without prior clustering/aggregation is [ScregSeg](https://github.com/BIMSBbioinfo/scregseg)

## Clustering, dimensionality reduction, cell type calling and annotation
* seurat v4: specific adaptions for zrbrafish such as finding mitochondrial features, annotation the ATAC modality [in this notebook](./notebooks/Seuratv4_zebrafish_notebook.ipynb)
* archR
* deeplearning based methods

## Sample integration
* How to integrate multiome samples
  * Using seurat for scRNA-seq modality (jupyternotebook? or notebook with summary, integrating, batch correction, visualisation, checking techinical aspects of integration (features, mitochondira)
* How to integrate samples with single modality datasets
