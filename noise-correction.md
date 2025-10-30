## Noise Correction in Reporting (Overview)

This document explains how “noise correction” (a.k.a. post-processing/consistency correction) works for reporting outputs. The logic lives primarily in `src/main/python/wfa/measurement/reporting/postprocessing/report/report.py` and relies on the `noiseninja` library to formulate and solve a constrained optimization problem that adjusts noisy measurements to be mutually consistent.

### What inputs are corrected

Measurements come in three groups per metric (e.g., MRC, AMI):
- **Weekly cumulative reach**: time series per EDP combination
- **Whole campaign measurements**: reach, k-reach (by frequency), impressions
- **Weekly non-cumulative measurements**: reach, k-reach (by frequency), impressions

Each measurement has:
- **value**: mean
- **sigma**: standard deviation (uncertainty)
- **name**: unique identifier

These are organized in:
- `MetricReport`: per-metric container exposing getters for the above
- `Report`: aggregates multiple `MetricReport`s and inter-metric relationships

### High-level flow

1. Build a `Report` from input `MetricReport`s and metric subset relationships.
2. Call `Report.get_corrected_report()`.
3. `Report.to_set_measurement_spec()` encodes all measurements and constraints into a `noiseninja.SetMeasurementsSpec`.
4. `noiseninja.Solver(spec).solve_and_translate()` runs a QP/convex solver and returns an adjusted solution plus status.
5. `Report.report_from_solution(solution)` reconstructs a fully corrected `Report` from the solution.
6. Large corrections are logged and captured in the `ReportPostProcessorResult`.
7. Pre-/post-correction data quality summaries are included in the result.

### Key normalization detail

- Before adding measurements to the spec, sigmas are normalized by the max sigma across all measurements so that the optimization is well-scaled:
  - `normalized_sigma = max(sigma / max_sigma, MIN_STANDARD_VARIATION_RATIO)`
  - If `sigma == 0`, it stays `0` (the value is treated as fixed/unchanged).

### Constraints encoded (what “correct” means)

The `Report` converts the domain rules into constraints on set sizes via helper methods that add constraints to `SetMeasurementsSpec`:

- Subset relations within a metric and across metrics
  - Example: child metric reach ≤ parent metric reach for the same EDP combination and period
- Cover relations (sum of subsets ≥ union)
  - For a union EDP combination, the sum of singles must be at least the union
- Overlap constraints via “ordered sets” inequalities
  - Derived from set algebra of smaller vs. larger collections of sets
- Cumulative monotonicity
  - Weekly cumulative reach must be non-decreasing over time for each EDP combination
- Cumulative vs. whole campaign equality
  - Final cumulative reach equals the whole campaign reach for the same EDP combination
- k-reach consistency
  - Reach = sum over `k` of k-reach
- Impression bounds
  - Impressions ≥ weighted sum over `k` of k-reach (weight = frequency `k`)
- Cross-metric ordering
  - If metric A is a parent of metric B, then all corresponding B measurements ≤ A

These are wired up through the following internal helpers in `Report`:
- `_add_cover_relations_to_spec`
- `_add_subset_relations_to_spec`
- `_add_overlap_relations_to_spec`
- `_add_cumulative_relations_to_spec`
- `_add_k_reach_and_reach_relations_to_spec`
- `_add_impression_relations_to_spec`
- `_add_reach_impression_relations_to_spec`
- `_add_k_reach_impression_relations_to_spec`
- `_add_cumulative_whole_campaign_relations_to_spec`
- `_add_metric_relations_to_spec`

Measurements are added to the spec via:
- `_add_weekly_cumulative_measurements_to_spec`
- `_add_whole_campaign_measurements_to_spec`
- `_add_weekly_non_cumulative_measurements_to_spec`

All of the above are orchestrated by:
- `_add_measurements_to_spec`
- `_add_set_relations_to_spec`

### Solving and reconstructing the corrected report

- The spec is solved by `noiseninja.Solver.solve_and_translate()` producing:
  - `solution`: corrected values for each indexed measurement
  - `status`: feasibility/optimality and residuals
- `Report.report_from_solution(solution)` builds a new `Report` by:
  - Mapping each measurement index back to its name
  - Creating `Measurement` objects with corrected values but original sigmas
  - Reassembling `MetricReport`s for:
    - weekly cumulative time series
    - whole campaign reach/k-reach/impressions
    - weekly non-cumulative reach/k-reach/impressions

### Large correction detection and logging

After solving, the original report and corrected report are compared per measurement:
- If `|corrected - original| > max(7 * sigma, LARGE_CORRECTION_THRESHOLD)`, the correction is considered “large.”
- Large corrections are logged and appended to `ReportPostProcessorResult.large_corrections`.

### Data quality summary (pre and post)

`Report.get_report_quality()` evaluates:
- **Zero-variance EDP consistency**
  - If an EDP combination has all `sigma == 0`, its measurements must be mutually consistent
- **Independence check for unions** (optional; requires `population_size > 0`)
  - Tests whether measured union reach is within a multi-σ bound of expected union (assuming conditional independence across EDPs), for both whole-campaign and weekly cumulative reaches

These statuses are returned as a `ReportQuality` proto, included pre- and post-correction in `ReportPostProcessorResult`.

### Special handling

- EDP combinations with all-zero σ are treated as fixed and must already be consistent; they are neither relaxed nor adjusted by the solver.
- For some single-EDP cumulative time series (e.g., certain TV measurements), monotonicity constraints can be disabled by listing the EDP key in `cumulative_inconsistency_allowed_edp_combinations`.

### Where to look in code

- Entry point for correction: `Report.get_corrected_report()`
- Spec construction: `Report.to_set_measurement_spec()` and all `_add_*_to_spec` helpers
- Sigma normalization: `Report._normalized_sigma`
- Solution translation: `Report.report_from_solution()` and `_metric_report_from_solution()`
- Data quality: `Report.get_report_quality()` and helpers
- Independence check math: `is_union_reach_consistent`

### External dependency: noiseninja

`noiseninja` provides the types and solver used for the correction:
- `Measurement`, `KReachMeasurements`, `MeasurementSet`, `OrderedSets`, `SetMeasurementsSpec`
- `Solver(spec).solve_and_translate()` performs the optimization and returns both a solution and a processor status (including feasibility and residuals).

### Outputs

Calling `Report.get_corrected_report()` returns:
- The corrected `Report`
- A `ReportPostProcessorResult` containing:
  - `updated_measurements` (reserved — current code populates the corrected report instead)
  - `status` (solver/processor status)
  - `pre_correction_quality` and `post_correction_quality`
  - `large_corrections` entries when applicable
