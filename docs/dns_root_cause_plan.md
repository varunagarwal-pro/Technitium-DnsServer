# 🛡️ Architectural Root-Cause Analysis & Native Remediation Plan
## Technitium DNS First-Query Stalling & CNAME Cloaking Resolution Failures

> **📌 Target Subsystem:** Technitium DNS Server (`/opt/technitium/dns`), `AdvancedBlockingApp` v12.0, `blocklist-compiler`  
> **🎯 Target Fleet Nodes:** Oracle Mumbai (`pisg.agrawal.pro`), Oracle Phoenix (`vastrikafashion.in`), Google Cloud Oregon (`agarwal.us.kg`)  
> **📅 Date:** September 25, 2026  
> **👤 Maintainer:** Varun Agarwal (Sentinel Labs)  
> **🏛️ Architecture Mandate:** Pure Recursive Authoritative PISG Gateway with **ZERO LEAK** (Strictly NO upstream forwarders like 1.1.1.1 or 8.8.8.8)  

---

## 🎯 1. Problem Statement & User Symptoms

Following the deployment of the enhanced `blocklist-compiler` and `AdvancedBlockingApp` v12.0:
1. **Google Pay & Banking Stalls:** Google Pay fails to display bank balances or stalls on balance checks. Quitting and reopening the app temporarily bypasses the initial screen, but selecting a bank or balance inquiry stalls again.
2. **First-Query Stalling / "Website Not Available":** Web browsers on Windows and Android show *"Website not available"* or *"This site can't be reached"* on the first attempt, but upon pressing **Refresh** (or waiting 10-15 seconds), the website loads instantly.

---

## 🔬 2. Empirical Investigation & Live Evidence

Investigation across the live Mumbai node (`80.225.252.237`), the 33.4 MB query log dump (`pisg.agrawal.pro_2026-09-25_17-05-35_query_logs.csv`), and historical rotated server logs (`/var/log/technitium/dns/*.log.1`) revealed the exact mechanisms:

### A. Rotated Log Baseline Comparison
Historical log analysis proved that prior to September 24, resolution errors were near zero:

| Date | `resolver limits reached` | `failed to resolve request` | `intercepted CNAME cloaking` | Notes |
| :--- | :---: | :---: | :---: | :--- |
| **2026-09-04 to 2026-09-19** | **0** | **4 – 25** | **0** | Clean baseline. |
| **2026-09-20 to 2026-09-23** | 31 – 481 | 33 – 537 | **0** | Technitium v13 upgrade with KeyTrap limits. |
| **2026-09-24** | 342 | 494 | **556** | `AdvancedBlockingApp` v12.0 deployed. |
| **2026-09-25 (Today)** | 120 | 128 | **820** | Full symptom manifestation. |

### B. Smoking Gun 1: CNAME Cloaking Overkill on Banking Endpoints (820 Interceptions Today)
In `pisg.agrawal.pro` query logs:
```text
2026-09-25T15:50:19.751Z | Recursive | RTT: --- | NxDomain | arcotw.icici.bank.in
2026-09-25T16:38:32.576Z | Recursive | RTT: --- | NxDomain | arcotw.icici.bank.in
2026-09-25T17:02:44.865Z | Recursive | RTT: --- | NxDomain | mbrww.hdfcbank.com
2026-09-25T17:02:47.316Z | Recursive | RTT: --- | NxDomain | nextgenhtml.hdfcbank.com
```
- `arcotw.icici.bank.in` is the Broadcom Arcot / RiskFort 3D-Secure authentication and balance verification gateway for ICICI Bank and Google Pay UPI.
- `mbrww.hdfcbank.com` and `nextgenhtml.hdfcbank.com` are HDFC Bank's mobile SDK and NetBanking webview endpoints.
- Both banks alias these subdomains to WebEngage analytics (`*.ind.v2.we-stats.com`).
- Because `we-stats.com` is in `ads.txt`, `AdvancedBlockingApp`'s `PostProcessAsync` unmasked the CNAME and **rewrote the entire DNS response to `NXDOMAIN`**.
- When Google Pay or banking apps receive `NXDOMAIN` for their authentication/SDK endpoints, the app hangs waiting for a timeout, stalling bank balance retrieval.

### C. Smoking Gun 2: Cold-Resolution KeyTrap Limit Throttling (`resolver limits reached`)
In `/var/log/technitium/dns/2026-09-25.log`:
```text
[2026-09-25 15:49:23 UTC] DNS Server failed to resolve the request 'now.hdfc.bank.in.cdn.cloudflare.net. A IN'.
TechnitiumLibrary.Net.Dns.DnsClientNoResponseException: DnsClient failed to recursively resolve the request 'now.hdfc.bank.in.cdn.cloudflare.net. A IN': resolver limits reached for name servers [...] at delegation cloudflare.net.
   ---> TechnitiumLibrary.Net.Dns.DnsClientNoResponseException: DnsClient failed to resolve the request 'now.hdfc.bank.in.cdn.cloudflare.net. A IN': resolver limits reached.
   at TechnitiumLibrary.Net.Dns.DnsClient.<>c__DisplayClass92_0.<<InternalResolveAsync>g__DoResolveAsync|1>d.MoveNext() in Z:\Technitium\Projects\TechnitiumLibrary\TechnitiumLibrary.Net\Dns\DnsClient.cs:line 4485
```
- In the query log at `15:49:23.538Z`, the cold recursive query for `now.hdfc.bank.in` failed in **21.8 ms** with `ServerFailure` (SERVFAIL). The browser displayed *"Website not available"*.
- At `15:49:39.035Z` (16 seconds later, upon browser refresh), `now.hdfc.bank.in` returned `NoError` from cache in **< 1 ms**.
- At `17:02:43.481Z` (hours later, warm cache), recursive resolution succeeded in **59.3 ms** with `NoError`.
- **Root Mechanism:** Technitium's `DnsClient` enforces CVE-2023-50387 (KeyTrap) safeguards: `KEY_TRAP_MAX_SUSPENSIONS = 16`, `MAX_OUTBOUND_REQUESTS = 100`, `RE_TRAP_MAX_HASH_OPERATIONS = 32`.
- When an uncached query traverses 5-6 delegation tiers (`.` -> `.in` -> `bank.in` -> `hdfc.bank.in` -> `cloudflare.net`) with DNSSEC validation across 30+ nameserver IPs, a completely cold cache exhausts the 16 suspension / 100 request limits in a single `ResolverContext`. Technitium aborts with SERVFAIL.
- **Why refresh works:** The failed initial attempt successfully cached the parent delegation records and NS glue. When the user hits refresh, Technitium resumes from the cached delegation tier and resolves the query in 59ms without exceeding any limits.
- **Why the cache was cold:** Technitium's cache size on Mumbai was set to only `cacheMaximumEntries: 10000`. On a server processing ~100,000 queries/day, 10,000 entries causes constant churn, evicting vital NS glue and forcing frequent cold recursions.

---

## 🛠️ 3. Native Architectural Remediation (Zero-Leak, No Forwarders)

### Component 1: `AdvancedBlockingApp` CNAME Cloaking Fix (`App.cs`)
1. **Protected Zone Bypass in `PostProcessAsync`:**
   - In `PostProcessAsync`, before unmasking a CNAME, inspect the query domain.
   - If the query domain belongs to financial/banking infrastructure (`*.bank.in`, `*.hdfcbank.com`, `*.icicibank.com`, `*.axisbank.com`, `*.sbi.co.in`, `*.npci.org.in`, `*.juspay.in`, `*.paytm.com`), or is listed in `_allowListZones`, **completely bypass CNAME cloaking inspection**.
   - Legitimate first-party services must NEVER have their primary DNS responses rewritten to `NXDOMAIN` due to an aliased analytics vendor.
2. **Add Per-Group CNAME Cloaking Configuration:**
   - Add `"enableCnameCloaking": true/false` to `Group` in `dnsApp.config`. Allows explicit administrative control per group rather than an unconfigurable global interceptor.
3. **Restrict CNAME Cloaking Target Lists:**
   - Evaluate CNAME targets strictly against ad/tracking blocklists, not against broad domain lists or social lists (preventing collateral damage like `reddit.com` aliasing to Fastly CDN).

### Component 2: Infrastructure & Financial Safety Allowlist Expansion (`compile.py`)
1. Update `CORE_INFRA_ALLOWLIST` in `Code/services/blocklist-compiler/compile.py`:
   - Add authoritative national financial and payment infrastructure:
     - `bank.in` (all Indian banks under RBI/IDRBT root)
     - `hdfcbank.com`, `icicibank.com`, `axisbank.com`, `sbi.co.in`, `kotak.com`, `indusind.com`, `rbi.org.in`, `npci.org.in`
     - `juspay.in`, `paytm.com`, `paytmpayments.com`, `billdesk.com`, `razorpay.com`, `cashfree.com`
   - Add major CDN edge infrastructure hostnames to `SHARED_CDN_ALLOWLIST` where needed.
2. Re-run `compile.py` to regenerate `safety-allowlist.txt` and publish atomically to `/srv/blocklists/current/`.

### Component 3: Technitium DNS Server Native Cache & Resolver Hardening
Tune Technitium DNS Server native parameters via REST API across Mumbai, Phoenix, and Oregon:
1. **Increase `cacheMaximumEntries`:**
   - From `10,000` to `200,000`.
   - On Mumbai/Phoenix (8 GB RAM), 200k entries consumes only ~40 MB of RAM, but prevents premature eviction of authoritative nameserver glue and delegation records.
2. **Increase `resolverTimeout`:**
   - From `1500` ms to `3000` ms. Prevents premature SERVFAIL on international recursive delegations.
3. **Increase `resolverConcurrency`:**
   - From `2` to `3`. Allows parallel nameserver queries during delegation discovery.
4. **Preserve Zero-Leak Architecture:**
   - `forwarders: None` (Strictly maintained).
   - `recursion: UseSpecifiedNetworkACL` (Strictly maintained).
   - `dnssecValidation: True` (Strictly maintained).

---

## 📋 4. Step-by-Step Implementation Protocol

| Step | Action | Target Node / Path | Verification |
| :---: | :--- | :--- | :--- |
| **1** | Patch `PostProcessAsync` in `App.cs` with Protected Zone Bypass & config toggle | `temp/pr-staging/DnsServer/Apps/AdvancedBlockingApp/App.cs` | Code diff review |
| **2** | Compile `AdvancedBlockingApp.dll` Release binary | Local .NET SDK build | 0 warnings, 0 errors |
| **3** | Update `CORE_INFRA_ALLOWLIST` in `compile.py` with banking/payment domains | `Code/services/blocklist-compiler/compile.py` | AST syntax check (`python -m py_compile`) |
| **4** | Run `compile.py` to regenerate `safety-allowlist.txt` | Local & remote staging | Verify `safety-allowlist.txt` entries |
| **5** | Deploy patched `AdvancedBlockingApp.dll` and updated configs to Mumbai, Phoenix, Oregon | Multi-node parallel via `tools/remote_exec.ps1` | DLL timestamp & md5 parity |
| **6** | Apply Technitium native cache & resolver tuning (200k cache, 3000ms timeout) | All 3 nodes via REST API | API `/api/settings/get` verification |
| **7** | End-to-end DNS cold-query verification | Mumbai, Phoenix, Oregon | `dig @80.225.252.237 arcotw.icici.bank.in` -> NOERROR<br>`dig @80.225.252.237 now.hdfc.bank.in` -> NOERROR in < 60ms |

---

## 🛡️ 5. Traceability & Zero-Downgrade Guarantee

- **Zero Third-Party Leaks:** No DNS queries are forwarded to Cloudflare, Google, or any external resolver. Every query remains 100% recursive and authoritative from ICANN root with DNSSEC validation.
- **Enhanced Adblocking Preserved:** All 5 blocklist channels (plain domains, ABP exception rules, .NET regexes, safety allowlists, and C2 NXDOMAIN feeds) remain 100% active.
- **Anti-Scope-Creep:** Confined strictly to fixing the CNAME cloaking false-positive bug and recursive cache tuning.
