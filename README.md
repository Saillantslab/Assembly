# Seatrout Genome Assembly
###### by Pavel Dimens - May 2018
## 0.0 Before you start
###0.1 Genome size
Assembly requires knowing (or having an estimation of) the genome size of the species you are trying to assemble. The genome size of sea trout isn't available, so we can try using the Shi Drum (*Umbrina cirrosa*), a relative sciaenid with an avaiable genome size. Genome sizes are often reported as C-values in picograms, which are the weight of the haploid gamete genomes of the species. The conversion looks like this:
genome size (bp) = (0.978 x 109) x DNA content (pg)\\
DNA  content (pg) =  {genome ~size (bp)}/{(0.978 x 109)
1 pg = 978 mb

The *U. cirrosa* C-value is 0.97, which is 0.97pg, so if we plug that into the conversion:
978mb * 0.97pg = 948.66mb

This gives us the haploid genome size, so let's double it for diploid
948.66mb x 2 = 1897.3232mb
1897.32mb = 1.89732 gigabases


The C-value conversion is a bit more nuanced than this, as it depends on the G-C ratio for the species and this method assumes a 50/50 G-C ratio, but ~**1.9gb** is our best estimate so far. As more information becomes avaiable, we can try tweaking this value.

### 0.2 Assemblers
We have two options here, the first is `canu` which uses PacBio (or nanopore) sequences for assembly. It's modular with 3 steps that can be done automatically or manually, and then you can polish your assembly with Illumina reads using `pilon`. The second option is to make a hybrid assembly using `DBG2OLC`, which is a sparsity-based assembler that first makes an Illumina-based assembly, then "overlays" a PacBio assembly onto it.

### 0.3 software required
| Software | Description | URL |
|:---|:---|:---|
| Canu | Modular fork of Celera Assembler specialized in high-noise sequences suitable for PacBio and Nanopore sequence data | https://canu.readthedocs.io/en/latest/ |
| SAMtools | Samtools is a suite of programs for interacting with high-throughput sequencing data | http://www.htslib.org/ |
| Quast | Quality assessment tool for evaluating and comparing genome assemblies | http://quast.sourceforge.net/quast |
| pilon | Uses read alignment analysis to diagnose, report, and automatically improve de novo genome assemblies as well as call variants | http://software.broadinstitute.org/software/pilon/ |
| Trimmomatic | Performs a variety of useful trimming tasks for illumina paired-end and single ended data | http://www.usadellab.org/cms/?page=trimmomatic |
| DBG2OLC | Genome assembler that utilizes long erroneous 3GS sequencing reads and short accurate NGS sequencing reads for hybrid assembly | https://github.com/yechengxi/DBG2OLC |
| SparseAssembler | Exploiting sparseness in de novo genome assembly | https://github.com/yechengxi/SparseAssembler |
| Sparc | A sparsity-based consensus algorithm for long erroneous sequencing reads | https://github.com/yechengxi/Sparc |
| BLASR | Mapping single molecule sequencing reads using Basic Local Alignment with Successive Refinement (BLASR) | https://github.com/PacificBiosciences/blasr |


## 1.0 canu
### 1.1 invoking canu (automatic)
The easiest way to run `canu` is by wrapping it in a bash script with some basic variables (to make tweaking easier later) and running that script.
```sh
#! /usr/bin/env bash

PREFIX=seatrout                 # this is just a filename prefix
DIR=seatrout-PacBio             # this is just an output directory name
GSIZE=1.9G                      # has to end in a capital G, no spaces
DATA=seatrout_pacbio.fasta.gz   # path to your input file

canu \
 -p $PREFIX -d $DIR \
 genomeSize=$GSIZE \
 -pacbio-raw $DATA
```

### 1.2 invoking canu (manually)
Running `canu` without any arguments, like above, will make it automatically go through all three modules using default parameters. If you want to run the three modules manually, you will need to lightly tweak the code.
#### Correction
```bash
canu -correct \
 -p $PREFIX -d $DIR \
 genomeSize=$GSIZE \
 -pacbio-raw $DATA
```
#### Trimming
``` bash
canu -trim \
 -p $PREFIX -d $DIR \
 genomeSize=$GSIZE \
 -pacbio-corrected $DATA        # notice that this is different
```
#### Assembly
``` bash
canu -assemble \
 -p $PREFIX -d $DIR \
 genomeSize=$GSIZE \
 correctedErrorRate=$ERR \        # this is new and requires tweaking based on coverage
 -pacbio-corrected $DATA          # uses corrected data like Trimming does
```
### 1.3 Evaluating the assemblies using `quast`
To optimize your assembly, you will need to use a tool that calculates and outputs the necessary metrics. This can be done rather easily with `quast`, a python-based script that will give you everything you need. After installation, `quast` should be made executable with `chmod +x quast.py` and moved to the `$PATH` (e.g. `/bin` or `/usr/bin` ) for convenience of use, at which point you can call it up using
``` sh
# without the <>
quast.py -o <output.directory> <input.fasta.file>

# or if not in the $PATH
location/of/quast.py -o <output.directory> <input.fasta.file>
```
By default `quast` will use 25% of avaiable memory, and might take a few seconds to several minutes depending on the assembly. After it finishes, you can view the output report it created (`report.txt`) in the directory you gave it with the `-o` argument. It outputs several different formats of the report, but you can ignore most of them.
``` html
user@hostname~/directory/ $ more report.txt

Assembly                    assembly.filename
# contigs (>= 0 bp)         10986             
# contigs (>= 1000 bp)      10986             
# contigs (>= 5000 bp)      6921              
# contigs (>= 10000 bp)     4003              
# contigs (>= 25000 bp)     148               
# contigs (>= 50000 bp)     0                 
Total length (>= 0 bp)      96345898          
Total length (>= 1000 bp)   96345898          
Total length (>= 5000 bp)   85076105          
Total length (>= 10000 bp)  63491866          
Total length (>= 25000 bp)  4156956           
Total length (>= 50000 bp)  0                 
# contigs                   10986             
Largest contig              39776             
Total length                96345898          
GC (%)                      50.52             
N50                         13227             
N75                         8157              
L50                         2667              
L75                         4971              
# N's per 100 kbp           0.00             

```
Since the goal is to assemble sequences into the longest possible single contigs (ideally into entire chromosomes), you will want to maximize the largest contig size and N50 score, while minimizing the number of contigs. In other words, you want fewer contigs that are really long rather than many smaller contigs.These are the metrics to pay attention to as you tweak assembly parameters.

### 1.4 Polishing the assembly with Illumina sequences
`canu` has no features of using Illumina sequences in the assembly, so instead they recommend using a separate script called `pilon`. `Pilon` uses read alignment analysis to improve genome assemblies. It requires an input `.FASTA` file (from `canu`) and with the short-reads it attempts to make improvements in the assembly by way of single base differences, small indels, large indels or block substitutions, gap filling, identification of local misassemblies, and an optionally opening new gaps. The raw Illumina sequences will need to be trimmed using `trimmomatic`⁠ to prepare them as `.BAM` files for use in `pilon`. Once installed, pilon can be invoked with

``` sh
java -Xmx16G -jar <pilon.jar>` # 16G is the amount of memory allocated (16G is recommended)

```

Like before, it would be most convenient to wrap that in a simple BASH script for easy tweaking later

``` sh
#!/usr/bin/env bash

PACBIO=file1          # the draft assembly fasta
ILLUMINA=file2        # the aligned illumina BAM files
OUT=prefix.name
ODIR=folder.name
FIX=all               # what kind of fixes you want to apply
KMER=47               # Kmer size used by internal assembler (default 47)
THREADS=20            # how many CPU cores to use, default is 1 and this argument is experimental

java -Xmx16G -jar pilon.jar --genome $PACBIO \
--bam $ILLUMINA \
--output $OUT --outdir $ODIR \
--fix $FIX --diploid \
--k $KMER \
--threads $THREADS
```
The above example script only includes a few generally useful options, please see the full `pilon` documentation for a detailed list and description of all the possible arguments, as some of them may be useful or necessary to your assembly. After polishing with `pilon`, use `quast` as outlined in section 1.3.

## 2.0 DBG2OLC
### 2.1 Invoking DBG2OLC
There are many arguments necessary to run `DBG2OLC`, so again we will wrap the command in a simple BASH script with some variables to make tweaking easier. Like before, you can run it from whatever folder it was compiled into, or you can move a copy to the $PATH and invoke it from there.
```sh
#!/usr/bin/env bash

KMER=number1
GSIZE=est.genome.size             
NODECOV=node.thres               # not used in this example
EDGECOV=edge.thresh              # not used in this example
THRESH_VALUE=number2
KCOV=number3
OVERLAP=number4
ILLUMINA=aligned.illumina.BAM
PACBIO=pacbio.fasta.file

location/to/DBG2OLC k $KMER \
GS $GSIZE \
AdaptiveTh $THRESH_VALUE \
KmerCovTh $KCOV \
MinOverlap $OVERLAP \
Contigs $ILLUMINA \
f $PACBIO \
RemoveChimera 1
```

### 2.2 DBG2OLC arguments
The code above is an example/template, please refer to the `DBG2OLC` documentation for more arguments that may be necessary for assembly. This assembly method requires more user input and tweaking than `canu`, so it is important to understand what some of these arguments are and how to adjust them.
``` sh
k                   # kmer size from 15-127 (suggested to start at 17)
GS                  # estimated genome size
NodeCovTh           # converge threshold for spurious kmers (0-16)
EdgeCovTh           # coverage threshold for spurious links (0-16)
```
The assembly generates internal `M` values, which are matched k-mers between a contig and a long read. These `M` values are necessary for the three main parameters that may be adjusted to improve assembly quality, which are
| argument | description | 10x-20x PacBio | 50x-100x PacBio |
|:----|:---|:---|:---|
| `AdaptiveTh` | The adaptive k-mer matching threshold. If `M` < `AdaptiveTh` x `Contig_Length`, this contig cannot be used as an anchor to the long read | 0.001~0.01 | 0.01-0.02 |
| `KmerCovTh` | The fixed k-mer matching threshold. If `M` < `KmerCovTh`, this contig cannot be used as an anchor to the long read | 2-5 | 2-10 |
| `MinOverlap` | The minimum overlap score between a pair of long reads. For each pair of long reads, an overlap score is calculated by aligning the compressed reads and score with the matching k-mers | 10-30 | 50-150 |

### 2.3 Call Consensus
After the addition of PacBio data to the assembly, call consensus must be performed, which is done using `blasr` (provided by PacBio) and `sparc`⁠. First, you must concatenate the contigs and raw reads for consensus:

```#!/bin/sh
cat Contigs.txt pb_reads.fasta > ctg_pb.fasta
```

The documentation suggests invoking `ulimit -n unlimited` before running the script. Then, perform the consensus
```sh
sh ./split_and_run_sparc.sh raw.fasta DBG2OLC_Consensus_info.txt ctg_pb.fasta ./consensus_dir 2 >cns_log.txt
```
Metrics for this method can also be obtained using `quast` as outlined in section 1.3.
