# Insights log: LendingClub loan default project

A running record of what each step of the analysis found and what it means for the next step. Decisions and the options behind them are kept separately in `decision_log.pdf`.

- **Notebook:** `lending_club_1st_draft.ipynb`
- **Data:** `accepted_2007_to_2018Q4.csv` (LendingClub accepted loans, 2007 to 2018)
- **Goal:** predict whether a loan ends up Charged Off or Fully Paid, using only what was known at application
- **Last updated:** October 9, 2026, during Stage 1, Step 5 (free text, joint figures, over-30% columns, leakage and imputation done)

---

## Stage 1, Step 1: Inspect the raw data

**What we did:** looked at the shape, first and last rows, column types, non-null counts and the numeric summary.

| Fact | Value |
|---|---|
| Rows | 2,260,701 |
| Columns | 151 |
| Numeric columns | 113 |
| Text columns | 38 |

**Insights**

- **`describe()` hides nothing.** It returns 113 columns, the same as the number of numeric columns, so no numeric column is being silently skipped.
- **The file contains rows that are not loans.** The last two rows are summary lines ("Total amount funded in policy code..."), and most columns have 2,260,668 values, 33 fewer than the row count.
- **`id` is stored as text.** The non-loan rows put text into a column of numbers.
- **`member_id` is completely empty** and `policy_code` has the same value on every loan.
- **`funded_amnt` nearly duplicates `loan_amnt`.** Their averages are $15,041.66 and $15,046.93.
- **Several text columns are really numbers or dates:** `term`, `emp_length`, `issue_d`, `earliest_cr_line`, `last_pymnt_d`, `next_pymnt_d`, `last_credit_pull_d`.
- **A block of columns is filled on only 1,394,539 loans,** all by the same amount, which pointed to fields added to the data partway through (confirmed in Step 3).

**What it means for later steps:** every item above became a check in Step 2.

---

## Stage 1, Step 2: What is broken?

**What we did:** ran six checks on the raw data without changing it.

| Check | Result |
|---|---|
| Junk rows | 33 rows that are not loans; only `id` holds a value |
| Columns with no information | `member_id` (empty); `policy_code`, `hardship_type`, `deferral_term`, `hardship_length` (one value each) |
| Duplicates | 0 repeated ids; 0 identical loans once `id` and `url` are ignored |
| Numbers and dates stored as text | `term` has two values with a leading space; `emp_length` has 11 text levels; all five date columns are in `Mon-YYYY` form with 0 exceptions |
| Stand-in values in text columns | `home_ownership` has ANY (996), OTHER (182), NONE (54) |
| Impossible numbers | `dti` of 999 (135 loans) and -1 (2 loans); `dti` above 100 (2,561); zero income (1,667); `revol_util` above 100% (7,343) |

**Insights**

- **The junk rows are scattered through the file, not only at the bottom.** They sit at 16 places, each a pair of "Total amount funded" lines, plus one "Loans that do not meet the credit policy" header. The file is several LendingClub exports stacked together, so looking at the last rows alone finds only 2 of the 33.
- **`emp_length` is missing on 146,907 loans (6.5%).** That is in the "impute carefully" band.
- **Placeholder values hide a little missing data in `dti`.** The reported missing rate is 0.077%; counting 999 and -1 as missing makes it 0.083%. The effect is real but small.
- **Most "suspicious" dollar values are legitimate.** A balance of exactly $999 or $9,999 is a normal amount.
- **Two columns top out at 999 months** (`mo_sin_old_il_acct`, `mo_sin_old_rev_tl_op`), which looks like a placeholder on 3 loans.

### Follow-up: zero income and very high `dti`

| Group | Loans | Application type |
|---|---|---|
| `dti` above 100 | 2,561 | all Joint App |
| Zero annual income | 1,667 | all Joint App |
| Both at once | 0 | n/a |

- **These are not errors.** Every one is a joint application, where the individual applicant's income and `dti` are not the figures the loan was judged on.
- **Zero income explains the missing `dti`:** 1,659 of the 1,667 zero-income loans have no `dti`.
- **Action (decision D3):** for joint applications, use `annual_inc_joint` and `dti_joint` in place of the individual figures, and keep `application_type` as a feature.
- **Consequence:** the joint columns are about 95% empty but must not be dropped until the combined columns are built.

### Follow-up: `home_ownership` small groups

| Value | Loans | Years they appear | Finished loans |
|---|---|---|---|
| ANY | 996 | 2014 to 2018, almost all 2016 to 2018 | 286 |
| OTHER | 182 | 2008 to 2012 | 144 |
| NONE | 54 | 2007, 2008, 2012, 2017 | 48 |

- **They are artifacts of the application form changing over time,** and together they are only 478 finished loans.
- **Proposed action (not yet confirmed):** merge them into a single OTHER category.

---

## Stage 1, Step 3: Pre-cleaning visuals

**What we did:** drew the missing rate per column (in detail and grouped by family) and the loan-status distribution, all on the raw data.

### Missing values

| Columns with | Count |
|---|---|
| No missing values | 38 |
| Under 5% missing | 50 |
| 5% to 30% missing | 5 |
| Over 30% missing | 58 |

| Family of columns | Columns | Missing | Why it is empty |
|---|---|---|---|
| Hardship plan fields | 14 | 99.5% | Only filled for borrowers on a hardship plan |
| Debt settlement fields | 6 | 98.5% | Only filled for settled loans |
| Empty identifier and optional free text | 2 | 94% to 100% | `member_id` is empty; `desc` is optional |
| Second applicant and joint fields | 16 | 95% to 98% | Only filled for joint applications |
| Months since last delinquency or record | 5 | 51% to 84% | Blank means the event never happened |
| Next payment date | 1 | 59.5% | Finished loans have no next payment |
| Newer credit-bureau fields | 14 | 38% to 47% | Not collected in the early years |
| Other borrower and credit fields | 5 | 6% to 13% | Ordinary missing values |

**Insights**

- **58 of 151 columns are more than 30% empty, almost all for structural reasons.** The field does not apply to that loan; the data is not randomly lost.
- **The "newer fields" explanation is confirmed.** `open_acc_6m` is 100% missing for every issue year from 2007 to 2014, 94.9% in 2015, and 0% from 2016 on.
- **Different reasons call for different treatment.** "The event never happened" is information; "not collected yet" is not.
- **The 63-bar chart is the detailed evidence; the grouped chart is the one for the slides.**

### Loan status

| Status | Loans | Share |
|---|---|---|
| Fully Paid | 1,076,751 | 47.6% |
| Current | 878,317 | 38.9% |
| Charged Off | 268,559 | 11.9% |
| Late (31-120 days) | 21,467 | 0.9% |
| In Grace Period | 8,436 | 0.4% |
| Late (16-30 days) | 4,349 | 0.2% |
| Does not meet the credit policy: Fully Paid | 1,988 | 0.1% |
| Does not meet the credit policy: Charged Off | 761 | 0.0% |
| Default | 40 | 0.0% |

**Insights**

- **Only two statuses are a final outcome for a normal loan:** Fully Paid and Charged Off.
- **Current, Late and In Grace Period loans are still open.** Their status is a snapshot at the time the data was captured, and they can still go either way.
- **Among finished loans the charge-off rate is 20.0%,** a class imbalance of about 4 to 1.

Status meanings are from general knowledge of LendingClub reporting and should be confirmed against the data dictionary.

---

## Stage 1, Step 4: Define the target

**What we did:** kept only Fully Paid and Charged Off loans in a new data frame (`loans`), created the 0/1 target column `charged_off`, rechecked missing rates, and looked at the effect year by year. The raw data frame (`df`) is untouched. This applies decision D4.

### The target

| | Loans |
|---|---|
| Rows before (raw data) | 2,260,701 |
| Rows after (finished loans) | 1,345,310 |
| Rows removed | 915,391 |

| `charged_off` | Loans | Share of finished loans |
|---|---|---|
| 0 = Fully Paid | 1,076,751 | 80.0% |
| 1 = Charged Off | 268,559 | 20.0% |

### Missing rates after the filter

| Columns with | Raw data | Finished loans |
|---|---|---|
| No missing values | 38 | 46 |
| Under 5% missing | 50 | 21 |
| 5% to 30% missing | 5 | 26 |
| Over 30% missing | 58 | 58 |

**Insights**

- **`next_pymnt_d` is now 100% empty.** Drop it.
- **The 14 newer credit-bureau fields went from about 38% to about 60% empty** (`il_util` to 65%), because finished loans are mostly older loans. Drop them.
- **Joint columns are 98.1% empty among finished loans.** Only about 1.9% of the loans we keep are joint applications, so decision D3 affects a small slice.
- **The 5% to 30% band grew from 5 columns to 26, and all 21 new arrivals are detailed credit-bureau fields** (balances, credit limits, account counts and account ages such as `tot_cur_bal`, `total_rev_hi_lim`, `num_*`, `mo_sin_*`). They were under 5% missing in the raw data and now need careful imputation.
- **Eight columns became complete:** seven core credit-history columns and one borrower column went from under 5% missing to no missing values.
- **The over-30% band did not change:** the same 58 columns, none in and none out.

### Year by year

Share of each year's loans that are finished, and the charge-off rate among those finished loans:

| Issue year | All loans | Finished | Finished (%) | Charged off (% of finished) |
|---|---|---|---|---|
| 2007 | 603 | 251 | 41.6 | 17.9 |
| 2008 | 2,393 | 1,562 | 65.3 | 15.8 |
| 2009 | 5,281 | 4,716 | 89.3 | 12.6 |
| 2010 | 12,537 | 11,536 | 92.0 | 12.9 |
| 2011 | 21,721 | 21,721 | 100.0 | 15.2 |
| 2012 | 53,367 | 53,367 | 100.0 | 16.2 |
| 2013 | 134,814 | 134,804 | 100.0 | 15.6 |
| 2014 | 235,629 | 223,102 | 94.7 | 18.4 |
| 2015 | 421,095 | 375,545 | 89.2 | 20.2 |
| 2016 | 434,407 | 293,095 | 67.5 | 23.3 |
| 2017 | 443,579 | 169,300 | 38.2 | 23.1 |
| 2018 | 495,242 | 56,311 | 11.4 | 15.7 |

The same two rates split by loan term:

| Issue year | Finished (%), 36 months | Finished (%), 60 months | Charged off (%), 36 months | Charged off (%), 60 months |
|---|---|---|---|---|
| 2007 | 41.6 | n/a | 17.9 | n/a |
| 2008 | 65.3 | n/a | 15.8 | n/a |
| 2009 | 89.3 | n/a | 12.6 | n/a |
| 2010 | 92.5 | 90.8 | 9.9 | 21.0 |
| 2011 | 100.0 | 100.0 | 10.6 | 23.6 |
| 2012 | 100.0 | 100.0 | 13.6 | 27.7 |
| 2013 | 100.0 | 100.0 | 12.3 | 25.1 |
| 2014 | 100.0 | 82.9 | 13.7 | 31.1 |
| 2015 | 99.9 | 67.1 | 14.9 | 36.4 |
| 2016 | 71.8 | 54.8 | 19.8 | 36.4 |
| 2017 | 40.1 | 33.1 | 20.0 | 32.9 |
| 2018 | 12.0 | 10.0 | 13.2 | 22.6 |

"n/a" means no 60-month loans were issued that year.

**Insights**

- **60-month loans charge off about twice as often as 36-month loans** in the complete years (23.6% to 27.7% against 10.6% to 13.6%). Term is a strong candidate feature.
- **36-month loans are complete through 2015; 60-month loans are complete through 2013.** After that, a growing share is still open.
- **The incomplete years are biased.** A recent loan that is already "finished" either defaulted early or was paid off early. The 36.4% charge-off rate for 60-month loans from 2015 and 2016 is not a real default rate.
- **The 2007 to 2010 gaps are a different issue.** The missing loans there are almost certainly the "does not meet the credit policy" loans removed by decision D4 (about 2,750 by arithmetic, against 2,749 in the file). Not yet checked directly.

### Keep only loans old enough to have finished (decision D5)

**What we did:** added a rule on top of the status filter. A loan is kept only if its full term (36 or 60 months) had passed by the time the data was captured. This removes the biased recent years found above.

| Fact | Value |
|---|---|
| Data captured | March 2019 (the latest payment date in the file; 853,003 loans have a last payment in that month) |
| Latest loan issued | December 2018 |
| Finished loans before the rule | 1,345,310 |
| Loans kept | 776,276 (57.7%) |
| Loans removed | 569,034 |

| Loan term | Finished loans before | Kept | Removed | Last issue month kept | Charged off before | Charged off after |
|---|---|---|---|---|---|---|
| 36 months | 1,020,743 | 708,368 | 312,375 | March 2016 | 16.0% | 14.2% |
| 60 months | 324,567 | 67,908 | 256,659 | March 2014 | 32.4% | 25.3% |

| `charged_off` | Loans | Share of loans kept |
|---|---|---|
| 0 = Fully Paid | 658,666 | 84.8% |
| 1 = Charged Off | 117,610 | 15.2% |

**Insights**

- **The real charge-off rate is 15.2%, not 20.0%.** The 20% figure was inflated by recent loans that only counted as finished because they ended early. Every earlier mention of "20%" as the class imbalance should now read 15.2% (about 5.6 fully paid loans for each charged-off one).
- **The bias was strongest for 60-month loans:** their charge-off rate falls from 32.4% to 25.3% once the incomplete years are removed.
- **60-month loans are now a small part of the data:** 67,908 loans, 8.7% of what is kept, all issued between 2010 and March 2014.
- **The rule works.** Of all 785,915 loans in the kept age range, 98.8% are finished. The rest are the 2,749 "does not meet the credit policy" loans removed by decision D4 and about 6,900 loans still open past their original term.
- **60-month loans still charge off nearly twice as often** as 36-month loans (25.3% against 14.2%), so term remains a strong candidate feature.

**What it means for later steps**

- Missing rates have to be rechecked on the 776,276 loans before cleaning, because the kept loans are older and the bureau fields are emptier for older loans.
- Joint applications started late in the period, so very few are left. The effect of decision D3 should be recounted.
- Loans issued in the first quarter of 2016 sit right at the cutoff. Their charge-off rate (16.1%) is slightly above 2015's (14.9%), which may be a small leftover of the same bias.

These numbers were computed directly from the CSV with the same code that is in the notebook cell.

---

## Stage 1, Step 5: Clean the data (in progress)

### Free-text and location columns (decision D6)

**What we did:** looked at the four text columns with many distinct values, on the 776,276 loans kept, and dropped two of them.

| Column | Distinct values | Values that appear once | Missing | Most common value | Decision |
|---|---|---|---|---|---|
| `emp_title` | 272,023 | 224,000 | 6.6% | Teacher (11,258) | Drop |
| `title` | 61,678 | 52,638 | 0.9% | Debt consolidation (330,469) | Drop |
| `zip_code` | 924 | 32 | 0% | 945xx (8,859) | Keep |
| `addr_state` | 51 | 0 | 0% | CA (118,049) | Keep |

**Insights**

- **`emp_title` is hand-typed and unusable as it stands.** The same job appears as "Registered Nurse", "RN", "Registered nurse", "registered nurse", "Rn", "rn" and "REGISTERED NURSE".
- **`title` repeats `purpose`.** The most common title for each purpose is that purpose in words ("Car financing" for `car`, "Medical expenses" for `medical`).
- **`zip_code` holds only the first three digits** (for example `945xx`), which is a postal area named after a city or region, not a full ZIP code.
- **Location carries signal.** Among the 402 ZIP areas with at least 500 loans, the charge-off rate runs from 8.7% to 21.3%, against 15.2% overall.
- **Charge-off rate by state** runs from 9.6% (Maine) and 9.7% (DC) to 18.9% (Mississippi) and 19.6% (Nebraska).
- **Count before comparing places.** Iowa has 7 loans and Idaho 214; 168 of the 924 ZIP areas have fewer than 100 loans.

**What it means for later steps**

- A map of charge-off rate by state works directly in Tableau and is a strong dashboard candidate.
- A three-digit ZIP map needs an outside lookup file joined to the data. It is optional, for the end if time allows, and would also count as a merge.

### Combined income and `dti` for joint applications (decision D3)

**What we did:** built `annual_inc_combined` and `dti_combined`, which take the joint figure for joint applications and the individual figure otherwise.

| | Loans | Share of loans kept |
|---|---|---|
| All loans kept | 776,276 | 100% |
| Joint applications | 1,627 | 0.21% |
| Income taken from the joint figure | 1,627 | 0.21% |
| `dti` taken from the joint figure | 1,624 | 0.21% |

| Problem | Using individual figures | Using combined figures |
|---|---|---|
| Loans with an income of zero | 8 | 0 |
| Loans with `dti` missing | 9 | 0 |
| Loans with `dti` above 100 | 22 | 0 |

**Insights**

- **Joint applications are now a tiny slice: 1,627 loans, 0.21% of the data.** They started late in the period, and the loan-age rule removed most of them (they were 5.3% of the raw file).
- **The fix still removes every problem value:** no zero incomes, no missing `dti`, no `dti` above 100 once the combined figures are used.
- **This decision now matters much less than it looked in Step 2.** It is worth one line in the presentation, not a slide.

### Columns more than 30% empty (decision D7)

**What we did:** recalculated missing rates on the 776,276 loans and dropped every column over 30%.

| Kind of column | Columns dropped | Missing on the loans kept |
|---|---|---|
| Second applicant and joint fields | 16 | 99.8% to 100% |
| Hardship plan fields | 14 | 99.8% |
| Debt settlement fields | 6 | 98.3% |
| Newer credit-bureau fields | 14 | 86.5% to 88.4% |
| Months since last delinquency or record | 5 | 51.3% to 84.2% |
| Empty identifier and optional free text (`member_id`, `desc`) | 2 | 84.1% to 100% |
| Next payment date | 1 | 100% |
| **Total** | **58** | |

**Insights**

- **The same 58 columns as in the raw data, but emptier.** The newer credit-bureau fields went from about 38% missing in the raw file to about 87% here, because the loans kept are mostly older than those fields.
- **Dropping the "months since" columns gives up some information.** A blank there means the event never happened. Other columns (`delinq_2yrs`, `pub_rec`) still record whether it did.

### Leakage columns (decision D8)

**What we did:** dropped the 17 columns recorded after the loan was issued.

| Evidence | Fully Paid | Charged Off |
|---|---|---|
| Loans with money recovered after charge-off | 0.0% | 78.7% |
| Average share of the loan amount repaid | 99.9% | 39.8% |
| Average latest FICO score (after issue) | 700.9 | 573.9 |
| Average FICO score at application | 701.1 | 690.7 |

**Insights**

- **These columns give the outcome away.** A model using them would look excellent and be useless, because none of this is known when an application comes in.
- **The FICO comparison is the clearest example.** At application the two groups were only 10 points apart (701.1 against 690.7). The latest score shows a 127-point gap, which opened after the loans went bad.
- **FICO at application is a real but modest signal,** and it stays in the data.

### Missing values in numeric columns (decision D9)

**What we did:** filled the 39 numeric columns that still had missing values. A skewed column is filled with its median; a roughly symmetric column (skew between -0.5 and 0.5) is filled with its mean. A new yes/no column, `bureau_not_collected`, marks the loans whose credit-bureau fields were never collected.

**Why the values were missing**

| Fields | Missing | First collected |
|---|---|---|
| `mths_since_recent_inq` | 16.1% | March 2012 |
| `num_tl_120dpd_2m` | 12.1% | August 2012 |
| `mo_sin_old_il_acct` | 11.9% | August 2012 |
| 21 balance, limit, account-count and account-age fields | 8.7% | August 2012 |
| `num_sats`, `num_bc_sats` | 7.2% | June 2012 |
| `bc_util`, `percent_bc_gt_75`, `bc_open_to_buy`, `mths_since_recent_bc` | 6.9% to 7.1% | March 2012 |
| `mort_acc`, `acc_open_past_24mths`, `total_bal_ex_mort`, `total_bc_limit` | 6.1% | March 2012 |
| `pub_rec_bankruptcies`, `revol_util`, `chargeoff_within_12_mths`, `collections_12_mths_ex_med`, `tax_liens` | under 0.1% | n/a |

**How they were filled**

| Fill | Columns | Which |
|---|---|---|
| Median (skewed) | 35 | All the rest, for example `tot_cur_bal` (skew 3.2, filled with $69,424) and `avg_cur_bal` (skew 4.3, filled with $6,653) |
| Mean (roughly symmetric) | 4 | `mo_sin_old_il_acct` (skew 0.36, filled with 125.47), `bc_util` (-0.46, 62.67), `percent_bc_gt_75` (0.09, 48.24), `revol_util` (-0.08, 53.88) |

The full list, with each column's skew and fill value, is Table 14 in the notebook.

**Insights**

- **The missing values are an era, not random gaps.** Every credit-bureau field is 100% missing for loans issued from 2007 to 2011 and nearly complete (31 of 34 under 1.4%) once it started being collected in 2012.
- **Most of these fields are strongly skewed:** 29 of the 34 have a skew above 1. For the balance and limit fields the mean is up to twice the median (`tot_cur_bal`: mean $132,392, median $69,424).
- **The early loans behave like the rest.** Loans issued before September 2012 charged off at 15.3%, against 15.1% for later loans, so a typical fill value does not create a false pattern.
- **67,527 loans are marked `bureau_not_collected`.** Any chart of a credit-bureau field should leave them out, or it will show an artificial spike at the fill value.
- **Some fields are almost all zeros** and are drop candidates at feature selection: `num_tl_120dpd_2m` (99.93% zero), `num_tl_30dpd` (99.67%), `chargeoff_within_12_mths` (99.2%), `collections_12_mths_ex_med` (98.6%), `tax_liens` (97.0%).

**What the before/after chart shows**

The notebook draws each filled column as a small panel: real values in blue, the same column after filling as a red outline, with the median (solid line) and mean (dashed line) of the real values. There is a 12-panel version for the slides and a 39-panel version for the record.

- **Filling changes exactly one bar per column.** The red outline rises above the blue only at the fill value. For `tot_cur_bal` that jump is 67,527 loans at $69,424; for `mths_since_recent_inq` it is 124,713 loans at 5 months.
- **The chart justifies the rule.** In the skewed panels the dashed mean line sits well to the right of the solid median line (`tot_cur_bal`: mean $132,392, median $69,424). In `revol_util` and `mo_sin_old_il_acct` the two lines nearly coincide.
- **"Roughly symmetric" by skew is not the same as bell-shaped.** `percent_bc_gt_75` has a skew of 0.09 but is not a bell: its values pile up at 0, 50 and 100. Its mean (48.24) lands next to the common value 50, so the fill is reasonable, but the column should be described as balanced, not normal. `bc_util` (skew -0.46) leans left, and its mean (62.67) sits a little below its median (66.6).
- **Several skewed columns are a single bar:** `num_tl_120dpd_2m`, `num_tl_30dpd`, `tax_liens`, `collections_12_mths_ex_med`, `chargeoff_within_12_mths` and `pub_rec_bankruptcies` are almost all zeros.

**Caveats**

- **`mths_since_recent_inq` is filled with its median of 5 months, which is not quite right.** A blank there means no inquiry in about two years (74,666 of 74,677 blanks have zero inquiries in the last six months), and those borrowers charged off at 10.5% against 15.7%. The median fill hides that. A "no recent inquiry" flag would keep it.
- **For the model, fill values should be recalculated on the training data only.** Filling before a train/test split lets test data influence the values.
- **Still missing after this step:** `emp_length` (44,755 loans, 5.8%; it is text) and the individual `dti` (9 loans; replaced by `dti_combined`).

### Where Step 5 stands

| | Columns |
|---|---|
| Start of Step 5 (151 original, plus the target) | 152 |
| Added: combined income and `dti` | +2 |
| Dropped: free text | -2 |
| Dropped: more than 30% empty | -58 |
| Dropped: leakage | -17 |
| Added: `bureau_not_collected` marker | +1 |
| **Now** | **78** |

Missing values still to handle, on the loans kept: `emp_length` (5.8%, text) and the individual `dti` (9 loans).

Still in the data and not yet decided: the identifiers `id` and `url`, the single-value `policy_code`, the near-duplicates `funded_amnt` and `funded_amnt_inv`, and the individual `annual_inc` and `dti` alongside the combined versions.

---

## Open items

| Item | Status |
|---|---|
| Restrict to loans old enough to have finished | Decided (D5): 36-month loans through March 2016, 60-month through March 2014 |
| Recheck missing rates and count joint applications on the 776,276 kept loans | Done in Step 5 |
| Rest of Step 5: identifiers and single-value columns, text to numbers and dates, placeholder values, `home_ownership` merge, `emp_length`, outliers, derived features | To do |
| Decide whether `mths_since_recent_inq` gets a "no recent inquiry" flag | Open |
| Replace "20%" with "15.2%" wherever the class imbalance is quoted (slides, ML direction) | To do |
| Exclude the lender's rating columns (`grade`, `sub_grade`, `int_rate`, `installment`) from the model | Proposed, not confirmed |
| Merge `home_ownership` ANY, OTHER, NONE into OTHER | Proposed, not confirmed |
| Remaining fixes from the Step 2 problem list | Proposed, not confirmed |
| Confirm column and status meanings against LendingClub's data dictionary | To do |
| GPU-accelerated EDA line in the rubric | Not addressed |
| Remove the Colab leftover cells at the top of the notebook | To do |
