# Computational Workflow for Hi-C Analysis of Esophageal Squamous Cell Carcinoma

> Notes: This repository contains the computational scripts used for Hi-C data analysis in this study. The pipeline includes Hi-C mapping, compartment / TAD / loop calling, SV detection, CNV estimation and complex SV-associated neo-loop identification. All parameters and file paths reflect those used in our study.

## Installation

Please install HiCPro (https://genomebiology.biomedcentral.com/articles/10.1186/s13059-015-0831-x) as v3.1.0 and build the supporting environment. Conda is recommended for the process.

## HiC-Pro mapping

Run the following code in terminal.

```bash
/lustre/home/biosoft/Hicpro/HiC-Pro-3.1.0/bin/HiC-Pro \
	-c /lustre/home/anaylis/HiC_hg19_data/configure/config-hicpro_hg19.txt \
	-i /lustre/home/data/hic/ESCC/ \
	-o /lustre/home/anaylis/hic/hicpro/ESCC
```

The configuration file used in the process needs to be carefully edited, here is an example of our work:

```
#Please change the variable settings below if necessary

##Paths and Settings  - Do not edit !
TMP_DIR = tmp
LOGS_DIR = logs
BOWTIE2_OUTPUT_DIR = bowtie_results
MAPC_OUTPUT = hic_results
RAW_DIR = HiCPro_data

##SYSTEM AND SCHEDULER - Start Editing Here !!
N_CPU = 64
LOGFILE = hicpro.log

JOB_NAME =  hictest
JOB_MEM = 3TB
JOB_WALLTIME = 1000:00:00
JOB_QUEUE = cpuPartition
JOB_MAIL = 2211110778@stu.pku.edu.cn

##Data
PAIR1_EXT = _raw_1
PAIR2_EXT = _raw_2

##Alignment options
FORMAT = phred33
MIN_MAPQ = 0

BOWTIE2_IDX_PATH = /lustre/home/anaylis/config/HiC_hg19_data/bowtie2_build/hg19
BOWTIE2_GLOBAL_OPTIONS = --very-sensitive -L 30 --score-min L,-0.6,-0.2 --end-to-end --reorder
BOWTIE2_LOCAL_OPTIONS =  --very-sensitive -L 20 --score-min L,-0.6,-0.2 --end-to-end --reorder

##Annotation files
REFERENCE_GENOME = hg19
GENOME_SIZE = /lustre/home/anaylis/config/HiC_hg19_data/bowtie2_build/hg19/hg19_sizes.txt
CAPTURE_TARGET =

##Allele specific analysis
ALLELE_SPECIFIC_SNP =

##Digestion Hi-C
GENOME_FRAGMENT = /lustre/home/anaylis/config/HiC_hg19_data/restriction_sites/hg19_dpnII.bed
LIGATION_SITE = GATCGATC
MIN_FRAG_SIZE =
MAX_FRAG_SIZE =
MIN_INSERT_SIZE =
MAX_INSERT_SIZE =

##Hi-C processing
MIN_CIS_DIST =
GET_ALL_INTERACTION_CLASSES = 1
GET_PROCESS_SAM = 1
RM_SINGLETON = 1
RM_MULTI = 1
RM_DUP = 1

##Contact Maps
BIN_SIZE = 1000000
MATRIX_FORMAT = upper

##Normalization
MAX_ITER = 10
FILTER_LOW_COUNT_PERC = 0.02
FILTER_HIGH_COUNT_PERC = 0
EPS = 0.1
```

## Results visualization

The output of HiC-Pro is a matrix file containing interaction pairs. Using juicer tools, it can be converted into hic file and viewed with juicebox. After installation of juicer (https://github.com/aidenlab/juicer/wiki/Juicer-Tools-Quick-Start), run the following code in the terminal to perform the conversion.

```bash
/lustre/home/biosoft/Hicpro/HiC-Pro-3.1.0/bin/utils/hicpro2juicebox.sh \
-i /lustre/home/anaylis/hic/hicpro/ESCC/Validpairs/*allValidPairs \
-g /lustre/home/anaylis/config/HiC_hg19_data/bowtie2_build/hg19/hg19_sizes.txt \
-j /lustre/home/biosoft/juicer_tools_jar/juicer_tools_1.19.02.jar \
-o /lustre/home/anaylis/hic/hicpro/ESCC/hic
```

## Call compartments

To call compartments, cool files must firstly be converted from hic files. Using the code below to generate cool files with required resolutions. hicexplorer v3.7 should be installed before the process.

```bash
cd /lustre/home/anaylis/hic/hic_files
for sample in *.hic; do
hicConvertFormat --matrices $sample  --outFileName $sample.cool_10k --inputFormat hic --outputFormat cool --resolutions 10000
hicConvertFormat --matrices $sample  --outFileName $sample.cool_25k --inputFormat hic --outputFormat cool --resolutions 25000
hicConvertFormat --matrices $sample  --outFileName $sample.cool_50k --inputFormat hic --outputFormat cool --resolutions 50000
hicConvertFormat --matrices $sample  --outFileName $sample.cool_100k --inputFormat hic --outputFormat cool --resolutions 100000
done
```

After acquiring cool files, we can call compartments with cooltools (v0.5.1).

```bash
cd /lustre/home/anaylis/hic/cool_files
for sample in *.cool; do
cooltools eigs-cis --n-eigs 1 -o /lustre/home/anaylis/hic/call_compartment/ESCC_allsample/25k/$sample.compartment $sample;
done
```

## Call insulation score with cooltools (v0.5.1)

```bash
cd /lustre/home/anaylis/hic/cool_files
for sample in *.cool; do
  base_name=$(basename "$sample" .cool)
  output_file="/lustre/home/anaylis/hic/call_tad/cooltools/Escc/25k/${base_name}.tsv"

  cooltools insulation -o "$output_file" --ignore-diags 1 --window-pixels --threshold 0.1 --bigwig "$sample" 12
done
```

## Call loops

The loops are called in two resolutions, 10 kb and 25 kb.

```bash
# Call loops with Hiccups, 10 kb resolution
cd /lustre/home/anaylis/hic/hic_files
module load cuda-10.0.130-gcc-9.2.0
for sample in *.hic; do
java -jar /lustre/home/biosoft/juicer_tools_jar/juicer_tools_1.22.01.jar hiccups --threads 1 \
     --ignore-sparsity -m 1024 -k KR -r 10000 -f 0.2 -p 4 -i 16 -d 10000 \
     $sample /lustre/home/anaylis/hic/call_loop/hiccup/ESCC_sample_pdo_cellline_loop/10k/${sample%.hic}.hiccups
done

# Call loops with Hiccups, 25 kb resolution
cd /lustre/home/anaylis/hic/hic_files
module load cuda-10.0.130-gcc-9.2.0
for sample in *.hic; do
java -jar /lustre/home/biosoft/juicer_tools_jar/juicer_tools_1.22.01.jar hiccups --threads 1 \
     --ignore-sparsity -m 1024 -k KR -r 25000 -f 0.1 -p 5 -i 15 -d 50000 \
     $sample /lustre/home/anaylis/hic/call_loop/hiccup/ESCC_sample_pdo_cellline_loop/25k/${sample%.hic}.hiccups
done
```
After obtaining loops identified at 10 kb and 25 kb resolutions, we merged and filtered loop results to integrate loop calls from different resolutions and remove redundant signals. 
Scripts for loop merging and filtering were adapted with reference to the pipeline at [https://zenodo.org/records/14043231]

## Predict SVs

Please install EagleC (https://github.com/XiaoTaoWang/EagleC) as v0.1.9 and build the supporting environment. Conda is recommended for the process.

Please install NeoLoopFinder (https://github.com/XiaoTaoWang/NeoLoopFinder) as v0.4.3-r2 and build the supporting environment. Conda is recommended for the process.

```bash
module load anaconda
source activate EagleC

# calculate SV by EagleC predictSV
cd /lustre/home/anaylis/hic/cool_files
ls 5k/*.cool | sed 's/\.allValidPairs\.hic_5000\.cool_5k\.cool//; s/5k\///' > sample.txt

while read -r id; do
    predictSV \
    --hic-5k /lustre/home/anaylis/hic/cool_files/5k/${id}.allValidPairs.hic_5000.cool_5k.cool \
    --hic-10k /lustre/home/anaylis/hic/cool_files/10k/${id}.allValidPairs.hic_10000.cool_10k.cool \
    --hic-50k /lustre/home/anaylis/hic/cool_files/50k/${id}.allValidPairs.hic_50000.cool_50k.cool \
    -O /lustre/home/anaylis/hic/EagleC/output/Output_Format_NeoLoopFinder/${id} \
    -g hg19 \
    --balance-type CNV \
    --output-format NeoLoopFinder \
    --prob-cutoff-5k 0.8 \
    --prob-cutoff-10k 0.8 \
    --prob-cutoff-50k 0.99999
done < sample.txt
```

## Calculate CNV and plot CNV

```bash
cd /lustre/home/anaylis/hic/cool_files
ls *.cool | sed 's/\.allValidPairs\.hic_5000\.cool_5k\.cool//' > sample.txt
while read -r id; do
calculate-cnv -H ${id}.allValidPairs.hic_5000.cool_5k.cool \
-g hg19 \
-e DpnII \
--output /lustre/home/anaylis/hic/neoloopfinder/calculate-cnv/ESCC/5k/${id}_5k.CNV-profile.bedGraph
done < sample.txt

cd /lustre/home/anaylis/hic/neoloopfinder/calculate-cnv/ESCC/5k
ls *.bedGraph | sed 's/_5k\.CNV-profile\.bedGraph//' > sample.txt
while read -r id; do
segment-cnv --cnv-file ${id}_5k.CNV-profile.bedGraph --binsize 5000 \
--ploidy 2 --output /lustre/home/anaylis/hic/neoloopfinder/segment-cnv/ESCC/5k/${id}_5k.CNV-seg.bedGraph --nproc 8
done < sample.txt

cd /lustre/home/anaylis/hic/neoloopfinder/calculate-cnv/ESCC/5k
while read -r id; do
    plot-cnv --cnv-profile "${id}_5k.CNV-profile.bedGraph" \
    --cnv-segment "/lustre/home/anaylis/hic/neoloopfinder/segment-cnv/ESCC/5k/${id}_5k.CNV-seg.bedGraph" \
    --output-figure-name "/lustre/home/anaylis/hic/neoloopfinder/plot-cnv/ESCC/5k/${id}_5k.CNV.genome-wide.png" \
    --dot-size 0.5 \
    --dot-alpha 0.2 \
    --line-width 1 \
    --boundary-width 0.5 \
    --label-size 7 \
    --tick-label-size 6 \
    --clean-mode
done < sample.txt
```

## Assemble complex SVs and call neo-loops

Step 1: CNV correction using `correct-cnv`. Cool files at 5 kb, 10 kb and 25 kb resolutions were processed with `correct-cnv`. The workflow for the 5 kb resolution is shown below as an example; identical procedures were applied to the 10 kb and 25 kb resolutions.

```bash
module load anaconda
source activate EagleC
COOL_DIR="/lustre/home/anaylis/hic/cool_files/5k"
SEG_DIR="/lustre/home/anaylis/hic/neoloopfinder/segment-cnv/ESCC/5k"
cd "${COOL_DIR}"
ls *.cool | sed 's/\.allValidPairs\.hic_5000\.cool_5k\.cool//' > sample.txt
while read -r id; do
    coolfile="${COOL_DIR}/${id}.allValidPairs.hic_5000.cool_5k.cool"
    seg_file="${SEG_DIR}/${id}_5k.CNV-seg.bedGraph"
    correct-cnv \
        -H "${coolfile}" \
        --cnv-file "${seg_file}" \
        --nproc 8 \
        -f
done < sample.txt
```

Step 2: assemble complex SVs.

```bash
module load anaconda
source activate EagleC
COOL_DIR="/lustre/home/anaylis/hic/cool_files"
SV_DIR="/lustre/home/anaylis/hic/EagleC/output/Output_Format_NeoLoopFinder"
out_root="/lustre/home/hic/neoloopfinder/assemble_complexSVs"
while read -r id; do
    sv_file="${SV_DIR}/${id}_SVs.5K_combined.txt"
    f25k="${COOL_DIR}/25k/${id}.allValidPairs.hic_25000.cool_25k.cool"
    f10k="${COOL_DIR}/10k/${id}.allValidPairs.hic_10000.cool_10k.cool"
    f5k="${COOL_DIR}/5k/${id}.allValidPairs.hic_5000.cool_5k.cool"
    out_dir="${out_root}/${id}"
    mkdir -p "$out_dir"
    assemble-complexSVs \
        -O "$out_dir" \
        -B "$sv_file" \
        --balance-type CNV \
        --protocol insitu \
        --nproc 8 \
        -H "$f25k" \
          "$f10k" \
          "$f5k"
done < sample.txt
```

Step 3: call neo-loops

```bash
module load anaconda
source activate EagleC
COOL_DIR="/lustre/home/anaylis/hic/cool_files"
CACHEFOLDER="/lustre/home/data/model/peakachu"
asm_root="/lustre/home/hic/neoloopfinder/assemble_complexSVs"
out_root="/lustre/home/hic/neoloopfinder/call_neoloop"

while read -r id; do
    f25k="${COOL_DIR}/25k/${id}.allValidPairs.hic_25000.cool_25k.cool"
    f10k="${COOL_DIR}/10k/${id}.allValidPairs.hic_10000.cool_10k.cool"
    f5k="${COOL_DIR}/5k/${id}.allValidPairs.hic_5000.cool_5k.cool"
    asm_file="${asm_root}/${id}/${id}.assemblies.txt"
    out_file="${out_root}/${id}.neo-loops.txt"
    mkdir -p "${out_root}"
    neoloop-caller  \
        -O "${out_file}" \
        --cachefolder "${CACHEFOLDER}" \
        --assembly "${asm_file}" \
        --balance-type CNV \
        --protocol insitu \
        --prob 0.95 \
        --nproc 8 \
        -H "${f25k}" \
          "${f10k}" \
          "${f5k}"
done < sample.txt
```
