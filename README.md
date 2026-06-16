# ProClean

ProClean is a Python data-cleaning pipeline that detects and repairs dirty tabular data through several complementary stages: automatic tuple labeling, format cleaning, functional-dependency cleaning, and semantic cleaning.

The project is designed for datasets where a clean reference CSV is available for evaluation and guidance. It can use an OpenAI-compatible LLM endpoint for rule generation, judgment, fallback repair, and semantic repair. It can also use local FastText and semantic language models for anomaly scoring and clustering.

## Recommended Python Version

Python **3.11** is recommended.

## Installation

Create and activate a virtual environment, then install dependencies:

```bash
python -m venv .venv
.venv\Scripts\activate  # Windows
pip install -r requirements.txt
```

For Linux or macOS, activate the environment with:

```bash
source .venv/bin/activate
pip install -r requirements.txt
```

## Configuration

The project reads optional configuration from environment variables. You can also pass equivalent values directly to the Python entrypoints.

| Environment Variable | Default | Description |
| --- | --- | --- |
| `PROCLEAN_RESULT_ROOT` | `results` | Root directory for all generated outputs. |
| `PROCLEAN_FASTTEXT_MODEL_PATH` | `models/cc.en.300.bin` | Local FastText model used for semantic similarity and clustering. |
| `PROCLEAN_SEMANTIC_MODEL_PATH` | `models/qwen-0.6B` | Local semantic model used by `SemanticDetector`. |
| `PROCLEAN_LLM_BASE_URL` | Empty | OpenAI-compatible LLM API base URL. |
| `PROCLEAN_LLM_API_KEY` | Empty | LLM API key. |
| `PROCLEAN_LLM_MODEL` | Empty | LLM model name. |

Example Windows PowerShell setup:

```powershell
$env:PROCLEAN_RESULT_ROOT="results"
$env:PROCLEAN_FASTTEXT_MODEL_PATH="models/cc.en.300.bin"
$env:PROCLEAN_SEMANTIC_MODEL_PATH="models/qwen-0.6B"
$env:PROCLEAN_LLM_BASE_URL="https://your-openai-compatible-endpoint/v1"
$env:PROCLEAN_LLM_API_KEY="your-api-key"
$env:PROCLEAN_LLM_MODEL="your-model-name"
```

## Project Entry Points

### Recommended API Entry Point

Use `project_interface.py` as the main project-level interface:

```python
from project_interface import run_proclean

result = run_proclean(
    dataset="beers",
    dirty_path="data/datasets/beers/beers_error-01.csv",
    clean_path="data/datasets/beers/beers_clean.csv",
    result_root="results",
    debug_mode=False,
    llm_base_url="https://your-openai-compatible-endpoint/v1",
    llm_api_key="your-api-key",
    llm_model="your-model-name",
    semantic_model_path="models/qwen-0.6B",
    fasttext_model_path="models/cc.en.300.bin",
)

print(result)
```

This runs the full pipeline:

1. `AutoLabeler`
2. `FormatCleaner`
3. `FDCleaner`
4. `SemanticCleaner`

The returned dictionary includes key output paths, runtime information, and the final cleaned CSV path.

### Phase Scripts

Each phase can also be executed directly when its input assumptions are satisfied:

```bash
python AutoLabeler.py
python FormatCleaner.py
python FDCleaner.py
python SemanticCleaner.py
```

For normal use, prefer `project_interface.run_proclean(...)` because it coordinates the required phase order and shared configuration.

## Pipeline Overview

### 1. Auto Labeling

`AutoLabeler.py` compares dirty and clean CSV files, selects candidate tuples, and asks an LLM to explain selected dirty-to-clean value changes.

Outputs include:

- Labeled tuple data
- Value statistics
- Prompt and response logs

These labeled examples are reused by later phases.

### 2. Format Cleaning

`FormatCleaner.py` detects formatting issues such as inconsistent value patterns, invalid characters, or regex-style constraints. It can ask an LLM to generate simple Python detection and repair functions, then validates them against sampled clean and dirty values.

Outputs include:

- Format-cleaned CSV
- Detection and repair metrics
- Generated rule logs
- Prompt and response files

### 3. Functional Dependency Cleaning

`FDCleaner.py` detects a likely entity identifier and columns that should be functionally dependent on it. For each dependent column, it checks whether values should be consistent within each entity group. When there is no clear majority value, it can ask an LLM to resolve the conflict.

Outputs include:

- FD-cleaned CSV
- FD judgment results
- Per-column repair metrics
- Fallback resolution logs
- Phase summary

### 4. Semantic Cleaning

`SemanticCleaner.py` combines typo detection and semantic anomaly detection.

It includes:

- `TypoDetector`: detects null-like values and likely typos using frequency and edit-distance heuristics.
- `SemanticDetector`: computes semantic perplexity-style losses with a local language model, then verifies suspicious values with an LLM.
- `SemanticCleaner`: clusters similar rows with FastText, sends repair prompts to the LLM, applies extracted repairs, and evaluates the final result.

Outputs include:

- Merged semantic detection results
- Clustered data
- Semantic-cleaned CSV
- Detection and repair evaluations
- Final cleaned dataset copied to the result root

## Typical Output Layout

With `result_root="results"`, outputs are organized by dataset and phase. A typical structure looks like:

```text
results/
  <dataset>/
    phase_label/
    phase_format/
    phase_fd/
    phase_semantic/
    <dataset>_cleaned.csv
    result.txt
    time.txt
```

Each phase writes its own cleaned CSV and evaluation files. The final fully cleaned dataset is usually available as:

```text
results/<dataset>/<dataset>_cleaned.csv
```

## Evaluation

The project evaluates both detection and repair quality when a clean reference CSV is available.

Repair metrics include:

- `wrong_2_right`: dirty values correctly repaired
- `wrong_2_wrong`: dirty values repaired to another wrong value
- `right_2_wrong`: clean values incorrectly changed
- `wrong_not_change`: dirty values left unchanged
- Precision
- Recall
- F1

Detection metrics include:

- `all_need_detect`
- `all_detected`
- `correctly_detect`
- `wrongly_detect`
- `missing_errors`
- Precision
- Recall
- F1

## Notes

- `debug_mode=True` avoids real LLM calls in some phases and returns fixed test responses. Use it only for quick structural checks.
- `debug_mode=False` requires valid `llm_base_url`, `llm_api_key`, and `llm_model`.
- The LLM client expects an OpenAI-compatible API.
- The semantic model and FastText model are expected to exist locally unless you override their paths.
- Do not include comments with non-English text in source files if this project is being maintained for English-only documentation.

## Requirements Summary

Core external dependencies are listed in `requirements.txt`. Standard-library modules such as `os`, `sys`, `json`, `re`, `shutil`, and `datetime` are not listed there.
