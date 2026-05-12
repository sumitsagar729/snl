# TunnelSNL – IIS Deployment Guide

This document explains how to build and deploy the **client** (React + Vite) and the **server** (Node.js + Express + MSSQL) on a Windows box running **IIS**.

---

## 1. Architecture overview

| Piece       | Tech                        | Listens on              | Deploy strategy                                                  |
| ----------- | --------------------------- | ----------------------- | ---------------------------------------------------------------- |
| Client      | React 19 + Vite             | Static files (`dist/`)  | IIS site serving static files + URL Rewrite for SPA routing      |
| Server/API  | Node.js + Express + `mssql` | `http://localhost:3001` | Windows Service (via NSSM or PM2) fronted by IIS reverse proxy   |

The recommended topology:

```
Browser ──► IIS (port 80/443) ─┬──► dist\  (static files for the React app)
                                └──► /api/*  reverse-proxied to http://localhost:3001
```

> Note: `src/context/AppContext.jsx` currently hard-codes `API_BASE = 'http://localhost:3001/api'`. To make the build work on end-user machines behind IIS, change that line to `const API_BASE = '/api';` **before** running `npm run build`. The reverse proxy rule below forwards `/api/*` to the Node server.

---

## 2. Prerequisites (one-time, on the IIS server)

Run PowerShell **as Administrator**.

```powershell
# 1. Enable IIS and required features
Enable-WindowsOptionalFeature -Online -FeatureName IIS-WebServerRole, IIS-WebServer, `
  IIS-CommonHttpFeatures, IIS-HttpErrors, IIS-HttpRedirect, IIS-ApplicationDevelopment, `
  IIS-NetFxExtensibility45, IIS-HealthAndDiagnostics, IIS-HttpLogging, IIS-Security, `
  IIS-RequestFiltering, IIS-Performance, IIS-WebServerManagementTools, IIS-ManagementConsole, `
  IIS-StaticContent, IIS-DefaultDocument, IIS-DirectoryBrowsing -All

# 2. Install URL Rewrite + Application Request Routing (needed for SPA fallback and /api proxy)
#    Download and install these two MSIs from Microsoft:
#    https://www.iis.net/downloads/microsoft/url-rewrite
#    https://www.iis.net/downloads/microsoft/application-request-routing
#    (or use Web Platform Installer / winget)
winget install Microsoft.URLRewrite
winget install Microsoft.ApplicationRequestRouting

# 3. Install Node.js LTS (v20+)
winget install OpenJS.NodeJS.LTS

# 4. Install NSSM (to run the Node server as a Windows service)
winget install NSSM.NSSM
```

After installing ARR, enable the **proxy** toggle once:

1. Open **IIS Manager** → click the server node (top-left).
2. Double-click **Application Request Routing Cache**.
3. Click **Server Proxy Settings…** on the right.
4. Check **Enable proxy** → **Apply**.

---

## 3. Build the client

From the repo root on your dev machine (or the server – either works):

```powershell
cd C:\Users\sagarsu\Documents\snl\tunnelsnl

# Install deps (only first time or when package.json changes)
npm ci

# Produce a production build in .\dist
npm run build
```

The build artifacts land in `dist\` (index.html, assets, etc.).

---

## 4. Build / prepare the server

```powershell
cd C:\Users\sagarsu\Documents\snl\tunnelsnl\server

# Install production deps only
npm ci --omit=dev

# The 'mssql' driver uses msnodesqlv8 for trusted/SQL auth on Windows.
# If it's not already a dep, add it:
npm install msnodesqlv8 --save
```

There is no separate “build” step for the server – Node runs `index.js` directly.

---

## 5. Copy files to the server

Pick a deployment root, e.g. `C:\inetpub\tunnelsnl`.

```powershell
$Target = 'C:\inetpub\tunnelsnl'
New-Item -ItemType Directory -Force -Path "$Target\client", "$Target\server" | Out-Null

# Client: only the dist folder is needed
Copy-Item -Recurse -Force .\dist\*            "$Target\client\"

# Server: everything except node_modules (we'll reinstall on the box)
robocopy .\server "$Target\server" /E /XD node_modules

# On the server box:
cd "$Target\server"
npm ci --omit=dev
```

Grant the IIS app-pool identity read access to `client\` and the Node service account read/write to `server\data\`:

```powershell
icacls "$Target\client" /grant "IIS_IUSRS:(OI)(CI)RX" /T
icacls "$Target\server\data" /grant "NETWORK SERVICE:(OI)(CI)M" /T
```

---

## 6. Run the Node API as a Windows service (NSSM)

Using NSSM is the most reliable way on Windows — it auto-starts on boot and restarts on crash.

```powershell
# Install the service
nssm install TunnelSNL-API "C:\Program Files\nodejs\node.exe" "C:\inetpub\tunnelsnl\server\index.js"

# Configure it
nssm set TunnelSNL-API AppDirectory      "C:\inetpub\tunnelsnl\server"
nssm set TunnelSNL-API AppStdout         "C:\inetpub\tunnelsnl\server\logs\out.log"
nssm set TunnelSNL-API AppStderr         "C:\inetpub\tunnelsnl\server\logs\err.log"
nssm set TunnelSNL-API Start             SERVICE_AUTO_START
nssm set TunnelSNL-API AppEnvironmentExtra NODE_ENV=production

# Create log dir and start
New-Item -ItemType Directory -Force C:\inetpub\tunnelsnl\server\logs | Out-Null
nssm start TunnelSNL-API

# Verify
Get-Service TunnelSNL-API
Invoke-WebRequest http://localhost:3001/api/devices -UseBasicParsing | Select-Object -ExpandProperty StatusCode
```

Useful management commands:

```powershell
nssm restart TunnelSNL-API
nssm stop    TunnelSNL-API
nssm remove  TunnelSNL-API confirm   # uninstall
```

---

## 7. Create the IIS site for the client

```powershell
Import-Module WebAdministration

# App pool (No Managed Code – this is just a static site)
New-WebAppPool -Name TunnelSNL
Set-ItemProperty IIS:\AppPools\TunnelSNL -Name managedRuntimeVersion -Value ''

# Site bound to port 80 (change as needed)
New-Website -Name TunnelSNL `
            -Port 80 `
            -PhysicalPath 'C:\inetpub\tunnelsnl\client' `
            -ApplicationPool TunnelSNL
```

For HTTPS, add a binding after installing a certificate:

```powershell
New-WebBinding -Name TunnelSNL -Protocol https -Port 443 -SslFlags 0
# Then bind the cert in IIS Manager or via netsh / New-Item IIS:\SslBindings\...
```

---

## 8. `web.config` for the client (SPA routing + API reverse proxy)

Drop this file at `C:\inetpub\tunnelsnl\client\web.config`. It does two things:

1. Reverse-proxies every request that starts with `/api/` to `http://localhost:3001/api/`.
2. Sends everything else that is **not** a real file/folder back to `index.html`, so React Router (BrowserRouter) works.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
  <system.webServer>
    <rewrite>
      <rules>

        <!-- 1) Reverse proxy /api/* to the Node server -->
        <rule name="ReverseProxyToNodeAPI" stopProcessing="true">
          <match url="^api/(.*)" />
          <action type="Rewrite" url="http://localhost:3001/api/{R:1}" />
        </rule>

        <!-- 2) SPA fallback: serve index.html for any non-file/non-dir URL -->
        <rule name="ReactRouter" stopProcessing="true">
          <match url=".*" />
          <conditions logicalGrouping="MatchAll">
            <add input="{REQUEST_FILENAME}" matchType="IsFile"      negate="true" />
            <add input="{REQUEST_FILENAME}" matchType="IsDirectory" negate="true" />
            <add input="{REQUEST_URI}"      pattern="^/api/"        negate="true" />
          </conditions>
          <action type="Rewrite" url="/index.html" />
        </rule>

      </rules>
    </rewrite>

    <!-- Cache static assets aggressively, but not index.html -->
    <staticContent>
      <clientCache cacheControlMode="UseMaxAge" cacheControlMaxAge="365.00:00:00" />
    </staticContent>

    <httpProtocol>
      <customHeaders>
        <add name="X-Content-Type-Options" value="nosniff" />
      </customHeaders>
    </httpProtocol>
  </system.webServer>
</configuration>
```

> The `ReverseProxyToNodeAPI` rule requires both **URL Rewrite** and **Application Request Routing** to be installed and ARR’s proxy feature to be enabled (step 2 above). Without ARR, the rewrite rule cannot target an external URL and will 500.

---

## 9. Smoke-test the deployment

From the server box:

```powershell
# Node API directly
Invoke-WebRequest http://localhost:3001/api/devices -UseBasicParsing

# API via IIS reverse proxy
Invoke-WebRequest http://localhost/api/devices -UseBasicParsing

# SPA
Start-Process http://localhost/
```

From a client machine: browse to `http://<server-name>/` and verify:

- The dashboard loads.
- Network tab shows calls going to `http://<server-name>/api/...` and returning 200.
- Deep-linking a route (e.g. `/devices/abc`) refreshes cleanly without a 404.

---

## 10. Redeploy / update cycle

Quickest way to ship changes:

```powershell
# --- on dev machine ---
npm run build
cd server ; npm ci --omit=dev ; cd ..

# --- copy to server (adjust target path / share) ---
robocopy .\dist                \\SERVER\c$\inetpub\tunnelsnl\client /E /MIR /XF web.config
robocopy .\server              \\SERVER\c$\inetpub\tunnelsnl\server /E /XD node_modules

# --- on server ---
cd C:\inetpub\tunnelsnl\server
npm ci --omit=dev
nssm restart TunnelSNL-API
iisreset /noforce
```

> `robocopy … /MIR /XF web.config` mirrors the new build into the client folder **without** deleting the `web.config` you created in step 8.

---

## 11. Troubleshooting

| Symptom                                       | Likely cause / fix                                                                                       |
| --------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| 404 on page refresh for a client route        | `web.config` SPA fallback rule missing or URL Rewrite module not installed.                              |
| 500.19 on first request                       | `web.config` references modules that aren’t installed (URL Rewrite / ARR).                               |
| `/api/*` returns 502                          | Node service is down (`Get-Service TunnelSNL-API`) or ARR proxy not enabled at the server level.         |
| CORS errors in browser                        | You forgot to change `API_BASE` to `/api` and the browser is calling `localhost:3001` directly.          |
| SQL connection errors in `server\logs\err.log`| Check `sqlConfig` in `server/index.js`; ensure the service account can reach `10.14.65.238` and SQL auth is valid. Set env var `USE_FALLBACK_DATA=true` via `nssm set TunnelSNL-API AppEnvironmentExtra` to run without DB while debugging. |
| Changes to `index.html` not showing up        | Browser cached it. Hard-refresh (Ctrl+F5). Consider adding `<clientCache cacheControlMode="DisableCache" />` for `index.html` specifically. |

---

## 12. Quick command reference (copy/paste)

```powershell
# Build everything
npm ci
npm run build
cd server ; npm ci --omit=dev ; cd ..

# Deploy (local box example)
$Target = 'C:\inetpub\tunnelsnl'
robocopy .\dist   "$Target\client"  /E /MIR /XF web.config
robocopy .\server "$Target\server"  /E /XD node_modules
cd "$Target\server" ; npm ci --omit=dev ; cd \

# (First time only) create service + site
nssm install TunnelSNL-API "C:\Program Files\nodejs\node.exe" "$Target\server\index.js"
nssm set     TunnelSNL-API AppDirectory "$Target\server"
nssm start   TunnelSNL-API

Import-Module WebAdministration
New-WebAppPool TunnelSNL
Set-ItemProperty IIS:\AppPools\TunnelSNL -Name managedRuntimeVersion -Value ''
New-Website -Name TunnelSNL -Port 80 -PhysicalPath "$Target\client" -ApplicationPool TunnelSNL

# Restart
nssm restart TunnelSNL-API
iisreset /noforce
```
