# Report: AD Applicability Extraction Pipeline

## Approach
I used a combination of text extraction and an LLM. Both source PDFs (FAA 2025-23-53 and EASA 2025-0254) are text-based documents, so pdfplumber could pull the text cleanly without needing OCR or any vision model at all.

Once I had the raw text, I sent it to an LLM with a prompt asking it to return the applicability rules in a fixed JSON schema, defined as a Pydantic model (ADRules). I ended up extending the schema beyond the example given in the brief, because that example was too simple for the EASA AD. That one has exclusions based on modification AND on service bulletin revision, and they're different depending on the aircraft model. So I added fields for excluded_if_modifications (tagged as either production_mod or service_bulletin), required_modifications, and a free text notes field to capture anything that didn't fit cleanly into the structured fields, like the "Group 1 to 4" applicability structure that EASA uses.

The evaluation part is kept completely separate from the LLM. The evaluate() function is plain Python, it doesn't call the LLM again at all. The reasoning here is simple: this is a regulatory compliance use case, so the evaluation results need to be reproducible and auditable. You can't have the answer change depending on what mood the model is in that day.

## Why text extraction plus LLM, not regex, not VLM
This felt like the most sensible choice for a business that processes hundreds of new ADs a month, each written a bit differently, especially since the documents themselves are text based and don't need the extra overhead of a VLM.

I skipped VLM because the documents are already clean and text based. Sending page images to a vision model would just add cost and latency without improving accuracy. VLM only really makes sense once you're dealing with scanned or image based documents.

I skipped pure regex too, because even these two ADs alone are structured completely differently. FAA writes applicability under a numbered paragraph, something like "(c) Applicability," which is fairly straightforward. EASA on the other hand defines it through a "Groups" system with cross references between groups. If I hardcoded a regex for one of these formats, it would almost certainly break on the next AD that comes in with yet another format. An LLM generalizes much better to wording it hasn't seen before. The tradeoff is that you now have to validate its output, since it can misread things or hallucinate.

## Challenges
Honestly, the hardest part was finding a free LLM API that actually worked. I tried Gemini first and got hit with a 429 RESOURCE_EXHAUSTED error showing a hard limit of 0. Turns out that wasn't a usage based rate limit, it was a regional restriction on Gemini's free tier, and my region happened to fall under that restriction. I ended up switching to OpenRouter using a free model (google/gemma-4-26b-a4b-it:free), and that worked fine.

On the extraction side, the part most likely to go wrong was the EASA AD, since its applicability is defined through a nested "Group" structure (Group 1 through 4), and it has two separate exclusion conditions: mod 24591 or SB A320-57-1089 Rev 04 for the A320s, and mod 24977 for the A321s. I checked the extracted JSON manually against the original AD text, then ran it through 6 verification checks based on the 3 examples given in the assignment (3 examples times 2 ADs). All 6 passed before I moved on to the full 10 aircraft test set.

## Limitations
- The excluded_if_modifications list is flat. It doesn't explicitly tag which aircraft model family each exclusion belongs to. That worked out fine here because the mod and SB numbers happen to be unique per model family, but at a larger scale it would be safer to add an applies_to_models field to each exclusion, just to avoid any accidental cross matching.
- The modification matching logic (has_exclusion) relies on substring and number matching between what the aircraft declares and what the AD's exclusion text says. That's a bit fragile. A differently formatted string, like "Mod24591" instead of "mod 24591 (production)," could fail to match. Normalizing modification identifiers into one canonical format before comparing them would make this more robust.
- I used a free tier LLM model instead of a flagship one. That's a real accuracy tradeoff, and I picked it mainly because this is a takehome assignment and needed to be free. For an actual production pipeline handling hundreds of ADs, a stronger model or a self consistency check (running extraction twice and flagging any disagreement for human review) would be a safer bet.
- This pipeline was only tested against 2 ADs, both with reasonably clean applicability sections. ADs with messier formatting, table based applicability, or rules based on flight hours, cycles, or manufacturing dates (mentioned as a possibility in the assignment background but not present in either test AD) aren't necessarily handled well by the current schema or prompt.

## What I'd do differently with more time
- Add a human review step for any AD where the model seems uncertain or where the MSN and modification logic gets unusually complex.
- Test the pipeline against a wider and more varied sample of ADs, different authorities, different formats, different ways of structuring applicability, just to find where the current prompt and schema actually break.
- Extend the schema so exclusions and requirements are properly scoped to specific aircraft models instead of sitting in one flat list.
- Add automated regression tests that run whenever the prompt or schema changes, instead of relying on manual verification every time.
