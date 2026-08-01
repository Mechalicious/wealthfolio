# RCA: Multi-GB Memory Spike After Holdings CSV Import

**Status:** Root cause identified (mechanism confirmed in code; awaiting confirmation from the
reporter's data)
**Date:** 2026-07-25
**Severity:** Critical — app becomes unusable, macOS kills the process
**Affected area:** Holdings CSV import → post-import refresh (market sync + valuation
recalculation)

---

## 1. Context

### 1.1 Reports

| Report | Symptom | Scale |
|---|---|---|
| Direct user report (email, 2026-07) | Imported **51 tickers** via holdings CSV into a brokerage (HOLDINGS-mode) account. Import succeeded; during the post-import "refresh" the app climbed to **44.9 GB** RAM, macOS raised "system has run out of application memory", force-quit required. Heavy resource usage recurs on **every** subsequent account add / import until restart. | 44.9 GB |
| [#1347](https://github.com/wealthfolio/wealthfolio/issues/1347) | Memory impact and timeouts/slowdowns when viewing 5+ years of history. | GBs |
| [PR #1218](https://github.com/wealthfolio/wealthfolio/pull/1218) (author's motivation) | ~3k activities over ~5y: slow rendering, **1.5–2 GB** peak RAM on full history update. | 1.5–2 GB |

All three share one pipeline; the 44.9 GB case additionally requires the date-range defect
described below. Related design review:
[`valuation-performance-review-and-target-design.md`](./valuation-performance-review-and-target-design.md).

### 1.2 Why 51 tickers cannot explain 44.9 GB on its own

The post-import refresh materialises roughly **1 KB per asset per day** of history (per-day
snapshot clones + forward-filled quote structs + FX maps — see §2.3). For a sane single-date
holdings snapshot the range is days-to-weeks: 51 tickers compute to **tens of MB**. Even a
snapshot backdated 15 years stays under ~300 MB per pipeline run. The reported magnitude
therefore requires the *date range itself* to be pathological. 44.9 GB is not a working-set
size; it is where macOS killed a process whose allocation was growing without bound.

---

## 2. Root Cause Analysis

Three defects compose. The first supplies the unbounded input; the second and third turn it
into unbounded memory.

### 2.1 Primary: snapshot dates are format-validated but never range-validated

- `check_holdings_import` (`apps/tauri/src/commands/portfolio.rs:1222-1228`) validates CSV
  dates with `NaiveDate::parse_from_str(&snapshot.date, "%Y-%m-%d")` — **format only**. Years
  `0224`, `1924`, `2924`, `9999` all parse successfully. There is no "not in the future" and no
  lower-bound check.
- The same gap exists in `save_manual_holdings` (`portfolio.rs:1085-1089`) and in
  `ManualSnapshotService` / `SnapshotService::save_manual_snapshot`
  (`crates/core/src/portfolio/snapshot/`): no path rejects an absurd `snapshot_date`.
- Holdings CSV semantics make a single typo consequential: *"Rows with the same date form one
  snapshot; multiple dates create multiple snapshots"* (`portfolio.rs`, `import_holdings_csv`
  doc comment). **One mangled year in one of 51 hand-edited rows silently creates a second
  snapshot at that bogus date.** The import reports success — matching the reporter's "import
  was good".

### 2.2 The valuation window is derived from stored dates, unclamped

`get_daily_holdings_snapshots` (`crates/core/src/portfolio/snapshot/snapshot_service.rs:1221-1250`)
derives the reconstruction window as:

```
start = MIN(snapshot_date)                       -- no floor
end   = MAX(snapshot_date).max(today)            -- no ceiling
```

The only guard is `start > end → return empty`. A snapshot at year 0224 sets
`start = 0224-07-20` (≈ 658,000 days to today); a snapshot at year 2924 sets
`end = 2924` (≈ 328,000 days). Either poisons the window for **every subsequent
recalculation** of that account.

### 2.3 The pipeline materialises O(days × assets) in memory

For the poisoned window, `calculate_valuation_history`
(`crates/core/src/portfolio/valuation/valuation_service.rs:1661`) allocates, simultaneously:

| Allocation | Source | ~658k-day window, 51 tickers |
|---|---|---|
| One deep-cloned `AccountStateSnapshot` **per calendar day** (all positions, ~380 B each) | `get_daily_holdings_snapshots`, `snapshot_service.rs:1313` | ~13 GB |
| One forward-filled `Quote` struct (~6 heap `String`s) **per asset per day** | `fill_missing_quotes`, `crates/core/src/quotes/service.rs:2429-2435` | ~12 GB |
| Per-day FX maps, per-day cloned quote maps, output valuation rows | `valuation_service.rs:342-379, 1838-1844` | ~1–2 GB |

≈ **25–27 GB per pipeline run**, and the run never completes — allocation grows with the day
loop until the OS intervenes. `fetch_fx_rates_for_range` additionally performs a graph BFS per
(day × currency-pair), so the refresh also spins CPU for hours — matching "application was
trying to refresh data".

### 2.4 Amplifier: the import triggers TWO concurrent full pipelines

A holdings import triggers the refresh twice, concurrently:

1. `import_holdings_csv` / `save_manual_holdings` explicitly emit
   `PORTFOLIO_TRIGGER_RECALCULATE` (`portfolio.rs:1160-1170`) → `listeners.rs`
   `handle_portfolio_request` → market sync + `SnapshotRecalcMode::Full` +
   `ValuationRecalcMode::Full`. **`listeners.rs` spawns unconditionally — it has no
   `is_processing` guard** (`apps/tauri/src/listeners.rs:65`).
2. `SnapshotService::save_manual_snapshot` emits `DomainEvent::HoldingsChanged`
   (`snapshot_service.rs:1399-1405`) → domain-event queue → planner maps it with
   `since_date = None` (`apps/tauri/src/domain_events/planner.rs:118-136, 194-198`) →
   `run_portfolio_job` maps `None → Full` (`queue_worker.rs:317-324`). The queue worker's
   `is_processing` guard does not see pipeline (1).

Two concurrent ~25 GB runs ≈ **~50 GB attempted** — the reported 44.9 GB is where macOS killed
it. The double-trigger also explains why *sane* imports feel heavy (double market sync + double
recalc), and the persistence of the poisoned snapshot explains the recurrence: every later
import, and app startup's backfill check (`apps/tauri/src/lib.rs:188`), re-enters the poisoned
account.

### 2.5 Verification status

**Confirmed in code:** the validation gap; the unclamped window; the per-day materialisation
sizes; the double-trigger with no shared guard; the `since_date=None → Full` planner mapping.

**Reproduced:** pending — repro CSVs are prepared (§3).

**Inferred, awaiting reporter's data:** that their CSV actually contained a mangled date. The
arithmetic requires it (§1.2), but it has not yet been observed. Diagnostic for the reporter:

```sql
SELECT account_id, snapshot_date FROM holdings_snapshots ORDER BY snapshot_date ASC  LIMIT 3;
SELECT account_id, snapshot_date FROM holdings_snapshots ORDER BY snapshot_date DESC LIMIT 3;
```

Any date outside the user's real investing horizon confirms the diagnosis. If their dates are
clean, the same unclamped-window mechanism still holds but the bad date entered through another
source (broker sync, device sync) — the clamp in fix F2 covers all of them.

**Ruled out during analysis:** per-holding concurrency (the frontend sends one command with all
51 holdings); a health-check auto-fix loop (`rebuild_account_history` is user-initiated); the
net-worth quote grid (alternative assets only).

---

## 3. Reproduction

Two CSVs (51 tickers + `$CASH`, all dated `2026-07-20`, except **one row — DIS — with a
mangled year**), matching the holdings CSV wizard format:

| File | Bad row | Expected |
|---|---|---|
| `holdings-repro-mild.csv` | `1924-07-20,DIS,60,98.75,USD` | ~37k-day window: minutes-long refresh, multi-GB spike, survivable. Validate the mechanism with this first. |
| `holdings-repro-oom.csv` | `0224-07-20,DIS,60,98.75,USD` | ~658k-day window: runaway allocation until the macOS out-of-memory dialog. The reporter's scenario. |

Steps: back up the DB → new HOLDINGS-mode account → import CSV via the holdings wizard
(observe: **no validation error on the mangled row**) → watch Activity Monitor as the
post-import refresh starts. Recovery:
`DELETE FROM holdings_snapshots WHERE snapshot_date < '1990-01-01';` then recalculate.

---

## 4. Proposed Fixes

In priority order. F1–F3 are small, independent patches; F4 is the structural fix tracked in
the design document.

### F1 — Range-validate snapshot dates at every write path (closes the door)

Reject `snapshot_date > today` and `snapshot_date < 1900-01-01` (constant, with a clear user
message) in:
- `check_holdings_import` (surface as a per-row validation error in the wizard),
- `import_holdings_csv` / `save_manual_holdings` (defence in depth at the command layer),
- `SnapshotService::save_manual_snapshot` (covers broker import, device sync, and any future
  caller).

### F2 — Clamp the valuation window defensively (contains any bad date)

In `get_daily_holdings_snapshots` (and the valuation range derivation), clamp to
`[max(earliest_snapshot, FLOOR), min(latest_snapshot, today)]` and **log loudly** when a stored
date falls outside the clamp instead of silently walking the range. This protects against bad
dates from *any* source, past or future, including pre-existing poisoned rows.

### F3 — Deduplicate the post-import trigger (halves every import's cost)

Remove the explicit `emit_portfolio_trigger_recalculate` from `import_holdings_csv` and
`save_manual_holdings` — the `HoldingsChanged` domain event already schedules the job through
the guarded queue worker. Alternatively (or additionally), give `listeners.rs` the same
`is_processing` guard so direct Tauri triggers and queue jobs cannot run the pipeline
concurrently.

Follow-on in the same area: the planner should map `HoldingsChanged` / `ManualSnapshotSaved` /
`AccountsChanged` to a since-date (the snapshot/import date) instead of `None → Full`
(`planner.rs:194-198`).

### F4 — Structural: stop materialising O(days × assets) (design doc WS-B)

Even with sane dates the refresh allocates ~1 KB per asset-day. The design document's WS-B
(interval-based valuation, `(asset, date) → close` numeric map shared across accounts, bounded
account fan-out instead of unbounded `join_all` in `listeners.rs:342`) reduces this to
O(assets) working memory and directly addresses #1347 and the PR #1218 author's 1.5–2 GB
report. See
[`valuation-performance-review-and-target-design.md`](./valuation-performance-review-and-target-design.md),
§4.3.

### Workaround for affected users (no release required)

Delete the bad-dated snapshot row (SQL above, or the snapshot UI once visible) and trigger a
recalculation. Memory returns to normal immediately; the poisoned row is the persistent state
that makes the problem recur.

---

## 5. Timeline / Links

- [PR #1218](https://github.com/wealthfolio/wealthfolio/pull/1218) — snapshot bloat fix; same
  pipeline, sane-date magnitude (1.5–2 GB).
- [#1347](https://github.com/wealthfolio/wealthfolio/issues/1347) — 5+ year history memory and
  timeout reports.
- Direct user report (2026-07) — 44.9 GB spike on 51-ticker holdings CSV import; this RCA.
- [`valuation-performance-review-and-target-design.md`](./valuation-performance-review-and-target-design.md)
  — full architecture review; §2.2 (compute hot spots) and §4.3 (WS-B) are the structural
  context for F4.
