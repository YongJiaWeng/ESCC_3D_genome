## ATAC-seq / CUT&Tag analysis
need to change directory respectively
```bash
# Adapter trimming and quality filtering with fastp
fastp \
-G -q 5 -u 50 -A -n 15 -l 150 --overlap_diff_limit 1 --overlap_diff_percent_limit 10 \
-i sample_raw_1.fq.gz \
-o sample_clean_1.fastq.gz \
-I sample_raw_2.fq.gz \
-O sample_clean_2.fastq.gz

# Quality control of clean reads
fastqc -t 10 -1 sample_clean_1.fastq.gz -2 sample_clean_2.fastq.gz -o fastqc_out
multiqc fastqc_out

# Alignment to hg19 using bowtie2
bowtie2 -p 8 -q -I 10 -X 1000 --dovetail --no-unal --very-sensitive-local --no-mixed --no-discordant \
-x /lustre/home/qqsong/wyj/reference/index/bowtie2/genome_index/hg19/hg19 \
-1 sample_clean_1.fastq.gz \
-2 sample_clean_2.fastq.gz \
| samtools view -q 30 -F 4 -b - \
| samtools sort -@ 6 -o sample_sorted.bam -

# Filter properly paired and high-quality alignments
samtools view -b -f 2 -q 30 -o sample_f2.q30.bam sample_sorted.bam

# Remove PCR duplicates with picard
java -jar /lustre/home/qqsong/wyj/biosoft/picard/picard.jar MarkDuplicates \
PG=null \
VERBOSITY=ERROR \
QUIET=true \
CREATE_INDEX=false \
REMOVE_DUPLICATES=true \
INPUT=sample_f2.q30.bam \
OUTPUT=sample_f2.q30.dedup.bam \
M=sample.markduplicates.log

samtools index -@ 10 sample_f2.q30.dedup.bam

# Peak calling using MACS2
GSIZE=$(bowtie2-inspect ~/wyj/reference/genome/hg19/hg19 | perl -ne 'BEGIN{$n=0} next if(/^>/);s/[^ATGC]//gi; $n+=length($_); END{print int($n*0.85);}')

macs2 callpeak \
-t sample_f2.q30.dedup.bam \
-f BAMPE \
-n sample \
-g ${GSIZE} \
-B \
-q 0.01 \
--keep-dup all \
--outdir macs2_out

# Generate CPM-normalized bigwig tracks
bamCoverage -bs 10 -p 10 -b sample_f2.q30.dedup.bam -o sample_cpm.bw --normalizeUsing CPM
