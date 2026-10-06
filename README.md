# Data & Open-Source Information

This directory contains training code, deployment scripts, baseline evaluation, and reference implementations for testing closed-source APIs (Gemini and GPT). It supports reproducing the pipeline: **train classifier & generation models → deploy with classifier + PDD → evaluate on ATT / PATT / Normal / GSM8K**.

---

## 1. Training (`train/`)

Training reference code lives under **`train/`**. All scripts assume a base model (e.g. Qwen2.5-7B-Instruct) and produce LoRA adapters.

| Path | Description |
|------|-------------|
| **`train/cls_model/train_cls_lora.py`** | **Classifier LoRA**: fine-tunes the base model to classify user input into 6 categories (1–5 attack types, 6 = normal). Input: prompt; output: single digit 1–6. Training data: JSONL with `prompt` and `label`. |
| **`train/lora/train_lora.py`** | **Generation LoRA**: safety-oriented SFT for the base model. Supports configurable LoRA rank, EOS-based early stopping, and recursive JSONL data loading. |
| **`train/cot/train_lora_cot.py`** | **CoT LoRA**: same as generation LoRA but with chain-of-thought style (e.g. `<think>...</think>`) for reasoning. |

You need to set base model path, data paths, and output adapter paths inside each script (or via args if supported). The classifier’s system prompt in `train_cls_lora.py` must match the one used in deployment (`deployment/deploy_filter.py` and related scripts).

---

## 2. Deployment (`deployment/`)

Deployment scripts show **how the classifier is embedded into the full model pipeline** and how to run the service.

### 2.1 Classifier integration and code flow

- **Two models in one process**:  
  - **Classifier**: base model + classifier LoRA (from `train/cls_model`).  
  - **Generation**: base model + generation LoRA (from `train/lora` or `train/cot`).  
  Both can share the same GPU(s); scripts use a single device per worker (e.g. GPU 0/1 by PID).

- **Per-request flow** (e.g. in `deploy_filter.py` or `deploy.py`):
  1. **Classify**: User text → classifier (with fixed system prompt) → label in `1–6`.
  2. **PDD (Prompt-Dependent Defense)**: Load PDD prefix from `pdd.txt` / `pdd_en.txt` by label (and language). If label is 6 (normal), prefix may be empty.
  3. **Augment**: `augmented_content = pdd_prefix + "\n" + user_content` (or raw user content if no prefix).
  4. **Generate**: Build messages with a strict system prompt + augmented user content; call the **generation model** (base + gen LoRA) to produce the reply.
  5. **Response**: Return the generated text; optionally log request/response and classification label.

So the classifier is **not** inside the generation model; it runs first, and its output (label) selects the PDD prefix that is prepended to the user input before generation.

### 2.2 Main scripts

| Script | Purpose |
|--------|--------|
| **`deploy.py`** | Full pipeline: classifier + PDD prefix + generation (main deployment with defense). |
| **`deploy_filter.py`** | Same idea: classifier → PDD → generation; alternate configuration. |
| **`deploy_raw.py`** | No classifier, no PDD: base (or base+gen LoRA) only, same system prompt. |
| **`deploy_*_llama*.py`** | Variants for LLaMA-family base models. |
| **`deploy_filter_only*.py`** / **`deploy_unfilter_*.py`** | Ablation variants (e.g. only filter, or no filter). |
| **`test_classify.py`** | Test classifier standalone (input text → label). |

Configuration: set `BASE_MODEL_DIR`, `GEN_ADAPTER_PATH`, `CLS_ADAPTER_PATH`, and PDD file paths in the deploy script you use. See **`deployment/README.md`** for environment, API (`/model`), and hardware notes.

---

## 3. Baseline evaluation (`baseline/`)

Baseline defines **how we evaluate models on different data dimensions**: direct attack (ATT), mixed attack (PATT), normal queries, and GSM8K.

### 3.1 Evaluation scripts and data

- **Direct Attack (ATT)**  
  - Scripts: `att/att_api.py`, `att/att_api_cot.py` (optional CoT judge).  
  - Template: `att/att.md`.  
  - Input: e.g. `result/{target_dir}/att.xlsx` (columns: prompt, response).  
  - Output: `result/{target_dir}/att_evaluated.xlsx` with judge fields (e.g. `judge_is_success`, `judge_harmful_leakage`, `judge_specificity_score`).  
  - Success: score ≥ 3 and no harmful leakage.

- **Mixed Attack (PATT)**  
  - Scripts: `matt/patt_api.py`, `matt/patt_api_cot.py`.  
  - Template: `matt/patt.md`.  
  - Input/Output: `result/{target_dir}/patt.xlsx` → `patt_evaluated.xlsx`.  
  - Metrics: success, harmful leakage, specificity, whether normal part was answered.

- **Normal samples**  
  - Scripts: `normal/normal_api.py`, `normal/normal_api_cot.py`.  
  - Template: `normal/normal.md`.  
  - Input/Output: `result/{target_dir}/normal.xlsx` → `normal_evaluated.xlsx`.  
  - Metrics: normal answered, false refusal, timeout/truncation.

- **GSM8K**  
  - Script: `gsm8k/gsm8k_judgs.py`.  
  - Purpose: math reasoning quality on GSM8K (see `gsm8k/gsm8k.md`).

Evaluation is done by calling an external judge API (e.g. DeepSeek); configure API key in `.env`. See **`baseline/README.md`** for dependencies and usage.

### 3.2 Aggregation and figures

- **`result/baseline.py`**: Reads `att_evaluated.xlsx`, `patt_evaluated.xlsx`, `normal_evaluated.xlsx` for a given `result/{target_dir}`, computes metrics (e.g. DSR, leakage rate, BRR, FAR, specificity, over-refusal), writes `metrics.jsonl` and generates report figures (e.g. `evaluation_report.png`).
- **`result/data_baseline.py`**: Data-level baseline analysis.

So: **run ATT/PATT/Normal (and optionally GSM8K) evaluation scripts → get *_evaluated.xlsx → run baseline.py for metrics and plots.**

---

## 4. Closed-source API test references (`gemini_test/`, `gpt_test/`)

These folders provide **reference implementations for evaluating closed-source APIs** (Gemini and GPT) on the same tasks and data formats used for our open-source pipeline.

- **`gemini_test/`**: Subdirs per model (e.g. `gemini-2.5-flash`, `gemini-2.5-pro`, `gemini-3-pro-preview`). Each contains:
  - `test.py`: Batch send prompts (e.g. from JSONL) to the Gemini API, collect responses, write to Excel.
  - `test_ous.py`: Variant (e.g. different endpoint or “ours” setting).
  - Shared `pdd.txt` / `pdd_en.txt` if needed for prompts.

- **`gpt_test/`**: Same idea for GPT models (e.g. `gpt-4o-mini`, `gpt-5-mini`, `gpt5.2`):
  - `test.py`, `test_ous.py`: Call GPT API, same input/output pattern (JSONL → Excel).

Usage: point scripts at your API keys (e.g. `.env`), set input JSONL and output Excel paths, then run. The resulting Excel files can be fed into the **baseline** evaluation (ATT/PATT/Normal) so that Gemini and GPT are evaluated with the same metrics and templates as our deployed model.

---

## 5. Quick reference

| Goal | Where |
|------|--------|
| Train classifier | `train/cls_model/train_cls_lora.py` |
| Train generation (or CoT) | `train/lora/train_lora.py`, `train/cot/train_lora_cot.py` |
| Deploy with classifier + PDD | `deployment/deploy.py` or `deployment/deploy_filter.py` |
| Understand classifier + PDD flow | `deployment/README.md` + `deploy_filter.py` (classify → PDD → generate) |
| Evaluate ATT / PATT / Normal | `baseline/att/`, `baseline/matt/`, `baseline/normal/` + `baseline/README.md` |
| Aggregate metrics and plots | `baseline/result/baseline.py` |
| Test Gemini / GPT (closed-source) | `gemini_test/<model>/`, `gpt_test/<model>/` |

All paths and model names in scripts are examples; adjust base model dirs, adapter paths, and result directories to your environment.
