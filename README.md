# AI-Extracted Medical Tender Data — Accuracy Audit

An audit of how accurately an AI model extracts structured, lot-level data from EU public procurement notices for medical products, measured against a hand-built ground truth.

**Tools:** Google Sheets (REGEXREPLACE, REGEXEXTRACT, COUNTIF/COUNTIFS, EXACT), EU TED portal, Gemini Flash-Lite
**Workbook tabs:** `data_dictionary`, `sources`, `ground_truth`, `ai_run1`, `ai_run2`, comparison tabs, `errors`, `summary`

---

## Why this matters

Healthcare procurement intelligence companies turn thousands of public tender notices into structured data so that medical-device and pharma suppliers can find bidding opportunities and size markets. AI agents increasingly do the first extraction pass. This project measures where such an extraction is reliable, where it fails, and what errors exist in the source notices themselves.

## Method

**1. Data standard.** A 15-field data dictionary defines every field (procedure ID, notice ID, lot ID, buyer, country, CPV code, lot title, quantity, estimated value, currency, publication date, submission deadline, number of lots, framework agreement, comments) with a definition, format and edge-case rule. Data is captured at lot level: one row per lot.

**2. Sample.** Notices were retrieved from TED on 2026-10-06 with these filters: main classification CPV 33 (medical equipment, pharmaceuticals and personal care products), contract notices (standard regime), nature of contract Supplies, buyer country Ireland or Malta (English-language notices). Search URLs are recorded in `sources`.

- Inclusion rule: main CPV begins with 33 and the purchase is a medical device or pharmaceutical.
- 22 notices were screened; 5 were excluded on verification (main CPV not 33; two of these were versions of the same procedure).
- 8 notices were selected (14 lot rows): 6 Malta, 2 Ireland; 3 buyers; 4 multi-lot and 4 single-lot; 2 change notices.
- Selection was purposive: multi-lot notices were deliberately included as harder cases.

**3. Ground truth.** Captured by hand from each notice. Raw source text (European number format, DD/MM/YYYY dates, time-zone labels) was converted to the standard formats with regex formulas, keeping a raw copy of every converted value.

**4. AI extraction.** Each notice was given to Gemini Flash-Lite in a fresh temporary chat (no memory) with one fixed prompt containing the field rules (`prompt.txt`). Every notice was run twice (`ai_run1`, `ai_run2`) to test consistency.

**5. Comparison.** Every cell was compared with an exact, case-sensitive match (`EXACT`). Deadlines were also compared after removing seconds, to separate format errors from wrong values. Where the AI and ground truth disagreed, the source notice was re-checked to decide the correct value.

## Results

| Metric | Run 1 | Run 2 |
|---|---|---|
| Exact-match accuracy (196 cells) | 92.9% | 93.4% |
| Value accuracy (format differences ignored) | 100% | 100% |
| Deadline format followed | 0 of 14 | 1 of 14 |
| Fabricated values (quantity) | 0 | 0 |

- **Consistency between runs:** 195 of 196 cells identical (99.5%).
- **All 27 mismatches were format errors:** the model added seconds to the deadline (`09:30:00` instead of the requested `09:30`).
- **Framework status was read correctly in every lot**, from the lot's structured field rather than the notice title, even where the two contradict each other.
- **No fabrication:** the model left quantity empty in all 28 rows, as instructed, since no notice states one.
- **The model did not "correct" the source:** it kept the UTC offsets exactly as the notices state them, including the incorrect ones below.

## Source data issues found

1. **Lot values repeat the notice total.** In all 4 multi-lot notices, each lot's estimated value equals the notice-level total, even for clearly different products (e.g. 50 mg and 100 mg doses). Summing lot values would overstate procurement value by a factor of up to 4 in this sample.
2. **Wrong UTC offsets after the clock change.** EU clocks went back on 25 October 2026, but 4 Maltese notices (5 lots) with later deadlines still state +02:00 instead of +01:00. A supplier converting these deadlines to another time zone would be one hour out.
3. **Titles contradict structured fields.** All 6 Maltese notices are titled "Framework Contract" while the lot field (section 5.1.15) states "No framework agreement".
4. **Formatting artefacts in the source:** lot titles beginning with a bullet character, and buyer names carrying numeric suffixes (e.g. `_403`, `_1193`).

## Recommendations

- Add automated format validation after AI extraction (e.g. a regex check on every date and deadline). It would have fixed every error in this audit.
- Flag lots whose value equals the notice total before using lot values for market sizing.
- Check deadline UTC offsets against the daylight-saving calendar before converting deadlines for users.
- Take structured fields (e.g. section 5.1.15) over titles when they disagree.

## Limitations

- Small sample: 8 notices, 14 lot rows, two countries. Results show patterns, not precise rates.
- One model (Gemini Flash-Lite) with one prompt; other models or prompts may behave differently.
- In every multi-lot notice, each lot's value equals the notice total, so the audit cannot tell whether the model read lot values or copied the notice total.
- No sampled notice lacked a stated value, so missing-value handling was tested only through the quantity field.

## Next steps

- Repeat with a larger model to compare cost against accuracy.
- Extend to more countries and to notices without stated values.
- Load the results into PostgreSQL and build a dashboard of accuracy by field and error type.
