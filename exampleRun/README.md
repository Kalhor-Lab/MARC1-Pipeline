 
# MARC1 analysis pipeline

This is a pipeline for analysis of MARC1 genotyping data, originally described in [Developmental barcoding of whole mouse via homing CRISPR](https://science.sciencemag.org/content/361/6405/eaat9804.long) and detailed in a forthcoming protocols paper. It was designed to run on a Unix-based system, and has been tested on OSX and Fedora 31. 

The pipeline compiles paired-end reads and filters sequencing errors to prepare the output of Illumina sequencing data for genotyping or more in-depth barcoding analysis. It runs entirely from the terminal and relies on a combination of Perl, the R programming language, and a small number of specific R packages.

### Required software

* A Unix system, basic familiarity with terminal commands and operation
* A standard installation of the R statistical software (version 3.6.1)
* R software packages
  * VGAM (version 1.1.1)
  * stringdist (version 0.9.5.1)


# Usage guide

The pipeline is composed of 4 sequential analysis parts. Each step is run from within a folder and acts on the output of the previous step; they should be run in sequential order.

* 0-raw_data_PB/  
* 1-pair_counting/
* 2-ID_err_correction/
* 3-SP_err_correction/  
* 4-pair_filtering/ 

## Testing the pipeline
For testing, there are two founder files supplied in 0-raw_data_PB. The entire pipeline can be run with those files to ensure software setup is accurate and complete. They are also used for analysis of newly generated data and thus it is important that they are present and correct for all subsequent analysis runs.

## Examples

An example run is provided in exampleRun/. This run contains the output from analyzing three mutant and one genotyping sample in full. 

# Running the pipeline

## 0 Set up an analysis workspace
  ```
 $ mkdir analysis 
 $ cd analysis/
 $ git clone https://github.com/Kalhor-Lab/MARC1-Pipeline.git
 $ cd MARC1-Pipeline
 ```

Move demultiplexed FASTQ files, two for each sample, to /0-raw_data_PB. Decompress FASTQ files with:
 ```
 $ cd 0-raw_data_PB
 $ gunzip *.gz
 ```
## 1 Compile paired end sequences from each sample to list of identifiers
This step is requires some commitment of system resources; we typically run this analysis on a cluster. Check that PB7-founder and PB3-founder files are included, as they provide both analysis controls and are used in step 4 of the pipeline. 

  ```
  $ cd ../1-pair_counting
  $ chmod +x submit.sh
  $ ./submit.sh 
  ```
## 2 Correct sequencing errors associated with identifiers 
This script is compiles a list of high-confidence identifier sequences that exist in each sample. The outputs are written to  _[sample]\_trueID.txt_ .
  ```
  $ cd ../2-ID_err_correction
  $ Rscript Filter-identifiers.R
  ```

## 3 Correct sequencing errors associated with spacers

For each sample, this step generates: 
  * _[sample]\_genotypes.txt_  - lists each identifier and the most common associated spacer, useful for genotyping applications
  * _[sample]\_truepairs.txt_ - lists all identifier-spacer pairs that were confidently assigned to an identifier-spacer group. Column 3 denotes the observed frequency of that pair. 
  * _[sample]\_ambigpairs.txt_ - lists all identifier-spacer pairs that could not be confidently assigned to an identifier-spacer group. Column 4 of this file denotes this source of ambiguity (bc for identifier, sp for spacer).
  * _[sample]\_allpairs.txt_  - lists all identifier-spacer pairs with observation counts without subjective filtering, i.e. the [sample]_allpairs.txt and [sample]_ambigpairs.txt combined. This is useful for downstream analysis applications. 
 
  ```
  $ cd ../3-SP_err_correction
  $ Rscript Filter-spacers.R
  ```
  
## 4 Filtering identifier-spacer pairs, reporting on mutation levels & creating full barcode tables

The script _4-pair_filtering/Final-filtering.R_ starts with the sequencing error-corrected files in 3-SP_err_correction (including the PB3 and PB7 founders) and performs the following:
1) Filters out samples with low coverage (MinRd criterion)
2) Filters out IDs with low coverage (PFCOFF_bc and PFCOFF_bc_exp criteria)
3) Filters out pairs with low read counts (PFCOFF_pair and PRCOFF_pair criteria)
4) Corrects known PCR or sequencing artifacts that are unique to MARC1 samples
5) Corrects template-switching during PCR
6) Corrects early-cycle PCR mutations that can make a non-mutated spacer appear mutated (max_dist_spacer criterion)
7) Corrects one-base displacements due to sequencing error
8) Removes spacers with short reads.
9) Corrects IDs (orphan barcodes) that have been completely or partially deleted due to a large deletion.
This code needs two data files to function properly: _INUSE_AllPB-BarcodesMasterTable.txt_ and _INUSE-barcode_classification.txt_

Filtering as presented here is subjective; parameters were designed based on our experience and current best understanding of error correction tactics. All parameters are contained within the code and can be modified. 


**PB3 and PB7 differences** 
Specific corrections are based on known particularities of the PB3 and PB7 sequences. Additionally, hgRNA sequences in the PB3 and PB7 founders is used to determine whether sequences observed in other samples are mutated or not. Therefore founder strain should be specified accordingly.

Change the at the top of "/4-pair_filtering/Final-Filtering.R" for your use case.

For PB3:
```
  Founder <- c('PB3', 'PB7')[1]                   
  ```
 For PB7:
 ```
  Founder <- c('PB3', 'PB7')[2]                   
  ```
  
Then run:
  ```
  $ cd ../4-pair_filtering
  $ Rscript Final-Filtering.R
  ```
The above commands run the script in batch mode. For users that are familiar with R Studio or R GUI, we recommend running this script interactively using these interfaces to adjust the code according to their needs.

**Truncated or orphan barcodes** In some experiments, some errors in IDs cannot be resolved by automatic filtering and need to be manually accounted for (orphan barcodes). Running "/4-pair_filtering/Final-Filtering.R" will print such IDs in stdout.  To identify the parents of orphan barcodes, you can extract their corresponding spacer sequence from the allbarcodes.txt file in "/3-SP_err_correction/" to determine which of the founder spacers they resemble the most. If you can identify the parents of the orphan barcodes, populate the following vectors (located at the top section of the code) with pairs of orphan barcodes and their real parent barcode and run the code again. If you cannot identify the parent of an orphan barcode, add its identifier to the spurious_barcodes vector to remove it from analysis.
```
trunc_barcodes      <- c()          # trunc_barcodes <- c('[orphan_barcode_1]', '[orphan_barcode_2]', ...)
trunc_barcodes_refs <- c()          # trunc_barcodes_refs <- c('[parental_barcode_for_orphan_barcode_1]', '[parental_barcode_for_orphan_barcode_2]', ...)
```

**Clustering the full barcode table** A full barcode table is created and written to a file (yyyy-mm-dd_BarcodeTable.txt). In this table, each row corresponds to a sample and each column to a spacer allele. The value in each cells represents the abundance of the corresponding spacer allele in the sample. This table can be clustered to obtain a preliminary tree. For instance, for basic clustering based on Manhattan distances run (in R):
```
barcode_table <- read.table(file = 'analysis/4-pair_filtering/yyyy-mm-dd_BarcodeTable.txt', check.names = FALSE)
barcode_table <- barcode_table[,-grep("par", colnames(barcode_table))]   # Removing the parental alleles from the table.
manhattan_distance_matrix <- dist(barcode_table, method = "manhattan")   # Calculating Manhattan (L1) distances of all samples' full barcodes.
dendrogram <- as.dendrogram(hclust(manhattan_distance_matrix, method = "ward.D2")) # Clustering samples based on full-barcode Manhattan distance to create a dendrogram.
plot(dendrogram)    # Plotting the clustering results as a dendrogram.
```
Note that the unresolved identifiers (orphan barcodes) are excluded from the barcode table. The barcode_table object is also automatically generated in the end of Final-filtering.R script.

# Technical troubleshooting

## Step 1 (compiling paired end sequences from each sample to list of identifiers) too slow.

* Use a computing cluster and submit each paired-end file as a separate job.

## Blat versions and step 1

* submit.sh should automatically pass in the correct operating system for both OSX and Linux users. If this fails, submit.sh can be edited manually for your OS.
* OS updates may render the versions of blat provided here for use in step 1 out of date or incompatible. The most up to date versions can be downloaded directly from http://hgdownload.soe.ucsc.edu/admin/exe/. The script submit.sh 

## R versions

* We do not anticipate R versioning being problematic; explicit version numbers are provided for reference only.
