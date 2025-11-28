# RESCRIPt_SILVA138_Workflow
QIIME2 RESCRIPt workflow for preparing SILVA 138 SSURef NR99 16S rRNA reference database and training a Naive Bayes classifier (341F–785R).

## Overview
- **Objective:** Prepare curated reference sequences and taxonomy for accurate 16S rRNA classification.
- **Software:** QIIME2 (2023.9+), RESCRIPt plugin
- **Database:** SILVA 138 SSU NR99
- **Target region:** 341F–785R (V3–V4)

- ---

## Workflow Steps
All commands are executed in bash shell with QIIME2 environment activated.

# 1. Download SILVA 138 SSU NR99
## This step downloads the **SILVA 138 SSURef NR99** reference database, which contains ribosomal RNA sequences and taxonomy information. We include **species labels** to enable species-level assignment wherever possible.

```bash
qiime rescript get-silva-data \
    --p-version '138' \
    --p-target 'SSURef_NR99' \
    --p-include-species-labels \
    --o-silva-sequences silva-138-ssu-nr99-rna-seqs.qza \
    --o-silva-taxonomy silva-138-ssu-nr99-tax.qza
```

# 2. Reverse transcribe RNA → DNA
## SILVA sequences are stored as RNA sequences, but QIIME2 and most analysis tools require DNA sequences. This command converts RNA → DNA by replacing "U" with "T".
```bash
qiime rescript reverse-transcribe \
    --i-rna-sequences silva-138-ssu-nr99-rna-seqs.qza \
    --o-dna-sequences silva-138-ssu-nr99-seqs.qza
```

# 3. Clean sequences (remove ambiguous bases/homopolymers)
## This step removes: Sequences with ambiguous bases (e.g., N, R, W). Sequences containing long homopolymers (AAAAAAA…). Cleaning improves classifier accuracy by ensuring high-quality reference sequences.
```bash
qiime rescript cull-seqs \
    --i-sequences silva-138-ssu-nr99-seqs.qza \
    --o-clean-sequences silva-138-ssu-nr99-seqs-cleaned.qza
```

# 4. Filter by length per taxon
## Different domains require different minimum lengths: Archaea (16S): ≥ 900 bp, Bacteria (16S): ≥ 1200 bp, Eukaryota (18S): ≥ 1400 bp. This keeps only biologically meaningful sequences and removes incomplete entries.
  
```bash
qiime rescript filter-seqs-length-by-taxon \
    --i-sequences silva-138-ssu-nr99-seqs-cleaned.qza \
    --i-taxonomy silva-138-ssu-nr99-tax.qza \
    --p-labels Archaea Bacteria Eukaryota \
    --p-min-lens 900 1200 1400 \
    --o-filtered-seqs silva-138-ssu-nr99-seqs-filt.qza \
    --o-discarded-seqs silva-138-ssu-nr99-seqs-discard.qza
```

# 5. Dereplicate unique sequences
## Dereplication removes duplicate sequences while respecting SILVA's taxonomy structure. Using `--p-mode uniq` keeps identical sequences even if their taxonomy differs, preserving all annotations. This step reduces database size and improves classifier speed.
```bash
qiime rescript dereplicate \
    --i-sequences silva-138-ssu-nr99-seqs-filt.qza  \
    --i-taxa silva-138-ssu-nr99-tax.qza \
    --p-rank-handles 'silva' \
    --p-mode 'uniq' \
    --o-dereplicated-sequences silva-138-ssu-nr99-seqs-derep-uniq.qza \
    --o-dereplicated-taxa silva-138-ssu-nr99-tax-derep-uniq.qza
```

# 6. Extract target amplicon (341F–785R)
### We extract the V3–V4 region of the 16S rRNA gene using the primers: Forward: 341F (CCTACGGGNGGCWGCAG), Reverse: 785R (GACTACHVGGGTATCTAATCC). This produces reference sequences matching your experimental region, which greatly improves taxonomy accuracy.
```bash
qiime feature-classifier extract-reads \
    --i-sequences silva-138-ssu-nr99-seqs-derep-uniq.qza \
    --p-f-primer CCTACGGGNGGCWGCAG \
    --p-r-primer GACTACHVGGGTATCTAATCC \
    --p-n-jobs -1 \
    --p-read-orientation 'forward' \
    --o-reads silva-138-ssu-nr99-seqs-341f-785r-shiva.qza
``` 

# 7. Dereplicate extracted region
## After extracting the V3–V4 region, many sequences become identical. This second dereplication step removes redundant entries and produces a clean, compact reference dataset for classifier training.
```bash
qiime rescript dereplicate \
    --i-sequences silva-138-ssu-nr99-seqs-341f-785r-shiva.qza \
    --i-taxa silva-138-ssu-nr99-tax-derep-uniq.qza \
    --p-rank-handles 'silva' \
    --p-mode 'uniq' \
    --o-dereplicated-sequences silva-138-ssu-nr99-seqs-341f-785r-uniq-shiva.qza \
    --o-dereplicated-taxa silva-138-ssu-nr99-tax-341f-785r-derep-uniq-shiva.qza
```

# 8. Train Naive Bayes classifier
## Using the cleaned, filtered, dereplicated, and region-extracted sequences, we train a Naive Bayes classifier compatible with QIIME2. This classifier will be used for taxonomy assignment in your actual dataset.
```bash
qiime feature-classifier fit-classifier-naive-bayes \
    --i-reference-reads silva-138-ssu-nr99-seqs-341f-785r-uniq-shiva.qza \
    --i-reference-taxonomy silva-138-ssu-nr99-tax-341f-785r-derep-uniq-shiva.qza \
    --o-classifier silva-138-ssu-nr99-rescript-classifier-shiva.qza
```

## Output Files
- `silva-138-ssu-nr99-seqs.qza` – original sequences  
- `silva-138-ssu-nr99-tax.qza` – taxonomy reference  
- `silva-138-ssu-nr99-rescript-classifier-shiva.qza` – final trained classifier

## Citation
** Yet to add **

## Author
**G. Sivasubramaniyan**  
Assistant Professor & PhD Research Scholar, Department of Microbiology   
Sri Ramachandra Institute of Higher Education and Research (SRIHER), Chennai, India  
Email id: shivaviro24@gmail.com, sivasubramaniyan@sriramachandra.edu.in

**Acknowledgment:**  Workflow guidance provided by a collaborator from Christian Medical College Vellore.



