Assembly with canu
==================
Canu is a fork of the Celera Assembler, designed for high-noise single-molecule sequencing (such as the PacBio RS II/Sequel or Oxford Nanopore MinION). Documentation can be found here:
http://canu.readthedocs.io/en/latest/

Canu is a hierarchical assembly pipeline which runs in four steps:
- Detect overlaps in high-noise sequences using MHAP
- Generate corrected sequence consensus
- Trim corrected sequences
- Assemble trimmed corrected sequences

Get a usage message of canu on how to use the assembler::

  canu --help

  usage:   canu [-version] [-citation] \
                [-correct | -trim | -assemble | -trim-assemble] \
                [-s <assembly-specifications-file>] \
                 -p <assembly-prefix> \
                 -d <assembly-directory> \
                 genomeSize=<number>[g|m|k] \
                [other-options] \
                [-pacbio-raw |
                 -pacbio-corrected |
                 -nanopore-raw |
                 -nanopore-corrected] file1 file2 ...

  example: canu -d run1 -p godzilla genomeSize=1g -nanopore-raw reads/*.fasta.gz 


    To restrict canu to only a specific stage, use:
      -correct       - generate corrected reads
      -trim          - generate trimmed reads
      -assemble      - generate an assembly
      -trim-assemble - generate trimmed reads and then assemble them

    The assembly is computed in the -d <assembly-directory>, with output files named
    using the -p <assembly-prefix>.  This directory is created if needed.  It is not
    possible to run multiple assemblies in the same directory.

    The genome size should be your best guess of the haploid genome size of what is being
    assembled.  It is used primarily to estimate coverage in reads, NOT as the desired
    assembly size.  Fractional values are allowed: '4.7m' equals '4700k' equals '4700000'

    Some common options:
      useGrid=string
        - Run under grid control (true), locally (false), or set up for grid control
          but don't submit any jobs (remote)
      rawErrorRate=fraction-error
        - The allowed difference in an overlap between two raw uncorrected reads.  For lower
          quality reads, use a higher number.  The defaults are 0.300 for PacBio reads and
          0.500 for Nanopore reads.
      correctedErrorRate=fraction-error
        - The allowed difference in an overlap between two corrected reads.  Assemblies of
          low coverage or data with biological differences will benefit from a slight increase
          in this.  Defaults are 0.045 for PacBio reads and 0.144 for Nanopore reads.
      gridOptions=string
        - Pass string to the command used to submit jobs to the grid.  Can be used to set
          maximum run time limits.  Should NOT be used to set memory limits; Canu will do
          that for you.
      minReadLength=number
        - Ignore reads shorter than 'number' bases long.  Default: 1000.
      minOverlapLength=number
        - Ignore read-to-read overlaps shorter than 'number' bases long.  Default: 500.
    A full list of options can be printed with '-options'.  All options can be supplied in
    an optional sepc file with the -s option.

    Reads can be either FASTA or FASTQ format, uncompressed, or compressed with gz, bz2 or xz.
    Reads are specified by the technology they were generated with:
      -pacbio-raw         <files>
      -pacbio-corrected   <files>
      -nanopore-raw       <files>
      -nanopore-corrected <files>

We will run the assembly on the small dataset, for which we have computed the basecalling to save time. The assembly for the complete dataset will take about 2 and a half hours. 
We will perform the assembly in two steps:

Error correction with the parameter::

  -correct       - generate corrected reads
  
followed by trimming and assembly with the following parameters::

  -trim-assemble - generate trimmed reads and then assemble them


Generate corrected reads
------------------------

The correction stage selects the best overlaps to use for correction, estimates corrected read lengths, and generates corrected reads::

  canu -correct -d ~/workspace/canu_correct_small -p canuAssembly genomeSize=3m useGrid=false -nanopore-raw ~/workspace/1D_basecall_small.fastq

It is also possible to run multiple correction rounds to eliminate errors. This has been done on a S. cerevisae dataset in the canu publication. We will not do this in this course due to time limitations, but a script to do this, would look like this::

  COUNT=0
   NAME=input.fasta
   for i in `seq 1 10`; do
   canu -correct -p asm -d round$i \
   corOutCoverage=500 corMinCoverage=0 corMhapSensitivity=high \
   genomeSize=12.1m -nanopore-raw $NAME
   NAME="round$i/asm.correctedReads.fasta.gz"
   COUNT=`expr $COUNT + 1`
   done
   canu -p asm -d asm genomeSize=12.1m -nanopore-corrected $NAME utgGraphDeviation=50
  batOptions=”-ca 500 -cp 50”
  done




Generate and assemble trimmed reads
-----------------------------------

The trimming stage identifies unsupported regions in the input and trims or splits reads to their longest supported range. The assembly stage makes a final pass to identify sequencing errors; constructs the best overlap graph (BOG); and outputs contigs, an assembly graph, and summary statistics::

  canu -trim-assemble -d ~/workspace/canu_assembly_small -p canuAssembly genomeSize=3M useGrid=false -nanopore-corrected ~/workspace/canu_correct_small/canuAssembly.correctedReads.fasta.gz -nanopore-corrected ~/workspace/1D2_basecall.fastq
  

After that is done, inspect the results and copy the precomputed assembly with the complete dataset into your working directory::

  cp -r ~/Results/canu_assembly/ ~/workspace/

  
  
  
