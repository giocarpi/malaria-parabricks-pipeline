# Malaria-Parabricks-Pipeline

###### Download the 90 Day Trail license from the link

###### [Genome Sequencing Analysis: Free 90day Trial | NVIDIA](https://www.nvidia.com/en-us/clara/genomics/)

###### Software, Hardware and Nvidia Driver Requirements can be found

###### [GETTING STARTED &mdash; Clara Parabricks Pipelines 3.6 documentation](https://docs.nvidia.com/clara/parabricks/v3.6/text/getting_started.html)

###### 

#### Running samples using Nvidia Clara Parabricks pipelines

```
#!/bin/bash
#################################################################################################################################
## bash run_pbricks_malaria_pipeline.sh $SAMPLE $FQ1 $FQ2 $VERSION
## eg. bash run_pbricks_malaria_pipeline.sh ERS740936 ERS740936-R1.fq ERS740936-R2.fq pb-v3.2.0.1
#################################################################################################################################
set -eu
set -o pipefail

export SAMPLE=$1
export FQ1=$2
export FQ2=$3
export VESRION=$4
#version "v3.2.0.1"
export SAMPLEDIR=~/${SAMPLE}-pb-${VERSION}
mkdir -p $SAMPLEDIR

#### Alignment
d=$(date)
printf "############\nstart time\n$d\n###########\n"
if [ -s $SAMPLEDIR/$SAMPLE-bwa-options-K-Y.bam ]; then
   echo "Alignment bam file is available"
else
   echo "alignment started"
   time pbrun fq2bam --ref pf_ref/Pfalciparum.genome.fasta --in-fq $FQ1 $FQ2 --bwa-options=-Y --out-bam $SAMPLEDIR/${SAMPLE}-bwa-options-Y.bam

fi

#### BQSR
if [ -s $SAMPLEDIR/$SAMPLE.bqsr.txt ]; then
   echo "BQSR file exist"
else
   echo "Running BQSR"
   time pbrun bqsr --in-bam $SAMPLEDIR/${SAMPLE}-bwa-options-Y.bam --out-recal-file $SAMPLEDIR/$SAMPLE.bqsr.txt --ref pf_ref/Pfalciparum.genome.fasta --knownSites pf_ref/3d7_hb3.combined.final.vcf.gz --knownSites pf_ref/7g8_gb4.combined.final.vcf.gz --knownSites pf_ref/hb3_dd2.combined.final.vcf.gz
fi

#### APPLYBQSR
if [ -s $SAMPLEDIR/$SAMPLE.bqsr.bam ];then
   echo "BQSR BAM exist"
else
   echo "Running AppplyBQSR"
   time pbrun applybqsr --in-bam $SAMPLEDIR/${SAMPLE}-bwa-options-Y.bam  --out-bam $SAMPLEDIR/$SAMPLE.bqsr.bam --ref pf_ref/Pfalciparum.genome.fasta --in-recal-file $SAMPLEDIR/$SAMPLE.bqsr.txt

fi

#### BQSR on APPLIEDBAM
if [ -s $SAMPLEDIR/$SAMPLE.after.bqsr-bam.txt ]; then
   echo "BQSR file exist"
else
   echo "Running BQSR on APPLIEDBQSRBAM"
   time pbrun bqsr --in-bam $SAMPLEDIR/$SAMPLE.bqsr.bam --out-recal-file $SAMPLEDIR/$SAMPLE.after.bqsr-bam.txt --ref pf_ref/Pfalciparum.genome.fasta --knownSites pf_ref/3d7_hb3.combined.final.vcf.gz --knownSites pf_ref/7g8_gb4.combined.final.vcf.gz --knownSites pf_ref/hb3_dd2.combined.final.vcf.gz
fi

#### HAPLOTYPECALLER
if [ -s $SAMPLEDIR/$SAMPLE.afterbqsr.g.vcf.gz ]; then
   echo "GVCF file exist"
else
   echo "Running HAPLOTYPECALLER"
   echo "CMD USE: time pbrun haplotypecaller --in-bam  $SAMPLEDIR/$SAMPLE.bqsr.bam --out-variants $SAMPLEDIR/$SAMPLE.afterbqsr.g.vcf.gz --ref pf_ref/Pfalciparum.genome.fasta  --gvcf  -G StandardAnnotation -G AS_StandardAnnotation -G StandardHCAnnotation"
   time pbrun haplotypecaller --in-bam  $SAMPLEDIR/$SAMPLE.bqsr.bam --out-variants $SAMPLEDIR/$SAMPLE.afterbqsr.g.vcf.gz --ref pf_ref/Pfalciparum.genome.fasta  --gvcf -G StandardAnnotation -G AS_StandardAnnotation -G StandardHCAnnotation

fi

##
        bgzip -d $SAMPLEDIR/$SAMPLE.afterbqsr.g.vcf.gz
##
        time pbrun genotypegvcf --ref pf_ref/Pfalciparum.genome.fasta --in-gvcf $SAMPLEDIR/$SAMPLE.afterbqsr.g.vcf --out-vcf $SAMPLEDIR/$SAMPLE.afterbqsr.gvcftovcf.vcf --tmp-dir tmp/
        bgzip -c $SAMPLEDIR/$SAMPLE.afterbqsr.gvcftovcf.vcf >$SAMPLEDIR/$SAMPLE.afterbqsr.gvcftovcf.vcf.gz
        tabix $SAMPLEDIR/$SAMPLE.afterbqsr.gvcftovcf.vcf.gz
e=$(date)

printf "############\nJob strated at:$d \n Job ends at:$e \n############\n"

exit
```
