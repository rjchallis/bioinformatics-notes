---
scope:
  - rjchallis/bioinformatics-notes
tags:
  - blobtoolkit
---

# Intro to BlobToolKit – 2020-08-05

## What is BlobToolKit?

If there is more than one organism in a DNA sample, then there will be more than one genome represented in the read data and any assemblies based on these reads. BlobToolKit is a toolkit to aid identification and separation of sequences from different biological sources (contaminants and cobionts) in a genome assembly.

BlobToolKit provides tools to facilitate exploration of per-contig assembly metrics and to use these to filter an assembly and raw data files to separate data from a mixed assembly into distinct or overlapping bins for further analysis. The contig metrics are either computed directly from the sequence data (e.g. `GC proportion`, `length`, `N count`), directly from analyses run on the assembly (e.g. `BUSCO genes`), or computed from analyses (e.g. `inferred phylum` from BLAST/Diamond hits and `base coverage` from read mapping).

---

More generally, BlobToolKit is a set of command line and browser-based tools built around a generic data model designed to support real-time interaction with large datasets. This introduction focuses on current applications, but the implementation is largely modular to provide flexibility to incorporate new data and methods into the existing architecture.

---

BlobToolKit is made up of several components that can each be configured in various ways ([related slides](https://docs.google.com/presentation/d/1RkEIzgxRTU-mN5E4B5A06unVOq5buzCZZIXpeKABKzE/edit?usp=sharing)):

1. At the centre of it all is the BlobDir format, which is a directory with a collection of JSON format files containing all of the data for an analysed assembly. For each field there is an array of values with one entry per contig in the assembly and these are treated genericly as numbers (such as `GC proportions`), categories (such as `inferred phylum`), or for more complex data arrays of numbers and categories (such as `BLAST hit locations`).

2. BlobTools2 is a command line program that can take assembly files, read mappings, BLAST results and BUSCO scores to generate a BlobDir dataset and has various functions to summarise and filter data.

3. The process of fetching and formatting databases, running analyses and loading them into A BlobDir can be automated using the BTK Pipeline, which is implemented as a Snakemake workflow.

4. For interactive visualisation, there is a browser-based Viewer that can display various views on a BlobDir and lets you view the effects of changing filter parameters in real time. There is a public instance of the Viewer at [blobtoolkit.genomehubs.org/view](https://blobtoolkit.genomehubs.org/view) with currently just over 6,000 analysed assemblies and static versions of the main views are loaded alongside the assemblies in...

5. ...the ENA browser, which links back to BlobToolKit.

## Blob plots

BlobToolKit produces blob plots with GC-content and coverage as primary axes of separation as these are two metrics that we can hope will differ between the different organisms in a sample. If the differences in GC and coverage between 2 taxa are great enough, and the distributions about the mean values are narrow enough then this can present a visual separation of data from different biological sources in an assembly.

![A blob plot](https://blobtoolkit.genomehubs.org/wp-content/uploads/2019/07/ACVV01.blob_.circle.png "a blob plot")

An additional layer of information is added, colouring each contig according to an assessment of its likely taxonomic origin. This is a relatively crude measure as it is based on similarity to sequences in the public sequence databases which don't have complete taxonomic coverage and have many sequences assigned to the wrong taxa because of contamination in previously submitted assemblies.

## BlobToolKit Viewer

BlobToolKit as a whole should make more sense in the context of how the information is displayed in the Viewer. The example above is on the public instance at [ACVV01](https://blobtoolkit.genomehubs.org/view/ACVV01/dataset/ACVV01/blob?plotShape=circle). The Settings and Filter menus provide lots of ways to explore the dataset further, and alternate views can highlight different properties of the dataset.

## BlobTools2

All filter and selection options in the Viewer can also be applied on the command line using BlobTools2. Methods for parsing and filtering each analysis type are implemented in separate Python modules so new analyses can be supported by adding a new module with any methods specific to the associated file formats. There is also a generic text file parser so many analyses could be added without any additional code.

### `blobtools`

```
commands:
    add             add data to a BlobDir
    create          create a new BlobDir
    filter          filter a BlobDir
    host            host interactive view of all BlobDirs in a directory
    view            generate plots using BlobToolKit Viewer
```

### `blobtools create` / `blobtools add`

```
Options:
    --fasta FASTA         FASTA sequence file.
    --hits TSV            Tabular BLAST/Diamond output file.
    --cov BAM             BAM/SAM/CRAM read alignment file.
    --busco TSV           BUSCO full_table.tsv output file.
    --taxid INT           Add ranks to metadata for a taxid.
    --meta YAML           Dataset metadata.
    --taxrule rulename[=prefix]
                          Rule to use when assigning BLAST hits to taxa
                          (bestsum, bestsumorder,
                          bestdistsum, bestdistsumorder).
                          An alternate prefix may be specified.
                          [Default: bestsumorder]
    --evalue FLOAT        Set evalue cutoff when parsing hits file. [Default: 1]
    --bitscore FLOAT      Set bitscore cutoff when parsing hits file. [Default: 1]
    --text TXT            Generic text file.
    --text-cols LIST      Comma separated list of <column number>[=<field name>].
    --trnascan TSV        tRNAscan2-SE output
```

### `blobtools filter`

```
Options:
    --param STRING            String of type param=value.
    --query-string STRING     List of param=value pairs from url query string.
    --json JSON               JSON format list file as generated by BlobtoolKit Viewer.
    --list TXT                Space or newline separated list of identifiers.
    --invert                  Invert filter (exclude matching records).

    --output DIRECTORY        Path to directory to generate a new, filtered BlobDir.
    --fasta FASTA             FASTA format assembly file to be filtered.
    --fastq FASTQ             FASTQ format read file to be filtered (requires --cov).
    --cov BAM                 BAM/SAM/CRAM read alignment file.
    --text TXT                generic text file to be filtered.

    --summary FILENAME        Generate a JSON-format summary of the filtered dataset.
    --summary-rank RANK       Taxonomic level for summary. [Default: phylum]
    --table FILENAME          Tabular output of filtered dataset.
    --table-fields STRING     Comma separated list of field IDs to include in the
                              table output. Use 'plot' to include all plot axes.
                              [Default: plot]
```

## Pipeline

In order to allow BlobToolKit to be run at scale, the Pipeline can automate all of the steps required for the analyses of public (INSDC) assemblies.

![Pipeline DAG](https://blobtoolkit.genomehubs.org/wp-content/uploads/2019/11/Figure_2-1.png "Pipeline DAG")

While this automates file fetching from public repositories, local files can be substituted in provided they follow the appropriate naming convention. Each rule is implemnented in a separate file so it is straightforward to recompose the workflow importing only the required steps or substituting an existing rule with one more suitable to a specific application.

The pipeline requires a YAML format confuguration file to define settings for each assembly it is run on (see [configuring the pipeline](https://blobtoolkit.genomehubs.org/pipeline/pipeline-tutorials/configuring-the-pipeline/)), which at a minimum would be as follows:

```
assembly:
  level: scaffold
  span: 253560284
  prefix: ACVV01
busco:
  lineages:
    - diptera_odb9,
    - arthropoda_odb9
    - eukaryota_odb9
reads:
  paired:
    - [SRR026696, ILLUMINA]
    - [SRR026697, ILLUMINA]
similarity:
  defaults:
    mask_ids:
      - 7215
taxon:
  taxid: 7291
  name: Drosophila albomicans
keep_intermediates: true
```

## More information

More documentation and tutorials are available from [blobtoolkit.genomehubs.org](https://blobtoolkit.genomehubs.org)
