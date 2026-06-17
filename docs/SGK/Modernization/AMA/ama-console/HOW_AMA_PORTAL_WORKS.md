# How AMA Portal Works

> A technical walkthrough of the **AMA Portal** — a web portal that serves IBM Application Migration Accelerator (AMA) workspace outputs for the SGK WebSphere → Liberty modernization program.

---

## 1. What it is

AMA Portal is a **data-serving portal**. Its job is to take the artifacts produced during an AMA-driven modernization (workspace exports, architecture documents, migration plans, technical reports) and present them through a single, organized web interface that colleagues and customers can browse independently.

Unlike a live API client, the portal is **content-driven**: it ingests files (mostly ZIP exports and documents) at build or runtime, transforms them into a structured data model, and serves that model. It does not talk to the AMA REST API in real time — it works from exported artifacts.

---

## 2. The core idea — ingest, transform, serve

```
Workspace ZIPs ─┐
Architecture MDs ┼─► build/ingest step ─► structured data ─► web portal ─► browser
Migration plans ─┘    (parse + normalize)   (workspaces.json    (FastAPI +
Tech reports    ─┘                            + per-app records)  HTML frontend)
```

The portal has three logical layers:

1. **Ingestion** — scaffold/import scripts read the raw inputs:
   - Workspace export ZIPs → parsed into per-application assessments.
   - Migration plan documents → attached to the relevant applications/workspaces.
   - Architecture documents and dependency analysis → imported as supporting material.
   - Technical reports → served as downloadable deliverables.

2. **Data model** — the ingestion produces a consolidated structure (e.g. `workspaces.json`) holding: workspaces, their applications, per-application estimated development cost, issue counts, dependency information, and global aggregates (total workspaces, total applications, total issues, total estimated cost).

3. **Serving** — a FastAPI backend exposes this model to a single-file HTML frontend that renders the workspace browser, per-application detail, cost summaries, and report downloads.

---

## 3. Operating model — data baked in, optionally persisted

The portal is packaged as a container image. Two patterns are supported:

- **Image-seeded:** the workspace ZIPs and documents are copied into the image at build time, so the portal ships already populated. On first start, the data is built from those seeds.
- **Persisted (PVC):** a PersistentVolumeClaim is mounted so that runtime uploads, derived data, aliases, and audit logs survive restarts and version upgrades. Environment variables redirect the data/workspace/report directories onto the persistent volume.

This is why AMA Portal uses a PVC while lighter tools may not — the portal accumulates state (uploaded artifacts, runtime-generated data) that must outlive a pod restart.

---

## 4. Observability — admin and master panels

The portal includes operational views beyond the data browser:

- An **admin panel** for managing portal content.
- A **master panel** for observability: visitor counts, unique IP tracking, and per-visitor breakdown (IP + browser). In SGK's environment all office traffic appears behind a single corporate proxy IP, so a user-agent fingerprint is used to distinguish individual visitors behind that shared IP.

---

## 5. Deployment lifecycle

```
Prepare data (Mac)         → unzip release, import workspaces / plans / architecture
Build image (amd64)        → podman build, data seeded into the image
Push to OCP registry       → external route for push
Deploy                     → oc set image (internal registry), rollout
First-deploy only: PVC     → create + mount + redirect env vars, then rollout
Subsequent versions        → build → push → set image (PVC data preserved)
```

Because the persistent data lives on the PVC, upgrading to a new version only swaps the code image — uploaded artifacts, derived data, and audit logs are preserved. Rollback swaps the image back without touching persisted data.

---

## 6. Where AMA Portal fits

AMA Portal is the **broad, content-rich deliverable**: it aggregates the full set of modernization artifacts across many workspaces and presents them as a browsable portal for stakeholders. It is the place where exported analyses, plans, architecture, and reports come together.

It is complemented by AMA Live Console (see the companion document), which instead connects **live** to the AMA REST API for real-time, single-workspace analysis.

| | AMA Portal | AMA Live Console |
|---|------------|------------------|
| Data source | Ingested artifacts (ZIPs, docs) | AMA REST API (live) or baked snapshot (offline) |
| Real-time API | No — works from exports | Yes — in live mode |
| State | Persistent (PVC) | Stateless (data in image or fetched live) |
| Scope | Many workspaces + documents | One workspace at a time, deep |
| Primary audience | Stakeholders, cross-workspace overview | Engineers, live per-app analysis |

---

*Designed & developed by Mert Ertuğral · IBM Expert Labs*
