# Two major enhancements submitted for Technitium DNS: Modern Dashboard (Dual Layout + Telemetry Filters) & Full AdGuard/CNAME Syntax in Advanced Blocking App (Feedback & Testers Wanted!)

Hey everyone,

Like many of you here, I rely on Technitium DNS Server as the backbone of my home lab and network infrastructure. It’s easily one of the fastest, most reliable, and extensible DNS solutions available. 

Over the past few weeks, while running large-scale blocklists and monitoring daily traffic across my cluster, I worked on two major architectural enhancements aimed at solving two common challenges:
1. **Making community AdGuard-syntax blocklists and CNAME unmasking fully native and performant.**
2. **Giving the web console a high-performance, modern dashboard with dual layout options and interactive telemetry filtering.**

Both feature sets are now open as Pull Requests on upstream Technitium, and I’d love to get feedback, real-world testing, and thoughts from the community and developers.

---

### 1. Advanced Blocking App: Full AdGuard Rule Syntax, CNAME Cloaking Unmasking & Core Whitelist Parity ([PR #2179](https://github.com/TechnitiumSoftware/DnsServer/pull/2179))

`AdvancedBlockingApp` has always been great for basic domain and regex lists, but widely-used community filter lists (AdGuard DNS filter, OISD, Hagezi, AdAway) heavily utilize modifiers, client scoping, and CNAME cloaking unmasking. Previously, many of these lines were silently dropped or mis-parsed.

This PR re-architects the rule parsing and processing pipeline:

* **Full AdGuard/AdBlock DNS-layer Syntax:**
  * **Directives & Modifiers:** Support for `$important` (highest precedence tier), `$badfilter` (cross-list rule cancellation), `$dnstype=` (e.g. `$dnstype=A|AAAA`, `~$dnstype=AAAA`), and `$denyallow=` (subdomain block exceptions).
  * **Client IP / CIDR Scoping:** Target rules to specific devices or subnets (`$client=192.168.1.50`, `$client=10.0.0.0/24`, `~$client=...`).
  * **Rich `$dnsrewrite` Actions:** Synthesizes custom A/AAAA, CNAME, TXT, PTR, MX, `$empty` (NODATA), and DNS codes (`NXDOMAIN`, `SERVFAIL`, `REFUSED`) without worker thread stalls.
  * **Exact Domain Matching:** Distinct matching for `|example.com|` vs subdomain-matching `||example.com^`.
* **Harmonized CNAME Cloaking Unmasking:**
  * Uses `IDnsPostProcessor` to intercept upstream responses and unmask third-party tracking domains hidden behind CNAME aliases (with RFC-compliant Extended DNS Error / EDE codes).
  * Strictly preserves preceding CNAME delegation records so clients receive a valid delegation chain instead of an unexplained bare drop.
* **Core Allowed Zones Inheritance:**
  * Dynamically integrates with Technitium Core's `AllowedZoneManager`. If you whitelist a parent domain like `reddit.com` in Technitium's Allowed Zones, it automatically inherits to subdomains and protects canonical CNAME targets (like CDN endpoints) without forcing you to hunt down and manually whitelist internal CDN aliases.
* **Multi-Threaded Parallel Engine:**
  * Regex compilation leverages `Parallel.ForEach` across all CPU cores with thread-safe collections.
  * Base-domain partitioned suffix indexing ensures sub-millisecond evaluation even with 1M+ rules loaded.

👉 **Review the code & discussion:** [Pull Request #2179](https://github.com/TechnitiumSoftware/DnsServer/pull/2179)

---

### 2. Dashboard Modernization: Dual Layouts, 3 Themes & Interactive Telemetry Filters ([PR #2182](https://github.com/TechnitiumSoftware/DnsServer/pull/2182))

The Technitium dashboard is information-dense, but as networks scale, analyzing query trends and isolating traffic patterns directly from the UI becomes essential. This PR introduces a non-destructive UI modernization that preserves complete backward compatibility:

* **Dual Layout System (Classic vs. Modern):**
  * **Classic Layout:** Retains the familiar Technitium layout you know, but with cleaned-up typography, refined grid alignment, and crisp modern chart rendering.
  * **Modern Layout:** A brand-new card-based layout featuring a dedicated live telemetry row, synchronized timeline charts, and streamlined quick-action panels.
  * You can switch between layouts instantly with a single toggle in the dashboard controls.
* **Modernized Charting (No More Spaghetti Dots):**
  * Replaced the legacy dot-heavy lines with smooth, crisp vector paths.
  * Includes interactive hover crosshairs, synchronized multi-metric tooltips, and distinct color-coded series.
* **Interactive Telemetry Filters:**
  * **Query Type / Record Filter:** Isolate trends by A, AAAA, HTTPS, PTR, SRV, etc., with a single click.
  * **Protocol Filter:** Break down traffic dynamically by standard UDP, TCP, DNS-over-HTTPS (DoH), and DNS-over-TLS (DoT).
* **3 Theme Variants:**
  * **Classic Dark:** Sleek slate-dark palette for everyday monitoring.
  * **Midnight OLED:** True pure-black background (#000000) for OLED panels, control centers, and reduced power consumption.
  * **Modern Light:** Clean, high-contrast daylight theme with subtle elevation and soft borders.

👉 **Review the code & screenshots:** [Pull Request #2182](https://github.com/TechnitiumSoftware/DnsServer/pull/2182)

---

### 🛠️ Want to Test the Consolidated Build?

If you’d like to test both the modernized dashboard and the enhanced Advanced Blocking App running together in your own lab environment, I maintain the combined code in my fork:

🔗 **Consolidated Repository:** [https://github.com/varunagarwal-pro/Technitium-DnsServer.git](https://github.com/varunagarwal-pro/Technitium-DnsServer.git)

You can clone and build it directly using standard .NET SDK tooling:
```bash
git clone https://github.com/varunagarwal-pro/Technitium-DnsServer.git
cd Technitium-DnsServer
dotnet build -c Release
```

### 🤝 How You Can Help

1. **Test your favorite blocklists:** Throw complex AdGuard/AdBlock filter lists at the app and verify rule matching, exemptions (`@@`), and `$important` overrides.
2. **Dashboard usability feedback:** Try out both Classic and Modern layouts, test the record/protocol filters under load, and let me know how the charts feel on your displays (desktop and mobile).
3. **Leave comments/reviews on GitHub:** Constructive reviews on [PR #2179](https://github.com/TechnitiumSoftware/DnsServer/pull/2179) and [PR #2182](https://github.com/TechnitiumSoftware/DnsServer/pull/2182) help upstream evaluate and merge these features faster.

Looking forward to hearing your thoughts and edge-case testing results!
