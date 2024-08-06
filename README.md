# Table of Contents

- [cram2fq](#cram2fq)
- [Usage](#usage)
  - [cram2fq](#cram2fq-1)
  - [fqsum](#fqsum)
- [CRAM→FASTQ conversion process](#cramfastq-conversion-process)
  - [Personalis](#personalis)
  - [Customer](#customer)
- [Common questions:](#common-questions)
  - [Why do you use fqsum instead of md5/sha1sum, etc?](#why-do-you-use-fqsum-instead-of-md5sha1sum-etc)
  - [Why do you have to use Personalis' cram2fq with Personalis CRAM files?](#why-do-you-have-to-use-personalis-cram2fq-with-personalis-cram-files)
    - [Original base quality scores](#original-base-quality-scores)
    - [FASTQ comments](#fastq-comments)
    - [Secondary/non-primary alignments](#secondarynon-primary-alignments)
    - [Automatic calculation of fqsum](#automatic-calculation-of-fqsum)
  - [What is an "SBA" FASTQ file?](#what-is-an-sba-fastq-file)


# cram2fq

Personalis' cram2fq tool is written specifically to generate original FASTQ files from CRAM files delivered by Personalis.

Personalis' cram2fq is optimized for, and only works with CRAM files delivered by Personalis.

[CRAM files delivered by Personalis should _only_ be converted back to FASTQ using the Personalis cram2fq tool.](#why-do-you-have-to-use-personalis-cram2fq-with-personalis-cram-files)

# Download
```
curl -L 'https://github.com/personalis/cram2fq/releases/download/v1.0.39/cram2fq-1.0.39.tar.gz' | tar -xzv
cd cram2fq-1.0.39/bin
```

# Usage
## cram2fq
```
Usage: ./cram2fq [-d] (-f <fqsum_prefix> OR -o <output_fastq_prefix>) [-r reference] [-t threads] <cram> <aux_cram>

Options:

    -d
        dedup, do not write FASTQ records identified as duplicates (BAM_FDUP flag). Off by default, do not use if you are trying to construct original FASTQ files that match our delivered FQSUMs.
    -f
        FQSUM output prefix, if you only wish to generate FQSUM checksums directly from CRAM. If you want both FASTQ and FQSUM, use -o instead.
        If you wish to generate FQSUM checksums from FASTQ, use the 'fqsum' tool instead
    -h
        Display this help section
    -o
        FASTQ output prefix. This will generate both FASTQ files and FQSUMs in the same pass.
    -r
        Reference genome used for decoding CRAM files. Only necessary if your
        host is not connected to the internet and REF_PATH environmental
        variable is not set. In that case, use a local uncompressed copy of
        ftp://ftp-trace.ncbi.nih.gov/1000genomes/ftp/technical/reference/phase2_reference_assembly_sequence/hs37d5.fa.gz
    -t
        threads, number of threads. Set to system threads by default.

Positional Arguments:

    <cram>
        The "chief" CRAM file, which contains all reads used during analysis
    <aux_cram>
        The "aux" CRAM file, which contains all auxiliary reads that were not
        used during analysis, but are necessary for faithful reconsturction of
        the original FASTQ files

    Note that cram2fq will use less memory if the chief cram is specified
    before the aux cram file in your positional arguments

Environmental Variables:

    REF_PATH
        Used internally by htslib to specify the location of reference contigs.

        If not set, htslib will use "https://www.ebi.ac.uk/ena/cram/md5/%s"
        and populate REF_CACHE for future use.

    REF_CACHE
        Used internally by htslib to store local copies of reference contigs.

        If not set, htslib will use $HOME/.cache/hts-ref

    For more information on the REF_PATH/REF_CACHE variables, see the discussion
    in the samtools manual at https://www.htslib.org/doc/samtools.html

```

Example usage:

```
./cram2fq -o /path/to/output/PREFIX /path/to/input/SAMPLE.chief.cram /path/to/input/SAMPLE.aux.cram
```

## fqsum

fqsums are calculated within the `cram2fq` program, so you typically won't need to invoke it independently.

If you desire to calculate an FQSUM directly without using CRAM files, you can do so with the `fqsum` program.

The fqsum program reads in STDIN, and writes the corresponding FQSUM to STDOUT.
```
< input.fastq ./fqsum > output.fqsum
```


# CRAM→FASTQ conversion process

The process for customers to acquire original FASTQ files for analysis is a two-part process.

Personalis generates a set of CRAM files and corresponding FQSUM files, which are delivered to the customer.

The customer generates the original FASTQ files from the CRAM files, and compares the new FQSUMs to the delivered FQSUMs, to ensure integrity.

## Personalis

Personalis' part of the process involves the following:
- We use the fqsum tool to calculate a pair of FQSUM checksums on the analysis FASTQ files, independent of all pipeline analysis.
- The analysis pipeline creates a "chief" CRAM file, which contains all reads that were analyzed.
- The analysis pipeline creates an "auxiliary" CRAM file, which contains all reads which were not analyzed, but are necessary to reconstruct the original FASTQs, such as sequencing duplicates, off-target/unmapped reads, and in the RNA case, ribosomal reads.

Here's an illustration:
```
------------------------------------------------
Original:
               R1.fastq      R2.fastq
------------------------------------------------
                 /  \          /  \
                /    \        /    \
               /      \      /      \
              /        \    /        \
         [fqsum]        \  /       [fqsum]
           /             \/            \
          /    [analysis pipeline]      \
         /           /        \          \
        |           /          \          |
        |          |           |          |
        V          V           V          V
------------------------------------------------
Delivered:
    R1.fqsum     cram      aux.cram   R2.fqsum
------------------------------------------------
```

## Customer

The customer then uses the chief & aux CRAM files as input to our cram2fq tool, which will both generate the original FASTQ files and calculate the FQSUM of these generated FASTQ files.

```
------------------------------------------------
Delivered:
  R1.fqsum      cram     aux.cram     R2.fqsum
------------------------------------------------
                  |           |
                   \         /
                    \       /
                    [cram2fq]
            /------ /        \ ---------\
           /       /          \          \
          /       /            \          \
         V       V              V          V
------------------------------------------------
Customer generated:
  R1.sba.fqsum  R1.sba.fastq  R2.sba.fastq R2.sba.fqsum
------------------------------------------------
```
To verify FASTQ integrity, the customer can then compare the FQSUM files generated by cram2fq to the FQSUM files delivered by Personalis to ensure they are identical the `diff` commmand, like this:

```
------------------------------------------------
Delivered:
  R1.fqsum      cram     aux.cram     R2.fqsum
------------------------------------------------
      ^                                   ^
      |                                   |
      |   Comparison of delivered         |
    <diff>    vs generated FQSUM        <diff>
      |                                   |
      |                                   |
      V                                   V
------------------------------------------------
Customer generated:
 R1.sba.fqsum  R1.sba.fastq R2.sba.fastq R2.sba.fqsum
------------------------------------------------
```

We suggest comparing FQSUMs with the `diff -s` command, like this:

```
$ diff -s /path/to/delivered/R1.fqsum /path/to/generated/R1.sba.fqsum
Files /path/to/delivered/R1.fqsum and /path/to/generated/R1.sba.fqsum are identical
```



# Common questions:

## Why do you use fqsum instead of md5/sha1sum, etc?

fqsum is an algorithm developed by Personalis with the following mathematical properties, assuming `{A}`, `{B}` and `{C}` are different FASTQ records, and `{A,B,C}` is a set of three FASTQ records:

- idempotent:
```fqsum ({A}) = fqsum (fqsum ({A}))```
-  associative
```fqsum({fqsum ({A, B}), C}) = fqsum ({A, fqsum({B, C})})```
-  commutative
```fqsum ({A, B}) = fqsum ({B, A})```

The commutative property is crucial, as it allows us to checksum FASTQ files in an order-invariant manner. By using the fqsum program, one can confirm that two FASTQ files contain an identical set of records, even if the order of those FASTQ records is different, with `O(n)` time complexity and `O(1)` memory complexity.

By allowing us to checksum two FASTQ files in an order-invariant manner, the cram2fq utility is able to improve the compression ratio of output gzip'd FASTQ files by writing records in a roughly "sorted by alignment" ("SBA") order. The "sba.fq" suffix on the output FASTQ files is an indication that the FASTQ records are roughly sorted by alignment. See [What is an "SBA" FASTQ file?](#what-is-an-sba-fastq-file)

If we were to use an algorithm such as md5, we would need to sort the FASTQ files before comparing checksums, which typically balances `O(nlogn)` time complexity with `O(n)` memory complexity.

## Why do you have to use Personalis' cram2fq with Personalis CRAM files?


### Original base quality scores

Personalis CRAM files contain the original, pre-BQSR base quality score encoded in the "XQ" SAM field. The Personalis cram2fq program ignores the "QUAL" field of the CRAM files, which contains BQSR quality scores, and instead uses the XQ SAM field for quality scores.

### FASTQ comments

Personalis CRAM files store the original FASTQ comments (i.e, the second column on the 1st line of every FASTQ record) in the "XC" SAM field. The cram2fq program ensures that the original FASTQ comments are properly placed in the output FASTQ files.

### Secondary/non-primary alignments

Personalis CRAM files typically contain supplementary/non-primary alignments. It is imperative to ignore such alignments when converting from CRAM to FASTQ, otherwise one will see the same record appear multiple times in the output FASTQ file, which is not correct.

### Automatic calculation of fqsum

To save time, the Personalis cram2fq tool will calculate the FQSUM while it is generating FASTQ files. This removes the need to run the fqsum independently of the cram2fq program, although one can still do this if desired.

## What is an "SBA" FASTQ file?

SBA stands for "sorted by alignment", which is roughly the order in which FASTQ records will appear in a ".sba.fq" FASTQ file created by cram2fq. The cram2fq utility writes read pairs in the order in which they are aligned to the reference genome, which in turn reduces the entropy of the FASTQ file, allowing for better compression with algorithms such as DEFLATE, used by gzip.
