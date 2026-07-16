# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-file, client-only web calculator that computes **ГЭСВ** (Kazakhstan's annual effective interest rate, analogous to APR) for loans with an **equal-principal (равнодолевой) repayment schedule**. There is no backend, no build step, and no package manager — the entire application is `index.html` (HTML + CSS + vanilla JS in one `<script>` tag). `PRD.md` is the authoritative product/functional spec written in Russian; the UI itself is in Russian.

## Running and testing

There is no build system, package.json, or test suite. To work on this app:

- Open `index.html` directly in a browser, or serve it with any static server (e.g. `python3 -m http.server`) and open it.
- There is no linter or automated test command configured. Verify changes manually in a browser: enter loan parameters, click «Рассчитать» (Calculate), and check the ГЭСВ value plus the four control tiles (Погашено ОД / Остаток пула / Остаток бакета / Остаток долга) turn green.
- When changing calculation logic, sanity-check against the acceptance criteria in `PRD.md` §9, in particular: default params (1,000,000 / 20% / 12 mo, no grace, no fees) must produce all-green checks; principal sum must always equal loan amount even with ОД grace months; pool must fully drain to 0 by term end under all three pool-distribution methods.

## Architecture (all inside `index.html`)

The script is organized into clearly commented sections, in this order:

1. **Constants** — `KURBAN_AIT_KNOWN` (hardcoded confirmed Kurban Ait dates by year — update this dict as new dates are officially announced by ДУМК) and `PERIODICITY_INTERVAL` (maps repayment frequency labels to a month interval).
2. **Date utilities** — `addMonthsClamped`, `daysBetween`, `formatDateRu`, `formatMoney`.
3. **Kazakhstan holiday/business-day calendar** (`buildCalendar`, `getTransferableHolidays`, `getKurbanAitDate`, `nextWorkday`) — computes the non-working-day set per year including weekend-transfer rules (a fixed holiday landing on Sat/Sun pushes the next free workday to non-working too, and chains correctly for consecutive holidays like Nauryz Mar 21–23). Payment dates falling on a non-working day roll forward to the next workday; the disbursement date itself is never auto-adjusted.
4. **XIRR** (`xnpv`, `calcXIRR`) — bisection solver over actual (adjusted) cash-flow dates, no external math libraries. This is the ГЭСВ result.
5. **Calculation core** (`generateSchedule`, `validateInputs`, `resolveInterval`) — the actual amortization engine. Produces the month-by-month schedule and the four reconciliation checks (principal sum, pool residual, bucket residual, balance residual).
6. **UI state + rendering** (`monthsState`, `rebuildGraceTable`, `renderGraceTable`, `renderResults`, `doCalculate`, `exportCSV`) — DOM manipulation, grace-period-per-month table, CSV export, error display.
7. **Init** (`init()`) — wires up event listeners, called at the bottom of the script.

### Core domain concepts (see `PRD.md` §5 for full formulas)

- **balance**: principal outstanding at start of month.
- **due dates**: principal and interest have *independent* payment frequencies (`odPeriodicity`/`pctPeriodicity` → `dueOD(i)`/`duePct(i)`), each resolved via `resolveInterval`. The final month is always a forced due date regardless of interval.
- **grace (льгота)**: per-month flags (`graceOD`, `gracePct`) set by the user in the grace table. A grace month is excluded from the "remaining due dates" denominator, so principal/interest normally due that month gets redistributed across the remaining non-grace due dates instead of skipped.
- **bucket**: interest accrued but not yet due for payment (accumulates between interest due-dates when interest frequency is less than monthly).
- **pool**: interest that *was* due but deferred by a grace-по-% month; drained via one of three distribution methods (`poolMethod`): `nearest` (dump whole pool into the next non-grace interest due-date), `all` (spread evenly across all remaining non-grace due-dates, recalculated as new pool accumulates), `manual` (per-month weight 0–1 of the *current* pool balance; forced full drain on the last remaining due date regardless of weight).
- **Reconciliation checks**: after generating the schedule, `sumPrincipal` must equal loan amount, and `poolEnd`/`bucketEnd`/`balanceEnd` must be ~0 at term end (tolerance 0.5, currency units) — these are the four UI tiles and the main correctness signal when modifying the engine.

### Key constraints from the PRD (do not violate silently)

- No `localStorage`/`sessionStorage` — everything is in-memory for the session only, by design (preview-environment restriction, see PRD §8).
- No external runtime dependencies except Google Fonts (must degrade gracefully offline).
- Only equal-principal amortization is in scope; annuity/equal-payment schedules are explicitly out of scope (PRD §2).
- Term is bounded 1–60 months; validation lives in `validateInputs` and must produce human-readable Russian error messages (not raw exceptions), shown in `#error-block`.
- CSV export (`exportCSV`) must use `;` delimiter, comma decimal separator, and a UTF-8 BOM (`'﻿'`) so it opens correctly in Excel with Cyrillic text — see the `num()` helper and `formatDateRu`.
