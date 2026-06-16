# AMA Live Console — Technical Architecture

> SGK Modernization Project · A live + offline console over the IBM Application Migration Accelerator (AMA) REST API
> Designed & developed by **Mert Ertuğral** · IBM Expert Labs

-----

## 1. Purpose

AMA Live Console is a lightweight web application that consolidates IBM Application Migration Accelerator (AMA) analysis output into a single, navigable interface for a large-scale WebSphere-to-Liberty modernization program (500+ applications migrating from WebSphere Application Server Traditional to WebSphere Liberty on OpenShift).

The AMA product exposes its analysis through a REST API, but that raw API is not directly consumable by migration engineers or stakeholders: responses are deeply nested, identifiers are opaque hashes, effort and issue data are split across multiple endpoints, and there is no aggregated cross-application view. AMA Live Console sits in front of that API as a **purpose-built aggregation and presentation layer**.

It operates in two interchangeable modes from the **same codebase and the same image**:

- **Live mode** — proxies the AMA REST API in real time, reflecting the current state of a workspace.
- **Offline mode** — serves a pre-processed snapshot baked into the container image, requiring no token and no network access to AMA.

-----

## 2. The AMA REST API (the data source)

The application is built entirely around the AMA REST API, served under a base path of `/lands_advisor` on the AMA host. All endpoints are version `v2` under `/advisor/v2`.

### 2.1. Authentication

Every request to AMA carries the OpenShift bearer token in **two headers simultaneously**, which the AMA gateway requires:

```
Authorization: Bearer <token>
apiKey: <token>
```

The token is the OpenShift session token (`oc whoami -t`). TLS verification is disabled for the internal `.intra` endpoint (self-signed corporate CA). The token is never persisted — in live mode it is entered in the UI and forwarded per request.

### 2.2. Consumed Endpoints

|Method|AMA Endpoint                                               |Purpose                                                                                                                         |
|------|-----------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------|
|`GET` |`/advisor/v2/workspaces`                                   |List workspaces, with per-workspace target preferences (selected Java SE / Java EE levels)                                      |
|`GET` |`/advisor/v2/workspaces/{id}/costDetails`                  |Per-target cost summary **and** per-application effort, complexity, issue counts, and the versioned JAR inventory (`commonCode`)|
|`GET` |`/advisor/v2/workspaces/{id}/assessmentUnits`              |Application inventory (name, host, middleware, packaging)                                                                       |
|`GET` |`/advisor/v2/workspaces/{id}/issues?targetPlatform=X`      |Migration rules (issues) for a specific target platform                                                                         |
|`GET` |`/advisor/v2/targetCompatibility`                          |Java SE / Java EE matrix supported by each migration target                                                                     |
|`GET` |`/advisor/v2/workspaces/{id}/collectionArchives/bulkExport`|Full workspace export as an `application/octet-stream` ZIP                                                                      |

### 2.3. Response Shapes That Drive the UI

The non-trivial value of the application lives in how it parses these responses.

**`costDetails`** is the richest endpoint. It carries three distinct layers:

1. `domains[].recommendations.summary.targets[]` — workspace-level rollup per target:
   
   ```
   totalCost = commonCodeCost + uniqueCodeCost
   e.g. 73.1 = 11.1 (shared libraries) + 62.0 (application-specific)
   ```
   
   The split between **common** and **unique** code cost is what makes the live figure realistic — shared JARs are counted once, not per application.
1. `domains[].recommendations.assessmentUnits[].targets[].summary` — **per-application** detail:
- `effort.total` (person-days)
- `complexity.score` (`simple` / `moderate` / `complex`)
- `issues` (`critical`, `suggested`, `potential`)
- `requiredCodeChanges` (`none` / `manual` / `partAutomated`)
- `technologyType` (e.g. `Liberty Base`)
   
   Example extracted: `WS_BankaIslemleri4a.war → 61.7 days, 32 critical, partAutomated`.
1. `assessmentUnits[].commonCode[]` — the **versioned JAR inventory** per application (e.g. `jaxb-api-2.3.1`, `javax.servlet-api-4.0.1`, `javax.jws-api-1.1`). This is the raw material for downstream `pom.xml` generation.

**`issues`** returns, per rule: `ruleId`, `ruleName`, `totalApps`, and `commonCodeAppCounts[]` — which JAR triggers the rule and in how many applications. Real rules observed include `DetectJAXRPCSourceMigration` (JAX-RPC → JAX-WS), `WebSphereWebServicesGeneratedClassesRule` (WSDL2Java), and the full Java 11/17/21 and Jakarta EE compatibility rule set.

**`targetCompatibility`** confirms that targets are genuinely different, not cosmetic variants:

```
websphereLiberty     : Java 8/11/17/21,  EE 6–10   (flexible)
websphereTraditional : Java 8 only,      EE 7 only (highly constrained)
managedLiberty       : Java 17/21,       EE 10
openLiberty          : Java 8/11/17/21,  EE 7–10
```

This is why the same application yields different issue sets and effort per target — the per-target `issues?targetPlatform=` call is essential.

-----

## 3. Application Architecture

### 3.1. Components

```
┌──────────────────────────────────────────────────────────┐
│  Frontend — single-file HTML/JS                           │
│  (workspace switcher, KPI cards, findings, workbench)     │
└────────────────────────┬─────────────────────────────────┘
                         │ internal JSON API (/api/*)
┌────────────────────────┴─────────────────────────────────┐
│  Backend — FastAPI (Python)                               │
│                                                           │
│   is_offline() ?                                          │
│     ├─ TRUE  → read baked snapshot (bulk_detail.json,     │
│     │           drill_index.json)                         │
│     └─ FALSE → call AMA REST API, parse, normalize        │
└────────────────────────┬─────────────────────────────────┘
                         │ (live mode only)
                ┌────────┴─────────┐
                │  AMA REST API     │
                │  /advisor/v2/...  │
                └───────────────────┘
```

### 3.2. Internal API Surface

The backend exposes a clean, flat `/api/*` contract to the frontend, hiding the complexity (and the dual-mode branching) of the AMA API. The frontend never knows whether data came from the live API or the offline snapshot — the shape is identical.

|Internal Endpoint                                          |Backing Source (live)                       |Description                                                          |
|-----------------------------------------------------------|--------------------------------------------|---------------------------------------------------------------------|
|`GET /api/health`                                          |—                                           |Reports `offline` flag and configured server                         |
|`GET /api/workspaces`                                      |`/advisor/v2/workspaces`                    |Workspace list with targets                                          |
|`GET /api/workspaces/{id}`                                 |`costDetails` + `assessmentUnits` + `issues`|Aggregated workspace detail (apps, per-app effort, per-target issues)|
|`GET /api/target-compatibility`                            |`/advisor/v2/targetCompatibility`           |Target → Java SE/EE matrix                                           |
|`GET /api/workspaces/{id}/bulk-export`                     |`collectionArchives/bulkExport`             |Streams the ZIP through to the browser                               |
|`GET /api/insights`                                        |snapshot                                    |Cross-application aggregates (offline)                               |
|`GET /api/detail/{rules,applications,categories,hosts,...}`|snapshot                                    |Faceted breakdowns                                                   |
|`GET /api/workbench/apps`                                  |snapshot                                    |Migration Workbench application list                                 |
|`GET /api/workbench/app/{name}/categories`                 |snapshot                                    |Per-app issue categories                                             |
|`GET /api/workbench/app/{name}/findings`                   |snapshot                                    |Per-app, multi-category finding list                                 |

### 3.3. Key Backend Transforms

The backend’s job is **normalization** — turning AMA’s nested, target-keyed structures into flat, render-ready objects:

- `_cost_by_target()` — extracts the workspace-level `common / unique / total` cost per target.
- `_app_costs()` — flattens `assessmentUnits[].targets[].summary` into per-application `{effort, complexity, critical, suggested, code_changes, tech_type}`.
- `_packaging_of()` — derives `ear` / `war` / `jar` from the application file name (the basis for later `pom.xml` packaging).
- `_api()` — the single AMA call helper: attaches both auth headers, disables TLS verification, handles raw (binary ZIP) vs JSON responses.

-----

## 4. What the Console Presents

The UI turns the normalized data into four working views:

1. **Workspace overview** — KPI cards (applications, Liberty-ready ratio, total effort split into common/unique), target-compatibility summary, and the dominant migration blockers (e.g. JAX-RPC across N apps).
1. **All Findings** — a searchable, filterable, drill-down table of every rule hit, with severity and the triggering JARs.
1. **Migration Workbench** — a three-column working surface (application → issue categories → consolidated fix list) supporting multi-category selection, so an engineer can scope “all rules to fix in N categories” for one application at a time.
1. **Per-application detail** — effort, complexity, critical-issue count, and required code-change classification, sourced live from `costDetails`.

-----

## 5. Deployment Model

- **Runtime:** FastAPI under `uvicorn`, single worker, port **8090**.
- **Packaging:** UBI9 Python 3.11 base image. Python sources are byte-compiled and the `.py` files removed, leaving only `.pyc` (source confidentiality). The offline snapshot is baked into the image, so the Migration Workbench works with zero external dependencies.
- **Platform:** OpenShift. Exposed via an edge-TLS Route. No PersistentVolumeClaim — all data is either in the image (offline) or fetched live, which avoids the Multi-Attach volume contention seen with stateful workloads. Deployment strategy is `Recreate`.
- **Mode switching:** controlled at deploy time by the `AMA_OFFLINE` environment variable; switching between live and offline is a single `oc set env` + rollout, no rebuild.

-----

## 6. Why This Design

|Decision                               |Rationale                                                                                                       |
|---------------------------------------|----------------------------------------------------------------------------------------------------------------|
|Dual-mode, one image                   |The same artifact demos offline (no token, no network) and runs live against AMA — no separate builds, no drift.|
|Backend normalization layer            |AMA’s API is target-keyed and deeply nested; the frontend stays simple because the backend flattens once.       |
|Both `Authorization` + `apiKey` headers|The AMA gateway rejects requests missing either; discovered empirically against the live endpoint.              |
|Common vs unique cost preserved        |Reflects AMA’s real costing model — shared libraries counted once — instead of naive per-app summation.         |
|Versioned JAR inventory captured       |Feeds downstream `pom.xml` generation for the Liberty migration (Java EE 7, `javax.*`).                         |
|Source compiled to bytecode            |The console is a customer-facing deliverable; `.pyc`-only shipping protects the implementation.                 |

-----

*Designed & developed by Mert Ertuğral · IBM Expert Labs*