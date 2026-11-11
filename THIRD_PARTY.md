# Third-party components and data

## LOTUS

`src/lotus/` is a vendored fork of
[lotus-data/lotus](https://github.com/lotus-data/lotus), the semantic-operator
library used to execute `sem_filter`, `sem_map`, `sem_agg`, `sem_topk`,
`sem_join`, and the relational operators. LOTUS is distributed under the
Apache-2.0 license. Local modifications in this fork are limited to:

- lazy imports of `torch` / `sentence-transformers` so API-only installations
  do not need a GPU stack;
- an OpenAI-compatible local-model adapter used by the runners;
- small fixes for the table/column formats in this project.

If you use LOTUS, please also cite the original paper:

```bibtex
@article{patel2024lotus,
  title   = {LOTUS: Enabling Semantic Queries with LLMs over Tables of Unstructured and Structured Data},
  author  = {Patel, Liana and Jha, Siddharth and Guestrin, Carlos and Zaharia, Matei},
  journal = {arXiv preprint arXiv:2407.11418},
  year    = {2024}
}
```

## Benchmarks

The query files and tables come from the original benchmark releases. They are
not relicensed by this repository; follow the terms of the corresponding
project.

| Benchmark | Source |
|---|---|
| WikiTableQuestions | <https://github.com/ppasupat/WikiTableQuestions> |
| TabFact | <https://github.com/wenhuchen/Table-Fact-Checking> |
| FinQA | <https://github.com/czyssrs/FinQA> |
| BIRD / TAG-Bench | <https://github.com/AlibabaResearch/DAMO-ConvAI/tree/main/bird> · <https://github.com/Leolty/TAG-Bench> |
| HyQBench | <https://github.com/XMUDM/HyQBench> |

The full tables are downloaded from the HyQBench Google Drive folder referenced
in `data/README.md`. The sample cases in `data/samples/` are excerpts of the
same data.

## Prompts and baselines

The planner and judge prompts in `src/hyq/plan.py` and `src/runners/` are the
implementation used for this paper. Baselines (BINDER, BlendSQL, ReAcTable,
H-STAR, ProTrix, Weaver) are external projects; their code is not vendored here.
