# ANF SMB Volume Creation — AD Integration Flow

**Captured environment:** Windows Server 2019 DC  
**ANF node:** `10.10.1.10`  
**AD Domain Controller:** `10.10.10.8` (gary-ad.corp.azure)  
**Domain:** corp.azure  
**Computer account created:** `ANF-AU-AA30`

---

## Introduction

> **Disclaimer:** This is a personal experience-sharing write-up, not an official Microsoft or NetApp document. The observations, interpretations, and conclusions here are based on my own lab experiments and customer engagements experiences — they may be incomplete, contain mistakes, or become inaccurate as ANF, Windows Server, and Active Directory evolve over time. Always refer to the [official Azure NetApp Files documentation](https://learn.microsoft.com/azure/azure-netapp-files/) for authoritative guidance before making production decisions.

ANF with SMB protocol is one of the most powerful ways to deliver enterprise file services in Azure — but it is also one of the most common sources of head-scratching support cases. Across customer engagements, the same pain points come up again and again:

- Volume creation fails with a cryptic error like *"SASL bind to LDAP failed — PTR record missing"* even though forward DNS resolves correctly
- The domain join appears to succeed but clients cannot authenticate to the share
- Intermittent Kerberos errors that only surface in certain subnets or regions
- NSG rules that silently break LDAP or KPASSWD while leaving DNS and SMB reachable
- AD Sites and Services misconfiguration causing ANF to reach out to a DC in a different region

In most cases the root cause is not a product bug — it is that ANF performs a **full AD domain join**, the same way a Windows Server would. That means DNS, Kerberos, LDAP, SMB/IPC$, KPASSWD, and Netlogon all need to work correctly, in the right order, before a single SMB volume can be used.

### The Troubleshooter's Baseline Problem

One of the hardest parts of diagnosing ANF SMB issues is not knowing what *normal* looks like. When something goes wrong, it is difficult to tell whether a particular error message, a "missing" packet type, or an unexpected protocol sequence is the cause of the problem — or just routine background behaviour that always happens.

This write-up addresses that directly. It uses a **minimal Azure lab** and **Wireshark packet captures** taken during a clean, successful run to document exactly what the normal protocol flow looks like — for both volume creation and client access. Every phase, every protocol, every expected "error" response (like `STATUS_OBJECT_NAME_NOT_FOUND` or `STATUS_INVALID_DEVICE_REQUEST`) is called out and explained so you know it is expected and harmless.

The intended use is as a **reference baseline for troubleshooting**: when something breaks in your environment, compare your capture against the normal sequence here to find where it diverges.

### What a Successful ANF SMB Setup Looks Like End-to-End

A fully working setup involves two distinct flows, each with its own protocol sequence:

**Flow 1 — ANF SMB Volume Creation (AD domain join)**

When you create an ANF SMB volume, the ANF node performs the equivalent of a Windows domain join in the background. The following protocols must all succeed in order:

| Step | Protocol | Port | What happens |
|------|----------|------|--------------|
| 1 | DNS | 53 | ANF resolves the domain and discovers the DC via SRV and A records |
| 2 | LDAP | 389 | ANF reads the AD Root DSE anonymously to confirm domain info (×3) |
| 3 | DNS | 53 | ANF queries site-scoped SRV records to find the closest DC |
| 4 | Kerberos | 88 | ANF authenticates and obtains an LDAP service ticket |
| 5 | LDAP | 389 | ANF binds with Kerberos (SASL/GSS-API) and searches for existing accounts |
| 6 | LDAP | 389 | ANF creates the computer account (`CN=ANF-AU-AA30`) in AD |
| 7 | SMB / IPC$ | 445 | ANF opens an IPC$ session and calls LSARPC to resolve domain SIDs |
| 8 | KPASSWD | 464 | ANF sets the machine account password |
| 9 | LDAP | 389 | ANF writes SPNs, `dnsHostName`, and `userAccountControl` to the account |
| 10 | SMB / Netlogon | 445 | ANF establishes a Netlogon secure channel — the formal domain join |
| 11 | Kerberos + SMB | 88 + 445 | ANF obtains a Kerberos identity for the SMB volume itself |
| 12 | DNS | 53 | ANF begins periodic site-aware DC keepalive queries |

If any one of these steps fails, volume creation fails — and the error message surfaced in the Azure portal often points to step 4 (Kerberos / PTR record) even when the real failure is at step 2 (LDAP channel binding) or step 8 (KPASSWD port blocked).

**Flow 2 — Windows 11 Client Mount and File Access**

Once the volume exists, a Windows client mounting the share goes through its own multi-step flow. Most of it is invisible to the user:

| Step | Protocol | What happens |
|------|----------|--------------|
| A | DNS + Kerberos | Client resolves the ANF hostname and obtains a CIFS service ticket from the KDC |
| B | SMB | Client negotiates dialect, sets up authenticated session, and connects to the share |
| 1 | SMB | Explorer probes for `Desktop.ini` and `AutoRun.inf` — both return NOT_FOUND (normal) |
| 2 | SMB | Explorer lists the share root — directory entries returned via `Find` |
| 3 | SMB | Explorer queries share capacity for the status bar display |
| 4 | SMB | User browses into a subfolder — Explorer registers a change notification watch, then cancels it on navigation |
| 5 | SMB / IPC$ | Client connects to `IPC$`, checks for DFS (NOT_FOUND is expected), and calls `NetShareGetInfo` via SRVSVC |
| 6 | SMB | Client issues `FSCTL_CREATE_OR_GET_OBJECT_ID` when opening files — ANF returns `STATUS_INVALID_DEVICE_REQUEST` (normal, NTFS Object IDs not supported) |

Steps A and B are not visible in the capture because Windows reuses an already-established SMB session. The "errors" at steps 1, 5, and 6 are all expected and non-fatal — they are included here specifically because they appear alarming in a capture if you do not know to expect them.

### How to Use This Write-Up

1. **Read the overview diagrams** to understand the full sequence at a glance
2. **Read the phase-by-phase explanations** when you need to understand what a specific protocol exchange is doing and why
3. **When troubleshooting**, take a Wireshark capture at the DC during your failing volume creation (see the Troubleshooting Guide at the end), then compare each phase against the expected sequence documented here — the first phase that diverges is where to focus
4. **When planning a new deployment**, use the protocol and subnet details here as a concrete reference for network address design, NSG rules, AD DS site naming, and subnet-to-site mapping — the lab setup shows exactly which subnets need to reach which ports on the DC, and Phase 3 shows why every ANF and workload subnet must be registered in AD Sites and Services for site-scoped DC discovery to work



---

## Lab Setup

All components are deployed in Azure within a single VNet:

- **Windows 11 VM** (`10.10.10.9/24`) — client used to map and test the SMB volume; packet captures taken here
- **Windows Server 2019 DC** (`10.10.10.8/24`) — Active Directory domain controller for `corp.azure`; packet captures taken here
- **ANF SMB volume** (`10.10.1.10/24`) — Azure NetApp Files node hosting the `/smb1` SMB share; domain-joined as `ANF-AU-AA30`

![Lab network topology](network-diagram.jpg)

---

## ANF Active Directory Connection Configuration

The ANF AD connection was configured in the Azure portal with the following settings, which directly correspond to the Windows AD environment above:

![ANF AD connection settings in the Azure portal](anf-ad-connection.jpg)

| Setting | Value | Notes |
|---------|-------|-------|
| Primary DNS | `10.10.10.8` | IP address of the Windows Server DC |
| AD DNS Domain Name | `corp.azure` | Domain FQDN |
| AD Site Name | `Default-First-Site-Name` | Must match the AD Sites and Services site name |
| SMB Server (Computer Account) Prefix | `anf-au` | Prefix used to generate the machine account name (`ANF-AU-AA30`) |
| Organizational Unit Path | `CN=Computers` | Default Computers container in AD |
| LDAP Signing | Unchecked | Windows Server 2019 DC does not enforce LDAP signing by default |
| LDAP over TLS | Unchecked | Plain LDAP port 389 used throughout the capture |
| AES Encryption | Unchecked | RC4 accepted; enabling AES requires the DC to have AES keys for the account |

---

## Introduction — Role of DNS, Kerberos, and LDAP

Creating an ANF SMB volume is not a simple file-share operation. ANF must fully join the Active Directory domain as a first-class member — the same way a Windows Server would. Three protocols work together in sequence to make this happen.

### DNS — The Navigation Layer

DNS is the first protocol ANF uses and the foundation everything else depends on. Before any authentication or directory operation can occur, ANF must answer two questions:

1. **Where is the domain controller?** — ANF queries DNS SRV records (`_kerberos._tcp.dc._msdcs.CORP.AZURE`, `_ldap._tcp.Default-First-Site-Name._sites.CORP.AZURE`) to locate a DC by name.
2. **What IP does that DC have?** — A follow-up A record query resolves the DC hostname to an IP address.

![DNS A record for the domain controller](dns-forward.jpg)

DNS also serves a critical security role during Kerberos: ANF performs a **reverse PTR lookup** on the DC's IP address to construct the Kerberos Service Principal Name (SPN). If the PTR record is missing, wrong, or returns an IPv6 address, Kerberos cannot build the SPN and the entire domain join fails — even though forward DNS is healthy. This is why the reverse lookup zone and PTR record are mandatory prerequisites, not optional configurations.

![PTR record for the domain controller in the reverse lookup zone](dns-ptr.jpg)

AD Sites and Services extends DNS by publishing **site-scoped SRV records**. ANF uses these to select a DC that is topologically close (in the same Azure region/subnet), avoiding cross-region Kerberos and LDAP latency.

![AD DS Sites and Services configuration](ad-ds.jpg)

### Kerberos — The Authentication Layer

Once DNS has resolved the DC location, ANF authenticates using **Kerberos** — a ticket-based protocol where passwords are never transmitted over the network. The exchange works in two stages:

1. **TGT (Ticket Granting Ticket):** ANF presents its identity to the Key Distribution Center (KDC, running on the DC at port 88) with an encrypted timestamp as proof. The KDC responds with a TGT — a time-limited credential that proves ANF is who it claims to be.
2. **Service Ticket:** ANF presents the TGT back to the KDC and requests a specific service ticket (e.g., for `ldap/gary-ad.corp.azure` or `cifs/gary-ad.corp.azure`). The KDC issues a service ticket encrypted with the target service's key.

Kerberos is used four times during SMB volume creation:
- To authenticate ANF's AD connection account for LDAP (Phase 4)
- To open the SMB2/IPC$ session for LSARPC domain SID lookup (Phase 7)
- To obtain a ticket for the KPASSWD service to set the machine account password (Phase 8)
- To establish the Netlogon secure channel and the SMB volume's domain identity (Phases 10–11)

Clock synchronisation is critical: Kerberos rejects any ticket where the client and server clocks differ by more than **5 minutes** (`KRB_AP_ERR_SKEW`). The domain controller must sync with a reliable NTP source.

### LDAP — The Directory Operations Layer

With a valid Kerberos service ticket, ANF opens an authenticated LDAP session (port 389) to the DC using **SASL GSS-API** — a standardised mechanism that wraps the Kerberos ticket inside the LDAP bind request. This proves identity without re-sending credentials.

Over this authenticated session, ANF performs all Active Directory operations required to provision the SMB volume:

| Operation | Purpose |
|-----------|---------|
| Anonymous Root DSE read | Discover domain naming context and DC capabilities |
| Site SRV lookup (via DNS) | Select the site-local DC for all further operations |
| Authenticated bind | Establish a secure, identity-verified LDAP session |
| OU and partition discovery | Confirm target container exists before writing |
| `addRequest` — computer account | Create the ANF machine account (`CN=ANF-AU-AA30`) in AD |
| `modifyRequest` — set attributes | Write SPNs, `dnsHostName`, and `userAccountControl` flags |

LDAP is also subject to **channel binding and signing policies** on the DC. If the DC requires LDAP signing or channel binding (enforced by default on Windows Server 2025), ANF's plain-port-389 SASL bind will be rejected — producing the misleading *"PTR record missing"* error even when DNS is correct.

---

## End-to-End Flow Overview

![Wireshark packet capture of ANF SMB volume creation — taken on the domain controller](anf-smb-volume-creation-at-dc.jpg)

The 12 phases below show the sequence of protocol operations between the ANF node and the domain controller during SMB volume creation. Each phase is expanded with full packet detail in the [Phase-by-Phase Explanation](#phase-by-phase-explanation) section below.

```mermaid
sequenceDiagram
    participant ANF as ANF Node<br/>10.10.1.10
    participant DC as Domain Controller<br/>10.10.10.8

    rect rgb(194, 211, 233)
        Note over ANF,DC: Phase 1 — Domain & DC Discovery (DNS :53)
        ANF->>DC: DNS SRV / A queries
        DC-->>ANF: DC hostname + IP
    end

    rect rgb(181, 218, 192)
        Note over ANF,DC: Phase 2 — Anonymous LDAP Root DSE Probe (LDAP :389)
        ANF->>DC: Anonymous LDAP bind + Root DSE query ×3
        DC-->>ANF: defaultNamingContext, domain info
    end

    rect rgb(219, 205, 147)
        Note over ANF,DC: Phase 3 — AD Site-Aware DC Discovery (DNS :53)
        ANF->>DC: Site-scoped DNS SRV queries
        DC-->>ANF: Site-local DC for LDAP + Kerberos
    end

    rect rgb(225, 171, 161)
        Note over ANF,DC: Phase 4 — Kerberos Authentication (KDC :88)
        ANF->>DC: AS-REQ → AS-REP (TGT) + TGS-REQ → TGS-REP
        DC-->>ANF: LDAP service ticket issued
    end

    rect rgb(185, 158, 220)
        Note over ANF,DC: Phase 5 — Authenticated LDAP Bind + AD Queries (LDAP :389)
        ANF->>DC: SASL GSS-API bind + OU / partition discovery
        DC-->>ANF: Naming contexts confirmed, no existing account
    end

    rect rgb(157, 217, 204)
        Note over ANF,DC: Phase 6 — Computer Account Creation (LDAP :389)
        ANF->>DC: addRequest CN=ANF-AU-AA30
        DC-->>ANF: addResponse: success
    end

    rect rgb(213, 191, 152)
        Note over ANF,DC: Phase 7 — LSARPC Domain SID Lookup (SMB :445 / IPC$)
        ANF->>DC: SMB2 session + lsa_LookupSids2
        DC-->>ANF: Domain SIDs resolved
    end

    rect rgb(218, 187, 151)
        Note over ANF,DC: Phase 8 — Set Machine Account Password (KPASSWD :464)
        ANF->>DC: KPASSWD request
        DC-->>ANF: Password set: success
    end

    rect rgb(147, 175, 215)
        Note over ANF,DC: Phase 9 — Finalise Computer Account Attributes (LDAP :389)
        ANF->>DC: modifyRequest (SPNs, dnsHostName, userAccountControl)
        DC-->>ANF: modifyResponse: success
    end

    rect rgb(209, 143, 209)
        Note over ANF,DC: Phase 10 — Netlogon Secure Channel (SMB :445 / NETLOGON)
        ANF->>DC: NetrServerReqChallenge + NetrServerAuthenticate2
        DC-->>ANF: Secure channel established
    end

    rect rgb(151, 222, 151)
        Note over ANF,DC: Phase 11 — SMB Volume Kerberos Identity (KDC :88 + SMB :445)
        ANF->>DC: AS-REQ / TGS-REQ for volume identity
        DC-->>ANF: SMB service ticket issued
    end

    rect rgb(212, 151, 151)
        Note over ANF,DC: Phase 12 — Ongoing Site-Aware DC Keepalive (DNS :53)
        loop Per SMB node refresh
            ANF->>DC: Site-scoped SRV queries
            DC-->>ANF: Site-local DC refreshed
        end
    end
```

---

## Phase-by-Phase Explanation

### Phase 1 — Domain & DC Discovery

```mermaid
sequenceDiagram
    participant ANF as ANF Node<br/>10.10.1.10
    participant DC as DNS / DC<br/>10.10.10.8

    rect rgb(194, 211, 233)
        Note over ANF,DC: Phase 1 — Domain & DC Discovery (DNS :53)
        ANF->>DC: A example.corp.azure?
        DC-->>ANF: NXDOMAIN (confirms DNS is authoritative)
        ANF->>DC: SRV _kerberos._tcp.dc._msdcs.CORP.AZURE?
        DC-->>ANF: gary-ad.corp.azure:88
        ANF->>DC: A gary-ad.corp.azure?
        DC-->>ANF: 10.10.10.8
    end
```

ANF validates DNS reachability and discovers the DC location.

| Frames | Query | Response | Purpose |
|--------|-------|----------|---------|
| 12830–12831 | A `example.corp.azure` | NXDOMAIN | Confirms DNS is reachable and authoritative for `corp.azure` |
| 12866–12867 | SRV `_kerberos._tcp.dc._msdcs.CORP.AZURE` | `gary-ad.corp.azure:88` | Locates a KDC for the domain |
| 12868–12869 | A `gary-ad.corp.azure` | `10.10.10.8` | Resolves the KDC hostname to an IP address |

**Why the DNS A record is critical:**  
SRV records return hostnames, not IPs. ANF resolves the SRV target with a follow-up A query. If the A record is missing or wrong, ANF cannot open a TCP connection to the DC even though DC discovery succeeded.

---

### Phase 2 — Anonymous LDAP Root DSE Probe

```mermaid
sequenceDiagram
    participant ANF as ANF Node<br/>10.10.1.10
    participant DC as LDAP / DC<br/>10.10.10.8 :389

    rect rgb(181, 218, 192)
        Note over ANF,DC: Phase 2 — Anonymous LDAP Root DSE Probe (×3)
        ANF->>DC: bindRequest (anonymous simple)
        DC-->>ANF: bindResponse: success
        ANF->>DC: searchRequest "<ROOT>" baseObject
        DC-->>ANF: searchResEntry [defaultNamingContext, supported features]
        ANF->>DC: unbindRequest
        Note over ANF,DC: Repeated on 2 more independent connections
    end
```

ANF opens a plain LDAP connection (port 389) with an **anonymous simple bind** to read the Root DSE — the DC's public capability advertisement. Three separate anonymous connections are made from different source ports (`19718`, `57030`, `55314`), each performing the same Root DSE query independently.

- Retrieves: `defaultNamingContext` (`DC=corp,DC=azure`), supported LDAP versions, domain FQDN
- This probe is unauthenticated — any reachable DC responds without credentials
- Each connection terminates with an `unbindRequest` before the next phase begins

**Why this step exists:**  
ANF reads `defaultNamingContext` and confirms it matches the configured domain before investing in Kerberos. The multiple independent connections reflect parallel ANF SMB subsystem probes confirming site and domain consistency.

---

### Phase 3 — AD Site-Aware DC Discovery

```mermaid
sequenceDiagram
    participant ANF as ANF Node<br/>10.10.1.10
    participant DC as DNS / DC<br/>10.10.10.8 :53

    rect rgb(219, 205, 147)
        Note over ANF,DC: Phase 3 — AD Site-Aware DC Discovery (DNS :53)
        ANF->>DC: SRV _ldap._tcp.Default-First-Site-Name._sites.dc._msdcs.CORP.AZURE?
        DC-->>ANF: gary-ad.corp.azure:389
        ANF->>DC: SRV _ldap._tcp.Default-First-Site-Name._sites.CORP.AZURE?
        DC-->>ANF: gary-ad.corp.azure:389
        ANF->>DC: SRV _kerberos._tcp.Default-First-Site-Name._sites.CORP.AZURE?
        DC-->>ANF: gary-ad.corp.azure:88
    end
```

After confirming the domain, ANF queries for **site-scoped** SRV records using the AD site name `Default-First-Site-Name`:

| DNS Query | Purpose |
|-----------|---------|
| `_ldap._tcp.Default-First-Site-Name._sites.dc._msdcs.CORP.AZURE` | LDAP-capable DC in the configured site |
| `_ldap._tcp.Default-First-Site-Name._sites.CORP.AZURE` | Any LDAP server in the site |
| `_kerberos._tcp.Default-First-Site-Name._sites.CORP.AZURE` | KDC in the site |

**Why AD Sites and Services is critical:**  
These SRV records only exist when subnets are mapped to a site in **AD Sites and Services**. Without subnet-to-site mapping:
- Site-scoped SRV queries return NXDOMAIN
- ANF falls back to domain-wide DC selection, which may select a distant or wrong DC
- In multi-DC environments, inconsistent DC selection causes intermittent failures

**Required configuration:**
- A Site object (e.g., `Default-First-Site-Name`)
- Subnet objects for all Azure subnets linked to that site:
  - `10.10.10.0/24` — workload subnet
  - `10.10.1.0/24` — ANF delegated subnet
  - `10.10.2.0/24` — other subnets
- DC placed into the same site

---

### Phase 4 — Kerberos Authentication

```mermaid
sequenceDiagram
    participant ANF as ANF Node<br/>10.10.1.10
    participant DC as KDC / DC<br/>10.10.10.8 :88

    rect rgb(225, 171, 161)
        Note over ANF,DC: Phase 4 — Kerberos Authentication (KDC :88)
        ANF->>DC: AS-REQ (no pre-auth)
        DC-->>ANF: KRB5KDC_ERR_PREAUTH_REQUIRED (normal)
        ANF->>DC: AS-REQ (with encrypted timestamp pre-auth)
        DC-->>ANF: AS-REP — TGT issued
        ANF->>DC: TGS-REQ (ticket for ldap/gary-ad.corp.azure)
        DC-->>ANF: TGS-REP — LDAP service ticket issued
    end
```

ANF authenticates to the DC using Kerberos. No passwords are sent over the wire.

| Frames | Exchange | Outcome |
|--------|----------|---------|
| 12923–12924 | AS-REQ (no pre-auth) → `KRB5KDC_ERR_PREAUTH_REQUIRED` | Normal — DC challenges for proof of identity |
| 12934–12935 | AS-REQ (encrypted timestamp) → AS-REP | **TGT issued** |
| 12961–12963 | TGS-REQ → TGS-REP | **LDAP service ticket issued** |

**Why the PTR record is critical:**  
To build the LDAP Kerberos Service Principal Name (`ldap/gary-ad.corp.azure`), ANF takes the DC's IP (`10.10.10.8`) and performs a **reverse DNS lookup**. The PTR record must:
- Exist in a reverse lookup zone on the AD DNS server
- Return the exact FQDN: `gary-ad.corp.azure` (not a short name, not an IPv6 address)

If the PTR record is missing or returns an incorrect value, Kerberos SPN construction fails and the authenticated LDAP bind in Phase 5 cannot proceed.

**Why NTP/clock sync matters:**  
Kerberos rejects tickets with a timestamp difference greater than **5 minutes** from the DC clock (`KRB5KRB_AP_ERR_SKEW`). The DC must sync to a reliable time source (e.g., `time.windows.com`).

---

### Phase 5 — Authenticated LDAP Bind (SASL/Kerberos)

```mermaid
sequenceDiagram
    participant ANF as ANF Node<br/>10.10.1.10
    participant DC as LDAP / DC<br/>10.10.10.8 :389

    rect rgb(185, 158, 220)
        Note over ANF,DC: Phase 5 — Authenticated LDAP Bind (SASL/Kerberos)
        ANF->>DC: bindRequest (SASL GSS-API + Kerberos service ticket)
        DC-->>ANF: bindResponse: success
        ANF->>DC: searchRequest "<ROOT>" baseObject
        DC-->>ANF: searchResEntry [naming contexts]
        ANF->>DC: searchRequest cn=Partitions,CN=Configuration,DC=corp,DC=azure
        DC-->>ANF: searchResEntry CN=CORP partition info
        ANF->>DC: searchRequest CN=Computers,dc=CORP,dc=AZURE (verify OU)
        DC-->>ANF: searchResEntry CN=Computers [exists]
        ANF->>DC: searchRequest dc=CORP,dc=AZURE (check for existing account)
        DC-->>ANF: searchResDone [no pre-existing ANF account]
    end
```

ANF uses the LDAP service ticket from Phase 4 to perform a **SASL GSS-API bind** — a cryptographically authenticated LDAP session.

| Frames | Exchange | Outcome |
|--------|----------|---------|
| 12974–12976 | `bindRequest (SASL)` | `bindResponse: success` |
| 12977–12978 | `searchRequest <ROOT>` | Returns naming contexts and supported controls |
| 12979–12980 | `searchRequest cn=Partitions,CN=Configuration` | Returns `CN=CORP` — confirms domain partition |
| 12981–12982 | `searchRequest CN=Computers,dc=CORP,dc=AZURE` | Confirms the target OU exists and is accessible |
| 12983–12984 | `searchRequest dc=CORP,dc=AZURE` (subtree) | Checks for any pre-existing ANF computer account |

**Why the SASL bind is the key gate:**  
All subsequent AD operations (account creation, modification) run over this authenticated LDAP session. If the SASL bind fails, nothing beyond Phase 4 can proceed.

---

### Phase 6 — Computer Account Creation

```mermaid
sequenceDiagram
    participant ANF as ANF Node<br/>10.10.1.10
    participant DC as LDAP / DC<br/>10.10.10.8 :389

    rect rgb(157, 217, 204)
        Note over ANF,DC: Phase 6 — Computer Account Creation (LDAP :389)
        ANF->>DC: addRequest "cn=ANF-AU-AA30,CN=Computers,dc=CORP,dc=AZURE"
        DC-->>ANF: addResponse: success
        ANF->>DC: searchRequest (verify ANF-AU-AA30 exists)
        DC-->>ANF: searchResEntry CN=ANF-AU-AA30 [confirmed]
    end
```

ANF creates its own computer object in Active Directory using the authenticated LDAP session.

| Frames | Operation | Outcome |
|--------|-----------|---------|
| 12985 | `addRequest "cn=ANF-AU-AA30,CN=Computers,dc=CORP,dc=AZURE"` | — |
| 12986 | `addResponse` | **success** — computer object created |
| 12987–12988 | `searchRequest` (verify) | Returns `CN=ANF-AU-AA30` — confirmed in AD |

The computer name `ANF-AU-AA30` is derived from the `--smb-server-name` prefix in the ANF AD connection combined with a generated suffix. This object represents the ANF SMB server identity in the domain.

**Why the service account needs Create Computer Objects permission:**  
The ANF AD connection username performs the `addRequest`. If the account lacks the right to create computer objects in the target OU, `addResponse` returns `insufficientAccessRights` (LDAP result code 50).

---

### Phase 7 — LSARPC Domain SID Lookup (via SMB2/IPC$)

```mermaid
sequenceDiagram
    participant ANF as ANF Node<br/>10.10.1.10
    participant DC as SMB / DC<br/>10.10.10.8 :445

    rect rgb(213, 191, 152)
        Note over ANF,DC: Phase 7 — LSARPC Domain SID Lookup (SMB :445 / IPC$)
        ANF->>DC: SMB2 Negotiate
        DC-->>ANF: Negotiate Response
        ANF->>DC: Session Setup Request (Kerberos ticket)
        DC-->>ANF: Session Setup Response
        ANF->>DC: Tree Connect \\gary-ad.corp.azure\ipc$
        DC-->>ANF: Tree Connect Response
        ANF->>DC: Create lsarpc + DCE/RPC Bind (LSARPC V0.0)
        DC-->>ANF: Bind_ack (Acceptance)
        ANF->>DC: lsa_OpenPolicy2 request
        DC-->>ANF: lsa_OpenPolicy2 response
        ANF->>DC: lsa_LookupSids2 request (×3 domain SIDs)
        DC-->>ANF: lsa_LookupSids2 responses (SIDs resolved)
    end
```

Immediately after the computer account is created in LDAP, ANF opens a new SMB2 session to the DC's `IPC$` share and binds the **LSARPC** (Local Security Authority RPC) pipe to resolve domain Security Identifiers (SIDs).

| Frames | Protocol | Exchange |
|--------|----------|---------|
| 12994–12995 | SMB2 | Negotiate → Response |
| 12999–13001 | SMB2 | Session Setup (Kerberos) → Response |
| 13002–13003 | SMB2 | Tree Connect `\\gary-ad.corp.azure\ipc$` → Response |
| 13004–13009 | SMB2 + LSARPC | Create `lsarpc` → DCE/RPC Bind (LSARPC V0.0) → Bind_ack |
| 13010–13013 | LSARPC | `lsa_OpenPolicy2` request → response (policy handle obtained) |
| 13014–13025 | LSARPC | Three rounds of `lsa_LookupSids2` request/response |

**Why domain SID resolution is needed:**  
ANF must know the domain's SID (`S-1-5-21-...`) to:
- Set correct ACL entries on the SMB share using well-known domain SIDs
- Map NFS UIDs/GIDs to AD SIDs for dual-protocol volumes
- Construct the machine account's full SID (`domain SID + RID`)

`lsa_LookupSids2` converts binary SIDs to their domain+account-name representations. ANF performs three separate lookups covering the domain SID and well-known SIDs (e.g., BUILTIN, NT AUTHORITY).

**Why port 445 must be open for LSARPC:**  
LSARPC has no dedicated port — it is multiplexed through SMB2/IPC$ on TCP 445. If port 445 is blocked between the ANF subnet and the DC subnet, LSARPC fails silently: LDAP can succeed but the volume will fail during domain SID resolution.

---

### Phase 8 — KPASSWD — Set Machine Account Password

```mermaid
sequenceDiagram
    participant ANF as ANF Node<br/>10.10.1.10
    participant DC as KDC / KPASSWD<br/>10.10.10.8 :88 / :464

    rect rgb(218, 187, 151)
        Note over ANF,DC: Phase 8 — Set Machine Account Password (KPASSWD :464)
        ANF->>DC: TGS-REQ (ticket for kadmin/changepw) :88
        DC-->>ANF: TGS-REP — KPASSWD service ticket
        ANF->>DC: KPASSWD Request (set machine account password) :464
        DC-->>ANF: KPASSWD Reply: success
    end
```

With the computer account created and domain SIDs resolved, ANF sets the machine account password using the **Kerberos Password Change (KPASSWD)** protocol.

| Frames | Protocol | Port | Exchange |
|--------|----------|------|---------|
| 13032–13035 | KRB5 | 88 | TGS-REQ/REP — obtain ticket for `kadmin/changepw` |
| 13075–13082 | KPASSWD | 464 | KPASSWD Request → Reply: success |

**Why port 464 must be open:**  
KPASSWD is a distinct protocol (RFC 3244) on TCP/UDP 464. If the ANF delegated subnet is blocked from reaching the DC on port 464, the computer account is created (Phase 6) but its password is never set, leaving it in a broken, unusable state. Volume creation fails at this point.

---

### Phase 9 — Finalise Computer Account Attributes

```mermaid
sequenceDiagram
    participant ANF as ANF Node<br/>10.10.1.10
    participant DC as LDAP / DC<br/>10.10.10.8 :389

    rect rgb(147, 175, 215)
        Note over ANF,DC: Phase 9 — Finalise Computer Account Attributes (LDAP :389)
        ANF->>DC: searchRequest CN=ANF-AU-AA30 (get current attributes)
        DC-->>ANF: searchResEntry CN=ANF-AU-AA30
        ANF->>DC: modifyRequest CN=ANF-AU-AA30 (SPNs, userAccountControl, dnsHostName)
        DC-->>ANF: modifyResponse: success
        ANF->>DC: unbindRequest (close authenticated session)
    end
```

After the password is set, ANF writes the remaining attributes to the computer account over the still-open authenticated LDAP session.

| Frames | Operation | Outcome |
|--------|-----------|---------|
| 13088–13089 | `searchRequest CN=ANF-AU-AA30` | Retrieves current object state |
| 13090–13091 | `modifyRequest CN=ANF-AU-AA30` | **success** — SPNs, `userAccountControl`, `dnsHostName` written |
| 13199 | `unbindRequest` | Closes the authenticated LDAP session |

Attributes written include:
- `servicePrincipalName`: Multiple SPNs for Kerberos SPN validation (e.g., `cifs/ANF-AU-AA30.corp.azure`, `HOST/ANF-AU-AA30`)
- `dnsHostName`: `ANF-AU-AA30.corp.azure`
- `userAccountControl`: Flags marking it as a workstation/server account

---

### Phase 10 — Netlogon Secure Channel Establishment

```mermaid
sequenceDiagram
    participant ANF as ANF Node<br/>10.10.1.10
    participant DC as KDC + SMB / DC<br/>10.10.10.8

    rect rgb(209, 143, 209)
        Note over ANF,DC: Phase 10 — Netlogon Secure Channel (KDC :88 + SMB :445)
        ANF->>DC: AS-REQ → AS-REP (machine account TGT) :88
        ANF->>DC: TGS-REQ → TGS-REP (ticket for cifs/GARY-AD) :88
        ANF->>DC: SMB2 Negotiate + Session Setup (machine identity) :445
        DC-->>ANF: Session Setup Response
        ANF->>DC: Tree Connect \\GARY-AD\ipc$ + Create NETLOGON
        ANF->>DC: DCE/RPC Bind RPC_NETLOGON V1.0
        DC-->>ANF: Bind_ack (Acceptance)
        ANF->>DC: NetrServerReqChallenge (ANF-AU-AA30)
        DC-->>ANF: NetrServerReqChallenge response (server challenge)
        ANF->>DC: NetrServerAuthenticate2 (credential + flags)
        DC-->>ANF: NetrServerAuthenticate2 — secure channel established
    end
```

After the computer account is fully provisioned, ANF establishes a **Netlogon Secure Channel** — the challenge-response handshake that formally admits the ANF node as a trusted domain member.

| Frames | Protocol | Exchange |
|--------|----------|---------|
| 13097–13098 | SMB2 | Negotiate (new TCP session, port 27277→445) |
| 13107–13119 | KRB5 | AS-REQ → PREAUTH → AS-REQ → AS-REP (machine account TGT) |
| 13132–13134 | KRB5 | TGS-REQ → TGS-REP (service ticket for `cifs/GARY-AD`) |
| 13141–13145 | SMB2 | Session Setup + Tree Connect `\\GARY-AD\ipc$` |
| 13146–13151 | SMB2 + DCERPC | Create `NETLOGON` pipe → DCE/RPC Bind (RPC_NETLOGON V1.0) → Bind_ack |
| 13152–13155 | RPC_NETLOGON | `NetrServerReqChallenge` (ANF-AU-AA30) → server challenge response |
| 13156–13159 | RPC_NETLOGON | `NetrServerAuthenticate2` (credential + flags) → secure channel established |
| 13160–13167 | SMB2 + DCERPC | Close `NETLOGON` → reopen `netlogon` (second bind — lowercase) |

**What the secure channel proves:**  
`NetrServerReqChallenge` and `NetrServerAuthenticate2` implement a challenge-response mechanism where the DC verifies ANF knows the machine account password set in Phase 8. A shared session key is derived from the password hash. This is the fundamental mechanism Windows uses to admit any computer to the domain — it runs over the same `NETLOGON` named pipe that all domain members use.

The second lowercase `netlogon` bind establishes the encrypted/signed channel for ongoing domain authentication operations.

**Why this phase matters:**  
If the Netlogon secure channel fails (e.g., password mismatch, name collision with a stale account), the ANF computer account exists in AD but ANF is not a trusted domain member. Client authentication against the SMB share will fail.

---

### Phase 11 — SMB Volume Kerberos Identity

```mermaid
sequenceDiagram
    participant ANF as ANF Node<br/>10.10.1.10
    participant DC as KDC + SMB / DC<br/>10.10.10.8

    rect rgb(151, 222, 151)
        Note over ANF,DC: Phase 11 — SMB Volume Kerberos Identity (KDC :88 + SMB :445)
        ANF->>DC: AS-REQ (no pre-auth) — volume identity :88
        DC-->>ANF: KRB5KDC_ERR_PREAUTH_REQUIRED (normal)
        ANF->>DC: AS-REQ (with pre-auth)
        DC-->>ANF: AS-REP — TGT for volume identity
        ANF->>DC: TGS-REQ (SMB service ticket)
        DC-->>ANF: TGS-REP — SMB service ticket issued
        ANF->>DC: SMB2 Session Setup (volume identity) :445
        DC-->>ANF: SMB2 Session Setup Response
    end
```

ANF obtains an independent Kerberos identity for the SMB volume itself. This identity is what clients authenticate against when they mount the share.

- Multiple rounds of: AS-REQ → PREAUTH error → AS-REQ (with pre-auth) → AS-REP (TGT)
- TGS-REQ → TGS-REP (SMB service ticket)
- SMB2 Session Setup to DC using the volume's own Kerberos identity

(Frames 13220+ / 13285+ / 13350+ — multiple parallel credential exchanges)

---

### Phase 12 — Ongoing Site-Aware DC Keepalive

```mermaid
sequenceDiagram
    participant ANF as ANF Node<br/>10.10.1.10
    participant DC as DNS / DC<br/>10.10.10.8 :53

    rect rgb(212, 151, 151)
        Note over ANF,DC: Phase 12 — Ongoing Site-Aware DC Keepalive (DNS :53)
        loop Per SMB node refresh
            ANF->>DC: SRV _ldap._tcp.Default-First-Site-Name._sites.CORP.AZURE?
            DC-->>ANF: gary-ad.corp.azure:389
            ANF->>DC: SRV _ldap._tcp.Default-First-Site-Name._sites.dc._msdcs.CORP.AZURE?
            DC-->>ANF: gary-ad.corp.azure:389
        end
    end
```

After the volume is provisioned, each ANF SMB node continues refreshing site-local LDAP and Kerberos SRV records in parallel. The repeated DNS SRV queries, multiple SMB2 Negotiate sessions, and multiple Kerberos AS-REQ/TGS-REQ cycles at frames 13194+ are normal keepalive behaviour — **not errors**.

These parallel connections establish the SMB server's ongoing domain connectivity, including:
- Simultaneous SRV lookups for both `_ldap._tcp` and `_ldap._tcp..dc._msdcs` variants
- Multiple SMB2 Negotiate sessions probing different network paths
- Multiple Kerberos exchanges for each parallel SMB node ANF provisions

---

## Why Each Infrastructure Component Is Critical

### DNS Forward A Record (`gary-ad.corp.azure → 10.10.10.8`)

Used in: Phase 1 (DC IP resolution), Phase 4 (SPN construction)

- SRV discovery returns hostnames — an A record is required to obtain the IP
- Kerberos SPN (`ldap/gary-ad.corp.azure`) is built from the name returned by this record
- Must match the DC's actual `DNSHostName` attribute in AD

### DNS Reverse PTR Record (`10.10.10.8 → gary-ad.corp.azure`)

Used in: Phase 4 (SPN construction from DC IP)

- ANF verifies the DC's IP against its Kerberos identity using a PTR lookup
- The returned value must be the full FQDN — not a short name, not an IPv6 address
- Must resolve from the DNS server configured in the ANF AD connection

**Common mistakes:**

| Reverse zone | PTR hostname (wrong) | PTR hostname (correct) |
|--------------|----------------------|------------------------|
| `10.10.in-addr.arpa` | `8` | `8.10` |
| `10.10.10.in-addr.arpa` | `8.10` | `8` |
| Any zone | `gary-ad` (short name) | `gary-ad.corp.azure` |
| Any zone | `::1` or `fe80::...` | `gary-ad.corp.azure` |

### AD Sites and Services — Site + Subnet Mapping

Used in: Phase 3 (site-scoped SRV discovery)

- ANF queries site-specific SRV records using the AD site name
- If the ANF subnet (`10.10.1.0/24`) is not mapped to a site, the site-scoped SRV query returns NXDOMAIN
- All Azure subnets should be mapped to the same site as the DC

### DNS Servers Field in ANF AD Connection

- The ANF AD connection has its own DNS Servers field — independent of VNet-level DNS
- Must be set to the AD DC IP (`10.10.10.8`), not Azure-provided DNS (`168.63.129.16`)
- Azure DNS has no reverse lookup zone for private ranges → PTR lookup fails

---

## Required Port Checklist (ANF Subnet → DC Subnet)

| Protocol | Port | Phase | Purpose |
|----------|------|-------|---------|
| UDP/TCP | 53 | 1, 3, 12 | DNS |
| TCP | 88 | 4, 8, 10, 11 | Kerberos KDC |
| TCP | 389 | 2, 5, 6, 9 | LDAP |
| TCP | 445 | 7, 10, 11 | SMB (LSARPC domain SID lookup, Netlogon, volume identity) |
| TCP | 464 | 8 | Kerberos password change (KPASSWD) |
| TCP | 636 | — | LDAPS (if `--ldap-over-tls true`) |
| TCP | 3268 | — | Global Catalog (multi-domain environments) |

---


**Failure pattern reference:**

| Packet observed | Root cause |
|-----------------|------------|
| LDAP `bindResponse` result code `8` (strongerAuthRequired) | `LdapEnforceChannelBinding = 2` on DC (default on WS2025) |
| LDAP `bindResponse` result code `49` (invalidCredentials) | Wrong password in ANF AD connection |
| KRB5 error `KRB5KRB_AP_ERR_SKEW` | Clock skew > 5 min between ANF and DC |
| KRB5 error `KRB5KDC_ERR_C_PRINCIPAL_UNKNOWN` | Service account username not found in AD |
| DNS PTR query → NXDOMAIN | Reverse lookup zone missing or incorrect zone name |
| DNS PTR returns `::1` or `fe80::` | IPv6 AAAA record in forward zone — disable IPv6 on DC |
| TCP SYN to port 389 → no response or RST | NSG blocking ANF subnet → DC subnet |
| LDAP `addResponse` result code `50` (insufficientAccessRights) | Service account lacks Create Computer Objects permission in target OU |
| LDAP `addResponse` result code `68` (entryAlreadyExists) | Stale computer account from previous attempt — delete `CN=ANF-*` objects in AD |

---

## Windows 11 Client SMB Mount Flow

**Captured environment:**  
**Client:** Windows 11 `10.10.10.9` (source port `64125`)  
**ANF SMB volume:** `10.10.1.10` port `445`, share name `smb1`  
**AD / KDC:** `10.10.10.8` (gary-ad.corp.azure)  
**ANF computer account:** `ANF-AU-AA30`

> **Note on capture scope:** The packet capture (`smb mount export text.txt`) begins at SMB2 MessageId 548 — the TCP session and authentication were already established before the capture started. The pre-mount phases below are documented from standard Windows SMB behaviour. The post-mount phases are directly from the capture.

![Windows 11 Map Network Drive — connecting to the ANF SMB share](win11-map-network-drive.jpg)

---

### End-to-End Client Mount Flow Overview

![Wireshark packet capture of Windows 11 client SMB mount and file access](windows11-smb-mount.jpg)

The 8 phases below (2 pre-capture + 6 captured) cover the full Windows 11 client SMB mount flow. Each phase is expanded with full packet detail in the [Phase-by-Phase Explanation (Client Mount)](#phase-by-phase-explanation-client-mount) section below.

```mermaid
sequenceDiagram
    participant WIN as Windows 11 Client<br/>10.10.10.9
    participant DC as AD / KDC<br/>10.10.10.8
    participant ANF as ANF SMB Volume<br/>10.10.1.10 :445

    rect rgb(144, 176, 216)
        Note over WIN,DC: Pre-Mount Phase A — Kerberos Authentication (not in capture)
        WIN->>DC: DNS + AS-REQ/TGS-REQ for cifs/ANF-AU-AA30
        DC-->>WIN: CIFS service ticket issued
    end

    rect rgb(150, 221, 193)
        Note over WIN,ANF: Pre-Mount Phase B — SMB Session Establishment (not in capture)
        WIN->>ANF: TCP connect + SMB2 Negotiate
        ANF-->>WIN: Negotiate Response (SMB 3.x)
        WIN->>ANF: Session Setup (Kerberos) + Tree Connect smb1
        ANF-->>WIN: Share mounted
    end

    rect rgb(211, 170, 143)
        Note over WIN,ANF: Post-Mount Phase 1 — Explorer Initial Probes (frames 1–8)
        WIN->>ANF: Create Desktop.ini / AutoRun.inf
        ANF-->>WIN: STATUS_OBJECT_NAME_NOT_FOUND (expected)
    end

    rect rgb(207, 224, 156)
        Note over WIN,ANF: Post-Mount Phase 2 — Share Root Listing (frames 9–22)
        WIN->>ANF: Create + Notify + Find *
        ANF-->>WIN: 5 entries + STATUS_NO_MORE_FILES
    end

    rect rgb(154, 200, 215)
        Note over WIN,ANF: Post-Mount Phase 3 — Capacity Query (frames 23–26)
        WIN->>ANF: GetInfo FileFsFullSizeInformation ×2
        ANF-->>WIN: Total / free capacity
    end

    rect rgb(214, 151, 198)
        Note over WIN,ANF: Post-Mount Phase 4 — Subfolder Browse + Change Notification (frames 27–49)
        WIN->>ANF: Create project1 + Find * + Notify
        ANF-->>WIN: 4 entries + STATUS_PENDING
        WIN->>ANF: Cancel
        ANF-->>WIN: STATUS_CANCELLED
    end

    rect rgb(166, 223, 223)
        Note over WIN,ANF: Post-Mount Phase 5 — IPC$ / SRVSVC / DFS Check (frames 56–71)
        WIN->>ANF: Tree Connect IPC$ + FSCTL_DFS_GET_REFERRALS
        ANF-->>WIN: STATUS_NOT_FOUND (DFS not configured — expected)
        WIN->>ANF: SRVSVC bind + NetShareGetInfo (smb1)
        ANF-->>WIN: Share properties returned
    end

    rect rgb(220, 204, 158)
        Note over WIN,ANF: Post-Mount Phase 6 — NTFS Object ID Probes (frames 72–87)
        WIN->>ANF: Create file/folder + FSCTL_CREATE_OR_GET_OBJECT_ID
        ANF-->>WIN: STATUS_INVALID_DEVICE_REQUEST (expected)
    end
```

---

### Phase-by-Phase Explanation (Client Mount)

#### Pre-Mount Phase A — Kerberos Authentication *(not in capture)*

```mermaid
sequenceDiagram
    participant WIN as Windows 11 Client<br/>10.10.10.9
    participant DC as AD / KDC<br/>10.10.10.8

    rect rgb(144, 176, 216)
        Note over WIN,DC: Pre-Mount Phase A — Kerberos Authentication (not in capture)
        WIN->>DC: DNS A? ANF-AU-AA30.corp.azure
        DC-->>WIN: 10.10.1.10
        WIN->>DC: AS-REQ (no pre-auth)
        DC-->>WIN: KRB5KDC_ERR_PREAUTH_REQUIRED
        WIN->>DC: AS-REQ (with pre-auth)
        DC-->>WIN: AS-REP — TGT issued
        WIN->>DC: TGS-REQ (ticket for cifs/ANF-AU-AA30.corp.azure)
        DC-->>WIN: TGS-REP — CIFS service ticket issued
    end
```

Before opening a single SMB packet to the ANF volume, the Windows client must obtain a **Kerberos service ticket** for the ANF SMB server identity.

1. **DNS A lookup** — client resolves `ANF-AU-AA30.corp.azure` to `10.10.1.10`
2. **AS exchange** — client presents its identity to the KDC at `10.10.10.8:88` and receives a TGT
3. **TGS exchange** — client requests a service ticket for `cifs/ANF-AU-AA30.corp.azure` — the SPN registered by ANF in Phase 11 of the volume creation flow

The service ticket is encrypted with the ANF machine account's key. ANF decrypts it to verify the client's identity without contacting the DC during the session.

---

#### Pre-Mount Phase B — SMB Session Establishment *(not in capture)*

```mermaid
sequenceDiagram
    participant WIN as Windows 11 Client<br/>10.10.10.9
    participant ANF as ANF SMB Volume<br/>10.10.1.10 :445

    rect rgb(150, 221, 193)
        Note over WIN,ANF: Pre-Mount Phase B — SMB Session Establishment (not in capture)
        WIN->>ANF: TCP SYN → port 445
        ANF-->>WIN: TCP SYN-ACK
        WIN->>ANF: SMB2 Negotiate Request (dialects: 2.0.2, 2.1, 3.0, 3.0.2, 3.1.1)
        ANF-->>WIN: SMB2 Negotiate Response (selected: SMB 3.x)
        WIN->>ANF: SMB2 Session Setup Request (Kerberos CIFS ticket)
        ANF-->>WIN: SMB2 Session Setup Response — session established
        WIN->>ANF: SMB2 Tree Connect \\10.10.1.10\smb1
        ANF-->>WIN: Tree Connect Response — share mounted
    end
```

| Step | SMB Exchange | Outcome |
|------|-------------|---------|
| TCP connect | 3-way handshake to port 445 | TCP session established |
| Negotiate | Client offers dialect list (2.0.2 → 3.1.1) | ANF selects SMB 3.x |
| Session Setup | Client presents Kerberos CIFS service ticket | Session authenticated |
| Tree Connect | Client requests `\\10.10.1.10\smb1` | Share handle assigned |

The `Session Setup` is the security gate — the Kerberos ticket is verified here. If it fails, all subsequent SMB operations are rejected with `STATUS_ACCESS_DENIED`.

**Why the capture starts at MessageId 548:**  
Windows keeps SMB sessions alive after the first mount. This capture was started after the session was already active, so the authentication frames are absent.

---

#### Post-Mount Phase 1 — Explorer Initial Probes (frames 1–8)

```mermaid
sequenceDiagram
    participant WIN as Windows 11 Client<br/>10.10.10.9
    participant ANF as ANF SMB Volume<br/>10.10.1.10 :445

    rect rgb(211, 170, 143)
        Note over WIN,ANF: Post-Mount Phase 1 — Explorer Initial Probes (frames 1–8)
        WIN->>ANF: Create Request — Desktop.ini
        ANF-->>WIN: STATUS_OBJECT_NAME_NOT_FOUND
        WIN->>ANF: Create Request — AutoRun.inf
        ANF-->>WIN: STATUS_OBJECT_NAME_NOT_FOUND
        Note over WIN,ANF: Normal — Explorer probes for shell metadata files
    end
```

| Frame | File | Result | Purpose |
|-------|------|--------|---------|
| 1–3 | `Desktop.ini` | `STATUS_OBJECT_NAME_NOT_FOUND` | Custom folder appearance metadata |
| 6–8 | `AutoRun.inf` | `STATUS_OBJECT_NAME_NOT_FOUND` | AutoPlay detection |

Both `NOT_FOUND` responses are **expected and normal**. Windows Explorer sends these probes every time a folder is opened. The absence of these files is not an error.

---

#### Post-Mount Phase 2 — Share Root Directory Listing (frames 9–22)

```mermaid
sequenceDiagram
    participant WIN as Windows 11 Client<br/>10.10.10.9
    participant ANF as ANF SMB Volume<br/>10.10.1.10 :445

    rect rgb(207, 224, 156)
        Note over WIN,ANF: Post-Mount Phase 2 — Share Root Listing (frames 9–22)
        WIN->>ANF: Create Request — share root
        ANF-->>WIN: Create Response
        WIN->>ANF: Notify Request (change watch)
        ANF-->>WIN: STATUS_PENDING (watch queued)
        WIN->>ANF: Find SMB2_FIND_ID_BOTH_DIRECTORY_INFO Pattern: *
        ANF-->>WIN: Find Response — 5 entries returned
        ANF-->>WIN: STATUS_NO_MORE_FILES
    end
```

Windows issues multiple batched requests in a single TCP segment:

- **Create** (share root) + **Notify** (change watch registration) → compound request
- **Find** `SMB2_FIND_ID_BOTH_DIRECTORY_INFO, Pattern: *` → two back-to-back find requests
- Server returns **5 directory entries** in a single response, then `STATUS_NO_MORE_FILES`

The `Notify` registration (`STATUS_PENDING`) tells the server to push change notifications to the client without requiring polling — this is Windows Explorer's live folder refresh mechanism.

---

#### Post-Mount Phase 3 — Capacity Query (frames 23–26)

```mermaid
sequenceDiagram
    participant WIN as Windows 11 Client<br/>10.10.10.9
    participant ANF as ANF SMB Volume<br/>10.10.1.10 :445

    rect rgb(154, 200, 215)
        Note over WIN,ANF: Post-Mount Phase 3 — Capacity Query (frames 23–26)
        WIN->>ANF: GetInfo FS_INFO/FileFsFullSizeInformation
        ANF-->>WIN: GetInfo Response (total/free sectors)
        WIN->>ANF: GetInfo FS_INFO/FileFsFullSizeInformation
        ANF-->>WIN: GetInfo Response
        Note over WIN,ANF: Queried twice — Explorer sidebar + status bar
    end
```

Two `GetInfo FileFsFullSizeInformation` requests are issued 70ms apart. Windows Explorer reads share capacity:
- Once for the **drive properties** display (total/free space in Explorer sidebar)
- Once for the **status bar** at the bottom of the Explorer window

ANF responds with `TotalAllocationUnits`, `CallerAvailableAllocationUnits`, `SectorsPerAllocationUnit`, and `BytesPerSector`.

---

#### Post-Mount Phase 4 — Subfolder Browse + Change Notification (frames 27–49)

```mermaid
sequenceDiagram
    participant WIN as Windows 11 Client<br/>10.10.10.9
    participant ANF as ANF SMB Volume<br/>10.10.1.10 :445

    rect rgb(214, 151, 198)
        Note over WIN,ANF: Post-Mount Phase 4 — Subfolder Browse + Change Notification (frames 27–49)
        WIN->>ANF: Create Request — project1
        ANF-->>WIN: Create Response — directory handle
        WIN->>ANF: Find SMB2_FIND_ID_BOTH_DIRECTORY_INFO Pattern: * (project1)
        ANF-->>WIN: Find Response — 4 entries + STATUS_NO_MORE_FILES
        WIN->>ANF: Notify Request — watch project1 for changes
        ANF-->>WIN: STATUS_PENDING (watch registered)
        WIN->>ANF: Cancel Request (user navigated away)
        ANF-->>WIN: STATUS_CANCELLED
    end
```

When the user clicks into `project1`:

1. **Create** `project1` → directory handle issued
2. **Find** `*` → 4 entries returned + `STATUS_NO_MORE_FILES`
3. **Notify** `project1` → `STATUS_PENDING` (watch registered)
4. **Cancel** → `STATUS_CANCELLED` (user navigated away before any change was detected)

The Cancel/STATUS_CANCELLED cycle at frames 42–47 is **normal** — it happens every time Explorer leaves a directory before the Notify fires.

---

#### Post-Mount Phase 5 — IPC$ Sub-connection, DFS Check, and SRVSVC (frames 56–71)

```mermaid
sequenceDiagram
    participant WIN as Windows 11 Client<br/>10.10.10.9
    participant ANF as ANF SMB Volume<br/>10.10.1.10 :445

    rect rgb(166, 223, 223)
        Note over WIN,ANF: Post-Mount Phase 5 — IPC$ / SRVSVC / DFS Check (frames 56–71)
        WIN->>ANF: Tree Connect \\10.10.1.10\IPC$
        ANF-->>WIN: Tree Connect Response
        WIN->>ANF: Ioctl FSCTL_DFS_GET_REFERRALS (\10.10.1.10\smb1)
        ANF-->>WIN: STATUS_NOT_FOUND (DFS not configured — expected)
        WIN->>ANF: Create srvsvc + DCE/RPC Bind (SRVSVC V3.0)
        ANF-->>WIN: Bind_ack (Acceptance — 32-bit NDR)
        WIN->>ANF: NetShareGetInfo request (smb1)
        ANF-->>WIN: NetShareGetInfo response (share properties)
        WIN->>ANF: Close srvsvc
    end
```

This phase reveals important details about the Windows SMB client's interaction with ANF:

| Frame | Operation | Result | Explanation |
|-------|-----------|--------|-------------|
| 56–57 | Tree Connect `\\10.10.1.10\IPC$` | Success | Client opens admin pipe for RPC calls |
| 58–59 | `FSCTL_DFS_GET_REFERRALS` `\10.10.1.10\smb1` | `STATUS_NOT_FOUND` | Client checks if share is a DFS target — it is not |
| 60–65 | Create `srvsvc` + DCE/RPC Bind SRVSVC V3.0 | Bind_ack (32-bit NDR accepted) | Client opens the Server Service RPC pipe |
| 68–69 | `NetShareGetInfo` (smb1) | Response OK | Client queries share permissions, comment, and flags |
| 70–71 | Close `srvsvc` | Success | Pipe closed after query |

**Why `STATUS_NOT_FOUND` for DFS is expected:**  
ANF SMB volumes are not DFS targets by default. The client always probes for DFS referrals when connecting to a share — a `NOT_FOUND` response simply means DFS is not involved, and the client proceeds with the direct path.

**Why SRVSVC matters:**  
`NetShareGetInfo` populates the share permissions and description in Explorer's "Map Network Drive" and "Properties" dialogs. It runs over `IPC$` using DCE/RPC — which is why port 445 must be open for the client-to-ANF path, not just the ANF-to-DC path.

---

#### Post-Mount Phase 6 — File Access + FSCTL Object ID Probes (frames 72–87)

```mermaid
sequenceDiagram
    participant WIN as Windows 11 Client<br/>10.10.10.9
    participant ANF as ANF SMB Volume<br/>10.10.1.10 :445

    rect rgb(220, 204, 158)
        Note over WIN,ANF: Post-Mount Phase 6 — NTFS Object ID Probes (frames 72–87)
        WIN->>ANF: Create project1\file.txt.txt + FSCTL_CREATE_OR_GET_OBJECT_ID
        ANF-->>WIN: Create OK + STATUS_INVALID_DEVICE_REQUEST
        WIN->>ANF: Create project1\file.txt.txt + FSCTL_CREATE_OR_GET_OBJECT_ID
        ANF-->>WIN: Create OK + STATUS_INVALID_DEVICE_REQUEST
        WIN->>ANF: Create project1 (folder) + FSCTL_CREATE_OR_GET_OBJECT_ID
        ANF-->>WIN: Create OK + STATUS_INVALID_DEVICE_REQUEST
        WIN->>ANF: Create project1 (folder) + FSCTL_CREATE_OR_GET_OBJECT_ID
        ANF-->>WIN: Create OK + STATUS_INVALID_DEVICE_REQUEST
        Note over WIN,ANF: ANF does not support NTFS Object IDs — expected, not an error
    end
```

When the client tries to open a file or folder, Windows automatically issues `FSCTL_CREATE_OR_GET_OBJECT_ID` alongside the Create request:

| Frame | File | FSCTL Result | Explanation |
|-------|------|-------------|-------------|
| 72–75 | `project1\file.txt.txt` | `STATUS_INVALID_DEVICE_REQUEST` | ANF does not support NTFS Object IDs |
| 76–79 | `project1\file.txt.txt` | `STATUS_INVALID_DEVICE_REQUEST` | Second probe for same file |
| 80–83 | `project1` (folder) | `STATUS_INVALID_DEVICE_REQUEST` | Folder also probed |
| 84–87 | `project1` (folder) | `STATUS_INVALID_DEVICE_REQUEST` | Repeated probe |

**Why `STATUS_INVALID_DEVICE_REQUEST` is expected:**  
NTFS Object IDs are a Windows-specific filesystem feature used for tracking files across moves/renames (used by Volume Shadow Copy, DFS, and some backup tools). ANF's SMB implementation does not support this IOCTL — it returns `STATUS_INVALID_DEVICE_REQUEST`, which Windows silently ignores and continues with normal file access. This is **not an error condition** and does not affect file access.

---

### Client Mount — Key Observations

| Observation | Meaning |
|-------------|---------|
| MessageId starts at 548 — no Negotiate/Session Setup in capture | Session was pre-established; authentication happened before the capture started |
| `Desktop.ini` and `AutoRun.inf` → `NOT_FOUND` | Normal Explorer shell probes — safe to ignore |
| `FSCTL_DFS_GET_REFERRALS` → `STATUS_NOT_FOUND` | ANF share is not a DFS target — expected |
| `FSCTL_CREATE_OR_GET_OBJECT_ID` → `STATUS_INVALID_DEVICE_REQUEST` | ANF does not support NTFS Object IDs — expected, no impact on file access |
| `NetShareGetInfo` via SRVSVC | Client reads share metadata — requires IPC$ access on port 445 from client subnet to ANF subnet |
| `STATUS_PENDING` on Notify + `STATUS_CANCELLED` after Cancel | Normal Explorer change notification lifecycle — watch registered then cancelled on navigation |

### Wireshark Display Filter for Client Mount Capture

To capture a full client mount from scratch (including auth), run this filter on the DC or a VM in the DC's subnet:

```
(ip.addr == 10.10.10.9 and (ip.addr == 10.10.10.8 or ip.addr == 10.10.1.10)) and (kerberos or dns or smb2 or dcerpc)
```
