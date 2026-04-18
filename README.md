# TeloScan

[![CI](https://github.com/glbala87/TeloScan/actions/workflows/ci.yml/badge.svg)](https://github.com/glbala87/TeloScan/actions/workflows/ci.yml)
[![Python](https://img.shields.io/badge/python-3.9%20%7C%203.10%20%7C%203.11%20%7C%203.12-blue)](https://www.python.org)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

**TeloScan** is an open-source Python tool for **de novo discovery of telomeric repeat blocks**
directly from **long-read sequencing data (Oxford Nanopore / PacBio)** when the telomeric motif is unknown.

Unlike reference-based approaches, TeloScan operates in an **alignment-free** manner, enabling
robust telomere discovery in non-model organisms and incomplete genome assemblies.

---

## Key Features

- **Alignment-free** telomeric repeat discovery — no reference genome required
- **Multi-format input** — FASTQ, FASTA, BAM, CRAM (plain or gzip-compressed), stdin
- **De novo k-mer detection** with configurable k-mer sizes (single, range, or mixed)
- **Error-tolerant refinement** (fuzzy matching) for noisy ONT reads
- **Strand-aware** detection with proper forward/reverse complement handling
- **Confidence scoring** combining copy count, sequence entropy, and detection mode
- **Quality filtering** — filter reads by average base quality
- **Multi-threaded** processing with configurable thread count and chunk size
- **Rich output formats** — TSV, BED, GFF3, VCF, self-contained HTML report, PNG plots
- **Library API** — use TeloScan programmatically in Python scripts
- **Zero hard dependencies** — core runs on the Python standard library alone

---

## Installation

### From source

```bash
git clone https://github.com/glbala87/TeloScan.git
cd TeloScan
pip install -e .
```

### Optional dependencies

Install only what you need, or install everything at once:

```bash
# Progress bar
pip install -e ".[progress]"

# Visualization (plots + HTML report)
pip install -e ".[plots]"

# BAM/CRAM support
pip install -e ".[bam]"

# Everything
pip install -e ".[all]"

# Development (testing)
pip install -e ".[dev]"
```

### Requirements

- Python >= 3.9
- No required dependencies (all optional):

| Extra | Package | Purpose |
|-------|---------|---------|
| `progress` | tqdm >= 4.60 | Progress bars during processing |
| `plots` | matplotlib >= 3.5 | PNG plots and HTML report charts |
| `bam` | pysam >= 0.19 | BAM/CRAM file support |

---

## Quick Start

```bash
# Basic run — scans k=4-15 by default
teloscan -i reads.fastq.gz

# With error-tolerant refinement (recommended for ONT)
teloscan -i reads.fastq.gz --refine

# Full analysis with all outputs
teloscan -i reads.fastq.gz \
  --refine \
  --out-html report.html \
  --out-gff3 telomeres.gff3 \
  --out-vcf telomeres.vcf \
  --out-plots plots/
```

### Try with sample data

```bash
teloscan -i tests/data/sample.fastq -k 6-7 --refine --out-html report.html
```

---

## Usage

```
teloscan -i <input> [options]
```

### Input options

| Flag | Description |
|------|-------------|
| `-i`, `--input` | Input file: FASTQ, FASTA, BAM, CRAM (.gz supported), or `-` for stdin |
| `--reference` | Reference FASTA for CRAM decoding |
| `--format` | Force input format: `fasta`, `fastq`, `bam`, `cram` (default: auto-detect) |
| `--min-quality` | Minimum average base quality to keep a read |

### Algorithm options

| Flag | Default | Description |
|------|---------|-------------|
| `-k`, `--k` | `4-15` | k-mer sizes: `6`, `4-15`, or `5,6,7` |
| `--min-run-bp` | `150` | Minimum repeat block length in bp |
| `--refine` | off | Enable error-tolerant fuzzy matching pass |
| `--max-mismatch` | `1` | Max mismatches per k-mer copy (fuzzy mode) |
| `--top-motifs-per-k` | `10` | Top motifs per k used as fuzzy seeds |

### Performance options

| Flag | Default | Description |
|------|---------|-------------|
| `-t`, `--threads` | CPU/2 | Number of worker threads |
| `--chunk-size` | `2000` | Reads per processing chunk |
| `--no-progress` | off | Disable progress bar |
| `--verbose` | off | Enable debug logging |

### Output options

| Flag | Default | Description |
|------|---------|-------------|
| `--out-per-read` | `teloscan.per_read.tsv` | Per-read blocks TSV |
| `--out-bed` | `teloscan.blocks.bed` | BED format coordinates |
| `--out-summary` | `teloscan.summary.tsv` | Motif-level summary TSV |
| `--out-gff3` | *(off)* | GFF3 annotation file |
| `--out-vcf` | *(off)* | VCF annotation file |
| `--out-html` | *(off)* | Self-contained HTML report with embedded plots |
| `--out-plots` | *(off)* | Directory for PNG plot files |

---

## Output Files

### per_read.tsv

One row per detected telomeric block:

| Column | Description |
|--------|-------------|
| `read` | Read identifier |
| `start` | Block start position (0-based) |
| `end` | Block end position |
| `k` | k-mer size |
| `canonical` | Canonical (rotation + strand normalized) repeat motif |
| `mode` | Detection mode: `perfect`, `fuzzy`, or `mixed` |
| `strand` | `+` or `-` |
| `copies` | Number of repeat copies |
| `run_bp` | Total block length in base pairs |
| `confidence` | Confidence score (0–1) |
| `entropy` | Shannon entropy of motif base composition |

### summary.tsv

Aggregated statistics per motif with strand breakdown:

| Column | Description |
|--------|-------------|
| `canonical` | Canonical repeat motif |
| `k` | k-mer size |
| `observed_runs` | Number of blocks found |
| `total_bp` | Total base pairs across all blocks |
| `strand_plus_runs` | Blocks on + strand |
| `strand_minus_runs` | Blocks on - strand |

### blocks.bed

Standard BED6 format for genome browsers (IGV, UCSC).

### GFF3 / VCF

Standard annotation formats for integration with genomics pipelines. Include confidence scores, copy counts, and motif information in attributes/INFO fields.

### HTML Report

Self-contained HTML file (open in any browser) with:
- Summary statistics cards
- Block length distribution histogram
- Top motif abundance bar chart
- Per-read block location heatmap
- Strand distribution pie chart
- Sortable motif and block tables

---

## How It Works

TeloScan uses a **two-pass algorithm**:

**Pass 1 — Motif Discovery:**
Scans all reads for in-frame consecutive identical k-mers. Counts canonical motif occurrences across all reads to identify the most abundant telomeric repeat candidates.

**Pass 2 — Block Detection:**
Re-scans reads using the top motifs from Pass 1. In perfect mode, finds exact tandem repeats. With `--refine`, additionally performs fuzzy matching (Hamming distance) to recover blocks with sequencing errors. Outputs are written incrementally for memory efficiency.

**Canonicalization:**
Repeat motifs are normalized by selecting the lexicographically smallest string among all rotations and the reverse complement. This merges equivalent representations (e.g., `TTAGGG`, `GGGTTA`, `CCCTAA` all map to the same canonical form).

**Confidence Scoring:**
Each detected block receives a confidence score (0–1) combining three factors:
- **Copy count** — more tandem copies increase confidence (asymptotic to 1.0)
- **Sequence entropy** — higher base composition complexity scores higher; single-base repeats score 0 by design
- **Detection mode** — perfect matches score higher than fuzzy matches

---

## Python Library API

```python
from teloscan import read_fasta_fastq, parse_k, detect_blocks_for_read

for read_id, seq in read_fasta_fastq("reads.fastq.gz"):
    blocks = detect_blocks_for_read(
        read_id, seq,
        k_values=parse_k("6"),
        min_run_bp=150,
    )
    for b in blocks:
        print(f"{b.read}  {b.canonical}  {b.strand}  {b.run_bp}bp  conf={b.confidence}")
```

### Exported API

| Name | Type | Description |
|------|------|-------------|
| `read_fasta_fastq()` | function | Universal sequence reader (FASTQ/FASTA/BAM/CRAM) |
| `parse_k()` | function | Parse k-mer size specification |
| `detect_blocks_for_read()` | function | Detect repeat blocks in a single read |
| `canonical_repeat_unit()` | function | Canonicalize a repeat motif |
| `RepeatBlock` | dataclass | Result object with all block attributes |

### RepeatBlock fields

| Field | Type | Description |
|-------|------|-------------|
| `read` | str | Read identifier |
| `start` | int | Block start position (0-based) |
| `end` | int | Block end position |
| `k` | int | k-mer size |
| `canonical` | str | Canonical repeat motif |
| `mode` | str | `perfect`, `fuzzy`, or `mixed` |
| `strand` | str | `+` or `-` |
| `copies` | int | Number of repeat copies |
| `run_bp` | int | Total block length in bp |
| `confidence` | float | Confidence score (0–1) |
| `entropy` | float | Shannon entropy of motif base composition |

---

## Examples

### Stdin piping

> **Note:** Stdin input requires buffering all records into memory for the two-pass algorithm.
> For large datasets, prefer passing a file path instead of piping through stdin.

```bash
cat reads.fastq | teloscan -i -
cat sequences.fasta | teloscan -i - --format fasta
```

### Specific k-mer sizes

```bash
# Human telomere (TTAGGG, k=6)
teloscan -i reads.fastq.gz -k 6 --refine

# Arabidopsis (TTTAGGG, k=7)
teloscan -i reads.fastq.gz -k 7 --refine

# Bombyx mori (TTAGG, k=5)
teloscan -i reads.fastq.gz -k 5 --refine

# Scan a range
teloscan -i reads.fastq.gz -k 4-15 --refine

# Mixed specification
teloscan -i reads.fastq.gz -k 5,6,7 --refine
```

### BAM input

```bash
teloscan -i aligned.bam -k 6 --refine
teloscan -i aligned.cram --reference ref.fa -k 6
```

### Quality filtering

```bash
teloscan -i reads.fastq.gz --min-quality 10 --refine
```

### Multi-threaded processing

```bash
# Use 8 threads with larger chunks for high-throughput datasets
teloscan -i reads.fastq.gz --refine -t 8 --chunk-size 5000
```

---

## Performance Considerations

- **Memory:** When reading from a file path, TeloScan streams data in chunks and writes outputs incrementally — memory usage stays low regardless of input size. Stdin mode (`-i -`) buffers all records into memory; avoid this for datasets exceeding ~1M reads.
- **Threading:** Default thread count is half the available CPUs. For I/O-bound workloads (e.g., gzip-compressed input), increasing `--threads` beyond CPU count may help. For CPU-bound workloads (large k ranges with `--refine`), match `--threads` to physical cores.
- **k-mer range:** Wider k ranges (e.g., `4-15`) scan more thoroughly but take proportionally longer. If you know the expected motif length, narrow the range (e.g., `-k 6` for human telomeres).
- **Chunk size:** The default (`--chunk-size 2000`) works well for most datasets. Larger values reduce multiprocessing overhead but increase per-chunk memory usage.

---

## Project Structure

```
TeloScan/
├── src/teloscan/
│   ├── __init__.py       # Package exports and version
│   ├── cli.py            # Command-line interface and output writers
│   ├── engine.py         # Core detection algorithms and multiprocessing
│   ├── io.py             # Multi-format sequence reader
│   ├── repeats.py        # k-mer specification parser
│   ├── report.py         # Self-contained HTML report generator
│   └── visualize.py      # Matplotlib plotting utilities
├── tests/
│   ├── data/
│   │   └── sample.fastq  # Bundled sample data (5 reads)
│   └── test_basic.py     # 59 tests covering all modules
├── .github/workflows/
│   └── ci.yml            # CI: Python 3.9–3.12 matrix
├── pyproject.toml
├── CITATION.cff
├── CHANGELOG.md
└── LICENSE
```

---

## Testing

```bash
pip install -e ".[dev]"
pytest -v
```

59 tests cover:
- k-mer specification parsing (single, range, mixed, edge cases)
- I/O across all formats (FASTQ, FASTA, gzip, BAM/CRAM detection, quality filtering)
- DNA helpers (reverse complement, canonicalization, Hamming distance, entropy)
- Detection algorithms (perfect matching, fuzzy matching, strand consistency, block merging)
- Multiprocessing (chunking, pass 1/pass 2, incremental summarization)
- Visualization (plot generation and file saving)
- HTML report generation with XSS prevention
- CLI output formats (GFF3, VCF)
- End-to-end integration with sample FASTQ data
- Input validation and error handling

CI runs the full test suite on Python 3.9, 3.10, 3.11, and 3.12 via GitHub Actions.

---

## Notes & Limitations

- **Stdin buffers into memory:** The two-pass algorithm requires reading data twice. When reading from stdin, all records are buffered in memory. For large datasets (>1M reads), pass a file path instead.
- **Single-base repeat confidence:** Motifs with zero Shannon entropy (e.g., `AAAAAA`) will receive a confidence score of 0, since the entropy component is multiplicative. This is by design — low-complexity repeats are less likely to be true telomeric motifs.
- **BAM/CRAM filtering:** Secondary and supplementary alignments are silently skipped. Only primary alignments are processed.
- **Large k values:** k-mer sizes above ~50 rarely match biological telomeric repeats. The tool will warn if large k values are specified.
- **Fuzzy mode and short motifs:** Fuzzy matching with `--max-mismatch 1` at small k (e.g., k=4) allows 25% divergence per k-mer copy, which may produce false positives. Consider using `--max-mismatch 0` or higher `--min-run-bp` for small k values.

---

## Scientific Rationale

Telomeric regions are composed of tandem repeats and are often poorly represented in reference
genomes due to their repetitive nature. Long-read sequencing (ONT / PacBio) frequently captures
telomeric arrays directly within reads, making alignment-free discovery both feasible and
advantageous — especially for non-model organisms where the telomeric motif may be unknown.

TeloScan's two-pass approach first discovers candidate motifs without prior knowledge, then
re-scans with error tolerance to recover blocks degraded by sequencing noise. Canonicalization
ensures that equivalent motif representations (rotations and reverse complements) are unified,
enabling accurate quantification across strands.

---

## Contributing

Contributions are welcome. To get started:

```bash
git clone https://github.com/glbala87/TeloScan.git
cd TeloScan
pip install -e ".[dev]"
pytest -v
```

Please ensure all tests pass before submitting a pull request.

---

## Citation

If you use TeloScan in your research, please cite:

```
Gattu Linga, B.S. (2025). TeloScan: De novo discovery of telomeric repeat blocks
from long-read sequencing. https://github.com/glbala87/TeloScan
```

---

## License

MIT License. See [LICENSE](LICENSE) for details.
