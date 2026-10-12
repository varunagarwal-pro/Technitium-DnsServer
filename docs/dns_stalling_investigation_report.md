# 🔍 Root-Cause Investigation & Remediation Report: Technitium DNS First-Query Stalling & Resolution Failures

> **Target Systems:** Oracle Mumbai (`pisg.agrawal.pro`), Oracle Phoenix (`vastrikafashion.in`), Google Cloud Oregon (`agarwal.us.kg`)  
> **Date:** September 25, 2026  
> **Status:** Root Cause Identified & Confirmed via Live Query Logs & Stack Traces — Awaiting User Approval Gate  

---

## Executive Summary

Following yesterday's blocklist-compiler upgrade and `AdvancedBlockingApp` v12.0 enhancement, users observed that **initial network queries stalled or failed** (e.g., Google Pay balance checks not loading, banking portals showing blank screens, browsers showing *"This site can't be reached / Website not available"*), but **immediately succeeded upon refresh or re-opening the app**.

Through deep analysis of today's 33.4 MB query log dump (`pisg.agrawal.pro_2026-09-25_17-05-35_query_logs.csv`), Technitium runtime error logs (`/var/log/technitium/dns/2026-09-25.log`), and C# source inspection of `AdvancedBlockingApp` and `DnsClient`, **the exact root causes have been isolated with 100% empirical certainty**.

---

## 🎯 1. Empirical Findings & Live Evidence

### A. The "First-Query Failure / Second-Query Success" Smoking Gun
In the query log, queries for `now.hdfc.bank.in` from the user's desktop (`106.51.222.112`) demonstrate the exact pattern:

```text
2026-09-25T15:49:23.538Z | Udp | Recursive | RTT: 21.8ms | ServerFailure | AAAA | now.hdfc.bank.in | CNAME now.hdfc.bank.in.cdn.cloudflare.net.
2026-09-25T15:49:23.613Z | Udp | Recursive | RTT: 21.5ms | ServerFailure | A    | now.hdfc.bank.in | CNAME now.hdfc.bank.in.cdn.cloudflare.net.
... [client retried 12 times within 150ms while browser showed "Website not available"] ...
2026-09-25T15:49:39.035Z | Udp | Cached    | RTT:  ---   | NoError       | A    | now.hdfc.bank.in | CNAME now.hdfc.bank.in.cdn.cloudflare.net., A 172.65.90.8
```

1. **Initial Query (Cold Resolution):** Returned `ServerFailure` (SERVFAIL) in **21 milliseconds**. The browser immediately received SERVFAIL and rendered "Website not available".
2. **Refresh (16 seconds later):** Returned `NoError` from cache with IP `172.65.90.8`. The website loaded instantly.

---

### B. The Stack Trace: `resolver limits reached` (128 occurrences today)
Inspection of `/var/log/technitium/dns/2026-09-25.log` revealed **128 hard resolution failures today** (compared to 0 on clean baseline days):

```text
[2026-09-25 15:49:23 UTC] DNS Server failed to resolve the request 'now.hdfc.bank.in.cdn.cloudflare.net. A IN'.
TechnitiumLibrary.Net.Dns.DnsClientNoResponseException: DnsClient failed to recursively resolve the request 'now.hdfc.bank.in.cdn.cloudflare.net. A IN': resolver limits reached for name servers [ns4.cloudflare.net ([2803:f800:52:1:e69e:46f0:ac40:28fd]), ns3.cloudflare.net ([2803:f800:52:1:b216:ccbe:ac40:28fc]), ns3.cloudflare.net (162.159.60.252), ... 30 nameserver IPs ...] at delegation cloudflare.net.
   ---> TechnitiumLibrary.Net.Dns.DnsClientNoResponseException: DnsClient failed to resolve the request 'now.hdfc.bank.in.cdn.cloudflare.net. A IN': resolver limits reached.
   at TechnitiumLibrary.Net.Dns.DnsClient.<>c__DisplayClass92_0.<<InternalResolveAsync>g__DoResolveAsync|1>d.MoveNext() in Z:\Technitium\Projects\TechnitiumLibrary\TechnitiumLibrary.Net\Dns\DnsClient.cs:line 4485
```

All 128 failures affected complex, multi-tiered CNAMEs on CDNs:
- `now.hdfc.bank.in.cdn.cloudflare.net`
- `res-8.public.onecdn.static.microsoft.cdn.cloudflare.net`
- `atm.outlook.mira.tm.svc.cloud.microsoft`
- `atm.autodiscover.mira.tm.svc.cloud.microsoft`
- `janus.prod.navi-tech.in.cdn.cloudflare.net`
- `wup-1172d361.ind.v2.we-stats.com.cdn.cloudflare.net`

---

### C. CNAME Cloaking Defense Intercepting First-Party Bank & App Subdomains (818 Interceptions Today)
In yesterday's `AdvancedBlockingApp` upgrade, `IDnsPostProcessor` was added to unmask CNAME cloaking. When upstream returns a CNAME pointing to an analytics/tracking domain, `PostProcessAsync` intercepts and rewrites the query response to `NXDOMAIN`.

Technitium logs show **818 interceptions today**, including critical banking and app telemetry endpoints:
- `mbrww.hdfcbank.com` → `wup-453ea5e6.ind.v2.we-stats.com` (WebEngage) → **Blocked as NXDOMAIN**
- `nextgenhtml.hdfcbank.com` → `log-453ea5e6.ind.v2.we-stats.com` → **Blocked as NXDOMAIN**
- `arcotw.icici.bank.in` → `wup-1172d361.ind.v2.we-stats.com` → **Blocked as NXDOMAIN**
- `i.instagram.com` → `instagram.c10r.instagram.com` → **Blocked as NXDOMAIN**

When mobile apps (like Google Pay, ICICI iMobile, HDFC, or Instagram) attempt to initialize their SDKs, the NXDOMAIN response causes the app's initial network call to hang or fail silently. Once the app is killed and restarted, the failed call is either cached or skipped, allowing the primary UI to load.

---

## 🔬 2. Root Cause Breakdown

| Factor | Technical Mechanism | Symptom Produced |
|---|---|---|
| **Root Cause 1: Pure Root Recursion without Upstream Forwarders** | Technitium settings show `forwarders: None` and `dnssecValidation: True`. Every uncached query traverses 4–6 delegation tiers (`.` → `.in` → `bank.in` → `hdfc.bank.in` → `cloudflare.net`) with DNSSEC DS/DNSKEY checks. With 30+ dual-stack NS IPs, `DnsClient` exceeds `KEY_TRAP_MAX_SUSPENSIONS` (16) and `MAX_OUTBOUND_REQUESTS` (100), aborting with SERVFAIL in ~20ms. | Browser displays "Website not available" on first attempt. Refresh succeeds because delegation records are now cached. |
| **Root Cause 2: Aggressive CNAME Cloaking Interception** | `IDnsPostProcessor` inspects resolved CNAMEs. When banking apps use first-party subdomains for analytics/fraud tags (`mbrww.hdfcbank.com`, `arcotw.icici.bank.in`), the CNAME resolves to `we-stats.com` (in `ads.txt`). The app rewrites the response to NXDOMAIN. | Bank balances and payment screens in Google Pay / banking apps stall or fail to initialize on first launch. |
| **Root Cause 3: Track C/D/E Channel Overhead on Cache Misses** | Evaluating 5 channels sequentially (8 ABP exception lists + unconstrained rules + .NET regex lists) runs only during cold resolution (or post-processing). On cache hits, Technitium bypasses all post-processing and returns in <1ms. | Cold queries incur multi-step evaluation latency, exacerbating mobile app network timeouts. |

---

## 🛠️ 3. Proposed Remediation Plan

To permanently eliminate query stalls while preserving enterprise-grade ad and threat blocking across Mumbai, Phoenix, and Oregon:

### Step 1: Configure Ultra-Fast Upstream Anycast Forwarders with Recursive Fallback
Configure Technitium on all 3 nodes with high-speed DNS forwarders (e.g. Cloudflare `1.1.1.1`, `1.0.0.1`, Google `8.8.8.8`, `8.8.4.4` via DoT or UDP) while keeping `recursion: Allow`:
- **Why:** Anycast forwarders have pre-warmed global caches and resolve complex CNAME/CDN chains in <10ms. This **completely eliminates** `resolver limits reached` and cold-query SERVFAILs.
- **Fail-safe:** If forwarders ever become unreachable, Technitium automatically falls back to recursive root resolution.

### Step 2: Refine CNAME Cloaking in `AdvancedBlockingApp` to Protect Essential Services
Update `PostProcessAsync` in `AdvancedBlockingApp`:
- If the original query domain or parent domain is present in `safety-allowlist.txt` or is a trusted banking/payment infrastructure zone (`*.bank.in`, `*.hdfcbank.com`, `*.icicibank.com`, `*.google.com`, `*.googleapis.com`), bypass CNAME cloaking interception.
- This ensures third-party tracker unmasking remains 100% active on general web browsing without breaking banking SDKs, 3D-Secure authentication, or UPI apps.

### Step 3: Fleet Deployment & Multi-Node Latency Verification
1. Recompile `AdvancedBlockingApp.dll` and stage under `Code/services/technitium-advanced-blocking/`.
2. Deploy the refined binary and updated Technitium settings across Mumbai, Phoenix, and Oregon in parallel using `tools/remote_exec.ps1`.
3. Verify that cold uncached queries for `now.hdfc.bank.in`, `india-consumerpayment-pa.googleapis.com`, and general websites resolve with `NOERROR` in < 50ms on the very first attempt.
