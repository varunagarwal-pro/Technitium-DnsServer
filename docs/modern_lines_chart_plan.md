# 🎯 Architectural Plan: Modern Telemetry Lines & Full Record/Protocol Filtering

> **Target Project:** `Sentinel-Core` (`Technitium DNS Server UI`)  
> **Staging Path:** `D:\Sentinel-Labs\Technitium-DnsServer\DnsServerCore\www\`  
> **Target Nodes:** Mumbai (`80.225.252.237`), Phoenix (`129.146.39.180`)  
> **Rule Compliance:** Rule 3 (Universal Approval Gate), Rule 4 (WHAT/WHY Announcements), Rule 8 (Confined to temp/), Rule 11 (Artifact Planning Mandate), Rule 12 (Traceability)

---

## 📌 1. Root Cause Analysis

### Issue 01: Elimination of Lines Instead of Modern Replacement
- **What happened:** In an effort to eliminate the legacy "spaghetti lines", the main chart was set to `stacked: true` with secondary resolution types (`Cached`, `Recursive`, `Authoritative`, `Blocked`, `Dropped`) converted into stacked filled areas, while `Total Queries`, `No Error`, `NX Domain`, `Clients`, and `Server Failure` lines were set to `ds.hidden = true`.
- **The User's True Requirement:** The user did **not** ask to delete the lines or replace them with stacked areas. They wanted the **legacy spaghetti lines replaced with modern, sleek telemetry lines** (smooth cubic curves, curated modern colors, unstacked true Y-axis scale, clean interactive legend, and no visual clutter).

### Issue 02: Blank Graph When Filtering by Record Type or Protocol
- **What happened:** The Technitium backend API (`/api/dashboard/stats/get`) provides per-minute time series for `Total`, `No Error`, `Server Failure`, `NX Domain`, `Refused`, `Authoritative`, `Recursive`, `Cached`, `Blocked`, `Dropped`, and `Clients`. It does not contain pre-computed datasets with the names `"A"`, `"AAAA"`, `"UDP"`, or `"TLS"`.
- When filtering by `Record: A` or `Protocol: UDP`, `applyActiveDashboardFilter()` searched `mainChartData.datasets` for an exact dataset name match. Finding none, it set all datasets to `hidden = true`, rendering a 100% empty graph (0.0 to 1.0 Y-axis with no lines).

---

## 🛠️ 2. Architectural Solution

### A. Modern Unstacked Telemetry Lines (Replacing Spaghetti Lines)
1. **Unstacked True Scale (`stacked: false`):**
   - The Y-axis represents true query counts at every point in time (e.g. 228 Total, 195 Cached, 17 Recursive, 16 Blocked).
2. **Visual Hierarchy & Curated Modern Palette:**
   - **Primary Line — `Total Queries`:** Vibrant Electric Sky Blue (`#38bdf8`), 2.5px line width, subtle translucent gradient area fill (`rgba(56, 189, 248, 0.08)`), tension `0.36`.
   - **Performance Line — `Cached Queries`:** Vibrant Emerald (`#10b981`), 2px stroke, subtle fill (`rgba(16, 185, 129, 0.06)`).
   - **Resolution Line — `Recursive Queries`:** Electric Indigo (`#6366f1`), 2px stroke.
   - **Protection Line — `Blocked Queries`:** Vibrant Amber/Orange (`#f59e0b`), 2px stroke, subtle fill (`rgba(245, 158, 11, 0.06)`).
   - **Health Lines (Toggleable):** `NX Domain` (`#ec4899`), `Server Failure` (`#ef4444`), `Authoritative` (`#8b5cf6`), `Clients` (`#06b6d4`).
3. **Interactive Modern HTML Legend Bar:**
   - Interactive clickable pills directly above the chart:
     - `[ ● Total Queries ]`
     - `[ ● Cached ]`
     - `[ ● Recursive ]`
     - `[ ● Blocked ]`
   - Clicking any pill toggles that line on/off with an active/dimmed pill style.
4. **Enhanced Hover & Tooltip Experience:**
   - `pointRadius: 0` (clean, uncluttered lines).
   - `pointHoverRadius: 5` (crisp accent dots on hover).
   - Modern floating index tooltip displaying absolute counts and percentages for all visible lines.

---

### B. Dynamic Time-Series Projection for Record Types & Protocols
When a user selects a Record Type (e.g. `Record: A` - 10,711 queries, 79.7%) or a Protocol (e.g. `Protocol: UDP` - 12,224 queries, 89.9%):
1. **Time-Series Projection:**
   - Using the selected metric's query ratio (`ratio = filterCount / totalQueries`), project the metric across the time series:
     `filteredData[t] = Math.round(totalQueries[t] * ratio)`
2. **Dual-Curve Visual Representation:**
   - **Spotlight Filtered Line:**
     - Label: e.g. `A Record Queries (79.7%)` or `UDP Queries (89.9%)`
     - Stroke: 2.5px solid vibrant signature color (`#06b6d4` for Record Types, `#3b82f6` for Protocols).
     - Fill: Soft modern gradient area under the curve (`rgba(..., 0.15)`).
   - **Reference Baseline Curve:**
     - Label: `Total Queries (Reference)`
     - Stroke: 1.5px dashed (`[4, 4]`) translucent slate (`rgba(148, 163, 184, 0.45)`).
     - Fill: None.
3. **Direct Filter Resolution for Other Metrics:**
   - If filtering by `Cached`, `Recursive`, `No Error`, `NX Domain`, etc., spotlight that exact time series dataset in full color with the `Total Queries (Reference)` dashed baseline.
4. **Active Legend & Tooltip Integration:**
   - Legend updates to: `[ ● Filtered Metric ]   [ ┄ Total Queries (Reference) ]`
   - Tooltip shows the exact filtered count and percentage alongside the total for that minute.
5. **Revert on Reset:**
   - Selecting `All` or clicking `✕` immediately restores all modern telemetry lines.

---

## 🚀 3. Files to Update
- [`www/js/main.js`](file:///D:/Sentinel-Labs/Sentinel-Core/temp/pr-staging/DnsServer/DnsServerCore/www/js/main.js):
  - Configure modern unstacked line styling and interactive legend click handlers.
  - Implement time-series projection for Record Types and Protocols in `applyActiveDashboardFilter()`.
  - Fix tooltip callbacks and dynamic legend indicator updates.
- [`www/index.html`](file:///D:/Sentinel-Labs/Sentinel-Core/temp/pr-staging/DnsServer/DnsServerCore/www/index.html):
  - Ensure `#lblDashboardChartLegend` supports interactive clicking and dynamic state indicators.
- [`www/css/main.css`](file:///D:/Sentinel-Labs/Sentinel-Core/temp/pr-staging/DnsServer/DnsServerCore/www/css/main.css):
  - Styling for interactive legend pills (`.legend-pill.active`, `.legend-pill.dimmed`).
