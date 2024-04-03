# Using Qiime2 to analyze bacterial communites
## You should have a manifest file already! Make sure it is in your working directory

## Requesting resources:
```shell
sinteractive -c 28 -t 01:00:00 -J Bacterial -A PAS2658
```

## Importing fastq files for use in Qiime
```shell
cd /fs/scratch/PAS2658/Alexis/Microbiomes/Bacterial
apptainer run --writable-tmpfs ../Software/Qiime2.sif qiime tools import --type 'SampleData[PairedEndSequencesWithQuality]' --input-path Manifest.tsv --output-path Bacterial.qza --input-format PairedEndFastqManifestPhred33V2
```
## Removing primers
```shell
mkdir Adapter_Trimming
apptainer run --writable-tmpfs ../Software/Qiime2.sif qiime cutadapt trim-paired --i-demultiplexed-sequences Bacterial.qza --p-anywhere-f ACGCGHNRAACCTTACC --p-match-read-wildcards --p-anywhere-r ACGGGCRGTGWGTRCAA --p-match-adapter-wildcards --o-trimmed-sequences Adapter_Trimming/Bacterial_trimmed.qza --p-cores 27 --verbose
```
## Filtering reads (around 20 minutes)
```shell
mkdir Filtered
apptainer run --writable-tmpfs ../Software/Qiime2.sif qiime quality-filter q-score --i-demux Adapter_Trimming/Bacterial_trimmed.qza --p-min-quality 28 --o-filter-stats Filtered/filt_stats.qza --o-filtered-sequences Filtered/trimmed_filtered.qza
```
## Do you want to assess the quality? Then, use the following: 
```shell
mkdir Assessment
apptainer run --writable-tmpfs ../Software/Qiime2.sif qiime tools export --input-path Filtered/trimmed_filtered.qza --output-path Assessment
mkdir Assessment/FastQC_assessment Assessment/MultiQC_assessment
apptainer exec ../Software/FastQC.sif fastqc -t 27 Assessment/*fastq.gz --outdir=Assessment/FastQC_assessment
apptainer exec ../Software/MultiQC.sif multiqc Assessment/FastQC_assessment -o Assessment/MultiQC_assessment
```
## Read correction and amplicon sequence variants (ASVs) identification for Bacteria using Deblur (around 15 minutes)
```shell
apptainer run --writable-tmpfs ../Software/Qiime2.sif qiime deblur denoise-16S --i-demultiplexed-seqs Filtered/trimmed_filtered.qza --p-trim-length 260 --p-sample-stats --p-jobs-to-start 27 --p-min-reads 1 --output-dir deblur_output
```
## Convert the Qiime QZA format (for handling in Qiime) to TSV (tab-separated file) for easier visualization 
```shell
apptainer run --writable-tmpfs ../Software/Qiime2.sif qiime tools export --input-path deblur_output/stats.qza --output-path deblur_output
```
## Summarize table.qza
```shell
apptainer run --writable-tmpfs ../Software/Qiime2.sif qiime feature-table summarize --i-table deblur_output/table.qza --o-visualization deblur_output/table_summary.qzv
```
## Filtering out rare ASVs
```shell
# Filtering by features
apptainer run --writable-tmpfs ../Software/Qiime2.sif qiime feature-table filter-features --i-table deblur_output/table.qza --p-min-frequency 0 --p-min-samples 1 --o-filtered-table deblur_output/table_filt.qza
# Filtering by sequences
apptainer run --writable-tmpfs ../Software/Qiime2.sif qiime feature-table filter-seqs --i-data deblur_output/representative_sequences.qza --i-table deblur_output/table_filt.qza --o-filtered-data deblur_output/rep_seqs_filt.qza
```
## Building a phylogeny
```shell
# Multiple sequence alignment using MAFFT
mkdir tree_out
apptainer run --writable-tmpfs ../Software/Qiime2.sif qiime alignment mafft --i-sequences deblur_output/rep_seqs_filt.qza --p-n-threads 28 --o-alignment tree_out/rep_seqs_filt_aligned.qza
# Filtering multiple sequence alignment
apptainer run --writable-tmpfs ../Software/Qiime2.sif qiime alignment mask --i-alignment tree_out/rep_seqs_filt_aligned.qza --o-masked-alignment tree_out/rep_seqs_filt_aligned_masked.qza
# Running FastTree
apptainer run --writable-tmpfs ../Software/Qiime2.sif qiime phylogeny fasttree --i-alignment tree_out/rep_seqs_filt_aligned_masked.qza --p-n-threads 28 --o-tree tree_out/rep_seqs_filt_aligned_masked_tree
# Adding root to the tree
apptainer run --writable-tmpfs ../Software/Qiime2.sif qiime phylogeny midpoint-root --i-tree tree_out/rep_seqs_filt_aligned_masked_tree.qza --o-rooted-tree tree_out/rep_seqs_filt_aligned_masked_tree_rooted.qza
```
## Producing rarefaction curves-start here after 4/3
```shell
# Without metadata
apptainer run --writable-tmpfs ../Software/Qiime2.sif qiime diversity alpha-rarefaction --i-table deblur_output/table_filt.qza --p-max-depth 6000 --p-steps 20 --i-phylogeny tree_out/rep_seqs_filt_aligned_masked_tree_rooted.qza --o-visualization rarefaction_curves_eachsample.qzv
# With metadata, copy metadata file from here: cp /fs/ess/PASXXXX/HCS7004_Files/Microbiomes/Metadata.txt The actual datasets are in Zoe's GitHub (https://github.com/zoemigicovsky/cool_climate_grape_microbiome).
apptainer run --writable-tmpfs ../Software/Qiime2.sif qiime diversity alpha-rarefaction --i-table deblur_output/table_filt.qza --p-max-depth 6000 --p-steps 20 --i-phylogeny tree_out/rep_seqs_filt_aligned_masked_tree_rooted.qza --m-metadata-file Metadata.txt --o-visualization rarefaction_curves_metadata.qzv
```
## Taxonomy assignment
```shell
mkdir Classifiers
# Downloading classifier files for 515F/806R, this file was prepared by Qiime collaborators form the SILVA database (https://www.arb-silva.de)
wget https://data.qiime2.org/2024.2/common/silva-138-99-515-806-nb-classifier.qza -O Classifiers/silva_16S_515F_806R.qza
# Running classification, it takes a while (you may need to request compuing power again)
apptainer run --writable-tmpfs ../Software/Qiime2.sif qiime feature-classifier classify-sklearn --i-reads deblur_output/representative_sequences.qza --i-classifier Classifiers/silva_16S_515F_806R.qza --p-n-jobs 8 --verbose --output-dir Bacterial_taxa/
# Visualization
apptainer run --writable-tmpfs ../Software/Qiime2.sif qiime metadata tabulate --m-input-file Bacterial_taxa/classification.qza --o-visualization classification.qzv
# Exporting the output
apptainer run --writable-tmpfs ../Software/Qiime2.sif qiime tools export --input-path Bacterial_taxa/classification.qza --output-path Taxa
# Stacking bar plot
apptainer run --writable-tmpfs ../Software/Qiime2.sif qiime taxa barplot --i-table deblur_output/table_filt.qza --i-taxonomy Bacterial_taxa/classification.qza --m-metadata-file Metadata.txt --o-visualization Bacterial_taxa/taxa_barplot.qzv
```
## Look for contaminants and clean
```shell
apptainer run --writable-tmpfs ../Software/Qiime2.sif qiime taxa filter-table --i-table deblur_output/table_filt.qza --i-taxonomy Bacterial_taxa/classification.qza --p-exclude Chloroplast --o-filtered-table deblur_output/table_filt_contam.qza
# New stacking plot
apptainer run --writable-tmpfs ../Software/Qiime2.sif qiime taxa barplot --i-table deblur_output/table_filt.qza --i-taxonomy Bacterial_taxa/classification.qza --m-metadata-file Metadata.txt --o-visualization Bacterial_taxa/taxa_barplot_nocom.qzv
```
## Tables and plots by category in metadata
```shell
# Cultivar
mkdir sample_composition
apptainer run --writable-tmpfs ../Software/Qiime2.sif qiime feature-table group --i-table deblur_output/table_filt_contam.qza --p-axis sample --p-mode sum --m-metadata-file Metadata.txt --m-metadata-column cultivar --o-grouped-table sample_composition/table_filt_cultivar.qza
apptainer run --writable-tmpfs ../Software/Qiime2.sif qiime taxa barplot --i-table sample_composition/table_filt_cultivar.qza --i-taxonomy Bacterial_taxa/classification.qza --m-metadata-file sample_composition/table_filt_cultivar.qza --o-visualization sample_composition/taxa_barplot_cultivar.qzv
# Rootstock
apptainer run --writable-tmpfs ../Software/Qiime2.sif qiime feature-table group --i-table deblur_output/table_filt_contam.qza --p-axis sample --p-mode sum --m-metadata-file Metadata.txt --m-metadata-column rootstock --o-grouped-table sample_composition/table_filt_rootstock.qza
apptainer run --writable-tmpfs ../Software/Qiime2.sif qiime taxa barplot --i-table sample_composition/table_filt_rootstock.qza --i-taxonomy Bacterial_taxa/classification.qza --m-metadata-file sample_composition/table_filt_rootstock.qza --o-visualization sample_composition/taxa_barplot_rootstock.qzv
# Species
apptainer run --writable-tmpfs ../Software/Qiime2.sif qiime feature-table group --i-table deblur_output/table_filt_contam.qza --p-axis sample --p-mode sum --m-metadata-file Metadata.txt --m-metadata-column species --o-grouped-table sample_composition/table_filt_species.qza
apptainer run --writable-tmpfs ../Software/Qiime2.sif qiime taxa barplot --i-table sample_composition/table_filt_species.qza --i-taxonomy Bacterial_taxa/classification.qza --m-metadata-file sample_composition/table_filt_species.qza --o-visualization sample_composition/taxa_barplot_species.qzv
# Depth
apptainer run --writable-tmpfs ../Software/Qiime2.sif qiime feature-table group --i-table deblur_output/table_filt_contam.qza --p-axis sample --p-mode sum --m-metadata-file Metadata.txt --m-metadata-column depth --o-grouped-table sample_composition/table_filt_depth.qza
apptainer run ../Software/Qiime2.sif qiime taxa barplot --i-table sample_composition/table_filt_depth.qza --i-taxonomy Bacterial_taxa/classification.qza --m-metadata-file sample_composition/table_filt_depth.qza --o-visualization sample_composition/taxa_barplot_depth.qzv
```
## Calculation of diversity metrics and ordination plots
```shell
# you might need computing resources again
apptainer run --writable-tmpfs ../Software/Qiime2.sif qiime diversity core-metrics-phylogenetic --i-table deblur_output/table_filt_contam.qza --i-phylogeny tree_out/rep_seqs_filt_aligned_masked_tree_rooted.qza --p-sampling-depth 500 --m-metadata-file Metadata.txt --p-n-jobs-or-threads 27 --output-dir Diversity
```
## Identification of deferentially abundant features using ANCOM
```shell
apptainer run --writable-tmpfs ../Software/Qiime2.sif qiime composition add-pseudocount --i-table deblur_output/table_filt_contam.qza --o-composition-table deblur_output/table_filt_pseudocount.qza
# Per category
tee -a Bacterial_ANCOM_depth.sh <<EOF
#!/bin/bash
#SBATCH -J ondemand/sys/myjobs/basic_sequential
#SBATCH --job-name Bacterial_ANCOM_depth
#SBATCH --time=04:00:00
#SBATCH --ntasks=28
#SBATCH --exclusive
#SBATCH --mail-type=BEGIN,END,FAIL
#SBATCH --account=PASXXXX
cd /fs/scratch/PASXXXX/Your_OSC_ID/Microbiomes/Bacterial
apptainer run --writable-tmpfs ../Software/Qiime2.sif qiime composition ancom --i-table deblur_output/table_filt_pseudocount.qza --m-metadata-file Metadata.txt --o-visualization cultivar --m-metadata-column cultivar
apptainer run --writable-tmpfs ../Software/Qiime2.sif qiime composition ancom --i-table deblur_output/table_filt_pseudocount.qza --m-metadata-file Metadata.txt --o-visualization rootstock --m-metadata-column rootstock
apptainer run --writable-tmpfs ../Software/Qiime2.sif qiime composition ancom --i-table deblur_output/table_filt_pseudocount.qza --m-metadata-file Metadata.txt --m-metadata-column species --o-visualization species
apptainer run --writable-tmpfs ../Software/Qiime2.sif qiime composition ancom --i-table deblur_output/table_filt_pseudocount.qza --m-metadata-file Metadata.txt --m-metadata-column depth --o-visualization depth
EOF
```
## Exporting final abundance (BIOM) and sequence files
```shell
apptainer run --writable-tmpfs ../Software/Qiime2.sif qiime tools export --input-path deblur_output/table_filt_contam.qza --output-path deblur_output_exported
apptainer run --writable-tmpfs ../Software/Qiime2.sif qiime tools export --input-path deblur_output/representative_sequences.qza --output-path deblur_output_exported
```
