# Using Qiime2 to analyze fungal communites
## You should have a manifest file already! Make sure it is in your working directory

## Requesting resources:
```shell
sinteractive -c 28 -t 01:00:00 -J Fungal -A PAS2658
```
## Go to directory Fungal:
```shell
cd /fs/scratch/PASXXXX/Your_OSC_ID/Microbiomes/Fungal
```
## Then let's start by trimming the data
```shell
# Starting Qiime2
module load miniconda3/23.3.1-py310
conda activate qiime2-amplicon-2024.2
qiime

# Importing fastq files for use in Qiime
cp ../Bacterial/Manifest.tsv .
qiime tools import --type 'SampleData[PairedEndSequencesWithQuality]' --input-path Manifest.tsv --output-path Fungal.qza --input-format PairedEndFastqManifestPhred33V2

# Preparing visualization file of the sequences
qiime demux summarize --i-data Fungal.qza --o-visualization Fungal_vis.qzv
mkdir Trimming
# Trimming (this process takes around 1.5 hours, you will need to submit a batch job with the following instructions:)
tee -a Fungal_trimming.sh <<EOF
#!/bin/bash
#SBATCH -J ondemand/sys/myjobs/basic_sequential
#SBATCH --job-name Fungal_trimming
#SBATCH --time=02:00:00
#SBATCH --ntasks=28
#SBATCH --exclusive
#SBATCH --mail-type=BEGIN,END,FAIL
#SBATCH --account=PAS2658
cd /fs/scratch/PAS2658/Alexis/Microbiomes/Fungal
module load miniconda3/23.3.1-py310
conda activate qiime2-amplicon-2024.2
qiime q2-itsxpress trim-pair-output-unmerged --i-per-sample-sequences Fungal.qza --p-region ITS2 --p-taxa F --p-threads 28 --o-trimmed Trimming/trimmed.qza
EOF
```
```
sbatch Fungal_trimming.sh
```

## Using DADA2 to identify variants
```shell
# once you go the trimmed data you may need to start Qiime2 again
module load miniconda3/23.3.1-py310
conda activate qiime2-amplicon-2024.2
qiime dada2 denoise-paired --i-demultiplexed-seqs Trimming/trimmed.qza --p-trunc-len-f 0 --p-trunc-len-r 0 --p-n-threads 28 --output-dir dada2out
```
## Downloading UNITE database references for ITS (https://dx.doi.org/10.15156/BIO/2938079)
```shell
mkdir Classifiers
cd Classifiers
cp /fs/ess/PASXXXX/HCS7004_Files/Microbiomes/sh_qiime_release_25.07.2023.tgz .
tar zxvf sh_qiime_release_25.07.2023.tgz
rm sh_qiime_release_25.07.2023.tgz
cd ..
```
## Importing database in Qiime2
```shell
qiime tools import --type 'FeatureData[Sequence]' --input-path Classifiers/sh_refs_qiime_ver9_99_25.07.2023.fasta --output-path Classifiers/UNITE.qza
```
## Import the associated UNITE taxonomy file
```shell
qiime tools import --type 'FeatureData[Taxonomy]' --input-format HeaderlessTSVTaxonomyFormat --input-path Classifiers/sh_taxonomy_qiime_ver9_99_25.07.2023.txt --output-path Classifiers/UNITE_taxonomy.qza
```
## Train Qiime for the classification
```shell
# This process takes around 1.5 hours, you will need to submit a batch job with the following instructions:

tee -a Fungal_training.sh <<EOF
#!/bin/bash
#SBATCH -J ondemand/sys/myjobs/basic_sequential
#SBATCH --job-name Fungal_training
#SBATCH --time=02:00:00
#SBATCH --ntasks=28
#SBATCH --exclusive
#SBATCH --mail-type=BEGIN,END,FAIL
#SBATCH --account=PASXXXX
cd /fs/scratch/PASXXXX/Your_OSC_ID/Microbiomes/Fungal
module load miniconda3/23.3.1-py310
conda activate qiime2-amplicon-2024.2
qiime feature-classifier fit-classifier-naive-bayes --i-reference-reads Classifiers/UNITE.qza --i-reference-taxonomy Classifiers/UNITE_taxonomy.qza --o-classifier Classifiers/Classifier.qza
EOF
```
## Classify the sequence variants
```shell
qiime feature-classifier classify-sklearn --i-classifier Classifiers/Classifier.qza --i-reads dada2out/representative_sequences.qza --o-classification Classifiers/Taxonomy.qza
```
## Summarizing results
```shell
qiime metadata tabulate --m-input-file Classifiers/Taxonomy.qza --o-visualization Classifiers/Taxonomy_vis.qzv
```
## First interactive barplot
```shell
cp ../Bacterial/Metadata.txt .
qiime taxa barplot --i-table dada2out/table.qza  --i-taxonomy Classifiers/Taxonomy.qza --m-metadata-file Metadata.txt --o-visualization taxa-bar-plots.qzv
```
## Filtering out rare ASVs
```shell
qiime feature-table filter-features --i-table dada2out/table.qza --p-min-frequency 0 --p-min-samples 1 --o-filtered-table dada2out/table_filt.qza
qiime feature-table filter-seqs --i-data dada2out/representative_sequences.qza --i-table dada2out/table_filt.qza --o-filtered-data dada2out/rep_seqs_filt.qza
```
## Building a phylogeny
```shell
# Multiple sequence alignment using MAFFT
mkdir tree_out
qiime alignment mafft --i-sequences dada2out/representative_sequences.qza --p-n-threads 28 --o-alignment tree_out/rep_seqs_filt_aligned.qza
# Filtering multiple sequence alignment
qiime alignment mask --i-alignment tree_out/rep_seqs_filt_aligned.qza --o-masked-alignment tree_out/rep_seqs_filt_aligned_masked.qza
# Running FastTree
qiime phylogeny fasttree --i-alignment tree_out/rep_seqs_filt_aligned_masked.qza --p-n-threads 28 --o-tree tree_out/rep_seqs_filt_aligned_masked_tree
# Adding root to the tree
qiime phylogeny midpoint-root --i-tree tree_out/rep_seqs_filt_aligned_masked_tree.qza --o-rooted-tree tree_out/rep_seqs_filt_aligned_masked_tree_rooted.qza
```
## Producing rarefaction curves
```shell
# Without metadata
qiime diversity alpha-rarefaction --i-table dada2out/table_filt.qza --p-max-depth 2000 --p-steps 20 --i-phylogeny tree_out/rep_seqs_filt_aligned_masked_tree_rooted.qza --o-visualization rarefaction_curves_eachsample.qzv
# With metadata
qiime diversity alpha-rarefaction --i-table dada2out/table_filt.qza --p-max-depth 2000 --p-steps 20 --i-phylogeny tree_out/rep_seqs_filt_aligned_masked_tree_rooted.qza --m-metadata-file Metadata.txt --o-visualization rarefaction_curves.qzv
```
Tables and plots by category in metadata (you will need to produce new metadata files per category, see examples for cultivar and rootstock)
```shell
# Cultivar
qiime feature-table group --i-table dada2out/table.qza --p-axis sample --p-mode sum --m-metadata-file Metadata.txt --m-metadata-column cultivar --o-grouped-table dada2out/table_filt_cultivar.qza
qiime taxa barplot --i-table dada2out/table_filt_cultivar.qza --i-taxonomy Classifiers/Taxonomy.qza --m-metadata-file Metadata_cultivar.txt --o-visualization taxa_barplot_cultivar.qzv
# Rootstock
qiime feature-table group --i-table dada2out/table.qza --p-axis sample --p-mode sum --m-metadata-file Metadata.txt --m-metadata-column rootstock --o-grouped-table dada2out/table_filt_rootstock.qza
qiime taxa barplot --i-table dada2out/table_filt_rootstock.qza --i-taxonomy Classifiers/Taxonomy.qza --m-metadata-file Metadata_rootstock.txt --o-visualization taxa_barplot_rootstock.qzv
# Species
qiime feature-table group --i-table dada2out/table.qza --p-axis sample --p-mode sum --m-metadata-file Metadata.txt --m-metadata-column species --o-grouped-table dada2out/table_filt_species.qza
qiime taxa barplot --i-table dada2out/table_filt_species.qza --i-taxonomy Classifiers/Taxonomy.qza --m-metadata-file Metadata_species.txt --o-visualization taxa_barplot_species.qzv
# Depth
qiime feature-table group --i-table dada2out/table.qza --p-axis sample --p-mode sum --m-metadata-file Metadata.txt --m-metadata-column depth --o-grouped-table dada2out/table_filt_depth.qza
qiime taxa barplot --i-table dada2out/table_filt_depth.qza --i-taxonomy Classifiers/Taxonomy.qza --m-metadata-file Metadata_depth.txt --o-visualization taxa_barplot_depth.qzv
```
## Calculation of diversity metrics and ordination plots
```shell
qiime diversity core-metrics-phylogenetic --i-table dada2out/table_filt.qza --i-phylogeny tree_out/rep_seqs_filt_aligned_masked_tree_rooted.qza --p-sampling-depth 500 --m-metadata-file Metadata.txt --p-n-jobs-or-threads 28 --output-dir Diversity                      
```
Identification of deferentially abundant features using ANCOM
```shell
qiime composition add-pseudocount --i-table dada2out/table_filt.qza --o-composition-table dada2out/table_filt_pseudocount.qza
# Per category
qiime composition ancom --i-table dada2out/table_filt_pseudocount.qza --m-metadata-file Metadata.txt --o-visualization cultivar --m-metadata-column cultivar
qiime composition ancom --i-table dada2out/table_filt_pseudocount.qza --m-metadata-file Metadata.txt --o-visualization rootstock --m-metadata-column rootstock
qiime composition ancom --i-table dada2out/table_filt_pseudocount.qza --m-metadata-file Metadata.txt --o-visualization species --m-metadata-column species
qiime composition ancom --i-table dada2out/table_filt_pseudocount.qza --m-metadata-file Metadata.txt --o-visualization depth --m-metadata-column depth
```
## Exporting final abundance and sequence files
```shell
qiime tools export --input-path dada2out/table_filt.qza --output-path dada2_output_exported
qiime tools export --input-path dada2out/rep_seqs_filt.qza --output-path dada2_output_exported
```
