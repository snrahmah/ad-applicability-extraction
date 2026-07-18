# Airworthiness Directive (AD) Applicability Extraction Pipeline

## Overview
This project automatically extracts aircraft applicability rules from Airworthiness Directive (AD) PDFs, structures them into a machine-readable JSON format, and evaluates whether specific aircraft configurations are affected by each AD.

Two source documents were used:
- FAA AD 2025-23-53 (MD-11 / MD-10 / DC-10 engine pylon structure)
- EASA AD 2025-0254 (A320 / A321 main landing gear retraction actuator fitting)

## Pipeline Steps
1. **Fetch AD pages** — scrape the EASA AD registry page to locate the direct PDF download link (works generically for any AD page, not hardcoded to these two documents).
2. **Download PDFs** and extract raw text using `pdfplumber`.
3. **Extract structured rules** — the raw AD text is sent to an LLM (via OpenRouter, model: `google/gemma-4-26b-a4b-it:free`) with a fixed prompt, requesting output that conforms to a Pydantic schema (`ADRules`).
4. **Validate** the LLM output against the Pydantic schema, catching malformed JSON.
5. **Evaluate** — a separate, deterministic (non-LLM) Python function checks a given aircraft configuration (model, MSN, list of embodied modifications) against the extracted rules and returns `Affected`, `Not affected`, or `Not applicable`.
6. **Verify** — the pipeline includes 6 verification checks based on the 3 example cases provided in the assignment, all of which pass before running the full 10-aircraft test set.

## How to Run
1. Open `Soji_Al_Siti_Nur_Rahmah.ipynb` in Google Colab.
2. Add your OpenRouter API key as a Colab Secret named `OPENROUTER_API_KEY`, and enable notebook access.
3. Run all cells from top to bottom.
4. Outputs are written to `output/`:
   - `FAA-2025-23-53.json`, `EASA-2025-0254.json` — structured applicability rules
   - `test_results.csv` — evaluation results for the 10 provided aircraft configurations

## Repository Structure
```
submission/
├── Soji_Al_Siti_Nur_Rahmah.ipynb   # full pipeline: extraction, schema, evaluation
├── output/
│   ├── FAA-2025-23-53.json
│   ├── EASA-2025-0254.json
│   └── test_results.csv
├── README.md
└── report.md
```
