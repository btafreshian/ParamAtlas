# audit_vllm_cluster

All-in-one, **zero-knob** inspector to **map the internal structure** of LLMs/VLMs and MoE models.  
Works directly from a **Hugging Face model id/path** (no server needed) or can **auto-discover** the model from a running **vLLM** endpoint. Includes a **FAST mode** that builds the full module skeleton and per-module parameter counts **without loading any weights**.

---

## ✨ Features

- **FAST mode (no weights):** builds the module tree & counts from `config.json` using empty weights (super quick, no VRAM moves).
- **VLM-friendly:** tries `AutoModel` first (e.g., Qwen-VL, LLaVA-style), falls back to `AutoModelForCausalLM` when needed.
- **MoE awareness:** heuristically flags **routers** & **experts** and groups them by the nearest MoE block.
- **Safety checks:**
  - Writes **CSV param totals** and (in full mode) compares them to the model’s true parameter count; stamps **INTEGRITY STATUS: PASS/FAIL**.
  - Generates a `validation_report.txt` with **MoE per-group counts** and warnings for incomplete groups.
- **Outputs you can diff & script:** `skeleton.txt`, `modules.csv`, `routers.csv`, `experts.csv`, `summary.txt`, `validation_report.txt` (+ optional `hook_log.txt`).

---

## 🚀 Quickstart

```bash
# 0) Python deps
pip install -U torch "transformers>=4.42" accelerate requests

# 1) FAST mode (recommended first): no weights, instant structure
python audit_vllm_cluster.py --model <org/model> --fast

# 2) (Optional) Full mode: load weights, enumerate real modules
python audit_vllm_cluster.py --model <org/model> --no-hooks
```

### vLLM auto-discovery (optional)

If you have a vLLM server and want the tool to discover the model id:

```bash
# FAST mode via endpoint discovery
python audit_vllm_cluster.py --endpoint http://localhost:8000 --fast
```

> `--model` does **not** require vLLM. vLLM is only for auto-discovering the served model id.

---

## 📦 What you get (per run)

Each run writes a timestamped folder under `./reports/…` containing:

| File | What it is |
|---|---|
| `summary.txt` | Totals, counts, **INTEGRITY STATUS: PASS/FAIL**. |
| `skeleton.txt` | Indented module tree (dotted names + class). |
| `modules.csv` | One row per module: name, class, parent, depth, **param/buffer counts**, router/expert flags, **moe_group**. |
| `routers.csv` / `experts.csv` | Subsets flagged by the MoE heuristic. |
| `validation_report.txt` | MoE per-group stats (experts/routers/params), warnings, and (full mode) model vs CSV totals. |
| `hook_log.txt` | (Full mode + hooks only) one-shot shapes/stats from first router/expert. |
| `endpoint.json` | Only when `--endpoint` is used; records `/v1/models` payload. |

---

## 🧠 When to use which mode

- **Use FAST** when you need **structure now** and don’t want to load huge weights. It requires only `config.json` (+ small remote code files if `trust_remote_code`).
- **Use FULL** when you want enumeration based on **real instantiated modules** (and optional 1-shot hooks for CausalLMs). For VLMs, prefer `--no-hooks`.

---

## ✅ Integrity & Validation

The tool writes guardrails so you can trust the outputs:

- **CSV totals:** Sums the per-module parameter counts written in `modules.csv`.
- **Full-mode delta:** Compares the CSV total to the **actual model parameter total** and records the delta.
- **MoE consistency:** For each detected MoE container, reports the **number of routers/experts** and flags groups missing a router or experts.
- **PASS/FAIL stamp:** Added to `summary.txt`. Reasons are listed when FAIL.

---

## 🔧 Examples

```bash
# Mixtral (MoE LLM), structure-only
python audit_vllm_cluster.py --model mistralai/Mixtral-8x7B-v0.1 --fast

# Qwen-VL (VLM), structure-only (quick MoE/vision+text layout)
python audit_vllm_cluster.py --model Qwen/Qwen3-VL-30B-A3B-Instruct --fast

# Same model, full load (weights), but skip hooks (VLMs)
python audit_vllm_cluster.py --model Qwen/Qwen3-VL-30B-A3B-Instruct --no-hooks

# Discover the model from vLLM, then FAST inspect
python audit_vllm_cluster.py --endpoint http://localhost:8000 --fast
```

---

## 🔒 Deterministic runs (highly recommended)

To keep outputs stable across runs:

```bash
# Pin toolchain versions
pip install "transformers==4.44.2" "accelerate==0.34.2" torch==<your_version>

# Pin model revision (no drifting)
python audit_vllm_cluster.py --model org/model@<commit_sha> --fast

# After first successful online run, force offline for re-runs
export TRANSFORMERS_OFFLINE=1
export HF_HUB_OFFLINE=1
python audit_vllm_cluster.py --model org/model@<commit_sha> --fast
```

---

## 🧩 Supported models (typical)

- **LLMs:** Llama/Mistral/Gemma/Falcon/MPT/OPT/GPT-Neo/NeoX/Jamba/OLMo/DBRX/Yi/Phi/GLM/DeepSeek…  
- **MoE LLMs:** Mixtral, Ministral-MoE, Qwen-Moe/Qwen3-Moe, Granite-MoE, GLM-MoE…  
- **VLMs (use `AutoModel` path):** Qwen-VL/Qwen3-VL, LLaVA/Next-style, InternVL, MiniCPM-V, mPLUG-Owl, Kosmos-2, Emu…  
- Anything that provides a valid **`config.json`** and a `transformers` **`AutoModel*`** class (possibly via `trust_remote_code=True`) will work in **FAST** mode; most will also work in full mode.

**Won’t work**: GGUF-only, TensorRT-LLM engines, ONNX-only repos without a Transformers `config.json`.

---

## 🛠️ CLI

```
usage: audit_vllm_cluster.py [--endpoint URL] [--model HF_ID_OR_PATH]
                             [--fast] [--no-hooks] [--outdir DIR]

optional arguments:
  --endpoint URL   vLLM/OpenAI-compatible base URL (uses /v1/models)
  --model ID|PATH  HF model id or local path (skips endpoint discovery)
  --fast           Build skeleton from config (no weights)
  --no-hooks       Disable tiny one-shot hooks (full mode only)
  --outdir DIR     Output directory (default: reports)
```

---

## 🔍 Reading the outputs quickly

```bash
# Largest modules by parameter count
python - <<'PY'
import csv; rows=[]
with open('reports/<run>/modules.csv') as f:
    r=csv.DictReader(f)
    for x in r: x['n_params']=int(x['n_params']); rows.append(x)
rows.sort(key=lambda x:x['n_params'], reverse=True)
for x in rows[:25]:
    print(f"{x['n_params']:>12,}  {x['name']}  ({x['class']})")
PY

# MoE groups by total params, experts, routers
python - <<'PY'
import csv, collections
g=collections.defaultdict(lambda: {'experts':0,'routers':0,'params':0})
with open('reports/<run>/modules.csv') as f:
    r=csv.DictReader(f)
    for x in r:
        mg=x['moe_group'] or '(none)'
        g[mg]['params']+=int(x['n_params'])
        if x['is_expert']=='1': g[mg]['experts']+=1
        if x['is_router']=='1': g[mg]['routers']+=1
for k,v in sorted(g.items(), key=lambda kv: kv[1]['params'], reverse=True)[:20]:
    print(f"{k:60s}  params={v['params']:>12,}  experts={v['experts']:>4}  routers={v['routers']:>3}")
PY
```

---

## 🤝 Contributing

PRs welcome! Ideas:
- Family-specific MoE detectors (beyond heuristics).
- Per-layer param histograms & pruning-candidate suggestions.
- Optional JSON outputs for downstream tooling.
