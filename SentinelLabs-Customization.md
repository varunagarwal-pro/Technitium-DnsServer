# 🛠️ Sentinel-Labs Technitium DNS Server: Customization & Deployment Guide

![Project](https://img.shields.io/badge/Sentinel--Labs-Enterprise-0A66C2?style=flat-square)
![Status](https://img.shields.io/badge/Status-Active-brightgreen?style=flat-square)
![Version](https://img.shields.io/badge/Version-15.6.2-blue?style=flat-square)

> **Purpose:** Canonical source of truth for all customizations, modified files, build steps, and fleet deployment procedures for `Technitium-DnsServer`. Future agents must consult this document directly to avoid rediscovery overhead.

---

## 1. 🏗️ Repository Topology & Upstream Relationship

- **Fork Source:** `https://github.com/TechnitiumSoftware/DnsServer.git`
- **Our Repository (`origin`):** `https://github.com/varunagarwal-pro/Technitium-DnsServer.git`
- **Upstream Remote (`upstream`):** `https://github.com/TechnitiumSoftware/DnsServer.git`
- **Default Branch:** `master`
- **Dependency Repository:** `D:\Sentinel-Labs\TechnitiumLibrary\` (Tracked directly in superproject; pure stock upstream from `https://github.com/TechnitiumSoftware/TechnitiumLibrary.git`).

---

## 2. 📦 Inventory of Sentinel-Labs Customizations

The table below catalogs every file modified or added by Sentinel-Labs on top of upstream:

| Component / File Path | Customization Purpose & Mechanism |
|---|---|
| [`DnsServerCore/Dns/DnsServer.cs`](file:///D:/Sentinel-Labs/Technitium-DnsServer/DnsServerCore/Dns/DnsServer.cs) | **CNAME / ANAME Hop Isolation:** Each hop gets a fresh `ResolverContext`, preventing premature `MaxOutboundRequests` exhaustion. **DoT/TCP resilience (C1):** resolver-capacity drops and TCP QPM-limited requests receive per-query SERVFAIL instead of closing the stream; intentional silent-drop policies remain unchanged. **DoT visibility (C2):** TLS authentication failures, handshake I/O errors, and timeouts are globally limited to one log event per 10 seconds and include client IP, SNI, and reason. |
| [`DnsServerCore/www/`](file:///D:/Sentinel-Labs/Technitium-DnsServer/DnsServerCore/www/) | **Stock Upstream UI (Pristine):** Maintained as pure, untouched stock upstream to guarantee 100% conflict-free upstream git merges and baseline compatibility. Default fallback interface. |
| [`sentinel-webui/`](file:///D:/Sentinel-Labs/Technitium-DnsServer/sentinel-webui/) | **Sentinel-Labs Enterprise Glassmorphism UI:** Standalone, decoupled home of our modern design system (`--glass-radius`, backdrop-filters, custom pill tabs, floating date-range toolbar, donut chart visual analytics, `.enterprise-hero-grid`, Modern/Classic dual layout, and IST/UTC display timezone selector). Deployed to `/opt/technitium/sentinel-webui/` via `DNS_SERVER_WEB_SERVICE_WWW_FOLDER_PATH`. |
| [`technitium-webui/`](file:///D:/Sentinel-Labs/technitium-webui/) | **Modern React 19 SPA Console:** Alternative decoupled administration console (`byGarcia/technitium-console`). Features dedicated Clients page, 2-second live streaming, Command Palette (Ctrl+K), and cluster sync views. Deployed to `/opt/technitium/technitium-console/`. |
| [`Sentinel-Core/.../technitium-advanced-blocking/src/`](../Sentinel-Core/code/services/technitium-advanced-blocking/src/) (formerly `Apps/AdvancedBlockingApp/`) | **Enhanced AdBlock Engine:** AdGuard/AdBlock syntax support, CNAME cloaking unmasking, partitioned indexes, and async dnsrewrite. B1–B4 add endpoint suffix/wildcard precedence, bounded regex matching, opt-in decision logging, and snapshot reloads that reuse unchanged regexes and preserve active snapshots on read/build failures. AdBlock rules and group-wide `$badfilter` cancellation keys are published together; each request captures one policy for allowance, blocking, and CNAME post-processing. The shared `DnsRequestContext` associates a verified unsigned TSIG request with its original signed datagram before app callbacks, preserving policy identity without changing authentication or the post-processor's incoming-request contract. Duplicate regexes are removed within each inline array or individual feed. |

---

## 3. 🔄 Upstream Upgrade & Merge Protocol

When a new release tag (e.g. `v15.6.3`) is released by upstream:

1. **Fetch Upstream:**
   ```powershell
   git -C "D:\Sentinel-Labs\Technitium-DnsServer" fetch upstream --tags
   ```
2. **Align TechnitiumLibrary:**
   - Clone or pull latest upstream `TechnitiumLibrary` into `temp\TechnitiumLibrary`.
   - Update `D:\Sentinel-Labs\TechnitiumLibrary` via `robocopy /E /XD .git`.
   - Rebuild libraries:
     ```powershell
     dotnet build "D:\Sentinel-Labs\TechnitiumLibrary\TechnitiumLibrary.ByteTree\TechnitiumLibrary.ByteTree.csproj" -c Release
     dotnet build "D:\Sentinel-Labs\TechnitiumLibrary\TechnitiumLibrary.Net\TechnitiumLibrary.Net.csproj" -c Release
     dotnet build "D:\Sentinel-Labs\TechnitiumLibrary\TechnitiumLibrary.Security.OTP\TechnitiumLibrary.Security.OTP.csproj" -c Release
     ```
3. **Execute 3-Way Merge:**
   - Branch: `git checkout -b uprev/<date>` (see `Sentinel-Core/Docs/Upstream-Repo-Sync.md`; never use `-X ours` / `-X theirs`)
   - Merge: `git merge <tag> --no-ff --no-commit`
   - **Resolution Rules (resolve only the conflict hunks; keep every auto-merged upstream change):**
     - `DnsServer.cs`: Merges cleanly. Retain `hopContext` in CNAME/ANAME methods.
     - `main.css`: Keep our design system. Upstream `.stats-panel` styling changes do not apply because the panel is hidden.
     - `index.html`: Keep `.enterprise-hero-grid` and the hidden stats-ID block. **Do not re-add upstream's `.stats-panel` markup.** Port any new upstream dashboard behaviour (e.g. click-to-filter) onto the hero cards instead. Keep the alphabetical menu order in both nav lists. Adopt all other upstream markup changes.
   - **Verify before committing:** no duplicate element IDs, balanced tags, and the same ID set as the previous good commit (`temp/merge_repair/html_check.py`). Also confirm `git diff --stat <old-master> HEAD` touches only the files you intended.

> [!WARNING]
> **Lesson from v15.6.2 (2026-10-27):** The first v15.6.2 merge (`50e876e2`) re-added a hidden Classic `.stats-panel` with unsynchronised `*Legacy` IDs and duplicate IDs. It also deleted the hidden stats-ID block and broke tab navigation. It was redone from `7da1c2f0` on 2026-11-04 (`uprev/2026-11-04-v15.6.2`).
4. **Compile & Verify:**
   ```powershell
   dotnet build "D:\Sentinel-Labs\Technitium-DnsServer\DnsServerApp\DnsServerApp.csproj" -c Release
   ```
   Then check every theme (System, Dark Blue, Dark Grey, Light, Amber) and both layouts in a browser, at desktop and phone (~420 px) widths. Also check tab navigation, deep links, KPI / matrix links into Query Logs, and the IST / UTC display time zone.
5. **Open a PR, do not push to `master`:**
   - Push the `uprev/<date>` branch and open a PR. The `upstream-gate.yml` workflow is PR-only. Merging needs explicit owner approval.

---

## 4. 🚀 Build, Packaging & Multi-Server Deployment Pipeline

### 🌐 Target Fleet Topology

| Node Name | Role | Public IP | SSH Port | User | Identity Key |
|---|---|---|---|---|---|
| **Phoenix** | Secondary Node | `129.146.39.180` | `41022` | `vexa` | `D:\Sentinel-Labs\Sentinel-Core\Confidential\Keys\pisg_ed25519` |
| **Oregon** | Secondary Node | `35.212.133.199` | `41022` | `vexa` | `D:\Sentinel-Labs\Sentinel-Core\Confidential\Keys\pisg_ed25519` |
| **Mumbai** | **Primary Node** | `80.225.252.237` | `41022` | `vexa` | `D:\Sentinel-Labs\Sentinel-Core\Confidential\Keys\pisg_ed25519` |

> [!WARNING]
> **Dual-stack firewall parity:** Every node publishes an AAAA record for its DoT hostnames, so each inbound rule must exist for IPv6 as well as IPv4. On Oregon (GCP project `private-cloud-id`, VPC `psig-vpc`), an IPv4-only rule set silently broke Android Private DNS. On 2026-11-05 this was fixed with `psig-vpc-allow-dns-ipv6` (tcp/udp 53, tcp 853, `::/0`) and `psig-vpc-allow-https-ipv6` (tcp 443). After any firewall change, verify IPv6 TCP 853 from outside.

> [!IMPORTANT]
> **Cluster Maintenance Sequencing:** Upgrade secondary nodes (Phoenix, Oregon) before the primary node (Mumbai) by default. A simultaneous rollout requires explicit user approval accepting the shared outage window; the user explicitly overrode sequencing for the current three-node core/plugin release.

### 🧱 Step 1: Compile & Package Artifact
```powershell
# 1. Publish Release binaries
dotnet publish "D:\Sentinel-Labs\Technitium-DnsServer\DnsServerApp\DnsServerApp.csproj" -c Release -o "D:\Sentinel-Labs\Technitium-DnsServer\temp\publish"

# 2. Package tar.gz (must contain systemd.service at root)
tar -czf "D:\Sentinel-Labs\Technitium-DnsServer\temp\technitium-dns-publish.tar.gz" -C "D:\Sentinel-Labs\Technitium-DnsServer\temp\publish" .
```

### 🚀 Step 2: Deploy to Nodes (Phoenix -> Oregon -> Mumbai)
For each node in sequence:
```powershell
$key = "D:\Sentinel-Labs\Sentinel-Core\Confidential\Keys\pisg_ed25519"
$ip = "<NODE_IP>"

# 1. Upload payload and update script
scp -P 41022 -i $key "D:\Sentinel-Labs\Technitium-DnsServer\temp\technitium-dns-publish.tar.gz" "vexa@${ip}:/tmp/technitium-dns-publish.tar.gz"
scp -P 41022 -i $key "D:\Sentinel-Labs\Technitium-DnsServer\temp\update_technitium.sh" "vexa@${ip}:/tmp/update_technitium.sh"

# 2. Execute update
ssh -p 41022 -i $key "vexa@${ip}" "chmod +x /tmp/update_technitium.sh && /tmp/update_technitium.sh"

# 3. Verify DNS resolution and cleanup
ssh -p 41022 -i $key "vexa@${ip}" "dig +short @127.0.0.1 google.com; rm -f /tmp/technitium-dns-publish.tar.gz /tmp/update_technitium.sh"
```

### ⚙️ Remote Service Details
- **Installation Directory:** `/opt/technitium/dns/`
- **Config Directory:** `/etc/dns` (binary format, modified via web console or REST API)
- **Systemd Unit:** `/etc/systemd/system/dns.service` (runs as unprivileged user `dns-server:dns-server`)
- **Runtime Override:** `/etc/systemd/system/dns.service.d/override.conf`. Server GC `DOTNET_gcServer=1` (Workstation GC caused query stalls). Heap cap `0x80000000` (2 GB) on Mumbai/Phoenix and `0x10000000` (256 MB) on Oregon. See `Sentinel-Core/Docs/Infrastructure.md` §16.3–§16.4.
- **Resolver & DoT settings (live config, not in this repo):** Read-only `GET /api/settings/get` checks on **2026-11-04** returned the same values on Mumbai, Phoenix, and Oregon. Timeout values are milliseconds; each QPM entry is `prefix: UDP/TCP queries per minute`.

  | Setting | Mumbai | Phoenix | Oregon |
  |---|---:|---:|---:|
  | `tcpReceiveTimeout` | 60000 | 60000 | 60000 |
  | `tcpSendTimeout` | 60000 | 60000 | 60000 |
  | `clientTimeout` | 4000 | 4000 | 4000 |
  | IPv4 QPM prefixes | `/32: 600/600`; `/24: 6000/6000` | `/32: 600/600`; `/24: 6000/6000` | `/32: 600/600`; `/24: 6000/6000` |
  | IPv6 QPM prefixes | `/128: 600/600`; `/64: 1200/1200`; `/56: 6000/6000` | `/128: 600/600`; `/64: 1200/1200`; `/56: 6000/6000` | `/128: 600/600`; `/64: 1200/1200`; `/56: 6000/6000` |
  | QPM sample window | 5 minutes | 5 minutes | 5 minutes |
  | UDP truncation when QPM-limited | 50% | 50% | 50% |
  | QPM bypass list | Empty | Empty | Empty |
  | `resolverRetries` | 2 | 2 | 2 |
  | `resolverTimeout` | 3000 | 3000 | 3000 |
  | `resolverConcurrency` | 4 | 4 | 4 |
  | `resolverMaxStackCount` | 30 | 30 | 30 |
  | `maxConcurrentResolutionsPerCore` | 100 | 100 | 100 |

  `serveStale = false` remains the previously documented cache setting. Re-verify all live values after every upgrade.

### 🎛️ 3-Tier WebUI Switching Architecture & Deployment
Technitium DNS Server v15.5+ natively supports the environment variable `DNS_SERVER_WEB_SERVICE_WWW_FOLDER_PATH`. This allows our fleet to seamlessly hot-switch between three distinct WebUI tiers without modifying the core server binary or conflicting with upstream git updates:

| Tier | WebUI Name | Local Location | Remote Server Path | Systemd Drop-in Setting | Characteristics |
|---|---|---|---|---|---|
| **Tier 1** | **Stock Upstream UI** | `DnsServerCore/www/` | `/opt/technitium/dns/www/` | *(Unset / default)* | Pure stock upstream console; active fallback when no variable is set. 100% zero-conflict on git updates. |
| **Tier 2** | **Sentinel Glassmorphism UI** | `sentinel-webui/` | `/opt/technitium/sentinel-webui/` | `DNS_SERVER_WEB_SERVICE_WWW_FOLDER_PATH=/opt/technitium/sentinel-webui` | Enterprise Glassmorphism design system, Modern/Classic dual layout, KPI hero cards, status bar, and native IST display timezone. |
| **Tier 3** | **Technitium Console React SPA** | `technitium-webui/` | `/opt/technitium/technitium-console/` | `DNS_SERVER_WEB_SERVICE_WWW_FOLDER_PATH=/opt/technitium/technitium-console` | React 19 + TypeScript + Vite SPA (`byGarcia/technitium-console`). Dedicated Clients page, 2-second live streaming, Command Palette (Ctrl+K), and cluster sync views. |

#### 🔄 Switching Tiers on Fleet Nodes
Create or update `/etc/systemd/system/dns.service.d/console.conf` (keeps GC tuning in `override.conf` isolated):

```bash
# Switch to Tier 2: Sentinel Glassmorphism UI
sudo tee /etc/systemd/system/dns.service.d/console.conf > /dev/null << 'EOF'
[Service]
Environment="DNS_SERVER_WEB_SERVICE_WWW_FOLDER_PATH=/opt/technitium/sentinel-webui"
EOF
sudo systemctl daemon-reload && sudo systemctl restart dns.service

# Switch to Tier 3: Modern React SPA
sudo tee /etc/systemd/system/dns.service.d/console.conf > /dev/null << 'EOF'
[Service]
Environment="DNS_SERVER_WEB_SERVICE_WWW_FOLDER_PATH=/opt/technitium/technitium-console"
EOF
sudo systemctl daemon-reload && sudo systemctl restart dns.service

# Switch back to Tier 1: Pristine Stock Upstream UI
sudo rm -f /etc/systemd/system/dns.service.d/console.conf
sudo systemctl daemon-reload && sudo systemctl restart dns.service
```

## 5. 🔁 Forward-Confirmed Reverse DNS (Auto PTR + Wild IP Architecture)

The cluster utilizes the **Auto PTR (v5.0)** and **Wild IP (v6.1)** DNS applications to provide automated Forward-Confirmed Reverse DNS (FCrDNS) for WireGuard mesh identities (`10.13.13.x`), private subnets (`10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`), and cluster IPs without maintaining manual record tables.

### 🧭 Sub-Zone Topology (Zero Cloudflare Collision)
To prevent collisions with Cloudflare-hosted apex domains, primary zones are isolated strictly to dedicated sub-zones (`ip.<domain>`):
- **Mumbai (`80.225.252.237`):** `ip.agrawal.pro` + `10.in-addr.arpa` + `168.192.in-addr.arpa`
- **Phoenix (`129.146.39.180`):** `ip.vastrikafashion.in` + `10.in-addr.arpa` + `168.192.in-addr.arpa`
- **Oregon (`35.212.133.199`):** `ip.agarwal.us.kg` + `10.in-addr.arpa` + `168.192.in-addr.arpa`

### ☁️ Cloudflare Boundary
- **Internal / Cluster / VPN Resolution:** 100% confined to Technitium. Cloudflare requires **zero** configuration.
- **Optional Public Delegation:** If public internet resolvers ever need to resolve `*.ip.<domain>`, add an `NS` record in Cloudflare:
  - Record: `NS` | Name: `ip` | Nameserver: `hub.<domain>` (e.g. `hub.agrawal.pro`)

### 🧩 Configuration Schemas
1. **Forward Sub-Zone (`ip.<domain>`) APP Record:**
   - **Type:** `APP`
   - **Name:** `ip.<domain>`
   - **App Name:** `Wild IP` (`classPath`: `WildIp.App`)
   - **Record Data:**
     ```json
     {
       "allowedNetworks": [
         "10.0.0.0/8",
         "172.16.0.0/12",
         "192.168.0.0/16",
         "80.225.252.237/32",
         "129.146.39.180/32",
         "35.212.133.199/32"
       ]
     }
     ```
2. **Reverse Zones (`10.in-addr.arpa`, `168.192.in-addr.arpa`) APP Record:**
   - **Type:** `APP`
   - **Name:** `<zone>`
   - **App Name:** `Auto PTR` (`classPath`: `AutoPtr.App`)
   - **Record Data:**
     ```json
     {
       "prefix": "",
       "suffix": ".ip.<domain>",
       "ipSeparator": "-"
     }
     ```

## 6. 🧰 Custom DNS Applications & Ownership

> [!IMPORTANT]
> Production query logging is provided by **Sentinel Client ID**. Do not install Technitium's stock **Query Logs (Sqlite)** app on production nodes. Sentinel Client ID uses SQLite internally; that does not make it the stock app.

### 🛡️ Advanced Blocking

| Component | Canonical source | Purpose |
|---|---|---|
| Engine | [`technitium-advanced-blocking/src/`](../Sentinel-Core/code/services/technitium-advanced-blocking/src/) in Sentinel-Core (moved from `Apps/AdvancedBlockingApp/` on 2026-11-05; only a pointer file remains there) | Custom Technitium app implementing multi-profile blocking, regex and AdBlock rules, CNAME-cloaking checks, and optional query decision logs. Plugin version **12.0.100**, kept above stock `12.0` so the App Store never offers the stock app as an update. `DnsServer.sln` and `Tests/AdvancedBlockingApp` build it from the Sentinel-Core path. |
| Node configurations | [`technitium-advanced-blocking/`](../Sentinel-Core/code/services/technitium-advanced-blocking/) in Sentinel-Core | `mumbai-dnsApp.config`, `phoenix-dnsApp.config`, and `oregon-dnsApp.config` define node-specific profiles, feeds, and endpoint mappings. |
| Companion apps | [`drop-requests.config`](../Sentinel-Core/code/services/technitium-advanced-blocking/drop-requests.config), [`no-data.config`](../Sentinel-Core/code/services/technitium-advanced-blocking/no-data.config) | Drop Requests applies explicit network/malformed-packet policy; NO DATA returns `NOERROR` with no answers for configured cases. |
| Config rollout | [`deploy-app-configs.ps1`](../Sentinel-Core/code/services/technitium-advanced-blocking/deploy-app-configs.ps1) | Deploys the app configurations to all three nodes, restarts the DNS service unless skipped, and verifies A and HTTPS (Type 65) resolution. It does not replace the fork's engine build/release process. |

Sentinel-Core is the single source of truth for both the Advanced Blocking engine and its node configurations. When merging an upstream release, port upstream changes from `Apps/AdvancedBlockingApp/` onto the Sentinel-Core `src/` copy by hand, then bump the Sentinel build number (for example `12.0.101`, or `<new upstream>.100` after an upstream version change). Profile selection uses DoT/DoH endpoint hostnames with label-boundary matching and most-specific-match precedence.

> [!NOTE]
> Decision logging is disabled by default. Set `"enableDecisionLogging": true` at the top level of the Advanced Blocking configuration only when needed: the general-log records include device endpoints, client IPs, queried domains, profiles, rules, and list sources.

> [!WARNING]
> Keep `blockedQuestions` in `drop-requests.config` empty (`[]`). Never black-hole HTTPS (Type 65) or SVCB (Type 64): clients issue these modern service-binding queries alongside address lookups, and dropping them can cause multi-second client timeouts. Use the NO DATA app only when an explicit `NOERROR`/zero-answer response is intended.

### 🏷️ Sentinel Client ID

| Concern | Production behavior |
|---|---|
| Canonical source | [`sentinel-client-id/`](../Sentinel-Core/code/services/sentinel-client-id/) in Sentinel-Core; this fork consumes it as a Technitium app. |
| Role | Replaces the stock Query Logs (Sqlite) app and provides device-aware query logging through Technitium's query-log interfaces. |
| App directory | `/etc/dns/apps/Sentinel Client ID/` on Mumbai, Phoenix, and Oregon. |
| Database | `sqliteDbPath` defaults to `querylogs.db`, relative to the app's `ApplicationFolder` (normally `/etc/dns/apps/Sentinel Client ID/querylogs.db`). |
| Device identification | DoT TLS SNI, DoH request host, EDNS0 client tag (option 65001), then configured client-IP mappings. |
| Mapping configuration | `tagMappings` maps device tags to display names; `clientIpMappings` maps known addresses to display names in `dnsApp.config`. |

Build the app from the Sentinel-Core service directory with:

```powershell
dotnet publish D:\Sentinel-Labs\Sentinel-Core\code\services\sentinel-client-id\SentinelClientIdApp.csproj -c Release --no-self-contained -o D:\Sentinel-Labs\Sentinel-Core\temp\client-id-release\publish -p:BaseIntermediateOutputPath=D:\Sentinel-Labs\Sentinel-Core\temp\client-id-release\obj\ -p:MSBuildProjectExtensionsPath=D:\Sentinel-Labs\Sentinel-Core\temp\client-id-release\obj\ -p:BaseOutputPath=D:\Sentinel-Labs\Sentinel-Core\temp\client-id-release\bin\
```

Verify that the package contains the published app output and native SQLite dependencies for Linux ARM64 and x64. Install under `/etc/dns/apps/Sentinel Client ID/`, preserving the existing database and `dnsApp.config`. The service directory's [`coordinated_release.py`](../Sentinel-Core/Code/services/sentinel-client-id/coordinated_release.py) provides hash validation, stopped-service SQLite backups using the configured database path, bounded DNS/plugin readiness, and binary-only rollback. `install.sh` delegates to this transaction and rejects the unsafe app-only ZIP interface. Configuration-only backups use [`technitium-config-backup`](../Sentinel-Core/Code/services/technitium-config-backup/) and exclude the configured Client ID database and sidecars rather than making an inconsistent hot copy.

**Upgrade compatibility checks:** the current app creates `dns_logs` when absent, adds the `device_name` column to older databases, and maintains indexes during initialization. It does not declare a separate schema-version table. Resolve the configured `sqliteDbPath`, make an integrity-checked SQLite backup that includes WAL content, retain the current app files, and verify app startup plus Query Logs results afterward. Validate old/new schema compatibility before relying on binary-only rollback. Never automatically restore an older database over newer query history; any database recovery requires preserving the newest database/WAL and an explicit recovery decision.

> [!WARNING]
> DoT does **not** pass through Caddy. Android Private DNS connects directly to Technitium on port 853; Sentinel Client ID extracts the device name from the TLS SNI metadata. DoH is a separate path and uses the request host.

## 7. 💡 Approved Customization Ideas (Proposal Only)

The following ideas are approved for exploration, not implementation or production rollout:

| # | Idea | Source / behavior to define before implementation |
|---|---|---|
| E1 | Per-device dashboard tile for query count, blocked count, and last seen | Sentinel Client ID query-log data; define refresh interval and retention semantics. |
| E2 | Telegram alert when a device's DoT traffic stops or its NXDOMAIN rate spikes | Sentinel Client ID metrics; define per-device baselines, quiet periods, and alert thresholds. |
| E3 | One-click “Allow for this device/profile” from Query Logs | Advanced Blocking profile/rule model; require a bounded rule scope, confirmation, and audit trail. |
| E4 | Per-device profile schedules (for example, a family profile at night) | Device-to-profile mapping; define timezone, schedule precedence, and safe fallback behavior. |

> [!NOTE]
> E5, a vendor-essential allowlist for phone brands, was rejected and is intentionally excluded.
