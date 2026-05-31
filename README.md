# SentinelRAG: Synthetic Sentinel Knowledge for RAG Database Copyright Protection

This repo contains the code and data for the paper: SentinelRAG: Synthetic Sentinel Knowledge for RAG Database Copyright Protection

<p align="center">
  <img src="fig/sentinelrag.png" alt="SentinelRAG overview" width="95%">
</p>

## Installation

```bash
cd sentinelrag
uv venv
source .venv/bin/activate
uv pip install -e ".[dev]"
```

Create a local `models/` directory before running LLM-backed commands. Each
model preset must be a JSON file in `models/`. You can copy templates from `models.example/`,

```bash
cp -R models.example models
```

Then fill in your model endpoint and API key.

## Main Workflow

1. Download or prepare a BEIR-style dataset:

```bash
sentinelrag-download-beir nfcorpus
```

2. Build the ChromaDB retrieval collection:

```bash
sentinelrag-build-chroma \
  --eval_dataset nfcorpus \
  --eval_model_code contriever \
  --score_function cosine
```

3. Generate the KO pool:

```bash
sentinelrag-generate-ko-pool \
  --eval_dataset nfcorpus \
  --target_ko_count 50 \
  --num_examples 10 \
  --ko-generation-llm gpt-5-mini \
  --abstract-llm gpt-5-nano
```

4. Generate watermark passages and verification questions:

```bash
sentinelrag-inject-watermark \
  --ko_pool_path output/ko_pools/<preset>/<run>/ko_pool.json \
  --secret_key mykey \
  --eval_dataset nfcorpus \
  --eval_model_code contriever \
  --num_select_kos 50 \
  --llm gpt-5-nano
```

5. Detect the watermark:

```bash
sentinelrag-detect-watermark \
  --eval_dataset nfcorpus \
  --num_select_kos 50 \
  --eval_model_code contriever \
  --rllm gpt-5-mini \
  --dllm gemini-3.1-flash-lite
```

During detection, SentinelRAG checks the target ChromaDB collection for existing
watermark documents, removes any leftovers from prior runs, injects the
watermark passages from the selected `injection_result.json`, runs detection,
and deletes the injected watermark documents after detection finishes.

6. Evaluate interference:

```bash
sentinelrag-eval-interference \
  --eval_dataset nfcorpus \
  --num_select_kos 50 \
  --eval_model_code contriever \
  --num_questions 500 \
  --llm gpt-5-mini \
  --rllm gpt-5-mini
```

Interference evaluation measures behavior changes on normal dataset questions
than asking watermark-targeted questions. It first checks and cleans the
ChromaDB collection, samples main-task questions from the dataset, and retrieves
documents from the clean collection. It then injects the watermark passages from
`injection_result.json`, retrieves again on the watermarked collection, and
compares the top-k document IDs to compute retrieval interference. For answer
interference, it skips answer generation when retrieval is unchanged and no
watermark appears; otherwise it generates clean and watermarked RAG answers and
uses the evaluation LLM to judge whether the two answers are semantically
equivalent. After saving `clean_runs.json`, `watermarked_runs.json`, and
`interference_results.json`, it removes the injected watermark documents.

## Utility Commands

- `sentinelrag-download-beir`: download BEIR datasets.
- `sentinelrag-download-hf`: download Hugging Face datasets to disk.
- `sentinelrag-generate-embeddings`: generate Parquet embedding shards.
- `sentinelrag-build-chroma`: build a ChromaDB collection directly from a dataset.
- `sentinelrag-load-chroma`: load Parquet embedding shards into ChromaDB.

## Repository Layout

```text
sentinelrag/
  src/sentinelrag/core/       # KO pool, injection, detection, interference
  src/sentinelrag/rag/        # ChromaDB vector store and RAG visitor
  src/sentinelrag/cli/        # Paper workflow and utility entry points
  src/sentinelrag/utils/      # datasets, embeddings, model registry, IO, stats
  models.example/             # example OpenAI-compatible model preset JSON files
  docs/                       # migrated workflow notes
```

Generated datasets, embeddings, ChromaDB stores, and experiment outputs are
ignored by default.
