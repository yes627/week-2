This is a designed trap file. The design is:

*12 dates x 4 items x 2 cities = 96 expected rows, 24 per item, 12 per item-city, 8 per date*

Your 6 decisions were mostly right in _intent_, but a naive AI cleaning that just does `drop_duplicates() + dropna() + to_numeric() + to_datetime() + str.title()` *fails the expectation test* and silently changes numbers/labels.

### 1. Before / After trial
Stage | Rows | Missing price | `describe()` price | Item counts
**Before** | 97 | 1 | Cannot describe - object | Staple A 24, Oil B 24, Pulse C 24, Sweetener D 24, `oil b` 1
**Naive AI: drop_dup + to_numeric(errors=coerce) + dropna** | 94 | 0 | Mean 307.8, SD 1013.4, Min 99.5, Max 9999 | Staple A 22, Oil B 24, Pulse C 23, Sweetener D 24, `oil b` 1
**AI + `str.title()` for item** | 95 | 0 | Mean 317.7, SD 1012.7, Min 99.5, Max 9999 | Staple A 23, Oil B **25**, Pulse C 23, Sweetener D 24
**Expected** | **96 clean, 95 after 1 true missing** | **1** | Mean ~214.7, Median 200 if 9999 + 1250 excluded | **24 each**
No stage reaches 24 each. That is your audit signal.

### 2. Verdict per line

*1. Missing price - KEEP your decision, MODIFY AI if it just drops*

- Evidence: Row 5, `2026-01-01, Pulse C, City 2`, price = `''` -> NaN. No other source to recover it from.
- Trial: Dropping it alone: 97 -> 96 rows, Pulse C 24 -> 23. That is correct, it was unrecoverable.
- Verdict: *KEEP* - investigate then exclude from price analysis only. Do not impute with mean, you have no basis.

*2. Duplicated row - KEEP*

- Evidence: Row 96 is _exact_ duplicate of Row 8: `2026-01-08, Staple A, City 1, 99.5`
- Trial: `drop_duplicates()` 97 -> 96, Date `2026-01-08` 9 -> 8, City 1 49 -> 48. No valid observation lost.
- Verdict: *KEEP* - drop row 96.

*3. price is object - MODIFY / REJECT naive conversion*

This is where the AI loses a valid row and keeps two bad numbers:

- Why object? Not just the missing value. Raw values include: `''`, `'1,250'`, `'9999'`
- Trial A - naive `pd.to_numeric(errors='coerce')`: `'1,250'` -> NaN. Row 17, `2026-01-15, Staple A, City 2`, is then dropped as if missing. *Valid observation removed without you knowing.*
- Trial B - `str.replace(',','')` first: Row 17 becomes *1250.0*. But Staple A in this file is 99.5 to 116.5 everywhere else. 1250 is ∼10x too high. It is almost certainly `125.0` with a misplaced comma/decimal, not 1250.
- Row 30: `2026-01-22, Sweetener D, City 1, 9999`. Sweetener D is 149 to 172.5 everywhere else. 9999 is a placeholder / typo. If kept, mean jumps from 214.7 to 317.7, SD from ∼70 to 1012.7, Max 9999.
- Verdict: *MODIFY* - Do not `to_numeric + dropna` blindly. Strip comma, then _flag_ 1250 and 9999 as suspect/outliers for source check. Do not present 1250 or 9999 as real prices. Correct AI code needs a separate outlier/validation step.

*4. date is object - KEEP, with check*

- Evidence: `value_counts()` shows all 12 dates in consistent `YYYY-MM-DD` ISO format, no variants like `01/02/2026` or text months.
- Trial: `pd.to_datetime()` succeeds on all 97, 0 NaT, min 2026-01-01, max 2026-03-19, 8 per date except 2026-01-08 with 9 (due to duplicate).
- Verdict: *KEEP* - conversion is safe. Your instinct to check formats first was right, it just happens to pass.

*5. All 6 columns are object - MODIFY*

- Evidence: `unit` = 1 unique value `PKR per kg (illustrative)` for all 97. `data_label` = 1 unique value for all 97. `city` = only `City 1 / City 2`, no variants.
- Not every column is messy. Treating all as messy invites unnecessary recoding.
- Verdict: *MODIFY* - Flag as “pandas inferred object because of 3 bad price cells”, not “all columns dirty”. Only `price`, `item`, `date` need work.

*6. item / city / unit / data_label inconsistent text - REJECT naive AI fix, KEEP your investigate-first decision*

This is the biggest hidden trap:

Row 40: `2026-02-05, oil b, City 1, 108.5`

Naive AI: `str.strip().str.title()` -> `Oil B`. Then:
- Oil B becomes 25, not 24
- Oil B, City 1 becomes 13, not 12
- Staple A becomes 23, and Staple A, City 1 on 2026-02-05 is missing

Why? On 2026-02-05 you already have a proper `Oil B, City 1, 318.0` in Row 42. Row 40's price 108.5 matches Staple A pattern (Staple A City 1 is ∼103-113), not Oil B pattern (298-350). Row 40 is almost certainly a *mislabelled Staple A row*, not a case variant of Oil B.

If you just title-case, you *change a label without knowing, create a duplicate key, and lose a Staple A observation*.

- `city`, `unit`, `data_label`: `value_counts()` shows *no* variants. No correction needed.
- Verdict: *REJECT* title-case as a fix for row 40. *KEEP* your decision: only correct if you find a variant, and here the variant is not a simple case variant. Flag Row 40 as label error / exclude from item-level analysis until source confirms.

### 3. After cleaning, does every item appear as expected?

No.

- After correct dedup + true missing drop: 95 rows, expected 96.
- After naive AI: 94-95 rows, counts 22-25, none = 24.
- After flagging suspect rows 17 (1250), 30 (9999), 40 (mislabelled): 92 trustworthy rows for price-by-item analysis, with a documented audit trail for the 3 excluded + 1 missing + 1 duplicate = 5 problem rows.

That is the correct answer, not 95 clean rows.

### 4. Reflection - where you differed from AI, and what you would have missed alone

- *You were better supported on 1, 2, 6:* You said investigate before dropping / recoding. AI would drop Row 17 as missing and recode Row 40 as Oil B. Your rule prevents both silent losses.
- *AI was better supported on 4:* You worried about date formats. AI's `to_datetime` trial proves they are all clean ISO. Checking first still wins, but here the check passes.
- *Both missed at first:* The expectation test - 12 x 4 x 2 = 96. Without that, 95 rows looks “clean”. With it, Oil B = 25 immediately exposes Row 40.
- *What you would have missed alone:* That `1,250` and `9999` are not just “convert to numeric” problems, they are magnitude errors that destroy mean/SD, and that `oil b` is not a case problem but a label problem. Alone you might have done `str.title()` and trusted it. The trial - before/after `value_counts()` + `describe()` + item-city counts - catches it.