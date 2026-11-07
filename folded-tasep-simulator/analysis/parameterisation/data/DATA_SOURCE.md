# Data sources

## riboseq_reference/

Per-codon ribosome occupancy for 56 *Saccharomyces cerevisiae* genes, one CSV per ORF
(`<ORF>_Occupancy.csv`, columns `codon,count`). These are the measurements every rate parameter in
this project is fitted and tested against.

Derived from the ribosome profiling data published with:

> Papadopoulos C., Roginski P., Lopes A. (2024)

distributed by those authors as a complete analysis package (`MetaRiboSeq_All_Files.tar.gz`,
`Papadopoulos_et_al_data/`). Their pipeline and raw-data instructions are in that package; related
tooling is at https://github.com/i2bc/ORFmine.

The files here are the per-codon counts extracted for the genes used in this project. The underlying
data is theirs and is redistributed on the basis that it was published; cite the paper above, not
this repository, if you use it.

## gene_inputs/

23 gene directories — the 9 training and 14 held-out genes. Each holds the four files the simulator
needs for that transcript:

| File | Contents |
|---|---|
| `sequences.csv` | The mature transcript, 5'UTR through 3'UTR |

Transcript sequences are from the yeast genome and are public.

**Not included: the model specification.** Each gene directory also needs `reactions.csv` (the 29
reactions and their rate expressions), `parameters.csv` (the fitted rate constants) and
`aa_decoding.csv` (per-amino-acid decoding values). Those define the model rather than describe the
data, so they fall under the same code-availability note as the source — see the project README.

## What is not here: the structure libraries

Each gene also needs a library of starting structures — 40 to 200 independent folds from an open
chain, each run for twice that gene's measured metastable onset. Those are **1.65 GB across 5,229
files**, so they are not shipped. They are generated, not measured, and are reproducible from the fold times in `onsets.csv` by the
folding step described in `../../README.md`. Expect roughly 2 minutes per structure for a 1,200 nt transcript, scaling as about nt^1.9, and
budget memory at roughly 2.4e-4 x nt^2 MB per concurrent fold.
