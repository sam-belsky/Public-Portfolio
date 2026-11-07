# Optimization validation

Every performance change made to `Folded_TASEP_No_Diffusion.cpp`: what it does, how
much it is worth, and the evidence that it does not change the answers.

Start with **`figure/optimization_summary.png`**. Every number in it is computed from
the CSVs in these folders at render time — nothing is typed in.

```
benchmark/              one binary, one interleaved run, the master table
figure/                 the summary figure and the values it plots
01_graph_surgery/       Tree Surgery                        488b392
02_[withheld]/  Unpublished Optimization                  488b392
03_vienna_removal/      Local Energy Use                    3226dba
04_species_reuse/       Binder Reaction Salvage             55176bb + 2 patches
06_reverse_reaction/    Automatic Reverse Reaction Gen.     488b392
05_scanning_halt/       NOT an optimization - a bug fix     24607d0
07_external_comparison/ vs Kinfold on ADH1 (KFOLD pending)  2026-11-02
```

## The headline

ADH1 (1,174 nt), one core, every optimization on. **Median** of 8 replicates:

| species | computing time / biological time |
|---|---|
| generating a metastable structure | **1.3** |
| translating from a metastable structure | **0.2** (faster than real time) |
| translating, 8-ribosome polysome | **0.4** (faster than real time) |

Median rather than mean, and that is not a presentational choice. These runs vary
5–7× across replicates, and the variation is **real work**: nanoseconds per energy
evaluation is flat across replicates while energies *per event* varies four-fold,
because different trajectories generate different amounts of folding. That skews the
distribution right, so the mean (1.7 / 0.2 / 0.5) sits above what a run typically
costs.

Two caveats that do not go away with more replicates. Runs are capped at 6–10k events
and reach only ~3–5 simulated seconds, so this is a rate over the **opening phase**,
where folding is most expensive — a long run could land below these. And cost scales
steeply with transcript length: CFTR at 6,070 nt took ~4 h for a 400 s fold. This is
a per-gene figure, not a property of the simulator.

## The five optimizations

Ratios are "how much worse without it", geometric mean of per-replicate paired
ratios, n=8. **ns** = not resolved: the 95% CI spans 1.0.

| optimization | how it is switched | time (foldgen / metastable / polysome) |
|---|---|---|
| Tree Surgery | always on; no shipped switch | **14.0× / 39.1× / 77.7×** |
| Unpublished Optimization | always on; no shipped switch | **7.2× / 3.2× / 3.6×** |
| Local Energy Use | on by default; `--vienna-energy` reverts | **2.9×** / 1.00 ns / 1.06 ns |
| Binder Reaction Salvage | **OFF by default**; `--recycle-species` | 1.08 ns / **1.19×** / 0.87 ns |
| Automatic Reverse Reaction Gen. | always on; no shipped switch | ns / ns / ns |

**Tree Surgery and Unpublished Optimization are the figure.** Both are resolved on every
species and every metric, by wide margins, and both are validated exactly.

**Local Energy Use helps only the folding-heavy workload.** It replaces ViennaRNA
calls, and the translating runs make few of them.

**The reverse shortcut is exactly correct and unmeasurable.** It removes 3–38% of
ViennaRNA calls but a negligible share of *total* energy evaluations, so no timing
resolves it. Both statements are true and the figure says both.

**Three of the five have no off-switch in the shipped simulator.** `--no-surgery`,
`--no-[withheld]` and `--no-reverse` exist only to make these measurements possible;
`benchmark/build_bench_binary.sh` patches them in. They are benchmark scaffolding,
not program features.

## How each one was validated

`--check-surgery` is an exact oracle: it rebuilds the structure graph and every
**folding** reaction from scratch after each event and **throws** on any mismatch.
Identity, not similarity. It cannot be on during a timing run, and it does **not**
cover *species* reactions — which is why Binder Reaction Salvage rests on a weaker
statistical test.

| optimization | validation |
|---|---|
| Tree Surgery | `--check-surgery`: 0 mismatches, 29.7M energies |
| Unpublished Optimization | `--check-surgery`, both arms: 0 mismatches, 8.2M energies |
| Local Energy Use | fold energy within [−6.0, +2.4] kcal/mol, n=24 |
| Binder Reaction Salvage | 3 paired arms, n=20: on-vs-off −0.02% against a 0.18% same-code floor |
| Automatic Reverse Reaction Gen. | `--check-surgery`, both arms: 0 mismatches, 172.4M energies |

A passing oracle leaves no artifact, so a clean run and an unrun one look identical
on disk. The harnesses therefore echo their own command line into each log.

## Two mistakes this work found, and what they cost

**A correctness bug in `--recycle-species`.** The invalidation rule asked whether a
folding event came within `SPECIES_LOOKAHEAD_NT = 40` of a binder. But a binder's step
rate reads the stacking energy of the helix ahead of it, touching **both ends** of
each pair — and a pair's far end can be anywhere. 18% of ADH1's metastable pairs span
more than 40 nt; the longest spans 1,061. So an event could change the far end of a
pair in the very helix setting the rate, be classified unreachable, and leave a stale
rate. On a polysome this shifted dwell by **+25% against a 1% floor** (p = 0.013).
Fixed by following pairs instead of counting nucleotides.

**A measurement error that inflated one bar 3×.** The `--recycle-species` arm was
measured in its own session and divided by a baseline from a *different build*, so
part of the "speedup" was the gap between two run sessions. It reported **4.12×** on a
polysome. Measured against its own interleaved baseline it is **0.87× (ns)**. Every
arm now comes from one binary in one loop, which is what `run_benchmark_all.sh` is
for.

## Things that are easy to get wrong, and did get wrong here

**A pre-placed `80S` never elongates.** Reaction 24 needs a ribosome past its first
glide step, so `--pre-place <rna>:80S:12` parks a roadblock on one codon. Use
`40S+60S:24:28`. Symptom: one codon with nonzero dwell, and **no change when k8 is
lowered to 0**.

**A validation workload must contain the thing being validated.** The three-arm test
for Binder Reaction Salvage was vacuous twice: once with a frozen ribosome (all three
arms byte-identical, reporting a perfect pass), once with a single ribosome (almost
nothing to recycle). `local_rate_compare.py` now refuses a verdict unless the arms
did equal work and the ribosomes actually moved.

**Stopping on simulated time is the wrong instrument.** With `--standalone` every run
does a different amount of work; spreads reached 25–35% and two durations disagreed in
*direction*. `--max-events` fixed it.

**A workload with no written-down recipe is not reproducible, even with its logs.**
The polysome was once produced by a shell command that was lost. Its raw logs
survived and could not be regenerated or checked. It is now eight ribosomes
pre-placed at fixed positions — so `metastable` and `polysome` differ in exactly one
thing, the number of binders.

**`488b392` bundled physics fixes with tree surgery**, so its baseline runs 1.66×
more events, of a different kind. That is why there is no cumulative build-over-build
chain in the figure.

## Superseded, kept deliberately

Each is the "before" half of a documented finding:

- `benchmark/results/raw_chain/` — the confounded build chain
- `benchmark/results/timings_fixed_work_PARKED80S_invalid.csv` — the parked-80S runs
- `04_species_reuse/results/polysome_arms.csv` — the pre-fix arms showing the +25%
- `04_species_reuse/results/three_way.csv` — the three designs against each other
- `05_scanning_halt/results/first_buggy_attempt/` — the runs that caught the A-site bug

## Reproducing

Timings need an idle machine.

```bash
cd benchmark
REPO=~/Documents/Yeast_Translational_Modelling_Repo bash build_bench_binary.sh
bash check_default_path.sh          # the patches must not touch the default path
NREP=8 POLY_BUDGET=6000 bash run_benchmark_all.sh
python assemble_metrics.py && python headline_ratio.py

cd ../01_graph_surgery      && SIM=/tmp/bench_binary/sim bash check_surgery.sh
cd ../02_nucleation_salvage && SIM=/tmp/bench_binary/sim bash [withheld]_validate.sh
cd ../06_reverse_reaction   && SIM=/tmp/bench_binary/sim bash validate_reverse.sh
cd ../03_vienna_removal     && SIM=/tmp/bench_binary/sim NREP=12 bash backend_runs.sh && python backend_compare.py
cd ../04_species_reuse      && SIM=/tmp/bench_binary/sim NREP=20 bash local_rate_arms.sh && python local_rate_compare.py

cd ../figure && python make_figure.py
```

### Two problems worth fixing

**`stacking_energies.hpp` is not tracked by git**, and commits from `55176bb` onward
include it — so a clean clone of those commits does not compile. The build scripts
take headers from the working tree, which sidesteps it rather than fixing it.

**Neither `--recycle-species` patch is in the repo source.** Both copies of
`Folded_TASEP_No_Diffusion.cpp` still contain the original nucleotide-window guard
and the drop-and-rebuild design. The flag ships off, so the pipeline is unaffected —
but anyone passing `--recycle-species` to the committed binary gets stale rates.
`build_bench_binary.sh` applies both patches; the source does not have them.
