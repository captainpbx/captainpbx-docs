				# Welcome to CaptainPBX

		   ____            _        _       ____  ______  __
		  / ___|__ _ _ __ | |_ __ _(_)_ __ |  _ \| __ ) \/ /
		 | |   / _` | '_ \| __/ _` | | '_ \| |_) |  _ \\  /
		 | |__| (_| | |_) | || (_| | | | | |  __/| |_) /  \
		  \____\__,_| .__/ \__\__,_|_|_| |_|_|   |____/_/\_\
        		    |_|

		        Next-Generation Voice Platform


**CaptainPBX** is a modern, multi-tenant, secure-by-default open-source PBX platform built on a contemporary software stack.

Designed for service providers, enterprises, and developers who need a scalable communications platform that is easy to integrate, secure to operate, and simple to manage.

---

# Why CaptainPBX?

Traditional PBX platforms were designed for a different era.

Today's communication systems must integrate with:

- SIP trunks and telecom providers
- CRM and business applications
- Voice AI platforms
- Contact center solutions
- Automation and workflow engines
- Cloud-native infrastructure

CaptainPBX leverages modern open-source technologies to provide:

✅ Multi-tenant architecture

✅ Secure-by-default deployment

✅ API-first integrations

✅ Modern web administration

✅ Scalable service architecture

✅ Voice AI ready

✅ Developer-friendly ecosystem

✅ Enterprise-grade auditing and access control

---

# Where CaptainPBX Fits

CaptainPBX acts as the communication hub between users, carriers, and business applications.

```mermaid
flowchart LR

    Users["Users<br/>Phones • Softphones • WebRTC"]
    Apps["Business Apps<br/>CRM • Voice AI • Automation"]
    Carriers["Telecom Providers<br/>SIP Trunks • PSTN • DID"]

    Captain["CaptainPBX<br/>Communication Control Platform"]

    Users <--> Captain
    Apps <--> Captain
    Carriers <--> Captain

    Captain --> Features["Call Routing<br/>IVR<br/>Queues<br/>Recording<br/>Reporting"]
```

---

# High-Level Architecture

```mermaid
flowchart TB

    Admin["Admin Portal"]
    User["User Portal"]
    API["REST API / JWT"]

    Admin --> Core
    User --> Core
    API --> Core

    subgraph CaptainPBX
        Core["Captain Core"]

        Modules["PBX Modules"]
        Security["Captain Shield<br/>Security Layer"]
        Audit["Audit & Compliance"]
        Tenant["Multi-Tenant Engine"]

        Core --> Modules
        Core --> Security
        Core --> Audit
        Core --> Tenant
    end

    Core --> Asterisk["Asterisk Media Engine"]
    Core --> Database["MariaDB"]
    Core --> Cache["Redis"]

    Asterisk --> SIP["SIP Endpoints"]
    Asterisk --> Trunks["SIP Trunks"]
```

---

# Internal Components

```mermaid
flowchart TB

    subgraph Access
        Web["Admin SPA"]
        Portal["User Portal"]
        API["OpenAPI / JWT"]
    end

    subgraph Core
        Captain["Captain Core"]
        Modules["Independent Modules"]
        Jobs["Background Jobs"]
        Events["Event Processing"]
    end

    subgraph Security
        Shield["Captain Shield"]
        Audit["Audit Trail"]
        Tenant["Multi-Tenant Isolation"]
    end

    subgraph Data
        Maria[(MariaDB)]
        Redis[(Redis)]
    end

    subgraph Voice
        Ast["Asterisk"]
        SIP["PJSIP"]
    end

    Web --> Captain
    Portal --> Captain
    API --> Captain

    Captain --> Modules
    Captain --> Jobs
    Captain --> Events

    Captain --> Shield
    Captain --> Audit
    Captain --> Tenant

    Captain --> Maria
    Captain --> Redis

    Captain --> Ast
    Ast --> SIP
```

---

# Key Capabilities

| Area | Features |
|--------|----------|
| Telephony | SIP, PJSIP, Trunks, Extensions |
| Call Handling | IVR, Queues, Ring Groups, Time Conditions |
| Multi-Tenant | Tenant Isolation, Delegated Administration |
| Security | Firewall Controls, Access Policies, Auditing |
| Integration | REST APIs, Webhooks, JWT Authentication |
| Voice AI | AI Agents, Voice Bots, External AI Platforms |
| Operations | Monitoring, Logging, Reporting |
| Scalability | Service-Oriented Architecture |

---

# Built on Modern Open Source Technologies

- Asterisk
- PHP 8+
- MariaDB
- Redis
- Nginx
- JWT Authentication
- OpenAPI
- nftables
- Linux

---

# Vision

CaptainPBX is designed to be the next-generation open-source communication platform that combines traditional telephony, modern APIs, cloud-native architecture, and Voice AI into a single secure platform.
