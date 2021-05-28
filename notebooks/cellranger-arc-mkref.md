## Running cellranger-arc on multiome (scATAC+scRNA) data

10x provides a cellranger analysis suite for the new multiome data called cellranger-arc.

As usual they only provide precompiled reference genomes for mouse and human so we need to prepare our own for zebrafish. They provide guidance on how to make the reference genome [here](https://support.10xgenomics.com/single-cell-multiome-atac-gex/software/pipelines/latest/advanced/references), but particular attention is needed to filter the GTF files.

Here I pooled together Pedro's tutorial on how to do this for scRNA-seq with the description from 10X on how they prepared the mouse and human references [here](https://support.10xgenomics.com/single-cell-multiome-atac-gex/software/release-notes/references) and a few additional modifications.


The requrired annotation files can be downloaded here
```
wget "ftp://ftp.ensembl.org/pub/release-102/gtf/danio_rerio/Danio_rerio.GRCz11.102.gtf.gz"
wget "ftp://ftp.ensembl.org/pub/release-100/fasta/danio_rerio/dna/Danio_rerio.GRCz11.dna_sm.primary_assembly.fa.gz"
# in the absense of a targeted zebrafish motif repository, I used this file
wget "http://jaspar.genereg.net/download/CORE/JASPAR2020_CORE_vertebrates_non-redundant_pfms_jaspar.txt"
```

Here is a bash script to apply a few recommended filtering step to these files. It is mostly copied from the 10X script for mouse/human but I've added a few modifications. 

```
# Input files

fasta_in="Danio_rerio.GRCz11.dna_sm.primary_assembly.fa"
gtf="Danio_rerio.GRCz11.100.gtf"
gtf_in="$(basename "$gtf").replaced"

# Update the latest gtf release of danrer has around 2000 rows missing the gene_name attribute, which causes cellranger mkref to crash. 
# I contacted ensembl about this and they said it is due to an updated calculation of annotation confidence using human orthalogues now 
# the association drops below their minimum confidence. In order not to lose these protein coding genes, just becuase they don't have a 
# confident annotation I wrote a little script to add in a gene_name attribute and fill it with the ensembl id.

# write first metadata lines to file
head -n5 "$gtf" | head > "$gtf_in"
# skip metadata lines, if a row contains the gene_name attribute print that line to the file, if not add in a gene_name attribut and fill 
# in with the ensembl id then append to the modified gtf file
grep -vE "^#" "$gtf" | awk '{
        if ( $0 ~ /gene_name/)
                print $0;
        else
                print $1"\t"$2"\t"$3"\t"$4"\t"$5"\t"$6"\t"$7"\t"$8"\t"$9,$10, $11, $12, "gene_name", $10, substr($0, index($0,$13)) ;;}' >> "$gtf_in"
                
# Modify sequence headers in the Ensembl FASTA to match the file
# "GRCh38.primary_assembly.genome.fa" from GENCODE. Unplaced and unlocalized
# sequences such as "KI270728.1" have the same names in both versions.
#
# Input FASTA:
#   >1 dna:chromosome chromosome:GRCh38:1:1:248956422:1 REF
#
# Output FASTA:
#   >chr1 1
fasta_modified="$(basename "$fasta_in").modified"
# sed commands:
# 1. Replace metadata after space with original contig name, as in GENCODE (I'm not sure how necessary this first part is)
# 2. Add "chr" to names of autosomes and sex chromosomes # sex chromosomes redundant for zebrafish
# 3. Handle the mitochrondrial chromosome (in zebrafish GTF is is called chrMT, not chrM as used in human)
cat "$fasta_in" | sed -E 's/^>(\S+).*/>\1 \1/' | \
sed -E 's/^>([0-9]+|[XY]) />chr\1 /' | \
sed -E 's/^>(K.*) />chr\1 /' | \
sed -E 's/^>MT />chrMT /' > "$fasta_modified"


# Remove version suffix from transcript, gene, and exon IDs in order to match
# previous Cell Ranger reference packages
#
# Input GTF:
#     ... gene_id "ENSG00000223972.5"; ...
# Output GTF:
#     ... gene_id "ENSG00000223972"; gene_version "5"; ...
gtf_modified="$(basename "$gtf_in").modified"
# Pattern matches Ensembl gene, transcript, and exon IDs for human or mouse:
ID="(ENS(DAR)?[GTE][0-9]+)\.([0-9]+)"
cat "$gtf_in" | sed -E 's/gene_id "'"$ID"'";/gene_id "\1"; gene_version "\3";/' | sed -E 's/transcript_id "'"$ID"'";/transcript_id "\1"; transcript_version "\3";/' | sed -E 's/exon_id "'"$ID"'";/exon_id "\1"; exon_version "\3";/' > "$gtf_modified"

# Define string patterns for GTF tags
# NOTES:
# - Since GENCODE release 31/M22 (Ensembl 97), the "lincRNA" and "antisense"
#   biotypes are part of a more generic "lncRNA" biotype. (But for zebrafish the old terminology "lincRNA" and "antisense" still used)
# - These filters are relevant only to GTF files from GENCODE. The GTFs from
#   Ensembl release 98 have the following differences:
#   - The names "gene_biotype" and "transcript_biotype" are used instead of
#     "gene_type" and "transcript_type".
#   - Readthrough transcripts are present but are not marked with the
#     "readthrough_transcript" tag.
#   - Only the X chromosome versions of genes in the pseudoautosomal regions
#     are present, so there is no "PAR" tag.
BIOTYPE_PATTERN="(protein_coding|lincRNA|antisense|IG_C_gene|IG_D_gene|IG_J_gene|IG_LV_gene|IG_V_gene|IG_V_pseudogene|IG_J_pseudogene|IG_C_pseudogene|TR_C_gene|TR_D_gene|TR_J_gene|TR_V_gene|TR_V_pseudogene|TR_J_pseudogene)"
GENE_PATTERN="gene_biotype \"${BIOTYPE_PATTERN}\""
TX_PATTERN="transcript_biotype \"${BIOTYPE_PATTERN}\""
READTHROUGH_PATTERN="tag \"readthrough_transcript\""
PAR_PATTERN="tag \"PAR\""


# Construct the gene ID allowlist. We filter the list of all transcripts
# based on these criteria:
#   - allowable gene_type (biotype)
#   - allowable transcript_type (biotype)
#   - no "PAR" tag (only present for Y chromosome PAR)
#   # don't do this (in line with what Pedro does)- no "readthrough_transcript" tag
# We then collect the list of gene IDs that have at least one associated
# transcript passing the filters.
#cat "$gtf_modified" | awk '$3 == "transcript"' | grep -E "$GENE_PATTERN" | grep -E "$TX_PATTERN" | grep -Ev "$READTHROUGH_PATTERN" | grep -Ev "$PAR_PATTERN" | sed -E 's/.*(gene_id "[^"]+").*/\1/' | sort | uniq > "gene_allowlist"
cat "$gtf_modified" | awk '$3 == "transcript"' | grep -E "$GENE_PATTERN" | grep -E "$TX_PATTERN" | grep -Ev "$READTHROUGH_PATTERN" | grep -Ev "$PAR_PATTERN" | sed -E 's/.*(gene_id "[^"]+").*/\1/' | sort | uniq > "gene_allowlist"

# Filter the GTF file based on the gene allowlist
gtf_filtered="$(basename "$gtf_in").filtered"
# Copy header lines beginning with "#"
grep -E "^#" "$gtf_modified" > "$gtf_filtered"
# Filter to the gene allowlist
grep -Ff "gene_allowlist" "$gtf_modified" >> "$gtf_filtered"

# (from pedro) The final step for building a cellranger compatible GTF file is to append the 'chr' string to the field that specifies the chromosome. A simple awk line does the job.
gtf_chr="$(basename "$gtf_filtered").chr"
# write first 5 lines on metadata to final file
head -n5 "$gtf_filtered" | head > "$gtf_chr"
# to all remaining lines of gtf append "chr" to the beginning and append to the final file
tail -n +6 "$gtf_filtered" | awk '{print "chr"$0}' >> $gtf_chr


# Change motif headers so the human-readable motif name precedes the motif
# identifier. So ">MA0004.1    Arnt" -> ">Arnt_MA0004.1".
motifs_in="JASPAR2020_CORE_vertebrates_non-redundant_pfms_jaspar.txt"
motifs_modified="$(basename "$motifs_in").modified"
awk '{
    if ( substr($1, 1, 1) == ">" ) {
        print ">" $2 "_" substr($1,2)
    } else {
        print
    }
}' "$motifs_in" > "$motifs_modified"
}' "$motifs_in" > "$motifs_modified"

```

Once all the filtering is done and checked you need to generate a config file with all the paths. Here is my example:

```
{
    organism: "Danio rerio"
    genome: ["danrer11"]
    input_fasta: ["Danio_rerio.GRCz11.dna_sm.primary_assembly.fa.modified"]
    input_gtf: ["Danio_rerio.GRCz11.103.gtf.replaced.filtered.chr"]
    non_nuclear_contigs: ["chrMT"]
    input_motifs: "JASPAR2020_CORE_vertebrates_non-redundant_pfms_jaspar.txt.modified"
}
```

Then run the following commands. It can take a bit of time so I run is as a job on the cluster

```
cellranger-arc mkref --config danrer11.config
```

Then cellranger-arc count can be run by speicifying the new input
