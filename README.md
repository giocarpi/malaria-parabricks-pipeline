# Malaria-Parabricks-Pipeline

#### Introduction

Large-scale pathogen genomic data is crucial to characterize local and global transmission patterns and spread of important human infectious diseases. Yet, even as pathogen genome datasets grow, current methods to process raw sequence reads into analysis-ready variants remain slow to scale. Here we introduce a GPU framework to accelerate pathogen genomic variant identification. We optimize and evaluate the performance, sensitivity and precision of the GPU framework across benchmark sets of *Plasmodium falciparum* genomes and sequencing depths relative to the GATK Best Practices pipeline. We demonstrate superior throughput performance of the GPU framework with mean execution time reduced by ***27x*** and ***5x*** reduction in cost , while delivering **99.9**% accuracy and enhancing reproducibility over standard pipeline.

#### Requirements

First to run the Nvidia Clara Parabricks user need to download and install the trial version of the package using following steps.Request Nvidia Clara Parabricks trial access from https://developer.nvidia.com/clara-parabricks to get an installation package for your GPU server. If you are trying the trial version there is a license in the package already so you can skip the the rest of this step. If you are a paid customer, you need the license file to run Parabricks. You can get that by running the following on your GPU serverRequest Clara Parabricks trial access from [Genome Sequencing Analysis: Free 90day Trial | NVIDIA](https://developer.nvidia.com/clara-parabricks) to get an installation package for your GPU server. If you are trying the trial version there is a license in the package already so you can skip the the rest of this step. If you are a paid customer, you need the license file to run Parabricks. 

##### Software, Hardware and Nvidia Driver Requirements

- Nvidia Driver Requirements
  
  - nvidia-driver that supports cuda-9.0 or higher
  
  - nvidia-driver that supports cuda-10.0 or higher if you want to run deepvariant or cnnscorevariants
  
  - nvidia-driver that supports cuda-11.0 or higher if you want to run on ampere GPU

- [nvidia-docker2](https://docs.nvidia.com/datacenter/cloud-native/index.html) or [singularity](https://sylabs.io/guides/3.7/user-guide/index.html) version 3.0 or higher (Parabricks will work on singularity 3.0 and above. It has been tested extensively on Singularity v3.7)

- Python 3

- curl (Most Linux systems will already have this installed)

- tabix

##### Hardware Requirements

- Access to internet

- Any GPU that supports CUDA architecture 60, 61, 70, 75 and has 12GB GPU RAM or more. It has been tested on NVIDIA V100, NVIDIA A100, and NVIDIA T4 GPUs.

- System Requirements
  
  - 2 GPU server should have 100GB CPU RAM, at least 24 CPU threads
  
  - 4 GPU server should have 196GB CPU RAM, at least 32 CPU threads
  
  - 8 GPU server should have 392GB CPU RAM, at least 48 CPU threads

For more details please visit Clara Parabricks Documentation [GETTING STARTED ; Clara Parabricks Pipelines 3.6 documentation](https://docs.nvidia.com/clara/parabricks/v3.6/text/getting_started.html)

###### 

#### Parabricks workflow to perform read mapping and variant calling of malaria *P. falciparum* genomes.

###### ![](C:\Users\pvats\AppData\Roaming\marktext\images\2021-10-01-10-20-06-image.png)

#### Workflow commands to run the malaria *P. falciparum* genome  samples using Nvidia Clara Parabricks pipeline

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
#Parabricks Version "v3.2.0.1"
export SAMPLEDIR=~/${SAMPLE}-pb-${VERSION}
mkdir -p $SAMPLEDIR

#### Alignment
d=$(date)
printf "############\nstart time\n$d\n###########\n"
if [ -s $SAMPLEDIR/$SAMPLE-bwa-options-K-Y.bam ]; then
   echo "Alignment bam file is available"
else
   echo "Alignment started"
   pbrun fq2bam --ref pf_ref/Pfalciparum.genome.fasta --in-fq $FQ1 $FQ2 --bwa-options=-Y --out-bam $SAMPLEDIR/${SAMPLE}-bwa-options-Y.bam

fi

#### BQSR
if [ -s $SAMPLEDIR/$SAMPLE.bqsr.txt ]; then
   echo "BQSR file exist"
else
   echo "Running BQSR"
   pbrun bqsr --in-bam $SAMPLEDIR/${SAMPLE}-bwa-options-Y.bam --out-recal-file $SAMPLEDIR/$SAMPLE.bqsr.txt --ref pf_ref/Pfalciparum.genome.fasta --knownSites pf_ref/3d7_hb3.combined.final.vcf.gz --knownSites pf_ref/7g8_gb4.combined.final.vcf.gz --knownSites pf_ref/hb3_dd2.combined.final.vcf.gz
fi

#### APPLYBQSR
if [ -s $SAMPLEDIR/$SAMPLE.bqsr.bam ];then
   echo "BQSR BAM exist"
else
   echo "Running AppplyBQSR"
   pbrun applybqsr --in-bam $SAMPLEDIR/${SAMPLE}-bwa-options-Y.bam  --out-bam $SAMPLEDIR/$SAMPLE.bqsr.bam --ref pf_ref/Pfalciparum.genome.fasta --in-recal-file $SAMPLEDIR/$SAMPLE.bqsr.txt

fi

#### BQSR on APPLIEDBAM
if [ -s $SAMPLEDIR/$SAMPLE.after.bqsr-bam.txt ]; then
   echo "BQSR file exist"
else
   echo "Running BQSR on APPLIEDBQSRBAM"
   pbrun bqsr --in-bam $SAMPLEDIR/$SAMPLE.bqsr.bam --out-recal-file $SAMPLEDIR/$SAMPLE.after.bqsr-bam.txt --ref pf_ref/Pfalciparum.genome.fasta --knownSites pf_ref/3d7_hb3.combined.final.vcf.gz --knownSites pf_ref/7g8_gb4.combined.final.vcf.gz --knownSites pf_ref/hb3_dd2.combined.final.vcf.gz
fi

#### HAPLOTYPECALLER
if [ -s $SAMPLEDIR/$SAMPLE.afterbqsr.g.vcf.gz ]; then
   echo "GVCF file exist"
else
   echo "Running HAPLOTYPECALLER"
   pbrun haplotypecaller --in-bam  $SAMPLEDIR/$SAMPLE.bqsr.bam --out-variants $SAMPLEDIR/$SAMPLE.afterbqsr.g.vcf.gz --ref pf_ref/Pfalciparum.genome.fasta  --gvcf -G StandardAnnotation -G AS_StandardAnnotation -G StandardHCAnnotation

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
