# Handoff to the Engineering Team

This document maps every file in the demo repo to its target location in the production Vue + Azure codebase. Reference: `docs/PORTING_GUIDE.md`, `docs/SCHEMA_PROPOSAL.md`.

---

## Engine modules (`src/engine/`) — **port these verbatim**

These are pure data-transformation modules with no DOM dependencies. Lift them straight into TypeScript and pair every change with the test scenarios in `docs/ENGINE_VALIDATION_SCENARIOS.md`.

| Demo file | Production target | Notes |
|---|---|---|
| `src/engine/normalizers.js` | `src/ingest/normalizers.ts` | Phone, email, VIN, name parsing; junk-detector regexes |
| `src/engine/unionFind.js` | `src/engine/unionFind.ts` | Pure data structure |
| `src/engine/internalDetection.js` | `src/engine/internalDetection.ts` | The >25-service-no-sale heuristic |
| `src/engine/buckets.js` | `src/engine/buckets.ts` | **6-bucket Azure-spec classifier** (Apps Script v2 had 5) |
| `src/engine/eventStream.js` | `src/engine/eventStream.ts` | DMS rows → events keyed by VIN. Replace this module entirely with a DMS-feed adapter once daily pulls are wired up. |
| `src/engine/segments.js` | `src/engine/segments.ts` | **THE 7 RULES.** Highest-stakes file. Do not modify without running all scenarios. |
| `src/engine/buildLoyaltyTimeline.js` | Split into 3 files (per `PORTING_GUIDE.md`): | |
| | `src/engine/clustering.ts` | Buyer-of-Record Gate + PII clustering |
| | `src/engine/customer.ts` | Cluster → customer record assembly, vinFlags, drift detection |
| | `src/engine/index.ts` | The top-level orchestrator |
| `src/engine/targets.js` | `src/engine/targets.ts` | Per-(customer, current-VIN) row explosion. Also has `buildSalesHistoryRows`. |
| `src/engine/anomalies.js` | `src/anomalies/queue.ts` | Persists into `customer_history` per `ANOMALY_QUEUE_SPEC.md` |
| `src/engine/runEngine.js` | `src/jobs/dailyIngest.ts` | The single entry point; called nightly after the DMS pull |

---

## UI modules (`src/ui/`) — **rebuild as Vue 3 SFCs**

These are vanilla JS but written to map cleanly to Vue 3 + Pinia. Each file is one component. Filter / sort logic should be extracted to composables.

| Demo file | Production Vue SFC | Notes |
|---|---|---|
| `src/ui/app.js` | `src/App.vue` + `src/stores/profilesStore.ts` (Pinia) | View router → vue-router; global state → Pinia store |
| `src/ui/stats.js` | `src/components/StatsBar.vue` | Bucket strip + summary numbers |
| `src/ui/filters.js` | `src/composables/useFilters.ts` + `src/components/FilterPanel.vue` | Filter logic is pure — lift to a composable; the chip rendering is the SFC |
| `src/ui/currentlyOwned.js` | `src/views/CurrentlyOwned.vue` | Default view. Filter / sort logic should be pure functions in `src/api/queries.ts`. |
| `src/ui/salesHistory.js` | `src/views/SalesHistory.vue` | Status pills come from the engine; "Defected" status needs a data-provider feed (TODO). |
| `src/ui/anomalies.js` | `src/views/Anomalies.vue` | Wire Resolve / Suppress / Escalate to the backend per `ANOMALY_QUEUE_SPEC.md` |
| `src/ui/detail.js` | `src/components/CustomerDetailDrawer.vue` | The slide-over panel |
| `src/ui/export.js` | `src/components/ExportModal.vue` + `src/api/export.ts` | The schema (29 columns) goes in `api/export.ts` |
| `src/ui/settings.js` | `src/views/Settings.vue` | Thresholds come from the dealer record (per-tenant config) in production |
| `src/ui/uploader.js` | `src/components/UploadFlow.vue` | In production, replaced entirely by scheduled DMS pulls |
| `src/ui/notify.js` | Replace with any Vue toast library (e.g. `vue-sonner`) | |

---

## Data layer

| Demo file | Production approach |
|---|---|
| `src/data/sample.js` | **Discard.** Production loads from the database after the daily ingest job. |
| `src/data/csvSchema.js` | Keep for the manual CSV upload fallback. Build a **column-mapping UI** so any DMS format can be mapped to the canonical schema once per dealer (see `TODO` in the file). |

---

## Stylesheet

`src/styles.css` is a refresh of the Apps Script aesthetic — kept simple and CSS-variable-driven so your team can either:
- Convert variables to your Vue design system tokens, or
- Replace entirely with your component library

The electric purple (`#5E10BC`) is the Convergence brand accent and should carry over.

---

## What needs to be built fresh (not in this demo)

Per `docs/KNOWN_ISSUES_AND_NEXT_STEPS.md`:

1. **Multi-tenancy** — `dealer_id` on every table, per-dealer auth, Convergence admin role
2. **DMS ingestion adapters** — one module per DMS partner (`src/ingest/dms/*.ts`)
3. **Scheduled daily job** — Azure timer trigger → ingest → engine → DB write
4. **Suppression framework** — In-tool suppression rules per `docs/SUPPRESSION_RULES.md`
5. **History tracking** — `customer_history` table, drift events, bucket transitions
6. **Anomaly persistence** — Resolve / Suppress / Notes on each anomaly
7. **Business records** — `customer_type = 'business'` flag (currently filtered out entirely in Apps Script)
8. **Deal Type admin override** — Manual lease/retail override per sale event
9. **Time zone handling** — UTC storage, per-dealer local-time computation
10. **Defected status feed** — Sourced from data provider, not engine-inferred (the demo's Sales History view stubs this)

---

## Validation strategy

After porting:

1. Run `docs/ENGINE_VALIDATION_SCENARIOS.md` scenarios against the new engine
2. Confirm output matches the Python reference (`docs/vin_spine_engine_v2.py`) on identical input
3. Run the Apps Script tool and the new engine in parallel for one sprint on a real dealer (TTGM)
4. Diff customer counts, anomaly counts, per-customer details
5. Discrepancies investigated by hand, fixed in engine, scenarios extended

Locked decisions (don't re-litigate — see `docs/KNOWN_ISSUES_AND_NEXT_STEPS.md` for the full list).

---

## Sequence

Suggested build order:

1. Schema setup on Azure (`docs/SCHEMA_PROPOSAL.md`)
2. Port engine modules (`src/engine/*`) to TypeScript, validate against scenarios
3. Build DMS ingestion adapter for one DMS partner
4. Build daily scheduled ingest job
5. Build Vue UI on top of the API (port `src/ui/*` to SFCs)
6. Build anomaly queue UI + suppression framework
7. Multi-tenant access control
8. Migrate first dealer to Azure, run parallel for one sprint
9. Retire Apps Script

Each engine port should be a separate PR. Each Vue view should be a separate PR. Claude Code can take one PR at a time with the scenarios as acceptance tests.
