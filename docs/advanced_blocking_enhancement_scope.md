# 🛡️ Technitium AdvancedBlockingApp — Scope of Enhancement Analysis

> **Target Project:** `D:\Sentinel-Labs\Technitium-DnsServer\Apps\AdvancedBlockingApp\`  
> **Reference Baseline:** `temp\Adguard\AdGuardHome-master` (`urlfilter` & `internal/filtering`)  
> **Author:** Sentinel-Labs Senior Implementation Engineer  
> **Date:** September 2026  

---

## 📌 1. Executive Summary & Context

The previous pull request introduced foundational AdBlock-style filtering into Technitium DNS Server's `AdvancedBlockingApp` (version 12.0), incorporating:
- Basic `||domain^` and `@@||domain^` syntax.
- `/regex/` pattern rules and `*.wildcard` domain expansion.
- `$important`, basic single-list `$badfilter`, `$denyallow=`, `$dnstype=`, and basic `$dnsrewrite=` (for bare IP addresses and RCODEs `NOERROR`/`NXDOMAIN`/`REFUSED`).

However, comparing this implementation directly against **AdGuard Home's rule engine (`github.com/AdguardTeam/urlfilter` and `internal/filtering`)** and Technitium DNS Server's native extensibility interfaces (`IDnsServer`, `IDnsPostProcessor`) reveals **significant architectural and functional scope for enhancement**.

---

## 📊 2. Feature Comparison Matrix

| Capability / Feature | AdGuard Home Engine | Current Technitium `AdvancedBlockingApp` | Enhancement Feasibility |
| :--- | :--- | :--- | :--- |
| **`||domain^` / `@@` syntax** | ✅ Full support | ✅ Implemented | Baseline |
| **Wildcards & Suffixes** | ✅ Fast trie / label matching | ⚠️ Regex compilation per wildcard | 🚀 High (Optimize matching) |
| **Negated DNS Types (`$dnstype=~TYPE`)** | ✅ Fully supported | ❌ **Bug/Inversion:** Strips `~` and treats as positive match | 🚀 Urgent Bugfix & Enhancement |
| **CNAME Rewrite (`$dnsrewrite=domain`)** | ✅ Rewrites CNAME & resolves chain | ❌ Explicitly rejected as out-of-scope | 🚀 High (Via `IDnsServer.DirectQueryAsync`) |
| **TXT / MX / PTR / HTTPS Rewrites** | ✅ Supported | ❌ Unrecognized | 🚀 Medium |
| **Per-Rule Client Filters (`$client=`)** | ✅ Supported (IP, CIDR, ClientID) | ❌ Parsed and silently ignored | 🚀 High (`remoteEP` already available) |
| **Cross-List `$badfilter`** | ✅ Global cancellation across all lists | ⚠️ Local to same list file only | 🚀 High |
| **Direct Inline AdBlock Rules** | ✅ User custom rules textarea | ❌ Only supports remote/file URL lists | 🚀 High (`adblockRules: []` in config) |
| **CNAME Cloaking Defense** | ✅ Post-resolve response filtering | ❌ None (Only pre-resolve request filter) | 🚀 High (Via `IDnsPostProcessor`) |
| **IP-Level Blocklists in DNS** | ✅ Matches resolved A/AAAA against CIDRs | ❌ None | 🚀 Medium |
| **Blocked Services Toggles** | ✅ 50+ services (TikTok, YouTube, etc.) | ❌ Manual rule configuration only | 🚀 Medium-High |
| **Rule Engine Performance at Scale** | ✅ Label-trie / Hash-prefix indexing | ⚠️ Sequential $O(N)$ regex scanning loop | 🚀 High (Performance at 100k+ rules) |

---

## 🔍 3. Detailed Enhancement Opportunities

### 🚀 Enhancement Vector 1: Semantic Correction & Syntax Polish
1. **Fix Negated `$dnstype=~TYPE` Handling:**
   - *Current Code:* In `App.cs` line 1962, the code does:
     ```csharp
     string t = typeToken.Trim().TrimStart('~');
     if ((t.Length > 0) && Enum.TryParse(t, true, out DnsResourceRecordType parsedType))
         types.Add(parsedType);
     ```
     This strips the tilde `~` and adds the type to the positive match set! Thus, a rule like `||example.com^$dnstype=~HTTPS` (meaning: block everything *except* HTTPS queries) is inverted to *only* block HTTPS queries.
   - *Enhancement:* Track an `_excludedDnsTypes` set. When present, evaluate `!_excludedDnsTypes.Contains(qType)`.

2. **Support Exact Domain Match Anchors (`|domain|`):**
   - In AdGuard/AdBlock rules, `|example.com|` or `|example.com^` matches strictly `example.com` and **not** `sub.example.com`.
   - Current implementation discards all single-pipe rules as URL anchors:
     ```csharp
     if (rule.StartsWith('|')) { wasUnsupported = true; return null; }
     ```
   - *Enhancement:* Distinguish single-pipe domain rules (`|example.com|` or `|example.com^`) from URL path rules (`|https://...`). Match them using exact equality instead of subdomain suffix matching.

3. **Support `$empty` Modifier:**
   - AdGuard supports `$empty`, which returns an empty `NOERROR` answer (NODATA). This is equivalent to `$dnsrewrite=NOERROR` but is standard syntax in many popular lists (e.g. OISD, Hagezi).

---

### 🚀 Enhancement Vector 2: Full `$dnsrewrite` Protocol Support (CNAME, TXT, PTR)
1. **CNAME Aliasing / Redirection:**
   - *Current Status:* `App.cs` intentionally omits CNAME rewrites because it notes that resolver re-invocation is required.
   - *Enhancement:* Technitium's `IDnsServer` interface natively provides:
     ```csharp
     Task<DnsDatagram> DirectQueryAsync(DnsQuestionRecord question, int timeout = 4000, CancellationToken cancellationToken = default);
     ```
     When a rule like `||google.com^$dnsrewrite=forcesafesearch.google.com` matches:
     1. Construct a CNAME record: `google.com. CNAME forcesafesearch.google.com.`
     2. Invoke `_dnsServer.DirectQueryAsync()` for `forcesafesearch.google.com` (for the requested `A` or `AAAA` record).
     3. Append the resolved address records into the `Answer` section alongside the CNAME record.
     4. This unlocks parental controls, SafeSearch enforcement, and custom domain re-routing directly from AdBlock filter lists!

2. **Custom Resource Records (TXT, MX, PTR, SERVFAIL):**
   - Support `$dnsrewrite=TXT;text_payload` for verification token mocking or custom notices.
   - Support `$dnsrewrite=SERVFAIL` alongside existing `NXDOMAIN`/`REFUSED`/`NOERROR`.

---

### 🚀 Enhancement Vector 3: Per-Rule Client Scoping (`$client=`, `~$client=`)
1. **Client IP & Subnet Filtering:**
   - Modern adblock lists often include client-specific bypasses or rules, e.g.:
     - `||tiktok.com^$client=192.168.1.50` (block TikTok only for child's tablet).
     - `||ads.example.com^$~client=192.168.1.10` (block for all clients except developer workstation).
   - In Technitium's `ProcessRequestAsync(DnsDatagram request, IPEndPoint remoteEP)`, `remoteEP.Address` is always available.
   - *Enhancement:* Parse `$client=` into IP addresses or CIDR ranges. In `AdBlockRule.IsMatch()`, verify client IP membership before triggering the rule.

---

### 🚀 Enhancement Vector 4: CNAME Cloaking Defense via `IDnsPostProcessor`
1. **The CNAME Cloaking Problem:**
   - Advertisers and trackers bypass DNS blocklists by having the site owner create a first-party CNAME record (e.g., `track.example.com CNAME tracker.thirdparty-adtech.com`).
   - Standard DNS blocklists only see `track.example.com` during the request phase and cannot block it unless the specific third-party canonical target is evaluated.
2. **Technitium Implementation:**
   - Technitium provides the `IDnsPostProcessor` interface:
     ```csharp
     public interface IDnsPostProcessor
     {
         Task<DnsDatagram?> PostProcessAsync(DnsDatagram request, DnsDatagram response, IPEndPoint remoteEP);
     }
     ```
   - *Enhancement:* `AdvancedBlockingApp` can implement `IDnsPostProcessor`. After Technitium resolves a domain, the app inspects any `CNAME` records in `response.Answer`. If the CNAME target matches any blocklist domain, the response is replaced with a blocked response (NXDOMAIN / blocking IP), neutralizing CNAME cloaking just like AdGuard Home and Pi-hole.

---

### 🚀 Enhancement Vector 5: In-Config Custom Rules & Global `$badfilter`
1. **Direct `adblockRules` in `dnsApp.config`:**
   - Currently, users must host an external HTTP/file URL to use AdBlock syntax. Plain `blocked: []` only supports static domain names.
   - *Enhancement:* Add `"adblockRules": []` directly inside each group in `dnsApp.config`. Users can paste AdGuard syntax rules directly into Technitium's Web GUI app config editor.

2. **Global / Group-Wide `$badfilter` Precedence:**
   - Currently, `$badfilter` only cancels rules inside the *same downloaded list*. If a user wants to unblock a broken rule from an upstream list using `$badfilter`, it is ignored.
   - *Enhancement:* Store a group-wide `_globalBadFilterKeys` set so that any `$badfilter` rule in custom rules or an override list can cancel rules across all lists in that group.

---

### 🚀 Enhancement Vector 6: High-Scale Matching Engine Optimization
1. **Current Bottleneck:**
   - Simple domains use `HashSet<string>` (fast $O(1)$).
   - But all wildcards (`*.example.com`, `||ads*.com^`) and regexes are stored in a flat `List<AdBlockRule>`.
   - On every DNS query, `IsZoneBlocked()` sequentially executes `rule.IsMatch(domain)` across the entire list!
   - If a user subscribes to large lists with 10,000+ wildcard rules, every DNS request incurs 10,000 regex checks.
2. **Optimization Strategy (AdGuard Home approach):**
   - Partition rules by TLD or root domain suffix.
   - Fast label-based pre-filtering: before evaluating a regex, check if the query domain contains the fixed substring or root domain of the pattern.
   - For wildcard rules with a fixed suffix (e.g. `||*banner*.doubleclick.net^`), index under `doubleclick.net` so it is only tested when `doubleclick.net` is queried.

---

### 🚀 Enhancement Vector 7: Pre-packaged Blocked Services (Application Toggles)
- AdGuard Home features a "Blocked Services" catalog (`internal/filtering/servicelist.go`) with rules for services like WhatsApp, Facebook, Discord, Epic Games, YouTube, Netflix, TikTok, etc.
- In Technitium, groups could support a `"blockedServices": ["tiktok", "facebook", "youtube"]` array that automatically loads pre-defined rule sets, making parental controls and distraction blocking a 1-click configuration.

---

## 🎯 4. Recommendation & Upstream Feasibility

For submitting enhancements back to upstream Technitium (`Technitium-DnsServer`):

1. **Phase 1 (Immediate / High Value for Upstream PR):**
   - **Fix Negated `$dnstype=~TYPE`**: Critical correctness fix for existing parser.
   - **Add `adblockRules: []` to config**: High user convenience (testing rules directly in UI without external files).
   - **Support `$empty` & exact domain anchors `|domain|`**.
   - **Group-wide `$badfilter`**: Allows custom user rules to cancel broken upstream rules.

2. **Phase 2 (Architectural Upgrades):**
   - **CNAME `$dnsrewrite` using `IDnsServer.DirectQueryAsync`**: Enables SafeSearch and dynamic rewrites.
   - **Per-rule `$client=` scoping**: Unlocks device-specific adblock exceptions.

3. **Phase 3 (Enterprise Capabilities):**
   - **CNAME Cloaking via `IDnsPostProcessor`**: Positions Technitium as a tier-1 ad-blocking resolver on par with AdGuard Home and NextDNS.
