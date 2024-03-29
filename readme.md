# About

This is a fast pipeline for admixture mapping. It simplifies the [Tractor analysis](https://github.com/Atkinson-Lab/Tractor/wiki), 
and it can handle admixed population that has multiple ancestries. This pipeline is developed and tested under the linux/unix environment, and it can be easily adapted to high-performance-computing server for massive computation. 

Any questions or complaints are welcomed to post in the issue section or emailed to yingzhou at ds.dfci.harvard.edu .

Updated Date: 1/29/2024

# Requirement

To run this pipeline, you need to know the basic knowledge of linux/unix and some common bioinformatic tools list in the bellow section.

## Computational tools

* Rust, cargo [link](https://www.rust-lang.org/)
* bcftools v1.11 or later [link](https://samtools.github.io/bcftools/)
* htslib v1.15.1 or later [link](https://github.com/samtools/htslib)
* plink 1.9 [link](https://zzz.bwh.harvard.edu/plink/)
* flare [link](https://github.com/browning-lab/flare)
* vcf_anc_formatter [link](src.v1/vcf_anc_formatter)
* tractor_anc_formatter [link](src.v1/tractor_anc_formatter)

## Data

* 1000 genome high coverage WGS data [link](http://ftp.1000genomes.ebi.ac.uk/vol1/ftp/data_collections/1000G_2504_high_coverage/working/20201028_3202_phased/), and meta information [link](https://ftp.1000genomes.ebi.ac.uk/vol1/ftp/data_collections/1000G_2504_high_coverage/20130606_g1k_3202_samples_ped_population.txt)
* Recombination map in plink format [link](https://bochet.gcc.biostat.washington.edu/beagle/genetic_maps/)

# How to use this pipeline

1. Download all folders to your local disk and install all required software.
3. Follow the procedures in the order of folders' names.
4. In each folder, the "record.sh" records function scripts, and the "run.sh" specifies input, output and also job submission commands. Feel free to modify them based on your own understanding.

We included an example data set, so you can just run the example with minimum changing input/output location.

# Folders & Procedures

## 1.target.cohort

* requirement: bcftools

In this folder, we will process the cohort genotype data in vcf format. 
Assume the genotype data is well phased and imputed, we will process the genotype data and only keep binary sites with imputation $R^2$ larger than 0.8 and minimum MAF larger than 0.5%. Duplicated sites are removing by keeping the first occurrence in the vcf record.

If the imputation score $R^2$ is not in the vcf file, you need to remove "-i 'INFO/R2>0.8'" and ",^INFO/R2" from the following command and use other tools to create a filtered genotype data set.

```bash
bcftools norm -m +any ${sourcevcf} \
| bcftools view --type snps -i 'INFO/R2>0.8' --min-alleles 2 \
--max-alleles 2 --min-af 0.005:minor \
| bcftools annotate -x ^FORMAT/GT,^INFO/R2 \
| bcftools norm --rm-dup all -Oz --output ${outvcf}

bcftools query -f '%CHROM:%POS:%REF:%ALT\n' ${outvcf} > ${snplist}
```

We may also use the '-S' option for "bcftools view" to subset samples for further analysis.

In this procedure, we will have high quality genotype data of the target cohort and the list of SNPs we are interested in.


## 2.reference.panel

* requirement: bcftools

In this folder, we will process the reference panel with diverse ancestries, which is the 1000 genome high coverage data set. 
We will only keep the sites that overlap with the cohort genotypes.

```bash
##join biallelic sites into multiallelic records
bcftools norm -m +any ${sourcevcf} \
| bcftools query -f '%CHROM:%POS:%REF:%ALT\n' > ${tmpsnplist}

##find out the shared sites with the targeted cohort
grep -f ${tmpsnplist} ${cohortsnplist} | cut -f1,2 -d":" \
| sed "s/:/\t/g"  > ${snplist}
rm ${tmpsnplist}

##filtration
### 'samples' are the selected samples of ancestry panel
bcftools view --types snps -S ${samples} -R ${snplist} ${sourcevcf} \
| bcftools annotate -x ^FORMAT/GT,INFO --force -Oz --output ${outvcf}
```

**Note:** you need to re-header the X chromosome if you use the data downloaded from this [link](http://ftp.1000genomes.ebi.ac.uk/vol1/ftp/data_collections/1000G_2504_high_coverage/working/20201028_3202_phased/), a [new header file](data/1000g.new.header.txt) is included in this pipeline.

```bash
bcftools reheader -h 1000g.new.header.txt ${old_vcf} \
| bcftools annotate -x INFO,^FORMAT/GT -Oz -o ${new_vcf}
bcftools index -t ${new_vcf}
```

## 3.local.ancestry.inference

* requirement: flare, bcftools

From the first two steps, we have the input genotypes for local ancestry inference, the other required inputs for flare are recombination map and ancestry map for each reference sample. See flare [link](https://github.com/browning-lab/flare) for more information.

REF: Fast, accurate local ancestry inference with FLARE [link](https://www.biorxiv.org/content/10.1101/2022.08.02.502540v1)

```bash
flare=flare.jar
java -jar ${flare} ref=${refvcf} ref-panel=${refpanel} gt=${gt} \
map=${map} out=${out} nthreads=${nthreads} minmac=15 # minmac default value is 50, if sample size is large enough, change it back to 50.
bcftools index -t -f ${out}.anc.vcf.gz

```
After this step, we will have ancestry information for each allele; the output is in compressed vcf format.


**Note:** the chromosome identifier should be the same format between the vcf of genotypes and the recombination map.


## 4.ancestry.extraction 

* requirement: vcf_anc_formatter, htslib

"cd" into the script folder and compile the c program "vcf_anc_formatter" [link](src.v1/vcf_anc_formatter). You need to install htslib first and modify the library link in the "makefile". Then type "make" to compile to get the executable program `vcf_anc_formatter`. Seek help from your IT department if you have any difficulties. 

```bash
## 'ancvcf' is output of flare
## 'anc' is the ancestry code for each ancestry, see the header of flare's output
## 'outpref' is the output prefix, two outputs will be generated: 
#### ${vcfpref}.anc${anc}.vcf.gz is haplotypes from genome tracts of 
#### the specific ancestry, defined by 'anc' variable; 
#### ${vcfpref}.hapanc${anc}.vcf.gz is ancestry copy number (1/0) 
#### of every variant on each haplotype.

extractAncestry=../src.v1/vcf_anc_formatter/vcf_anc_formatter
${extractAncestry} ${ancvcf} ${anc} ${outpref}
```
In this step, we will extract dosage and genotypes from each ancestry for admixture mapping and the Tractor analysis.


## 5.association.and.admixture.mapping

* requirement: plink1.9

With all data ready we can use plink1.9 to conduct association analysis on alleles from each ancestry tracts and admixture mapping. Do feel free to modify the codes based on your own understanding of plink for your own research interest.

```bash
## Tractor analysis
plink --vcf ${vcfpref}.anc${anc}.vcf.gz --vcf-half-call haploid \
--allow-no-sex --pheno ${phenofile} --pheno-name ${pheno} \
--covar ${covarfile} --covar-name  ${covarnames} \
--logistic beta hide-covar --out ${outpref1}.anc${anc} \
--ci 0.95 --freq case-control

paste -d' ' ${outpref1}.anc${anc}.assoc.logistic ${outpref1}.anc${anc}.frq.cc \
| sed "s/ \+/ /g" | sed "s/^ //g" | cut -f1-12,16-20 -d' ' \
| grep -v NA | gzip -c > ${outpref1}.anc${anc}.assoc.simple.gz

mv ${outpref1}.anc${anc}.assoc.simple.gz ${final}/

## admixture mapping
plink --vcf ${vcfpref}.hapanc${anc}.vcf.gz --keep-allele-order --allow-no-sex \
--pheno ${phenofile} --pheno-name ${pheno} --covar ${covarfile} \
--covar-name  ${covarnames} --logistic beta hide-covar \
--out ${outpref2}.anc${anc} --ci 0.95 --freq case-control

paste -d' ' ${outpref2}.anc${anc}.assoc.logistic ${outpref2}.anc${anc}.frq.cc \
| sed "s/ \+/ /g" | sed "s/^ //g" | cut -f1-3,5-12,17-20 -d' ' \
| grep -v NA | gzip -c > ${outpref2}.anc${anc}.admap.simple.gz

mv ${outpref2}.anc${anc}.admap.simple.gz ${final}/

```

**Note:** 
 1. Gender specific or combined analyses on chrX can be done through modifying the input file of phenotypes and covariables.
 2. :warning: Attention :warning: For users in Consortium of African Ancestry Working Group for Coronary Artery Disease, the covariables may include age, sex, PCs or another metric of overall admixture fraction. 

## 6.ancestry.extraction.for.tractor
* requirement: tractor_anc_formatter, Rust, cargo

We need to rebuild a Rust program tractor_anc_formatter [link](src.v1/tractor_anc_formatter) for fast extracting hapcount files and dosage files for the Tractor analysis, the original python version is [here](https://github.com/Atkinson-Lab/Tractor/blob/master/scripts/extract_tracts_flare.py). In the 'src.v1/tractor_anc_formatter' folder, type 'cargo build -r', the excutable tool is located at 'src.v1/tractor_anc_formatter/target/release', then you can use it to format flare's output for tractor analysis:

```
formatter=../src.v1/tractor_anc_formatter/target/release/tractor_anc_formatter
ancvcf=${dir}/${cohorttag}.${chr}.flare-lai.anc.vcf.gz
outpref=${dir}/${cohorttag}.${chr}.flare-lai

${formatter} ${ancvcf} ${outpref}
```

## 7.run_tractor
* requirement: run_tractor_yz.R

The original Rscript for admixture-corrected-association analysis is from [link](https://github.com/Atkinson-Lab/Tractor/blob/master/scripts/run_tractor.R), we post a modified version [here](src.v1/run_tractor_yz.R) with following improvements:

1. Support compressed input (*.gz)
2. Don't require that covars file and genotype (hapcount/dosage) files share exact the same samples and the same order, which saves the data formatting time. 
3. Lower the memmory requirements by loading genotype per SNP.

It could be bit slower but more robust to run.

You can follow [the original description](https://github.com/Atkinson-Lab/Tractor/blob/master/README.md) or our examples to run this script.

```
hapdosepref=${inpdir}/${cohorttag}.${chr}.flare-lai.anc
covar=../data/covars.for.tractor/${cohorttag}.${phe}.csv
out=${outdir}/${cohorttag}.${chr}.${phe}.tractor.out.txt

Rscript ../src.v1/run_tractor_yz.R \
--hapdose ${hapdosepref} \
--phe ${covar} \
--method logistic \
--out ${out}
```


# License

This pipeline is licensed under the Apache License, Version 2.0 (the License). You may obtain a copy of the License from [https://www.apache.org/licenses/LICENSE-2.0](//www.apache.org/licenses/LICENSE-2.0)

# Acknowledgement

[Yingchang (Kevin) Lu](https://medicine.vumc.org/person/yingchang-kevin-lu-md-phd) tested this pipeline and provided practical suggestions. 

[Charles Kooperberg](https://www.fredhutch.org/en/faculty-lab-directory/kooperberg-charles.html) proofreaded this pipeline and provided helpful comments. 
