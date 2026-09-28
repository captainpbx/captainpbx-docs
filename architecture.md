# Captain Core Architecture

## Overview

Captain Core is the reusable platform layer that powers CaptainPBX.

It sits between Symfony and CaptainPBX product modules, providing the core services required to build a secure, scalable, and multi-tenant communications platform.

```text
Symfony
    ↓
Captain Core
    ↓
CaptainPBX Modules
    ↓
Asterisk
```

Captain Core is not a framework replacement.

Symfony remains responsible for application infrastructure, while Captain Core provides communications-specific platform services and CaptainPBX modules provide business functionality.

This separation allows the platform to evolve independently from product features while maintaining a clean and maintainable architecture.

---

# Architectural Philosophy

CaptainPBX is built on proven open-source technologies and follows a simple principle:

> Use established components for general infrastructure and focus engineering effort on communications, telephony, security, and multi-tenancy.

Instead of reinventing technologies that already exist, CaptainPBX builds upon mature and widely adopted projects including:

* Symfony
* MariaDB
* Redis
* Nginx
* Asterisk
* Linux

This approach allows the project to focus on delivering PBX and communications functionality while benefiting from the stability, security, and ecosystem of these technologies.

---

# Why Symfony

CaptainPBX is built on Symfony because it provides a mature and well-established foundation for long-term platform development.

Symfony offers:

* HTTP Kernel
* Dependency Injection
* Event Dispatcher
* Console Framework
* Security Components
* Session Management
* Cache Components
* Mail Services
* Testing Infrastructure

These capabilities allow CaptainPBX to focus engineering effort on communications and platform services rather than implementing and maintaining general application infrastructure.

Symfony's modular architecture also aligns well with the Captain Core philosophy, where platform services are built as reusable components on top of a stable application foundation.

---

# Why Captain Core

Captain Core complements Symfony by providing communications-specific platform services.

While Symfony provides application infrastructure, Captain Core provides capabilities commonly required by modern PBX platforms, including:

* Multi-tenant isolation
* Telephony event processing
* Asterisk integration
* Configuration compilation
* Audit services
* Job processing
* Security services
* Platform lifecycle management

This separation keeps platform functionality organized while allowing product modules to focus on business features.

```text
Symfony
    ↓
Captain Core
    ↓
Product Modules
    ↓
Asterisk
```

Captain Core acts as the platform layer between the application framework and the communications stack.

---

# Building On Proven Foundations

CaptainPBX intentionally builds upon established open-source projects rather than replacing them.

Benefits include:

* Long-term maintainability
* Security updates
* Community support
* Extensive documentation
* Operational familiarity
* Predictable upgrade paths

This allows engineering effort to remain focused on PBX innovation rather than rebuilding infrastructure components that already exist.

---

# Why Nginx

Nginx serves as the front-end gateway for CaptainPBX.

Responsibilities include:

* TLS termination
* HTTP request routing
* Reverse proxying
* Static asset delivery
* Compression
* Rate limiting
* Security headers

Benefits include:

### Performance

Nginx uses an event-driven architecture that efficiently handles large numbers of concurrent connections.

### Resource Efficiency

Memory usage remains predictable under load.

### Security

Nginx provides an additional boundary between external traffic and application services.

### Static Asset Delivery

JavaScript, CSS, fonts, images, and SPA assets are served directly without consuming PHP resources.

---

# Why PHP-FPM

PHP-FPM (FastCGI Process Manager) executes CaptainPBX application code.

Nginx and PHP-FPM have separate responsibilities.

```text
Browser
    ↓
Nginx
    ↓
PHP-FPM
    ↓
Symfony
    ↓
Captain Core
```

Benefits include:

### Process Isolation

PHP execution is isolated from the web server.

Application failures do not directly impact Nginx.

### Worker Management

PHP-FPM manages worker pools independently, allowing predictable scaling and resource control.

### Resource Limits

Memory usage, execution limits, and worker counts can be tuned without modifying application code.

### Security

Application services run under a dedicated service account rather than elevated operating system privileges.

---

# Why Separate Linux Users

CaptainPBX intentionally separates responsibilities between Linux service accounts.

```text
root
 └── captain-system-agent

captain
 ├── PHP-FPM
 ├── Scheduler
 ├── Workers
 ├── CLI
 └── Event Services

asterisk
 └── SIP and RTP Processing
```

Benefits include:

### Principle Of Least Privilege

Each component receives only the permissions required for its responsibilities.

### Security Containment

Compromise of one component does not automatically grant access to all platform resources.

### Operational Clarity

Application services, operating system services, and media services remain clearly separated.

---

# Why Asterisk Is Not The Source Of Truth

CaptainPBX separates configuration management from call execution.

Asterisk is responsible for media processing, SIP signaling, and dialplan execution.

CaptainPBX owns platform state and configuration.

```text
Admin UI / API
        ↓
     MariaDB
        ↓
  Captain Core
        ↓
 Configuration Compiler
        ↓
 Generated *_captain.conf
        ↓
     Asterisk
```

Benefits include:

* Auditability
* Multi-tenant consistency
* API-driven management
* Repeatable deployments
* Centralized validation
* Easier automation
* Simplified backup and restore

Asterisk executes calls.

CaptainPBX manages platform state.

---

# Responsibilities

Captain Core owns the platform services that are shared across all CaptainPBX modules.

## Multi-Tenancy

* TenantContext
* Tenant isolation
* Tenant-aware repositories
* Cross-tenant protection

## Security

* Authentication infrastructure
* Authorization
* Permission evaluation
* Audit framework

## Data Access

* Tenant-aware Records
* Doctrine integration
* Repository abstractions

## Telephony Platform

* Asterisk integration
* AMI integration
* ARI integration
* Dialplan compilation
* Configuration generation

## Platform Services

* Jobs
* Scheduler
* Event processing
* Redis integration
* Secrets management
* Module management

---

# What Captain Core Is Not

Captain Core provides platform services.

It does not provide product functionality.

| Platform Services (Captain Core) | Product Features (Modules) |
| -------------------------------- | -------------------------- |
| TenantContext                    | Extensions                 |
| Audit Engine                     | Queues                     |
| Authorization                    | IVR                        |
| Records                          | Trunks                     |
| Dialplan Compiler                | Inbound Routes             |
| Event Pipeline                   | Voice AI                   |
| Module Loader                    | User Portal Features       |
| Asterisk Manager                 | Call Center Features       |

If functionality can exist independently as a product capability, it belongs in a module.

---

# Core Services

## TenantContext

TenantContext is the foundation of platform isolation.

Responsibilities:

* Resolve tenant identity
* Enforce tenant boundaries
* Prevent cross-tenant access
* Supply tenant information to repositories

Rules:

* TenantContext is mandatory for tenant-owned operations
* Frontend tenant selectors are convenience only
* Browser supplied tenant identifiers are never trusted

---

## Records

Records provide fail-closed repository access.

Responsibilities:

* Automatic tenant scoping
* Repository abstraction
* Safe query patterns
* Consistent data access

Without TenantContext, tenant-owned data cannot be accessed.

This helps prevent accidental cross-tenant data exposure.

---

## Audit Engine

Every significant platform action generates an audit record.

Examples:

* User creation
* Extension creation
* Permission changes
* Configuration deployment
* Security changes
* Administrative actions

Goals:

* Compliance
* Traceability
* Operational visibility

---

## Authorization

Authorization determines what an authenticated identity may perform.

Responsibilities:

* Role evaluation
* Permission evaluation
* Tenant scope enforcement
* Administrative boundary protection

Authorization decisions are centralized and shared across modules.

---

## Module Manager

The Module Manager provides:

* Module discovery
* Module registration
* Dependency validation
* Service loading
* Lifecycle management

Modules remain independent while integrating through common platform services.

---

# Asterisk Integration

Captain Core owns interaction with Asterisk.

Modules communicate through platform services rather than directly managing telephony infrastructure.

```text
Module
    ↓
Captain Core Port
    ↓
Asterisk Manager
    ↓
AMI / ARI
    ↓
Asterisk
```

This provides a consistent integration layer across all modules.

---

# Dialplan Manager

The Dialplan Manager compiles configuration from platform records.

Inputs include:

* Extensions
* Queues
* IVRs
* Routes
* Feature Codes

Outputs:

```text
*_captain.conf
```

Generated configuration is considered output rather than the source of truth.

---

# Event Processing

Captain Core owns the telephony event pipeline.

Sources include:

* AMI
* CEL
* CDR
* ARI Events

Pipeline:

```text
Asterisk
    ↓
Event Pump
    ↓
Normalization
    ↓
Redis
    ↓
Consumers
```

Consumers may include:

* Reporting
* Queues
* Presence
* Voice AI
* CRM integrations
* Automation services

---

# Job Framework

Captain Core provides platform scheduling and background processing.

```text
Scheduler
    ↓
Redis Queue
    ↓
Workers
    ↓
Job Handlers
```

Responsibilities:

* Scheduling
* Concurrency control
* Timeouts
* Retry policies
* Execution history
* Tenant fairness

Modules register JobHandlers.

Captain Core executes them.

---

# System Agent Integration

Captain Core is the only platform component that communicates with the System Agent.

```text
Module
    ↓
Captain Core
    ↓
System Agent
    ↓
Operating System
```

This provides a controlled boundary between application services and privileged operating system operations.

---

# Design Goals

Captain Core exists to provide:

1. Multi-tenant isolation
2. Security by default
3. Reusable PBX platform services
4. Consistent module development
5. Safe Asterisk integration
6. Operational observability
7. Long-term maintainability

Captain Core provides the platform.

Modules provide the product.

Together they form the foundation of CaptainPBX.
