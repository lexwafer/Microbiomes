Login in OSC
```shell
ssh wafer2@osc.edu
#enter password
````

## Preparing basic directory structure
Directories:
```shell
cd /fs/scratch/PAS2658/Alexis
mkdir Microbiomes
cd Microbiomes
mkdir Raw_Data Software QC Bacterial Fungal
```
Installing software:
```shell
cd Software

# ENA FTP downloader
wget http://ftp.ebi.ac.uk/pub/databases/ena/tools/ena-file-downloader.zip
unzip ena-file-downloader.zip
rm ena-file-downloader.zip

# Get container of Qiime2 Reproducible, interactive, scalable and extensible microbiome data science using QIIME 2. Nature Biotechnology 37:852–857. [https://doi.org/10.1038/s41587-019-0209-9)]
apptainer pull Qiime2.sif docker://quay.io/qiime2/amplicon:2024.2

# Testing Qiime2
apptainer exec Qiime2.sif qiime

# Getting FastQC and MultiQC containers
apptainer pull FastQC.sif docker://quay.io/biocontainers/fastqc:0.12.1--hdfd78af_0
apptainer pull MultiQC.sif docker://quay.io/biocontainers/multiqc:1.21--pyhdfd78af_0

# Does they work?
apptainer exec FastQC.sif fastqc --help
apptainer exec MultiQC.sif multiqc --help
```

## Let's intall Qiime using conda, so we can work with ITSexpress for the fungal data
```shell
module load miniconda3/23.3.1-py310
wget https://data.qiime2.org/distro/amplicon/qiime2-amplicon-2024.2-py38-linux-conda.yml
conda env create -n qiime2-amplicon-2024.2 --file qiime2-amplicon-2024.2-py38-linux-conda.yml
rm qiime2-amplicon-2024.2-py38-linux-conda.yml
conda activate qiime2-amplicon-2024.2
qiime --help
conda install -c bioconda itsxpress
pip install q2-itsxpress
qiime dev refresh-cache
qiime
qiime q2-itsxpress
```

## Get the data from ENA-SRA, try this first, if it Does not work for you go to the option of copying the files from another directory
```shell
cd /fs/scratch/PASXXXX/Your_OSC_ID/Microbiomes/Raw_Data

# Download the data from ENA for BioProject PRJEB45622
wget 'https://www.ebi.ac.uk/ena/portal/api/filereport?accession=PRJEB45622&result=read_run&fields=study_accession,fastq_md5,fastq_ftp,sample_alias,sample_title&format=tsv&download=true&limit=0' -O - | tee ENA_Run_Info.tsv

# Dowload the data
module load java/1.8.0_131
java -jar ../Software/ena-file-downloader.jar --accessions=ENA_Run_Info.tsv --format=READS_FASTQ --protocol=FTP --asperaLocation=null --email wafer.2@osu.edu --location=$PWD
