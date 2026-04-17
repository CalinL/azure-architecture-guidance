# Microsoft Entra ID — Group Membership Limits, Kerberos Token Size, and Login Impact

**Audience:** IT Identity, Security, and Platform Engineering teams
**Scope:** Entra ID (cloud), hybrid identity (Entra Connect / Entra Connect Sync / Entra Cloud Sync), on‑premises Active Directory Domain Services (AD DS), Entra Domain Services, Entra Kerberos (cloud Kerberos trust / FIDO2 / Azure Files).
**Last reviewed:** April 2026
**Status:** Draft — AI-authored technical reference. **Requires human subject-matter expert review before any customer or production distribution.**

> 🤖 **Document provenance & quality assurance**
> This document was **authored and quality-reviewed by AI agents** operating against the official Microsoft Learn / Microsoft Support knowledge base. Three independent review passes were performed in parallel against the first-party documentation (full model attribution and validation log in **§12**). All substantive claims are cited against Microsoft sources — see **§10 References**.
>
> **This is an AI-generated draft.** It has **not** been reviewed by a human Microsoft subject-matter expert. Before sharing with customers, escalating to support, or using it to drive production changes, a qualified identity/AD specialist should validate the content. Configuration changes must always be piloted in a non-production environment.

---

## 1. Executive summary

| Question | Short answer |
|---|---|
| Do the classic "too many groups" login failures still exist? | **Yes**, in on‑prem / hybrid scenarios that rely on **Windows Kerberos or NTLM**. |
| Does pure cloud Entra ID (OAuth2 / OIDC / SAML) have the same problem? | **No** — but there are different limits (notably the **200‑group overage claim** in JWT/SAML tokens). |
| Is there a hard ceiling that will block user logon? | **Yes** — the **LSA access token limit of ~1,010 security groups per user**. This cannot be tuned. |
| Is there a tunable limit? | **Yes** — the **Kerberos `MaxTokenSize`**, default **48,000 bytes** (Windows Server 2012+), max 65,535. |
| What membership count is safe? | Aim for **< 500 groups per user** in hybrid environments; **< 1,000 absolute maximum**. |

---

## 2. Two distinct limits you must plan for

There are **two independent failure modes** frequently confused with one another. A robust design must account for both.

### 2.1 The LSA access token limit (~1,010 group SIDs) — HARD LIMIT

When a user logs on to any Windows host (interactive, service, or network logon), the **Local Security Authority (LSA)** builds an **access token** representing the user's security context. That token contains the **SIDs of every security group the user belongs to**, evaluated transitively, plus any SIDs stored in `SIDHistory`.

- The SID array in the access token is capped at **1,024 SIDs**.
- Windows inserts approximately **14 well‑known SIDs** automatically (e.g., `Everyone`, `Authenticated Users`, logon type SIDs).
- The **practical maximum is therefore ~1,010 custom group SIDs** per user.
- **Exceeding the limit causes logon to fail.** The LSA cannot drop SIDs — it fails closed.
- This limit applies **regardless of the authentication protocol** (Kerberos, NTLM, certificate, etc.).
- **This limit is not configurable** and has not changed across Windows versions.

**Authoritative reference:** [Logging on a user account that is a member of more than 1,010 groups may fail on a Windows Server‑based computer (KB 328889)](https://learn.microsoft.com/troubleshoot/windows-server/windows-security/logging-on-user-account-fails)

### 2.2 The Kerberos ticket buffer (`MaxTokenSize`) — TUNABLE

Kerberos encodes the user's group membership SIDs inside the **Privilege Attribute Certificate (PAC)** of the Ticket Granting Ticket (TGT) and service tickets. The client and server allocate a buffer (`MaxTokenSize`) to hold this.

| Windows version | Default `MaxTokenSize` |
|---|---|
| Windows Server 2008 R2 / Windows 7 and earlier | **12,000 bytes** |
| Windows Server 2012 / Windows 8 and later (current default) | **48,000 bytes** |
| Microsoft **recommended maximum** | **48,000 bytes** |
| **Absolute maximum** supported | **65,535 bytes** |

Registry location: `HKLM\SYSTEM\CurrentControlSet\Control\Lsa\Kerberos\Parameters\MaxTokenSize` (DWORD, decimal).

**Authoritative references:**
- [Problems with Kerberos authentication when a user belongs to many groups (KB 327825)](https://learn.microsoft.com/troubleshoot/windows-server/windows-security/kerberos-authentication-problems-if-user-belongs-to-groups)
- [Active Directory Domain Services maximum limits and scalability — Recommended maximum Kerberos ticket size](https://learn.microsoft.com/windows-server/identity/ad-ds/plan/active-directory-domain-services-maximum-limits#recommended-maximum-kerberos-ticket-size)

#### 2.2.1 Estimating Kerberos ticket size

Use this formula (Windows Server 2012+):

```
TokenSize = 1200 + 40d + 8s
```

Where:
- **1200** — overhead (Kerberos ticket header, user/domain name, signatures).
- **d** = memberships in universal groups **outside** the user's account domain + `SIDHistory` entries.
- **s** = memberships in universal groups **inside** the user's domain + **domain‑local** groups + **global** groups.

**Multiply the entire `TokenSize` result by 2 if unconstrained delegation is used** (i.e. `2 × (1200 + 40d + 8s)`). Unconstrained delegation **across incoming forest trusts** has been disabled by default since 2019 — see [KB 4490425](https://support.microsoft.com/help/4490425). Note: administrators can still explicitly enable unconstrained delegation on individual accounts; the 2019 update specifically changed the default behavior for TGT delegation across *incoming* trusts.

**Rule of thumb:** The default 48,000‑byte buffer comfortably holds a user in **~120+ universal groups**, or several hundred domain‑local/global groups. With **KDC Resource SID Compression** (Server 2012+), you can go considerably higher.

#### 2.2.2 Why you must NOT raise `MaxTokenSize` above 48,000 bytes

- Kerberos tickets ride inside HTTP `Authorization` headers, **Base64‑encoded** → +33% size overhead.
- **HTTP.sys / IIS enforces a hard architectural maximum of 64 KB for any single HTTP request header** (the configurable ceiling for `MaxFieldLength`). The *default* registry value is much smaller — **16,384 bytes** — and can be raised up to 65,534 bytes. 48,000 bytes × 1.33 ≈ 64 KB, which is why `MaxTokenSize` should not exceed 48,000.
- Setting `MaxTokenSize` > 48,000 typically produces **HTTP 400 "Bad Request — Request Header Too Long"** in IIS, SharePoint, Exchange, WinRM, and browser‑based SSO.
- Values **> 65,535** break IPsec IKE and some legacy management tools.

**Authoritative reference:** [HTTP 400 Bad Request (Request Header too long) (KB 2020943)](https://learn.microsoft.com/troubleshoot/developer/webapps/iis/www-authentication-authorization/http-bad-request-response-kerberos)

---

## 3. Current state in 2026 — what has improved?

Since the original issues in the Windows Server 2003/2008 era, several platform changes have materially reduced token bloat — **but the underlying limits still apply**.

| Improvement | Effect |
|---|---|
| Default `MaxTokenSize` raised from 12,000 → **48,000** bytes (Server 2012+) | 4× more headroom out of the box. |
| **KDC Resource SID Compression** (Server 2012+) | Compresses resource‑domain SIDs in the PAC; reduces ticket size noticeably. |
| **Unconstrained delegation disabled by default** (2019 security update, KB 4490425) | Eliminates the "double the ticket size" penalty. |
| **Dynamic Access Control (DAC) / claims** | Replaces group‑based ACLs with user/device claims; reduces group sprawl. |
| **Entra ID (cloud) apps use OAuth2 / OIDC / SAML**, not Kerberos | No PAC, no `MaxTokenSize` — different constraints apply (see §4). |

**The LSA 1,010‑SID ceiling has not changed and is not expected to change.**

---

## 4. Pure cloud Entra ID — different model, different limits

Entra ID issues **JWT (OAuth2 / OIDC)** or **SAML** tokens. These are not Kerberos tickets and are not bound by `MaxTokenSize` or the LSA SID array.

However, **group claims inside Entra ID tokens have their own limits**:

| Token type | Max groups emitted directly in token | Behavior when exceeded |
|---|---|---|
| **JWT** (OIDC / OAuth2 access/ID token) | **200 groups** | `_claim_names` / `_claim_sources` **overage claim** — app must call Microsoft Graph (`/v1.0/users/{id}/getMemberObjects` or `/me/transitiveMemberOf`) to retrieve full membership. **Do not** use `/me/memberOf` (direct memberships only — incomplete for overage). **Do not** call the legacy `graph.windows.net` URL that may appear in `_claim_sources.endpoint` — construct a `graph.microsoft.com` URL instead. |
| **SAML** token | **150 groups** | Same overage‑claim mechanism; app must query Graph. |
| **Implicit flow** (legacy SPAs / hybrid-flow ID tokens returned via URL) | **6 groups** (some docs say 5) | Token includes a `"hasgroups": true` claim instead of `groups`; app must query Graph. Implicit flow is deprecated — use Authorization Code + PKCE. |

Configure which groups flow via:
- Application manifest `groupMembershipClaims` (`All`, `SecurityGroup`, `DirectoryRole`, `ApplicationGroup`), **or**
- Enterprise Application → **Single sign‑on → User Attributes & Claims → Groups assigned to the application** (emit only assigned groups, typically the best practice).

**Authoritative references:**
- [Configure group claims for applications with Microsoft Entra ID](https://learn.microsoft.com/entra/identity/hybrid/connect/how-to-connect-fed-group-claims)
- [Microsoft Entra service limits and restrictions](https://learn.microsoft.com/entra/identity/users-groups-roles/directory-service-limits-restrictions)

### Key differences vs. Kerberos
- Entra ID **sign-in itself is not blocked** by any group-count ceiling — users with thousands of groups can still authenticate to Entra ID.
- However, **application authorization can still fail** if the app expects the `groups` claim in the token and does not implement the overage fallback. This is an app-development concern, not an identity-platform failure.
- **App logic** must handle group overage gracefully (Graph call fallback).
- Per‑app **restrict to "Groups assigned to the application"** to avoid overage entirely — this is the recommended configuration for most enterprise scenarios.
- **Prefer App Roles over groups** where feasible — roles are more secure, decouple app config from group membership, and do not have overage limits.

---

## 5. Hybrid & mixed scenarios — where customers get hit most often

These scenarios **re‑introduce the Kerberos / LSA limits** even if your directory is "cloud‑first":

| Scenario | Uses Kerberos/NTLM? | Limits apply? |
|---|---|---|
| User signs in to an on‑prem domain‑joined Windows PC | ✅ Yes | ✅ 1,010 + `MaxTokenSize` |
| Entra‑joined PC accessing on‑prem file share or app via Kerberos | ✅ Yes (via Entra Kerberos / cloud Kerberos trust / resource‑based KCD) | ✅ Both limits |
| **Azure Files** with on‑prem AD or Entra Kerberos authentication | ✅ Yes | ✅ Both limits |
| **Entra Domain Services** (managed AD) | ✅ Yes | ✅ Both limits |
| Windows Hello for Business **Cloud Kerberos Trust** | ✅ Yes (to on‑prem resources) | ✅ Both limits |
| FIDO2 sign‑in to Entra + on‑prem Kerberos SSO | ✅ Yes | ✅ Both limits |
| IIS/SharePoint/Exchange with Windows Integrated Authentication | ✅ Yes | ✅ Both limits |
| Pure cloud SaaS via OIDC/SAML (M365, Salesforce, etc.) | ❌ No | ❌ — but 200‑group JWT overage applies |

If you synchronize Entra ID security groups back into AD DS (or if users are members of large numbers of on‑prem groups), you are exposed to the Kerberos limits.

### 5.1 Cross-domain / cross-forest / trust nuance

The effective access token a user receives is **resource-dependent**. A user can authenticate successfully in one domain or against one server and **fail in another**, even within the same forest. Factors that change the SID set materialized on the target host:

- **Domain-local groups** of the resource domain are added only at the resource DC / server.
- **Server-local groups** (groups from the local SAM of the target machine) are added only at that host.
- **SID filtering / SID quarantine** on forest or external trusts may strip SIDs traversing the trust boundary.
- **`SIDHistory`** contributes to the token in the trusting domain only if SID filtering does not remove it.

**Implication:** Always measure `whoami /groups` and `tokensz.exe` **on the actual target host where the failure occurs**, not on a generic domain-joined workstation.

### 5.2 Hybrid provisioning — Entra Connect Sync vs. Entra Cloud Sync (current state)

- **Entra Cloud Sync** is Microsoft's preferred path going forward for most greenfield hybrid deployments, including cloud-to-AD security group writeback scenarios.
- **Entra Connect Sync** (aka Azure AD Connect) remains in support but **Group Writeback v2** (cloud security groups → AD DS) is deprecated as of mid-2024; plan migrations accordingly. Group Writeback v1 for Microsoft 365 Groups is still supported.
- Either topology can *inflate* on-prem group memberships if cloud groups are synced back without governance — audit the set of groups that actually land in AD DS and evaluate their contribution to the Kerberos token.

---

## 6. Symptoms — how to recognize the issue

| Symptom | Likely cause |
|---|---|
| User logon fails with **"Not enough storage is available to complete this operation"** | `MaxTokenSize` exceeded |
| Security log **Event ID 4625** with status **`0xC000015A` (`STATUS_TOO_MANY_CONTEXT_IDS`)** | LSA 1,010‑SID access-token limit |
| **HTTP 400 – Bad Request (Request header too long)** on IIS/SharePoint/Exchange/WinRM | Kerberos ticket too large for HTTP header buffer. Check `%windir%\System32\LogFiles\HTTPERR` for reason codes `FieldLength` / `RequestLength`. |
| Group Policy fails to apply; `gpresult` shows empty or partial results | `MaxTokenSize` or SID limit |
| Works from some machines, fails from others | Downlevel / legacy OS with smaller default `MaxTokenSize` (12 KB), or non-uniform `MaxTokenSize` deployment across estate |
| Sporadic Kerberos errors after mergers / migrations | Large `SIDHistory` inflating token |
| User succeeds on one server/domain but fails on another | Effective token is **resource-dependent**: domain-local groups in the resource domain, SID filtering across trusts, and server-local groups all change the SID set that is materialized on the target host |
| JWT missing `groups` claim for heavy users | Entra ID 200‑group overage (not a failure — by design; app must handle via Microsoft Graph) |

---

## 7. How to assess your exposure

### 7.1 Count transitive group memberships for a user (PowerShell)

**On‑prem AD — direct memberships only (quick inventory):**
```powershell
# Requires RSAT / ActiveDirectory module. Counts DIRECT memberships only.
$user = 'jdoe'
Get-ADPrincipalGroupMembership -Identity $user | Measure-Object | Select-Object -ExpandProperty Count
```

> ⚠️ **Caveats:** Directory-side queries do **not** accurately reflect the *effective* access token a user will receive on a given host. Transitive / nested groups, `SIDHistory`, domain-local groups in the **resource** domain, server-local groups, SID filtering across trusts, and well-known SIDs inserted at logon all change the final SID set. The only reliable measurements come from a live logon on the target host (see below).

**Entra ID (Microsoft Graph PowerShell):**
```powershell
Connect-MgGraph -Scopes 'User.Read.All','GroupMember.Read.All'
$u = Get-MgUser -UserId 'jdoe@contoso.com'
$all = Get-MgUserTransitiveMemberOf -UserId $u.Id -All
"Total transitive groups/roles: $($all.Count)"
```

### 7.2 Measure the actual Kerberos ticket size

On a client logged on as the target user:

```cmd
whoami /groups             :: count SIDs actually present in the access token
klist                      :: view Kerberos tickets
klist purge                :: clear tickets so next request forces a fresh one
tokensz.exe /compute_tokensize    :: from Windows Server 2003 Resource Kit Tools
```

`tokensz.exe` outputs the calculated and measured token sizes and compares them against `MaxTokenSize`. It is distributed with the **Windows Server 2003 Resource Kit Tools** (still usable on current Windows versions) — Microsoft Download Center ID 17657.

### 7.3 Inventory users approaching the limits (AD)

```powershell
Get-ADUser -Filter * -Properties memberOf |
    Select-Object SamAccountName, @{n='Groups';e={$_.memberOf.Count}} |
    Where-Object Groups -gt 400 |
    Sort-Object Groups -Descending
```

(Note: `memberOf` is direct only. For accurate counts use transitive enumeration.)

---

## 8. Recommendations — avoiding login failures

### 8.1 Governance (do this first)
1. **Target < 500 security groups per user** (transitive) as a soft cap; **< 1,000 as a hard cap**.
2. **Regularly audit** top‑N users by group count; alert when > 500.
3. **Flatten nested group hierarchies** where practical — nesting multiplies transitive SIDs.
4. **Clean up `SIDHistory`** after migrations once trust is established.
5. **Distinguish "distribution/collaboration" from "security" groups** — Microsoft 365 Groups and distribution lists don't consume SID space in the access token, but security‑enabled groups do.
6. **Use Entra ID dynamic groups and entitlement management** to right‑size membership automatically.

### 8.2 Configuration
7. **Verify effective `MaxTokenSize` everywhere.** Default is 48,000 on Windows 8 / Server 2012+ and 12,000 on earlier versions. Don't assume estate-wide consistency — audit the registry value and deployed GPO on clients, web/app servers, and downstream services.
8. **Deploy `MaxTokenSize` via Group Policy** (*Computer Configuration → Policies → Administrative Templates → System → Kerberos → "Set maximum Kerberos SSPI context token buffer size"*) rather than ad-hoc registry edits. Changes require a reboot/service restart on every machine in the authentication path — **partial rollout produces inconsistent, hard-to-diagnose failures**.
9. **Avoid raising `MaxTokenSize` above 48,000 bytes** unless you have a measured, validated, non-HTTP scenario and fully understand the Base64 / IIS header consequences.
10. **PAC / Resource SID Compression** reduces ticket growth automatically on Windows Server 2012+ when the domain functional level supports it — there is no separate "enable" toggle; it is a consequence of running modern OS / domain versions.
11. **Keep unconstrained delegation disabled** (default since 2019 for incoming trusts); prefer **resource‑based constrained delegation (RBCD)**.
12. **Emit only app‑assigned groups** in Entra ID token configuration (avoid the 200‑group overage in JWTs and 150‑group overage in SAML).
13. **Implement overage claim handling** in custom apps: detect `_claim_names.groups` or `hasgroups` and call Microsoft Graph (`getMemberObjects` / `transitiveMemberOf`) on demand.
14. **After any remediation, refresh tokens:** AD replication must complete across all DCs; users must **log off / log on** (or run `klist purge`) to pick up the new token. Otherwise the old, oversized token persists until natural expiry.

### 8.3 Architectural
12. **Replace group‑based ACLs with claims** via Dynamic Access Control on file servers.
13. **Consolidate app permissions** using app roles instead of security groups where possible.
14. **Use Conditional Access and PIM** for privileged access instead of permanent group membership.

### 8.4 Emergency stopgaps for HTTP 400 (last resort, change-controlled)

> ⚠️ **WARNING — enterprise change control required.** Microsoft explicitly describes `HTTP.sys` tuning as *"extremely dangerous"*. Changes affect **every HTTP.sys-hosted service on the machine** (IIS, WinRM, WSUS, PowerShell remoting, WCF). Perform only inside a maintenance window, after group-reduction efforts, and size values based on *measured* token size using Microsoft's sizing formula **`(4/3) × T + 200`** (where `T` is the user's actual Kerberos token size), not arbitrary maximums.

16. **First: reduce group memberships and `SIDHistory`** — tuning HTTP.sys does not solve the underlying issue; it only delays failure.
17. If temporary tuning is unavoidable, edit:
    ```
    HKLM\SYSTEM\CurrentControlSet\Services\HTTP\Parameters
      MaxFieldLength    (DWORD)   :: header field size; default 16,384; max 65,534
      MaxRequestBytes   (DWORD)   :: total request size; default 16,384; max 16,777,216 (16 MB)
    ```
    Size both to `(4/3) × T + 200`, rounded up. Restart `HTTP.sys` (`net stop http /y && net start http`) — this will bounce IIS, WinRM, WSUS, and all other HTTP.sys listeners.
18. After changes, validate via `%windir%\System32\LogFiles\HTTPERR` (look for `FieldLength` / `RequestLength` reason codes) and targeted user traffic tests.

---

## 9. Quick reference — the numbers

| Limit | Value | Tunable? | Effect when exceeded |
|---|---|---|---|
| LSA access token SID array | **1,024 SIDs** (~1,010 custom) | ❌ No | **Logon fails** |
| Kerberos `MaxTokenSize` default (Server 2012+) | **48,000 bytes** | ✅ Yes | Logon / HTTP failures |
| Kerberos `MaxTokenSize` absolute max | **65,535 bytes** | ✅ Yes | IPsec / IIS breakage above recommended |
| IIS `MaxFieldLength` (HTTP.sys header field size) | default **16,384**; max **65,534** | ✅ Yes | HTTP 400 |
| IIS `MaxRequestBytes` (HTTP.sys total request size) | default **16,384**; max **16,777,216 (16 MB)** | ✅ Yes | HTTP 400 |
| JWT `groups` claim emission (Entra ID) | **200 groups** | ❌ No (overage claim instead) | Overage — requires Graph call |
| SAML `groups` claim emission (Entra ID) | **150 groups** | ❌ No (overage claim instead) | Overage — requires Graph call |
| Implicit-flow `groups` claim (Entra ID) | **6 groups** (some docs: 5) | ❌ No (`hasgroups` claim instead) | Requires Graph call |
| Entra ID groups an object can be member of | **No fixed directory limit** — but service ceilings apply: Entra Kerberos **1,010**; SharePoint Online **2,047**; Conditional Access warning **2,048**; Conditional Access hard fail **4,096** | — | Service-specific (auth failure, access block, policy evaluation failure) |
| Entra ID groups a non‑admin user can create | **250** | ✅ Via role | — |

---

## 10. References

### Kerberos / LSA token
- [Problems with Kerberos authentication when a user belongs to many groups (KB 327825)](https://learn.microsoft.com/troubleshoot/windows-server/windows-security/kerberos-authentication-problems-if-user-belongs-to-groups)
- [Logging on a user account that is a member of more than 1,010 groups may fail (KB 328889)](https://learn.microsoft.com/troubleshoot/windows-server/windows-security/logging-on-user-account-fails)
- ["Not enough storage is available to complete this operation" (KB 935744)](https://learn.microsoft.com/troubleshoot/windows-client/windows-security/not-enough-storage-available-complete-operation-error)
- [HTTP 400 Bad Request (Request Header too long) (KB 2020943)](https://learn.microsoft.com/troubleshoot/developer/webapps/iis/www-authentication-authorization/http-bad-request-response-kerberos)
- [Active Directory Domain Services — maximum limits and scalability](https://learn.microsoft.com/windows-server/identity/ad-ds/plan/active-directory-domain-services-maximum-limits)
- [Updates to TGT delegation across incoming trusts (KB 4490425)](https://support.microsoft.com/help/4490425)
- [How to limit the header size of HTTP transmission that IIS accepts (KB 310156)](https://support.microsoft.com/help/310156)

### Entra ID (cloud)
- [Microsoft Entra service limits and restrictions](https://learn.microsoft.com/entra/identity/users-groups-roles/directory-service-limits-restrictions)
- [Configure group claims for applications with Microsoft Entra ID](https://learn.microsoft.com/entra/identity/hybrid/connect/how-to-connect-fed-group-claims)
- [Access tokens in the Microsoft identity platform — overage claim](https://learn.microsoft.com/entra/identity-platform/access-tokens)
- [Microsoft Graph `getMemberObjects` API](https://learn.microsoft.com/graph/api/directoryobject-getmemberobjects) (recommended for groups-overage handling — returns transitive memberships)
- [Microsoft Graph `user: transitiveMemberOf`](https://learn.microsoft.com/graph/api/user-list-transitivememberof)

### Hybrid Kerberos
- [Windows Hello for Business cloud Kerberos trust deployment](https://learn.microsoft.com/windows/security/identity-protection/hello-for-business/hello-hybrid-cloud-kerberos-trust)
- [Azure Files identity‑based authentication with Entra Kerberos](https://learn.microsoft.com/azure/storage/files/storage-files-identity-auth-hybrid-identities-enable)

---

## 11. Appendix — Recommended action plan for the customer

1. **Measure** — run §7 scripts to inventory top users by group count and by Kerberos ticket size.
2. **Classify** — separate security groups from distribution / Microsoft 365 Groups; identify candidates for removal or replacement.
3. **Remediate** — prune stale memberships, flatten nesting, retire `SIDHistory` after migration validation.
4. **Harden apps** —
   - For on‑prem IIS/SharePoint/Exchange: verify `MaxTokenSize` is at default 48,000 and IIS defaults are healthy.
   - For custom Entra ID apps: implement groups‑overage claim handling via Microsoft Graph.
   - Restrict per‑app group claims to "Groups assigned to the application".
5. **Monitor** — alert on:
   - Users > 500 transitive groups.
   - **Security Event ID 4625 with status `0xC000015A`** (`STATUS_TOO_MANY_CONTEXT_IDS`).
   - HTTP 400 spikes in IIS logs (`sc‑status=400, sc‑substatus=0`) **and** `HTTPERR` reason codes `FieldLength` / `RequestLength`.
6. **Govern** — adopt Entra ID entitlement management, access reviews, and dynamic groups to prevent future sprawl.

---

## 12. Review & Validation Log

This document was authored and independently reviewed by AI agents against the first-party Microsoft Learn / Microsoft Support knowledge base. Full model attribution is provided below as part of the integrity record.

### Authoring & review — models used

| Role | Model | Vendor | Purpose |
|---|---|---|---|
| **Primary author / orchestrator** | `claude-opus-4.7` | Anthropic | Initial drafting of all sections, research orchestration, cross-review synthesis, corrections application |
| **Reviewer 1 — on-prem Kerberos / LSA accuracy** | `claude-sonnet-4.6` | Anthropic | Independent fact-check of §1–3, §5–9 against KBs 327825, 328889, 938118, 935744, 2020943, 4490425 and HTTP.sys registry documentation |
| **Reviewer 2 — cloud Entra ID / OAuth / SAML / Graph accuracy** | `claude-sonnet-4.6` | Anthropic | Independent fact-check of §1, §4, §5, §7.2, §8–10 against Entra ID token claims reference, group-claims configuration, Graph API, Entra Kerberos, Azure Files, and WHfB cloud Kerberos trust documentation |
| **Reviewer 3 — quality, completeness & dangerous-advice critique** | `gpt-5.4` | OpenAI | Independent rubber-duck critique across all sections — identified dangerous advice (HTTP.sys workaround), internal contradictions, incorrect event IDs, missing cross-forest/trust nuance, missing Group Policy deployment path, and clarity issues |

All three reviewer agents had full access to the **Microsoft Learn MCP server** (`microsoft_docs_search` and `microsoft_docs_fetch`) and executed live lookups against first-party Microsoft documentation during their passes. No training-data-only recall was relied upon for numerical or configuration claims.

### Review passes performed

| # | Scope | Reviewer | Outcome |
|---|---|---|---|
| 1 | On-premises Kerberos & LSA token claims (§1, §2, §3, §5, §6, §7, §8, §9) | Accuracy agent — validated 13 factual claims against first-party KBs (327825, 328889, 938118, 935744, 2020943, 4490425, HTTP.sys registry docs) | Confirmed 10; **3 corrections applied** (unconstrained-delegation formula, `MaxRequestBytes` maximum value, `tokensz.exe` attribution); **1 high-impact correction applied** (Event ID 45058 replaced with Security 4625 / `STATUS_TOO_MANY_CONTEXT_IDS`); IIS "64 KB default" language clarified. |
| 2 | Cloud Entra ID / OAuth / SAML / Graph claims (§1, §4, §5, §7.2, §8, §9, §10) | Accuracy agent — validated 11 factual claims against first-party Microsoft Learn / Graph / Azure Files / WHfB / Entra Kerberos documentation | Confirmed JWT 200 / SAML 150 overage, `_claim_names` / `_claim_sources` mechanism, `groupMembershipClaims` values, "Groups assigned to the application", 250-group non-admin creation cap, `Get-MgUserTransitiveMemberOf` usage, and that Entra Domain Services / Entra Kerberos / WHfB cloud Kerberos trust / Azure Files all re-introduce the 1,010-SID Kerberos limit. **2 corrections applied**: (a) Graph API endpoint corrected to `getMemberObjects` / `transitiveMemberOf` with explicit warning against legacy `graph.windows.net` URL in `_claim_sources.endpoint`; (b) "No fixed limit" row in §9 qualified with service-specific ceilings (Entra Kerberos 1,010; SharePoint 2,047; Conditional Access 2,048 / 4,096). |
| 3 | General quality, completeness, and dangerous-advice critique | Rubber-duck review agent | 16 findings across CRITICAL/HIGH/MEDIUM/LOW. **All CRITICAL and HIGH items addressed**: dangerous HTTP.sys workaround rewritten as change-controlled emergency stopgap with sizing formula; internal contradiction on 64 KB default resolved; event-ID guidance replaced; assessment-script caveats added; cross-forest/trust nuance added (§5.1); `MaxTokenSize` deployment via Group Policy added; Connect Sync vs. Cloud Sync comparison added (§5.2); ticket-refresh operational note added; overage-does-not-equal-login-failure clarified; implicit-flow limit added; SID compression rewording applied. |

### Corrections applied relative to initial draft

1. **Event ID mapping (§6, §11)** — removed incorrect `LsaSrv 45058` (cached-credential FIFO eviction, per KB 2555663); replaced with **Security Event ID 4625, status `0xC000015A` (`STATUS_TOO_MANY_CONTEXT_IDS`)** per KB 328889.
2. **Unconstrained delegation formula (§2.2.1)** — corrected to *"multiply the entire `TokenSize` by 2"* (i.e., `2 × (1200 + 40d + 8s)`) rather than doubling only the variable term. Added trust-boundary scope caveat for KB 4490425.
3. **`MaxRequestBytes` maximum (§8.4, §9)** — corrected from 65,534 to the actual 16,777,216 (16 MB) per HTTP.sys registry documentation.
4. **`tokensz.exe` attribution (§7.2)** — corrected from "Windows SDK" to **Windows Server 2003 Resource Kit Tools** (Microsoft Download Center ID 17657).
5. **IIS 64 KB framing (§2.2.2)** — clarified as the *hard architectural maximum* configurable for `MaxFieldLength`, not the registry default (which is 16,384); eliminated internal contradiction with the §9 quick-reference table.
6. **HTTP.sys tuning (§8.4)** — reframed as a last-resort, change-controlled emergency mitigation with Microsoft's sizing formula `(4/3) × T + 200`, explicit warning that HTTP.sys restart bounces IIS/WinRM/WSUS/PowerShell Remoting, and `HTTPERR` validation guidance.
7. **Assessment scripts (§7.1)** — added explicit caveat that directory-side queries do not reflect the effective resource-side token; `[WindowsIdentity]` approach removed; guidance redirected to on-host `whoami /groups` and `tokensz.exe`.
8. **Implicit-flow 6-group limit (§4, §9)** — added alongside the existing JWT/SAML limits; noted `hasgroups` claim behavior.
9. **Graph API reference (§4, §8.2)** — switched from `/me/memberOf` / `getMemberGroups` to the more accurate `transitiveMemberOf` / `getMemberObjects` pair used by the identity platform.
10. **Cross-domain / cross-forest nuance (new §5.1)** — added to explain resource-dependent effective tokens (domain-local groups, server-local groups, SID filtering across trusts).
11. **Entra Connect Sync vs. Cloud Sync (new §5.2)** — added current-state guidance on Group Writeback v2 deprecation in Connect Sync.
12. **`MaxTokenSize` deployment (§8.2)** — added Group Policy path and estate-wide consistency warning; clarified that PAC/Resource SID Compression is a consequence of modern OS/domain versions, not a toggle.
13. **Ticket refresh operational note (§8.2)** — added requirement to log off / `klist purge` after remediation so users pick up the new token.

### Primary sources cross-checked

- [KB 327825 — Problems with Kerberos authentication when a user belongs to many groups](https://learn.microsoft.com/troubleshoot/windows-server/windows-security/kerberos-authentication-problems-if-user-belongs-to-groups) — verified `MaxTokenSize` defaults (12,000 / 48,000), recommended max (48,000), absolute max (65,535), formula, Base64 arithmetic (48,000 × 1.33 ≈ 64 KB), IPsec IKE break above 66,536, unconstrained-delegation doubling.
- [KB 328889 — Logging on a user account that is a member of more than 1,010 groups may fail](https://learn.microsoft.com/troubleshoot/windows-server/windows-security/logging-on-user-account-fails) — verified 1,024 SID array limit, ~1,010 practical, protocol-independence, not configurable.
- [KB 938118 — Add MaxTokenSize registry entry via Group Policy](https://learn.microsoft.com/troubleshoot/windows-server/group-policy/group-policy-add-maxtokensize-registry-entry) — verified registry path, GPO deployment, 65,535 max.
- [KB 935744 — "Not enough storage is available to complete this operation"](https://learn.microsoft.com/troubleshoot/windows-client/windows-security/not-enough-storage-available-complete-operation-error) — verified symptom mapping.
- [KB 2020943 — HTTP 400 (Request Header Too Long)](https://learn.microsoft.com/troubleshoot/developer/webapps/iis/www-authentication-authorization/http-bad-request-response-kerberos) — verified HTTP header overflow, sizing formula, 16 MB `MaxRequestBytes` cap.
- [KB 4490425 — Updates to TGT delegation across incoming trusts](https://support.microsoft.com/help/4490425) — verified 2019 default change scope (incoming forest trusts).
- [HTTP.sys registry settings reference](https://learn.microsoft.com/troubleshoot/iis/windows/httpsys-registry-settings) — verified `MaxFieldLength` default 16,384 / max 65,534; `MaxRequestBytes` default 16,384 / max 16,777,216.
- [Active Directory Domain Services maximum limits and scalability](https://learn.microsoft.com/windows-server/identity/ad-ds/plan/active-directory-domain-services-maximum-limits) — verified recommended Kerberos ticket size guidance.
- [Configure group claims for applications with Microsoft Entra ID](https://learn.microsoft.com/entra/identity/hybrid/connect/how-to-connect-fed-group-claims) — verified `groupMembershipClaims` values, app-assigned groups guidance, implicit-flow 5/6 limit, `hasgroups` behavior.
- [ID token claims reference](https://learn.microsoft.com/entra/identity-platform/id-token-claims-reference) — verified JWT 200 / SAML 150 overage thresholds, `_claim_names` / `_claim_sources`, `groups:src1` mechanics.
- [Microsoft Entra service limits and restrictions](https://learn.microsoft.com/entra/identity/users-groups-roles/directory-service-limits-restrictions) — verified non-admin 250-group creation cap.
- [KB 2555663 — LSASRV 45058 cached credential FIFO deletion](https://learn.microsoft.com/troubleshoot/windows-client/user-profiles-and-logon/lsasrv-event-45058-cached-credential) — verified Event 45058 is **not** the too-many-groups indicator.

### Residual caveats

- The "~14 well-known SIDs" subtraction (1,024 − 1,010) is an arithmetic inference, not a documented fixed count — Microsoft notes the exact number varies by logon type, OS version, and security features (MFA, Credential Guard, DAC).
- The implicit-flow group limit appears as both **5** and **6** across different Microsoft Learn pages; both values are cited above for transparency. Treat as "≤ 6".
- The one Microsoft Support URL that could not be reached during verification (KB 310156) is referenced by current in-support Microsoft KBs and is retained as-cited.

---

*AI-generated draft. Content consolidates official Microsoft Learn and Microsoft Support documentation as of April 2026 and has been cross-checked by three independent AI-agent review passes (see §12 for model attribution). **Not a Microsoft-published document. Requires human SME validation before customer distribution.** Pilot all configuration changes in a non-production environment.*

*🤖 **Authored and quality-reviewed by AI agents.** This document is the product of AI-driven research and multi-agent validation using Anthropic's `claude-opus-4.7` (primary author) and `claude-sonnet-4.6` (accuracy reviewers), and OpenAI's `gpt-5.4` (quality critique), all with live access to Microsoft Learn. **It is not a Microsoft-published document and has not been reviewed by a human Microsoft subject-matter expert.** It is provided as AI-generated practitioner guidance grounded in first-party Microsoft documentation. Human SME validation is required before customer distribution.*
