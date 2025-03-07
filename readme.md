# hash-cgMLST
A tool for core-genome MLST typing for bacterial data. This has been initially developed for Clostridium difficile, but could be adpated to other bacteria.

## Dependencies
* Singularity - instructions can be found here https://github.com/sylabs/singularity/blob/master/INSTALL.md
* Java version 8 or later (required for nextflow)
* Nextflow - https://www.nextflow.io

## System requirements
This workflow will run on systems that support the dependencies above. It has been tested on MacOS, Ubuntu and CentOS. Minimal system resources are required to generate hash-cgMLST profiles alone, however generating assemblies using SPAdes is the main resource constraint for the whole pipeline, the amount of memory to use per core can be set in the nextflow.config file, e.g. 8 Gb.

## Installation

### Singularity

Build image
```
cd singularity
sudo singularity build hash-cgmlst.img Singularity
```

If you do not have sudo access to your compute machine, the image can be built on another installation and copied across.

The image can be mounted and tested if desired:
```
singularity shell hash-cgmlst.img
```

### miniKraken2
Fetch the miniKraken2 database, from the root of the repository
```
mkdir minikraken2
cd minikraken2
wget ftp://ftp.ccb.jhu.edu/pub/data/kraken2_dbs/minikraken2_v1_8GB_201904_UPDATE.tgz
tar -xzf minikraken2_v1_8GB_201904_UPDATE.tgz
rm minikraken2_v1_8GB_201904_UPDATE.tgz
cd minikraken2_v1_8GB
mv * ../
cd ..
rmdir minikraken2_v1_8GB
```

### cgmlst.org scheme
A version of this is included - it can be updated if desired, but this is not required for hash-cgMLST.

To update this, visit cgmlst.org and download the allele files for each gene to `ridom_scheme/files` and then run:
```
cd bin
python makeReferenceDB.py
```


## Running hash-cgMLST
At present the scripts are designed to be run on two specific server set-ups, each with its own profile. Updates can be made to the nextflow.config file for other set-ups. 

The two profiles are:
* cluster - an example implementation for a SGE based cluster
* ophelia - an example for a local bare-metal server (would adapt to a stand-alone machine if the amount of memory available per core is changed)

### Example input file types
Two sources of raw read files for the workflow are supported
* ebi - supports running workflow on files downloaded from EBI
* local - support running on local gzipped paired fastq files

(The third option bam can be ignored as it has been developed for local testing only.)

To set up two test files for testing the local input option, the following command or similar can be run:

```
mkdir comparison_study_data
cd comparison_study_data
wget ftp://ftp.sra.ebi.ac.uk/vol1/fastq/ERR257/ERR257052/ERR257052_1.fastq.gz -O a.1.fq.gz
wget ftp://ftp.sra.ebi.ac.uk/vol1/fastq/ERR257/ERR257052/ERR257052_2.fastq.gz -O a.2.fq.gz
wget ftp://ftp.sra.ebi.ac.uk/vol1/fastq/ERR257/ERR257068/ERR257068_1.fastq.gz -O b.1.fq.gz
wget ftp://ftp.sra.ebi.ac.uk/vol1/fastq/ERR257/ERR257068/ERR257068_2.fastq.gz -O b.2.fq.gz
```

Examples of the csv files that specify the input files for the pipeline can be found in the `example_data` folder:
* `example_input.csv` obtain 2 files from EBI by accession number, leave columns fq1 and fq2 empty
* `example_input_local.csv` obtain files from local path relative to root of this repository specified in fq1 and fq2, file_name column is used a readable name for the files
* `example_input_bam.csv` ignore
* `example_six_hospitals.csv` re-run the analysis of samples from six hospitals used in the paper describing hash-cgMLST
* `all_cdiff_ncbi_20190611.csv` run the analysis on all C diff in EBI/NCBI as of 6 June 2019

### Example command to run workflow

To run the cgMLST analysis, e.g. using files from EBI
```
nextflow hash-cgMLST.nf --seqlist example_data/example_input.csv --outputPath comparison_study_data/example_output -resume -profile ophelia
```

This will download 2 example pairs of fastq files from EBI and process these through the pipeline. Please see `example_data/example_input.csv` for an example of an input file. The file type column should be set to ebi to download from EBI and the file_name column should be a sample or run identifier. 

Alternatively
```
nextflow hash-cgMLST.nf --seqlist example_data/example_input_local.csv --outputPath comparison_study_data/example_output -resume -profile ophelia
```

Runs the locally downloaded files from above.


## Outputs
Outputs provided include:
 - QC data: `*_base_qual.txt`, `*_kraken.txt`, `*_length.txt`, `*.raw_fastqc.html`, `*.clean_fastqc.html`
 - Spades contigs and assembly stats: `*_spades_contigs.fa`, `*_cgmlst.stats`
 - cgMLST genes as a multifasta file: `*_cgmlst.fa`
 - standard cgMLST calls as a json file: `*_cgmlst.profile` (no tracking of novel alleles is done, just recorded missing for now)
 - hash-cgMLST calls as a json file: `*_cgmlst.json`
 - standard MLST calls: `*_mlst.txt`


## Comparison scripts
To run a comparison for hash-cgMLST profiles after the nextflow pipeline above is complete use the `bin/compareProfiles.py` script:
```
bin/compareProfiles.py -i comparison_study_data/example_output -o  comparison_study_data/example_compare.txt
```

The `bin/compareProfilesExclude.py` script ignores the 26 genes likely prone to mis-assembly.

These scripts search for json files with the folder specified and any immediate sub-folders. Please ensure that the only json files present are those created by this workflow.

## Bring your own assemblies
If you wish to simply call the hash-cgMLST of an existing assembly this can be done by running:

```
bin/getCoreGenomeMLST.py -f assembly_contigs.fa \
	-n contig_name \
	-s ridom_scheme/files \
	-d ridom_scheme/ridom_scheme.fasta \
	-o output_file_prefix \
	-b blastn_path 
```

Here `assembly_contigs.fa` is the input contigs file, `contig_name` is the name to use for the assembly in the output json files, `ridom_scheme/files` is the path to files for the ridom scheme and `ridom_scheme/ridom_scheme.fasta` a fasta file of each gene in the ridom scheme, `output_file_prefix` needs to be set as does the path to the blastn binary `blastn_path`.

If running outside of the Singularity container then a local installation of blastn, python 3 and biopython is required.

The resulting summary files can be compared as above.


## Limitations
### File size
Each downloaded set of gzipped fastq files will be stored in the working directory, as will a pair of gzipped cleaned fastq files. You will need to periodically delete your working directory to prevent it from growing too large for very large projects.