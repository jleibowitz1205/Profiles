# Profiles

> Demo standup of the VIN-spine engine + customer-vehicle UI for Convergence Auto.

A web app that ingests dealer DMS exports (Sales + Service feeds) and produces clean, texting-ready customer-vehicle records.

This repo is a **single-page demo** built to run on GitHub Pages with zero build step. It exists so the Convergence engineering team can see the engine and UI working end-to-end before porting to the production Vue + Azure stack.

---

## Run locally

No build step. Just open `index.html` in a browser, or:

```bash
# Any static server works
python3 -m http.server 8000
# then visit http://localhost:8000
```

Or click **"Load sample data"** in the UI to see the engine produce ~30 synthetic customers, 60+ events, and a handful of anomalies.

---

## Deploy to GitHub Pages

1. Push this repo to GitHub.
2. Repo → **Settings** → **Pages** → Source: **Deploy from a branch**, branch: `main` (or `master`), folder: `/ (root)`.
3. Wait ~30 seconds. Your site is live at `https://<user>.github.io/<repo>/`.

The included `.github/workflows/deploy.yml` will auto-deploy on push if you'd rather use GitHub Actions.

---

## What's in here

| File / Folder | What it is |
|---|---|
| `index.html` | App shell. Loads engine + UI scripts via `<script>` tags. |
| `src/engine/` | **The engine** — verbatim Apps Script v2 modules. This is what your team should port. |
| `src/ui/` | Vanilla JS UI modules. Each one is a candidate Vue SFC — see `HANDOFF.md`. |
| `src/data/sample.js` | Synthetic sample dataset that exercises every engine rule and anomaly type. No real customer data. |
| `src/data/csvSchema.js` | Strict column-name validation. Contains a `TODO` marker for the future column-mapping UI. |
| `src/styles.css` | Single stylesheet. Refresh of the Apps Script look — same electric purple, more whitespace, modern type. |
| `samples/` | Downloadable CSVs (generated from `sample.js`) for testing the upload flow. |
| `HANDOFF.md` | File-by-file map from this repo to the production Vue + Azure structure. |
| `docs/` | Full spec docs (architecture, schema proposal, anomaly queue, porting guide, validation scenarios). |

---

## Views

- **Currently Owned** (default) — One row per (customer, currently-owned VIN). Texting-first. Per-row flags fire only on the vehicle that has the condition.
- **Sales History** — One row per SALE EVENT, with Status pills (Currently Owned / Traded Back / Stopped Servicing / Defected).
- **Anomalies** — Queue of engine-flagged data-quality concerns: cross-household trades, inter-owner services, possible duplicates, drift, service gaps, high-volume.
- **Settings** — Defection / service-gap thresholds + a full engine run summary.

Clicking any row opens a side **detail drawer** with the full customer record, all vehicles, drift notes, and event timeline.

---

## Engine pipeline

```
CSVs ──▶ eventStream ──▶ internalDetection ──▶ segments (THE 7 RULES)
                                                       │
                                                       ▼
                                          Buyer-of-Record Gate
                                          + Union-Find clustering
                                                       │
                                                       ▼
                                          customer records (vinFlags, drift, buckets)
                                                       │
                                                       ▼
                                          target rows + sales history + anomalies
```

See `docs/ARCHITECTURE.md` for the full design. See `docs/ENGINE_VALIDATION_SCENARIOS.md` for the regression test cases the engine must continue to pass after the port.

---

## What this demo is NOT

- **Not multi-tenant.** Single dealer, no auth. Your team adds tenancy (per `docs/SCHEMA_PROPOSAL.md`).
- **Not connected to a DMS.** Loads sample data or CSV upload only. Production pulls daily DMS feeds.
- **Not the production UI.** The visual language is refreshed, but the production version should be Vue SFCs styled to match Convergence's design system.
- **Not the production data layer.** No database, no history tracking. Everything is in-memory.

---

## For the engineering team

Start with [`HANDOFF.md`](HANDOFF.md). It maps every file in this repo to its target location in your Vue + Azure codebase, with notes on what should be ported verbatim, refactored, or rebuilt.

---

Built with Claude as a scaffold. Engine logic is Convergence's — extracted verbatim from the Apps Script v2 production tool.
