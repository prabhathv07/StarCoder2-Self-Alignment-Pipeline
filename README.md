# TypeScript Instruction Data Pipeline — StarCoder2 Self-Alignment

A three-stage pipeline that synthesizes TypeScript coding instruction data from raw open-source code, adapting the SelfOSSInstruct methodology from the StarCoder2-15B paper. Runs on NJIT's Wulver HPC cluster using StarCoder2-3B as the backbone model via vLLM.

![Python](https://img.shields.io/badge/Python-3.10-blue?logo=python&logoColor=white)
![TypeScript](https://img.shields.io/badge/TypeScript-tree--sitter-3178C6?logo=typescript&logoColor=white)
![HuggingFace](https://img.shields.io/badge/HuggingFace-StarCoder2-FFD21E?logo=huggingface&logoColor=black)
![vLLM](https://img.shields.io/badge/vLLM-inference-blueviolet)
![SLURM](https://img.shields.io/badge/HPC-SLURM%20%7C%20Wulver-orange)

---

## What This Does

Large language models trained on raw code learn syntax but not intent — they don't know why code is written the way it is, or how a human would ask for it. Instruction-tuned models are much better at actually helping programmers, but building high-quality instruction datasets by hand is expensive and slow.

This project automates that process for TypeScript. Given a corpus of real open-source TypeScript functions, the pipeline uses StarCoder2-3B to generate the instruction data automatically — no human labeling required. The key insight from the original paper is that a code model already understands code well enough to describe what it does, which you can then use to create training pairs that make it even better at following instructions.

The pipeline runs in three stages:

1. **Seed Gathering** — collect and filter real TypeScript functions from open-source repos
2. **Self-OSS-Instruct (S→C→I→R)** — use StarCoder2 to synthesize concept annotations, instructions, and responses
3. **Validation** — filter out low-quality generated pairs

---

## Background

This work follows the approach described in:

> *StarCoder2 and The Stack v2: The Embarrassment of Riches* (BigCode, 2024)

The SelfOSSInstruct pipeline is: **Seed → Concepts → Instruction → Response**. A seed function from real code is passed through a chain of LLM prompts, each building on the previous output, until you have a full instruction-response pair grounded in actual TypeScript patterns.

The original paper ran this at scale on StarCoder2-15B. This project adapts it to StarCoder2-3B and targets TypeScript specifically, using tree-sitter for language-aware parsing and TypeScript's own compiler for type-checking quality filters.

---

## Pipeline Architecture

```
The Stack v2 (TypeScript) — 30,000 files streamed from S3
         │
         ▼
Step 1 — Seed Gathering
  ├── tree-sitter parsing → extract TypeScript functions
  ├── 8,372 raw functions extracted
  ├── Filter: must have return statement → 6,497 functions
  └── Type-check with TypeScript compiler → 5,791 HQ functions (seed dataset)
         │
         ▼
Step 2 — Self-OSS-Instruct  [StarCoder2-3B via vLLM on Wulver GPU]
  ├── S→C: Seed function → TypeScript concept list
  │         5,791 records → 5,791 concept annotations
  ├── C→I: Concepts + seed → natural language instruction
  │         5,791 records → 5,791 instructions
  └── I→R: Instruction → code response
            5,791 records → 448 instruction-response pairs (partial run)
         │
         ▼
Step 3 — Validation
  └── Filter generated pairs for instruction quality and response correctness
         │
         ▼
Final: TypeScript instruction-response dataset for fine-tuning
```

---

## Stage Details

### Stage 1 — Seed Gathering (`Seed_Gathering/`)

The seed dataset comes from `bigcode/the-stack-v2-dedup`, streamed directly from Software Heritage's S3 bucket. 30,000 TypeScript files were downloaded and parsed using tree-sitter with a TypeScript grammar to extract individual function declarations.

Two quality filters were applied before anything reaches the model:

**Return statement filter.** Functions that don't return a value are less useful for instruction generation — you can't ask the model to "write a function that returns X" if the original function doesn't return anything. This filter cut 8,372 functions down to 6,497.

**Type-checking filter.** The remaining functions were run through TypeScript's compiler to catch type errors and malformed snippets. This is important because the-stack-v2 contains code scraped from across the web, including snippets with missing imports, broken syntax, or incorrect types. After type-checking, 5,791 functions remained as the final seed dataset.

Seed functions were saved as both Arrow datasets (checkpointed) and JSONL format for the next stage.

### Stage 2 — Self-OSS-Instruct (`Step2_OSS_Final/`)

This is the core of the pipeline. Three sequential LLM inference passes transform each seed function into an instruction-response pair:

**S→C (Seed → Concepts).** StarCoder2 is prompted to act as a TypeScript concept extraction assistant. Given a raw function, it produces a comma-separated list of TypeScript concepts the function uses — things like `async/await`, `interface`, `type annotations`, `destructuring`, `error handling`. This grounds the subsequent instruction in actual language features rather than letting the model hallucinate arbitrary topics.

**C→I (Concepts → Instruction).** Using the concept list alongside the seed function, the model generates a natural language instruction: a realistic task description that a developer might type into an assistant. The seed function serves as the target answer — so the instruction is constrained to be achievable and relevant to real code.

**I→R (Instruction → Response).** The model generates a TypeScript code response to the instruction. This is the output that would be used to fine-tune a model — the instruction is the question, the response is what the model should learn to produce.

All three passes ran via vLLM on NJIT's Wulver cluster using a single A100 GPU, with `StarCoder2-3B` loaded from the cluster's shared model path. Inference was batched (batch size 32) and written to JSONL files incrementally.

**Run stats:**
- S→C: 5,791 / 5,791 records completed
- C→I: 5,791 / 5,791 records completed
- I→R: 448 / 5,791 records completed (partial — GPU time limit reached)

### Stage 3 — Validation (`Step3_Validation/`)

The validation step filters the generated instruction-response pairs using StarCoder2 as a judge. The model evaluates whether the generated response actually satisfies the instruction, flagging low-confidence pairs for removal. This mirrors the self-consistency validation used in the original paper.

---

## Sample Output

**Seed function (from The Stack v2):**
```typescript
async function getAllUrl() {
    const mongoDbUrl = 'mongodb://' + dbUser + ':' + dbPassword + '@127.0.0.1:27017/';
    let connection = await MongoClient.connect(mongoDbUrl, { useNewUrlParser: true });
    let db = await connection.db(dbName);
    const urls = db.collection('urls');
    const allUrls = await urls.find().toArray();
    await connection.close();
    return allUrls;
}
```

**After S→C — extracted concepts:**
```
async function, await, MongoClient, connect, db, collection, find, toArray, string concatenation, return
```

**After C→I — generated instruction:**
```
Write an async TypeScript function that connects to a MongoDB database using
MongoClient, retrieves all documents from a 'urls' collection, and returns them
as an array. The function should properly close the connection after retrieving the data.
```

**After I→R — generated response:**
```typescript
import { MongoClient } from 'mongodb';

async function getAllUrls(dbUser: string, dbPassword: string, dbName: string): Promise<object[]> {
    const mongoDbUrl = `mongodb://${dbUser}:${dbPassword}@127.0.0.1:27017/`;
    const connection = await MongoClient.connect(mongoDbUrl, { useNewUrlParser: true });
    try {
        const db = connection.db(dbName);
        const urls = db.collection('urls');
        return await urls.find().toArray();
    } finally {
        await connection.close();
    }
}
```

Sample output files are in `sample_outputs/`.

---

## Dataset Summary

| Stage | Output | Records |
|---|---|---|
| Raw extraction | TypeScript functions from The Stack v2 | 8,372 |
| After return filter | Functions with return statements | 6,497 |
| After type-check | Seed dataset (HQ functions) | 5,791 |
| S→C output | Functions + concept annotations | 5,791 |
| C→I output | Functions + concepts + instructions | 5,791 |
| I→R output | Instruction-response pairs | 448 (partial) |

---

## Compute Environment

Ran on **NJIT Wulver HPC cluster**:

| Resource | Config |
|---|---|
| Cluster | NJIT Wulver (SLURM) |
| GPU | 1× A100 |
| Memory | 64 GB RAM |
| CPUs | 16 |
| Python | 3.10.8 (foss/2022b) |
| Model | StarCoder2-3B (`bigcode/starcoder2-3b`) |
| Inference | vLLM |
| QOS | low (`--qos=low --account=phan`) |

Seed gathering ran on Google Colab (T4 GPU) for the S3 streaming + tree-sitter parsing steps.

---

## Setup

**Environment:**
```bash
# On Wulver — request GPU node
srun -p gpu -n 1 --ntasks-per-node=2 --qos=low --account=phan \
     --mem-per-cpu=64G --gres=gpu:1 --time=72:00:00 --pty bash

module load foss/2022b Python/3.10.8

# Create and activate your environment
python -m venv ~/.venv/starcoder_align
source ~/.venv/starcoder_align/bin/activate

pip install -r requirements.txt
```

**Authentication:**
```bash
# Set your Hugging Face token as an environment variable
export HF_TOKEN="your_token_here"
```

The notebooks use `os.environ.get("HF_TOKEN")` — never hardcode tokens in notebooks.

**vLLM server (for inference):**
```bash
export OPENAI_API_KEY="EMPTY"
export OPENAI_BASE_URL="http://0.0.0.0:8000/v1/"
```

---

## Repo Structure

```
.
├── Seed_Gathering/
│   └── Seed_Gathering_Final/
│       └── Seed_Gathering_Final.ipynb   # Step 1: extract + filter TypeScript functions
├── Step2_OSS_Final/
│   └── 2_SelfOSSInstruct&3_Validation.ipynb  # Step 2: S→C→I→R pipeline
├── Step3_Validation/
│   └── Step3_Validation.ipynb           # Step 3: instruction quality validation
├── sample_outputs/
│   ├── seed_sample.jsonl                # 20 seed functions
│   ├── ts_s2c_sample.jsonl              # 20 S→C outputs (concepts)
│   ├── ts_c2i_sample.jsonl              # 20 C→I outputs (instructions)
│   └── ts_i2r_output.jsonl             # 448 I→R outputs (instruction-response pairs)
└── README.md
```

---

## Key Dependencies

| Package | Purpose |
|---|---|
| `vllm` | Fast LLM inference for S→C→I→R generation |
| `tree-sitter==0.20.1` | TypeScript function parsing and AST queries |
| `datasets` | HuggingFace dataset loading and Arrow format |
| `huggingface_hub` | Model access and authentication |
| `boto3` / `smart_open` | Streaming TypeScript files from Software Heritage S3 |
| `torch` | PyTorch backend for vLLM |

---

## What I'd Do Next

- **Complete I→R generation** — the current run stopped at 448 records due to GPU time limits; a full run would produce 5,791 instruction-response pairs
- **Run full validation pass** — filter the 448 pairs with the validation step to get a clean subset
- **Scale to StarCoder2-15B** — the 3B model generates reasonable but sometimes shallow instructions; the 15B model produces significantly better concept extraction and instruction quality
- **Push dataset to HuggingFace Hub** — publish the final instruction dataset for community use and reproducibility
- **Fine-tune on the output** — use the validated instruction-response pairs to fine-tune a smaller model and benchmark on HumanEval-TypeScript
- **Extend to other languages** — the pipeline is language-agnostic beyond the tree-sitter grammar; Python and Go would be natural next targets

---

## Reference

BigCode Team. *StarCoder2 and The Stack v2: The Embarrassment of Riches.* 2024.
GitHub: [bigcode-project/starcoder2-self-align](https://github.com/bigcode-project/starcoder2-self-align)

---

## Author

**Prabhath Vinay Vipparthi**
MS Data Science — New Jersey Institute of Technology (May 2026)
Course: DS677
[GitHub](https://github.com/prabhathv07) · [LinkedIn](https://linkedin.com/in/prabhath-vinay-vipparthi)
