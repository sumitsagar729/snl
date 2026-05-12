# TunnelSNL — Tunnel Analytics & Per-App Domain Control

> **Status:** Proof of Concept (PoC) &nbsp;·&nbsp; **Audience:** Leadership review,
> Tunnel / UEM product stakeholders, security operations &nbsp;·&nbsp;
> **Sample environment:** <https://snl.ssdevrd.com/snlweb/>

TunnelSNL is a self-contained analytics and policy-management front end that
sits *next to* — not *inside* — the existing UEM (Unified Endpoint
Management) console. It demonstrates how Tunnel telemetry can be surfaced
to administrators with rich, interactive visualizations and how blocking
decisions can be taken at **device** scope *and* **per-application**
scope without modifying the UEM product itself.

---

## Table of contents

1. [Executive summary](#1-executive-summary)
2. [Why this PoC — problem statement](#2-why-this-poc--problem-statement)
3. [Capabilities at a glance](#3-capabilities-at-a-glance)
4. [Solution architecture](#4-solution-architecture)
5. [Decoupling rationale — independent service layer](#5-decoupling-rationale--independent-service-layer)
6. [Service-layer responsibilities](#6-service-layer-responsibilities)
7. [Technology stack & rationale](#7-technology-stack--rationale)
8. [Live sample URLs](#8-live-sample-urls)
9. [Repository layout](#9-repository-layout)
10. [Local development](#10-local-development)
11. [Production build](#11-production-build)
12. [Deployment to IIS](#12-deployment-to-iis)
13. [Runtime configuration reference](#13-runtime-configuration-reference)
14. [API surface (reference)](#14-api-surface-reference)
15. [Operations — smoke tests & troubleshooting](#15-operations--smoke-tests--troubleshooting)
16. [Productionization roadmap](#16-productionization-roadmap)
17. [Glossary](#17-glossary)

---

## 1. Executive summary

TunnelSNL is a thin, decoupled web experience built on top of the UEM
Tunnel data set. Its goals are:

- **Make Tunnel telemetry actionable.** Replace static tabular reports
  with an interactive dashboard that highlights risk (red-flag domains,
  device risk scores, browsing categories) and lets administrators take
  policy actions in two clicks.
- **Prove per-application policy.** Demonstrate that the same domain can
  be allowed for one application and blocked for another, by maintaining
  a dedicated app-scoped rule-set that is published back to UEM through
  its existing public APIs.
- **Validate a non-invasive integration pattern.** Build the PoC as an
  independent service that talks to the UEM database (read) and the UEM
  configuration API (write), so the UEM product team has zero coupling
  to the PoC and the PoC can be iterated on independently.

The current deployment in
<https://snl.ssdevrd.com/snlweb/> hosts a fully working version of the
dashboard, persisting policy changes through the production UEM Tunnel
APIs.

---

## 2. Why this PoC — problem statement

| Pain point in the existing UEM Tunnel experience | What TunnelSNL demonstrates |
| ------------------------------------------------ | --------------------------- |
| Tunnel data is captured but surfaced primarily as flat reports — administrators must hunt for risk. | A single dashboard with 12+ chart cards correlating devices, domains, applications, categories, hours and risk scores. |
| Blocking a domain is a global decision; if Chrome needs `yahoo.com` but Firefox abuses it, admins must choose all-or-nothing. | Side-by-side block/unblock at **device scope** *and* **per-application scope**, with the rule scoped to the application's executable path. |
| Adding new analytics requires shipping a UEM build. | Analytics is computed in an independent service. New charts ship without touching the UEM core. |
| Threat-intel context is hard to associate with traffic. | Red-flag reasons, threat-intel categorization, and curated dictionaries are surfaced inline next to the offending domain. |

---

## 3. Capabilities at a glance

### 3.1 Visual analytics (twelve interactive charts)

| # | Chart | Insight surfaced |
| - | ----- | ---------------- |
| 1 | **Device Count by OS Build** | Fleet composition, click-through to all devices on a build. |
| 2 | **Top Visited Domains** | Aggregate domain popularity across the fleet. |
| 3 | **Red Flag Domains Detected** | Threat-intel-flagged domains, ranked by visits, with one-click *Take Action*. |
| 4 | **Top Flagged Domains by App** | Dual pie chart highlighting per-app exposure (e.g. Firefox vs OpenClaw AI). |
| 5 | **App-Level Red Flag Domains** | Horizontal bar chart with a per-user **chart-customization panel** (width, height, bar thickness, layout, font size, max items) persisted to `localStorage`. |
| 6 | **Browsing by Category** | Treemap of categories (Productivity, Streaming, Malicious, …). |
| 7 | **Device Risk Scores** | 0–100 composite score per device with severity bucketing. |
| 8 | **Traffic by Hour** | Stacked safe-vs-flagged timeline, peak and most-flagged hours called out. |
| 9 | **Unique Domains per Device** | Surface-area indicator. |
| 10 | **Device Enrollment Timeline** | Enrollment volume per month. |
| 11 | **Managed vs Unmanaged** | Mix of managed devices in the fleet. |
| 12 | **Compliance Status** | Compliant / Non-compliant distribution. |

Every chart has a **More**, **See more** or **Take Action** affordance
that opens a detail popup with search, filtering and bulk actions.

### 3.2 Policy actions

- **Block / Unblock at device scope.** Updates the system rule-set, bumps
  the rule-set version and publishes via the UEM configuration API.
- **Block / Unblock at application scope.** Inserts a new
  `DeviceTrafficRule` and `TrafficRulesApplicationMapping` in a
  dedicated app-scoped rule-set, scoped to the application's bundle ID
  (executable path on Windows). Authoritative blocked-domain state is
  pulled back from the UEM **v2 query API** to confirm the change.
- **Live Device Traffic Rules viewer.** Inspect the consolidated rule
  set straight from the UEM SQL store, grouped by rule-set, with
  destination patterns and per-app mappings visible.

### 3.3 Administrative configuration

A separate **Configuration** page (`/configuration` in the SPA) lets an
administrator manage the supporting data used by the analytics layer:

- Applications catalog (display name, icon, executable path / bundle ID).
- Master domain list and red-flag domain list.
- Red-flag *reasons* shown in the **Why are these flagged?** popup.
- Domain → category mappings (used by the Treemap).
- Per-graph visibility toggles, titles and tunable messages
  (`server/data/config.json`).

### 3.4 Resilience

If the SQL Server is unreachable, the service automatically falls back
to deterministic JSON snapshots under `server/data/*.json` so the UI
remains useful for demos and screenshots.

---

## 4. Solution architecture

```
                ┌──────────────────────────────────────────────────────┐
                │                       IIS                            │
 Browser ──HTTPS▶│   Default Web Site  ·  host: snl.ssdevrd.com         │
                │                                                      │
                │   /snlweb/  ─────▶  C:\snl\web\        (static SPA)  │
                │                                                      │
                │   /snlapi/  ─────▶  C:\snl\api\        (URL Rewrite) │
                │                       │                              │
                └───────────────────────┼──────────────────────────────┘
                                        │  rewrite  ^api/(.*)
                                        ▼  →  http://localhost:3001/api/{R:1}
                              ┌──────────────────────────────┐
                              │  TunnelSNL API service       │
                              │  Node.js + Express + MSSQL   │
                              │  http://localhost:3001       │
                              │  (independent of UEM core)   │
                              └──────────┬────────────┬──────┘
                                         │            │
                            reads / writes│            │HTTPS (publish & query)
                                         ▼            ▼
                              ┌────────────────┐  ┌────────────────────────────┐
                              │ MS SQL Server  │  │ UEM / MDM backend          │
                              │  snl-db        │  │ snl.ssdevrd.com/API/mdm/.. │
                              │  10.14.65.238  │  │ (config v1 + query v2)     │
                              └────────────────┘  └────────────────────────────┘
```

**Traffic flow.** The browser only ever sees one origin
(`https://snl.ssdevrd.com`). IIS serves the static SPA from
`/snlweb/` and reverse-proxies `/snlapi/api/*` to the Node service on
`localhost:3001`. The Node service performs all SQL queries and all
UEM API calls server-side, never exposing credentials or internal
hostnames to the browser.

---

## 5. Decoupling rationale — independent service layer

A core design decision of this PoC is that **the analytics service does
not live inside the UEM product**. It runs as its own Node process with
its own data plane and its own deployment lifecycle.

| Concern | How decoupling helps |
| ------- | -------------------- |
| **Iteration speed** | New visualizations, new policy controls, and UI experiments ship on the PoC cadence (hours/days) without waiting for a UEM release train. |
| **Risk isolation** | A bug in the PoC cannot affect the UEM console, agent, or device fleet. The PoC has read-mostly access to the database and only writes through the same public APIs that any third-party integrator would use. |
| **Demonstrability** | The PoC can be stood up against any UEM environment by changing two URLs (DB connection + UEM API base) and one auth header. No code branches in UEM are required. |
| **Future deployment options** | The same service can later be repackaged as a sidecar in the UEM topology, as a SaaS analytics tier, or as an on-prem add-on, without architectural rewrites. |
| **Tech freedom** | The PoC team picks the best tools for analytics UX (React 19, Recharts, Vite) without inheriting constraints from the UEM stack. |
| **Read/write segregation** | Read paths (analytics) go straight to the database for performance. Write paths (block/unblock) always round-trip through the **public UEM configuration API**, so the UEM remains the system of record. |

In short: the PoC is *compositional*, not *invasive*.

---

## 6. Service-layer responsibilities

`server/index.js` is the only backend component. It hosts an Express
application that exposes the JSON API consumed by the SPA and owns three
outbound dependencies:

### 6.1 SQL Server (`snl-db`)

Used for **all read paths**: device inventory, OS distribution,
compliance, applications, traffic-rule listing, enrollment history.
Configured at the top of `server/index.js`:

```js
const sqlConfig = {
  server: '10.14.65.238',
  database: 'snl-db',
  options: { trustServerCertificate: true, trustedConnection: true },
  driver: 'msnodesqlv8',
  authentication: {
    type: 'default',
    options: { domain: '', userName: 'sa', password: '<redacted>' },
  },
  pool: { max: 10, min: 0, idleTimeoutMillis: 30000 },
};
```

The driver is **`msnodesqlv8`** (chosen for native Windows
authentication support on the deployment box).

### 6.2 UEM configuration API (write path)

When an administrator clicks **Block** or **Unblock** the service:

1. Updates the relevant `DeviceTrafficRule` / `TrafficRulesApplicationMapping`
   row in SQL.
2. Bumps the rule-set `Version`.
3. POSTs to the UEM configuration endpoint to publish the new rule-set
   to the agent fleet.

Endpoint (overridable via env):

```
POST https://snl.ssdevrd.com/API/mdm/tunnel/configuration-actions/
     device-traffic-rule-sets
```

### 6.3 UEM v2 query API (authoritative read-back)

Immediately after a successful publish, the service calls the v2 query
API to fetch the authoritative list of blocked domains per application
and reconciles it with local state. The dashboard reflects the resulting
authoritative status (not optimistic UI).

```
GET  https://snl.ssdevrd.com/API/mdm/tunnel/v2/device-traffic-rule-sets
```

### 6.4 Local JSON data (fallback & configuration)

The service also reads / writes a handful of JSON files under
`server/data/`:

| File | Purpose |
| ---- | ------- |
| `config.json` | Chart visibility, titles, default messages. |
| `applications.json` | Catalog of monitored applications (icon, name, executable path). |
| `masterDomains.json` | Seed domain list used for synthetic demo data. |
| `redFlagDomains.json` | Red-flag domain list. |
| `redFlagReasons.json` | Threat-intel reason text per domain. |
| `domainCategories.json` | Domain → category mapping for the Treemap. |
| `deviceDomainsSnapshot.json` | Cached chart snapshot for offline / SQL-down mode. |
| `appRedFlagsSnapshot.json` | App-level red-flag snapshot. |

These files are intentionally human-editable so the PoC can be re-shaped
quickly for different demos.

---

## 7. Technology stack & rationale

| Layer | Choice | Why |
| ----- | ------ | --- |
| **UI framework** | React 19 | Industry-standard, vast component ecosystem, concurrent rendering helps with chart-heavy pages. |
| **Charting** | Recharts 3 | Declarative, composable React charts. Custom shapes, treemaps, and per-chart customization map naturally to React props. |
| **Build** | Vite 8 | Sub-second HMR, native ESM, simple production builds. |
| **Routing** | React Router 7 | Stable browser routing for the SPA's Dashboard / Configuration pages. |
| **API runtime** | Node.js 20 LTS + Express 5 | Minimal surface, easy to operate as a Windows service. |
| **SQL access** | `mssql` + `msnodesqlv8` | Native Windows-auth path to the existing UEM SQL Server. |
| **Hosting** | IIS + URL Rewrite | Reuses existing Windows infrastructure; no new ingress to operate. |
| **Process supervision** | NSSM (recommended) or interactive `npm start` | NSSM provides auto-start on reboot and crash recovery without introducing a containerization story for the PoC. |

---

## 8. Live sample URLs

| Surface | URL |
| ------- | --- |
| Web application | <https://snl.ssdevrd.com/snlweb/> |
| API (via IIS reverse proxy) — sample endpoint | <https://snl.ssdevrd.com/snlapi/api/traffic-rules> |
| API (via IIS reverse proxy) — devices | <https://snl.ssdevrd.com/snlapi/api/devices> |
| API (via IIS reverse proxy) — red-flag analytics | <https://snl.ssdevrd.com/snlapi/api/analytics/redflag-domains> |

All API responses are JSON. The reverse proxy preserves query strings
(`appendQueryString="true"`).

---

## 9. Repository layout

```
tunnelsnl/
├── public/                          Static assets bundled by Vite
├── src/                             React app
│   ├── components/                  Shared UI (ConfirmModal, Notification, …)
│   ├── context/AppContext.jsx       Global state + API client (one source of API_BASE)
│   └── pages/
│       ├── Dashboard.jsx            12 charts + popups + per-chart settings
│       └── Configuration.jsx        Admin editor for applications / domains / categories
├── server/
│   ├── index.js                     Express app + SQL + UEM integration (~1.2k LoC)
│   ├── data/                        JSON seed & snapshot files
│   ├── package.json                 Server-only dependencies (express, cors, mssql)
│   └── package-lock.json
├── package.json                     Web app (React + Vite), homepage: "/snlweb"
├── vite.config.js
├── index.html
└── README.md                        (this file)
```

`package.json` sets `"homepage": "/snlweb"` so that Vite emits an
`index.html` whose asset URLs are prefixed with `/snlweb/`. This must
match the IIS application alias chosen in §12.

---

## 10. Local development

**Prerequisites**

- Node.js 20+ (LTS)
- Optional: VPN / network access to the UEM SQL Server at
  `10.14.65.238`. If unavailable, the service falls back to the JSON
  snapshots automatically.

**Boot sequence**

```powershell
# 1. Install web-app dependencies
npm ci

# 2. Install API dependencies
cd server
npm ci
cd ..

# 3. Start the API service (terminal A) — listens on http://localhost:3001
cd server
npm start

# 4. Start the Vite dev server (terminal B) — http://localhost:5173
npm run dev
```

**Pointing the SPA at the right API base.** Open
`src/context/AppContext.jsx` and toggle between local development and
the proxied production base:

```js
// Local development
// const API_BASE = 'http://localhost:3001/api';
// Production behind IIS proxy
const API_BASE = 'https://snl.ssdevrd.com/snlapi/api';
```

For local dev you'll use the commented line. Vite's HMR will reload the
app immediately.

---

## 11. Production build

```powershell
# Lint (optional but recommended pre-build)
npm run lint

# Bundle the SPA into .\dist
npm run build

# Locally preview the production bundle
npm run preview
```

`npm run build` emits a fully hashed, minified bundle:

```
dist/
├── index.html
└── assets/
    ├── index-<hash>.js   ~720 kB (~210 kB gzipped)
    └── index-<hash>.css  ~22 kB  (~5 kB  gzipped)
```

---

## 12. Deployment to IIS

The live PoC is hosted as **two sibling IIS applications under the
existing Default Web Site**, with the Node API running out-of-band on
`localhost:3001`. This is intentionally light-touch: no new IIS sites,
no new bindings, no new certificates.

```
Default Web Site  (https://snl.ssdevrd.com)
├── /snlweb/  ──►  C:\snl\web   (static SPA — copy of dist\)
└── /snlapi/  ──►  C:\snl\api   (web.config only — URL Rewrite proxy)

Node API process (managed via NSSM or interactive shell)
└── http://localhost:3001
    Working dir: e.g. C:\snl\api-src\server
```

### 12.1 One-time prerequisites on the server

Run PowerShell **as Administrator**:

```powershell
# Enable IIS basics
Enable-WindowsOptionalFeature -Online -All -FeatureName `
  IIS-WebServerRole, IIS-WebServer, IIS-CommonHttpFeatures, `
  IIS-StaticContent, IIS-DefaultDocument, IIS-RequestFiltering, `
  IIS-ManagementConsole

# URL Rewrite (required for the proxy rule)
winget install Microsoft.URLRewrite

# Node.js LTS (v20+)
winget install OpenJS.NodeJS.LTS

# (Optional, recommended) NSSM to run the API as a Windows service
winget install NSSM.NSSM
```

> **About ARR.** Because the rewrite target is `http://localhost:...`
> on the *same* box, **Application Request Routing (ARR) is not
> required** — URL Rewrite alone proxies to localhost. ARR is only
> needed if the API later moves to a remote host (then enable IIS Manager →
> server node → *Application Request Routing Cache* → *Server Proxy
> Settings* → check **Enable proxy**).

### 12.2 Create the folders

```powershell
New-Item -ItemType Directory -Force C:\snl\web | Out-Null
New-Item -ItemType Directory -Force C:\snl\api | Out-Null
```

### 12.3 Deploy the SPA (`snlweb`)

1. On your dev machine, build:

   ```powershell
   cd C:\path\to\tunnelsnl
   npm ci
   npm run build
   ```

2. Copy the contents of `dist\` into `C:\snl\web\`:

   ```powershell
   robocopy .\dist C:\snl\web /E
   ```

3. In **IIS Manager**:

   - Right-click **Default Web Site** → **Add Application…**
   - **Alias:** `snlweb`
   - **Physical path:** `C:\snl\web`
   - Application pool: **No Managed Code**.

4. Verify by browsing to <https://snl.ssdevrd.com/snlweb/>. The page
   should load (with an error banner until §12.4 is done because the
   API isn't reachable yet).

### 12.4 Deploy the API reverse proxy (`snlapi`)

1. Create **`C:\snl\api\web.config`** with the following content. This
   is the only file in `C:\snl\api`. It tells IIS to rewrite every
   request matching `^api/(.*)` to the local Node service on port 3001:

   ```xml
   <?xml version="1.0" encoding="UTF-8"?>
   <configuration>
     <system.webServer>
       <rewrite>
         <rules>
           <rule name="ProxyApiToLocalhost3001" stopProcessing="true">
             <match url="^api/(.*)" />
             <action
               type="Rewrite"
               url="http://localhost:3001/api/{R:1}"
               appendQueryString="true" />
           </rule>
         </rules>
       </rewrite>
     </system.webServer>
   </configuration>
   ```

2. In **IIS Manager**:

   - Right-click **Default Web Site** → **Add Application…**
   - **Alias:** `snlapi`
   - **Physical path:** `C:\snl\api`
   - Same app pool as `snlweb` is fine (**No Managed Code**).

3. Start the Node API (§12.5), then validate end-to-end:

   ```powershell
   curl https://snl.ssdevrd.com/snlapi/api/traffic-rules
   ```

   You should see a JSON array of traffic rules.

### 12.5 Run the API service

The Node service is **not** hosted by IIS — IIS only forwards requests
to it. Copy `server/` anywhere on the server box (e.g.
`C:\snl\api-src\server`), install dependencies, and start it.

**Option A — Interactive (good for demos & first-time validation)**

```powershell
Expand-Archive .\server.zip -DestinationPath C:\snl\api-src -Force

cd C:\snl\api-src\server
npm ci --omit=dev

# Start the API — listens on http://localhost:3001
npm start
```

**Option B — Windows service via NSSM (recommended for any
multi-day deployment)**

```powershell
nssm install TunnelSNL-API "C:\Program Files\nodejs\node.exe" `
  "C:\snl\api-src\server\index.js"

nssm set TunnelSNL-API AppDirectory   "C:\snl\api-src\server"
nssm set TunnelSNL-API AppStdout      "C:\snl\api-src\server\logs\out.log"
nssm set TunnelSNL-API AppStderr      "C:\snl\api-src\server\logs\err.log"
nssm set TunnelSNL-API Start          SERVICE_AUTO_START

New-Item -ItemType Directory -Force C:\snl\api-src\server\logs | Out-Null
nssm start TunnelSNL-API

# Sanity check
Invoke-WebRequest http://localhost:3001/api/traffic-rules -UseBasicParsing |
  Select-Object -ExpandProperty StatusCode
```

Useful management commands:

```powershell
nssm restart TunnelSNL-API
nssm stop    TunnelSNL-API
nssm remove  TunnelSNL-API confirm    # uninstall
```

### 12.6 Redeploy / update cycle

```powershell
# --- on dev machine ---
npm ci
npm run build

# --- copy artifacts to the server (adjust paths / share as needed) ---
robocopy .\dist           \\SERVER\c$\snl\web                /E /MIR
robocopy .\server         \\SERVER\c$\snl\api-src\server     /E /XD node_modules

# --- on the server ---
cd C:\snl\api-src\server
npm ci --omit=dev
nssm restart TunnelSNL-API           # or restart your interactive shell
iisreset /noforce
```

> `/MIR` mirrors the new build into the static folder; it is safe
> because `C:\snl\web\` only holds Vite output. The `web.config` in
> `C:\snl\api\` is **not** touched by these commands.

---

## 13. Runtime configuration reference

### 13.1 Database connection

Open `server/index.js` and edit `sqlConfig` to point at the target SQL
Server. For non-PoC deployments, move credentials into environment
variables and consume them via `process.env`.

### 13.2 Environment variables consumed by the API

Set these via `nssm set TunnelSNL-API AppEnvironmentExtra KEY=VALUE`
(NSSM) or via your shell session (interactive).

| Variable | Default | Purpose |
| -------- | ------- | ------- |
| `EXTERNAL_BLOCK_API_BASE` | `https://snl.ssdevrd.com/API/mdm/tunnel/configuration-actions/device-traffic-rule-sets` | UEM rule-set configuration endpoint (publish path). |
| `EXTERNAL_BLOCK_OG_UUID`  | `15269a42-de03-4b53-8c54-d12617118205` | Organization Group / tenant UUID for the UEM API. |
| `EXTERNAL_BLOCK_AUTH`     | _(empty)_ | Authorization header forwarded to the UEM API (e.g. `Bearer …` or `Basic …`). |
| `MDM_V2_LIST_URL`         | `https://snl.ssdevrd.com/API/mdm/tunnel/v2/device-traffic-rule-sets` | UEM v2 query endpoint used to read authoritative per-app blocked-domain state. |
| `PORT`                    | `3001` | TCP port for the Express server. Must match the URL Rewrite target in §12.4. |

### 13.3 Front-end API base

Set in `src/context/AppContext.jsx`:

```js
const API_BASE = 'https://snl.ssdevrd.com/snlapi/api';
```

For staging vs production builds, this is the single line that needs
updating before `npm run build`.

---

## 14. API surface (reference)

Selected endpoints (all returning JSON). Full list lives in
`server/index.js`.

### 14.1 Analytics (read)

| Method | Path | Description |
| ------ | ---- | ----------- |
| GET | `/api/devices` | Full device inventory. |
| GET | `/api/devices/:deviceId/domains` | Per-device domain visit history. |
| GET | `/api/analytics/top-domains` | Top visited domains, fleet-wide. |
| GET | `/api/analytics/redflag-domains` | Red-flag domains with device counts. |
| GET | `/api/analytics/all-domain-data` | Raw per-device domain data (used for client-side cross-cuts). |
| GET | `/api/analytics/browsing-categories` | Treemap data (category → totalVisits / deviceCount). |
| GET | `/api/analytics/domain-risk-scores` | Per-device risk scores (0–100). |
| GET | `/api/analytics/hourly-traffic` | Safe vs flagged traffic per hour. |
| GET | `/api/analytics/unique-domains-per-device` | Distinct safe / flagged domains per device. |
| GET | `/api/analytics/app-redflag-domains` | Per-application red-flag rollups. |
| GET | `/api/traffic-rules` | Consolidated rule-set listing from SQL. |
| GET | `/api/applications/blocked-domains` | Authoritative blocked-domain state from UEM v2. |

### 14.2 Policy actions (write — round-trips through UEM)

| Method | Path | Description |
| ------ | ---- | ----------- |
| POST | `/api/domains/block` | Block a domain at device scope. |
| POST | `/api/domains/unblock` | Unblock a domain at device scope. |
| POST | `/api/applications/block-domain` | Block a domain for a single application. |
| POST | `/api/applications/unblock-domain` | Unblock a domain for a single application. |
| POST | `/api/applications/reset-redflag-data` | Reset per-app red-flag snapshot (PoC convenience). |

### 14.3 Configuration & seed data

| Method | Path | Description |
| ------ | ---- | ----------- |
| GET/PUT | `/api/config` | Chart visibility, titles, messages. |
| GET/PUT | `/api/data/applications` | Applications catalog. |
| GET/PUT | `/api/data/master-domains` | Seed domain list. |
| GET/PUT | `/api/data/redflag-domains` | Red-flag domain list. |
| GET/PUT | `/api/data/redflag-reasons` | Threat-intel reasons per domain. |
| GET/PUT | `/api/data/domain-categories` | Domain → category mapping. |
| POST | `/api/refresh` | Regenerate analytics snapshots. |
| POST | `/api/chart-data/reset` | Reset cached chart data. |

---

## 15. Operations — smoke tests & troubleshooting

### 15.1 Smoke test

```powershell
# Node API directly
Invoke-WebRequest http://localhost:3001/api/traffic-rules -UseBasicParsing

# API via IIS reverse proxy
Invoke-WebRequest https://snl.ssdevrd.com/snlapi/api/traffic-rules -UseBasicParsing

# SPA
Start-Process https://snl.ssdevrd.com/snlweb/
```

End-user verification:

- Dashboard renders without an error banner.
- DevTools **Network** tab shows requests to
  `https://snl.ssdevrd.com/snlapi/api/...` returning HTTP 200.
- Clicking **Block** / **Unblock** on a domain produces a success toast
  and the relevant chart re-renders.

### 15.2 Troubleshooting matrix

| Symptom | Likely cause / fix |
| ------- | ------------------ |
| `/snlweb/` shows IIS default page or 403 | The `snlweb` IIS application was not created, or `C:\snl\web\index.html` is missing. Re-run §12.3. |
| Assets 404 with paths like `/assets/...` | `package.json` `homepage` is not `/snlweb`, or only `dist\index.html` was copied. Rebuild and recopy `dist\`. |
| `/snlapi/api/*` returns **500.19** | `web.config` malformed or **URL Rewrite** not installed on the IIS box. Install URL Rewrite (§12.1) and reset IIS. |
| `/snlapi/api/*` returns **502 / 504** | The Node API is not running. Run `Get-Service TunnelSNL-API` or `npm start` in `C:\snl\api-src\server`. |
| Dashboard loads but every chart says **"No data"** | API is up but SQL is unreachable — the service falls back to seed JSON. Inspect `server\logs\err.log`. |
| Block / Unblock returns success but UEM state doesn't update | `EXTERNAL_BLOCK_AUTH` is empty / expired, or the UEM endpoint is unreachable from the API box. |
| CORS errors in browser DevTools | The build is still pointing at `http://localhost:3001/api`. Update `API_BASE` in `src/context/AppContext.jsx` to `https://snl.ssdevrd.com/snlapi/api`, rebuild, redeploy. |

### 15.3 Quick command reference

```powershell
# Build
npm ci
npm run build
cd server ; npm ci --omit=dev ; cd ..

# Deploy (single-box example; adjust paths as needed)
robocopy .\dist   C:\snl\web              /E /MIR
robocopy .\server C:\snl\api-src\server   /E /XD node_modules
cd C:\snl\api-src\server ; npm ci --omit=dev ; cd \

# Start API
cd C:\snl\api-src\server
npm start                       # interactive
# or
nssm restart TunnelSNL-API      # service-managed

# Reload IIS
iisreset /noforce
```

---

## 16. Productionization roadmap

The current PoC is intentionally minimal. The following work would move
it from "shareable demo" to "supported product extension":

1. **Secrets management.** Move SQL credentials and UEM auth headers
   into a secrets store (Windows DPAPI, Azure Key Vault, or HashiCorp
   Vault) instead of `server/index.js` / NSSM env vars.
2. **AuthN / AuthZ.** Front the SPA and API with the UEM SSO provider
   (OIDC / SAML). Restrict policy actions to administrators with the
   appropriate role.
3. **High availability.** Run two API instances behind an ARR /
   load-balancer; move the JSON snapshot store to a shared location or
   replace it with a small SQL table.
4. **Observability.** Emit structured logs and metrics (request
   latency, UEM API error rate, SQL query timings) to the platform's
   standard log sink.
5. **Test coverage.** Add unit tests for the policy-action path
   (synthetic UEM mocks) and Playwright tests for the four critical
   user journeys (view fleet, see red flags, block at device scope,
   block at app scope).
6. **Internationalization.** All chart titles, hints and confirmation
   messages already flow through `config.json`; add a language switch
   on top of that.
7. **Audit trail.** Persist every block / unblock with the actor,
   timestamp, target, scope and UEM response — exposed in a new
   *Audit Log* page.
8. **Packaging.** Ship the Node service as a single MSI (Node + code +
   NSSM wrapper) so an operator only runs one installer.

---

## 17. Glossary

| Term | Meaning |
| ---- | ------- |
| **UEM** | Unified Endpoint Management — the parent product. |
| **MDM** | Mobile Device Management — used interchangeably with UEM in this codebase for historical API paths. |
| **Tunnel** | UEM's per-device VPN / traffic-policy agent. |
| **Rule-set** | A collection of traffic rules attached to a device or application. |
| **Device-scope block** | A traffic rule that blocks a domain for *every* application on a device. |
| **App-scope (per-app) block** | A traffic rule scoped to a specific application's executable path, so the same domain can be allowed for other apps. |
| **Red-flag domain** | A domain marked as malicious, phishing, mining, or otherwise high-risk by threat-intel feeds or curation. |
| **OG (Organization Group)** | UEM tenancy unit; identified by `EXTERNAL_BLOCK_OG_UUID`. |
| **Risk score** | 0–100 composite per device blending flagged-visit ratio, flagged-domain breadth, behavioral anomalies, and device posture. |

---

*TunnelSNL — Proof of Concept. Internal project. All rights reserved.*
