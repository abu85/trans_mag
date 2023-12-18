<font size=20>__Metagenome (metatranscriptome) data analyses with nf-cor/mag pipeline (step by step)__</font>

### Contents:
1. [Pipeline summary](#sec1) </br>

2. [metagenome (metatranscriptome) data anaylsis](#sec2)</br>
    2.1. [log in to the server (hpc)](#sec2.1)</br>
    2.2. [Set your working directory and activate modules or packages](#sec2.2)</br>
    2.3. [Tmux overview](#sec2.3)</br>
          2.3.1. [Create a new tmux session (example: mag_trans) so that nextflow session does not collapse](#sec2.3.1)</br>
          2.3.2. [All the modules needs to be loaded from HPC cluster](#sec2.3.2)</br>
    2.4. [Sample sheet and meta data](#sec2.4)</br>
          2.4.1. [Make a soft copy of the raw reads so that main raw reads remain intact/ not touched](#sec2.4.1)</br>
    2.5. [Reference database for taxonomic classification](#sec2.5)</br>
    2.6. [Nextflow (nf-core mag pipeline) parameters and main commands](#sec2.6)</br>
          2.6.1 [MAG pipeline parameters](#sec2.6.1)</br>
          2.6.2. [nf-core mag pipeline execution or commands](#sec2.6.2)</br>
    

nf-core/mag is used here as a bioinformatics best-practise analysis pipeline for assembly, binning and annotation of metagenome (metatranscriptome)s.

![image](https://github.com/abu85/trans/assets/73113157/e199a164-3ec1-4d63-bfa0-250cb11d788f)

<a name="sec1"></a>
## 1. Pipeline summary
By default, the pipeline currently perform on short and long reads, quality trims the reads and adapters with fastp, and performs basic QC with FastQC, and merge multiple sequencing runs.

The pipeline then:
- assigns taxonomy to reads using Kraken2
- performs assembly Megahit or Spades (the script is for megahit) and checks their quality using Quast
- predicts protein-coding genes for the assemblies using Prodigal, and bins with Prokka and optionally MetaEuk
- performs Metagenome (metatranscriptome) binning using Metabat2, MaxBin2, concord (we choose maxbin2) and checks the quality of the genome bins using Busco, or CheckM, and optionally GUNC.
- optionally refines bins with DAS Tool
- assigns taxonomy to bins using GTDB-Tk and/or CAT and optionally identifies viruses in assemblies using geNomad, or Eukaryotes with Tiara
- Furthermore, the pipeline creates various reports in the results directory specified, including a MultiQC report summarizing some of the findings and software versions.

<a name="sec2"></a>
## 2. Metagenomic (metatranscriptome) data anaylsis
<a name="sec2.1"></a>
#### 2.1. Log in to uppmax again (if needed)

<a name="sec2.2"></a>
#### 2.2. Set your working directory and activate modules or packages
cd absulote_path_to_the_working_folder

change working directory absulote_path_to_the_working_folder to 

/path_to/trans/nxf/ # create one with mkdir if it is not there

```
cd /path_to/trans/nxf/
```
<a name="sec2.3"></a>
#### 2.3 Tmux:
You do not want to quit by mistake or system crush then use 'tmux'. Tmux is a tool that will allow you to start any pipeline or workflow and run it in the background, allowing you to do other stuff during long calculations. As an added bonus, it will keep your processes going even if you leave the server or your connection is unstable and crashes. First you needs to be log in virtual (in my case UPPMAX’s module system), after which you can initiate a new terminal in tmux by following commands:

module load tmux # Now a new setup will pop up, anything you do in this new tmux terminal session is “safe”. When the connection to the server crashes mid-session, just reconnect to UPPMAX and do

<a name="sec2.3.1"></a>
#### 2.3.1 Create a new tmux session (example: mag_trans) so that nextflow session does not collapse
tmux new -s mag_trans

tmux set mouse on # enable mouse support for things like scrolling and selecting text

To put tmux in background and keep the processes inside running, press Ctrl+B, release, press D. With tmux ls you can see which sessions are ongoing # (can be multiple ones!) and you could connect to. To reattach to your earlier session type tmux attach -t mag_nxf as shown below.

To kill a tmux session and stop any process running in it, press Ctrl+B, release, press X followed by Y.

if you like to enter into previous ongoing session tmux attach -t mag_nxf

All of this might seem to add unnecessary hassle but tmux is extremely valuable when working on a server. Instead of having to redo a long list of computational step when the connection to a server inevitably crashes, just reconnect to the ongoing tmux session and you are back exactly where you were when the crash happened! Tmux actually can do even more useful things, so if you want to know more, have a look at this quick and easy guide to tmux:https://www.hamvocke.com/blog/a-quick-and-easy-guide-to-tmux/.

<a name="sec2.3.2"></a>
#### 2.3.2 All the modules needs to be loaded from HPC cluster 
(here it is shown for UPPMAX computer cluster )

module load uppmax bioinfo-tools # load base UPPMAX environment modules, needed for everything else
module load Nextflow # load Nextflow,  Note: Capital N!
module load nf-core-pipelines/latest # load nf-core nexflow pipelines

Limit the catche and directory for temporary files: 
export NXF_OPTS='-Xms1g -Xmx4g'
####### export NXF_HOME=/absulote_path_to_the_working_directory , for me it was: 
export NXF_HOME=/path_to/trans/nxf/
######### to store temporary file:
export NXF_TEMP=${SNIC_TMP:-$HOME/glob/nxftmp}

<a name="sec2.4"></a>
#### 2.4. Sample sheet and meta data
Make youre own sample sheet and meta data according to your sample names, fastq file names, path (samples raw files path) and experimental setup (for metadata file), make sure that 1st column of the sample sheet and metadata should be identical. see [usage](https://nf-co.re/mag/2.5.1/docs/usage#running-the-pipeline)

###### import samplesheet to the server to use it:
scp /source/trans/data_mag/samplesheet.csv username@node.cluster.uni.country:/path_destination/

get host genome to filter your reads:
[GRCh37](https://s3.amazonaws.com/igenomes.illumina.com/Homo_sapiens/Ensembl/GRCh37/Homo_sapiens_Ensembl_GRCh37.tar.gz)

<a name="sec2.4.1"></a>
#### 2.4.1. Make a soft copy of the raw reads so that main raw reads remain intact/ not touched.
they are here /path_to/trans/

made a soft links for raw reads
```
mkdir /path_to/trans/raw_reads_soft_link

cd /path_to/trans/raw_reads_soft_link
 
ln -s /path_to/trans/*.fastq.gz /path_to/trans/raw_reads_soft_link/
``` 

<a name="sec2.5"></a>
### 2.5. Reference database for taxonomic classification

We have used a prebuilt standard databases for k2 classification, chose one that fits your experimantal questions. The db included here is **Refeq archaea, bacteria, viral, plasmid, human1, UniVec_Core**. See details of the database [here](https://benlangmead.github.io/aws-indexes/k2), mentioned as Standard in the table. The db was created on 2023-06-05 & accessed on  2023-12-*.

The db is here:
/sw/data/Kraken2_data/prebuilt/k2_standard_20230605/


<a name="sec2.6"></a>
## 2.6 Nextflow parameters and main commands

<a name="sec2.6.1"></a>
### 2.6.1 MAG parameters
You always have your own parameters (such as spades or megahit assembler) so change it in the nf-params.json file

<a name="sec2.6.2"></a>
### 2.6.2 nf-core/mag pipeline or command
```
cd /path_to/trans/
mkdir nxf
cd /path_to/trans/nxf
```
 
You can run this pipeline with just Nextflow installed by copying the JSON below to a file called 'nf-params.json':
```
{
    "input": "\/path_to_samplesheet\/nobackup\/abu\/samplesheet.csv",
    "outdir": "\/path_to_output\/output",
    "email": "email@slu.se",
    "multiqc_title": "trans_report",
    "host_genome": "GRCh37", #if you want humuan data removal, in case you have contamination
    "kraken2_db": "/path_to/data/Kraken2_data/latest_nt/", #you can define reference data, for bacteria use bacteria db from rackham prebuilt db
    "spades_options": "-k 21,33",# to make less memory intensive
    "metaeuk_mmseqs_db": "eggNOG", # functinal annotation
    "skip_metabat2": true,# skip metabat2
    "bowtie2_mode": "--very-fast",
    "run_gunc": true # bin qc
}
```
Then, launch Nextflow with the following command from the working directory in your server:

###### 1st run:
```
nextflow run nf-core/mag -r 2.5.1 -name trans -profile uppmax \
--project hpc_cluster -params-file nf-params.json \
--coassemble_group \# computation intensive, remove this parameter if you want assembly on indvidual sample
--host_removal_save_ids \# if needed
--save_hostremoved_reads # if needed
```

#### sample run

```
nextflow run nf-core/mag -r 2.5.1 -name trans_v1 -profile uppmax --project snic**** -params-file nf-params.json --coassemble_group --host_removal_save_ids --save_hostremoved_reads --kraken2_db "/sw/data/Kraken2_data/prebuilt/k2_standard_20230605/" -resume --skip_spades --megahit_options "--cleaning-rounds 1" "--disconnect-ratio 0" "--continue -o out" --skip_concoct > log_v1_run.txt
```

This is the workflow template created by @Abu Bakar Siddique for @SLUBI Facility.
