# Sentinel-Labs Technitium DNS Server: Customization & Deployment Guide

> **Purpose:** Canonical source of truth for all customizations, modified files, build steps, and fleet deployment procedures for `Technitium-DnsServer`. Future agents must consult this document directly to avoid rediscovery overhead.

---

## 1. Repository Topology & Upstream Relationship

- **Fork Source:** `https://github.com/TechnitiumSoftware/DnsServer.git`
- **Our Repository (`origin`):** `https://github.com/varunagarwal-pro/Technitium-DnsServer.git`
- **Upstream Remote (`upstream`):** `https://github.com/TechnitiumSoftware/DnsServer.git`
- **Default Branch:** `master`
- **Dependency Repository:** `D:\Sentinel-Labs\TechnitiumLibrary\` (Tracked directly in superproject; pure stock upstream from `https://github.com/TechnitiumSoftware/TechnitiumLibrary.git`).

---

## 2. Inventory of Sentinel-Labs Customizations

The table below catalogs every file modified or added by Sentinel-Labs on top of upstream:

| Component / File Path | Customization Purpose & Mechanism |
|---|---|
| [`DnsServerCore/Dns/DnsServer.cs`](file:///D:/Sentinel-Labs/Technitium-DnsServer/DnsServerCore/Dns/DnsServer.cs) | **CNAME / ANAME Hop Isolation:** In `ProcessCNAMEAsync` and `ProcessANAMEAsync`, instantiates `hopContext = new DnsClient.ResolverContext()` for each hop rather than reusing a shared context. Prevents premature `MaxOutboundRequests` exhaustion during complex CDN chains. |
| [`DnsServerCore/www/css/main.css`](file:///D:/Sentinel-Labs/Technitium-DnsServer/DnsServerCore/www/css/main.css) | **Enterprise Glassmorphism UI:** Complete modern design system (`--glass-radius`, backdrop-filters, custom pill tabs, floating date-range toolbar, donut chart visual analytics, and `.enterprise-hero-grid`). Clickable hero cards use `.hero-clickable` / `.hero-link`. Upstream's classic `.stats-panel` is not rendered (`.stats-panel { display: none; }`). |
| [`DnsServerCore/www/css/dark-grey-mode.css`](file:///D:/Sentinel-Labs/Technitium-DnsServer/DnsServerCore/www/css/dark-grey-mode.css) | **Dark Grey / Charcoal Glass Theme:** Dedicated enterprise dark palette distinct from the default slate blue dark mode. |
| [`DnsServerCore/www/index.html`](file:///D:/Sentinel-Labs/Technitium-DnsServer/DnsServerCore/www/index.html) | **Modern Dual-Layout Dashboard:** Provides collapsible left sidebar, top action bar, Hero Stat Cards that replace upstream's `.stats-panel`, and a theme switch modal (System, Dark Blue, Dark Grey, Light, Amber, and Classic Layout toggle). The hero cards open filtered query logs through upstream's `showQueryLogs(domain, clientIp, node, rcode, responseType)` (Total, Blocked, ServFail, Refused, Cached, Recursive, Authoritative; Dropped is not a valid filter). A hidden `display: none` block keeps the upstream stats IDs that the hero cards do not show (NoError, NxDomain, per-item percentages), so upstream `main.js` updates never hit missing elements. Every element ID must stay unique. |
| [`DnsServerCore/www/js/main.js`](file:///D:/Sentinel-Labs/Technitium-DnsServer/DnsServerCore/www/js/main.js) | **Layout & Theme Switcher Engine:** Functions `setLayout(mode)`, `applyLayout(mode)`, `toggleLayout()` and theme persistence in `localStorage`. **URL hash deep-linking:** portal sections are addressable by URL hash, and browser back/forward navigation works. Stats counters are written by upstream's code to the standard `divDashboardStats*` IDs only. No `*Legacy` IDs exist. |
| [`DnsServerCore/DnsServerCore.csproj`](file:///D:/Sentinel-Labs/Technitium-DnsServer/DnsServerCore/DnsServerCore.csproj) | Preserves `dark-grey-mode.css` content packaging (`CopyToOutputDirectory=PreserveNewest`). |
| [`Apps/AdvancedBlockingApp/`](file:///D:/Sentinel-Labs/Technitium-DnsServer/Apps/AdvancedBlockingApp/) | **Enhanced AdBlock Engine:** Full AdGuard/AdBlock syntax support, CNAME cloaking unmasking, partitioned indexing, and non-blocking async dnsrewrite. |

---

## 3. Upstream Upgrade & Merge Protocol

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
   - Branch: `git checkout -b uprev/<date>` (see `Sentinel-Core/Docs/Repos-Sync.md`; never use `-X ours` / `-X theirs`)
   - Merge: `git merge <tag> --no-ff --no-commit`
   - **Resolution Rules (resolve only the conflict hunks; keep every auto-merged upstream change):**
     - `DnsServer.cs`: Merges cleanly. Retain `hopContext` in CNAME/ANAME methods.
     - `main.css`: Keep our design system. Upstream `.stats-panel` styling changes do not apply because the panel is hidden.
     - `index.html`: Keep `.enterprise-hero-grid` and the hidden stats-ID block. **Do not re-add upstream's `.stats-panel` markup.** Port any new upstream dashboard behaviour (e.g. click-to-filter) onto the hero cards instead. Adopt all other upstream markup changes.
   - **Verify before committing:** no duplicate element IDs, balanced tags, and the same ID set as the previous good commit (`temp/merge_repair/html_check.py`). Also confirm `git diff --stat <old-master> HEAD` touches only the files you intended.

> [!WARNING]
> **Lesson from v15.6.2 (2026-10-27):** The first v15.6.2 merge (`50e876e2`) re-added a hidden Classic `.stats-panel` with unsynchronised `*Legacy` IDs and duplicate IDs. It also deleted the hidden stats-ID block and broke tab navigation. It was redone from `7da1c2f0` on 2026-11-04 (`uprev/2026-11-04-v15.6.2`).
4. **Compile & Verify:**
   ```powershell
   dotnet build "D:\Sentinel-Labs\Technitium-DnsServer\DnsServerApp\DnsServerApp.csproj" -c Release
   ```
   Then check every theme (System, Dark Blue, Dark Grey, Light, Amber) and both layouts in a browser, along with tab navigation and deep links.
5. **Open a PR, do not push to `master`:**
   - Push the `uprev/<date>` branch and open a PR. The `upstream-gate.yml` workflow is PR-only. Merging needs explicit owner approval.

---

## 4. Build, Packaging & Multi-Server Deployment Pipeline

### Target Fleet Topology

| Node Name | Role | Public IP | SSH Port | User | Identity Key |
|---|---|---|---|---|---|
| **Phoenix** | Secondary Node | `129.146.39.180` | `41022` | `vexa` | `D:\Sentinel-Labs\Sentinel-Core\Confidential\Keys\pisg_ed25519` |
| **Oregon** | Secondary Node | `35.212.133.199` | `41022` | `vexa` | `D:\Sentinel-Labs\Sentinel-Core\Confidential\Keys\pisg_ed25519` |
| **Mumbai** | **Primary Node** | `80.225.252.237` | `41022` | `vexa` | `D:\Sentinel-Labs\Sentinel-Core\Confidential\Keys\pisg_ed25519` |

> [!IMPORTANT]
> **Cluster Maintenance Sequencing:** Per Technitium clustering rules, **always upgrade all secondary nodes (Phoenix, Oregon) before upgrading the primary node (Mumbai)**.

### Step 1: Compile & Package Artifact
```powershell
# 1. Publish Release binaries
dotnet publish "D:\Sentinel-Labs\Technitium-DnsServer\DnsServerApp\DnsServerApp.csproj" -c Release -o "D:\Sentinel-Labs\Technitium-DnsServer\temp\publish"

# 2. Package tar.gz (must contain systemd.service at root)
tar -czf "D:\Sentinel-Labs\Technitium-DnsServer\temp\technitium-dns-publish.tar.gz" -C "D:\Sentinel-Labs\Technitium-DnsServer\temp\publish" .
```

### Step 2: Deploy to Nodes (Phoenix -> Oregon -> Mumbai)
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

### Remote Service Details
- **Installation Directory:** `/opt/technitium/dns/`
- **Config Directory:** `/etc/dns` (binary format, modified via web console or REST API)
- **Systemd Unit:** `/etc/systemd/system/dns.service` (runs as unprivileged user `dns-server:dns-server`)
- **Runtime Override:** `/etc/systemd/system/dns.service.d/override.conf`. Server GC `DOTNET_gcServer=1` (Workstation GC caused query stalls). Heap cap `0x80000000` (2 GB) on Mumbai/Phoenix and `0x10000000` (256 MB) on Oregon. See `Sentinel-Core/Docs/Infrastructure.md` §16.3–§16.4.
- **Resolver & DoT settings (live config, not in this repo):** `tcpReceiveTimeout` / `tcpSendTimeout` = 60000 ms and `clientTimeout` = 4000 ms. These fix Android Private DNS "server unavailable" errors. Also `resolverTimeout` = 3000 ms, `resolverRetries` = 2, `resolverConcurrency` = 4, `serveStale` = false. See `Infrastructure.md` §16.3 and §16.8. Re-verify them after every upgrade.

## 5. Forward-Confirmed Reverse DNS (Auto PTR + Wild IP Architecture)

The cluster utilizes the **Auto PTR (v5.0)** and **Wild IP (v6.1)** DNS applications to provide automated Forward-Confirmed Reverse DNS (FCrDNS) for WireGuard mesh identities (`10.13.13.x`), private subnets (`10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`), and cluster IPs without maintaining manual record tables.

### Sub-Zone Topology (Zero Cloudflare Collision)
To prevent collisions with Cloudflare-hosted apex domains, primary zones are isolated strictly to dedicated sub-zones (`ip.<domain>`):
- **Mumbai (`80.225.252.237`):** `ip.agrawal.pro` + `10.in-addr.arpa` + `168.192.in-addr.arpa`
- **Phoenix (`129.146.39.180`):** `ip.vastrikafashion.in` + `10.in-addr.arpa` + `168.192.in-addr.arpa`
- **Oregon (`35.212.133.199`):** `ip.agarwal.us.kg` + `10.in-addr.arpa` + `168.192.in-addr.arpa`

### Cloudflare Boundary
- **Internal / Cluster / VPN Resolution:** 100% confined to Technitium. Cloudflare requires **zero** configuration.
- **Optional Public Delegation:** If public internet resolvers ever need to resolve `*.ip.<domain>`, add an `NS` record in Cloudflare:
  - Record: `NS` | Name: `ip` | Nameserver: `hub.<domain>` (e.g. `hub.agrawal.pro`)

### Configuration Schemas
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
