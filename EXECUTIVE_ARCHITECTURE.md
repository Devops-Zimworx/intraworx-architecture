# IntraWorX Platform — Executive Architecture Overview

**Version:** 1.0
**Date:** 20 April 2026
**Audience:** Executive Leadership & Stakeholders

---

## What Is IntraWorX?

IntraWorX is a **cloud-native back-office operations platform** that consolidates internal services into a single, secure ecosystem — with the intent to extend its capabilities to the rest of the client-facing team members. It provides one login, one domain (`intraworx.cloud`), and one consistent experience for everything from HR onboarding to coffee shop orders, health clinic visits, shuttle transport, facilities management, and analytics dashboards.

Every employee logs in once through Google and immediately has access to the services relevant to their role — no separate passwords, no separate systems.

---

## Platform at a Glance

| | |
|---|---|
| **Cloud Provider** | Amazon Web Services (us-west-2, Oregon) |
| **Domain** | `intraworx.cloud` |
| **Authentication** | Google Single Sign-On via AWS Cognito |
| **Total Services** | 16 microservices + IoT hardware |
| **Infrastructure** | 100% containerized, Terraform-managed |
| **Operating Countries** | Zimbabwe, Zambia, Costa Rica, US |

---

## How It Works — The Big Picture

```
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│                              AUTHENTICATION LAYER                                        │
│        User → Google OAuth → AWS Cognito → JWT Token → Services [via RBAC]               │
│        * BusyBee uses Supabase Auth (SSO migration planned)                              │
└───────────────────────────────────────┬──────────────────────────────────────────────────┘
                                        │
                   ┌────────────────────┴─────────────────────┐
                   │           AUTH PORTAL (SSO)               │
                   │          intraworx.cloud                  │
                   │                                           │
                   │   Hosts Internal Modules:                 │
                   │   (All modules consume Employees Service  │
                   │    via IntraWorX Backend)                 │
                   │                                           │
                   │   ┌──────────────┐  ┌──────────────┐      │
                   │   │ Client Succ. │  │ FunWorX      │      │
                   │   │ Task & Docs  │  │              │      │
                   │   │              │  │              │      │
                   │   │ Integrations:│  │ Integrations:│      │
                   │   │ • SES Email  │  │ • RFID       │      │
                   │   │ • S3 Storage │  │ • QR Codes   │      │
                   │   └──────────────┘  └──────────────┘      │
                   │   ┌──────────────┐  ┌──────────────┐      │
                   │   │ Wellness     │  │ Philanthropy │      │
                   │   │ Associates   │  │              │      │
                   │   │              │  │ Integrations:│      │
                   │   │ Integrations:│  │ • Leaflet    │      │
                   │   │ • SES Email  │  │   Maps       │      │
                   │   │ • S3 Storage │  │ • S3 Storage │      │
                   │   └──────────────┘  └──────────────┘      │
                   │   ┌──────────────┐  ┌──────────────┐      │
                   │   │ Facilities   │  │ Admin        │      │
                   │   │ Management   │  │ Console      │      │
                   │   │              │  │              │      │
                   │   │ Integrations:│  │ Integrations:│      │
                   │   │ • Vendor Mgmt│  │ • ECS API    │      │
                   │   │ • S3 Storage │  │ • Cognito API│      │
                   │   └──────────────┘  └──────────────┘      │
                   └──────────┬────────────────┬───────────────┘
                              │                │
                     JWT pass-through     Fastify API
                              │                │
                              │          ┌─────┴──────────────┐
                              │          │ IntraWorX Backend  │
                              │          │ (Prisma + DB)      │
                              │          │                    │
                              │          │ Integrations:      │
                              │          │ • Employees Svc    │
                              │          │ • AWS SES/SNS      │
                              │          │ • S3 Storage       │
                              │          └─────┬──────────────┘
                              │                │
                              ▼                ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                      EMPLOYEES SERVICE (Foundation Hub)                             │
│                      employees.intraworx.cloud                                      │
│                                                                                     │
│  Single source of truth for TEAM MEMBER data:                                       │
│  employee records, org structure, departments, skills, documents, countries         │
│                                                                                     │
│  Integrations: HubSpot CRM, Freshservice, Cognito JWT                               │
│                                                                                     │
│  Consumed by: Auth Portal (all modules), Wellness Center, Shuttle,                  │
│  The Grind, Seating Map, TeamEase, BusyBee                                          │
└──┬─────────┬─────────┬─────────┬─────────┬──────────────┬──────────────────────────-┘
   │         │         │         │         │              │
   ▼         ▼         ▼         ▼         ▼              ▼
┌────────┐┌────────┐┌────────┐┌────────┐┌──────────────┐┌──────────────────────┐
│Wellness││Shuttle ││The     ││Seating ││ TeamEase     ││ BusyBee ERP          │
│Center  ││Mgmt    ││Grind   ││Map     ││              ││ Auth: Supabase       │
│        ││        ││        ││        ││ Submodules:  ││ (not SSO yet)        │
│Integr: ││Integr: ││Integr: ││Integr: ││ • Onboarding ││                      │
│• Emp Sv││• Emp Sv││• Emp Sv││• Emp Sv││ • TM Data    ││ Submodules:          │
│• SES   ││• RFID  ││• Cognit││• Cognit││   Management ││ • Client & TM Mgmt   │
│• Cognit││• SES   ││• Promet││        ││ • Offboarding││ • SME Hub            │
│• Promet││• Cognit││        ││        ││              ││ • Offboarding        │
│• Celery││• WS    ││        ││        ││ Integr:      ││ • Reporting          │
│        ││        ││        ││        ││ • Emp Svc    ││                      │
│        ││        ││        ││        ││ • Cognito JWT││ Integr:              │
│        ││        ││        ││        ││ • Sentry     ││ • Emp Svc            │
│        ││        ││        ││        ││ • Offline PWA││ • HubSpot CRM        │
└────────┘└───┬────┘└────────┘└────────┘└──────────────┘│ • Supabase DB        │
              │                                         │ • SES Email          │
              ▲  RFID tap data (SHA256 + HTTPS)         └──────────────────────┘
              │
     ┌────────┴────────┐
     │ TapCard ESP32   │
     │ (IoT Hardware)  │
     │ Integrations:   │
     │ • WiFi/HTTPS    │
     │ • SHA256 hashing│
     └─────────────────┘

  ┌───────────────────────────────────────────────────────────────────────────────────┐
  │  OTHER INTRAWORX SERVICES (SSO-integrated, own data sources — not consuming      │
  │  Employees Service directly)                                                      │
  └───────────────────────────────────────────────────────────────────────────────────┘

┌────────────────┐ ┌────────────────┐ ┌────────────────┐ ┌────────────────┐
│ DailyFlash     │ │ ZimStat        │ │ Buzz AI        │ │ CheetahHub     │
│ (Reports)      │ │ (Demographics) │ │ (IT Support)   │ │ (Documents)    │
│                │ │                │ │                │ │                │
│ Integrations:  │ │ Integrations:  │ │ Integrations:  │ │ Integrations:  │
│ • HubSpot CRM  │ │ • HubSpot CRM  │ │ • Freshservice │ │ • Google Drive │
│ • Google Sheets│ │ • MapLibre GL  │ │ • OpenAI       │ │ • Google Svc   │
│ • Celery/Redis │ │ • Cognito JWT  │ │ • Claude AI    │ │   Account      │
│ • SES Email    │ │                │ │ • ChromaDB     │ │ • Cognito JWT  │
│                │ └────────────────┘ │ • Cognito JWT  │ └────────────────┘
└────────────────┘                    └────────────────┘
┌────────────────┐ 
│ PayrollGuard   │ 
│ (Backup)       │ 
│                │ 
│ Integrations:  │ 
│ • AWS S3 (KMS) │ 
│ • CloudFront   │
│ • Cognito SSO  │ 
│ • PowerShell   │ 
│   Sync Client  │
└────────────────┘
```

---

## Services Overview

### Core Platform

| Service | What It Does |
|---------|-------------|
| **Auth Portal** | Central login, service dashboard, and host for internal modules (Client Success Task and Documents Manager, FunWorX events, Wellness Associates, Philanthropy, Facilities) |
| **Employees Service** | Foundation HR system — employee records, org structure, departments, skills, documents. Used by nearly every other service |

### Back-Office & Operational Services

| Service | What It Does | Domain |
|---------|-------------|--------|
| **Wellness Center** | Medical records, appointments, pharmacy, patient management | `clinic.intraworx.cloud` |
| **Shuttle Management** | Transport booking, RFID boarding, route management, automated billing | `shuttle.intraworx.cloud` |
| **The Grind** | Internal café — menu, POS orders, inventory, promotions | `thegrind.intraworx.cloud` |
| **TeamEase** | Employee onboarding workflows (mobile-first, works offline) | `teamease.intraworx.cloud` |

### Internal Modules (within Auth Portal)

| Module | What It Does |
|--------|-------------|
| **Client Success Task and Documents Manager** | Document management, task tracking, digital signatures |
| **FunWorX** | Company events, invitations, QR check-ins, attendance |
| **Wellness Associates** | Wellness activities, counseling sessions, emergency tracking |
| **Philanthropy** | CSR projects, proposals, budgets, beneficiary tracking, impact maps |
| **Facilities** | Asset management, maintenance, fault reporting, vendor management |

### Operations & Analytics

| Service | What It Does | Domain |
|---------|-------------|--------|
| **DailyFlash** | Automated daily business reports with PDF email distribution | `dailyflash.intraworx.cloud` |
| **ZimStat** | Client demographics dashboard with US heat map visualization | `zimstat.intraworx.cloud` |
| **BusyBee** | ERP — client relationships, team management, sales, training, PTO | `busybee.intraworx.cloud` |
| **BeeCompliant** | Mobile device compliance tracking and reporting | `beecompliant.intraworx.cloud` |
| **CorePTO** | Leave request management with payroll integration | `corepto.intraworx.cloud` |
| **PayrollGuard** | Secure payroll data backup from on-premise to cloud | `payroll.intraworx.cloud` |

### Specialized

| Service | What It Does | Domain |
|---------|-------------|--------|
| **Buzz AI** | AI-powered IT support agent (learns from Freshservice tickets) | `buzzai.intraworx.cloud` |
| **CheetahHub** | Internal Google Drive document viewer | `cheetahhub.intraworx.cloud` |
| **Seating Map** | Multi-site floor plans and seat assignments | `seatingmap.intraworx.cloud` |
| **TapCard (IoT)** | ESP32 RFID readers in shuttles for contactless boarding | Hardware |

---

## How Services Talk to Each Other

```
┌──────────────────────────────────────────────────────────────────────────┐
│                         AUTHENTICATION LAYER                             │
│                                                                          │
│     User → Google → AWS Cognito → JWT Token → All Services               │
│                                                                          │
│     One login. Token carries user identity + role permissions.           │
└──────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                         EMPLOYEES SERVICE (Core)                         │
│                                                                          │
│     ● Every service calls Employees for user data                        │
│     ● Single source of truth for: names, departments, countries,         │
│       org structure, skills, documents                                   │
│     ● Acts as the "employee directory API" for the entire platform       │
└──────────────────────────────────────────────────────────────────────────┘
                                    │
                    ┌───────────────┼───────────────┐
                    ▼               ▼               ▼
             ┌───────────┐   ┌──────────┐   ┌──────────────┐
             │  Clinic   │   │ Shuttle  │   │  Wellness    │
             │  checks   │   │  checks  │   │   checks     │
             │ employee  │   │ employee │   │  employee    │
             │  records  │   │  records │   │   records    │
             └───────────┘   └────┬─────┘   └──────────────┘
                                  │
                                  ▲
                           ┌──────┴──────┐
                           │ IoT Readers │  RFID tap → encrypted → shuttle API
                           └─────────────┘
```

### Communication Patterns

| Pattern | Description |
|---------|-------------|
| **SSO Token Passing** | Auth Portal passes JWT to services via URL; services validate against Cognito |
| **REST API Calls** | Services call each other over HTTPS (e.g., Clinic → Employees Service, BusyBee → Employees Service) |
| **Event-Driven Notifications** | SNS → SQS → Lambda → SES email pipeline for cross-service notifications |
| **IoT → API** | ESP32 devices send encrypted RFID data to Shuttle backend over HTTPS |
| **Shared Database** | All services share one PostgreSQL RDS instance, but each has its own isolated database |
| **External Data Stores** | BusyBee (Supabase), CorePTO (Neon), BeeCompliant (Google Sheets) use external DBs but still consume Employees Service |

---

## Infrastructure Summary

### Hosting

| Layer | Technology | Details |
|-------|-----------|---------|
| **CDN** | AWS CloudFront | Global content delivery with security headers |
| **Load Balancer** | AWS ALB | Routes traffic by hostname to correct service |
| **Compute** | AWS ECS on EC2 | ARM-based containers (t4g.medium) |
| **Database** | AWS RDS PostgreSQL 16 | Single instance, per-service databases |
| **Cache** | AWS ElastiCache Redis 7 | Session caching, task queues |
| **Storage** | AWS S3 | File uploads, backups, logs |
| **Email** | AWS SES | Transactional email from `@intraworx.cloud` |
| **DNS** | AWS Route 53 | Full domain management |

### Security Layers

| Layer | Protection |
|-------|-----------|
| **Edge** | CloudFront + WAF (rate limiting, OWASP rules, SQL injection prevention) |
| **Transport** | TLS 1.3 everywhere — browser to CDN, CDN to ALB, ALB to services, services to DB |
| **Identity** | AWS Cognito with MFA support, Google OAuth, role-based access |
| **Data** | KMS encryption at rest (database, files, secrets, cache) |
| **Secrets** | AWS Secrets Manager — no credentials in code or environment variables |
| **Threat Detection** | AWS GuardDuty with automated email alerts |
| **Audit** | AWS CloudTrail — full API activity logging |
| **Patching** | Automated weekly OS patching via AWS Systems Manager |

### Cost Profile

| Item | Monthly |
|------|---------|
| **Total Forecast** | $893 |
| **EC2/ECS (Compute)** | ~$357 (40%) |
| **RDS (Database)** | ~$268 (30%) |
| **ElastiCache (Redis)** | ~$89 (10%) |
| **Other (S3, CloudFront, etc.)** | ~$179 (20%) |

Cost optimization through Spot instances for non-critical services and ARM-based (Graviton) compute.

---

## External Integrations

| Partner | Used For |
|---------|----------|
| **Google** | Authentication (OAuth), Sheets API (reports), Drive API (documents), Gemini AI (compliance) |
| **HubSpot** | CRM data for client analytics, sales reporting, and daily flash reports |
| **Freshservice** | IT ticketing system — data source for AI support agent |
| **OpenAI / Claude** | AI-powered IT ticket analysis and resolution |
| **Supabase** | External database backend for BusyBee ERP |
| **Calendly** | Meeting scheduling integration |

---

## Role-Based Access Model

Permissions are managed centrally through **Cognito User Groups**. When an employee is assigned to a group, they automatically gain access to the corresponding service.

| Role Level | Description |
|------------|-------------|
| **Admin** | Full access to all services and administrative functions |
| **Full** | Full access within a specific service |
| **Manager** | Management functions within a service |
| **User** | Standard employee access to a service |

Example: An employee with groups `SHUTTLE_User` and `GRIND_User` can use Shuttle Management and The Grind café, but cannot access the Clinic or Seating Map.

---

## Architecture Strengths

| Strength | Detail |
|----------|--------|
| **Unified Identity** | Single Google login provides seamless access to 16+ services |
| **Service Isolation** | Each service has its own database, codebase, and deployment pipeline |
| **Cost Efficiency** | ARM Graviton processors + Spot instances keep costs optimized at ~$893/month for 16+ services |
| **Security Depth** | 7 layers of security from edge (WAF) to data (KMS encryption) |
| **Infrastructure as Code** | Entire platform reproducible from Terraform — consistent, auditable, version-controlled |
| **Multi-Country** | Built for global operations across 5 countries |

---

## Network Flow Diagram

```
Internet
    │
    ▼
CloudFront (CDN + WAF + TLS)
    │
    ▼
Application Load Balancer (host-based routing)
    │
    ├── intraworx.cloud ──────────────▶ Auth Portal (Nuxt 3)
    ├── employees.intraworx.cloud ────▶ Employees Service (Django)
    ├── clinic.intraworx.cloud ───────▶ Wellness Center Frontend (Nuxt 3)
    ├── api.clinic.intraworx.cloud ───▶ Wellness Center Backend (Django)
    ├── shuttle.intraworx.cloud ──────▶ Shuttle Frontend (Nuxt 3)
    ├── api.shuttle.intraworx.cloud ──▶ Shuttle Backend (Fastify)
    ├── thegrind.intraworx.cloud ─────▶ Grind Frontend (React)
    ├── api.thegrind.intraworx.cloud ─▶ Grind Backend (Django)
    ├── cheetahhub.intraworx.cloud ───▶ CheetahHub Frontend (React)
    ├── seatingmap.intraworx.cloud ───▶ Seating Map Frontend (Nuxt 3)
    ├── zimstat.intraworx.cloud ──────▶ ZimStat (Nuxt 4)
    ├── beecompliant.intraworx.cloud ─▶ BeeCompliant (React)
    ├── dailyflash.intraworx.cloud ───▶ DailyFlash Frontend (Nuxt 3)
    ├── teamease.intraworx.cloud ─────▶ TeamEase (Nuxt 3 PWA)
    └── busybee.intraworx.cloud ──────▶ BusyBee (React)
    
    All services ──▶ PostgreSQL (private subnet, encrypted)
    All services ──▶ Redis (private subnet, TLS)
    All services ──▶ S3 (encrypted file storage)
```

---

*This document provides a high-level view of the IntraWorX platform architecture. For detailed technical specifications, refer to the Comprehensive Architecture Document.*

**Documents created by:** Tinashe Zvihwati
