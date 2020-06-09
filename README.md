# MARC1 analysis pipeline

This is a pipeline for analysis of MARC1 genotyping data, originally described in [Developmental barcoding of whole mouse via homing CRISPR](https://science.sciencemag.org/content/361/6405/eaat9804.long) and detailed in a forthcoming Nature Protocols paper. It was designed to run on a Unix-based system, and has been tested on OSX and Fedora 31. 

The pipeleline compiles paired-end reads and filters sequencing errors to prepare the output of Illumina sequencing data for genotyping or more in-depth barcoding analysis. It runs entirely from the terminal and relies on a combination of Perl, the R programming language, and a small number of specific R packages.

## required software

* A Unix system, basic familiarity with terminal commands and operation
* A basic R installation
* speciality R packages:
  - stringdist
  - VGAM

# How to use

The pipeline is composed of 4 sequential analysis parts. Each step is run from within a folder and acts on the output of the previous step; they should be run in sequential order.

* /0-raw_data_PB  
* /1-pair_counting 
* /2-ID_err_correction
* /3-SP_err_correction  
* /4-pair_filtering 

## 0 - Set up an analysis workspace
  ```
 $ mkdir analysis 
 $ cd analysis/
 $ git clone https://github.com/Kalhor-Lab/MARC1-Pipeline-NatProtoc.git
 ```
  
Move demultiplexed FASTQ files, two for each sample, to /0-raw_dataPB. If FASTQ files are compressed, decompress with:
 ```
 $ gunzip *.gz
 ```

## 1 - Compile paired end sequences from each sample to list of identifiers
This step is requires a commitment of system resources; we typically run this analysis on a cluster. Check that PB7-founder and PB3-founder files are included, as they provide both analysis controls and are used later in the pipeline. 
**For Linux Users** This step defaults to an OSX blat version; pass "linux" to submit.sh to use the correct version. OSX users do not need to modify anything.

In 1-pair_counting:
  ```
  $ chmod +x submit.sh
  $ ./submit.sh [linux]
  ```
## 2 - correct sequencing errors associated with identifiers 

In /2-ID_err_corrections:
  ```
  $ Rscript 200514_filter-identifiers.R
  ```

## 3 - Correct sequencing errors associated with spacers

For each sample, this generates: 
  * _[sample]\_genotypes.txt_  - lists each identifier and the most common associated spacer, useful for genotyping applications
  * _[sample]\_allpairs.txt_  - lists all identifier-spacer pairs with observation counts without subjective filtering; this is useful for downstream analysis applications
  
  ```
  $  Rscript 200514_filter-spacers.R
  ```
  
## 4 - Filtering identifier-spacer pairs & reporting on mutation levels

Filtering as presented here is subjective; parameters were designed based on our experience and current best understanding of error correction tactics. All parameters are contained within the code and can be modified.
  
**PB3 and PB7 variation** Specific corrections are based on empirically-determined particularities of the PB3 and PB7 sequences, and thus line should be specified accordingly. Add the number in brackets on line 26 for your use case.

PB3 samples, line 26:
  ```
  Founder <- c('PB3', 'PB7')[1]                   
  ```

PB7 samples, line 26:
  ```
  Founder <- c('PB3', 'PB7')[2]                   
  ```
