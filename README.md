### Overview

This repository hosts the unphased [FermiKit][fermikit] variant calls for the
263 public [SGDP][sgdp] samples across 128 diverse populations. The manuscript
describing the data set is under review. Users *shall* abidy by [Fort
Lauderdale principles][policy] before the final publication of the manuscript.

The data can be downloaded through the [release page][release] of this
repository, or via wget:
```
wget https://github.com/lh3/sgdp-fermi/releases/download/v1/sgdp-hs37d5.tgz
```
Unpacking the tar-ball creates the `sgdp-263-hs37d5` directory which consists
of the following files:
```
sgdp-263-hs37d5
|-- 263.bgt.spl         # sample names and sample information (text)
|-- 263.bgt.bcf         # site list (in BCFv2)
|-- 263.bgt.pbf         # BGT genotypes (custom binary)
|-- um75-hs37d5.bed.gz  # 75bp universal mask (regions to be filtered out)
|-- um35-hs37d5.bed.gz  # 35bp universal mask
|-- vep-impact.fmf.gz   # VEP annotations excluding MODIFER impact (gzip'd text)
`-- bgt                 # BGT precompiled binary for x64-linux
```
In particular, sites outside `um75-hs37d5.bed.gz` but absent from `263.bgt.bcf`
should be regarded as homozygous reference. This is important to many popgen
analyses.

The genotypes are stored in the [BGT][bgt] format, which can be exported to VCF
by the `bgt` command-line tool:
```sh
# export all variants to VCF
bgt view -C 263.bgt > 263.all.vcf
# exclude sites overlapping filtered regions
bgt view -CeB um75-hs37d5.bed.gz 263.bgt > 263.conf-only.vcf
# common variants only
bgt view -f'AC/AN>.05' 263.bgt
# VCF for one sample in a region (note the comma following -s)
bgt view -r 11:1,000,000-2,000,000 -s,S_French-1 -f'AC>0' 263.bgt
# VCF for two samples
bgt view -s,S_French-1,S_French-2 -f'AC>0' 263.bgt
# VCF for East Asian males (see 263.bgt.spl for annotations)
bgt view -s'region=="EastAsian"&&gender=="M"' -f'AC>0'
# coding variants
bgt view -d vep-impact.fmf.gz -a'cdsPos>0' 263.bgt
```
Please check out the [BGT README][bgt] for more advanced uses of bgt.

We are also releasing small variants called from human reference genome
[GRCh38][grc], though this call set lacks universal masks and variant
annotations.

### Data Processing

Each sample was independently *de novo* assembled with fermikit-0.8,
mapped with bwa-0.7.12 to reference genome hs37d5 and then sorted:
```sh
fermi.kit/fermi2.pl unitig -t 8 -p utg -s 3g reads.fq.fz > utg.mak
make -f utg.mak  # this takes a couple of wall-clock days
fermi.kit/bwa mem -x intractg hs37d5.fa utg.mag.gz | gzip -1 > utg.sam.gz
fermi.kit/htsbox samsort utg.sam.gz > utg.srt.bam
```
For 100bp reads, fermikit-0.8 should produce very similar results to the
lastest fermikit-0.12.  After mapping, small variants are called from all
samples together and filtered:
```sh
fermi.kit/htsbox pileup -cuf hs37d5.fa *.srt.bam | bgzip > raw.vcf.gz
fermi.kit/k8 fermi.kit/hapdip.js vcfsum -f raw.vcf.gz | bgzip > flt.vcf.gz
```
The `pileup` command line does not apply any statistical modeling. It simply
extracts differences from the reference genome and produces a multi-sample VCF.
The filtering script marks variants with 1) <50% calling rate or 2) <10
supporting reads in the sample with the highest allele depth. Post-filtered
variants are then imported to BGT:
```sh
bgt import -S flt.vcf.gz 263.bgt
```
with sample information added later. Functional annotations were provided
by Ensembl [Variant Effect Predictor][vep], version 80:
```sh
./variant_effect_predictor.pl -i input.vcf -o output.txt --offline --pick \
                              --cache --everything --quiet
```

### Additional Data Produced by FermiKit

Due to the space limit, we are only providing genotypes in highly compressed
BGT files for download here. We also have multi-sample VCF with read depth
information (15GB compressed), FermiKit unitigs with read depth information
(882GB) and the unitig FM-index for 232 with low-level microbiome sequences
(38GB). Please contact us if you need these data.

[release]: https://github.com/lh3/sgdp-fermi/releases
[fermikit]: https://github.com/lh3/fermikit
[bgt]: https://github.com/lh3/bgt
[sgdp]: https://www.simonsfoundation.org/life-sciences/simons-genome-diversity-project-dataset/
[policy]: http://www.genome.gov/Pages/Research/WellcomeReport0303.pdf
[grc]: http://www.ncbi.nlm.nih.gov/projects/genome/assembly/grc/human/
[vep]: http://www.ensembl.org/info/docs/tools/vep/index.html
