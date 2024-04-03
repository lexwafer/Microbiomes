# Performing basic QC for files
## Request a full node in Owens
```shell
sinteractive -c 28 -t 00:30:00 -J QC -A PAS2658
```
## Go to directory QC:
```shell
cd /fs/scratch/PAS2658/Alexis/Microbiomes/QC
mkdir FastQC_reports MultiQC_reports

# extracting fastq files from Raw_Data
find /fs/scratch/PAS2658/Alexis/Microbiomes/Raw_Data/reads_fastq/ -name "*fastq.gz" -exec cp {} . \;
ls *fastq.gz
```

## We will start with the QC of the data using FastQC and MultiQC:
```shell
apptainer exec ../Software/FastQC.sif fastqc *fastq.gz --outdir=FastQC_reports
apptainer exec ../Software/MultiQC.sif multiqc FastQC_reports --interactive -o MultiQC_reports
# Check your reports. Do you notice something you should be worried about?


```
We will need to do some serious trimming here, according to reports


# You will need to produce a tab separated file with the names of the samples (check the file used to download the data) and with the paths get the files, the file should look like the lines below (first line are headers). It is called manifest file:
```shell
# Create the header
echo -e "sample-id\tforward-absolute-filepath\treverse-absolute-filepath" > Manifest.tsv

# Append data to the file
echo -e "sample-1\tfilepath/sample0_R1.fastq.gz\tfilepath/sample1_R2.fastq.gz" >> Manifest.tsv
echo -e "sample-2\tfilepath/sample2_R1.fastq.gz\tfilepath/sample2_R2.fastq.gz" >> Manifest.tsv
echo -e "sample-3\tfilepath/sample3_R1.fastq.gz\tfilepath/sample3_R2.fastq.gz" >> Manifest.tsv
echo -e "sample-4\tfilepath/sample4_R1.fastq.gz\tfilepath/sample4_R2.fastq.gz" >> Manifest.tsv


# Getting basic files
find -name "*1_fastq.gz" > sample-id.txt
ls *_2.fastq.gz > paths_2.txt
find /fs/scratch/PAS2658/Alexis/Microbiomes/Raw_Data/reads_fastq/ -name "*1.fastq.gz" > forward-absolute-filepath.txt
find /fs/scratch/PAS2658/Alexis/Microbiomes/Raw_Data/reads_fastq/ -name "*2.fastq.gz" > reverse-absolute-filepath.txt


```
