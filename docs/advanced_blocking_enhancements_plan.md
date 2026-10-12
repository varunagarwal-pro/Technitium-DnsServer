# 📋 Implementation Plan: Enterprise-Grade AdGuard Rule Enhancements for AdvancedBlockingApp

> **Target Repository:** `D:\Sentinel-Labs\Technitium-DnsServer` & `temp\DnsServer-master`  
> **Target Project:** `Apps\AdvancedBlockingApp\AdvancedBlockingApp.csproj`  
> **Status:** Pending User Approval Gate (Rule 3)  
> **Author:** Sentinel-Labs Senior Implementation Engineer  
> **Date:** September 2026  

---

## 🎯 1. Architectural Scope & Objectives

To deliver an enterprise-grade, 100% bug-free enhancement to Technitium's `AdvancedBlockingApp`, the following 9 enhancement vectors will be implemented:

1. **Semantic Bugfix — Inverted Negated `$dnstype=~TYPE`:**
   - Correct parser to differentiate between positive types (`A`, `AAAA`) and excluded types (`~HTTPS`, `~TXT`).
   - Store `HashSet<DnsResourceRecordType>? _excludedDnsTypes`.
   - Update `IsMatch()` to check `!_excludedDnsTypes.Contains(qType)`.

2. **Exact Domain Match Anchors (`|example.com|` and `|example.com^`):**
   - Distinguish domain exact-anchors from URL path anchors.
   - Enforce exact equality matching (`domain.Equals(_domain)`) rather than subdomain suffix matching (`EndsWith("." + _domain)`).

3. **Standard `$empty` Modifier:**
   - Map `$empty` to a NOERROR NODATA response (zero answers), matching AdGuard Home and Pi-hole behavior.

4. **Full `$dnsrewrite` Protocol Support (CNAME, TXT, PTR, MX, SERVFAIL):**
   - **CNAME Rewriting:** When `$dnsrewrite=target.domain.com` matches:
     - Generate a CNAME resource record for the query name.
     - Asynchronously query the canonical target using `_dnsServer.DirectQueryAsync()` to fetch A/AAAA records.
     - Append both the CNAME and resolved address records in the answer section.
   - **TXT / PTR / MX / SERVFAIL:** Support typed rewrites (`$dnsrewrite=TXT;payload`, `$dnsrewrite=PTR;host`, `$dnsrewrite=SERVFAIL`).

5. **Per-Rule Client Scoping (`$client=`, `~$client=`):**
   - Parse `$client=IP|CIDR` and `~$client=IP|CIDR` on rules.
   - Pass client IP from `IPEndPoint remoteEP` into rule matching.
   - Allow per-client exceptions and blocking rules within the same blocklist.

6. **In-Config Direct AdBlock Rules (`adblockRules` in `dnsApp.config`):**
   - Add `"adblockRules": []` array to Group configuration schema.
   - Parse and load rules directly from JSON config, enabling users to enter custom AdGuard rules via the Technitium web console without an external HTTP/file URL.

7. **Group-Wide Global `$badfilter` Cancellation:**
   - Collect `$badfilter` rules across all lists and in-config rules in a group into a unified cancellation registry.
   - Allow user rules and override lists to cancel rules across any list in the group.

8. **CNAME Cloaking Defense via `IDnsPostProcessor`:**
   - Implement `IDnsPostProcessor` interface on `AdvancedBlocking.App`.
   - Inspect resolved responses: if upstream returns a CNAME pointing to a blocked domain, intercept and block the response (NXDOMAIN / blocking IP), neutralizing first-party CNAME cloaking.

9. **High-Scale Performance Partitioning (Suffix Trie / Partition Index):**
   - Index wildcard and suffix rules by their discernible base domain / TLD.
   - Eliminate $O(N)$ sequential regex evaluation on every DNS query.

---

## 🛠️ 2. Detailed Technical Design & Code Modifications

### 2.1 `AdBlockRule` Data Structure Updates
```csharp
class AdBlockRule
{
    readonly bool _isAllow;
    readonly bool _isRegex;
    readonly bool _hasWildcard;
    readonly bool _exactMatch; // New: exact domain match (|domain|)
    readonly string? _domain;
    readonly Regex? _regex;
    readonly bool _important;
    readonly bool _badFilter;
    readonly HashSet<DnsResourceRecordType>? _dnsTypes;
    readonly HashSet<DnsResourceRecordType>? _excludedDnsTypes; // New: ~TYPE support
    readonly HashSet<string>? _denyAllowDomains;
    readonly DnsRewriteAction? _dnsRewrite;
    readonly List<IPAddress>? _clientIps; // New: $client=
    readonly List<NetworkAddress>? _clientNetworks; // New: $client= CIDRs
    readonly List<IPAddress>? _excludedClientIps; // New: ~$client=
    readonly List<NetworkAddress>? _excludedClientNetworks; // New: ~$client= CIDRs

    public bool IsMatch(string domain, DnsResourceRecordType qType, IPAddress? clientIp);
}
```

### 2.2 `DnsRewriteAction` Enhancements
```csharp
class DnsRewriteAction
{
    bool _recognized;
    DnsResponseCode? _responseCode;
    List<IPAddress>? _addresses;
    string? _cnameTarget; // New: CNAME rewrite
    string? _txtData;     // New: TXT rewrite
    string? _ptrTarget;   // New: PTR rewrite
    DnsResourceRecordType? _customType;

    public static DnsRewriteAction Parse(string rawValue);
}
```

### 2.3 `IDnsPostProcessor` Implementation for CNAME Cloaking
```csharp
public sealed class App : IDnsApplication, IDnsRequestBlockingHandler, IDnsPostProcessor
{
    public Task<DnsDatagram?> PostProcessAsync(DnsDatagram request, DnsDatagram response, IPEndPoint remoteEP)
    {
        // 1. Verify blocking enabled for group
        // 2. Iterate response.Answer for CNAME records
        // 3. For each CNAME target, check group.IsZoneBlocked(cnameTarget)
        // 4. If blocked, return rewritten blocked response (NXDOMAIN / Blocked IP) with EDE option
    }
}
```

---

## 🧪 3. Verification & Validation Protocol

1. **Compilation Validation:**
   - Execute `dotnet build` on `AdvancedBlockingApp.csproj` in both `pr-staging` and `DnsServer-master` with zero warnings and zero errors.

2. **Automated Unit & Smoke Testing:**
   - Expand `SmokeTest/Program.cs` to test:
     - Negated `$dnstype=~HTTPS` matching on `A` and `AAAA` but passing on `HTTPS`.
     - Exact domain `|example.com|` matching `example.com` but NOT `sub.example.com`.
     - `$empty` producing empty NOERROR response.
     - `$dnsrewrite=forcesafesearch.google.com` CNAME resolution.
     - `$dnsrewrite=TXT;verification_key`.
     - `$client=192.168.1.50` matching client IP and rejecting other clients.
     - `~$client=192.168.1.10` rejecting excluded client.
     - Group-wide `$badfilter` across separate list sources.
     - Post-processor CNAME cloaking interception.

3. **Documentation Updates:**
   - Update `README.md` with complete documentation for all new modifiers, syntax rules, CNAME cloaking, and JSON config schema.

4. **Clean Git State:**
   - Keep working tree clean in `pr-staging/DnsServer` ready for commit/PR upon user approval.

---

## 🚦 4. Approval Gate

In accordance with **Rule 3 (Universal Approval Gate)**, execution will halt here.
Awaiting explicit user confirmation (`GO` / `Proceed` / `Approved`) before modifying source files.
