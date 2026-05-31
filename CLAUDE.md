# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Running the app

Open `retirement-calculator.html` directly in a browser — no build step, server, or dependencies to install. The only external dependency is Chart.js 4.4.0, loaded from CDN.

## Architecture

The entire app is a single self-contained file (`retirement-calculator.html`) with inline `<style>` and `<script>` blocks. There are no modules, bundlers, or frameworks.

**JS is organized into these sections (in order):**

- **State** — `STATE` object holds `chart` (Chart.js instance), `lastResult` (last computed result object), and `calculated`.
- **Formatters** — pure functions: `formatCurrency`, `formatCurrencyShort`, `formatAge`, `formatMonths`, `formatDelta`.
- **Math** — `calcTarget(retirementExpenses, withdrawalRate)` computes the FV target using the 25× rule. `monthsToRetirement(PV, PMT, r, FV)` solves the future-value annuity formula analytically (falls back to linear for near-zero `r`).
- **Input handling** — `getInputs()` reads and validates all fields, throws an array of error objects on failure. `extraAmount` is hardcoded to `0` here (the UI field was removed).
- **Chart** — `buildChartData(result)` computes month-by-month portfolio values for both baseline and adjusted scenarios. `renderChart(result, force=false)` either does a smooth in-place update (`chart.update()`) when the chart already exists, or a full destroy-and-recreate when `force=true` (used by theme toggle). The custom `retirementLinesPlugin` draws dashed overlay lines for the retirement target (horizontal) and retirement dates (vertical) directly onto the canvas.
- **Sensitivity table** — `renderSensitivity(result)` builds 23 rows across `[-2000…+2000]` offsets. `selectSensitivityRow(offset)` recalculates for a clicked offset and calls `updateChartAdjusted` (keeps existing labels/x-range, updates only `datasets[1]`) plus updates the Adjusted and Difference cards directly.
- **Rendering** — `renderResults(result)` updates all three summary cards, calls `renderChart`, and calls `renderSensitivity`.
- **Calculate** — `calculate()` reads inputs, runs the math, stores to `STATE.lastResult`, and calls `renderResults`. Auto-triggered via a 400ms debounce on every `input` event. Runs once on page load with defaults (or from URL hash).
- **URL hash permalink** — inputs are base64-encoded JSON in the hash; `loadFromHash()` restores them on load.
- **Event wiring** — all listeners set up in a single `DOMContentLoaded` handler, including the spinner buttons (injected dynamically into each `.input-wrap`).

## Key design constraints

- **Chart update strategy**: `renderChart` must use `force=true` only for theme changes (to rebuild chart options with new colors). All other recalculations use the smooth in-place path (`force=false`). Breaking this will cause jarring full-redraw animations on every keystroke.
- **`extraAmount` is always 0** from `getInputs()`. The sensitivity table drives scenario comparison by calling `selectSensitivityRow(offset)`, which mutates `STATE.lastResult.extraAmount` and `PMT_adj` temporarily for the selected row.
- **Spinner buttons** are injected via JS (not HTML) in `DOMContentLoaded`. Per-field step sizes are defined in the `spinnerSteps` map. Adding a new number input requires adding its id there.
- **Theme colors** for the chart are hardcoded strings (not CSS vars) because Chart.js reads them at render time, not from the DOM. When adding new chart elements, match the `dark ?` ternary pattern already used.
