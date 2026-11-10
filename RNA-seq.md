## RNA-seq analysis
need to change directory respectively
```bash
# Alignment using HISAT2 (v2.2.1, default parameters)
hisat2 -p 8 -x /lustre/home/reference/index/hisat2/hg19/hg19 \
-1 sample_raw_1.fq.gz \
-2 sample_raw_2.fq.gz \
-S sample.sam \
| samtools view -q 30 -F 4 -b - \
| samtools sort -@ 6 -o sample_sorted.bam -

samtools index -@ 10 sample_sorted.bam

# Generate CPM-normalized bigwig tracks
bamCoverage -bs 10 -p 10 -b sample_sorted.bam -o sample_cpm.bw --normalizeUsing CPM
