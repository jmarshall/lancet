lancet
======

Lancet is a somatic variant caller (SNVs and indels) for short read data. Lancet uses a localized micro-assembly strategy to detect somatic mutation with high sensitivity and accuracy on a tumor/normal pair.
Lancet is based on the colored de Bruijn graph assembly paradigm where tumor and normal reads are jointly analyzed within the same graph. On-the-fly repeat composition analysis and self-tuning k-mer strategy are used together to increase specificity in regions characterized by low complexity sequences. Lancet requires the raw reads to be aligned with BWA (See [BWA](http://bio-bwa.sourceforge.net/bwa.shtml) description for more info). Lancet is implemented in C++.

* Version: 1.0.0
* Author: Giuseppe Narzisi, [New York Genome Center](https://www.nygenome.org)

Lancet is freely available for academic and non-commercial research purposes ([`LICENSE.txt`](https://github.com/nygenome/lancet/blob/master/LICENSE.txt)).  

### Downloading and building lancet

Building and running lancet from source requires a GNU-like environment with 

1. GCC
2. GNU Make
3. GNU CMake

It should be possible to build lancet on most Linux installations 
or on a Mac installation with [Xcode and Xcode command line tools] installed.
Lancet is available through github and can be obtained and compiled with following command:

```sh
git clone git://github.com/nygenome/lancet.git
cd lancet
make
```
### Basic usage

A simple lancet command should look something like this:

```
Lancet --tumor T.bam --normal N.bam --ref ref.fa --reg 22:1-51304566 --num-threads 8 > out.vcf
```

The command above detects somatic variants in the tumor/normal pair of bam files (*T.bam* and *N.bam*) for chromosome 22 using 8 threads and saves the variant calls in the out VCF file *out.vcf*.

### Output

Lancet generates in output the list of variants in VCF format (v4.1). All variants (SNVs and indels either shared, specific to the tumor, or specific to the normal) are exported in output. Following VCF conventions, high quality variants are flagged as **PASS** in the FILTER column. For non-PASS variants the FILTER info reports the list of filters that were not satisfied by the variant.

### Filters

The list of filters applied and the thresholds used for filtering are included in the header section. For example:

```
##FILTER=<ID=LowFisherScore,Description="low Fisher's exact test score for tumor-normal allele counts (<10)">
```
The previous filter means that a variant flagged as **LowFisherScore** has not met the minimum Fisher's exact test score threshold for tumor-normal allele counts (default 10).

Below is the current list of filters:

1. **LowCovNormal**: low coverage in the normal
2. **HighCovNormal**: high coverage in the normal
3. **LowCovTumor**: low coverage in the tumor
4. **HighCovTumor**: high coverage in the tumor
5. **LowVafTumor**: low variant allele frequency in the tumor
6. **HighVafNormal**: high variant allele frequency in the normal
7. **LowAltCntTumor**: low alternative allele count in the tumor
8. **HighAltCntNormal**: high alternative allele count in the normal
9. **LowFisherScore**: low Fisher's exact test score for tumor-normal allele counts
10. **StrandBias**: rejects variants where the vast majority of alternate alleles are seen in a single direction
11. **MS**: microsatellite mutation (format: #LEN#MOTIF)

### Visual inspection of the DeBruijn graph

The DeBruijn graph representation of a genomic region can be exported to file in [DOT](http://www.graphviz.org/doc/info/lang.html) format using the -A flag. 

**NOTE:** *The following procedure does not scale to larger graphs. Please render a graph only to inspect a small genomic region of a few hundred basepairs. The -A flag must not be used during regular variant calling over the whole genome.*

For example the following command:

```
Lancet -A --tumor T.bam --normal N.bam --ref ref.fa --reg chr:start-end > out.vcf
```

will export the DeBruijn graph after every stage of the assembly (low covergae removal, tips removal, compression) to the following set of files:

1. chr:start-end.0.dot (initial graph)
2. chr:start-end.1l.cX.dot (after first low coverage nodes removal)
3. chr:start-end.2c.cX.dot (after compression)
4. chr:start-end.3l.cX.dot (after second low coverage nodes removal)
5. chr:start-end.4t.cX.dot (after tips removal)
6. chr:start-end.final.cX.dot (final graph)

Where X is the number of the correspending connected component (in most cases only one). 
These files can be rendered using the utilities available in the [Graphviz](http://www.graphviz.org/) visualization software package. Specifically we reccomand using the **sfdp** utlity which draws undirected graphs using the ``spring'' model and it uses a multi-scale approach to produce layouts of large graphs in a reasonably short time.

```
sfdp -Tpdf file.dot -O
```

Finally we recommend opening the pdf file using the "Preview" image viewer software available in MacOS.

An exemplary graph (before removal of low coverage nodes and tips) for a short region containing a somatic variant would look like this one:

![initial graph](https://github.com/nygenome/lancet/blob/master/doc/img/initial_graph.png)

where the blue nodes are k-mers shared by both tumor and normal; the white nodes are k-mer with low support (e.g., sequencing errors); the red nodes are k-mers only present in the tumor node.

A clean bubble whitin a graph is displayed below:

![initial graph](https://github.com/nygenome/lancet/blob/master/doc/img/clean_bubble.png)

The final graph (after compression) containing one single variant is depicted below. Yellow and orange nodes are the source and sink nodes respectively 

<img src="https://github.com/nygenome/lancet/blob/master/doc/img/final_graph.png" width="400">

### Complete command-line options

```
  |                           |
  |      _` | __ \   __|  _ \ __|
  |     (   | |   | (     __/ |
 _____|\__,_|_|  _|\___|\___|\__|

Program: Lancet (micro-assembly somatic variant caller)
Version: 1.0.0 (beta), September 18 2016
Contact: Giuseppe Narzisi <gnarzisi@nygenome.org>

Usage: Lancet [options] --tumor <BAM file> --normal <BAM file> --ref <FASTA file> --reg <chr:start-end>
 [-h for full list of commands]

Required
   --tumor, -t              <BAM file>    : BAM file of mapped reads for tumor
   --normal, -n             <BAM file>    : BAM file of mapped reads for normal
   --ref, -r                <FASTA file>  : FASTA file of reference genome
   --reg, -p                <string>      : genomic region (in chr:start-end format)
   --bed, -B                <string>      : genomic regions from file (BED format)

Optional
   --min-k, k                <int>         : min kmersize [default: 11]
   --max-k, -K               <int>         : max kmersize [default: 100]
   --trim-lowqual, -q        <int>         : trim bases below qv at 5' and 3' [default: 10]
   --min-base-qual, -C       <int>         : minimum base quality required to consider a base for SNV calling [default: 17]
   --quality-range, -Q       <char>        : quality value range [default: !]
   --min-map-qual, -b        <inr>         : minimum read mapping quality in Phred-scale [default: 15]
   --tip-len, -l             <int>         : max tip length [default: 11]
   --cov-thr, -c             <int>         : coverage threshold [default: 5]
   --cov-ratio, -x           <float>       : minimum coverage ratio [default: 0.01]
   --max-avg-cov, -u         <int>         : maximum average coverage allowed per region [default: 10000]
   --low-cov, -d             <int>         : low coverage threshold [default: 1]
   --window-size, -w         <int>         : window size of the region to assemble (in base-pairs) [default: 600]
   --dfs-limit, -F           <int>         : limit dfs/bfs graph traversal search space [default: 1000000]
   --max-indel-len, -T       <int>         : limit on size of detectable indel [default: 500]
   --max-mismatch, -M        <int>         : max number of mismatches for near-perfect repeats [default: 2]
   --num-threads, -X         <int>         : number of parallel threads [default: 1]
   --node-str-len, -L        <int>         : length of sequence to display at graph node (default: 100)

Filters
   --min-alt-count-tumor, -a  <int>        : minimum alternative count in the tumor [default: 3]
   --max-alt-count-normal, -m <int>        : maximum alternative count in the normal [default: 0]
   --min-vaf-tumor, -e        <float>      : minimum variant allele frequency (AlleleCov/TotCov) in the tumor [default: 0.05]
   --max-vaf-normal, -i       <float>      : maximum variant allele frequency (AlleleCov/TotCov) in the normal [default: 0]
   --min-coverage-tumor, -o   <int>        : minimum coverage in the tumor [default: 4]
   --max-coverage-tumor, -y   <int>        : maximum coverage in the tumor [default: 1000000]
   --min-coverage-normal, -z  <int>        : minimum coverage in the normal [default: 10]
   --max-coverage-normal, -j  <int>        : maximum coverage in the normal [default: 1000000]
   --min-phred-fisher, -s     <float>      : minimum fisher exact test score [default: 5]
   --min-strand-bias, -f      <float>      : minimum strand bias threshold [default: 1]

Short Tandem Repeat parameters
   --max-unit-length, -U      <int>        : maximum unit length of the motif [default: 4]
   --min-report-unit, -N      <int>        : minimum number of units to report [default: 3]
   --min-report-len, -Y       <int>        : minimum length of tandem in base pairs [default: 7]
   --dist-from-str, -D        <int>        : distance (in bp) of variant from STR locus [default: 1]

Flags
   --active-region-off, -W    : turn off active region module
   --kmer-recovery, -R        : turn on k-mer recovery (experimental)
   --print-graph, -A          : print graph (in .dot format) after every stage
   --verbose, -v              : be verbose
   --more-verbose, -V         : be more verbose
```