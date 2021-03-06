Assembly polishing with nanopolish
==================================

Nanopolish is a software package for signal-level analysis of Oxford Nanopore sequencing data. Nanopolish can calculate an improved consensus sequence for a draft genome assembly, detect base modifications, call SNPs and indels with respect to a reference genome and more. The following modules are available::

  nanopolish extract: extract reads in FASTA or FASTQ format from a directory of FAST5 files
  nanopolish call-methylation: predict genomic bases that may be methylated
  nanopolish variants: detect SNPs and indels with respect to a reference genome
  nanopolish variants --consensus: calculate an improved consensus sequence for a draft genome assembly
  nanopolish eventalign: align signal-level events to k-mers of a reference genome


Mapping of reads to assembly
----------------------------

In order to correct a given assembly, nanopolish needs a mapping of the original reads to this assembly. We are using the software package BWA to do this. BWA is a software package for mapping low-divergent sequences against a large reference genome. It consists of three algorithms: BWA-backtrack, BWA-SW and BWA-MEM. The first algorithm is designed for Illumina sequence reads up to 100bp, while the rest two for longer sequences ranged from 70bp to 1Mbp. BWA-MEM and BWA-SW share similar features such as long-read support and split alignment, but BWA-MEM, which is the latest, is generally recommended for high-quality queries as it is faster and more accurate.

First we need to create an index on our largest contig, this has already been done for the pilon polishing::
  
  #already done for pilon
  bwa index ~/canu_assembly/largestContig.fasta

Then we will run the mapping. Check the usage of ``bwa mem``::

  Usage: bwa mem [options] <idxbase> <in1.fq> [in2.fq]

  Algorithm options:

         -t INT        number of threads [1]
         -k INT        minimum seed length [19]
         -w INT        band width for banded alignment [100]
         -d INT        off-diagonal X-dropoff [100]
         -r FLOAT      look for internal seeds inside a seed longer than {-k} * FLOAT [1.5]
         -y INT        seed occurrence for the 3rd round seeding [20]
         -c INT        skip seeds with more than INT occurrences [500]
         -D FLOAT      drop chains shorter than FLOAT fraction of the longest overlapping chain [0.50]
         -W INT        discard a chain if seeded bases shorter than INT [0]
         -m INT        perform at most INT rounds of mate rescues for each read [50]
         -S            skip mate rescue
         -P            skip pairing; mate rescue performed unless -S also in use
         -e            discard full-length exact matches

  Scoring options:

         -A INT        score for a sequence match, which scales options -TdBOELU unless overridden [1]
         -B INT        penalty for a mismatch [4]
         -O INT[,INT]  gap open penalties for deletions and insertions [6,6]
         -E INT[,INT]  gap extension penalty; a gap of size k cost '{-O} + {-E}*k' [1,1]
         -L INT[,INT]  penalty for 5'- and 3'-end clipping [5,5]
         -U INT        penalty for an unpaired read pair [17]

         -x STR        read type. Setting -x changes multiple parameters unless overridden [null]
                       pacbio: -k17 -W40 -r10 -A1 -B1 -O1 -E1 -L0  (PacBio reads to ref)
                       ont2d: -k14 -W20 -r10 -A1 -B1 -O1 -E1 -L0  (Oxford Nanopore 2D-reads to ref)
                       intractg: -B9 -O16 -L5  (intra-species contigs to ref)

  Input/output options:

         -p            smart pairing (ignoring in2.fq)
         -R STR        read group header line such as '@RG\tID:foo\tSM:bar' [null]
         -H STR/FILE   insert STR to header if it starts with @; or insert lines in FILE [null]
         -j            treat ALT contigs as part of the primary assembly (i.e. ignore <idxbase>.alt file)

         -v INT        verbose level: 1=error, 2=warning, 3=message, 4+=debugging [3]
         -T INT        minimum score to output [30]
         -h INT[,INT]  if there are <INT hits with score >80% of the max score, output all in XA [5,200]
         -a            output all alignments for SE or unpaired PE
         -C            append FASTA/FASTQ comment to SAM output
         -V            output the reference FASTA header in the XR tag
         -Y            use soft clipping for supplementary alignments
         -M            mark shorter split hits as secondary

         -I FLOAT[,FLOAT[,INT[,INT]]]
                     specify the mean, standard deviation (10% of the mean if absent), max
                     (4 sigma from the mean if absent) and min of the insert size distribution.
                     FR orientation only. [inferred]


Note, that there is an option for Oxford Nanopore 2D-reads::

         -x STR        read type. Setting -x changes multiple parameters unless overridden [null]
                       pacbio: -k17 -W40 -r10 -A1 -B1 -O1 -E1 -L0  (PacBio reads to ref)
                       ont2d: -k14 -W20 -r10 -A1 -B1 -O1 -E1 -L0  (Oxford Nanopore 2D-reads to ref)
                       intractg: -B9 -O16 -L5  (intra-species contigs to ref)
                       
But we will adapt the parameters changed by this option a bit, and use the SMALL dataset for nanopolish::

  cd
  mkdir Mapping_1D_basecall_to_assembly
  bwa mem -t 16 -k11 -W17 -r10 -A1 -B1 -O1 -E1 -L0 ~/workdir/canu_assembly/largestContig.fasta ~/workdir/1D_basecall.fastq > ~/workdir/Mapping_1D_basecall_to_assembly/mapping.sam
  
We need to convert the resulting sam file to a sorted and indexed bam file::
  
  -bT ~/Data/Reference/CXERO_10272017.fna?
  samtools view -Sb ~/workdir/Mapping_1D_basecall_to_assembly/mapping.sam > ~/workdir/Mapping_1D_basecall_to_assembly/mapping.bam
  samtools sort -@16 ~/workdir/Mapping_1D_basecall_to_assembly/mapping.bam ~/workdir/Mapping_1D_basecall_to_assembly/mapping.sorted
  samtools index ~/workdir/Mapping_1D_basecall_to_assembly/mapping.sorted.bam
  

Indexing fastq from 1D Basecalling
----------------------------------

In order to prepare our 1D fastq file for nanopolish (so that the tool can find the original raw files), we need to index the fastq files from the 1D basecalling again with nanopolish::

  nanopolish index -d ~/Data/Nanopore/Raw/ ~/Results/1D_basecall.fastq

Call nanopolish
---------------

Now that all pieces are together, we can call nanopolish with:

- our largest contig
- our indexed fastq files from 1D basecalling
- our mapping of 1D fastq vs. our largest contig

Put together::

  nanopolish variants --threads 16 --consensus ~/workdir/polishedContig.fasta -b ~/workdir/Mapping_1D_basecall_to_assembly/mapping.sorted.bam -r ~/Results/1D_basecall.fastq -g ~/workdir/canu_assembly/largestContig.fasta

This will run a few hours over night and we can get to dinner. :)


Assembly evaluation with quast
------------------------------

We are going to evaluate our polished assembly. To call ``quast.py`` we have to provide a reference genome and an assembly as before. We will include our original canu assembly for comparison::
   
  quast.py -t 16 -o ~/workdir/quast_nanopolished_assembly -R ~/Reference/CXERO_10272017.fna ~/workdir/polishedContig.fasta ~/workdir/canu_assembly/largestContig.fasta

QUAST generates HTML reports including a number of interactive graphics. To access these reports, copy the
quast directory to your `www` folder::

  cp -r ~/workdir/quast_nanopolished_assembly ~/www/

You can load the reports in your web browser::

  http://YOUR_OPENSTACK_INSTANCE_IP/quast_nanopolished_assembly/summary/report.html

Compare to the previous results without polishing.

