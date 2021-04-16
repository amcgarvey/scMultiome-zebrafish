# scMultiome-zebrafish
Notes on scATAC+RNAseq analysis specifically for zebrafish.

For now mostly aimed at analysis of 10X multiome data [https://www.10xgenomics.com/products/single-cell-multiome-atac-plus-gene-expression](https://www.10xgenomics.com/products/single-cell-multiome-atac-plus-gene-expression). Specific notes for working with zebrafish as the default analysis packages are usually oriented towards human and mouse samples.

## Resources


## Making a reference 

* [Creating a zebrafish (danrer11) Reference Package with cellranger-arc mkref](cellranger-arc-mkref.md)

## Alignment, calling cells, QC

* Running cellranger-count, discussion on how it deals with nuclei RNA-seq data (including intronic reads)

## Clustering, dimensionality reduction, cell type calling and annotation
* seurat v4
* archR
* deeplearning based methods

## Sample integration
* How to integrate multiome samples
* How to integrate samples with single modality datasets
