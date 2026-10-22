# Cost-Aware Retail Search

A reproducible benchmark of semantic product ranking and selective cross-encoder reranking, focused on the trade-off between ranking quality and inference cost.

## Overview

Improving search relevance can increase inference latency and computational cost. This project evaluates whether selectively reranking semantic search results with a cross-encoder provides a measurable quality improvement worth its additional inference cost.

Using the Amazon ESCI product-search dataset (Task 1, US locale), the benchmark compares four ranking strategies:

- **BM25:** lexical ranking baseline.
- **Semantic ranking:** sentence embeddings generated with `all-MiniLM-L6-v2`.
- **Cross-encoder reranking:** reranking all candidates with `ms-marco-MiniLM-L-6-v2`.
- **Selective reranking:** reranking only the top K candidates from semantic ranking.

The objective is not to claim a novel ranking algorithm. It is to implement a consistent evaluation workflow, measure quality and latency, quantify uncertainty, and document the engineering trade-offs.

## Key findings

The reported offline benchmark uses 200 test queries containing 4,103 candidate rows, with a mean of approximately 20.5 candidates per query.

| Ranking strategy | NDCG@10 | Exact in top 5 | Cross-encoder pairs |
|---|---:|---:|---:|
| BM25 | 0.8178 | 0.905 | 0 |
| Semantic only | 0.8445 | 0.940 | 0 |
| Selective rerank K=5 | 0.8457 | 0.940 | 1,000 |
| Selective rerank K=10 | 0.8428 | 0.920 | 1,999 |
| Selective rerank K=20 | 0.8474 | 0.915 | 3,321 |
| Selective rerank K=30 | 0.8471 | 0.915 | 3,725 |
| Cross-encoder, all candidates | 0.8465 | 0.915 | 4,103 |

*Values are from the recorded notebook run. NDCG@10 uses graded ESCI relevance labels; exact-in-top-5 measures the share of queries with at least one exact-match product in the first five results.*

### What the results indicate

- Semantic ranking outperforms the BM25 baseline on this evaluation sample.
- Selective reranking produces small observed NDCG@10 changes, but paired bootstrap confidence intervals include zero for the comparisons between selective-reranking configurations and semantic-only ranking.
- Increasing K does not consistently improve ranking quality.
- Semantic-only ranking avoids cross-encoder inference, making it the sensible default for this workload unless further evaluation establishes a benefit from reranking.

These findings apply to the sampled queries and candidate sets. They do not establish that semantic ranking will outperform lexical retrieval for every query or catalogue.

## Operational benchmark

The notebook also measures the service with one query per call, using CPU inference, a reranking batch size of 32, one warm-up call, and no concurrency.

| Mode | Median total latency | P95 total latency |
|---|---:|---:|
| Semantic only | 16.76 ms | 26.12 ms |
| Selective rerank K=5 | 111.27 ms | 581.95 ms |
| Selective rerank K=20 | 345.70 ms | 655.15 ms |

The recorded offline product-index preparation time was 70.9 seconds. Model loading, network time, and one-time index preparation are excluded from the query-time latency figures.

These timings are machine- and workload-dependent measurements, not production latency guarantees. The relatively high P95 latency for selective reranking also shows why average or median latency alone is insufficient for evaluating an inference strategy.

## Methodology

### Dataset and relevance

The notebook uses the Amazon ESCI shopping-query dataset, restricted to Task 1 and the US locale. It selects 200 complete test queries using a fixed random seed of 42.

ESCI labels are mapped to graded relevance:

| Label | Meaning | Relevance |
|---|---|---:|
| E | Exact | 3 |
| S | Substitute | 2 |
| C | Complement | 1 |
| I | Irrelevant | 0 |

### Search index

Product titles are encoded with `sentence-transformers/all-MiniLM-L6-v2`. Normalized embeddings are stored in a NumPy matrix, with a title-to-row mapping used for candidate scoring.

Index preparation is separated from query-time inference. The current service requires all supplied candidate titles to exist in the index.

### Ranking and evaluation

The benchmark evaluates:

- NDCG@10 using graded relevance.
- Exact-match presence in the top 5 and top 10.
- Cross-encoder candidate-pair counts.
- Measured inference time.
- Paired bootstrap confidence intervals for per-query NDCG@10 differences.

The same ranking policy is shared by the evaluation code and the `search_candidates()` service function. Selective reranking reorders the top K semantic candidates by cross-encoder score and retains the remaining candidates in their semantic order.

Cross-encoder scores and semantic similarity scores are not directly compared because they have different scales.

## Repository structure

The main deliverable is the consolidated Google Colab notebook.

```text
cost-aware-retail-search/
├── README.md
└── Retail_Search_and_Ranking_Engine_Revised.ipynb
```

The notebook creates benchmark and bootstrap-confidence-interval CSV files under its configured results directory when executed successfully.

## Running the project

1. Open `Retail_Search_and_Ranking_Engine_Revised.ipynb` in Google Colab.
2. Run all cells from top to bottom in a fresh runtime.
3. Allow the notebook to install its required Python packages and download the dataset and pretrained models.
4. Review the benchmark table, bootstrap confidence intervals, service tests, and operational latency measurements.
5. Inspect the generated CSV files for the recorded evaluation results.

The notebook uses CPU inference and requires an internet connection for initial dataset and model downloads. Runtime and storage requirements depend on the execution environment; the source products file downloaded by the notebook is approximately 1.1 GB.

## Limitations

- **Candidate retrieval:** The service ranks a supplied candidate set. It does not independently retrieve candidates from a full product catalogue.
- **Evaluation size:** Results are based on one deterministic sample of 200 test queries.
- **Lexical baseline:** BM25 is fitted separately within each query's judged candidate set. Its inverse document frequencies are therefore estimated from a small pool and should not be interpreted as a tuned, production-grade BM25 baseline.
- **Model transfer:** The cross-encoder was pretrained on MS MARCO and was not adapted to ESCI.
- **Inference measurements:** Timing results describe sequential CPU calls without concurrency and may differ across machines.
- **Statistical uncertainty:** The observed gains from selective reranking over semantic-only ranking are not statistically conclusive under the reported paired bootstrap intervals.

## Conclusion

This project demonstrates a reproducible approach to evaluating ranking quality alongside inference cost. In the recorded workload, semantic ranking offers a strong baseline, while selective cross-encoder reranking adds substantial latency without establishing a statistically conclusive ranking-quality improvement.

The result is a measured engineering trade-off rather than a claim that more expensive ranking is necessarily better.

## Acknowledgements

- Amazon Science for the ESCI shopping-query dataset.
- Sentence Transformers for the embedding and cross-encoder models.
- The maintainers of the Python libraries used in the benchmark.
