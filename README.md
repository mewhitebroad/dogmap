# dogmap
Kidd lab draft pipeline for dog Illumina WGS alignment

**Last updated 3 March 2021**

This is a candidate pipeline and associated file set for consideration by the Dog 10K Project
This approach is not meant to be definitive and is a work in progress.

## Associated files
Associated files for alignment can be found at https://kiddlabshare.med.umich.edu/public-data/UU_Cfam_GSD_1.0-Y/

```
total 1.7G
 87K Feb 26 14:19 canFam4.chromAlias.txt
 144 Feb 26 14:20 chrY.chromAlias.txt
898M Mar  3 13:57 SRZ189891_722g.simp.GSD1.0.vcf.gz
1.6M Mar  3 14:01 SRZ189891_722g.simp.GSD1.0.vcf.gz.tbi
343K Feb 26 14:20 UU_Cfam_GSD_1.0_ROSY.dict
104K Feb 26 14:20 UU_Cfam_GSD_1.0_ROSY.fa.fai
750M Feb 26 14:20 UU_Cfam_GSD_1.0_ROSY.fa.gz
```


The genome reference is based on the German Shepherd assembly from [Wang et. al.](https://www.nature.com/articles/s42003-021-01698-x)
supplemented with Y chromosome sequence from a Labrador retriever [GCF_014441545.1](https://www.ncbi.nlm.nih.gov/assembly/GCF_014441545.1)

Specifically, the UU_Cfam_GSD_1.0 was downloaded from [UCSC](http://hgdownload.soe.ucsc.edu/goldenPath/canFam4/bigZips/).
UCSC naming conventions were utilized.  The following Y chromosome sequences were added:

```
sequenceName accession
chrY_NC_051844.1 NW_024010443.1
chrY_unplaced_NW_024010443.1 NW_024010443.1
chrY_unplaced_NW_024010444.1 NW_024010444.1
```
Chromosome order is as indicated in the genome [.fai file](https://kiddlabshare.med.umich.edu/public-data/UU_Cfam_GSD_1.0-Y/UU_Cfam_GSD_1.0_ROSY.fa.fai)

A set of known variants is required for Base Quality Score Recalibration (BQSR). Mismatches
at the known variant positions are ignored when building the quality recalibration model. For BQSR we utilized
the variant file of 91 million SNV and small indels derived from 772 canines reported in [Plassais et al.](https://www.nature.com/articles/s41467-019-09373-w)

The variant file was converted to UU_Cfam_GSD_1.0 coordinates using the LiftoverVcf from Picard/GATK, resulting in
a total of 71,541,892 SNVs and 16,939,218 indels found in [SRZ189891_722g.simp.GSD1.0.vcf.gz](https://kiddlabshare.med.umich.edu/public-data/UU_Cfam_GSD_1.0-Y/SRZ189891_722g.simp.GSD1.0.vcf.gz).

## Software versions

The following software and versions are used
```
bwa-mem2 version 2.1
gatk version 4.2.0.0
GNU parallel
samtools version >= 1.9
```

## Conceptual overview of pipeline

Pipeline steps are implemented in (process-illumina.py).  This script is designed to run on our
[cluster environment](https://arc-ts.umich.edu/greatlakes/configuration/) which features compute cores with local solid state drives.

All paths are given as examples, and should be modified for your own use.

### Step 1: read alignment using bwa-mem2

```
bwa-mem2 mem -K 100000000  -t NUM_THREADS -Y \
-R READGROUPINFO \
PATH_TO_UU_Cfam_GSD_1.0_ROSY.fa_with_index \
fq1.gz fq2.gz | samtools view -bS - >  /tmp/SAMPLE.bam 
```

This command is inspired by [Reiger et al. Functional equivalence of genome sequencing analysis](https://www.nature.com/articles/s41467-018-06159-4) and by 
discussions with colleagues. The -K option removes non-determinism in the fragment length distributions identified using different numbers of threads. -Y uses
soft clipping for supplementary alignments, which may aid down stream structural variation analyses. 

### Step 2: marking duplicates and sorting

Duplicate marking and sorting is performed in one step using MarkDuplicatesSpark.

```
gatk MarkDuplicatesSpark -I /tmp/SAMPLE.bam  -O /tmp/SAMPLE.sort.md.bam -M /final/SAMPLE.sort.md.metricts.txt \
--tmp-dir /tmp \
--conf 'spark.executor.cores=NUM_THREADS
--conf ''spark.local.dir=/tmp
```

### Step 3: BQSR step 1

BQSR is run in parallel using GNU parallel with 40 seperate jobs for chr1-chr38, chrX, and all other sequences together. The resulting
recalibration data tables are then combined using GatherBQSRReports.

Sample cmd:
```
gatk --java-options "-Xmx4G" BaseRecalibrator \
--tmp-dir /tmp \
-I /tmp/SAMPLE.sort.md.bam \
-R PATH_TO_UU_Cfam_GSD_1.0_ROSY.fa \
--intervals INTERVAL_FILE_TO_PROCESS \
--known-sites PATH_TO_SRZ189891_722g.simp.GSD1.0.vcf.gz \
-O /tmp/bqsr/INTERVAL.recal_data.table
```
Then when completed, the recalibration data is combined

```
gatk --java-options "-Xmx6G" GatherBQSRReports \
--input /tmp/bqsr/INTERVAL-1.recal_data.table \
--input /tmp/bqsr/INTERVAL-2.recal_data.table \
--input /tmp/bqsr/INTERVAL-3.recal_data.table \
...
--output /tmp/bqsrgathered.reports.list
```

### Step 4: BQSR step 2
The base calibration is then performed with ApplyBQSR based on the combined reports. This is done in parallel with
41 separate jobs for chr1-chr38, chrX, all other sequences together, and unmapped reads.
The recalibrated qualities scores are discretized into 4 bins with low quality values unchanged.

Sample cmd:
```
gatk --java-options "-Xmx4G" ApplyBQSR \
--tmp-dir /tmp \
-I /tmp/SAMPLE.sort.md.bam \
-R PATH_TO_UU_Cfam_GSD_1.0_ROSY.fa \
-O /tmp/bqsr/INTERVAL.bqsr.bam \
--intervals INTERVAL_FILE_TO_PROCESS \
--bqsr-recal-file /tmp/bqsrgathered.reports.list \
--preserve-qscores-less-than 6 \
--static-quantized-quals 10 \
--static-quantized-quals 20 \
--static-quantized-quals 30 \
--static-quantized-quals 40 
```
The 41 recalibrated BAMs are then combined using GatherBamFiles

```
gatk --java-options "-Xmx6G" GatherBamFiles \
--CREATE_INDEX true \
-I /tmp/bqsr/INTERVAL-1.bqsr.bam \
-I /tmp/bqsr/INTERVAL-2.bqsr.bam \
-I /tmp/bqsr/INTERVAL-3.bqsr.bam \
...
-O /tmp/SAMPLE.recal.bam
```
### Step 5: Create GVCF file

HaplotypeCaller is then run to create a per-sample GVCF file for subsequent cohort
short variant identification.  This is run in parallel across 39 separate jobs 
for chr1-chr38, chrX, and all other sequences together.  The resulting GVCF files
are then combined using GatherVcfs.

Sample cmd:
```
gatk --java-options "-Xmx4G" HaplotypeCaller \
--tmp-dir /tmp \
-R PATH_TO_UU_Cfam_GSD_1.0_ROSY.fa \
-I /tmp/SAMPLE.recal.bam \
--intervals INTERVAL_FILE_TO_PROCESS \
-O /tmp/GVCF/INTERVAL.g.vcf.gz \
-ERC GVCF 
```

The GVCFs are combined using
```
gatk --java-options "-Xmx6G" GatherVcfs \
-I /tmp/GVCF/INTERVAL-1.g.vcf.gz \
-I /tmp/GVCF/INTERVAL-2.g.vcf.gz \
-I /tmp/GVCF/INTERVAL-3.g.vcf.gz \
...
-O /tmp/SAMPLE.g.vcf.gz
 ```
The final GCF is then copied to a permanent storage location. 


### Step 6: Convert recalibrated BAM to CRAM

A cram file is created and copied to a permanent storage location

```
gatk --java-options "-Xmx6G" PrintReads \
-R PATH_TO_UU_Cfam_GSD_1.0_ROSY.fa \
-I /tmp/SAMPLE.recal.bam \
-O /permanent/storage/dir/SAMPLE.cram
```

## Use of pipeline script

On our cluster, this can be ran automatically as a job with multiple available processing 
cores on a single computer node. Local fast storage on the node is utilized.

Sample cmd:
```
python dogmap/process-illumina.py \
-t 24 \
--sn CH027 \
--lib CH027lib1 \
--fq1 dl/ERR2750983_1.fastq.gz \
--fq2 dl/ERR2750983_2.fastq.gz \
--ref UU_Cfam_GSD_1.0_ROSY.fa \
--refBWA bwa-mem2index/UU_Cfam_GSD_1.0_ROSY.fa \
--tmpdir /tmpssd/CH027tmp \
--finaldir genome-processing/aligned \
--knownsites vcf/SRZ189891_722g.simp.GSD1.0.vcf.gz
```

## Known issues and next steps

* Automate QC stats on final cram
* Calculate read depth at known variant sites and estimate X vs autosomes coverage
* Calculate IBS metric from collection of known samples, estimate sample sample identity
* Define known sites for use in sample variant calling (VQSR)
* Define known copy-number 2 regions for normalization in QuicK-mer2 and fastCN depth-of-coverage based pipeline










