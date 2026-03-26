# nf-core/rnaseq Benchmark: samtools markdup vs Picard MarkDuplicates

Full human genome RNA-seq benchmark comparing samtools markdup against Picard MarkDuplicates within the nf-core/rnaseq pipeline.

## Dataset

8 ENCODE samples, 4 human cell lines (GRCh38), 1.55 billion total reads:

| Sample | Cell line | Read pairs | BAM size |
|--------|-----------|-----------|----------|
| GM12878_REP1 | GM12878 (lymphoblastoid) | 83.5M | 9.0 GB |
| GM12878_REP2 | GM12878 (lymphoblastoid) | 83.7M | 9.1 GB |
| H1_REP1 | H1-hESC (stem cell) | 111.7M | 12 GB |
| H1_REP2 | H1-hESC (stem cell) | 93.3M | 9.2 GB |
| K562_REP1 | K562 (leukemia) | 78.1M | 8.7 GB |
| K562_REP2 | K562 (leukemia) | 97.9M | 12 GB |
| MCF7_REP1 | MCF-7 (breast cancer) | 111.4M | 13 GB |
| MCF7_REP2 | MCF-7 (breast cancer) | 115.4M | 12 GB |

Source: [ENCODE](https://www.encodeproject.org/) via nf-core test datasets (full-size).

## Pipeline

- **nf-core/rnaseq** 3.14.0
- **Aligner**: HISAT2
- **Server**: Hetzner CCX33 (8 dedicated vCPU, 32 GB RAM, 1 TB volume)
- **Phase 1**: Full pipeline with default Picard MarkDuplicates (268 tasks, ~41 hours)
- **Phase 2**: samtools markdup benchmark on the same sorted BAMs (72 runs: 8 samples x 3 thread configs x 3 repeats)

## Key results

### Duplicate counts: exact match

samtools markdup and Picard MarkDuplicates produce **identical duplicate counts** on all 8 samples:

| Sample | Picard dup pairs | samtools dup reads | Match |
|--------|-----------------|-------------------|-------|
| GM12878_REP1 | 62,979,540 | 125,959,080 (= pairs x 2) | exact |
| GM12878_REP2 | 63,273,661 | 126,547,322 | exact |
| H1_REP1 | 55,780,981 | 111,561,962 | exact |
| H1_REP2 | 48,681,430 | 97,362,860 | exact |
| K562_REP1 | 66,655,956 | 133,311,912 | exact |
| K562_REP2 | 80,482,338 | 160,964,676 | exact |
| MCF7_REP1 | 40,212,458 | 80,424,916 | exact |
| MCF7_REP2 | 49,079,584 | 98,159,168 | exact |

Note: optical duplicates = 0 on all samples (standard flowcell). On patterned flowcells (e.g. NovaSeq S4), results may diverge.

### Performance: samtools wins at >= 4 threads

| Metric | Picard | samtools t=1 | samtools t=4 | samtools t=8 |
|--------|--------|-------------|-------------|-------------|
| Mean time | 33 min | 87 min (2.7x slower) | 22 min (1.5x faster) | 17 min (1.9x faster) |
| Mean peak RAM | 21.5 GB | 7.2 GB | 7.5 GB | 9.2 GB |
| OOM on 32 GB | 5/8 crashed | 0 | 0 | 0 |

samtools markdup requires 4 passes (sort-n, fixmate, sort, markdup) vs Picard's single pass. At 1 thread, the extra I/O makes it 2.7x slower. With 4+ threads (standard in nf-core configs), all passes parallelize and samtools overtakes Picard.

### Thread scaling

| Scaling | Factor |
|---------|--------|
| t=1 -> t=4 | 4.1x (near-linear) |
| t=4 -> t=8 | 1.2x (I/O bound) |

### Memory: 2-3x less RAM

Picard: 19-25 GB peak RSS (JVM heap).
samtools t=8: 7-13 GB peak RSS.

On 32 GB machines, Picard OOM'd on 5 of 8 samples on first attempt (exit 137, required retry with memory increase). samtools completed all samples without issues.

## Repository structure

```
phase1_picard/
  trace/                    # Nextflow execution trace (268 tasks)
  picard_metrics/           # Picard MarkDuplicates metrics (8 files)
  hisat2_stats/             # samtools flagstat/idxstats/stats for aligned BAMs
  salmon_counts/            # Salmon merged gene/transcript counts
  multiqc/                  # MultiQC aggregated metrics
  pipeline_info/            # Nextflow reports, timelines, DAGs, params
  logs/                     # Nextflow log from Phase 1 run

phase2_samtools/
  timings/                  # Benchmark CSV: 72 runs (sample, tool, threads, run, time, dups, reads, RSS)
  samtools_stats/           # samtools markdup stats for each run (72 files)
  logs/                     # Phase 2 benchmark log

scripts/
  cloud_setup.sh            # Server provisioning (Docker, Nextflow, volume mount)
  cloud_benchmark.sh        # Phase 1: full nf-core/rnaseq pipeline runner
  cloud_phase2.sh           # Phase 2: samtools markdup benchmark
```

## Reproducing

```bash
# Phase 1: Run nf-core/rnaseq with Picard (default)
nextflow run nf-core/rnaseq -r 3.14.0 \
  --input samplesheet.csv \
  --outdir results_hisat2_picard \
  --genome GRCh38 \
  --aligner hisat2 \
  -profile docker

# Phase 2: Benchmark samtools markdup on the sorted BAMs
# See scripts/cloud_phase2.sh for the full benchmark script
```

## Caveats

1. **Single-thread samtools is slower than Picard** (2.7x). Only use with >= 4 threads.
2. **Standard flowcell only** (optical dups = 0). Patterned flowcell (NovaSeq S4) not tested.
3. **HISAT2 aligner only**. STAR not tested (different BAM characteristics).
4. **No downstream comparison** (featureCounts/DESeq2). Identical dup counts make output divergence unlikely but not formally excluded.

## License

GPL-3.0 (see [LICENSE](LICENSE))
