# Key Design Principles & Built-in Services

**Why CaptainPBX is built for modern telecommunications operators, MSPs, and open-source developers.**

CaptainPBX replaces legacy PBX systems—which were traditionally built as web interface wrappers around configuration files—with a **modular, API-first control plane** designed for high security, multi-tenancy, and automation.

---

## 1. Architectural Philosophy: The Four Pillars

```mermaid
flowchart TB
  subgraph Pillar1 [1. Security by Default]
    P1A[Non-Root Privilege Split]
    P1B[Captain Shield Host Firewall]
    P1C[Captain Auth MFA & Trust TLS]
  end

  subgraph Pillar2 [2. Fail-Closed Tenancy]
    P2A[Shared Schema Isolation]
    P2B[Custom FQDN SIP Domains]
    P2C[Tenant-Scoped JWTs]
  end

  subgraph Pillar3 [3. API-First Control Plane]
    P3A[Versioned /api/v1 REST]
    P3B[OpenAPI Documentation]
    P3C[Voice AI & Automation Ready]
  end

  subgraph Pillar4 [4. Modern Built-In Services]
    P4A[27 Queue Report Categories]
    P4B[Visual Canvas IVR Builder]
    P4C[Redacted Immutable Audit]
  end
```

---

## 2. Pillar 1: Built-in Security Architecture

### A. Captain Shield (Host Protection)
* **Default Deny**: Administrative ports (443, 22) and SIP signaling (5060/5061) are closed by default until explicitly allowlisted by IP, CIDR, or FQDN.
* **Intrusion Detection System (IDS)**: Automatically bans IPs attempting brute-force logins across HTTP, SIP REGISTER, or SSH.
* **GeoIP & Threat Feeds**: Filter incoming traffic by country (DB-IP Lite) and ingest community threat intelligence (APIBAN).
* **WAN Hiding**: Internal infrastructure ports (AMI 5038, MariaDB 3306, Redis 6379) are strictly bound to internal interfaces and never exposed to the WAN.

```mermaid
flowchart LR
  WAN[Public WAN Traffic] --> Shield{Captain Shield nftables}
  Shield -->|Allowlisted IPs| Nginx[Port 443: Web Admin]
  Shield -->|Allowlisted IPs| SIP[Port 5060: SIP Signaling]
  Shield -->|Dynamic UDP| RTP[Media RTP Stream]
  Shield -.-x|BLOCKED| AMI[Port 5038: AMI]
  Shield -.-x|BLOCKED| SQL[Port 3306: MariaDB]
```

### B. Captain Auth (Multi-Factor Authentication)
* Supports **TOTP MFA** and single-use backup recovery codes for web logins.
* Enforcement policy can be set per tenant: **Off**, **Optional**, or **Required**.

### C. Captain Trust (Automated TLS Management)
* Manages local certificates, uploaded PEM files, and automated **ACME HTTP-01 renewals** (Let's Encrypt).
* Automatically provisions and deploys certificates to Nginx and PJSIP without manual shell administration.

### D. Captain Audit (Security Trail)
* Intercepts all database modifications at the repository layer.
* Automatically redacts passwords, SIP secrets, and API tokens before persisting audit entries.

### E. Captain Bridge (OS Service Management)
* Exposes safe operating system management (timezones, network configuration, updates) via allowlisted JSON verbs sent to `captain-system-agent`.

---

## 3. Pillar 2: Fail-Closed Multi-Tenancy

CaptainPBX allows multi-tenant operation on a single shared media engine without risking cross-tenant data leaks.

```mermaid
flowchart TB
  subgraph PlatformAdmin [Platform Administration]
    PA[Platform Admin User]
  end

  subgraph TenantA [Tenant: Acme Corp]
    TA[Acme Admin] --> ExtA[Extensions: 1001, 1002]
    TA --> DomA[SIP Domain: pbx.acme.com]
  end

  subgraph TenantB [Tenant: Globex Corp]
    TB[Globex Admin] --> ExtB[Extensions: 1001, 1002]
    TB --> DomB[SIP Domain: pbx.globex.com]
  end

  PA -->|Manages Platform| TenantA
  PA -->|Manages Platform| TenantB
  ExtA -.->|Isolated Data Boundary| ExtB
```

* **Data Isolation**: `tenant_id` from client browsers is ignored. The active tenant is resolved server-side during authentication.
* **Overlapping Namespaces**: Extensions (e.g., `1001`) are scoped to individual tenants, allowing distinct tenants to share identical extension numbering schemes.
* **SIP Domains**: Each tenant registers endpoints using its dedicated `sip_domain`.

---

## 4. Pillar 3: Modular, Developer-First Architecture

```mermaid
flowchart LR
  CRM[CRM & ERP Systems] -->|JWT Auth| API["REST API (/api/v1)"]
  AI[Voice AI / Automated Agents] -->|AMI Events & REST| API
  WebUI[Admin / User SPA] -->|Session / CSRF| API
  API --> Engine[Captain Core Platform]
```

* **Unified API Catalog**: Every action available in the UI is backed by versioned REST endpoints (`/api/v1/`) documented with OpenAPI specifications.
* **Automation Ready**: Integrators can programmatically manage extensions, routes, and call handling, making CaptainPBX a solid base for **Voice AI agents, CRM call automation, and custom PBX workflows**.
* **Decoupled Frontend**: Built as a React Single Page Application (SPA) communicating exclusively via JSON APIs.

---

## 5. Pillar 4: Built-in Advanced Capabilities

### A. Queue Analytics & Reporting
* Features **27 distinct queue report categories**, including operational metrics, SLA calculations, ASA (Average Speed of Answer) trends, agent handle times, and heatmaps.
* Filters out short hangups (e.g., under 5 seconds) to ensure accurate SLA tracking.
* Offers instant export to **CSV, JSON, and PDF**.

```mermaid
flowchart LR
  Events[Asterisk Queue Events] --> Stream[Redis Queue Facts Stream]
  Stream --> Processing[Queue Analytics Engine]
  Processing --> Reports[27 Report Categories]
  Reports --> Export[CSV / JSON / PDF Export]
```

### B. Modern Visual IVR Builder
* Offers both a **Visual Canvas Interface** and a **Form-based Editor** built on a single graph model.
* Supports multi-level menu branching, digit routing, direct extension dialing, holiday schedules, and automated destination routing.

```mermaid
flowchart TD
  Start[Inbound Call] --> PlayGreeting[Play Welcome Greeting]
  PlayGreeting --> Menu{Collect Keypress}
  Menu -->|Option 1| Queue[Send to Sales Queue]
  Menu -->|Option 2| IVR2[Branch to Support Sub-IVR]
  Menu -->|Timeout / Invalid| Retry{Retry Count < 3?}
  Retry -->|Yes| PlayGreeting
  Retry -->|No| Voicemail[Transfer to General Voicemail]
```

---

## 6. Migration Matrix for Open Source Operators

| Feature | Legacy PBX Software | CaptainPBX |
| :--- | :--- | :--- |
| **Architecture** | Web GUI wrapper over Asterisk configuration files | Decoupled control plane with a Symfony/React core |
| **System Security** | Web server frequently runs as `root` with `sudo` access | Non-root `captain` user with a constrained `captain-system-agent` |
| **Host Protection** | Manual iptables scripts or external firewalls | Built-in nftables firewall (Captain Shield) with GeoIP and IDS |
| **Multi-Tenancy** | Requires separate physical or virtual instances | Enforced multi-tenant isolation on a single media engine |
| **API Support** | Custom PHP scripts or unversioned endpoints | Fully versioned `/api/v1/` REST catalog with OpenAPI specifications |
| **Queue Reporting** | Basic third-party addon scripts | 27 built-in report categories with SLA math and PDF exports |
| **TLS Automation** | Manual Certbot/Let's Encrypt command-line setups | Automated ACME management (Captain Trust) via web UI |
