# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

Single-page, client-only calculator for **ГЭСВ** (Kazakhstani annual effective loan rate, analogous to APR) on loans with an **equal-principal ("равнодолевой")** repayment schedule. It computes the rate via XIRR over the actual (holiday-adjusted) cash flow dates, renders a full payment schedule, and exports it to CSV. Everything runs in the browser with no backend and no build step.

`PRD.md` is the authoritative functional spec (in Russian) — it defines every formula, edge case, and UI rule implemented in `index.html`. When behavior is ambiguous or you're changing calculation logic, check `PRD.md` first (especially §5 "Логика расчёта" for the calc engine and §6 "Календарь" for holiday handling) and keep the two in sync.

## Commands

There is no package manager, build step, linter, or test suite in this repo — it is one static HTML file.

- **Run/preview:** open `index.html` directly in a browser (or serve the directory with any static file server, e.g. `python3 -m http.server`). There is no dev server, no npm scripts, nothing to install.
- **Verify a change:** manually exercise the UI in a browser — set inputs, click "Рассчитать" (Calculate), check the four verification tiles turn green, and try "Экспорт CSV". There are no automated tests, so this manual pass is the only verification available.

## Code architecture

Everything lives in `index.html`: styles in a `<style>` block, markup in `<body>`, logic in a single `<script>` block at the bottom. The script is organized into clearly commented sections (Russian headers) in this order — preserve this structure when editing:

1. **Константы (Constants)** — `KURBAN_AIT_KNOWN` (officially announced dates for the floating Islamic holiday, must be updated by hand as new years are announced) and `PERIODICITY_INTERVAL` (maps repayment frequency to a month interval).
2. **Дата: утилиты (Date utilities)** — `addMonthsClamped`, `daysBetween`, `formatDateRu`, `formatMoney`, etc. Plain `Date` math, no library.
3. **Календарь нерабочих дней РК (Kazakhstan non-working day calendar)** — `getTransferableHolidays`, `getKurbanAitDate`, `buildCalendar`, `nextWorkday`. Builds a set of non-working dates per year (weekends + fixed holidays) and applies the "перенос" rule: a fixed holiday landing on a weekend pushes the following free workday to non-working too (handled as a chain, sorted ascending). Fixed holiday dates and the Constitution Day transition rule (effective from 2027, with 2026 as a no-holiday transition year) are spec'd in `PRD.md` §6.2.
4. **XIRR** — `xnpv` / `calcXIRR`. Pure bisection solver (interval expands adaptively, precision 1e-9, up to 300 iterations) — deliberately dependency-free per the PRD's non-functional requirements.
5. **Расчётное ядро (Calculation engine)** — `generateSchedule(params)` is the core: validates inputs, builds holiday-adjusted month dates, computes due/grace flags per month, then loops month-by-month tracking three running state variables:
   - `balance` — principal outstanding.
   - `bucket` — accrued-but-not-yet-due interest (used when payment periodicity is less frequent than monthly).
   - `pool` — interest deferred by a grace period on interest, awaiting distribution back into future payments per the selected `poolMethod` (`nearest` / `all` / `manual`).
   Returns `{ rows, checks, gesv }` where `checks` are the four reconciliation values (sum of principal repaid, pool/bucket/balance end-of-term residuals) that must all resolve to ~0 (except principal sum, which must equal the loan amount) — this is the calculation's internal self-check, not a UI nicety.
6. **UI: состояние / рендер / init** — `monthsState` holds the per-month grace-period/weight table; `readParams()` pulls form values into the params object consumed by `generateSchedule`; `renderResults`/`exportCSV` render the schedule table and CSV respectively. `doCalculate` wires validation errors to `showError`. The grace-period table (`rebuildGraceTable`) is rebuilt whenever the term (`f-term`) changes.

### Key domain concepts (see PRD.md for full formulas)

- **Equal-principal, not annuity**: principal per due date = remaining balance / number of remaining non-grace due dates — recalculated dynamically as grace periods pull dates out of the denominator.
- **Principal and interest have independent schedules**: separate periodicity (`odPeriodicity`/`pctPeriodicity`: monthly/quarterly/semiannual/annual/end-of-term) and independent grace flags per month, per leg.
- **Grace period pooling**: interest deferred by a `gracePct` month accumulates in `pool` and is later redistributed by one of three methods (`nearest`, `all`, `manual` — weight is a fraction of the *current* pool balance, and the last remaining interest due-date force-clears the pool regardless of weight).
- **Date adjustment**: computed month dates are shifted forward to the next workday per the KZ calendar; the issue date itself is never adjusted (see PRD §6.5).
- **GESV = XIRR** of the cash flow: `-（amount − issuance fee）` at t=0, then `totalPayment(i)` at each adjusted payment date.

## Conventions

- All UI text, comments, and section headers are in **Russian** — keep new code consistent with this (comments explaining non-obvious business rules, e.g. holiday transfer edge cases, are in Russian like the rest of the file).
- Vanilla JS, `'use strict'`, no framework, no external runtime dependency besides the Google Fonts stylesheet link (must degrade gracefully offline).
- No `localStorage`/`sessionStorage` — state is memory-only for the session, by design (per PRD §8).
- Money formatting via `formatMoney`/`ru-RU` locale; dates displayed as `dd.mm.yyyy` via `formatDateRu`.
- Changes to the calculation engine (`generateSchedule` and its helpers) should keep the four verification tiles resolving to zero (except principal sum) across edge cases: term = 1 and term = 60, 0% rate, all months in grace, zero fees — these are the acceptance criteria in `PRD.md` §9.
