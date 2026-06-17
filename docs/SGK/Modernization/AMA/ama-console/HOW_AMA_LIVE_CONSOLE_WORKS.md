# How AMA Live Console Works

> A technical walkthrough of **AMA Live Console** — a live + offline console over the IBM Application Migration Accelerator (AMA) REST API.

---

## 1. What it is

AMA Live Console is an **API-driven console**. Its job is to sit in front of the AMA REST API, which exposes migration analysis (cost, effort, issues, target compatibility) in a deeply-nested, target-keyed, hash-identified form that is hard to consume directly. The console does the normalization once and presents a clean, navigable interface.

Crucially, it runs in **two interchangeable modes from a single image**:

- **Live** — proxies the AMA REST API in real time (token required).
- **Offline** — serves a pre-processed snapshot baked into the image (no token, no network).

The frontend is identical in both modes — only the backend's data source changes.

---

## 2. The core idea — one backend, two data sources

```
                         ┌─ is_offline() == TRUE ──► read baked snapshot
Browser ─► /api/* ─► backend                          (bulk_detail.json,
           (flat,        │                              drill_index.json)
            stable       └─ is_offline() == FALSE ─► call AMA REST API,
            contract)                                  normalize per request
```

The single decision `is_offline()` (driven by the `AMA_OFFLINE` environment variable) selects the data source. Everything downstream — the internal `/api/*` contract and the frontend — is unchanged. This is what makes the same artifact usable for an offline demo and a live session without a separate build.

---

## 3. Live mode — talking to the AMA REST API

In live mode the backend is essentially a **normalizing proxy**. For a given request it:

1. Reads the user's OpenShift token (entered in the UI, forwarded per request).
2. Calls one or more AMA endpoints with **both** required auth headers:
   ```
   Authorization: Bearer <token>
   apiKey: <token>
   ```
3. Parses the nested, target-keyed JSON and flattens it into render-ready objects.
4. Returns the result through the internal `/api/*` contract.

The AMA endpoints it consumes:

| AMA endpoint | What the console extracts |
|--------------|---------------------------|
| `/advisor/v2/workspaces` | Workspace list + selected target preferences |
| `/advisor/v2/workspaces/{id}/costDetails` | Per-target rollup (common/unique/total cost) **and** per-app effort, complexity, issue counts, plus the versioned JAR inventory |
| `/advisor/v2/workspaces/{id}/assessmentUnits` | Application inventory (name, host, middleware, ear/war) |
| `/advisor/v2/workspaces/{id}/issues?targetPlatform=X` | Migration rules for a specific target |
| `/advisor/v2/targetCompatibility` | Java SE/EE matrix per target |
| `/advisor/v2/workspaces/{id}/collectionArchives/bulkExport` | Full workspace export (streamed to the browser) |

### Why normalization matters

AMA's `costDetails` carries three layers in one response — a workspace rollup, a per-application block, and a versioned JAR list. Effort is split into **common** (shared libraries, counted once) and **unique** (application-specific) so the totals are realistic rather than a naive per-app sum. The backend's transforms (`_cost_by_target`, `_app_costs`, `_packaging_of`) turn this into flat per-app records the frontend can render directly.

Targets are genuinely different — `websphereLiberty` supports Java 8/11/17/21 and EE 6–10, while `websphereTraditional` supports only Java 8 and EE 7 — so the per-target `issues?targetPlatform=` call is essential; the same app yields a different issue set per target.

---

## 4. Offline mode — the baked snapshot

Offline mode answers the same internal `/api/*` calls from a pre-processed snapshot rather than the live API. The snapshot is produced once from an AMA **bulk export**:

```
bulkExport.zip ──► offline.py prepare ──► bulk_detail.json  (~1.6 MB, summary)
                                          drill_index.json  (~18 MB, app+category drill-down)
```

The `prepare` step parses the bulk export, verifies data integrity against AMA's own `severitySummary` (Critical / Warning / Information counts must match exactly), and writes the two JSON files. These are copied into the image at build time, so offline mode — including the Migration Workbench — works with no network and no token.

---

## 5. What the console presents

1. **Workspace overview** — KPI cards (apps, Liberty-ready ratio, total effort split into common/unique), target-compatibility summary, and the dominant migration blockers.
2. **All Findings** — a searchable, filterable, drill-down table of every rule hit, with severity and the triggering JARs.
3. **Migration Workbench** — a three-column working surface (application → issue categories → consolidated fix list) with multi-category selection, for scoping the fixes of one application at a time.
4. **Per-application detail** — effort, complexity, critical-issue count, and required code-change classification (live, from `costDetails`).

---

## 6. Packaging and runtime

- **Runtime:** FastAPI under `uvicorn`, single worker, port **8090**.
- **Image:** UBI9 Python 3.11. Sources are byte-compiled and `.py` files removed (`.pyc` only) for source confidentiality. The offline snapshot is baked in.
- **No PVC:** all data is either in the image (offline) or fetched live — so there is no persistent state and no Multi-Attach volume contention. Deploy strategy is `Recreate`.
- **Mode switching:** a single `oc set env ... AMA_OFFLINE=0|1` plus rollout; no rebuild.

---

## 7. Where AMA Live Console fits

AMA Live Console is the **focused, live analysis tool**: it connects directly to the AMA REST API for real-time, single-workspace depth — current effort, per-app issues, per-target compatibility — and falls back to a baked snapshot for offline demos.

It complements AMA Portal (see the companion document), which instead aggregates **ingested artifacts** across many workspaces into a stakeholder-facing portal.

| | AMA Live Console | AMA Portal |
|---|------------------|------------|
| Data source | AMA REST API (live) or baked snapshot (offline) | Ingested artifacts (ZIPs, docs) |
| Real-time API | Yes — in live mode | No — works from exports |
| State | Stateless (image or live) | Persistent (PVC) |
| Scope | One workspace, deep | Many workspaces + documents |
| Primary audience | Engineers, live per-app analysis | Stakeholders, cross-workspace overview |

---

*Designed & developed by Mert Ertuğral · IBM Expert Labs*
