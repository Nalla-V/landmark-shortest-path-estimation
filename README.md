# Landmark-based Shortest-Path Distance Estimation

Estimating shortest-path distances in large graphs without running a BFS per query, by
precomputing distances from a small set of landmark nodes and combining them with
triangle-inequality bounds. This project compares three standard landmark selection
heuristics against a **Brandes-inspired, structure-aware** strategy that picks landmarks by
how often they participate in shortest paths.

On the Douban and Twitch graphs, two landmarks chosen by the Brandes-inspired strategy gave
a lower error than 500 randomly chosen ones — a budget reduction of more than two orders of
magnitude.

**Full write-up, with per-dataset error curves and the complete comparison:**
[`Course_project_Report.pdf`](Course_project_Report.pdf)

## Background

Exact distance queries on large graphs are expensive: a BFS costs O(m) per source, and
precomputing all pairs needs O(mn) time and O(n²) space. Landmark methods trade a one-off
preprocessing cost for fast approximate answers. A set D of k landmarks is chosen, the
distance from every node to every landmark is precomputed, and a query (s, t) is answered
with the tightest upper bound over landmarks:

```
d̂(s, t) = min over u in D of  d(s, u) + d(u, t)
```

This never underestimates, and becomes exact whenever some landmark lies on a shortest path
between s and t. Accuracy therefore depends almost entirely on *which* nodes become
landmarks — and choosing an optimal set is NP-hard, so everything in practice is a
heuristic.

Ranking heuristics such as degree or closeness have a known failure mode: the top-ranked
nodes often sit in the same dense region and carry redundant information.

## Approach

Seven strategies, all evaluated under the same pipeline.

**Baselines** (`LM.py`)
- `random` — uniform selection, the reference point
- `degree` — highest-degree nodes
- `closeness` — sampled closeness centrality, using BFS from 200 sampled sources rather
  than the O(mn) exact computation
- `*_h` variants — hop-exclusion, forbidding a landmark within `h_min` hops of an
  already-selected one, to break up clustering

**Structure-aware** (`LM_brandes.py`)
- `brandes` — a seed landmark is chosen by degree, then each BFS additionally accumulates
  Brandes-style dependency values into a global participation score. The next landmark is
  the highest-scoring remaining node, falling back to degree when the score is
  uninformative.
- `brandes_h` — the same, with hop-exclusion

The dependency accumulation reuses the machinery behind Brandes' betweenness algorithm, but
only from the landmarks already selected, so it stays far cheaper than computing exact
betweenness.

## Datasets

Five public real-world graphs, all reduced to simple undirected graphs with self-loops
removed by a common preprocessing step.

| Dataset | \|V\| | \|E\| | ⟨k⟩ | clustering |
|---|---|---|---|---|
| [Facebook Page–Page](https://snap.stanford.edu/data/facebook-large-page-page-network.html) | 22,470 | 170,823 | 15.20 | 0.356 |
| [Douban Friendship](https://networks.skewed.de/net/douban) | 154,908 | 327,162 | 4.22 | 0.013 |
| [DBpedia Record Label](https://networks.skewed.de/net/dbpedia_recordlabel) | 186,689 | 233,286 | 2.50 | 0.008 |
| [Email-Enron](https://snap.stanford.edu/data/email-Enron.html) | 36,692 | 367,662 | 20.04 | 0.489 |
| [Twitch Gamers](https://snap.stanford.edu/data/twitch_gamers.html) | 168,114 | 6,797,556 | 80.87 | 0.159 |

The edge lists are not committed — download them from the links above and point `input_tsv`
at them.

## Evaluation

Per dataset, a fixed set of 500 query pairs is generated once and reused across every
strategy and budget. The set is stratified by hop distance — roughly 100 short (≤ 2), 200
medium (3–5) and 200 long (≥ 6) — so accuracy can be read per distance band instead of
being averaged into a single number that hides where the estimator fails.

Landmark budgets: |D| ∈ {2, 5, 10, 20, 50, 100, 500}, with `h_min` ∈ {0, 1}.
Metric: mean relative error |d̂ − d| / d over the 500 queries.

## Results

Random is the weakest strategy on every dataset. The interesting differences are all at
small budgets — once the budget is moderate, the informed strategies converge and the
choice of heuristic stops mattering much.

- **Douban and Twitch** — two landmarks from Brandes or Degree beat 500 random ones. The
  informed curves then converge quickly.
- **DBpedia and Email-Enron** — Brandes and Degree give the lowest error at |D| ∈ {2, 5};
  sampled closeness only becomes competitive once more landmarks are available.
- **Facebook** — the hop-excluded variants of closeness and Brandes are best at very small
  budgets, suggesting that forcing separation between landmarks helps most when there are
  few of them.

Hop-exclusion helps on some graphs and not others, and coarse statistics like average degree
or clustering coefficient do not predict which. Email-Enron gains a lot from informed
selection while DBpedia gains much less, despite comparable clustering.

**Index construction time**, |D| = 100 on Facebook (seconds):

| Strategy | T_LM | T_distance | T_index |
|---|---|---|---|
| Random | 0.0002 | 1.99 | 1.99 |
| Degree | 0.25 | 1.99 | 2.25 |
| Degree (hop-1) | 0.26 | 2.13 | 2.39 |
| Closeness | 4.85 | 1.97 | 6.82 |
| Closeness (hop-1) | 4.82 | 2.05 | 6.87 |
| Brandes | 1.35 | 6.55 | 7.90 |
| Brandes (hop-1) | 1.41 | 6.63 | 8.04 |

Preprocessing grows roughly linearly with |D|, dominated by the one BFS per landmark.
Brandes pays more in `T_distance` because each BFS also accumulates dependencies — the
accuracy gain at small budgets comes at a measurable offline cost.

Query time stays negligible throughout: 0.0046 ms at |D| = 2, rising to 0.68 ms at
|D| = 500, in line with the O(|D|) scan.

Single-core Python, AMD Ryzen 7 7435HS, 16 GB RAM. All times are wall-clock.

The per-dataset error curves for all five networks are Figure 4 of the paper.

## Requirements

Python 3.10+.

```bash
pip install -r requirements.txt
```

## Configuration

All scripts read `config.yaml`:

- `input_tsv` — path to the raw edge list (used by `preprocess.py`)
- `output_dir` — folder where outputs for the dataset are written
- `k` — number of landmarks
- `lm_sel` — `random`, `degree`, `degree_h`, `closeness`, `closeness_h`, `brandes`,
  `brandes_h`
- `h_min` — hop-exclusion threshold (0 = none, 1 = exclude neighbours)
- `closeness_samples` — sampled BFS sources for closeness (default 200)
- `queries` — list of query pairs, copied from `generated_queries.yaml`

Set `output_dir` to a dataset-specific folder (`Facebook/`, `Douban/`) so runs do not
overwrite each other.

## Running the pipeline

```bash
python preprocess.py             # edge list -> edges.parquet, node_map.json
python stats.py                  # optional: |V|, |E|, degree, clustering
python distance_distribution.py  # optional: hop-distance histogram
python generate_queries.py       # 500 stratified query pairs
```

Copy the generated queries from `<output_dir>/generated_queries.yaml` into `config.yaml`
under `queries:`, then:

```bash
python true_distance.py          # exact distances for the 500 queries
python LM.py                     # baselines  (or: python LM_brandes.py)
python evaluate.py               # approximation error per query
python approx_quality_plot.py
python timing_plot.py
```

## File layout

| File | Purpose |
|---|---|
| `preprocess.py` | edge list to dense-ID graph |
| `generate_queries.py` | stratified query pair generation |
| `true_distance.py` | exact distances for the query set |
| `LM.py` | random / degree / closeness selection and distance table |
| `LM_brandes.py` | structure-aware selection |
| `evaluate.py` | estimate distances, report error |
| `stats.py` | dataset statistics |
| `distance_distribution.py` | hop-distance distribution plot |
| `approx_quality_plot.py` | error vs. landmark budget |
| `timing_plot.py` | preprocessing time vs. landmark budget |

Outputs per dataset: `edges.parquet`, `node_map.json`, `landmarks.json`,
`distances.parquet`, `generated_queries.yaml`, `true_distances.json`,
`{k}_timing_{lm_sel}.json`, `{k}_approx_quality_{lm_sel}.json` and figures.

## Limitations and what I would change

- Query pairs have to be copied by hand from `generated_queries.yaml` into `config.yaml`.
  The pipeline should read the generated file directly — this is the most fragile part of
  the workflow.
- The participation score is accumulated only from BFS runs at already-selected landmarks,
  so it is biased toward the region around the seed. That bias was not measured against
  exact betweenness.
- Only unweighted, connected, undirected graphs are supported.
- Hop-exclusion was tested only at h_min ∈ {0, 1}. Larger thresholds might matter more on
  the high-clustering graphs, where redundancy is worst.

## Contributor

- Nallathambi Vethiappan
- Godwin

