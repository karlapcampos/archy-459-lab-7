# Consolidated Code Review Report

**Review ID:** cr-20261116-archy459
**Date:** 2026-11-16
**Mode:** full (all 6 agents)
**Scripts reviewed:** 1 (readme.qmd — Quarto document with embedded R chunks)
**Target journal:** Not specified (archaeology / Paleolithic research)

---

## 1. Script Inventory

| # | Script | Language | Lines | Packages | Produces |
|---|--------|----------|-------|----------|----------|
| 1 | readme.qmd | R (Quarto) | ~117 (total) / ~60 (R) | tidyverse, scales, knitr | Inline values, figure @fig-sites |

---

## 2. Overall Review Scorecard

| Dimension | Agent | Issues | Critical | Warnings | Score |
|-----------|-------|--------|----------|----------|-------|
| Correctness & Logic | review-code-correctness | 9 | 2 | 5 | FAIL |
| Robustness | review-code-robustness | 15 | 6 | 6 | FAIL |
| Statistical Fidelity | review-code-statistics | 6 | 2 | 1 | FAIL |
| Reproducibility | review-code-reproducibility | 11 | 3 | 5 | FAIL |
| Style & AI Patterns | review-code-style | 8 | 2 | 4 | FAIL |
| Data Handling | review-code-data-handling | 9 | 4 | 3 | FAIL |

**OVERALL**

| Total Issues | Critical | Warnings | ★★ Cross-Agent | Overall Grade |
|-------------|----------|----------|----------------|---------------|
| 58 | **19** | 24 | 12 | **F** |

**VERDICT: MAJOR ISSUES — DO NOT TRUST RESULTS**

> **Reason:** 19 CRITICAL issues (exceeds >5 threshold), including 12 ★★ cross-agent agreement findings. The code contains multiple silent failure modes that produce wrong results without errors: non-deterministic deduplication, join on wrong key, missing-sign percentages rendering as empty text, external data without verification, and hallucinated function arguments.

---

## 3. Fix Checklist (CRITICAL + WARNING)

### CRITICAL FIXES (must resolve before trusting any results)

#### □ [CRIT-001] readme.qmd, line 10 — **Hallucinated `read_csv` argument**
- **Agent(s):** style ★★ correctness robustness
- **Code:** `read_csv("https://...", show_col_types = FALSE)`
- **Problem:** `readr::read_csv()` does **not** have a `show_col_types` argument (it exists in `read_delim()` only). The script will error: `unused argument (show_col_types = FALSE)`.
- **Fix:** Remove the argument: `read_csv("https://ndownloader.figshare.com/files/24118688")` — modern readr suppresses column spec by default.
- **Affects:** Entire pipeline (no data = no analysis)

#### □ [CRIT-002] readme.qmd, line 64 — **Join on `site_name` only, ignoring `country`**
- **Agent(s):** correctness ★★ data-handling ★★ robustness
- **Code:** `left_join(select(site_counts, site_name, n_site = n), by = "site_name")`
- **Problem:** `site_counts` has rows per `country` × `site_name`. Joining on `site_name` alone cross-matches German sites to French counts and vice versa if any site name appears in both countries. Corrupts `n_site` used for figure y-axis ordering.
- **Fix:** `left_join(select(site_counts, country, site_name, n_site = n), by = c("country", "site_name"))`
- **Affects:** Figure @fig-sites (site ordering, point sizes)

#### □ [CRIT-003] readme.qmd, line 86 — **`sign_pct()` returns `numeric(0)` for missing sign-country combos**
- **Agent(s):** correctness ★★ data-handling ★★ robustness ★★ statistics ★★
- **Code:** `round(100 * filter(sign_freq, country == ctry, sign == s)$prop)`
- **Problem:** When no row matches (e.g., `vulva` in Germany), `filter()` returns 0-row tibble, `$prop` → `numeric(0)`, `round(100 * numeric(0))` → `numeric(0)` (length 0). Inline in text: `` `r notch_de`% `` renders as empty string or "numeric(0)%" — silently corrupts manuscript.
- **Fix:** 
```r
sign_pct <- function(ctry, s) {
  row <- filter(sign_freq, country == ctry, sign == s)
  if (nrow(row) == 0) return(NA_integer_)
  round(100 * row$prop)
}
```
And guard inline: `` `r ifelse(is.na(vulva_fr), "not observed", paste0(vulva_fr, "%"))` ``
- **Affects:** All 7 inline percentages in Conclusion (notch_de, notch_fr, line_de, line_fr, obline_de, obline_fr, vulva_fr)

#### □ [CRIT-004] readme.qmd, line 30 — **Non-deterministic deduplication on `site_name` + `object_type`**
- **Agent(s):** robustness ★★ statistics ★★ data-handling ★★ correctness
- **Code:** `distinct(site_name, object_type, .keep_all = TRUE)`
- **Problem:** Keeps first row for each `(site_name, object_type)` arbitrarily. If multiple objects of same type at same site have different sign patterns, one is silently selected and others discarded. Row order from remote CSV is not guaranteed stable.
- **Fix:** Make deterministic: `arrange(site_name, object_type, <tiebreaker>) |> distinct(site_name, object_type, .keep_all = TRUE)` or verify duplicates are exact: `distinct(.keep_all = TRUE)` (all columns). Document the rule.
- **Affects:** All downstream analyses (sample size, sign frequencies, site counts, figure, inline values)

#### □ [CRIT-005] readme.qmd, line 10 — **External data fetch without verification, caching, or error handling**
- **Agent(s):** robustness ★★ reproducibility ★★ data-handling
- **Code:** `signbase <- read_csv("https://ndownloader.figshare.com/files/24118688", ...)`
- **Problem:** Hardcoded URL with no checksum, no local fallback, no timeout/retry, no post-load validation (row count, expected columns). If URL changes/rotates, analysis uses different data silently. Network failure = render failure.
- **Fix:** (1) Download once to `data/raw/signbase_v1.0.csv`; (2) Record SHA256 in `DATA_CHECKSUMS.txt`; (3) Read from local path; (4) Add `stopifnot(nrow(signbase) > 0, all(c("country","site_name","object_type") %in% names(signbase)))`.
- **Affects:** Every reported number (sample sizes, proportions, figure, conclusion percentages)

#### □ [CRIT-006] readme.qmd, line 33 — **Positional column range `line:other` in `pivot_longer`**
- **Agent(s):** robustness ★★ data-handling ★★ correctness ★★ style
- **Code:** `pivot_longer(line:other, names_to = "sign", values_to = "present")`
- **Problem:** Selects columns by position, not name. If column order changes (new columns inserted, reordering), wrong columns included/excluded. No validation that selected columns are binary (0/1).
- **Fix:** Use explicit sign column vector from codebook: `sign_cols <- c("notch","line","dot","cross","obline","vulva","other")` then `pivot_longer(all_of(sign_cols), ...)`. Add `stopifnot(all(sign_cols %in% names(signs_focus)))`.
- **Affects:** `sign_freq`, `top10`, figure @fig-sites, conclusion percentages

#### □ [CRIT-007] readme.qmd, line 61 — **`all_of(top10)` assumes all top-10 signs exist as columns**
- **Agent(s):** robustness ★★ data-handling
- **Code:** `pivot_longer(all_of(top10), ...)`
- **Problem:** `top10` derived from `sign_freq` (long format). If any top-10 sign missing from `signs_focus` columns (e.g., only present in filtered-out countries), `all_of()` errors at render time.
- **Fix:** `top10_valid <- intersect(top10, names(signs_focus))`; warn if `length(top10_valid) < 10`; use `all_of(top10_valid)`.
- **Affects:** Figure @fig-sites (render failure or missing signs)

#### □ [CRIT-008] readme.qmd, lines 29/43/63 — **No row-count validation after critical filters**
- **Agent(s):** robustness
- **Code:** `filter(country %in% c("Germany", "France"))` → no `nrow > 0` check; `count()` → no check; `filter(n > 0)` → no check
- **Problem:** If source data lacks Germany/France, or all sign counts zero, downstream code runs on empty data frames. `slice_max` errors; `mean()` returns `NaN`; inline values become `NaN`/`NA` in text.
- **Fix:** After each critical transformation: `stopifnot(nrow(signs_focus) > 0, "No DE/FR records")`, `stopifnot(nrow(site_signs) > 0, "No sign-site combos")`.
- **Affects:** All reported numbers

#### □ [CRIT-009] readme.qmd, line 34 — **`na.rm = TRUE` in proportion calculation without documented missing-data policy**
- **Agent(s):** statistics ★★ data-handling
- **Code:** `summarise(prop = mean(present, na.rm = TRUE), n_present = sum(present, na.rm = TRUE), ...)`
- **Problem:** Excludes `NA` from both numerator and denominator. If `NA` = "not assessed" (vs 0 = "absent"), proportions are inflated (denominator = assessed objects only). Methods section does not specify missing-data semantics.
- **Fix:** Verify codebook: if `NA` = absent, recode `NA → 0` before pivot; if `NA` = not assessed, compute `prop = sum(present, na.rm=TRUE) / sum(!is.na(present))` explicitly and report assessment rate per sign.
- **Affects:** All sign proportions in Results, Discussion, Conclusion

#### □ [CRIT-010] readme.qmd, line 3 — **Dynamic `date: today` breaks bitwise reproducibility**
- **Agent(s):** reproducibility
- **Code:** `date: today` (YAML)
- **Problem:** Rendered document date changes every render. Output file differs even if code/data unchanged.
- **Fix:** Use fixed date: `date: "2024-01-15"` or parameterize.
- **Affects:** Document metadata (minor)

#### □ [CRIT-011] readme.qmd, lines 39/47/98 — **Arbitrary tie-breaking with `with_ties = FALSE`**
- **Agent(s):** robustness ★★ data-handling ★★ correctness
- **Code:** `slice_max(n, n = 10, with_ties = FALSE)` (top10), `slice_max(n, n = 1, by = country, with_ties = FALSE)` (top_site), `slice_max(n, n = 3, with_ties = FALSE)` (top3_de)
- **Problem:** When ties exist at rank boundaries, arbitrary first-row selection makes results unstable under small data changes. Top-10 signs, top site per country, and top-3 German sites all affected.
- **Fix:** Use `with_ties = TRUE` and handle ties explicitly (e.g., `arrange(sign)` before `slice_max` for alphabetical tiebreaker), or document tie-break rule.
- **Affects:** Figure @fig-sites (which signs appear), Discussion (top site share claims), Conclusion (top 3 sites list and percentage)

#### □ [CRIT-012] readme.qmd, lines 23/54/78/103 — **Unguarded inline R expressions in prose**
- **Agent(s):** robustness
- **Code:** Multiple `` `r ...` `` (e.g., `r nrow(signbase)`, `r sign_freq |> filter(...) |> slice_max(...) |> pull(sign)`)
- **Problem:** Inline code assumes upstream objects exist with expected structure. If any chunk fails or produces unexpected output (empty `sign_freq`), inline errors at render time with no trace to source. No guards on `pull()` returning length-0.
- **Fix:** Compute all inline values in a dedicated `echo: false` chunk with validation, store in variables, reference variables in text: `` `r ifelse(is.na(x), "N/A", x)` ``.
- **Affects:** All inline text in Methods, Results, Discussion, Conclusion

#### □ [CRIT-013] readme.qmd, line 9 — **No package version lockfile / environment specification**
- **Agent(s):** reproducibility ★★ style
- **Code:** `library(tidyverse)` (no versions)
- **Problem:** `tidyverse` loads dplyr, ggplot2, tidyr, readr, etc. — all evolve. `.by` argument in `summarise()` requires dplyr ≥ 1.1.0 (2023). `slice_max(by=)` same. No `renv.lock`, `sessionInfo()`, Dockerfile.
- **Fix:** Run `renv::init(); renv::snapshot()` to create `renv.lock`; commit it. Or add `sessionInfo()` chunk at document end.
- **Affects:** Computational reproducibility (figure rendering, dplyr behavior)

#### □ [CRIT-014] readme.qmd, line 67 — **`fct_reorder(site_name, n_site)` after cross-country join duplication**
- **Agent(s):** robustness
- **Code:** `fct_reorder(site_name, n_site)` after `left_join(by = "site_name")`
- **Problem:** If CRIT-002 (join on site_name only) causes cross-country duplication, `n_site` has multiple values per `site_name`, `fct_reorder` behavior undefined, site ordering in figure wrong.
- **Fix:** Fix join key first (CRIT-002); then `fct_reorder` is safe.
- **Affects:** Figure @fig-sites y-axis order

#### □ [CRIT-015] readme.qmd, line 30 — **Deduplication assumes at most one object per type per site**
- **Agent(s):** data-handling ★★ statistics
- **Code:** `distinct(site_name, object_type, .keep_all = TRUE)`
- **Problem:** If multiple objects of same type at same site exist with different sign patterns, this collapses them to one arbitrary record. The design says "removed duplicate records" but does not define duplicate criterion.
- **Fix:** Define duplicates explicitly. If unique object ID exists: `distinct(object_id, .keep_all = TRUE)`. Else deduplicate on all sign columns: `distinct(site_name, object_type, line, notch, dot, cross, obline, vulva, other, .keep_all = TRUE)`. Verify no non-exact duplicates exist.
- **Affects:** All downstream analyses

#### □ [CRIT-016] readme.qmd, line 33 — **No binary validation on sign columns**
- **Agent(s):** data-handling
- **Code:** `pivot_longer(line:other, ...)` → `summarise(prop = mean(present))`
- **Problem:** Assumes sign columns are binary (0/1). If any contain other values (2, -1, NA, text), proportions are meaningless. No `stopifnot(all(signs_focus[sign_cols] %in% c(0,1,NA)))`.
- **Fix:** Add validation: `sign_cols <- setdiff(names(signs_focus), c("country","site_name","object_type")); stopifnot(all(sapply(signs_focus[sign_cols], function(x) all(x %in% c(0,1,NA)))))`.
- **Affects:** All sign proportions

#### □ [CRIT-017] readme.qmd, line 33 — **Sign column completeness/order not verified (UNVERIFIABLE)**
- **Agent(s):** data-handling
- **Code:** `pivot_longer(line:other, ...)`
- **Problem:** Without access to actual data, cannot verify that `line:other` captures all sign columns and no non-sign columns. Codebook not provided.
- **Fix:** Provide data dictionary/codebook listing expected sign columns. Validate on load: `expected_signs <- c("notch","line","dot","cross","obline","vulva","other"); stopifnot(all(expected_signs %in% names(signbase)))`.
- **Affects:** All sign analyses (flagged UNVERIFIABLE — requires manual check)

#### □ [CRIT-018] readme.qmd, line 30 — **Deduplication key uniqueness not verified (UNVERIFIABLE)**
- **Agent(s):** data-handling
- **Code:** `distinct(site_name, object_type, .keep_all = TRUE)`
- **Problem:** Cannot verify without data whether `(site_name, object_type)` uniquely identifies objects. If not, silent data loss.
- **Fix:** Provide codebook with object identifier. Or check in data: `signbase |> count(site_name, object_type) |> filter(n > 1) |> nrow()`.
- **Affects:** Analytic sample size (flagged UNVERIFIABLE — requires manual check)

#### □ [CRIT-019] readme.qmd, line 33 — **Raw missing value codes undocumented**
- **Agent(s):** data-handling
- **Code:** `summarise(prop = mean(present, na.rm = TRUE), ...)`
- **Problem:** No documentation of what `NA` means in sign columns (absent vs not assessed vs data entry gap). No check for sentinel values (e.g., -99, 99) that would be treated as present.
- **Fix:** Document missing-value semantics in codebook. Add `stopifnot(!any(signs_focus[sign_cols] < 0, na.rm = TRUE))` to catch negative sentinels.
- **Affects:** Proportion estimates

---

### WARNINGS (should resolve before submission)

#### □ [WARN-001] readme.qmd, line 10 — **Hardcoded URL is single point of failure**
- **Agent(s):** robustness reproducibility data-handling
- **Fix:** Parameterize URL, add to config, document DOI (10.6084/m9.figshare.12000018.v1)

#### □ [WARN-002] readme.qmd — **No data dictionary / codebook**
- **Agent(s):** reproducibility data-handling
- **Fix:** Add `data/codebook.md` documenting expected columns, types, value ranges, SignBase version

#### □ [WARN-003] readme.qmd — **No master/runner script, no README with execution instructions**
- **Agent(s):** reproducibility
- **Fix:** Add `README.md` with: `quarto render readme.qmd`, renv restore, data download, expected outputs

#### □ [WARN-004] readme.qmd, line 58 — **Figure missing `fig-width`**
- **Agent(s):** robustness
- **Fix:** Add `#| fig-width: 10` (or appropriate) alongside `fig-height: 8`

#### □ [WARN-005] readme.qmd, line 8-12 — **No script-level purpose comment**
- **Agent(s):** reproducibility
- **Fix:** Add comment block: `# Purpose: Reproduce SignBase v1.0 analysis... Inputs: ... Outputs: ...`

#### □ [WARN-006] readme.qmd, lines 88–94 — **DRY violation: 7 near-identical `sign_pct()` calls**
- **Agent(s):** style
- **Fix:** Use `expand_grid(country, sign) |> mutate(pct = map2_int(country, sign, sign_pct)) |> pivot_wider(...)`

#### □ [WARN-007] readme.qmd, line 54 & 86 — **DRY violation: inline pipeline duplicates `sign_pct` logic**
- **Agent(s):** style
- **Fix:** Compute `top_sign_de`, `top_sign_fr` in a chunk, reference variables in prose

#### □ [WARN-008] readme.qmd, line 30 — **`distinct()` variable name `signs_focus` misleading**
- **Agent(s):** style
- **Fix:** Rename to `signs_dedup_site_type` or clarify deduplication key in comment

#### □ [WARN-009] readme.qmd, line 44 — **`n` → `n_site` rename at use-site, not definition**
- **Agent(s):** style
- **Fix:** `site_counts <- ... |> mutate(n_site = n, share = n / sum(n), .by = country) |> select(-n)`

#### □ [WARN-010] readme.qmd, line 66–73 — **ggplot pipeline formatting**
- **Agent(s):** style
- **Fix:** Add blank lines between layers, align `+` operators (see style report)

#### □ [WARN-011] readme.qmd, line 54 — **Inline pipeline in prose too complex**
- **Agent(s):** style
- **Fix:** Pre-compute in chunk, reference variable in prose

#### □ [WARN-012] readme.qmd — **No R version declared**
- **Agent(s):** reproducibility
- **Fix:** Add `r_version: "4.4.0"` to `renv.lock` or comment

#### □ [WARN-013] readme.qmd — **No computational requirements documented**
- **Agent(s):** reproducibility
- **Fix:** Add comment: `# Expected runtime: ~5s; requires internet; ~50 MB RAM`

#### □ [WARN-014] readme.qmd, line 47 — **`slice_max(by=)` requires dplyr ≥ 1.1.0**
- **Agent(s):** statistics correctness
- **Fix:** Use pre-1.1.0 equivalent: `group_by(country) |> slice_max(n, n=1) |> ungroup()` for portability

#### □ [WARN-015] readme.qmd, lines 54/103 — **No formal statistical test of repertoire difference**
- **Agent(s):** statistics
- **Fix:** Add chi-square/permutation test, or clarify comparison is descriptive only

#### □ [WARN-016] readme.qmd, line 11 — **Global `theme_set()` leaks in multi-figure docs**
- **Agent(s):** style
- **Fix:** Acceptable for single-figure doc; note in comment

#### □ [WARN-017] readme.qmd, line 78 — **Discussion acknowledges non-independence but analysis doesn't reflect it**
- **Agent(s):** statistics
- **Fix:** Add site-level analysis (mixed-effects model) or clarify descriptive only

#### □ [WARN-018] readme.qmd — **No verification step comparing rendered values to reference**
- **Agent(s):** reproducibility
- **Fix:** Add known-good reference values (e.g., `reference/sign_freq_ref.csv`) and comparison chunk

#### □ [WARN-019] readme.qmd — **No local cache of downloaded data**
- **Agent(s):** robustness
- **Fix:** Use `pins` or manual download + `read_csv("data/signbase.csv")`

#### □ [WARN-020] readme.qmd — **No package version pinning for long-term reproducibility**
- **Agent(s):** robustness
- **Fix:** `renv::record()` or `dependencies.R`

#### □ [WARN-021] readme.qmd — **Three-valued logic collapse in `sign_pct`**
- **Agent(s):** robustness
- **Fix:** Return `NA_integer_` on no match, guard inline (see CRIT-003)

#### □ [WARN-022] readme.qmd — **Three-valued logic collapse in `pivot_longer(line:other)`**
- **Agent(s):** robustness
- **Fix:** Explicit `sign_cols` vector validated against `names(signs_focus)`

#### □ [WARN-023] readme.qmd — **Three-valued logic collapse in `left_join(by="site_name")`**
- **Agent(s):** robustness
- **Fix:** Join on `c("country", "site_name")`; add duplicate check

#### □ [WARN-024] readme.qmd — **Prose fields assembled at run time without fail-closed guards**
- **Agent(s):** robustness
- **Fix:** Each inline field computed with validation, returning `NA` + flag rather than `numeric(0)`

---

## 4. ★★ Cross-Agent Agreement Items (Highest Confidence)

| Issue | Agents Agreeing | Description |
|-------|----------------|-------------|
| **Hallucinated `show_col_types`** | style ★★ correctness robustness | `read_csv()` argument doesn't exist — script fails |
| **Join on `site_name` only** | correctness ★★ data-handling ★★ robustness | Cross-country corruption of `n_site` |
| **`sign_pct()` returns `numeric(0)`** | correctness ★★ data-handling ★★ robustness ★★ statistics ★★ | Silent manuscript corruption |
| **Non-deterministic `distinct()`** | robustness ★★ statistics ★★ data-handling ★★ correctness | Arbitrary row retention changes all results |
| **External data no verification** | robustness ★★ reproducibility ★★ data-handling | Every number depends on unverified fetch |
| **Positional `line:other` range** | robustness ★★ data-handling ★★ correctness ★★ style | Wrong columns if schema drifts |
| **Arbitrary tie-breaking** | robustness ★★ data-handling ★★ correctness | Top-10, top site, top-3 unstable |
| **No CIs for proportions** | statistics ★★ data-handling | Point estimates only, no uncertainty |
| **`na.rm=TRUE` without policy** | statistics ★★ data-handling | Missing-data semantics undefined |
| **No package version lockfile** | reproducibility ★★ style | Computational irreproducibility |
| **Unguarded inline expressions** | robustness ★★ correctness | Garbage in manuscript on upstream failure |
| **Three-valued logic collapses** | robustness ★★ statistics | `numeric(0)`/`NA`/empty indistinguishable in prose |

---

## 5. Per-Script Scorecard

| Script | Lines | Critical | Warning | Info | Agents Flagging | Grade |
|--------|-------|----------|---------|------|-----------------|-------|
| readme.qmd | ~117 | 19 | 24 | 0 | all 6 | **F** |

**Grade rules:** A: 0 CRIT, ≤1 WARN; B: 0 CRIT, 2-3 WARN; C: 1 CRIT or >3 WARN; D: 2-3 CRIT; F: >3 CRIT

---

## 6. Top 3 Most Impactful Issues

### 1. **Hallucinated `read_csv` argument (CRIT-001)**
The script **will not run** — `readr::read_csv()` has no `show_col_types` argument. This is a hard failure at the first line of code. Every downstream result is uncomputable until fixed.

### 2. **`sign_pct()` returns `numeric(0)` for missing signs (CRIT-003)**
Seven inline percentage values in the Conclusion paragraph render as empty strings or "numeric(0)%" when the sign is absent in that country (e.g., `vulva` in Germany). The manuscript makes specific numerical claims ("oblique lines occur on `r obline_fr`% of French objects") that silently become malformed text. This is a **three-valued logic collapse** where "not assessed" becomes "missing output" rather than explicit 0% or "not observed".

### 3. **Non-deterministic deduplication + external data without verification (CRIT-004 + CRIT-005)**
The analytic sample `signs_focus` is constructed by:
- Fetching data from an external URL with no checksum, version pin, or local cache
- Deduplicating on `(site_name, object_type)` keeping the first row arbitrarily (row order from remote CSV not guaranteed)

This means **every reported number** (sample sizes, proportions, site counts, figure, conclusion percentages) is non-reproducible and potentially biased by arbitrary row selection. A re-download or CSV re-ordering changes results silently.

---

## 7. Agent Detail Reports

### 7.1 Correctness & Logic (review-code-correctness)
- **CRITICAL:** Join on `site_name` only (CRIT-CORR-001); Non-deterministic `distinct()` (CRIT-CORR-002)
- **WARNINGS:** Fragile `pivot_longer(line:other)`; `na.rm=TRUE` without missing-data policy; `slice_max` tie-breaking; `sign_pct()` returns `numeric(0)`; `all_of(top10)` assumes columns exist
- **INFO:** External URL no version pin; `filter(n>0)` before join compounds join issue

### 7.2 Robustness (review-code-robustness)
- **CRITICAL:** URL no error handling; `pivot_longer(line:other)` positional; `all_of(top10)` fragile; Join on `site_name` only; `sign_pct()` `numeric(0)`; No row-count validation after filters
- **WARNINGS:** Arbitrary `distinct()` keep; `slice_max` ties (3 locations); Unguarded inline expressions; `fct_reorder` after dup join
- **INFO:** No local cache; No `fig-width`; No package version pinning
- **3VL COLLAPSES:** `sign_pct`, `pivot_longer`, `left_join` — all need companion guards

### 7.3 Statistical Implementation (review-code-statistics)
- **CRITICAL:** `distinct()` arbitrary keep biases proportions; `na.rm=TRUE` inflates proportions if NA=not-assessed
- **WARNINGS:** No CIs for proportions; `sign_pct()` fragile; `distinct()` arbitrary; Top-10 selection pooled not per-country; No site-level inference despite acknowledged non-independence
- **INFO:** `slice_max(by=)` requires dplyr≥1.1.0; No formal test of repertoire difference

### 7.4 Reproducibility (review-code-reproducibility)
- **CRITICAL:** External data no local copy/hash; Non-deterministic `distinct()`; No network error handling
- **WARNINGS:** No lockfile; No data dictionary; `slice_max` ties; No runner/README; Dynamic `date: today`
- **INFO:** No R version; No computational requirements; No purpose comment
- **Grade: D** (significant gaps)

### 7.5 Style & AI Anti-Patterns (review-code-style)
- **AI ANTI-PATTERNS (CRITICAL):** `read_csv(show_col_types=FALSE)` — argument doesn't exist; `pivot_longer(line:other)` — hallucinated column range
- **DRY VIOLATIONS:** 7 `sign_pct()` calls; Inline pipeline duplicates function logic
- **NAMING:** `signs_focus` misleading; `n`→`n_site` rename at use-site
- **READABILITY:** ggplot formatting; Inline pipeline too complex
- **GOOD:** Modern `.by` syntax; `all_of()` correct; `knitr::combine_words()` good

### 7.6 Data Handling (review-code-data-handling)
- **CRITICAL:** Join missing `country`; `sign_pct()` empty return; Deduplication key assumes one object/type/site; Positional `line:other` fragile
- **WARNINGS:** Arbitrary tie-breaking; No binary validation; External URL no version pin
- **UNVERIFIABLE:** Deduplication key uniqueness; Sign column completeness/order
- **MISSING VALUE AUDIT:** Raw missing codes undocumented; No sentinel value checks
- **PATH PORTABILITY:** No absolute paths, but external HTTPS read is risk

---

## 8. Next Steps

**CRITICAL findings need fixing before these results can be trusted. Default path: I apply the fixes, then re-review is required (the affected reviewer agents re-adjudicate the fixed scripts; fixes are never self-certified).**

Proceed with fix-then-re-review, or do you want to handle the fixes yourself first?

---

## Output Files

- Consolidated report: `output/code-review/code-review-report-2026-11-16.md`
- Fix checklist: `output/code-review/fix-checklist-2026-11-16.md`
- Reviewed-scripts manifest: `output/code-review/reviewed-scripts-cr-20261116-archy459.json`

---

## Process Log Summary

- **Steps completed:** 5/5
- **Scripts reviewed:** 1
- **Total issues:** 58
- **Critical issues:** 19
- **Warnings:** 24
- **★★ cross-agent issues:** 12
- **Overall grade:** F
- **Verdict:** MAJOR ISSUES — DO NOT TRUST RESULTS