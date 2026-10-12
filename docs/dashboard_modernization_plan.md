# 🏛️ Master Implementation Specification: Complete Dashboard Modernization & Telemetry Suite

> **Target Project:** `Sentinel-Core` (`Technitium DNS Server UI`)  
> **Staging Path:** `DD:\Sentinel-Labs\Technitium-DnsServer\DnsServerCore\www\`  
> **Target Nodes:** Mumbai (`80.225.252.237`), Phoenix (`129.146.39.180`)  
> **Rule Compliance:** Anti-Silent-Omission Guardrail (Rule 12), Zero Root Pollution (Rule 8), Multi-Node Concurrency (Rule 7)

---

## 📋 Comprehensive Feature Matrix (100% Retained & Expanded)

### 01. Classic Header Action Alignment
- **Requirement:** Align Classic layout header actions with Modern layout.
- **Solution:** Right-aligned flexbox container (`#header .header-actions-flex`). Exact DOM and visual order:
  `[Change Layout Button] ➔ [Theme Changer Dropdown] ➔ [User Menu]`.

---

### 02. Hero Card Footer Pill Spacing & Visual Rhythm
- **Requirement:** Eliminate cramped text, abutting colons, and lack of margins between icons and numbers.
- **Solution:**
  - Ample spacing with explicit non-breaking spaces:
    `ServFail: <b>118</b> &nbsp;•&nbsp; Refused: <b>1</b> &nbsp;•&nbsp; Dropped: <b>53</b>`
    `Recursive: <b>1,296</b> (9.6%) &nbsp;•&nbsp; Auth: <b>6</b>`
    `~0.0 QPS &nbsp;(live) &nbsp;&bull;&nbsp; 12 Clients`
  - All icons formatted with `margin-right: 6px;`.
  - Pill padding expanded to `3.5px 10px; border-radius: 8px;` with `gap: 8px; flex-wrap: wrap;`.

---

### 03. Custom Date & Time Range Floating Toolbar & Theme Parity
- **Requirement:** Modernize the custom date selector and eliminate theme color mismatch in the calendar popup.
- **Solution:**
  - **Floating Glassmorphic Toolbar:** Dual inputs with calendar icon badges (`<i class="fa fa-calendar-o"></i> From`, `<i class="fa fa-calendar-check-o"></i> To`), rounded glass inputs (`.modern-date-input`), and gradient apply button.
  - **Smart Prefill & Presets:** 1-click presets (`[Today]`, `[Yesterday]`, `[Last 3 Days]`, `[Last 14 Days]`) and automatic 24-hour default prefill on click (never blank).
  - **Theme Accent-Color:** Enforce `accent-color` and `color-scheme` across themes (`#3b82f6` Dark Blue, `#f59e0b` Amber, `#6366f1` Dark Grey, `#2563eb` Light) so Chromium's native datepicker popup matches the active theme.

---

### 04. Eliminating Spaghetti Lines: The Stacked Area Telemetry Engine
- **Requirement:** Completely replace legacy unstacked spaghetti lines with an enterprise-grade telemetry visualization.
- **Solution:** Upgrade default chart to a **Stacked Area Telemetry Stream (`stacked: true`)**:
  - **Tier 1 (Base):** `Cached` (Emerald gradient `#10b981`)
  - **Tier 2:** `Recursive` (Electric Blue gradient `#3b82f6`)
  - **Tier 3:** `Authoritative` (Violet gradient `#8b5cf6`)
  - **Tier 4:** `Blocked` (Vivid Amber gradient `#f59e0b`)
  - **Tier 5 (Top):** `Errors / Dropped` (Crimson gradient `#ef4444`)
  - **Visual Result:** Zero criss-crossing lines. The cumulative top curve equals Total Queries. Both total volume and resolution composition are visible at a glance.

---

### 05. Chart Subview Toolbar: Adding Record Types & Protocol Distribution
- **Requirement:** Add ability to filter and visualize by "Query Record Types" and "Protocol Distribution" in the chart toolbar.
- **Solution:** Add two dedicated subviews to the chart toolbar:
  - `[ 📋 Record Types ]`: Visualizes all DNS query record types (`A`, `AAAA`, `HTTPS`, `TXT`, `PTR`, `SRV`, `CNAME`, `MX`...) sorted descending with curated colors, counts, and percentages.
  - `[ 🔀 Protocols ]`: Visualizes protocol distribution (`UDP`, `TCP`, encrypted `TLS/DoT`, encrypted `HTTPS/DoH`, `QUIC/DoQ`) with security badges, counts, and percentages.
  - **Full Toolbar:**  
    `[ 📈 Traffic Activity ]` `[ ⚡ Cache & Resolution ]` `[ 📋 Record Types ]` `[ 🔀 Protocols ]` `[ 👥 Top Clients ]` `[ 🛡️ Tracker Blocks ]` `[ ❤️ Health & Anomalies ]`.

---

### 06. Universal Descending Sort Pipeline Across ALL Cards
- **Requirement:** Ensure all matrix cards sort strictly from greater to less.
- **Solution:** Universal descending sort pipeline executed dynamically before DOM rendering:
  - **Card 1 (Response Codes):** `No Error ➔ NX Domain ➔ Server Failure ➔ Refused` (descending by hits).
  - **Card 2 (Resolution Type):** `Cached ➔ Recursive ➔ Dropped ➔ Authoritative` (descending by hits).
  - **Card 3 (Zone & Cache Telemetry):**
    1. `Cached Records: 2,916` (100% relative bar)
    2. `Blocked Zones: 176` (6.0% relative bar)
    3. `Allowed Zones: 34` (1.2% relative bar)
    4. `Authoritative Zones: 0` (0% relative bar)
    (Dynamic proportional scaling against maximum metric; eliminates the 100% bug).

---

### 07. Full Multi-Card Bidirectional Cross-Filtering (Power BI Standard)
- **Requirement:** Clicking one metric filters and cross-highlights all other cards.
- **Solution:**
  - Clicking **`Cached`**: Isolates Cached on main chart with emerald gradient area; cross-highlights `No Error` in Response Codes; cross-highlights `Cached Records` in Zone Telemetry; dims other rows (`opacity: 0.4`).
  - Clicking **`Blocked`**: Isolates Blocked on main chart with amber area; cross-highlights `Blocked Zones` in Zone Telemetry; cross-highlights `NX Domain` / `Refused`.
  - Clicking a **`Record Type`** (`A`, `AAAA`, `HTTPS`) or **`Protocol`** (`UDP`, `TLS`, `HTTPS`): Filters the chart and highlights that metric across all cards.
  - **Global Filter Chip:** `[ 🔍 Active Filter: <Metric> • <Count> (<Percent>) ✕ ]` displayed above chart. Clicking `✕` resets all cards instantly.

---

## 2. Target Files & Delivery
- [`index.html`](file:///D:/Sentinel-Labs/Sentinel-Core/temp/pr-staging/DnsServer/DnsServerCore/www/index.html)
- [`main.css`](file:///D:/Sentinel-Labs/Sentinel-Core/temp/pr-staging/DnsServer/DnsServerCore/www/css/main.css)
- [`dark-mode.css`](file:///D:/Sentinel-Labs/Sentinel-Core/temp/pr-staging/DnsServer/DnsServerCore/www/css/dark-mode.css) (and theme siblings)
- [`js/main.js`](file:///D:/Sentinel-Labs/Sentinel-Core/temp/pr-staging/DnsServer/DnsServerCore/www/js/main.js)
- Multi-node deployment via 64-bit PowerShell RunspacePool (`temp/deploy_fleet_dns_update.ps1`) to Mumbai and Phoenix.
