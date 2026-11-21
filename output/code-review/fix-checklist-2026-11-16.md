# Fix Checklist — Code Review of readme.qmd

**Review ID:** cr-20261116-archy459
**Date:** 2026-11-16
**Verdict:** MAJOR ISSUES — DO NOT TRUST RESULTS

---

## CRITICAL FIXES (must resolve before trusting any results)

### □ [CRIT-001] Hallucinated `read_csv` argument
**File:** readme.qmd, line 10
**Agents:** style ★★ correctness robustness
**Code:** `read_csv("https://...", show_col_types = FALSE)`
**Problem:** `readr::read_csv()` has no `show_col_types` argument. Script fails.
**Fix:** Remove argument: `read_csv("https://ndownloader.figshare.com/files/24118688")`
**Affects:** Entire pipeline

### □ [CRIT-002] Join on `site_name` only, ignoring `country`
**File:** readme.qmd, line 64
**Agents:** correctness ★★ data-handling ★★ robustness
**Code:** `left_join(select(site_counts, site_name, n_site = n), by = "site_name")`
**Problem:** Cross-matches German↔French sites if same name exists in both countries.
**Fix:** `left_join(select(site_counts, country, site_name, n_site = n), by = c("country", "site_name"))`
**Affects:** Figure @fig-sites (site ordering, point sizes)

### □ [CRIT-003] `sign_pct()` returns `numeric(0)` for missing sign-country combos
**File:** readme.qmd, line 86
**Agents:** correctness ★★ data-handling ★★ robustness ★★ statistics ★★
**Code:** `round(100 * filter(sign_freq, country == ctry, sign == s)$prop)`
**Problem:** Missing sign → `numeric(0)` → inline renders as empty/"numeric(0)%".
**Fix:** 
```r
sign_pct <- function(ctry, s) {
  row <- filter(sign_freq, country == ctry, sign == s)
  if (nrow(row) == 0) return(NA_integer_)
  round(100 * row$prop)
}
```
Guard inline: `` `r ifelse(is.na(vulva_fr), "not observed", paste0(vulva_fr, "%"))` ``
**Affects:** All 7 inline percentages in Conclusion

### □ [CRIT-004] Non-deterministic deduplication on `site_name` + `object_type`
**File:** readme.qmd, line 30
**Agents:** robustness ★★ statistics ★★ data-handling ★★ correctness
**Code:** `distinct(site_name, object_type, .keep_all = TRUE)`
**Problem:** Arbitrary first-row keep; row order from remote CSV not stable.
**Fix:** `arrange(site_name, object_type, <tiebreaker>) |> distinct(site_name, object_type, .keep_all = TRUE)` or `distinct(.keep_all = TRUE)` for exact duplicates only.
**Affects:** All downstream analyses

### □ [CRIT-005] External data fetch without verification, caching, or error handling
**File:** readme.qmd, line 10
**Agents:** robustness ★★ reproducibility ★★ data-handling
**Code:** `signbase <- read_csv("https://ndownloader.figshare.com/files/24118688", ...)`
**Problem:** No checksum, no local fallback, no validation, network failure = render failure.
**Fix:** (1) Download to `data/raw/signbase_v1.0.csv`; (2) Record SHA256; (3) Read local; (4) Add `stopifnot(nrow(signbase) > 0, all(c("country","site_name","object_type") %in% names(signbase)))`.
**Affects:** Every reported number

### □ [CRIT-006] Positional column range `line:other` in `pivot_longer`
**File:** readme.qmd, line 33
**Agents:** robustness ★★ data-handling ★★ correctness ★★ style
**Code:** `pivot_longer(line:other, names_to = "sign", values_to = "present")`
**Problem:** Selects by position; breaks if column order changes.
**Fix:** `sign_cols <- c("notch","line","dot","cross","obline","vulva","other"); pivot_longer(all_of(sign_cols), ...)` + `stopifnot(all(sign_cols %in% names(signs_focus)))`.
**Affects:** `sign_freq`, `top10`, figure, conclusion

### □ [CRIT-007] `all_of(top10)` assumes all top-10 signs exist as columns
**File:** readme.qmd, line 61
**Agents:** robustness ★★ data-handling
**Code:** `pivot_longer(all_of(top10), ...)`
**Problem:** `top10` from long format; if any missing from wide columns, errors.
**Fix:** `top10_valid <- intersect(top10, names(signs_focus))`; warn if `< 10`; use `all_of(top10_valid)`.
**Affects:** Figure @fig-sites

### □ [CRIT-008] No row-count validation after critical filters
**File:** readme.qmd, lines 29/43/63
**Agents:** robustness
**Code:** Multiple `filter()`/`count()` without `nrow > 0` checks
**Problem:** Empty data → `slice_max` errors, `NaN` proportions, garbage inline.
**Fix:** `stopifnot(nrow(signs_focus) > 0, "No DE/FR records")`, `stopifnot(nrow(site_signs) > 0, "No sign-site combos")`.
**Affects:** All reported numbers

### □ [CRIT-009] `na.rm = TRUE` in proportion calculation without documented missing-data policy
**File:** readme.qmd, line 34
**Agents:** statistics ★★ data-handling
**Code:** `summarise(prop = mean(present, na.rm = TRUE), n_present = sum(present, na.rm = TRUE), ...)`
**Problem:** If NA = "not assessed", proportions inflated (denominator = assessed only).
**Fix:** Verify codebook; if NA=absent: `mutate(across(sign_cols, ~replace_na(., 0)))` before pivot; if NA=not assessed: `prop = sum(present, na.rm=TRUE) / sum(!is.na(present))` explicitly.
**Affects:** All sign proportions

### □ [CRIT-010] Dynamic `date: today` breaks bitwise reproducibility
**File:** readme.qmd, line 3
**Agents:** reproducibility
**Code:** `date: today`
**Fix:** Fixed date: `date: "2024-01-15"`
**Affects:** Document metadata

### □ [CRIT-011] Arbitrary tie-breaking with `with_ties = FALSE`
**File:** readme.qmd, lines 39/47/98
**Agents:** robustness ★★ data-handling ★★ correctness
**Code:** Three `slice_max(..., with_ties = FALSE)` calls
**Problem:** Ties at rank boundaries → arbitrary, unstable results.
**Fix:** `with_ties = TRUE` + deterministic tiebreaker (e.g., `arrange(sign)`), or document rule.
**Affects:** Figure signs, Discussion top-site claims, Conclusion top-3 sites

### □ [CRIT-012] Unguarded inline R expressions in prose
**File:** readme.qmd, lines 23/54/78/103
**Agents:** robustness
**Code:** Multiple `` `r ...` `` expressions
**Problem:** Upstream failure → garbage in manuscript; no trace to source chunk.
**Fix:** Compute all inline values in `echo: false` chunk with validation, store in variables, reference variables in text with guards: `` `r ifelse(is.na(x), "N/A", x)` ``.
**Affects:** All inline text

### □ [CRIT-013] No package version lockfile / environment specification
**File:** readme.qmd, line 9
**Agents:** reproducibility ★★ style
**Code:** `library(tidyverse)` (no versions)
**Problem:** `.by` syntax requires dplyr ≥ 1.1.0; no `renv.lock`, `sessionInfo()`, Dockerfile.
**Fix:** `renv::init(); renv::snapshot()` → commit `renv.lock`. Or add `sessionInfo()` chunk.
**Affects:** Computational reproducibility

### □ [CRIT-014] `fct_reorder(site_name, n_site)` after cross-country join duplication
**File:** readme.qmd, line 67
**Agents:** robustness
**Code:** `fct_reorder(site_name, n_site)` after `left_join(by = "site_name")`
**Problem:** If join duplicates (CRIT-002), `n_site` ambiguous, ordering wrong.
**Fix:** Fix join key first (CRIT-002).
**Affects:** Figure @fig-sites y-axis order

### □ [CRIT-015] Deduplication assumes at most one object per type per site
**File:** readme.qmd, line 30
**Agents:** data-handling ★★ statistics
**Code:** `distinct(site_name, object_type, .keep_all = TRUE)`
**Problem:** Collapses multiple objects of same type with different signs to one arbitrary record.
**Fix:** Define duplicates explicitly: use object ID if exists, else all sign columns. Verify no non-exact duplicates.
**Affects:** All downstream analyses

### □ [CRIT-016] No binary validation on sign columns
**File:** readme.qmd, line 33
**Agents:** data-handling
**Code:** `pivot_longer(...) → summarise(prop = mean(present))`
**Problem:** Assumes 0/1; if other values present, proportions meaningless.
**Fix:** `sign_cols <- setdiff(names(signs_focus), c("country","site_name","object_type")); stopifnot(all(sapply(signs_focus[sign_cols], function(x) all(x %in% c(0,1,NA)))))`.
**Affects:** All sign proportions

### □ [CRIT-017] Sign column completeness/order not verified (UNVERIFIABLE)
**File:** readme.qmd, line 33
**Agents:** data-handling
**Code:** `pivot_longer(line:other, ...)`
**Problem:** Cannot verify without data that `line:other` captures all sign columns.
**Fix:** Provide codebook with expected sign columns. Validate: `expected_signs <- c("notch","line","dot","cross","obline","vulva","other"); stopifnot(all(expected_signs %in% names(signbase)))`.
**Affects:** All sign analyses (requires manual check)

### □ [CRIT-018] Deduplication key uniqueness not verified (UNVERIFIABLE)
**File:** readme.qmd, line 30
**Agents:** data-handling
**Code:** `distinct(site_name, object_type, .keep_all = TRUE)`
**Problem:** Cannot verify if `(site_name, object_type)` uniquely identifies objects.
**Fix:** Provide codebook with object identifier. Check: `signbase |> count(site_name, object_type) |> filter(n > 1) |> nrow()`.
**Affects:** Analytic sample size (requires manual check)

### □ [CRIT-019] Raw missing value codes undocumented
**File:** readme.qmd, line 34
**Agents:** data-handling
**Code:** `summarise(prop = mean(present, na.rm = TRUE), ...)`
**Problem:** No docs on what NA means (absent vs not assessed). No sentinel checks.
**Fix:** Document missing-value semantics. Add `stopifnot(!any(signs_focus[sign_cols] < 0, na.rm = TRUE))`.
**Affects:** Proportion estimates

---

## WARNINGS (should resolve before submission)

### □ [WARN-001] Hardcoded URL single point of failure
**Fix:** Parameterize URL, document DOI (10.6084/m9.figshare.12000018.v1)

### □ [WARN-002] No data dictionary / codebook
**Fix:** Add `data/codebook.md` with columns, types, value ranges, SignBase version

### □ [WARN-003] No master/runner script, no README with execution instructions
**Fix:** Add `README.md` with render command, renv restore, data download, expected outputs

### □ [WARN-004] Figure missing `fig-width`
**Fix:** Add `#| fig-width: 10` alongside `fig-height: 8`

### □ [WARN-005] No script-level purpose comment
**Fix:** Add comment block at top of Chunk 1

### □ [WARN-006] DRY: 7 near-identical `sign_pct()` calls
**Fix:** Use `expand_grid(country, sign) |> mutate(pct = map2_int(country, sign, sign_pct)) |> pivot_wider(...)`

### □ [WARN-007] DRY: inline pipeline duplicates `sign_pct` logic
**Fix:** Pre-compute `top_sign_de`, `top_sign_fr` in chunk, reference variables in prose

### □ [WARN-008] `signs_focus` misleading name
**Fix:** Rename to `signs_dedup_site_type` or clarify key in comment

### □ [WARN-009] `n` → `n_site` rename at use-site
**Fix:** `site_counts <- ... |> mutate(n_site = n, share = n / sum(n), .by = country) |> select(-n)`

### □ [WARN-010] ggplot pipeline formatting
**Fix:** Add blank lines between layers, align `+` operators

### □ [WARN-011] Inline pipeline in prose too complex
**Fix:** Pre-compute in chunk, reference variable in prose

### □ [WARN-012] No R version declared
**Fix:** Add `r_version: "4.4.0"` to `renv.lock` or comment

### □ [WARN-013] No computational requirements documented
**Fix:** Add comment: `# Expected runtime: ~5s; requires internet; ~50 MB RAM`

### □ [WARN-014] `slice_max(by=)` requires dplyr ≥ 1.1.0
**Fix:** Use `group_by(country) |> slice_max(n, n=1) |> ungroup()` for portability

### □ [WARN-015] No formal statistical test of repertoire difference
**Fix:** Add chi-square/permutation test, or clarify descriptive only

### □ [WARN-016] Global `theme_set()` leaks in multi-figure docs
**Fix:** Note in comment (acceptable for single-figure)

### □ [WARN-017] Discussion acknowledges non-independence but analysis doesn't reflect it
**Fix:** Add site-level analysis or clarify descriptive only

### □ [WARN-018] No verification step comparing rendered values to reference
**Fix:** Add known-good reference values and comparison chunk

### □ [WARN-019] No local cache of downloaded data
**Fix:** Use `pins` or manual download + `read_csv("data/signbase.csv")`

### □ [WARN-020] No package version pinning for long-term reproducibility
**Fix:** `renv::record()` or `dependencies.R`

### □ [WARN-021] Three-valued logic collapse in `sign_pct`
**Fix:** Return `NA_integer_` on no match, guard inline (see CRIT-003)

### □ [WARN-022] Three-valued logic collapse in `pivot_longer(line:other)`
**Fix:** Explicit `sign_cols` vector validated against `names(signs_focus)`

### □ [WARN-023] Three-valued logic collapse in `left_join(by="site_name")`
**Fix:** Join on `c("country", "site_name")`; add duplicate check

### □ [WARN-024] Prose fields assembled at run time without fail-closed guards
**Fix:** Each inline field computed with validation, returning `NA` + flag rather than `numeric(0)`