# Report — AD Applicability Extraction Pipeline

## Approach
The pipeline extracts applicability rules from AD PDFs using text extraction (`pdfplumber`) followed by LLM-based structuring. Both source ADs (FAA 2025-23-53 and EASA 2025-0254) are text-based PDFs, so raw text can be extracted cleanly without any OCR or vision model involved.

The extracted text is passed to an LLM (accessed via OpenRouter, `google/gemma-4-26b-a4b-it:free`) with a fixed prompt instructing it to return applicability rules conforming to a predefined Pydantic schema (`ADRules`). This schema was extended beyond the example given in the assignment brief to include `excluded_if_modifications` (with a `type` field distinguishing production mods from service bulletins), `required_modifications`, and a free-text `notes` field for nuance that doesn't fit cleanly into structured fields (e.g. EASA AD 2025-0254's "Group 1–4" applicability structure).

A separate, fully deterministic Python function (`evaluate()`) checks a given aircraft configuration against the extracted rules — this function does **not** call the LLM again, ensuring evaluation results are reproducible and auditable, which matters for a regulatory-compliance use case.

## Why text extraction + LLM (not regex, not VLM)
This combination is the most robust and scalable choice for a use case involving hundreds of new ADs per month with widely varying applicability wording, given that the source documents are text-based and therefore don't require the overhead of a VLM.

Text extraction over VLM: both source PDFs are clean, text-based documents — extracting raw text with `pdfplumber` preserves all applicability information without loss. A VLM would add cost and latency without improving accuracy for documents that are already machine-readable as text; it would only become necessary for scanned or image-based ADs.

LLM over pure regex/rule-based parsing: the business context specifically requires processing hundreds of new ADs monthly, each with different wording. FAA ADs structure applicability under a numbered paragraph (e.g. "(c) Applicability"), while this EASA AD defines applicability through a "Groups" structure with cross-references between groups. A regex pattern tuned to one AD's phrasing would likely fail on the next. An LLM generalizes better to unseen phrasing, which is the actual scalability requirement here — at the cost of needing validation, since LLM output isn't guaranteed correct.

## Challenges
The most significant practical challenge was **finding a usable, free LLM API** for the extraction step. I initially tried Google's Gemini API free tier, which returned a `429 RESOURCE_EXHAUSTED` error with a hard `limit: 0` — this turned out to be a region-based restriction on Gemini's free tier rather than a rate-limit-from-usage issue. I switched to OpenRouter using a free-tier model (`google/gemma-4-26b-a4b-it:free`), which resolved the access issue.

On the extraction side, the main risk was the LLM missing or misattributing modification exclusions in EASA AD 2025-0254, since it defines applicability through a nested "Groups" structure (Group 1–4) rather than a flat list, and has two separate exclusion conditions for A320s (mod 24591 OR SB A320-57-1089 Rev 04) versus one for A321s (mod 24977). I validated this by checking the extracted JSON manually against the AD text, and by running it through the 3 verification examples provided in the assignment (expanded to 6 checks: applicability x affected-status per AD) — all passed before running the full 10-aircraft test set.

## Limitations
- The `excluded_if_modifications` list is flat and doesn't explicitly tag which aircraft model family each exclusion applies to. This worked correctly here because mod/SB numbers are unique per model family in these two ADs, but at scale, the schema should be extended with an `applies_to_models` field per exclusion to avoid incorrect cross-matching on ADs where mod numbers might not be as cleanly separated.
- The evaluation logic's modification-matching (`has_exclusion`) relies on substring/number matching between the aircraft's declared modifications and the AD's exclusion text, which is somewhat fragile — a differently-formatted mod string (e.g. "Mod24591" vs "mod 24591 (production)") could fail to match. A more robust approach would normalize modification identifiers into a canonical format before comparison.
- Using a free-tier LLM model (rather than a larger flagship model) is a real accuracy trade-off — I chose it for cost reasons given this is a takehome assignment, but for a production pipeline processing hundreds of ADs, a stronger model or a self-consistency check (e.g. running extraction twice and flagging disagreements for human review) would be safer.
- The pipeline was only tested against 2 ADs with relatively clean, well-structured applicability sections. ADs with more irregular formatting, tables-based applicability, or applicability defined by flight hours/cycles/dates (mentioned as a possibility in the assignment background but not present in either test AD) are not yet handled by the schema or extraction prompt.

## What I'd do differently with more time
- Add a human-in-the-loop review step for any AD where the LLM's confidence is low or where extracted MSN/modification logic is unusually complex.
- Test the pipeline against a wider, more diverse sample of ADs (different authorities, formats, and applicability structures) to find where the current prompt and schema break.
- Extend the schema to properly scope exclusions/requirements to specific aircraft models rather than a flat list.
- Add automated regression tests (beyond the manual verification examples) that run on every prompt/schema change.
