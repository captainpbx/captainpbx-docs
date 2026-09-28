# CaptainPBX System Architecture

**One appliance. Four privilege boundaries. Zero manual configuration files.**

CaptainPBX is a modern telecommunications control plane built on **Debian 13**, separating user management, application logic, media execution, and system privileges into distinct, isolated operational layers.

---

## 1. System Architecture at a Glance

```mermaid
flowchart TB
  subgraph humans [User Surfaces]
    Admin[Admin SPA - React]
    Portal[User Portal - React]
    Phone[SIP / WebRTC Endpoints]
  end

  subgraph edge [OS & Edge Layer]
    Nginx[Nginx Reverse Proxy / TLS]
    Shield[Captain Shield - nftables Firewall]
  end

  subgraph app [Application Layer - User: captain]
    Core[Captain Core]
    Mods[Product Modules]
    FPM[PHP-FPM 8.4]
    Jobs[captain-jobs - BullMQ Queue Workers]
    Ev[captain-events - AMI Event Pump]
  end

  subgraph priv [Privileged OS Layer - Root]
    Agent[captain-system-agent - Unix Socket]
  end

  subgraph data [Data Layer]
    DB[(MariaDB - Tenant Data)]
    Redis[(Redis - Streams & Queues)]
  end

  subgraph media [Media Engine - User: asterisk]
    Ast[Asterisk PJSIP Engine]
  end

  Admin -->|HTTPS| Nginx
  Portal -->|HTTPS| Nginx
  Phone -->|SIP / RTP| Shield
  Nginx --> Shield
  Shield --> Nginx
  Shield --> Ast
  Nginx --> FPM
  FPM --> Core
  Core --> Mods
  Core --> DB
  Core --> Redis
  Core -->|Allowlisted Verbs| Agent
  Ev --> Redis
  Jobs --> Redis
  Jobs --> Core
  Ast -->|AMI / CEL| Ev
  Core -->|Generated *_captain.conf| Ast
```

---

## 2. Layered Responsibilities

| Layer | Component | Core Responsibilities | What It Must NOT Do |
| :--- | :--- | :--- | :--- |
| **OS Edge** | Debian 13, Nginx, Captain Shield | SSL termination, default-deny host firewall, rate limiting, GeoIP filtering | Execute application logic or bypass privilege boundaries |
| **Captain Core** | Symfony (PHP 8.4) Platform Services | Tenant Context isolation, auth, audit hooks, job queueing, config compiler | Contain custom UI feature pages or un-audited system calls |
| **Product Modules** | `/usr/src/captainpbx-modules/*` | Module business logic (Shield, Auth, Trust, IVR, Queues), REST APIs, UI components | Edit Asterisk `/etc/asterisk` files or issue `shell_exec` |
| **Media Engine** | Asterisk (PJSIP) | SIP registration, RTP routing, dialplan execution, channel bridging | Act as the source of truth for users, tenants, or extensions |
| **Privileged OS** | `captain-system-agent` | Execute specific system verbs (firewall apply, ACME certs, system time) | Allow unrestricted root shell execution or raw command string execution |

---

## 3. Non-Root Security & Privilege Split

CaptainPBX enforces a strict system-user privilege boundary across the operating system to prevent web application compromises from escalating to OS root access.

```mermaid
flowchart LR
  subgraph debian [Debian 13 Security Boundaries]
    R[User: root<br>captain-system-agent]
    C[User: captain<br>PHP-FPM, CLI, Workers, Ingest]
    A[User: asterisk<br>PJSIP, RTP, Media]
  end

  C -->|JSON Verbs over Unix Socket| R
  C -->|Write /var/lib/captainpbx/asterisk/*_captain.conf| A
```

* **`captain` user**: Runs PHP-FPM, asynchronous BullMQ workers, the AMI event ingest pump, and CLI commands. `shell_exec`, `exec`, `passthru`, and `system` are disabled in PHP-FPM.
* **`asterisk` user**: Dedicated strictly to media processing, RTP streams, and PJSIP channel execution.
* **`root` user**: Runs only the `captain-system-agent` daemon listening on a local Unix socket. It exposes an **allowlist of named verbs** (e.g., `firewall.apply`, `timezone.set`, `acme.renew`).

---

## 4. How Configuration Changes Become Calls

Application changes made in the Admin SPA are written to MariaDB as tenant-scoped records, compiled into configuration fragments, and applied to Asterisk without raw file editing.

```mermaid
sequenceDiagram
  autonumber
  participant UI as Admin SPA
  participant API as PHP Core / Modules
  participant DB as MariaDB
  participant Ag as System Agent
  participant Ast as Asterisk Engine

  UI->>API: POST /api/v1/extensions (or IVR / Firewall)
  API->>DB: Save Tenant-Scoped Record
  API->>API: Execute Audit Hook (Redact Secrets)
  
  alt OS Configuration Change (e.g., Shield, Hostname)
    API->>Ag: Send Allowlisted Verb JSON over Unix Socket
    Ag-->>API: Status OK / Error
  else Telephony Configuration Change
    UI->>API: Trigger Sync & Apply
    API->>Ast: Write /var/lib/captainpbx/asterisk/*_captain.conf
    API->>Ast: Trigger Asterisk Module Reload via AMI
    Ast-->>API: Confirmation
  end
```

---

## 5. Asynchronous Job Pipeline

Background tasks (such as CDR processing, email notifications, maintenance, and scheduled reports) run asynchronously through **Redis BullMQ** to prevent blocking web requests or media handling.

```mermaid
flowchart TB
  Timer[captain-scheduler.timer] -->|Scan Due Jobs| Scan[Scan captaincore_job_runs]
  Scan -->|Enqueue| Bull[Redis BullMQ]
  Bull -->|Worker Concurrency| Workers[captain-jobs Workers]
  Workers -->|Execute CLI| Exec[php captain jobs:exec]
  Exec -->|Run Handler| Handler[JobHandler]
  Handler -->|Update Status| DB[(MariaDB Log)]
```

---

## 6. Real-Time Telephony Event Architecture

Asterisk Manager Interface (AMI) events are ingested by a dedicated event pump service (`captain-events`) and split across isolated Redis stream lanes to eliminate bottlenecking.

```mermaid
flowchart LR
  Ast[Asterisk AMI Engine] -->|Raw Events| Pump[captain-events Pump]
  
  subgraph lanes [Isolated Event Lanes]
    Pump -->|Lane 1| CDR[CDR & CEL Event Stream]
    Pump -->|Lane 2| Queue[Queue Analytics Stream]
    Pump -->|Lane 3| BLF[BLF & Presence Fanout]
  end

  CDR --> DB[(MariaDB CDR Storage)]
  Queue --> Stats[Live Queue Analytics Engine]
  BLF --> WebSockets[Real-time WebSockets / SPA]
```

---

## 7. Host Security & Network Isolation (Captain Shield)

Captain Shield enforces a default-deny architecture. Internal infrastructure services are completely hidden from the public WAN.

```mermaid
flowchart TB
  P[Incoming Packet] --> Check1{Blocked IP / IDS Ban / APIBAN?}
  Check1 -->|Yes| Drop1[DROP Packet]
  Check1 -->|No| Check2{Admin / SIP Allowlist Match?}
  Check2 -->|No| Drop2[DROP Packet]
  Check2 -->|Yes| Check3{GeoIP Country Rule Allowed?}
  Check3 -->|No| Drop3[DROP Packet]
  Check3 -->|Yes| Check4{nftables Rate Limit OK?}
  Check4 -->|No| Drop4[DROP Packet]
  Check4 -->|Yes| Accept[ACCEPT Packet]
```

```mermaid
flowchart LR
  subgraph WAN [Exposed to Internet - Allowed Ports Only]
    HTTPS[Port 443 / 80 - Nginx Admin/Portal]
    SIP[Port 5060 / 5061 - SIP Signals]
    RTP[UDP Range - Media RTP]
  end

  subgraph Isolated [Hidden Internal Services - Never Exposed on WAN]
    AMI[AMI - Port 5038]
    ARI[ARI]
    SQL[MariaDB - Port 3306]
    RDS[Redis - Port 6379]
  end

  Shield[Captain Shield nftables] --> WAN
  Shield -.-x Isolated
```
