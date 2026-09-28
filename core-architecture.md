# CaptainPBX Core Architecture & Domain Model

**Product:** CaptainPBX  
**Platform:** Captain Core (Symfony / PHP 8.4)  
**Target Engine:** Asterisk PJSIP  

This document defines the underlying domain model, database boundaries, multi-tenant consistency rules, and module dependencies governing CaptainPBX.

---

## 1. Core Technology Stack

```mermaid
flowchart TB
  subgraph Presentation
    React[React SPA + TypeScript + Vite]
    Portal[User Portal Component Library]
  end

  subgraph CorePlatform [Captain Core Platform]
    Sym[Symfony - PHP 8.4 Framework]
    Domain[Domain Models & Repositories]
    TenantCtx[TenantContext & Fail-Closed Guard]
    ConfigComp[Asterisk Config Compiler]
  end

  subgraph Modules [Product Modules - /usr/src/captainpbx-modules/*]
    AuthMod[Captain Auth]
    ShieldMod[Captain Shield]
    TrustMod[Captain Trust]
    PBXMods[Extensions, Queues, IVR, CDR, Directory]
  end

  subgraph Infrastructure
    DB[(MariaDB - InnoDB)]
    Redis[(Redis - Streams & BullMQ)]
    AstEngine[Asterisk PJSIP Engine]
    Agent[captain-system-agent - Root Daemon]
  end

  React --> Sym
  Portal --> Sym
  Sym --> CorePlatform
  CorePlatform --> Modules
  Modules --> DB
  Modules --> Redis
  CorePlatform --> ConfigComp
  ConfigComp --> AstEngine
  CorePlatform --> Agent
```

---

## 2. Strict Domain Model

To maintain strict domain boundaries, core concepts are explicitly decoupled into individual entities joined through explicit relationship mapping.

```mermaid
flowchart TB
  subgraph Platform [Platform Identity Context]
    Identity[Identity<br>Auth Credentials / TOTP Secrets]
    User[User<br>Global Identity Record]
  end

  subgraph TenantBoundary [Tenant Isolation Boundary]
    Tenant[Tenant<br>Isolation Context]
    UserTenant[UserTenant<br>Membership Join]
    Ext[Extension<br>Telephony Identity e.g. 1001]
    Dev[Device<br>PJSIP Credentials & Endpoint]
    Group[Group / Team<br>Organizational Unit]
    DirContact[Directory Contact<br>External Phonebook Entry]
  end

  Identity -->|1:1 or 1:N| User
  User --> UserTenant
  Tenant --> UserTenant
  UserTenant --> Ext
  Ext -->|1:N| Dev
  User -->|Member of| Group
  Tenant --> DirContact
```

### Key Concept Rules

* **`User` ≠ `Identity`**: `Identity` handles authentication credentials (local, TOTP MFA, future SAML/OIDC). `User` represents the human entity.
* **`User` ≠ `Extension`**: Linked via a 1:1 primary owner relationship (`UserExtension`), but stored in separate tables.
* **`Extension` ≠ `Device`**: Extensions own 0 or more endpoints (`Device`). Each `Device` holds its own unique PJSIP authentication credentials (e.g., `acme-1001-1`, `acme-1001-2`).
* **`Group` ≠ `Role`**: `Group` represents organizational teams (Sales, Support). `Role` represents RBAC authorization bundles.
* **`Tenant` = Hard Isolation Boundary**: Every tenant-owned database table carries `tenant_id`. Cross-tenant queries are blocked at the repository level.

---

## 3. Entity Relationship Diagram (ERD)

```mermaid
erDiagram
    TENANTS ||--o{ USER_TENANTS : owns
    USERS ||--o{ USER_TENANTS : joins
    USERS ||--o{ USER_IDENTITIES : authenticates
    IDENTITIES ||--|| USER_IDENTITIES : holds
    
    TENANTS ||--o{ EXTENSIONS : scope
    TENANTS ||--o{ GROUPS : scope
    TENANTS ||--o{ DIRECTORY_CONTACTS : scope
    
    USER_TENANTS ||--o{ USER_EXTENSIONS : links
    EXTENSIONS ||--o{ USER_EXTENSIONS : assigns
    EXTENSIONS ||--o{ DEVICES : owns
    
    USERS ||--o{ GROUP_MEMBERSHIPS : belongs
    GROUPS ||--o{ GROUP_MEMBERSHIPS : contains
    
    ROLES ||--o{ ROLE_PERMISSIONS : defines
    USER_TENANTS ||--o{ USER_ROLES : grants
    ROLES ||--o{ USER_ROLES : assigns

    TENANTS {
        ulid id PK
        string name
        string sip_domain
    }

    EXTENSIONS {
        ulid id PK
        ulid tenant_id FK
        string extension_number
        string display_name
    }

    DEVICES {
        ulid id PK
        ulid tenant_id FK
        ulid extension_id FK
        string auth_username
        string sip_password
    }
```

---

## 4. Multi-Tenant Data Isolation & Request Lifecycle

Database isolation enforces a **fail-closed model**. Repositories verify `TenantContext` before building SQL queries.

```mermaid
flowchart TB
  Req[Incoming HTTP Request] --> Auth[1. Authenticate Token / Session]
  Auth --> ResolveCtx[2. Resolve Server-Side TenantContext]
  ResolveCtx --> Authz[3. Validate User Permissions for Tenant]
  Authz --> Repo[4. Invoke Captain Core Tenant Repository]
  
  subgraph DataGuard [Fail-Closed Tenant Guard]
    Repo --> Check{TenantContext Active?}
    Check -->|No| Fail[Throw SecurityException / Fail Closed]
    Check -->|Yes| Inject[Inject tenant_id = Context into Query]
    Inject --> DB[(Execute MariaDB SQL)]
  end

  DB --> Audit[5. Record Event in Captain Audit]
  Audit --> Resp[6. Return Response JSON]
```

---

## 5. Module System & Dependency Architecture

Product features are isolated into self-contained modules located in `/usr/src/captainpbx-modules/{id}`. Dependencies between modules are strictly acyclic.

```mermaid
flowchart TD
  Core[Captain Core Platform Services]
  
  subgraph Foundation Modules
    Identity[identity]
    Auth[auth]
    Tenants[tenants]
    Roles[roles]
    Users[users]
    AsteriskMod[asterisk]
  end

  subgraph Feature Modules
    ExtMod[extensions]
    DevMod[devices]
    CDRMod[cdr]
    ShieldMod[captainshield]
    TrustMod[captaintrust]
    SystemMod[captainsystem]
    GroupMod[groups]
    DirMod[directory]
  end

  Identity --> Core
  Auth --> Identity
  Tenants --> Core
  Tenants --> Auth
  Roles --> Core
  Users --> Core
  Users --> Identity
  Users --> Tenants
  Users --> Roles
  
  AsteriskMod --> Core
  ExtMod --> Tenants
  ExtMod --> AsteriskMod
  DevMod --> ExtMod
  DevMod --> AsteriskMod
  CDRMod --> Tenants
  CDRMod --> AsteriskMod
  ShieldMod --> Core
  TrustMod --> Core
  SystemMod --> Core
  GroupMod --> Tenants
  GroupMod --> Users
  DirMod --> Tenants
```

---

## 6. Asterisk PJSIP Endpoint Compilation

Creating an Extension automatically generates an owning User and a default SIP Device. The config compiler generates isolated PJSIP blocks for Asterisk.

```mermaid
sequenceDiagram
  autonumber
  participant Admin as Admin User
  participant API as Extensions Module
  participant Comp as Config Compiler
  participant File as /var/lib/captainpbx/asterisk/
  participant Ast as Asterisk Engine

  Admin->>API: Create Extension 1001 (Name: Alice)
  API->>API: Auto-provision Owner User "Alice" & Device "acme-1001-1"
  API->>Comp: Trigger PJSIP Build
  Comp->>File: Write pjsip_endpoints_captain.conf
  Comp->>File: Write pjsip_auths_captain.conf
  Comp->>File: Write pjsip_aors_captain.conf
  API->>Ast: Send AMI Reload Command (module reload res_pjsip.so)
  Ast-->>Admin: Device acme-1001-1 Ready for SIP Registration
```
