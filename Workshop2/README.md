Workshop 2: Aligning and quantifying reads from RNA-seq
=======================================================

Introduction
------------

Today, we are going to process RNA sequencing data into gene expression
values. We will be working with data from [a published study](http://journals.plos.org/plosone/article/file?type=supplementary&id=info:doi/10.1371/journal.pone.0097550.s002) comparing RNA-seq from healthy lung samples to lung samples from patients with idiopathic pulmonary fibrosis (IPF).

We have provided you with FASTQ files from 8 normal and 8 IPF
samples. For this activity, we will be restricting our analysis to
chromosome 10 to speed up some of the running times.

We will be mapping reads to build hg19 of the human genome. This build
is very similar to GRCh37, which we used last time. The primary difference
is in the naming of the chromosomes: in hg19, the chromosome names include
"chr", whereas in GRCh37 they do not. We chose hg19 because the chromosome
names match those in the gene annotation we are working with and that is
important for the relevant programs to run correctly. (As a side note, a
common source or errors and bugs when working with genomic data is a
mismatch in the chromosome naming schemes between files you are using.) 
There are a few other differences between the genome builds that you can read about
[here](https://wiki.dnanexus.com/Scientific-Notes/human-genome) if you are
interested. 

Goals
-----
At the end of this workshop you should be familiar with the following:
* Gene annotation file formats
* Spliced alignment and gene count generation for RNA-seq data
* Basics of SAM/BAM format
* Visualization of read alignments
* Generation of mapping metrics with Picard
* Preliminary inspection of count data in R

Setup
-----
Follow the setup steps outlined in the first workshop.
```
ssh <sunetID>@corn.stanford.edu
# e.g. ssh zappala@corn.stanford.edu
```

Once connected, run:
```
echo $SHELL
```

If the response is `tcsh`, run the following command:
```
source /afs/ir/class/bios201/setup/setup_tcsh.sh 
```

If the response is `bash`, run the following command:
```
source /afs/ir/class/bios201/setup/setup_bash.sh 
```

Then copy the workshop materials and `cd` into the copied directory:
```
cp -r /afs/ir/class/bios201/workshop2/ .
cd workshop2
```

1 Understanding gene annotation formats
----------------------------------------

Let's first get familiar with a couple of file formats used to specify gene
annotations. Gene annotations are useful because they let us know where
known and predicted genes are located in the genome, which parts are
introns, and which parts are exons. This gives us information about where
we expect RNA-seq reads to map and which genes those reads likely come
from. Additionaly, the position of exon-intron
boundaries assist aligners when mapping reads that span multiple exons.

### GenePred format
This format is closely related to refFlat used by Picard tools. This is a file
format used by UCSC to encode predicted gene positions. Each transcript is
described by a single line. Read about the format
[here](http://genome.ucsc.edu/FAQ/FAQformat.html#format9).

We have retrieved a GenePred file of genes on chromosome 10 from UCSC's
[table browser](https://genome.ucsc.edu/cgi-bin/hgTables) for you.

This is the query we ran:
![screenshot of UCSC table browser](http://web.stanford.edu/class/bios201/table_browser_screenshot.png)

Let's take a look at what this file looks like:
```
less -S annotation/UCSC_table_browser_chr10.txt
# use up/down and side arrows to navigate the file and type q to quit
```

Since we're working with lung samples, let's check out Surfactant Protein
A2 (SFTPA2), a gene highly expressed in lung.
```
grep "SFTPA2" annotation/UCSC_table_browser_chr10.txt | column -t | less -S
```
(`column -t` helps line up columns in a human-readable way)

:question: **1.1 How many different transcripts are there for this gene? How
	   many exons per transcript?**  
:question: **1.2 Can you figure out how these transcripts relate to each
	   other (which exons are shared, which are different)?**

### GTF/GFF format
Gene Transfer Format (GTF) is an extension of the General Feature Format
(GFF). Take a minute to read about these formats
[here](http://genome.ucsc.edu/FAQ/FAQformat.html#format3). GTF files are
used by many RNA-seq aligners and tools for counting the number of reads
mapping to each gene. In contrast to the GenePred format, each "feature"
occupies a line. There will often be a feature for a gene, a separate
feature for each transcript, and features for each exon, among
others. Therefore, the detailed exon information for each transcript is
spread over multiple lines.

We have provided you with the GTF for the version 19 comprehensive gene annotation available
from [Gencode](https://www.gencodegenes.org/releases/19.html), but
subsetted to chromosome 10.

Let's take a look at it:
```
less -S annotation/gencode.v19.annotation_chr10.gtf
```

Again, let's look at SFTPA2.
```
grep "SFTPA2" annotation/gencode.v19.annotation_chr10.gtf | less -S
```
You should be able to identify the same transcripts as in the GenePred format.

:question: **1.3 Do you notice anything different in the order in which the
	   exons are listed compared the GenePred format?**

2 Spliced Alignment and read counting
--------------------------------------

We will use STAR to align our RNA-seq reads. STAR performs best if you can
provide it with a gene annotation (in GTF format) as it uses the
information to identify known splice junctions. However, STAR also infers unannotated
splice junctions based on the data and outputs them. The authors of STAR suggest
that after doing an initial alignment, you provide STAR with the junctions
it has infered and re-run the mapping. This is called 2-pass
mapping.

Before aligning reads with STAR, we need to build the genome index.

```
## Do not run this, we did it for you already to save time.
genomeDir=/afs/ir/users/e/t/etsang/bios201/workshop2/STAR_hg19_chr10
STAR --runThreadN 20 \
     --runMode genomeGenerate \
     --genomeDir $genomeDir \
     --genomeFastaFiles chr10.fa \
     --sjdbGTFfile annotation/gencode.v19.annotation_chr10.gtf \
     --sjdbOverhang 100  # readLength - 1
```

Now we can do the mapping.
It takes a few minutes to run each alignment, so we will only have you map
one of the 16 samples and will provide you with the results for the rest.

The generic mapping command we will be using
```
STAR --runThreadN <NumberOfThreads> \
     --genomeDir </path/to/genomeDir> \
     --readFilesIn </path/to/read1> </path/to/read2> \
     --outFileNamePrefix </path/to/output> \
     --outSAMtype <OutputFormat> \
     --sjdbFileChrStartEnd <JunctionFile> \
     --quantMode <QuantificationType>
```
The first three arguments are required. The rest are optional but we will
use them to provide non-default values. The last two we will only use for
the second-pass alignment.

**NOTE:** There are *many* parameters that you can adjust. Most default
parameters work well for us, but there is a note on the STAR homepage:
"This release was tested with the default parameters for human and mouse
genomes. Please contact the author for a list of recommended parameters
for much larger or much smaller genomes." Keep that in mind if you want to
use STAR for other organisms.

<!-- STAR runs more quickly than some other spliced aligners because it loads
its genome representation into memory. If you are working with a large
genome and need to map multiple samples, you can instruct STAR to load it
once and share that genome between multiple processes. --> 

Run the first-pass alignment for Norm1. This can take a few minutes.
While you're waiting for it to run, you can read about the SAM/BAM format
from the links in the next section.
```
genomeDir=/afs/ir/users/e/t/etsang/bios201/workshop2/STAR_hg19_chr10

STAR --runThreadN 4 \
     --genomeDir $genomeDir \
     --readFilesIn fastq/Norm1_R1.fastq fastq/Norm1_R2.fastq \
     --outFileNamePrefix bam_pass1/Norm1_ \
     --outSAMtype BAM Unsorted
```

Then run the second-pass alignment inputting the junction files for **all
16 samples** (we've provided you with the 15 you didn't generate). 
We'll also get STAR to output the number of reads mapping to each gene.
```
# The first line collects the names of all the junction files from the first pass.
# We can then pass this information to the aligner below.

junctions=`ls bam_pass1/*_SJ.out.tab`

STAR --runThreadN 4 \
     --genomeDir $genomeDir \
     --readFilesIn fastq/Norm1_R1.fastq fastq/Norm1_R2.fastq \
     --outFileNamePrefix bam_pass2/Norm1_ \
     --outSAMtype BAM Unsorted \
     --sjdbFileChrStartEnd $junctions \
     --quantMode GeneCounts
```

Take a look at `bam_pass2/Norm1_Log.final.out`. This is a summary report
automatically generated by STAR. 

:question: **2.1 What percentage of reads map uniquely?**

3 Understanding SAM/BAM format
-------------------------------

SAM (Sequence Alignment/Map) format is used to store alignment
information. BAM is the binary version of the format, which occupies less
disk space but isn't human readable. The samtools suite of tools helps
view/manipulate SAM and BAM files. 

We worked with BAM files last week (they were the output from bwa that we provided
as input to GATK). The BAM files we are looking at this week follow the
same format; we are just taking more time to dig into what that looks like.

Let's take a look at the first line of one of the bam files we just
created with STAR:
```
samtools view bam_pass2/Norm1_Aligned.out.bam | head -n1 
```

A overview of each column is described
[here](http://www.htslib.org/doc/sam.html).

:question: **3.1 Where did this read map? What about its mate?**  
:question: **3.2 What information do we know about this read based on its
	   flag? You may want to use [Picard's Explain Flags
	   tool](https://broadinstitute.github.io/picard/explain-flags.html).**

Let's also take a look at a specific pair of reads:
```
samtools view bam_pass2/Norm1_Aligned.out.bam | \
	 grep "HWI-ST689:184:D0YYWACXX:1:2315:14384:11932_1:N:0:CGATGT"
```

Take a look at the CIGAR strings of these two reads. To understand them,
	   take a look at the CIGAR section of page 5 of the full SAM
	   specification
	   [here](http://samtools.github.io/hts-specs/SAMv1.pdf) and you
	   can also check out [this brief explanation of CIGAR
	   strings](http://genome.sph.umich.edu/wiki/SAM#What_is_a_CIGAR.3F).

:question: **3.3 What do we know about how these reads are mapped?**

4 Visualizing alignments with IGV
----------------------------------

We will now use the Integrative Genomics Viewer (IGV) to look at our
alignments graphically. IGV requires bam files to be coordinate-sorted and
indexed so let's do that first with samtools. Indexing the bam creates a
separate file with a `.bai` suffix that contains information to make it easy
for programs like IGV to quickly retrieve sections of the BAM file when
the user provides genomic coordinates.

```
samtools sort -o bam_pass2/Norm1_Aligned.out.sorted.bam bam_pass2/Norm1_Aligned.out.bam

samtools index bam_pass2/Norm1_Aligned.out.sorted.bam
```

If you're using a Mac, you can skip this next step. If you're running PuTTY on
Windows, you'll need to install [Xming](https://sourceforge.net/projects/xming/files/latest/download), an 
X11 forwarding client. Once you've installed Xming, open
a new PuTTY window and log in to corn as usual, except on the initial PuTTY
window, after typing in username<nolink>@corn.stanford.edu, then go to
Connection -\> SSH -\> X11 and check the box labeled "Enable X11
forwarding". Then press the "Open" button to open an SSH connection with
X11 forwarding, used for showing graphical interfaces over a remote connection.

If you are using a Mac, you need to have
[XQuartz](https://www.xquartz.org/) installed. You may need to restart
your computer after installing XQuartz for it to work.

Then we want to launch IGV. To do this on corn, open **a new terminal window**
and `ssh` again, this time providing -X. Then run through the remaining
[setup
steps](https://github.com/zaczap/bios201/blob/master/setup.md). You can
keep the other terminal window you were working with open because we will
go back to it.
```
ssh <sunet>@corn.stanford.edu -X

# run setup script one of either, depending on your shell:
# source /afs/ir/class/bios201/setup/setup_tcsh.sh
# or 
# source /afs/ir/class/bios201/setup/setup_bash.sh

igv.sh
```
Be patient while IGV lauches in a separate graphical window. This can take
a bit of time. Once the IGV window appears, go to **File** -> **Load from file**.
You should then navigate to and select `bam_pass2/Norm1_Aligned.out.sorted.bam`.

**NOTE**: If you have trouble running IGV through corn, try installing
[IGV](http://software.broadinstitute.org/software/igv/download) on you
laptop. If you run IGV from you laptop, go to **File** -> **Load from
URL**. Enter http://web.stanford.edu/class/bios201/workshop2_bam/Norm1_Aligned.out.sorted.bam.

### Looking at the read pair we inspected in the bam file

When it opens, you won't see anything until you zoom into a specific
locus. Enter "chr10:79,741,049-79,744,741" in the search box.
You should now be able to see the reads mapping to
this region as well as a coverage profile at the top.

To find the read pair we searched for in the bam, right click the area
with the reads. Choose "Select by name" and enter
"HWI-ST689:184:D0YYWACXX:1:2315:14384:11932_1:N:0:CGATGT". You should see
the pair of reads get a colored outline.

:question: **4.1 Did you correctly infer the relative mapping of the read pair before?**

### Viewing a particular gene and the strand of reads

You can also search for a specific gene.
Enter "SFTPA2" in the search box. This is the gene we were
considering before. 

If you right click in the read track, you'll see several options for
coloring and reordering the reads. Color the reads by the strand of the
first read in the pair. Pink reads are mapped to the forward strand and
purple ones to the reverse.

:question: **4.2 What do you notice about the read orientations? How does
	   this compare to the gene orientation?**

### Viewing a sashimi plot

Sashimi plots are a common way of visualizing splicing events.
Search for gene "SMNDC1". Once it loads, right click the read area and
select "Sashimi plot". A new window will appear with the plot.

:question: **4.3 What do the numbers and the curved lines represent?**

5 Generating mapping metrics with Picard
-----------------------------------------

Picard has a suite of tools for manipulating and summarizing BAM
files. One particularly useful for for RNA-seq data is
CollectRnaSeqMetrics and we'll run it here. It uses a refFlat file
(similar to the genePred format above) that we downloaded from
[here](http://hgdownload.cse.ucsc.edu/goldenPath/hg19/database/) and
subsetted to chromosome 10.

In your initial terminal window, run:  
```
java -jar $PICARD CollectRnaSeqMetrics \
     REF_FLAT=annotation/refFlat.chr10.txt \
     INPUT=bam_pass2/Norm1_Aligned.out.sorted.bam \
     OUTPUT=bam_pass2/Norm1_Aligned.out.sorted.metrics.txt \
     STRAND_SPECIFICITY=SECOND_READ_TRANSCRIPTION_STRAND
```
(If you closed that first window when you logged in with -X to use IGV, you'll
need to `cd workshop2` again.)

Let's take a look at the output. The following command will help produce
the output in a format that is easier for us to read:
```
cat bam_pass2/Norm1_Aligned.out.sorted.metrics.txt | \
    head -n 8 | tail -n 2 | sed 's/\t\t/\t.\t/g' | \
    column -t | less -S
```

:question: **5.1 What percentage of reads map to coding bases and UTR bases?**  
:question: **5.2 What would you infer if that percentage was very low, say
	   less than 5%?**

6 Preliminary inspection of count data
---------------------------------------

We will be looking at the count data in R. We encourage you to use RStudio on
your laptop. If you haven't already, you'll need to install 
[R](https://cran.rstudio.com/) and [RStudio Desktop](https://www.rstudio.com/products/rstudio/download/).

First we need to download the files we'll be working with. These are the
counts files that STAR generated for us.

### The rest of this is run on your laptop. Exit corn by typing `exit` or `CTRL-D`.

In the commands below, you need to replace your sunet ID and the path to
your copy of workshop 2.  
**NOTE**: You can first create and/or move to a folder where you want to put
the files.  
**Take note of where you download the files, we will need that information in a couple steps.**

```
# If you are using Mac or Linux, from the terminal, run:
scp <your_sunet>@corn.stanford.edu:<path/to/your/workshop2>/bam_pass2/*_ReadsPerGene.out.tab .
scp <your_sunet>@corn.stanford.edu:<path/to/your/workshop2>/annotation/ensg2hgnc.txt .

# If you are using windows, open a windows command prompt (not PuTTY).
# You can find the command prompt by searching for "cmd" or "command" in your programs.
# Then run:
pscp <your_sunet>@corn.stanford.edu:<path/to/your/workshop2>/bam_pass2/*_ReadsPerGene.out.tab .
pscp <your_sunet>@corn.stanford.edu:<path/to/your/workshop2>/annotation/ensg2hgnc.txt .
```

For example, my commands look like this:
```
pwd  # to get the path to my workshop 2
# /afs/ir/users/e/t/etsang/bios201/workshop2

scp etsang@corn.stanford.edu:/afs/ir/users/e/t/etsang/bios201/workshop2/bam_pass2/*_ReadsPerGene.out.tab .
scp etsang@corn.stanford.edu:/afs/ir/users/e/t/etsang/bios201/workshop2/annotation/ensg2hgnc.txt .
```

The above commands will download a file with gene name mappings and 16
count files: the one you created for Norm1, as well as the ones
for the other samples that we have provided you with.

If you have trouble getting the files through `scp`/`pscp` you can also
download the files from [here](http://web.stanford.edu/class/bios201/workshop2/).

Now start RStudio.

First let's install some packages that you're going to need.
If you get asked whether you want to update existing packages, you can
type "n" for no. Updating packages can take a long time and shouldn't be
necessary here.
```
source("http://bioconductor.org/biocLite.R")
biocLite("DESeq2")
install.packages("pheatmap")
```

If the installation worked correctly, you should be able to load them.
```
library(DESeq2)
library(pheatmap)
```

We need to move to the directory where the downloaded files are located.
(As a side note, RStudio support tab completion and that can help you fill
in the path.)
```
setwd('/path/to/downloaded/files')
# e.g., on mac/linux: setwd('/Users/Emily/Documents/BIOS201/workshop2')
# e.g., on windows: setwd('C:/Users/Emily/Documents/BIOS201/workshop2')
# If you are using windows, make sure you put forward slashes '/' even if your path may be displayed as back slashes!
```

We'll now make a list of our samples and read in the first one:
```
samples = paste0(rep(c('Norm','IPF'), each = 8), rep(c(1:8), 2))

## Read in counts for the first sample and look at the head
data = read.table(paste0(samples[1], '_ReadsPerGene.out.tab'), 
			  header = FALSE, stringsAsFactors = FALSE, 
			  col.names = c('Gene','Unstranded','First','Second'))
head(data)
```
The first column is the gene, the three other columns are different sets
of counts. 'Unstranded' includes reads mapping to either strand, 'First'
only includes read pairs where the first read maps to the strand of the
gene, and 'Second' only includes pairs where the second reads maps to the
gene's strand.

You can see that the first four lines are counts of unmapped, unassigned,
or ambigous reads. You'll also notice that 'First' has the most reads not
assigned to any gene ('N_noFeature'). Together, these indicate that we have
a stranded library where the second read maps to the gene's strand. You
could also know this from talking to the person who prepared the library...
That being the case, we'll drop the other columns.
```
counts = data[-c(1:4), c('Gene','Second')]
colnames(counts)[2] = samples[1]
## Take a look at what it looks like now
head(counts)
```

Now we'll read in the other samples and combine them into the same data
frame.
```
for (sample in samples[2:length(samples)]) {
    sampleData = read.table(paste0(sample,'_ReadsPerGene.out.tab'), 
    header = FALSE, stringsAsFactors = FALSE, skip = 4)[,c(1,4)]
    colnames(sampleData) = c('Gene', sample)
    counts = merge(counts, sampleData, by = 'Gene')
}
```

Let's add in the human-readable gene names and make those the row names.
```
genes = read.table('ensg2hgnc.txt', header = FALSE, 
      stringsAsFactors = FALSE, col.names = c('ENSG', 'HGNC'))
counts = merge(counts, genes, by.x = 'Gene', by.y = 'ENSG')
rownames(counts) = paste(counts$HGNC, counts$Gene, sep = '_')
counts = counts[, samples] # drop gene columns
```

Next week, we actually run differential expression between the two
conditions. Today we'll just look at how the samples cluster by the most
highly expressed genes on chromosome 10. We will use DESeq to generate
normalized expression values.
```
group = sapply(samples, function (s) substr(s, 1, nchar(s)-1))
columnData = as.data.frame(group)
dataset = DESeqDataSetFromMatrix(countData = counts,
                                 colData = columnData,
                                 design = ~ group)
dataset = estimateSizeFactors(dataset)
log2normCounts = log2(counts(dataset, normalized = TRUE) + 1)
```

Finally let's look at the top 30 most expressed genes in each sample and
plot it as a heatmap.
```
top =  order(rowMeans(log2normCounts), decreasing = TRUE)[1:30]
pheatmap(log2normCounts[top, ], show_rownames = TRUE, annotation_col = columnData)
```

:question: **6.1 One sample looks very different from the others. 
	   Which one?**  
:question: **6.2 How does the outlier sample compare to the other samples
	   in terms of SFTPA1 and SFTPA2, two lung-specific genes? What
	   does that suggest?** 


You can find the answers to the questions [here](https://github.com/zaczap/bios201/blob/master/Workshop2/Answers.md).