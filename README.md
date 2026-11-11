<div align="center">

# SGA-IEA

### Bridging Semantic Gaps and Acquiring Evidence for Reliable Hybrid Query Reasoning

Reference implementation of **Semantic Gap Alignment (SGA)** and
**Interactive Evidence Acquisition (IEA)** for hybrid query reasoning over tables.

[![License: Apache 2.0](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](LICENSE)
[![Python 3.10+](https://img.shields.io/badge/Python-3.10%2B-3776AB.svg)](https://www.python.org/)

</div>

![Motivation](assets/motivation.png)

## Overview

Hybrid queries combine relational operations (filter, join, aggregate) with
semantic operations whose predicates are not explicit in the table schema.
A system must decide **what** to execute and then keep intermediate decisions
**grounded** while it executes. Errors at either stage propagate to the answer.

This repository contains the code for two complementary mechanisms:

| Component | Stage | What it does | Code |
|---|---|---|---|
| **SGA** — Semantic Gap Alignment | before planning | Detects semantic gaps between the query and the table, assigns each gap a *source* (Explicit / Implicit / Derivable / External / Unresolved) and a confidence, resolves only sufficiently supported gaps, and verifies each alignment against the table. | [`src/hyq/parse.py`](src/hyq/parse.py), [`src/hyq/preprocess.py`](src/hyq/preprocess.py), [`src/hyq/plan.py`](src/hyq/plan.py), [`src/hyq/small_adapt.py`](src/hyq/small_adapt.py), [`src/hyq/enhanced_query.py`](src/hyq/enhanced_query.py) |
| **IEA** — Interactive Evidence Acquisition | during execution | Gives the planner read-only evidence access functions over the **immutable original table** `T0` (column profile, value lookup, row retrieval, statistics), so it can verify an assumption before committing to an operation. | [`src/hyq/probes.py`](src/hyq/probes.py), [`src/hyq/multi_probe.py`](src/hyq/multi_probe.py), [`src/hyq/solve.py`](src/hyq/solve.py) |

The planner–executor loop is implemented in [`src/hyq/solve.py`](src/hyq/solve.py),
[`src/hyq/plan.py`](src/hyq/plan.py) and [`src/hyq/execute.py`](src/hyq/execute.py).
Semantic and relational operators run on a vendored fork of
[LOTUS](https://github.com/lotus-data/lotus) in [`src/lotus/`](src/lotus).

## Repository layout

```text
SGA-IEA/
├── src/
│   ├── hyq/                 # solver, SGA, IEA, prompts, metrics
│   │   ├── solve.py         # planner–executor loop
│   │   ├── plan.py          # prompts for every planner variant
│   │   ├── parse.py         # SGA alignment / link / decomposition
│   │   ├── preprocess.py    # schema selection (consumes SGA)
│   │   ├── probes.py        # IEA evidence access functions
│   │   ├── multi_probe.py   # batched IEA tool
│   │   ├── execute.py       # operator execution + optional reflection
│   │   ├── llm.py           # API and local model adapters
│   │   ├── metrics.py       # tokens / actions / probes / latency
│   │   └── data/            # table loading and operator registry
│   ├── lotus/               # vendored LOTUS fork (semantic operators)
│   └── runners/             # dataset entry points + judge
├── scripts/                 # run / download / smoke / evaluate
├── data/
│   ├── samples/             # committed: 3 runnable cases per benchmark
│   ├── test/                # downloaded: evaluation queries
│   └── tables/              # downloaded: full tables
├── results/main/            # reference scores (predictions are not bundled)
├── docs/                    # architecture, data, reproduction, FAQ
└── assets/
```

## Installation

Requires Python 3.10+.

```bash
git clone https://github.com/your-org/SGA-IEA.git
cd SGA-IEA

python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

cp .env.example .env      # then edit HYQ_API_BASE / HYQ_API_KEY
python scripts/check_install.py
python -m unittest discover -s tests -v   # API-free checks
```

The API-only install needs no GPU. Local Qwen/Llama models additionally need
`pip install -r requirements-local.txt`.

## Data

The full evaluation queries and tables are **not** committed (the lightweight
release would otherwise contain thousands of files). Both archives are in the
[HyQBench Google Drive folder](https://drive.google.com/drive/folders/1VpyF05rvHxEAIU3bwERNnWosrpqK6rri):

```bash
bash scripts/download_data.sh /path/to/data.tar.gz /path/to/test.tar.gz
```

- `data.tar.gz` → shared tables (`data/tables/`)
- `test.tar.gz` → evaluation queries (`data/test/`) and per-case HyQBench-L tables

`data/samples/` contains three small cases per benchmark (query + tables) so the
code can be exercised without downloading anything. See
[`data/README.md`](data/README.md) for the schema of every benchmark.

## Quick start

Run one sample case per benchmark (requires an API key; costs a few cents):

```bash
bash scripts/smoke_test.sh
```

The equivalent single command for WikiTQ-H is:

```bash
PYTHONPATH=src python src/runners/run_wikitq.py \
  --model GPT-4.1 --use_evidence_grounded --max_samples 1 \
  --result_tag wikitq_smoke
```

## Reproducing the paper

Download the tables first, then run any benchmark:

```bash
bash scripts/run_wikitq.sh      # WikiTQ-H, 510 cases
bash scripts/run_tabfact.sh     # TabFact-H, 303 cases
bash scripts/run_finqa.sh       # FinQA-H
bash scripts/run_tag.sh         # TAG-Bench, 59-case paper subset
bash scripts/run_hyqbench.sh    # HyQBench-L, 76 cases
```

All runners accept `--shard i/n` for parallel execution, e.g.

```bash
bash scripts/run_wikitq.sh --shard 0/6 &   # shards are resumable
```

Outputs are written to `outputs/lotus/<model>/...`. Score a run with:

```bash
python src/runners/evaluate.py \
  --pred-dir outputs/lotus/GPT-4.1/answer/wikitq_sga_iea \
  --test-dir data/test/wikitq_hybrid --kind wikitq --judge v4
```

`--judge v4` is the bundled semantic judge (rules → single vote → 3 votes +
arbitration). `--judge rules` is free and deterministic but only resolves cases
the rules can decide. `--judge legacy` is the one-shot `canonical_v2` prompt used
for the paper table. HyQBench-L uses its own judge:

```bash
python src/runners/judge_hyqbench.py \
  --pred-dir outputs/lotus/GPT-4.1/hyqbench_sga_iea/answer \
  --test-dir data/test/end2end_light
```

Reference scores from the audited runs are in
`results/main/reference_scores.json`. This lightweight release does not bundle
the 948 per-case prediction files; they are part of the full release and can be
regenerated with the commands above.

See [`docs/reproduce.md`](docs/reproduce.md) for the full protocol, sharding
recipes and expected costs.

## Results

Answer Accuracy (%) with GPT-4.1 as planner and executor. The numbers below are
the audited reference runs; the camera-ready paper is authoritative.

| Method | WikiTQ-H | TabFact-H | TAG-Bench | HyQBench-L | HyQBench-F | FinQA-H |
|---|---:|---:|---:|---:|---:|---:|
| Ours (SGA + IEA) | 78.43 | 90.10 | 47.46 | 44.74 | — | — |

- WikiTQ-H / TAG use an LLM semantic-equivalence judge; TabFact-H is exact
  polarity match; HyQBench-L uses the factual-error prompt.
- FinQA-H and HyQBench-F runs are not part of this release; their data loaders
  and runners are included for completeness.
- The exact commands, judge versions and caveats are documented in
  [`results/README.md`](results/README.md). Per-case predictions are available in
  the full release.

## Ablations

Turn each component off from the command line:

```bash
# w/o SGA intent alignment (schema selection / decomposition remain)
bash scripts/run_wikitq.sh --no_sga

# IEA only (original ReAct prompt + probes, no SGA)
python src/runners/run_wikitq.py --model GPT-4.1 \
  --use_multi_probe_only --no_sga --result_tag wikitq_iea_only

# SGA only (no evidence tools)
python src/runners/run_wikitq.py --model GPT-4.1 --result_tag wikitq_sga_only
```

Passing neither `--use_evidence_grounded` nor `--use_multi_probe_only` gives the
SGA-only arm.

## Citation

```bibtex
@inproceedings{sga-iea,
  title     = {Bridging Semantic Gaps and Acquiring Evidence for Reliable Hybrid Query Reasoning},
  author    = {Anonymous},
  booktitle = {To appear},
  year      = {2026}
}
```

Update `CITATION.cff` and the BibTeX entry with the camera-ready author list.

## License

Apache-2.0. The vendored LOTUS fork and the benchmark datasets retain their
original licenses; see [`THIRD_PARTY.md`](THIRD_PARTY.md).
