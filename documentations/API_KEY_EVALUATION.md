# Running LEXTREME-style evaluations with only vendor API keys

This repository is designed for **fine-tuning Hugging Face models locally** and not for direct black-box API model evaluation. You can still run a useful Request-for-Proposal (RFP) evaluation with API-only access by building a lightweight adapter harness.

## Why the default pipeline is not API-key-only

- The main entrypoint (`main.py`) orchestrates local experiments and expects model names/checkpoints that can be loaded in local training/evaluation jobs. It does not expose a generic "call external chat/completions API" abstraction.  
- The official workflow in `README.md` is based on local fine-tuning runs (`python main.py ...`) and then score aggregation scripts.

## Practical API-only evaluation strategy

### 1) Decide benchmark scope for API inference

For vendor API evaluations, prioritize **inference-only tasks** first:

- Single-label classification (SLTC)
- Multi-label classification (MLTC)
- Named entity recognition (NER)

Do **not** compare to fine-tuned leaderboard numbers directly unless vendors are also allowed task-specific training.

### 2) Build a vendor adapter interface

Create one thin adapter per vendor that implements a shared function signature, for example:

```python
def predict(example: dict, task_name: str, label_space: list[str]) -> dict:
    """Returns normalized prediction payload for one example."""
```

Each adapter should:
- read API key from environment variable,
- format task-specific prompt,
- call vendor API,
- parse response into canonical schema,
- return confidence (optional) and raw response for auditing.

### 3) Use deterministic prompting protocol

To keep RFP comparisons fair:
- fix temperature (usually `0`),
- fix prompt template per task,
- fix decoding limits,
- disable tool-use / browsing unless explicitly part of your test,
- run multiple seeds only if provider supports true stochastic decoding and you need variance estimates.

### 4) Convert outputs to repository scoring format

This repo already contains score computation utilities under `statistics/`.

- Generate prediction files in a consistent intermediate format.
- Convert them into the same label representation expected by metric scripts.
- Reuse existing aggregation scripts for dataset/language/final score reporting.

### 5) Add robustness controls for API benchmarking

- Retries with exponential backoff for rate limits.
- Request logging with request-id/vendor model version.
- Cost and latency tracking per dataset.
- Hard validation for malformed outputs (invalid labels, malformed BIO tags, etc.).

### 6) Report apples-to-apples in the RFP

Split results into two tracks:

1. **API-only zero/few-shot track** (black-box, no fine-tuning)
2. **Fine-tuned track** (if vendor supports supervised adaptation)

This avoids unfairly penalizing API-only systems against models explicitly fine-tuned on the benchmark.

## Minimal execution blueprint

1. Select 3-5 representative LEXTREME datasets across task types/languages.
2. Build prompt templates + output schema validators.
3. Run batched API inference and persist raw responses + parsed predictions.
4. Convert predictions into metric input files.
5. Run repository score scripts to compute per-task and aggregate metrics.
6. Produce a final RFP table including quality, latency, cost, and failure rate.

## Recommendation

If your immediate goal is vendor comparison with only API keys, treat this benchmark as a **dataset + scoring backend**, and implement a separate API-inference harness that feeds into the existing metric/aggregation scripts.
