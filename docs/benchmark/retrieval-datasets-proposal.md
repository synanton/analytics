# Retrieval Benchmark Proposal: Mr. TyDi Russian, SciFact, and NFCorpus

**Status:** Proposed  
**Scope:** Offline information-retrieval evaluation for Synanton Search  
**Target repository:** `synanton/analytics`  
**Related platform capability:** BM25 + Dense retrieval + Reciprocal Rank Fusion (RRF)

## 1. Purpose

Introduce a reproducible benchmark suite that evaluates Synanton's retrieval quality against three public, judged datasets:

1. **Mr. TyDi Russian** — Russian-language monolingual retrieval.
2. **BEIR SciFact** — scientific-claim retrieval, where the system must find evidence relevant to a claim.
3. **BEIR NFCorpus** — biomedical/nutrition information retrieval using natural-language queries.

The benchmark should answer:

- Does Dense retrieval improve over BM25?
- Does RRF improve ranking quality over either individual retrieval leg?
- How do results differ by language and domain?
- What is the quality/latency/throughput trade-off?
- Are regressions detected when embeddings, analyzers, chunking, candidate limits, or fusion parameters change?

This is an **information-retrieval benchmark**, not a generative question-answering benchmark. The datasets provide queries, document collections, and relevance judgments (qrels); they do not consistently provide a single canonical natural-language answer. The initial scope evaluates retrieval of relevant evidence. Answer generation, answer correctness, and citation faithfulness are explicitly out of scope.

## 2. Dataset selection

| Dataset | Language / domain | Evaluation corpus | Query set | Why it is included |
| --- | --- | ---: | ---: | --- |
| Mr. TyDi Russian | Russian, Wikipedia-based | About 9.6M passages | 995 test queries | Tests Russian lexical analysis, morphology, and multilingual dense retrieval at meaningful corpus scale |
| SciFact (BEIR) | English, scientific fact checking | 5,183 documents | 300 test queries | Tests retrieval of scientific evidence for claims; useful for precise terminology and evidence discovery |
| NFCorpus (BEIR) | English, biomedical / nutrition | About 3.6K documents | 323 test queries | Tests natural-language domain queries and multiple relevant documents per query |

Counts above describe the commonly distributed benchmark versions and should be recorded from the exact downloaded artifacts in each benchmark manifest. The loader must not silently assume that a query count, qrels count, or split is unchanged across mirrors or releases.

### 2.1 Mr. TyDi Russian

Use the official Russian **test** topics and qrels for final evaluation, with the corresponding complete Russian document collection. The commonly exposed test split contains 995 queries; the Russian corpus is approximately 9.6 million passages. Mr. TyDi is a monolingual retrieval task: Russian queries retrieve from the Russian collection.

Use the **dev** split for configuration experiments. Keep the test qrels out of tuning and model selection. The training split may be used for model training only if a later, separately scoped experiment explicitly requires it.

Primary references:

- Dataset and license: https://github.com/castorini/mr.tydi
- Russian split details: https://ir-datasets.com/mr-tydi
- Test topics and qrels mirror: https://huggingface.co/datasets/irds/mr-tydi_ru_test

The upstream Mr. TyDi project states Apache-2.0 licensing. Preserve attribution and the upstream citation in reports and dataset metadata.

### 2.2 SciFact

Use the BEIR `scifact` corpus, queries, and the official train/test qrels. SciFact is a **claim-to-evidence retrieval** task, not ordinary user-question answering. A claim is used as the search query; relevant scientific abstracts are the evidence documents.

Use the train/dev material for development only where provided by the selected canonical distribution; use the held-out test qrels for final reporting. The BEIR distribution commonly lists 300 test queries and 5,183 corpus documents.

Primary references:

- BEIR implementation and dataset catalogue: https://github.com/beir-cellar/beir
- Dataset card: https://huggingface.co/datasets/BeIR/scifact
- Dataset/qrels card: https://huggingface.co/datasets/BeIR/scifact-qrels

The Hugging Face BEIR dataset card declares CC-BY-SA-4.0. Verify the license and attribution requirements for the exact upstream artifacts before redistribution or use outside the benchmark environment.

### 2.3 NFCorpus

Use the BEIR `nfcorpus` corpus, queries, and train/dev/test qrels. NFCorpus contains natural-language queries about nutrition and health-related facts and a small collection of biomedical/nutrition documents. It is useful for evaluating domain terminology, semantically phrased queries, and queries with multiple relevant documents.

The commonly distributed test set has 323 queries and the corpus has approximately 3,633 documents. Keep train/dev material for development and the test qrels for final reporting.

Primary references:

- BEIR implementation and dataset catalogue: https://github.com/beir-cellar/beir
- Dataset card: https://huggingface.co/datasets/BeIR/nfcorpus
- Qrels card: https://huggingface.co/datasets/BeIR/nfcorpus-qrels

The Hugging Face BEIR dataset cards declare CC-BY-SA-4.0. Verify attribution and share-alike obligations for the exact upstream artifacts before redistribution or use outside the benchmark environment.

## 3. Benchmark design

### 3.1 Retrieval tracks

Run each query against the same dataset corpus and compare the following tracks:

| Track | Description |
| --- | --- |
| BM25 | Lexical retrieval only, using the production-compatible analyzer/tokenization configuration |
| Dense | Vector retrieval only, using the configured Synanton embedding model and similarity metric |
| RRF | Fusion of the BM25 and Dense ranked candidate lists using the configured RRF implementation |
| Optional reranker | A separately reported reranking stage, added only after the baseline is stable |

For fair comparison:

- Use identical corpus records and stable external document IDs for all tracks.
- Use the same query text for BM25, Dense, and RRF; do not rewrite or translate queries in the primary benchmark.
- Record BM25 analyzer, tokenizer, stemming/lemmatization, stop-word, and language settings.
- Record embedding model identifier, revision, vector dimension, normalization, and similarity metric.
- Record BM25 and Dense candidate depths, RRF constant, deduplication rules, and final result depth.
- Do not change chunking or document granularity between retrieval tracks.
- Report any dataset-specific preprocessing separately; never silently normalize only one retrieval leg.

### 3.2 Document identity and granularity

The benchmark unit is the dataset's own judged document or passage. Preserve its external ID exactly and map it to a deterministic internal Synanton benchmark document ID. Store the external dataset name, split, and external document ID as provenance.

Do not split or merge judged records in the initial benchmark. If the production index requires chunking, introduce a separately named chunked-document experiment with a deterministic parent-document aggregation rule. Do not compare document-level qrels directly to arbitrary chunk IDs.

For BEIR records, preserve both `title` and `text`; use a documented title/body indexing policy. For Mr. TyDi, preserve the supplied passage text and ID. Avoid enrichment with external web content.

### 3.3 Relevance judgments

Treat qrels as the evaluation authority:

- A qrel links a query ID to a corpus document ID and a relevance score.
- Preserve the original score; do not collapse graded labels unless the metric explicitly requires binary relevance.
- A missing qrel is **unjudged**, not automatically irrelevant.
- Compute metrics using a standard IR evaluator with explicit treatment of unjudged documents.
- Record the evaluator and version in every run.

For SciFact, retrieval relevance means retrieving evidence documents for a scientific claim. It does not establish whether the claim is supported or refuted unless the dataset's separate evidence labels are explicitly consumed by a distinct claim-verification task.

## 4. Metrics and reports

### 4.1 Quality metrics

Report at minimum:

- **nDCG@10** — primary ranking-quality metric; supports graded relevance.
- **MRR@10** — reciprocal rank of the first relevant result.
- **Recall@100** — fraction of judged relevant documents retrieved in the top 100.
- **MAP@100** — mean average precision over the top 100.

Also report nDCG@5 and Recall@10 where useful. State the cutoff and relevance threshold in every report. Do not compare scores across datasets as if they represented the same user population or task.

### 4.2 Performance metrics

For each retrieval track, capture:

- p50, p95, and p99 query latency;
- queries per second at stated concurrency;
- candidate counts and per-leg latency (BM25, Dense, fusion);
- embedding generation throughput and indexing duration;
- index/storage size and approximate memory footprint;
- CPU, memory, and accelerator utilization where available.

Report cold-cache and warm-cache runs separately. Include hardware, deployment topology, corpus size, concurrency, warm-up policy, query order/randomization, and repetitions. Do not present a latency number without its workload and environment.

### 4.3 Required comparisons

Every benchmark report should include:

1. BM25 baseline.
2. Dense baseline.
3. RRF result.
4. Absolute and relative metric deltas between RRF and each baseline.
5. Per-dataset and macro-average results (macro-average is descriptive, not a substitute for per-dataset scores).
6. Quality versus latency comparison.
7. Failed queries, empty results, and query-level regressions.

The primary success criterion is not that RRF always wins. The benchmark must make regressions and dataset-specific trade-offs visible.

## 5. Data ingestion and isolation

Implement dataset loaders as benchmark-only ingestion adapters. Do not add public benchmark documents to the normal Synanton production corpus or mix them with demo/customer data.

Each dataset should be isolated by a dedicated benchmark tenant or equivalent hard namespace. The namespace must isolate:

- source and document identifiers;
- lexical indexes;
- vector indexes;
- caches;
- query histories and result artifacts;
- benchmark qrels and evaluation outputs.

A loader should be repeatable and idempotent. Re-running it with the same dataset release and manifest must not create duplicate documents or embeddings.

Persist a dataset manifest with:

- dataset name and upstream URL;
- upstream release/revision and download timestamp;
- license identifier and attribution;
- file checksums;
- corpus/query/qrels counts;
- split names;
- preprocessing version;
- loader version;
- embedding model and revision, once indexed.

Never store credentials or private dataset access tokens in manifests or logs.

## 6. Proposed Analytics integration

The benchmark is an Analytics workload: it produces versioned, reproducible observations about retrieval quality and performance. It must not become part of the production search request path.

Suggested logical records/tables:

| Record / table | Purpose |
| --- | --- |
| `benchmark_dataset_manifest` | Dataset release, license, checksums, counts, loader version |
| `benchmark_documents` | Dataset-scoped document metadata and text reference |
| `benchmark_queries` | Query ID, text, dataset, split, language |
| `benchmark_qrels` | Original relevance judgments |
| `benchmark_runs` | Run configuration, commit, model/analyzer revisions, environment |
| `benchmark_results` | Ranked document IDs, scores, per-leg ranks and timings |
| `benchmark_metrics` | Per-query and aggregate IR metrics |
| `benchmark_performance` | Latency, throughput, resource measurements |

These are logical datasets, not a mandate to create seven physical Iceberg tables in the first implementation. Start with the smallest schema that supports reproducibility and query-level diagnosis. Iceberg is appropriate for retained benchmark runs and analytical comparisons; the search engine's own indexes remain the retrieval execution layer.

### Engine responsibilities

- **Spark:** preferred for bounded dataset preparation, validation, embedding backfills, and offline metric aggregation where volume justifies distributed execution.
- **Flink:** not required for the first benchmark. Introduce it only if continuous ingestion of benchmark telemetry or streaming evaluation has a concrete use case.
- **Iceberg:** retain versioned manifests, run metadata, query-level results, and aggregate metrics for longitudinal comparison.

The benchmark should work locally on a small sample without requiring a Flink cluster. A full Mr. TyDi run may require more substantial indexing resources and should be a separately scheduled scale experiment.

## 7. Execution phases

### Phase 0 — Dataset contract and reproducibility

- Pin canonical upstream repositories, revisions, splits, and licenses.
- Implement manifest generation and checksum verification.
- Validate query IDs, document IDs, qrels references, duplicate IDs, and split boundaries.
- Confirm the exact counts from downloaded artifacts.
- Produce a dataset inventory report without indexing.

**Exit:** all three datasets can be downloaded or accessed reproducibly, validated, and attributed.

### Phase 1 — Small end-to-end benchmark

- Start with SciFact and NFCorpus because their corpora are small.
- Use Mr. TyDi Russian dev for pipeline validation, then test on the official Russian test split.
- Load a small deterministic Mr. TyDi sample only for smoke tests; never publish sample metrics as official benchmark results.
- Run BM25, Dense, and RRF with identical query/corpus identities.
- Generate nDCG@10, MRR@10, Recall@100, and MAP@100.

**Exit:** reproducible result files and per-query diagnostics exist for all three datasets.

### Phase 2 — Full retrieval evaluation

- Index the complete official corpus for each dataset.
- Run the official held-out test queries and qrels.
- Record the embedding/analyzer/fusion configuration.
- Generate comparable per-dataset reports.

**Exit:** complete, reproducible quality baseline for all three datasets.

### Phase 3 — Performance and regression suite

- Run concurrency and latency experiments.
- Add repeated runs and confidence intervals or paired query-level significance analysis where appropriate.
- Compare against a committed baseline with explicit regression thresholds.
- Publish a versioned report and machine-readable artifacts.

**Exit:** CI or a scheduled benchmark can identify quality and performance regressions without turning noisy measurements into flaky build failures.

## 8. Acceptance criteria

The proposal is implemented when:

1. All three datasets are loaded with exact upstream IDs, splits, and qrels preserved.
2. Dataset versions, licenses, attribution, checksums, and counts are recorded.
3. Benchmark data is isolated from production/customer data and tenant boundaries are tested.
4. BM25, Dense, and RRF can be run against the same corpus and query set.
5. Metrics are calculated by a standard, version-pinned evaluator.
6. Each run records source commit, configuration, model/analyzer revisions, environment, and timestamp.
7. Reports include per-dataset quality, latency, and retrieval-leg diagnostics.
8. A rerun with identical inputs/configuration produces equivalent rankings and metrics, subject to explicitly documented nondeterminism.
9. The official test split is not used for parameter tuning.
10. Documentation clearly distinguishes retrieval evaluation from answer-generation evaluation.

## 9. Risks and safeguards

| Risk | Safeguard |
| --- | --- |
| Test-set leakage through tuning | Use dev for tuning; reserve official test qrels for final reporting |
| Dataset mirrors differ | Pin upstream revision, checksums, and artifact counts |
| License incompatibility | Preserve attribution; review CC-BY-SA-4.0 terms; avoid redistributing raw data until approved |
| Qrels are incomplete | Treat unjudged documents explicitly; use standard evaluation tooling |
| Different document granularity | Preserve original judged unit; keep chunk-level experiments separate |
| Embedding model changes confound comparisons | Version model, revision, preprocessing, and vector settings |
| Benchmark contaminates production data | Dedicated benchmark namespace and isolated indexes/caches |
| Small datasets overstate scale readiness | Use Mr. TyDi for larger retrieval scale; separate synthetic/concurrency load tests |
| Retrieval results are mistaken for answer quality | Keep answer generation and answer faithfulness as a separate future evaluation track |

## 10. Out of scope

- Generating answers with an LLM.
- Evaluating answer correctness, hallucination, or citation faithfulness.
- Training or fine-tuning embedding/reranking models.
- Translating queries or building cross-lingual retrieval.
- Modifying production ranking parameters automatically from benchmark outcomes.
- Treating benchmark corpus size as a substitute for controlled high-concurrency load testing.

## 11. References

- Mr. TyDi repository and license: https://github.com/castorini/mr.tydi
- Mr. TyDi / Russian dataset documentation: https://ir-datasets.com/mr-tydi
- Mr. TyDi Russian test split: https://huggingface.co/datasets/irds/mr-tydi_ru_test
- BEIR repository and dataset catalogue: https://github.com/beir-cellar/beir
- BEIR SciFact dataset: https://huggingface.co/datasets/BeIR/scifact
- BEIR SciFact qrels: https://huggingface.co/datasets/BeIR/scifact-qrels
- BEIR NFCorpus dataset: https://huggingface.co/datasets/BeIR/nfcorpus
- BEIR NFCorpus qrels: https://huggingface.co/datasets/BeIR/nfcorpus-qrels

## Decision requested

Approve the three datasets and the retrieval-only scope as the initial Synanton benchmark. Implementation should begin with dataset manifests, validation, and small end-to-end runs; full Mr. TyDi indexing and performance testing should follow only after the pipeline is reproducible.
