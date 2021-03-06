# SARS-CoV-2 Illumina GeNome Assembly Line (SIGNAL) 

This snakemake pipeline is compatible with the [illumina artic nf pipeline](https://github.com/connor-lab/ncov2019-artic-nf).
It performs the same consensus and variant calling procedure using `ivar`.
In addition it adds screening with Kraken2/LMAT, enhanced contamination removal, and additional breseq mutation detection.
By default the SARS-CoV2 reference genome: MN908947.3 is used throughout the analysis. This can be changed in the data dependencies download script (`script/get_data_dependencies.sh`) and updating the `config.yaml` accordingly.  Similarly, the default sequencing primer and trimming settings can easily be added and adjusted in the `config.yaml`.
See below for full details.

Future enhancements are intended to aid/automate metadata management in accordance with PHA4GE guidelines, and manage upload to GISAID and INDSC compatible BioSamples.


## Setup/Execution

0. Clone the git repository (`--recursive` only needed to run`ncov-tools` postprocessing)
    
        git clone --recursive https://github.com/jaleezyy/covid-19-signal

1. Install `conda` and `snakemake` (version >5) e.g.

        wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
        bash Miniconda3-latest-Linux-x86_64.sh # follow instructions
        source $(conda info --base)/etc/profile.d/conda.sh
        conda create -n signal -c conda-forge -c bioconda -c defaults snakemake=5.11 pandas
        conda activate signal 

There are some issues with `conda` failing to install newer versions of snakemake
so alternatively install `mamba` and use that (snakemake has beta support for it within the workflow)
    
        conda install -c conda-forge mamba
        mamba create -c conda-forge -c bioconda -n signal snakemake conda
        conda activate signal

Additional software dependencies are managed directly by `snakemake` using conda environment files:

  - trim-galore 0.6.5 ([docs](https://www.bioinformatics.babraham.ac.uk/projects/trim_galore/))
  - kraken2 2.0.7-beta ([docs](https://ccb.jhu.edu/software/kraken2/))
  - quast 5.0.2 ([docs](http://quast.sourceforge.net/quast))
  - bwa 0.7.17 ([docs](http://bio-bwa.sourceforge.net/))
  - samtools 1.7/1.9 ([docs](http://www.htslib.org/))
  - bedtools 2.26.0 ([docs](https://bedtools.readthedocs.io/en/latest/))
  - breseq 0.35.0 ([docs](https://barricklab.org/twiki/bin/view/Lab/ToolsBacterialGenomeResequencing))
  - ivar 1.2 ([docs](https://github.com/andersen-lab/ivar))
  - ncov-tools postprocessing scripts require additional dependencies (see [file](conda_envs/ncov-tools.yaml)).

2. Download necessary database files

The pipeline requires:
 
 - Amplicon primer scheme sequences
 - SARS-CoV2 reference fasta
 - SARS-CoV2 reference gbk 
 - SARS-CoV2 reference gff3
 - kraken2 viral database
 - Human GRCh38 BWA indexed reference

        bash scripts/get_data_dependencies.sh -d data -a MN908947.3

3. Configure your `config.yaml` file

Either using the convenience python script (pending) or 
through modifying the `example_config.yaml` to suit your system

4. Specify your samples in CSV format (e.g. `sample_table.csv`)

See the example table `example_sample_table.csv` for an idea of how to organise this table. You can attempt to use `generate_sample_table.sh` to circumvent manual creation of the table.

5. Execute pipeline (optionally explicitly specify `--cores`):

      `snakemake -kp --cores=NCORES --use-conda --conda-prefix=$HOME/.snakemake all`

6. Postprocessing analyses:

      `snakemake -p --use-conda --cores=1 postprocess`

After postprocessing finishes, you'll see the following summary files:

```
  - summary.html                top-level summary, with links to per-sample summaries
  - {sample_name}/sample.html   per-sample summaries, with links for more detailed info
  - {sample_name}/sample.txt    per-sample summaries, in text-file format instead of HTML
  - summary.zip                 zip archive containing all of the above summary files.
```
Note that the pipeline postprocessing ('snakemake postprocess') is separated from
the rest of the pipeline ('snakemake all').  This is because in a multi-sample run,
it's likely that at least one pipeline stage will fail.  The postprocessing script
should handle failed pipeline stages gracefully, by substituting placeholder values
when expected pipeline output files are absent.  However, this confuses snakemake's
dependency tracking, so there seems to be no good alternative to separating piepline
processing and postprocessing into 'all' and 'postprocess' targets.

Related: because pipeline stages can fail, we recommend running 'snakemake all'
with the -k flag ("Go on with independent jobs if a job fails").

Additionally, you can run @jts' [ncov-tools](https://github.com/jts/ncov-tools)
to generate phylogenies and alternative summaries.

    snakemake --use-conda --cores 10 ncov_tools

Signal manages installing the dependencies and will invoke `ncov-tools` if 
it has been cloned as a sub-module and a fasta containing sequences to include in 
the tree has been specified using `phylo_include_seqs:` in the main signal `config.yaml`.

Outputs will be written as specified within the `ncov-tools` folder and documentation.

### Docker (pending)

Alternatively, the pipeline can be deployed using Docker (see `resources/Dockerfile_pipeline` for specification).
To pull from dockerhub:

        docker pull finlaymaguire/signal

Download data dependencies:

        mkdir -p data && docker run -v $PWD/data:/data finlaymaguire/signal:1.0.0 bash scripts/get_data_dependencies.sh -d /data

Add remaining files (e.g. primers) to your config and sample table in the data directory:

        cp config.yaml sample_table.csv $PWD/data && \ 
            docker run -v $PWD/data:/data finlaymaguire/signal:1.0.0 mv data/config.yaml data/sample_table.csv .

Then execute the pipeline:

        docker run -v $PWD/data:/data finlaymaguire/signal:1.0.0 conda run -n snakemake snakemake --use-conda --conda-prefix $HOME/.snakemake --cores 8 -s Snakefile all

## Summaries:

  - Generate summaries of BreSeq among many samples, [see](resources/dev_scripts/summaries/README.md)

## Pipeline details:

For a step-by-step walkthrough of the pipeline, see [pipeline/README.md](PIPELINE.md).

A diagram of the workflow is shown below.

![Workflow Version 6](./resources/Workflow_Version_6.pdf)
