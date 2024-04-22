
# KGWASMatrix

**KGWASMatrix** is an optimized workflow designed for producing k-mer count matrices for large GWAS panels. For an in-depth explanation of the parallelization strategy for the k-mer counting pipeline, please consult the `KGWAS.pdf` document included in this repository.

## Installation

To install and set up the KGWASMatrix pipeline, follow these steps:

### Environment Setup

1. **Create and navigate to your workspace directory:**
   ```bash
   mkdir KGWASMatrix
   cd KGWASMatrix
   ```


2. **Clone the github repository:**
   ```bash
   git clone git@github.com:githubcbrc/KGWASMatrix.git .
   ```

3. **Run the installation script:**

   ```bash
   ./install.sh
   ```

If you inspect the installation script, you can see that all is does is build a docker image, and spin up a container for compiling the source code (and preferably running the executables).
   ```bash
   #!/bin/bash
   cd init
   bash ../scripts/build_img.sh #--no-cache
   bash ../scripts/start_cont.sh
   docker exec -it gwascont bash /project/scripts/build_binaries.sh
   ```
After the installation the ``./build`` folder will contain two key executables:``kmer_count`` and ``matrix_merge``, which are all that is needed to run the pipeline. You have the option to run these executables directly from their current location, or you may choose to relocate them to a ``bin`` directory and include this directory in your system's ``$PATH`` for easier access. For execution on High-Performance Computing (HPC) clusters, it is recommended to transform the Docker image into a Singularity image. This conversion allows you to run the pipeline using ``singularity exec``. Alternatively, the executables can be run directly on the cluster, provided all the necessary libraries are installed.

### Data Preparation

The ``kmer_count`` executable operates on accession files and requires FASTQ files to be located in the ``./data`` folder. The current setup primarily supports paired-end sequencing data files, (mainly because this was our use case). For each accession, you should have two files: for example, ``./data/A123_1.fq`` and ``./data/A123_2.fq`` for accession ``A123``. In the future, we plan to expand the utility's flexibility by introducing additional options to accommodate a wider range of sequencing data types.

 To proceed, establish a ``./data`` directory at the root of the project, and move your sequencing data into this directory, maintaining the format specified above. 
```bash
mkdir ./data
mv path_to_your_sequencing_data/* ./data/
```
If the data size is large, either consider using a simbolic link, 
```bash
ln -s /path_to_large_data ./data
```


Or, amending the ``start_container.sh`` script to mount your external data path directly into the ``./data`` directory within the container by adding another volume mapping to the ``docker run`` command. Here is how you can adjust the ``docker run`` command to include a specific data volume:

```bash
id=$(docker run --rm -d --name ${cont} -it -v $projectDir:/project -v $dataDir:/project/data ${img})
```
In this command:

``$dataDir`` represents the path on your host system where your data is stored that you want to be accessible from within the container.
``/project/data`` is the path inside the container where this data will be accessible. This corresponds to the expected location where ``kmer_count`` will look for FASTQ files. This latter option is not practical for HPC scenarios, but in such cases, data transfer is usually unavoidable.




## Usage Instructions
### K-mer Counting
**Command:**
```bash
kmer_count <accession> <number of bins> <output folder>
```
**Example:**
```bash
kmer_count A123 200 ./output
```

The ``kmer_count`` tool requires three parameters to operate: ``<accession>``, ``<number of bins>``, and ``<output folder>``.

1. **accession:** This parameter specifies the name of the accession (e.g., A123). It is used to identify and load the corresponding paired-end sequencing data files, e.g. ``./data/A123_1.fq`` and ``./data/A123_2.fq``.
2. **number of bins:** This indicates how many k-mer bins to create, which are used to shard the k-mer index. This number directly influences the granularity of parallelism during the ``matrix_merge`` phase.
3. **output folder:** This is the directory where the results of the binned k-mer counts will be stored. It should be specified as a path relative to the root of the project or an absolute path.


For example, `kmer_count A123 200 ./output` would load the reads of accession A123 from the `./data` folder, index the k-mer occurence using 200 bins, and write the results into the `./output` folder. This will create 200 files, one accession index per bin:

```bash
./output/A123/1_nr.tsv
./output/A123/2_nr.tsv
...
./output/A123/200_nr.tsv
```

These are produced in parallel by splitting the accession files into `NUM_CHUNKS` chunks, creating a task per chunk, and running a thread pool to execute the tasks. So, ``NUM_CHUNKS`` governs the granularity of the parallelism for this phase, but the level of parallelism is defined by the number of threads in the pool (the current code uses all available threads on a computational node to maximize resource utilisation, but users can tweak that if they so wish). While ``NUM_CHUNKS`` is currently hard-coded to optimize performance through static array usage, future revisions may introduce dynamic configurations to increase flexibility. Access to the files is synchronized using a mutex array.

### Merging K-mer Bins
**Command:**
```bash
matrix_merge <output path> <accessions list> <bin index> <min occurrence threshold>
```
**Example:**
```bash
matrix_merge ./output accessions.txt 35 6
```

Once k-mer counts are done, all is left is to merge all the bins with the same index into a "matrix bin", and for that we use ``matrix_merge``. In a nutshell, `matrix_merge` takes all the files with the same name from different accessions, and merges them into one index.

`matrix_merge` expects the following parameters:

1. **input path:** Specifies the directory where the binned k-mer counts are stored. Typically, this is the output directory from the ``kmer_count`` phase.
2. **accessions path:** The path to a text file listing all accession names, one per line. Example file format can be found in ``accessions.txt`` included in this repository.
3. **file index:** Indicates the specific bin number to merge in this operation.
4. **min occurrence threshold:** Defines the minimum k-mer frequency necessary for a k-mer to be considered present in an accession, for binarizing the frequencies into presence/absence values. This is a bit of a misnomer as the threshold is applied in a dual manner:
K-mers with a frequency below this threshold are excluded (`frequency < minimum threshold`).
K-mers with a frequency exceeding the complement of this threshold relative to the panel size are also excluded (i.e., `frequency > panel size - minimum threshold`).

For example, `matrix_merge ./output accessions.txt 35 6` will look for all folders under `./output` with names in `accessions.txt`, and merge all bins with index 35:  
```
./output/A100/35_nr.tsv
./output/A101/35_nr.tsv
...
./output/A123/35_nr.tsv
```
This creates a binary matrix (using a k-mer minimum occurence of 6 for establishing presence), which is saved under `matrix_6/35_m.tsv`. Concatinating these results in the full k-mer GWAS matrix.






# HPC Job Examples
