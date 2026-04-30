# ANCHOR Draft

**ANCHOR Draft** is a research-note repository for exact independent sampling from cross-color box-intersection joins. The main document, [`Anchor.tex`](Anchor.tex), specifies **ANCHOR**, an exact sampler for two sets of axis-aligned boxes in constant dimension. The repository also collects two baseline algorithms, a controllable synthetic rectangle generator specification, and real spatial-join benchmark designs.

This repository is currently a **LaTeX draft/specification workspace**, not a packaged implementation. Its purpose is to make the algorithmic model, correctness arguments, complexity claims, and benchmark design choices explicit and auditable.

## Problem Setting

Given two color classes of $d$-dimensional axis-aligned boxes,

$$
\mathcal R \quad\text{and}\quad \mathcal S,
$$

consider the cross-color box-intersection join

$$
J = \{(r,s) \in \mathcal R \times \mathcal S : r \cap s \neq \emptyset\}.
$$

For the core ANCHOR and KDS specifications, boxes are represented as half-open intervals:

$$
b = \prod_{i=1}^d [L_i(b), U_i(b)).
$$

A sampling query of length $t$ should return an ordered sequence of $t$ **independent uniform samples from $J$ with replacement**. If $J$ is empty and $t>0$, the sampler reports an empty instance. If $t=0$, the sampler returns the empty sequence.

## Repository Contents

| File | Role | Summary |
| --- | --- | --- |
| [`Anchor.tex`](Anchor.tex) | Main algorithm | Specifies **ANCHOR**, an exact sampler based on anchored ownership, canonical dyadic decomposition, recursive suffix joins, terminal atoms, and exact integer-weighted sampling. |
| [`KDS.tex`](KDS.tex) | Baseline | Static KD-tree baseline. It lifts right-side boxes into $2d$-dimensional points, computes exact left-side join degrees by range counting, and samples by degree-weighted left selection plus uniform conditional right selection. |
| [`SweepRT.tex`](SweepRT.tex) | Baseline | Two-pass plane-sweep/range-tree baseline. It decomposes the join into disjoint START-event blocks, counts block weights in pass 1, assigns output positions by weight, and samples selected blocks in pass 2. |
| [`Alacarte.tex`](Alacarte.tex) | Synthetic data | Documents the selected synthetic rectangle/hyperrectangle generation method. It controls the expected join-output density by solving for a coverage parameter. |
| [`RealDataset.tex`](RealDataset.tex) | Real data | Documents the selected real spatial-join datasets: CMAB-Spatial-Join-0.08B, GeoLife-Spatial-Join-0.15B, and COCO-Spatial-Join-1.23B. |

## ANCHOR at a Glance

ANCHOR separates the geometric decomposition of the join from the probabilistic composition of samples.

### Core Ideas

1. **Anchored ownership.** At each dimension, every intersecting cross-color pair is assigned to a unique current-dimensional owner, avoiding duplicate representation.
2. **Canonical dyadic cover.** Each anchor routes its feasible opposite-side interval through a canonical dyadic cover, producing pair-disjoint recursive suffix instances.
3. **Recursive suffix joins.** Once a dimension has been certified by ownership, recursion continues only on the remaining suffix dimensions.
4. **Terminal atoms.** The final one-dimensional suffix instances are represented by disjoint atoms whose weights are exactly their cardinalities.
5. **Exact probabilistic composition.** Multinomial quota allocation, exact weighted choices, uniform within-atom draws, and Fisher--Yates shuffling produce ordered i.i.d. samples with replacement.

### Materialization Levels

ANCHOR supports three levels of materialization over the same underlying disjoint decomposition.

| Method | Persistent representation | Space | Query time |
| --- | --- | --- | --- |
| **ANCHOR-Streaming** | Top-level sorted views only | $O(N)$ auxiliary space | $O(N\log^{d-1}(N+1)+t)$ |
| **ANCHOR-PrefixCache$(k)$** | Frontier suffix views and a frontier weighted index | $O(N\log^k(N+1))$ | Adaptive suffix-query bound |
| **ANCHOR-Compiled** | Terminal atoms and a global weighted index | $O(N\log^{d-1}(N+1))$ | $O(t)$ |

The guarantees are stated for constant dimension $d$ under an exact-RAM model. If exact prefix sums and binary search are used instead of a constant-time weighted-index primitive, the distribution remains exact while weighted draws gain a logarithmic factor.

## Baselines

### KDS

[`KDS.tex`](KDS.tex) specifies a static KD-tree baseline for exact uniform sampling over half-open box-intersection joins.

The algorithm proceeds as follows:

1. Lift each right-side box $s \in \mathcal S$ to a point in $2d$ dimensions.
2. Convert the strict half-open overlap condition induced by each left-side box $r \in \mathcal R$ into a closed orthogonal range query using exact boundary handling.
3. Use KD-tree range counting to compute the degree

   $$
   w_i = |\{s_j \in \mathcal S : r_i \cap s_j \neq \emptyset\}|.
   $$

4. Sample a left-side box with probability proportional to $w_i$.
5. Sample a right-side partner uniformly from the KD-tree range-query result using a disjoint item decomposition over point items and contained-subtree items.

KDS is useful as a conceptually simple static baseline: it factorizes the uniform join distribution into degree-weighted left selection and conditional right selection.

### SweepRT

[`SweepRT.tex`](SweepRT.tex) specifies a pure two-pass baseline based on plane sweep and dynamic range trees.

The algorithm proceeds as follows:

1. Sweep one dimension to decompose the full join into disjoint START-event blocks.
2. **Pass 1: Count.** Maintain two dynamic range trees and compute the exact weight of every event block.
3. Assign each requested output position independently to an event block according to its exact weight.
4. **Pass 2: Sample.** Replay the same sweep order and perform within-block range-tree sampling only for selected event blocks.

This baseline deliberately excludes prefetching, cache budgets, residual refill logic, and prefetch-scoring mechanisms. It is intended to be a clean count/sample comparison point.

## Synthetic Data: Alacarte

[`Alacarte.tex`](Alacarte.tex) documents the selected synthetic rectangle and hyperrectangle generator.

The generator targets the normalized output density

$$
\alpha_{\mathrm{out}} = \frac{|J(R,S)|}{|R|+|S|}.
$$

Instead of directly materializing or tuning against the full join, Alacarte solves a one-dimensional inverse problem over a **coverage** parameter $C$:

1. Choose input sizes $n_R,n_S$, dimension $d$, target density $\alpha_{\mathrm{out}}^\star$, universe, volume distribution, aspect-ratio model, and random seed.
2. Estimate the expected random-pair intersection probability using Monte Carlo side-length samples and a closed-form one-dimensional interval-intersection formula.
3. Convert the probability estimate into an expected output density.
4. Use bracketing and log-space binary search to find a coverage $C^*$ whose expected density is close to the target.
5. Generate concrete half-open boxes for $R$ and $S$ under $C^*$ and emit reproducible metadata.

Alacarte controls the expected output density. The realized join size of one generated dataset may fluctuate around the target because the rectangles are random.

## Real Datasets

[`RealDataset.tex`](RealDataset.tex) documents three real-data benchmarks designed for spatial-join sampling experiments.

| Dataset | Dimension | Source | Main role |
| --- | --- | --- | --- |
| **CMAB-Spatial-Join-0.08B** | 2D projected meter coordinates | Chinese national-scale building rooftop geometries | Large geospatial building/influence joins with administrative partitions, urban skew, and multi-radius selectivity. |
| **GeoLife-Spatial-Join-0.15B** | 3D $(x,y,t)$ and 4D $(x,y,z,t)$ integer boxes | GeoLife GPS trajectories | Spatiotemporal encounter joins with strong trajectory correlation and controlled selectivity levels. |
| **COCO-Spatial-Join-1.23B** | 3D half-open boxes $(x,y,z)$ | MS COCO 2017 annotations and deterministic RPN proposals | Billion-scale visual-object joins with heavy within-image overlap. |

Boundary semantics should be reported explicitly in experiments. ANCHOR and KDS are specified for half-open boxes and strict overlap. CMAB and GeoLife are documented with closed/inclusive intersection semantics, while COCO uses half-open semantics. Integer closed intervals can often be converted to half-open intervals by replacing $[\ell,u]$ with $[\ell,u+1)$ when the coordinate grid permits it.

## Suggested Experimental Workflow

1. **Choose a workload.** Use Alacarte for controlled synthetic density, or select one of the real datasets for realistic skew, correlation, and scale.
2. **Normalize semantics.** State whether the experiment uses half-open or closed intersection semantics, and apply any required conversion consistently across all algorithms.
3. **Build the sampler.** Compare ANCHOR variants against KDS and SweepRT under the same input representation and random-sampling primitives.
4. **Measure preprocessing.** Record indexing/build time, persistent memory, and exact join-cardinality metadata when available.
5. **Measure sampling queries.** Vary query length $t$, output density, dimension, and dataset family.
6. **Validate distributional correctness.** On small instances, compare against a materialized join; on large instances, use sanity checks such as exact weights, boundary-case tests, and repeated-sample diagnostics.

## Building the Notes

The repository consists of standalone LaTeX documents. With a standard LaTeX distribution, compile a document with, for example:

```bash
pdflatex Anchor.tex
pdflatex KDS.tex
pdflatex SweepRT.tex
pdflatex Alacarte.tex
pdflatex RealDataset.tex
```

Alternatively, use `latexmk`:

```bash
latexmk -pdf Anchor.tex
```

Some documents may require common LaTeX packages such as `amsmath`, `amssymb`, `mathtools`, `booktabs`, `enumitem`, `hyperref`, `algorithm`, `algpseudocode`, `longtable`, and `tabularx`.

## Current Status

This repository is a draft workspace for algorithm design and benchmark specification. It does not yet provide:

- a packaged implementation,
- a command-line interface,
- executable benchmark scripts,
- unit tests,
- dataset download/build automation, or
- a public release artifact.

Recommended next steps are to add implementation code, reproducible benchmark scripts, a formal citation entry, and a license.

## Citation

If you use this draft in academic work, please cite the corresponding paper or preprint once it becomes available. Until then, cite this repository and the relevant `.tex` document.

## License

No license is specified in this draft repository yet. Please add an explicit license before redistribution or reuse by external users.
