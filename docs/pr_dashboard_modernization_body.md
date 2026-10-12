## 🚀 Overview & Summary

This pull request delivers a comprehensive modernization of the Technitium DNS Server Web Console dashboard. It introduces a **Dual Layout Architecture**, a **Multi-Theme Engine**, full **Visual Modernization** across both Classic and Modern layouts, a **Live Stats & Real-Time Telemetry Engine**, complete preservation and modernization of all **11 Main Telemetry Lines** with interactive dataset toggle pills, **New Telemetry Header Filters** with dynamic time-series projection, and **Universal Descending Sorting**.

All changes are strictly client-side UI/UX enhancements with **zero modifications to core DNS resolution logic**, **full backward compatibility** with upstream APIs and legacy DOM selectors, and **persistence** of all user preferences in `localStorage`.

---

## 🌟 Key Architectural Pillars

### 1. 🔄 Two Separate Layouts (Dual Layout Architecture)
Administrators can choose between two distinct workflows based on ergonomics, monitor form factor, or personal preference:
- **Classic Top-Nav Layout:**
  - Preserves the familiar upstream navigation bar structure at the top of the screen.
  - Receives a modern visual elevation: rounded card borders, cleaner elevation shadows, refined typography, and harmonized color contrast.
  - Action button layout modernized into a streamlined action bar: `[Change Layout] ➔ [Theme Changer] ➔ [User Menu]`.
- **Modern Collapsible Sidebar Layout:**
  - Features a collapsible left sidebar docked alongside the main viewport.
  - Contains brand icon, server version, live uptime tracker, navigation items with count indicators, and a floating user profile card.
  - Maximizes vertical and horizontal screen real estate for wide desktop monitors and NOC wall displays.
  - Fully responsive with smooth CSS transitions between collapsed (icon-only) and expanded modes.
- **Persistence:** Layout choice is saved to `localStorage.getItem("technitium_layout")` and persists across browser sessions and reloads.

---

### 2. 🎨 Multi-Theme Engine (3 Curated Themes + Light Mode)
A unified theme engine allows instant, single-click cycling across 4 themes without requiring a page reload:
- **Dark Blue (Midnight Navy):**
  - Modern dark sapphire/navy palette with frosted glass acrylic cards (`backdrop-filter: blur(12px)`), subtle borders (`rgba(255, 255, 255, 0.08)`), and balanced neon accent highlights.
- **Dark Grey (Graphite / Charcoal):**
  - High-contrast, neutral dark theme (`#18191c` / `#222429`) engineered for NOCs, server rooms, and extended operational monitoring without visual fatigue or blue light saturation.
- **Amber (Obsidian & Warm Gold):**
  - Warm golden amber accents on obsidian dark backgrounds providing high legibility, terminal-like warmth, and accessibility.
- **Light Mode (Frosted Acrylic):**
  - Clean, modern light theme with elevated white cards, refined box shadows, crisp contrast, and neutral gray accents.
- **Persistence:** Theme selection is saved to `localStorage.getItem("technitium_theme")` and persists seamlessly across reloads.

---

### 3. ⚡ Live Stats & Real-Time Telemetry Engine
Real-time visibility into DNS query throughput and server activity:
- **Live QPS (Queries Per Second) Dynamic Computation:**
  - Real-time throughput velocity calculated as `(deltaQ / deltaS)` computed across auto-refresh ticks.
  - Live visual pill (`<i class="fa fa-bolt"></i> ~X.X QPS (live)`) with amber lightning bolt indicating real-time traffic velocity.
- **Configurable Cadence Selector:**
  - Available in both the Modern Topbar and Classic Header:
    - **Live (3s):** High-frequency streaming mode for real-time monitoring and active troubleshooting.
    - **Normal (10s):** Recommended default mode providing balanced operational telemetry.
    - **Manual (Off):** Disables background polling for low-bandwidth connections or static review.
- **Animated Pulse Beacon (`.live-pulse-beacon`):**
  - Rhythmic pulsing indicator providing instant visual confirmation that live telemetry is actively syncing.
  - Visual pulse rate modulates with cadence (faster pulse on 3s live mode, steady pulse on 10s normal mode, paused/dimmed when off).
  - Subtle flash animation triggers on every HTTP payload synchronization.
- **Smart Power & Bandwidth Throttling (Tab-Aware Polling):**
  - Polling automatically suspends when the browser tab is hidden or minimized (`document.hidden`), or when user navigates away from the Dashboard tab.
  - Instantly resumes polling and requests fresh data when the tab is focused, preventing memory bloat and unnecessary background network/server load.

---

### 4. 💎 Modern Layout Deep Dive
The Modern Layout introduces dedicated operational panels designed for modern system administration:
- **4 Hero KPI Stat Cards:**
  - Large, glanceable metrics for `Total Queries`, `Blocked Queries`, `Active Clients`, and `Resolver Health`.
  - Decorated with subtle status glows, trend badges, and proportional horizontal progress bars.
  - Includes cache hit efficiency, recursion percentage, and error breakdown pills.
- **Interactive Distribution Breakdown Tables:**
  - Top Queried Domains, Top Blocked Domains, and Top Clients rendered as responsive tables with visual percentage distribution bars.
  - Quick-action buttons next to domains for instant lookup or blocklist inspection.
- **System Matrix & Zone Telemetry Cards:**
  - Live authoritative zone health cards with visual indicators (Healthy/Degraded), query throughput counters, and zone record totals.
- **Universal Descending Sort:**
  - All breakdown cards, tables, and telemetry lists automatically sort high-to-low dynamically so highest volume entities are always prioritized at the top.

---

### 5. 📊 Main Telemetry Chart Modernization (Zero Spaghetti, All 11 Metrics Retained)
- **100% Telemetry Dataset Preservation:**
  - Every single one of the 11 upstream telemetry metrics is fully retained:
    1. `Total Queries`
    2. `Cached`
    3. `Recursive`
    4. `Blocked`
    5. `Authoritative`
    6. `No Error`
    7. `NX Domain`
    8. `Server Failure`
    9. `Refused`
    10. `Dropped`
    11. `Clients`
- **Modern Smooth Cubic Splines:**
  - Completely replaced legacy dot-dense, stacked spaghetti lines with smooth, unstacked cubic curves (`lineTension: 0.36`).
  - Zero static point radius (`pointRadius: 0`) ensures clean, readable trend lines even under high query densities.
  - Dynamic hover points (`pointHoverRadius: 5`) and detailed interactive Chart.js tooltips appear upon hovering over any data point.
- **Permanent 2-Tier Chart Header:**
  - Restructured the chart header into two permanent, non-colliding tiers:
    - **Tier 1 (Top Row):** Chart Title (`Total Queries`), Quick Range Pills (`Last Hour`, `Last 24 Hours`, `Last 7 Days`, `Last 30 Days`), Filter Dropdowns, and Refresh action button.
    - **Tier 2 (Sub Row):** Interactive Dataset Legend Pills and Active Filter indicator chip.
  - Eliminates erratic multi-line flex-wrapping jumps when toggling filters or switching screen sizes.
- **Interactive Legend Pills:**
  - Each metric features a styled interactive pill displaying its line color.
  - Clicking any pill toggles the dataset visibility on the canvas and dims/un-dims the pill (`.legend-pill.active` vs. `.legend-pill.dimmed`).
  - Dataset visibility states are stored and remembered across dashboard refreshes.

---

### 6. 🔍 New Telemetry Filters (Query Record Types & Protocol Distribution)
- **Direct Header Filter Controls:**
  - Added `#btnFilterRecordType` (`Record Types`) and `#btnFilterProtocol` (`Protocols`) dropdown filters into the main telemetry header.
- **Real-Time Time-Series Projection:**
  - Selecting any record type (e.g. `A`, `AAAA`, `HTTPS`, `TXT`, `PTR`) or protocol (e.g. `UDP`, `TCP`, `DoH`, `DoT`, `QUIC`) projects that distribution proportionally against `Total Queries (Reference)`.
  - Re-scales the Y-axis smoothly to emphasize the filtered data trend.
- **Active Filter Chip & Reset:**
  - Displays a high-contrast badge (e.g. `Filter: HTTPS ✕` or `Filter: UDP ✕`) in Tier 2 of the chart header.
  - Clicking the chip or selecting "All" instantly clears the filter and restores full 11-line telemetry.
- **Bidirectional Cross-Filtering:**
  - Clicking slices or legends on the bottom breakdown charts automatically activates the corresponding filter on the main telemetry chart.

---

## 🛠️ File Changes Summary

| File | Changes | Description |
|------|---------|-------------|
| `DnsServerCore/DnsServerCore.csproj` | +4 lines | Registered `dark-grey-mode.css` for output directory deployment |
| `DnsServerCore/www/index.html` | Restructured | Added modern sidebar layout, hero stat cards, live pulse controls, 2-tier chart header, and filter dropdowns |
| `DnsServerCore/www/css/main.css` | Enhanced | Added glassmorphism styles, legend pills, filter buttons, live pulse beacon animations, and responsive grid rules |
| `DnsServerCore/www/css/dark-mode.css` | Enhanced | Refined dark blue palette, card elevation, modal contrast, and interactive pills |
| `DnsServerCore/www/css/dark-grey-mode.css` | New File | Created full Graphite / Charcoal dark theme suite |
| `DnsServerCore/www/css/amber-mode.css` | Enhanced | Styled Amber theme support for modern cards, sidebar, live pulse, and chart legend pills |
| `DnsServerCore/www/js/main.js` | Enhanced | 11-line smooth telemetry, live QPS engine, auto-refresh cadence, interactive legend, header filters, time-series projection, descending sort |
| `DnsServerCore/www/js/auth.js` | Updated | Set sidebar username display label on login and token verification |

---

## 🧪 Verification & Testing

- **Compilation:** Clean `dotnet build -c Release` with 0 Warnings and 0 Errors.
- **Browser Compatibility:** Verified in Google Chrome, Mozilla Firefox, Microsoft Edge, and Apple Safari.
- **Dual Production Node Deployment:** Tested live on production instances under real DNS query traffic.
- **Responsiveness:** Validated on wide desktop monitors (4K, 1440p, 1080p), laptops, and mobile/tablet breakpoints.
- **Persistence Verification:** Verified `localStorage` persistence for layout selection, theme selection, auto-refresh cadence, and chart dataset visibility toggles across hard reloads (`Ctrl+F5`).
