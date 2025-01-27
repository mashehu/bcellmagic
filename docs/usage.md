# nf-core/bcellmagic: Usage

## Table of contents

* [Introduction](#general-nextflow-info)
* [Running the pipeline](#running-the-pipeline)
* [Updating the pipeline](#updating-the-pipeline)
* [Reproducibility](#reproducibility)
* [Main arguments](#main-arguments)
  * [`-profile`](#-profile-single-dash)
    * [`docker`](#docker)
    * [`awsbatch`](#awsbatch)
    * [`standard`](#standard)
    * [`binac`](#binac)
    * [`cfc`](#cfc)
    * [`none`](#none)
  * [Input files](#input-files)
    * [`--metadata`](#--metadata)
    * [`--cprimers`](#--cprimers)
    * [`--vprimers`](#--vprimers)
    * [`--index_file`](#--index_file)
* [Reference Databases](#reference-databases)
  * [`--igblast_base`](#--igblast_base)
  * [`--imgtdb_base`](#--imgtdb_base)
* [Define clones](#Define-clones)
  * [Manually set cluster threshold](#manually-set-cluster-threshold)
  * [Only define clones](#only-define-clones)
* [Job Resources](#job-resources)
* [Automatic resubmission](#automatic-resubmission)
* [Custom resource requests](#custom-resource-requests)
* [AWS batch specific parameters](#aws-batch-specific-parameters)
  * [`-awsbatch`](#-awsbatch)
  * [`--awsqueue`](#--awsqueue)
  * [`--awsregion`](#--awsregion)
* [Other command line parameters](#other-command-line-parameters)
  * [`--outdir`](#--outdir)
  * [`--email`](#--email)
  * [`-name`](#-name-single-dash)
  * [`-resume`](#-resume-single-dash)
  * [`-c`](#-c-single-dash)
  * [`--max_memory`](#--max_memory)
  * [`--max_time`](#--max_time)
  * [`--max_cpus`](#--max_cpus)
  * [`--plaintext_emails`](#--plaintext_emails)
  * [`--sampleLevel`](#--sampleLevel)
  * [`--multiqc_config`](#--multiqc_config)

## General Nextflow info

Nextflow handles job submissions on SLURM or other environments, and supervises running the jobs. Thus the Nextflow process must run until the pipeline is finished. We recommend that you put the process running in the background through `screen` / `tmux` or similar tool. Alternatively you can run nextflow within a cluster job submitted your job scheduler.

It is recommended to limit the Nextflow Java virtual machines memory. We recommend adding the following line to your environment (typically in `~/.bashrc` or `~./bash_profile`):

```bash
NXF_OPTS='-Xms1g -Xmx4g'
```

## Running the pipeline

The typical command for running the pipeline is as follows:

```bash
nextflow run nf-core/bcellmagic -profile standard,docker --metadata metasheet_test.tsv --cprimers CPrimers.fasta --vprimers VPrimers.fasta --max_memory 8.GB --max_cpus 8
```

For more information about the parameters, please refer the corresponding sections.
This will launch the pipeline with the `docker` configuration profile. See below for more information about profiles.

Note that the pipeline will create the following files in your working directory:

```bash
work            # Directory containing the nextflow working files
results         # Finished results (configurable, see below)
.nextflow_log   # Log file from Nextflow
# Other nextflow hidden files, eg. history of pipeline runs and old logs.
```

### Updating the pipeline

When you run the above command, Nextflow automatically pulls the pipeline code from GitHub and stores it as a cached version. When running the pipeline after this, it will always use the cached version if available - even if the pipeline has been updated since. To make sure that you're running the latest version of the pipeline, make sure that you regularly update the cached version of the pipeline:

```bash
nextflow pull nf-core/bcellmagic
```

### Reproducibility

It's a good idea to specify a pipeline version when running the pipeline on your data. This ensures that a specific version of the pipeline code and software are used when you run your pipeline. If you keep using the same tag, you'll be running the same version of the pipeline, even if there have been changes to the code since.

First, go to the [nf-core/bcellmagic releases page](https://github.com/nf-core/bcellmagic/releases) and find the latest version number - numeric only (eg. `1.3.1`). Then specify this when running the pipeline with `-r` (one hyphen) - eg. `-r 1.3.1`.

This version number will be logged in reports when you run the pipeline, so that you'll know what you used when you look back in the future.

## Main Arguments

### `-profile`

Use this parameter to choose a configuration profile. Profiles can give configuration presets for different compute environments. Note that multiple profiles can be loaded, for example: `-profile standard,docker` - the order of arguments is important!

* `standard`
  * The default profile, used if `-profile` is not specified at all.
  * Runs locally and expects all software to be installed and available on the `PATH`.
* `docker`
  * A generic configuration profile to be used with [Docker](http://docker.com/)
  * Pulls software from dockerhub: [`nfcore/bcellmagic`](http://hub.docker.com/r/nfcore/bcellmagic/)
* `singularity`
  * A generic configuration profile to be used with [Singularity](http://singularity.lbl.gov/)
  * Pulls software from singularity-hub
* `conda`
  * A generic configuration profile to be used with [conda](https://conda.io/docs/)
  * Pulls most software from [Bioconda](https://bioconda.github.io/)
* `awsbatch`
  * A generic configuration profile to be used with AWS Batch.
* `test`
  * A profile with a complete configuration for automated testing
  * Includes links to test data so needs no other parameters
* `none`
  * No configuration at all. Useful if you want to build your own config from scratch and want to avoid loading in the default `base` config profile (not recommended).

### Input files

Use this to specify the location of your input files. Three input files are required for running the pipeline: a metadata sheet, the a fasta file containing the primer sequences for the C-region genes (cprimers) and a fasta file containing the primer sequences for the V-region genes (vprimers). This pipeline was originally designed for a special MiSEQ sequencing read setup requiring 3 fastq files: R1 (250bp), R2 (250bp), and I1 (14bp).

* R1: C-Primer + V(D)J
* R2: V-Primer + V(D)J
* I1: Illumina Index (6bp) + UMI (8bp)

The pipeline has been expanded to be able to process data where the UMI and index files are incorporated into the R1 read fastq files (see section `--index_file`)

#### `--metadata`

The metadata file is a TSV file with the following columns, including the exact same headers:

```bash
ID Source Treatment Extraction_time Population R1 R2 I1
QMKMK072AD Patient_2 Drug_treatment baseline p sample_S8_L001_R1_001.fastq.gz sample_S8_L001_R2_001.fastq.gz sample_S8_L001_I1_001.fastq.gz
```

This metadata will then be automatically annotated in a column with the same header in the tables outputed by the pipeline. Where:

* *ID*: sample ID.
* *Source*: patient or organism code.
* *Treatment*: treatment condition applied to the sample.
* *Extraction_time*: time of cell extraction for the sample.
* *Population*: B-cell population (e.g. naive, double-negative, memory, plasmablast).
* *R1*: path to fastq file with first mates of paired-end sequencing.
* *R2*: path to fastq file with second mates of paired-end sequencing.
* *I1*: path to fastq with illumina index and UMI (unique molecular identifier) barcode.

Specify the location of your metadata file like this:

```bash
--metadata 'path/to/metadata/metadata_sheet.tsv'
```

#### `--vprimers`

Path to fasta file containing your V-primer sequences. Specify like this:

```bash
--vprimers 'path/to/vprimers.fasta'
```

#### `--cprimers`

Path to fasta file containing your C-primer sequences. Specify like this:

```bash
--cprimers 'path/to/cprimers.fasta'
```

#### `--index_file``

Indicate if Illumina indices and UMI barcodes are provided in a separate fastq file (index_file true). If Illumina indices and UMI barcodes are integrated into R1 reads, leave the default `--index_file false`.

```bash
--index_file true
```

## Reference databases

By default, the pipeline will download the needed igblast and IMGT human databases unless the path to already downloaded databases is specified. To specify the paths set the `--igblast_base` and `--imgtdb_base` parameters.

### `--igblast_base`

Path to igblast downloaded database. Set as follows:

```bash
--igblast_base 'path/to/igblast_base'
```

### `--imgtdb_base`

Path to imgt downloaded database. Set as follows:

```bash
--imgtdb_base 'path/to/imgtdb_base'
```

## Define clones

By default the pipeline will define clones for each of the samples, as two sequences having the same V gene assignment, C gene assignment, J-gene assignment and junction lenght. Additionally, the similarity of the junction region sequences  will be assessed by hamming distances. A distance threshold for determining if two sequences come from the same clone or not is automatically determined by the process shazam. Alternatively, a hamming distance threshold can be  manually set   by setting the `--set_cluster_threshold` and `--cluster_threshold` parameters as follows:

### Manually set cluster threshold

Set the `--set_cluster_threshold` parameter to allow manual cluster hamming distance threshold definition. Then specify the value in the `--cluster_threshold` parameter as follows:

```bash
--set_cluster_threshold --cluster_threshold 0.14
```

### Only define clones

In some occasions you might just  want to run the pipeline to define clones for  some  samples for which you have already run the rest of the steps. For example, after pulling the results for the same patient together. In this case, run the pipeline setting the `--define_clones_only` parameter, and specify the path to your input Change-O tsv table with the parameter `--changeo_tsv` as follows:

```bash
--define_clones_only --changeo_tsv 'path/to/changeo/tables/*.tab'
```

## Job Resources

### Automatic resubmission

Each step in the pipeline has a default set of requirements for number of CPUs, memory and time. For most of the steps in the pipeline, if the job exits with an error code of `143` (exceeded requested resources) it will automatically resubmit with higher requests (2 x original, then 3 x original). If it still fails after three times then the pipeline is stopped.

### Custom resource requests

Wherever process-specific requirements are set in the pipeline, the default value can be changed by creating a custom config file. See the files hosted at [`nf-core/configs`](https://github.com/nf-core/configs/tree/master/conf) for examples.

If you are likely to be running `nf-core` pipelines regularly it may be a good idea to request that your custom config file is uploaded to the `nf-core/configs` git repository. Before you do this please can you test that the config file works with your pipeline of choice using the `-c` parameter (see definition below). You can then create a pull request to the `nf-core/configs` repository with the addition of your config file, associated documentation file (see examples in [`nf-core/configs/docs`](https://github.com/nf-core/configs/tree/master/docs)), and amending [`nfcore_custom.config`](https://github.com/nf-core/configs/blob/master/nfcore_custom.config) to include your custom profile.

If you have any questions or issues please send us a message on [Slack](https://nf-core-invite.herokuapp.com/).

## AWS Batch specific parameters

Running the pipeline on AWS Batch requires a couple of specific parameters to be set according to your AWS Batch configuration. Please use the `-awsbatch` profile and then specify all of the following parameters.

### `--awsqueue`

The JobQueue that you intend to use on AWS Batch.

### `--awsregion`

The AWS region to run your job in. Default is set to `eu-west-1` but can be adjusted to your needs.

Please make sure to also set the `-w/--work-dir` and `--outdir` parameters to a S3 storage bucket of your choice - you'll get an error message notifying you if you didn't.

## Other command line parameters

### `--outdir`

The output directory where the results will be saved.

### `--email`

Set this parameter to your e-mail address to get a summary e-mail with details of the run sent to you when the workflow exits. If set in your user config file (`~/.nextflow/config`) then you don't need to specify this on the command line for every run.

### `-name`

Name for the pipeline run. If not specified, Nextflow will automatically generate a random mnemonic.

This is used in the MultiQC report (if not default) and in the summary HTML / e-mail (always).

**NB:** Single hyphen (core Nextflow option)

### `-resume`

Specify this when restarting a pipeline. Nextflow will used cached results from any pipeline steps where the inputs are the same, continuing from where it got to previously.

You can also supply a run name to resume a specific run: `-resume [run-name]`. Use the `nextflow log` command to show previous run names.

**NB:** Single hyphen (core Nextflow option)

### `-c`

Specify the path to a specific config file (this is a core NextFlow command).

**NB:** Single hyphen (core Nextflow option)

Note - you can use this to override pipeline defaults.

### `--custom_config_version`

Provide git commit id for custom Institutional configs hosted at `nf-core/configs`. This was implemented for reproducibility purposes. Default is set to `master`.

```bash
## Download and use config file with following git commid id
--custom_config_version d52db660777c4bf36546ddb188ec530c3ada1b96
```

### `--custom_config_base`

If you're running offline, nextflow will not be able to fetch the institutional config files
from the internet. If you don't need them, then this is not a problem. If you do need them,
you should download the files from the repo and tell nextflow where to find them with the
`custom_config_base` option. For example:

```bash
## Download and unzip the config files
cd /path/to/my/configs
wget https://github.com/nf-core/configs/archive/master.zip
unzip master.zip

## Run the pipeline
cd /path/to/my/data
nextflow run /path/to/pipeline/ --custom_config_base /path/to/my/configs/configs-master/
```

> Note that the nf-core/tools helper package has a `download` command to download all required pipeline
> files + singularity containers + institutional configs in one go for you, to make this process easier.

### `--max_memory`

Use to set a top-limit for the default memory requirement for each process.
Should be a string in the format integer-unit. eg. `--max_memory '8.GB'`

### `--max_time`

Use to set a top-limit for the default time requirement for each process.
Should be a string in the format integer-unit. eg. `--max_time '2.h'`

### `--max_cpus`

Use to set a top-limit for the default CPU requirement for each process.
Should be a string in the format integer-unit. eg. `--max_cpus 1`

### `--plaintext_email`

Set to receive plain-text e-mails instead of HTML formatted.

### `--monochrome_logs`

Set to disable colourful command line output and live life in monochrome.

### `--multiqc_config`

Specify a path to a custom MultiQC configuration file.
