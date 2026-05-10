# Implementation Status — Claims 16 · 17 (Operational KPI + Closed-Loop Promotion)

> **Supplementary record to the Defensive Publication.** This file provides a 1:1
> mapping between the implementation flows described in [`PUBLIC_DISCLOSURE_EN.md`](./PUBLIC_DISCLOSURE_EN.md)
> (sections corresponding to claims 16 and 17) and the actual production code,
> production deployment, and test results as of **2026-05-10**.

---

## Metadata

| Field | Value |
| --- | --- |
| Inventor / Author | 김현구 (Hyungu Kim, hyungu.kim@bullsone.com) |
| Implementation completion date | **2026-05-10** |
| Production deployment version | `1b83bf8f-7e4b-44c7-8d98-84690a788bf3` |
| Production domain | `https://app.ax-atlas.com` |
| Defensive Publication commit | `9eef4d4` (this repository) |
| Corresponding claims | Claim 16 (Operational KPI Measurement), Claim 17 (Closed-Loop Promotion) |
| Corresponding paragraphs in spec | [0034]–[0040] (실시예 9·10), [0040-A] (post-publication addendum) |

This file is published under the same **CC0 1.0 Universal** license as the rest of
this repository.

---

## 1. Purpose

The `PUBLIC_DISCLOSURE_EN.md` file in this repository established prior art for
the integrated system as a *concept*. This document supplements that disclosure
by recording, for the benefit of any reader (including patent examiners,
third-party observers, and counsel), that the same flows have been
**implemented in working code** and **deployed to a production environment**.

The reader should treat this file as evidence that:

- The flow described in §6 of `PUBLIC_DISCLOSURE_EN.md` (the closed-loop
  promotion R1→R2→R3→R4) is not a paper concept but a runnable system.
- Each of the six metrics enumerated in claim 16 has a concrete persistence
  schema, a concrete API endpoint, and a concrete UI surface.
- The system has been exercised by a unit-test suite of 59 newly added tests
  (640 in total across the repository, 0 failing, 5 skipped) prior to public
  posting of this status note.

---

## 2. Module Inventory

### 2.1 Core service layer (Node.js / Cloudflare Workers)

| Concern (claim) | Source file | Key exported functions |
| --- | --- | --- |
| 6 metric collection + hour/day/week/month auto-rollup + HEALTHY / WARNING / UNHEALTHY labelling (claim 16) | `aits/src/services/operational-kpi-collector.js` | `recordExecution`, `evaluateStatus`, `getThresholds`, `setThresholds`, `getMetricsTimeseries`, `listLatestStatusByCompany`, `periodKeyFor` |
| R1 re-interview trigger + R2 score recomputation + R3 promote/demote recommendation + AX-leader 1-click approval (claim 17) | `aits/src/services/promotion-engine.js` | `shouldReinterview`, `recomputeAutomationClass`, `recommend`, `applyApproval`, `listPendingRecommendations`, `tierToAutomationClass` |
| R4 capacity reallocation (claim 17) | `aits/src/services/capacity-allocator.js` | `computePriorityScore`, `rankCandidates`, `redistribute`, `applyRedistributionDecision`, `listPendingRedistributions` |
| Session lifecycle → KPI auto-ingest bridge (claim 16) | `aits/src/services/session-kpi-bridge.js` | `onSessionCompleted`, `onSessionVerified`, `onSessionReanalyzed`, `deriveKpiDimensions` |

### 2.2 Persistence layer (Cloudflare D1 / SQLite-compatible)

| Migration | Tables created |
| --- | --- |
| `aits/migrations/0008_operational_kpi.sql` | `operational_kpi_metrics`, `kpi_status`, `kpi_thresholds` |
| `aits/migrations/0009_promotion_history.sql` | `promotion_history`, `capacity_redistribution` |

All five tables are net new — no existing data is altered or migrated.

### 2.3 HTTP API surface (`/api/operations/*`)

The endpoints are mounted in **both** the Cloudflare Workers entry
(`aits/src/worker.js`, via `createHonoOperationsRouter()`) and the local Node
development entry (`aits/src/server.js`, via `createOperationsRouter()`), so
the same surface is reachable from production and from local dev:

| Method | Path | Purpose |
| --- | --- | --- |
| GET | `/api/operations/kpi/status` | Latest HEALTHY / WARNING / UNHEALTHY label per (task × tier) for the company |
| GET | `/api/operations/kpi/timeseries` | Hour / day / week / month time series for one (task × tier) |
| POST | `/api/operations/kpi/evaluate` | Force re-evaluation + recommendation for one (task × tier) |
| POST | `/api/operations/kpi/thresholds` | Override company-specific thresholds |
| GET | `/api/operations/promotion/pending` | List pending R3 recommendations |
| POST | `/api/operations/promotion/:id/decide` | AX-leader 1-click approve / reject |
| GET | `/api/operations/capacity/pending` | List pending R4 capacity recommendations |
| POST | `/api/operations/capacity/:id/decide` | AX-leader approve / execute / reject |

Authorisation: roles `L4`, `company_super`, `platform_super` only.

### 2.4 UI surface

| File | Purpose |
| --- | --- |
| `aits/public/operations.html` | Standalone operations page (`/operations` after `.html` strip) |
| `aits/public/js/modules/dashboard-kpi.js` | KPI card grid + re-evaluation button |
| `aits/public/js/modules/promotion-recommendation.js` | R3 / R4 recommendation cards + 1-click approve / reject |

The `/operations` page is intentionally separate from the existing
`dashboard.js` (≈1,358 LoC) to avoid any regression risk on the main
dashboard during PoC introduction.

---

## 3. Algorithmic Mapping (Spec → Code)

### 3.1 Claim 16 — labelling rule

`PUBLIC_DISCLOSURE_EN.md` describes the labelling rule. The implementation in
`operational-kpi-collector.js#classifyLabel` is:

```js
const isHealthy =
  rates.success_rate            >= thresholds.healthy_success_rate &&
  rates.human_intervention_rate <= thresholds.healthy_intervention_rate &&
  rates.review_ng_rate          <= thresholds.healthy_ng_rate
if (isHealthy) return 'HEALTHY'

const isUnhealthy =
  rates.success_rate            <  thresholds.unhealthy_success_rate ||
  rates.human_intervention_rate >  thresholds.unhealthy_intervention_rate ||
  rates.review_ng_rate          >  thresholds.unhealthy_ng_rate
if (isUnhealthy) return 'UNHEALTHY'

return 'WARNING'
```

Default thresholds (overridable per company via `kpi_thresholds`):
HEALTHY ≥ 95% success ∧ ≤ 5% intervention ∧ ≤ 5% NG;
UNHEALTHY < 80% success ∨ > 20% intervention ∨ > 15% NG;
otherwise WARNING.

### 3.2 Claim 17 — R3 decision (promote / demote)

```js
// promotion-engine.js — classifyDecision (excerpt)
// 0) If R2 reclassification has shifted by ≥ 1 tier, that decision wins.
if (r2 && r2.shifted) {
  if (toRank > fromRank && tier !== 'T1')
    return { decision: 'recommend_promote', from_tier: tier, to_tier: tierUp(tier),
             reason: `R2 shift up (${r2.reason})`, r2 }
  if (toRank < fromRank && tier !== 'T5')
    return { decision: 'recommend_demote', from_tier: tier, to_tier: tierDown(tier),
             reason: `R2 shift down (${r2.reason})`, r2 }
}

// 1) Critical demote
if (evaluation.success_rate < rules.demoteCriticalSuccessRate && evaluation.executions_count > 0)
  return { decision: 'recommend_demote', ... }

// 2) Promote: 14 consecutive HEALTHY days, intervention ≤ 3%, cooldown 30d cleared
if (allHealthy && cooldownOk && reasons.length === 0 && tier !== 'T1')
  return { decision: 'recommend_promote', ... }
```

### 3.3 Claim 17 — R4 capacity reallocation formula

`PUBLIC_DISCLOSURE_EN.md` and the spec [0039] state:

```
priority_score(task_i)
   = business_impact(task_i)
   × automation_feasibility(task_i)
   × (1 − current_automation_coverage(task_i))
```

The implementation in `capacity-allocator.js`:

```js
const VALUE_WEIGHTS = { 가치창출: 1.0, 필요_부대업무: 0.7, 낭비: 0.0 }
const TIER_COVERAGE = { T1: 1.0, T2: 0.5, T3: 0.5, T4: 0.0, T5: 0.0 }

export function computePriorityScore(task) {
  const impact = computeBusinessImpact(task)   // value_class weight × monthly_hours
  const feas   = computeFeasibility(task)      // automation_score / 25
  const gap    = computeCoverageGap(task)      // 1 − tier_coverage
  return Math.round(impact * feas * gap * 100) / 100
}
```

This is a 1:1 mapping with the formula in §6 of `PUBLIC_DISCLOSURE_EN.md`.

---

## 4. Verification

### 4.1 Test results

| Suite | File | Tests | Result |
| --- | --- | --- | --- |
| operational-kpi-collector | `aits/tests/operational-kpi-collector.test.js` | 15 | ✅ all pass |
| promotion-engine | `aits/tests/promotion-engine.test.js` | 24 (incl. 6 R2 tests) | ✅ all pass |
| capacity-allocator | `aits/tests/capacity-allocator.test.js` | 20 | ✅ all pass |
| **New tests subtotal** | — | **59** | **✅** |
| **Whole-repo regression** | `npm test` | **640** | **635 PASS / 0 FAIL / 5 skipped** |

### 4.2 Production HTTP probes (curl, 2026-05-10 03:22 UTC)

| Path | Status | Note |
| --- | --- | --- |
| `https://app.ax-atlas.com/api/health` | 200 `{"status":"ok","runtime":"workers"}` | Worker live |
| `https://app.ax-atlas.com/operations` | 200 (HTML) | New ops page live |
| `/js/modules/dashboard-kpi.js` | 200 (4 698 bytes) | KPI UI module |
| `/js/modules/promotion-recommendation.js` | 200 (6 519 bytes) | Promotion UI module |
| GET `/api/operations/kpi/status` | 401 `"인증이 필요합니다."` | Auth gate enforced |
| GET `/api/operations/promotion/pending` | 401 | Auth gate enforced |
| GET `/api/operations/capacity/pending` | 401 | Auth gate enforced |
| POST `/api/operations/promotion/9999/decide` | 401 | Auth gate enforced |

### 4.3 D1 migration application

| Migration | Status | DB |
| --- | --- | --- |
| `0008_operational_kpi.sql` | ✅ applied (8 commands) | `ax-atlas-prod` (`2f4f5df8-0921-4420-adb7-cfbcedf7117a`) |
| `0009_promotion_history.sql` | ✅ applied (7 commands) | same |

---

## 5. What is *not* yet implemented (kept for follow-up PRs)

These items are deliberately out of PoC scope. None of them are required for
the prior-art / written-description purpose of this Defensive Publication; they
are listed here for completeness:

- Auto-mode toggle (executing promote/demote without the AX-leader 1-click
  gate). The PoC produces only `recommend_*` records.
- Cron-driven roll-up (hour → day → week → month). The PoC uses lazy roll-up.
- External FX-rate sync for `cost_per_execution`. Currently a fixed unit price.
- GUI for editing `kpi_thresholds` per company. The POST API exists; the
  screen does not.
- Promotion of `task_signature` to a first-class column. The PoC derives it
  from `${employee_id}::${task_title}`.
- Visual regression tests via Chrome DevTools MCP. The PoC relies on
  curl + unit tests.

---

## 6. Citation

If this file is cited as prior art (alongside `PUBLIC_DISCLOSURE_EN.md`), please
include the commit hash that contains it:

```
김현구 (Hyungu Kim), "AX Atlas — Defensive Publication: Implementation Status (Claims 16, 17)",
GitHub: macross-7/ax-atlas-defensive-publication, commit <sha>, 2026-05-10. CC0 1.0.
```

---

*This file is part of the AX Atlas Defensive Publication.
Released to the public domain under CC0 1.0 Universal — see [`LICENSE`](./LICENSE).*
