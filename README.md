# Receipt Intelligence

Multimodal **Extractor → Validator → Router** pipeline for automatically extracting and validating structured data from receipt photos, using exclusively local models via [Ollama](https://ollama.com/) (no API key, no token costs).

## Architecture

```
📷 Receipt image
       │
       ▼
┌─────────────────────┐
│  AGENT 1 (Vision)   │  gemma4 → reads the receipt, extracts structured JSON
│     Extractor       │
└─────────┬───────────┘
           │
           ▼
┌─────────────────────┐
│  AGENT 2 (Python)   │  sum of items vs declared total → math_error flag
│    Math Checker     │  (no LLM, 100% deterministic)
└─────────┬───────────┘
           │
           ▼
┌─────────────────────┐
│  AGENT 3 (Reasoning)│  deepseek-r1 → checks consistency, adds comment
│      Reviewer       │
└─────────┬───────────┘
           │
           ▼
    RunnableBranch
     /           \
  ✅ OK      ⚠️ NEEDS_REVIEW
  (save)    (requires review)
```

## Design Pattern

The project implements a variant of the **Reflection/Validator** pattern with conditional routing:

| Agent | Type | Model | Responsibility |
|-------|------|-------|----------------|
| Extractor | Vision LLM | gemma4 | Perception: image → structured JSON |
| Math Checker | Pure Python | — | Deterministic validation: Σ items = total? |
| Reviewer | Reasoning LLM | deepseek-r1 | Semantic validation: consistency, OCR errors, comment |
| Router | RunnableBranch | — | Routing: OK vs NEEDS_REVIEW |

**Key principle**: an LLM is used only when necessary. The Math Checker is pure Python because arithmetic logic is exactly computable — faster, zero tokens, zero hallucinations.

## Prerequisites

- Python 3.10+
- [Ollama](https://ollama.com/) installed and running

## Installation

```bash
git clone https://github.com/<your-username>/receipt-intelligence.git
cd receipt-intelligence

pip install -r requirements.txt

# Pull Ollama models
ollama pull gemma4:e4b
ollama pull deepseek-r1:8b
```

## Usage

### Single receipt

1. Open `receipt_intelligence.ipynb` in Jupyter
2. Edit `IMAGE_PATH` in the configuration cell
3. Run all cells

### Batch processing

```python
from pathlib import Path
df = process_batch(Path("./sample_receipts"))
df.to_csv("results.csv", index=False)
```

### Output

Each receipt produces:
- **receipt**: validated Pydantic object with all fields
- **status**: `OK` | `NEEDS_REVIEW`
- **math_difference**: difference in € between computed sum and declared total
- **review_comment**: comment from the LLM reviewer

## Project structure

```
receipt-intelligence/
├── receipt_intelligence.ipynb   # Main notebook
├── requirements.txt             # Python dependencies
├── sample_receipts/             # Sample receipts (anonymized)
│   └── receipt_01.jpg
├── output/                      # Batch results (auto-generated)
└── .gitignore
```

## Tech stack

- **[LangChain](https://python.langchain.com/)** (LCEL) — pipeline composition
- **[Ollama](https://ollama.com/)** — local inference
- **[Pydantic v2](https://docs.pydantic.dev/)** — output schema validation
- **[pandas](https://pandas.pydata.org/)** — CSV batch export

## Supported models

The project is configurable for any Ollama model with vision capabilities (Extractor) and any model with reasoning capabilities (Reviewer). The defaults are tested on consumer hardware (M-series Mac, 16GB RAM).

## License

MIT
