# Analysis and parameterisation scripts

The code in `../src` is the simulator. These are the scripts that drive it: fitting its rate
parameters against ribosome profiling data, evaluating the fit on held-out genes, and measuring what
each performance optimization is worth.

**The scripts themselves are not in this repository** (see the code-availability note in the project
README). What is here is the data they consume and a description of what each one does.

**The data is here.** `parameterisation/data/` holds the ribo-seq reference for
56 genes (from Papadopoulos et al. 2024, see `data/DATA_SOURCE.md`) and the simulator inputs for all
23 genes in the training and held-out panels — 620 KB in total.

The one missing piece is the **starting-structure libraries**: 1.65 GB across 5,229 files, too large
to ship. They are generated rather than measured, and `fold_heldout.sh` rebuilds them from the fold
times in `onsets.csv`, at roughly 2 minutes per structure for a 1,200 nt transcript.

## parameterisation/

| File | What it does |
|---|---|
| `select_genes_structure.py` | Picks the gene panel: coverage, reading-frame fraction, and how much secondary structure a transcript has |
| `fold_heldout.sh` | Builds a starting-structure library per gene — 40 independent folds from an open chain, each run for 2x that gene's measured metastable onset |
| `multi_gene_screen.py` | The screen driver: sweeps a rate parameter across trials x genes x replicates and reports the mean Spearman rho |
| `param_screen.py` | The scorer. Runs the replicates, maps simulated transcript codons onto CDS codons, matches smoothing between simulation and reference, and computes rho with a circular-shift null and a split-half reliability estimate |
| `evaluate_heldout.py` | Scores a fitted parameter on genes that were not used to fit it |
| `genes_train_structure_9.csv`, `genes_heldout8_noleak.csv` | The training and held-out panels, kept disjoint |
| `onsets.csv` | Measured metastable folding onset per gene, which sets each library's fold time |
| `data/riboseq_reference/` | Per-codon ribosome occupancy, 56 genes (Papadopoulos et al. 2024) |
| `data/gene_inputs/` | Transcript, reactions, parameters and decoding table for all 23 panel genes |

Two things in `param_screen.py` are worth reading if you only read one thing: the codon alignment at
`passed_codons()` — simulated codons are in transcript coordinates and have to be shifted onto CDS
codons — and the matched smoothing, since comparing a smoothed reference against an unsmoothed
simulation halves the measured correlation.

## optimization/

| File | What it does |
|---|---|
| `make_figure_noexplain.py` | Draws `../results/optimization_effects.png` from the measured CSVs; no number is typed in |
| `optimization_summary.csv` | Per-optimization, per-workload ratios with 95% confidence intervals |
| `OPTIMIZATION_NOTES.md` | The full record: what each optimization does, how it was validated, and two measurement mistakes that were caught and corrected |

`OPTIMIZATION_NOTES.md` is worth reading for the two self-caught errors it documents — a correctness
bug in one optimization found by a statistical test, and a benchmarking error that inflated a result
threefold by comparing across two run sessions instead of one interleaved loop.
