# IntraWorX Platform — Comprehensive Architecture Document

**Version:** 1.0
**Date:** 20 April 2026
**Classification:** Internal — Technical Reference

---

## Table of Contents

1. [Platform Overview](#1-platform-overview)
2. [System Landscape](#2-system-landscape)
3. [Authentication & Identity](#3-authentication--identity)
4. [Network Architecture](#4-network-architecture)
5. [Compute Infrastructure (ECS)](#5-compute-infrastructure-ecs)
6. [Service Catalog](#6-service-catalog)
7. [Data Layer](#7-data-layer)
8. [Content Delivery & DNS](#8-content-delivery--dns)
9. [Security Architecture](#9-security-architecture)
10. [Monitoring, Logging & Alerting](#10-monitoring-logging--alerting)
11. [CI/CD & Deployment](#11-cicd--deployment)
12. [External Integrations](#12-external-integrations)
13. [Disaster Recovery & Business Continuity](#13-disaster-recovery--business-continuity)
14. [Inter-Service Communication Map](#14-inter-service-communication-map)

---

## 1. Platform Overview

IntraWorX is a cloud-native, microservices-based internal operations platform built on AWS. It provides a unified employee experience across human resources, wellness, transport, facilities management, food services, compliance, and analytics — all accessible through a centralized Single Sign-On (SSO) portal.

The platform runs entirely on **AWS (us-west-2 — Oregon)** with infrastructure managed as code via **Terraform**. All services are containerized and deployed to **Amazon ECS** clusters using EC2 instances with a mix of On-Demand and Spot capacity.

### Design Principles

- **Microservices architecture** — each domain is an independently deployable service
- **SSO-first** — AWS Cognito with Google OAuth provides unified identity across all services
- **Infrastructure as Code** — all AWS resources provisioned via Terraform modules
- **Container-native** — every service runs as a Docker container on ECS
- **Security by default** — WAF, TLS everywhere, KMS encryption, private subnets for data and compute

### Platform Domain

| Domain | Provider | Purpose |
|--------|----------|---------|
| `intraworx.cloud` | AWS Route 53 | Primary platform domain |
| `worxconnect.com` | Cloudflare | WorxConnect talent marketplace (separate product) |

---

## 2. System Landscape

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              INTERNET / END USERS                               │
└──────────────────┬──────────────────────────────────────────────────────────────┘
                   │
         ┌─────────┴──────────┐
         │   AWS CloudFront   │  ← CDN + WAF + HTTPS termination
         │  (26+ domain       │
         │   aliases)         │
         └─────────┬──────────┘
                   │
         ┌─────────┴──────────┐
         │  Application Load  │  ← Host-based routing to 25+ target groups
         │    Balancer (ALB)  │
         │   TLS 1.3 policy   │
         └─────────┬──────────┘
                   │
    ┌──────────────┼──────────────┐
    │              │              │
    ▼              ▼              ▼
┌────────┐  ┌──────────┐  ┌──────────────┐
│  Auth  │  │ Internal │  │   External   │
│ Portal │  │ Modules  │  │ Microservices│
│ (Nuxt) │  │ (Fastify)│  │  (Mixed)     │
└────┬───┘  └────┬─────┘  └──────┬───────┘
     │           │               │
     └─────┬─────┘               │
           │                     │
    ┌──────┴───────┐             │
    │  AWS Cognito │◄────────────┘   (JWT validation)
    │  User Pool   │
    │ + Google IdP │
    └──────────────┘
           │
    ┌──────┴───────┐    ┌──────────┐    ┌──────────┐
    │  PostgreSQL  │    │  Redis   │    │   S3     │
    │  RDS 16.8   │    │  7.0     │    │ Storage  │
    │  (shared)   │    │ (cache)  │    │ (KMS)    │
    └──────────────┘    └──────────┘    └──────────┘
```

---

## 3. Authentication & Identity

### 3.1 AWS Cognito User Pool

The entire platform authenticates through a single **AWS Cognito User Pool** with Google as the identity provider.

| Setting | Value |
|---------|-------|
| Pool Name | `intraworx-intraworx-employees` |
| Username Attribute | Email |
| Password Policy | 12+ chars, upper + lower + number + symbol |
| MFA | Optional (TOTP) |
| Identity Provider | Google OAuth (profile, email, openid) |
| Custom Attributes | `employee_id`, `department`, `given_name`, `family_name`, `picture` |
| Advanced Security | AUDIT mode |

### 3.2 Authentication Flow

```
┌──────────┐    ┌──────────────┐    ┌───────────┐    ┌────────────┐
│  User    │───▶│ Auth Portal  │───▶│  Cognito  │───▶│  Google    │
│ Browser  │    │ intraworx.   │    │ Hosted UI │    │  OAuth     │
│          │    │ cloud        │    │           │    │            │
└──────────┘    └──────────────┘    └───────────┘    └────────────┘
     ▲                                    │
     │              ┌─────────────────────┘
     │              ▼
     │       ┌──────────────┐
     │       │ /auth/       │    Exchange auth code → ID + Access tokens
     └───────│ callback     │    Extract cognito:groups from ID token
             │              │    Sync employee profile with employees-service
             └──────────────┘    Set auto-refresh timer (5h 50m cycle)
```

1. User visits `intraworx.cloud` and clicks **"Continue with Google"**
2. AWS Amplify v6 triggers `signInWithRedirect()` → Cognito Hosted UI → Google OAuth
3. After Google authentication, Cognito exchanges the code and redirects to `/auth/callback`
4. The callback page polls `fetchAuthSession()` (up to 5 attempts) to obtain tokens
5. **Cognito groups** are extracted from the ID token `cognito:groups` claim
6. The employee profile is synced via `employees-service` `/api/employees/me/`
7. An auto-refresh timer is started (every 5 hours 50 minutes, before the 6-hour token expiry)

### 3.3 Role-Based Access Control (RBAC)

Access control is driven entirely by **Cognito User Groups**. Each group maps to a service and permission level following the pattern `{MODULE}_{Level}`.

#### Global Role

| Group | Access |
|-------|--------|
| `Admin` | Full access to all services and modules |

#### Per-Service Cognito Groups

| Module | Full | Manager | User / Other |
|--------|------|---------|--------------|
| Employees | `EMP_Full` | `EMP_Manager` | `EMP_User` |
| Shuttle | `SHUTTLE_Full` | `SHUTTLE_Manager` | `SHUTTLE_User` |
| The Grind | `GRIND_Full` | `GRIND_Manager` | `GRIND_User` |
| Wellness Center | `WCC_Full` | `WCC_Manager` | `WCC_User` |
| Seating Map | `SEATING_MAP_Full` | `SEATING_MAP_Manager` | `SEATING_MAP_User` |
| ZimStat | `ZIM_Full` | `ZIM_Manager` | `ZIM_User` |
| PayrollGuard | `PAYROLL_Full` | `PAYROLL_Manager` | `PAYROLL_User` |
| DailyFlash | `DF_Full` | `DF_Manager` | `DF_User` |

#### Internal Module Roles (within Auth Portal)

| Module | Roles |
|--------|-------|
| Client Success Task and Documents Manager (CS) | `CS_Full` → ADMIN, `CS_Manager` → TEAM_MEMBER, `CS_User` → VIEWER |
| FunWorX | `FUNWORX_Full` → ADMIN, `FUNWORX_Manager` → ORGANIZER, `FUNWORX_User` → ATTENDEE |
| Wellness Associates | `WELLNESS_ASSOC_Coordinator`, `WELLNESS_ASSOC_Counselor`, `WELLNESS_ASSOC_Associate`, `WELLNESS_ASSOC_Viewer` |
| Philanthropy | `PHILANTHROPY_Manager`, `PHILANTHROPY_Lead`, `PHILANTHROPY_User`, `PHILANTHROPY_External`, `PHILANTHROPY_Viewer` |
| Facilities | `FACILITIES_Manager`, `FACILITIES_Editor`, `FACILITIES_Viewer` |

### 3.4 Token Passing to External Services

When a user navigates to an external microservice (e.g., The Grind, Shuttle), the Auth Portal:

1. Retrieves a fresh **ID token** from Cognito
2. Appends it as a URL query parameter: `?token=<jwt>`
3. The receiving service validates the JWT against Cognito's JWKS endpoint
4. Cognito groups in the token determine the user's role in that service
5. Each service auto-provisions a local user record on first access

---

## 4. Network Architecture

### 4.1 VPC Design

The platform runs in a dedicated VPC in **us-west-2 (Oregon)** spanning **4 Availability Zones**.

| Component | CIDR |
|-----------|------|
| **VPC** | `10.0.0.0/16` |
| **Public Subnets** | `10.0.1.0/24`, `10.0.2.0/24`, `10.0.3.0/24`, `10.0.4.0/24` |
| **Private App Subnets** | `10.0.10.0/24`, `10.0.11.0/24`, `10.0.12.0/24`, `10.0.13.0/24` |
| **Private Data Subnets** | `10.0.20.0/24`, `10.0.21.0/24`, `10.0.22.0/24`, `10.0.23.0/24` |

### 4.2 Network Topology

```
┌──────────────────────────────────────────────────────────────────────┐
│                        VPC: 10.0.0.0/16                              │
│                                                                      │
│  ┌─── Public Subnets (10.0.1-4.0/24) ──────────────────────────┐    │
│  │  ALB ←→ Internet Gateway                                     │    │
│  │  NAT Gateway (Elastic IP)                                     │    │
│  └───────────────────────────────────────────────────────────────┘    │
│           │                                                          │
│  ┌─── Private App Subnets (10.0.10-13.0/24) ───────────────────┐    │
│  │  ECS Instances (EC2) ← containers                             │    │
│  │  → Internet via NAT Gateway                                   │    │
│  └───────────────────────────────────────────────────────────────┘    │
│           │                                                          │
│  ┌─── Private Data Subnets (10.0.20-23.0/24) ──────────────────┐    │
│  │  RDS PostgreSQL                                               │    │
│  │  ElastiCache Redis                                            │    │
│  │  (no internet access)                                         │    │
│  └───────────────────────────────────────────────────────────────┘    │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

### 4.3 Security Groups

| Security Group | Inbound Rules | Purpose |
|----------------|---------------|---------|
| **ALB** | TCP 80, 443 from `0.0.0.0/0` | Public-facing load balancer |
| **ECS Tasks** | TCP 0–65535 from ALB SG | Container instances (dynamic ports) |
| **RDS** | TCP 5432 from ECS Tasks SG | PostgreSQL database |
| **ElastiCache** | TCP 6379 from ECS Tasks SG | Redis cache |
| **VPC Endpoints** | TCP 443 from ECS Tasks SG | AWS service endpoints |

### 4.4 VPC Flow Logs

- **Destination:** CloudWatch Log Group `/aws/vpc/intraworx`
- **Retention:** 7 days
- **Traffic captured:** All (ACCEPT + REJECT)

---

## 5. Compute Infrastructure (ECS)

### 5.1 ECS Cluster

The platform runs on a single ECS cluster with **dual capacity providers** — On-Demand for critical services and Spot for cost-optimized workloads.

| Setting | Value |
|---------|-------|
| Cluster Name | `intraworx-cluster` |
| Instance Type | `t4g.medium` (ARM64, 4 GB RAM) |
| Spot Fallback Types | `t3a.medium`, `t4g.small` |
| Container Insights | Enabled |
| Log Retention | 7 days |

#### Capacity Providers

| Provider | ASG Min/Max/Desired | Purpose |
|----------|---------------------|---------|
| **On-Demand** | 2 / 4 / 3 | Critical services: Auth Portal, Employees, The Grind, Clinic |
| **Spot** | 3 / 10 / 8 | Non-critical: Shuttle, ZimStat, DailyFlash, etc. |

### 5.2 Launch Template

- **AMI:** Amazon ECS-Optimized Amazon Linux 2 (ARM64)
- **IMDSv2:** Required (`http_tokens = "required"`)
- **Monitoring:** Enhanced
- **Patching:** SSM PatchGroup tag for automated patching

### 5.3 Auto-Scaling

Each ECS service is configured with target-tracking auto-scaling:

- **CPU target:** 70%
- **Memory target:** 80%
- **Scale-out cooldown:** 60 seconds
- **Scale-in cooldown:** 300 seconds

---

## 6. Service Catalog

### 6.1 Auth Portal & Backend (Core)

The Auth Portal is the **central nervous system** — both the SSO gateway and the host application for internal operational modules.

| Component | Stack | Port | Domain |
|-----------|-------|------|--------|
| **Auth Portal** (Frontend) | Nuxt 3, Vue 3, TypeScript, Tailwind CSS, AWS Amplify v6 | 3000 | `intraworx.cloud` |
| **IntraWorX Backend** (Admin API) | Fastify, TypeScript, Prisma ORM, PostgreSQL | 3001 | Internal (via ALB) |

**Internal Modules Hosted within the Auth Portal:**

| Module | Route Prefix | Purpose |
|--------|-------------|---------|
| Admin Console | `/admin` | User management, modules, services, feature flags, ECS monitoring, audit logs |
| Client Success Task and Documents Manager | `/cs` | Document management, task tracking, digital signatures |
| FunWorX | `/funworx` | Company events, invitations, QR check-ins, RFID attendance |
| Wellness Associates | `/wellness-associates` | Wellness activities, counseling sessions, emergency tracking |
| Philanthropy | `/philanthropy` | CSR projects, proposals, budgets, beneficiaries, impact mapping |
| Facilities | `/facilities` | Asset management, locations, maintenance, fault reporting, vendor management |

### 6.2 Employees Service (Foundation)

| Component | Stack | Port | Domain |
|-----------|-------|------|--------|
| **Employees Service** | Django 3.2+, DRF, PostgreSQL, Redis, Celery | 8001 | `employees.intraworx.cloud` |

The **foundational service** consumed by virtually every other microservice. Manages employee lifecycle, organizational structure, departments, skills, documents, and multi-country operations (US, Costa Rica, Zambia, Zimbabwe). Integrates with Cognito JWT auth with auto-provisioning.

### 6.3 Wellness Center

| Component | Stack | Port | Domain |
|-----------|-------|------|--------|
| **Wellness Center Frontend** | Nuxt 3, Vue 3, TypeScript, Tailwind CSS, Pinia | 3000 | `clinic.intraworx.cloud` |
| **Wellness Center Microservice** | Django 3.2+, DRF, PostgreSQL, Redis, Celery | 8000 | `api.clinic.intraworx.cloud` |

HIPAA-compliant medical records, appointments, patient management, pharmacy/inventory, visit tracking, follow-up reminders. Calls `employees-service` with circuit breaker pattern.

### 6.4 Shuttle Management

| Component | Stack | Port | Domain |
|-----------|-------|------|--------|
| **Shuttle Backend** | Fastify 4, Node.js, PostgreSQL, Redis | 3334 | `api.shuttle.intraworx.cloud` |
| **Shuttle Frontend** | Nuxt 3, Vue 3, Tailwind CSS, Pinia | 3000 | `shuttle.intraworx.cloud` |

Employee shuttle transport management with ESP32 RFID boarding, real-time trip tracking, automated monthly billing, route management. Receives RFID tap data from `tapcard-system` ESP32 devices.

### 6.5 The Grind (Coffee Shop)

| Component | Stack | Port | Domain |
|-----------|-------|------|--------|
| **Grind Backend** | Django 3.2+, DRF, PostgreSQL | 8000 | `api.grind.intraworx.cloud` |
| **Grind Frontend** | React 19, TypeScript, Vite, Tailwind CSS, Zustand | 3000 | `thegrind.intraworx.cloud` |

Internal café management — menu, POS sales, inventory, stock movements, promotions/vouchers, analytics.

### 6.6 Automated Seating Map (Project Atlas)

| Component | Stack | Port | Domain |
|-----------|-------|------|--------|
| **Seating Map Backend** | Django 5.1, DRF, PostgreSQL | 8000 | `api.seatingmap.intraworx.cloud` |
| **Seating Map Frontend** | Nuxt 3, Vue 3, Tailwind CSS | 3000 | `seatingmap.intraworx.cloud` |

Multi-site seating management — floor layouts, zones, clusters, seats, team assignments, work schedules across Zimbabwe, Zambia, Costa Rica.

### 6.8 DailyFlash

| Component | Stack | Port | Domain |
|-----------|-------|------|--------|
| **DailyFlash Backend** | Django, DRF, PostgreSQL, Celery, Redis | 8000 | `api.dailyflash.intraworx.cloud` |
| **DailyFlash Frontend** | Nuxt 3, Vue 3, Tailwind CSS | 3000 | `dailyflash.intraworx.cloud` |

Aggregates data from Google Sheets and HubSpot CRM, generates daily flash reports with automated PDF email distribution.

### 6.9 ZimStat

| Component | Stack | Port | Domain |
|-----------|-------|------|--------|
| **ZimStat** | Nuxt 4, Vue 3, TypeScript, Tailwind CSS, MapLibre GL | 3010 | `zimstat.intraworx.cloud` |

Client demographics dashboard with US heat map visualization. Pulls data from HubSpot CRM via server-side API.

### 6.9 BusyBee Hive ERP

| Component | Stack | Port | Domain |
|-----------|-------|------|--------|
| **BusyBee** | React 18, TypeScript, Tailwind CSS, Supabase | 3000 | `busybee.intraworx.cloud` |

Comprehensive ERP for client relationships, team members, training, attendance, performance, PTO, offboarding, attrition analytics, sales reports. Uses **Supabase** as backend (external PostgreSQL, not on platform RDS). HubSpot CRM sync. Consumes `employees-service` for employee data.

### 6.10 PayrollGuard

| Component | Stack | Port | Domain |
|-----------|-------|------|--------|
| **PayrollGuard Web UI** | Nuxt 3 | 3000 | Via CloudFront |
| **PayrollGuard Sync Client** | PowerShell (Windows) | — | On-premise |

Secure payroll data backup — auto-syncs from on-premise Windows systems to AWS S3 with AES-256 encryption, versioning, audit logging. Google SSO via Cognito.

### 6.11 TapCard System (IoT)

| Component | Stack | Domain |
|-----------|-------|--------|
| **ESP32 RFID Readers** | C++ / Arduino | Posts to `api.shuttle.intraworx.cloud` |

Hardware firmware for RFID card readers installed in shuttles. Employees tap cards; data is hashed (SHA256 + salt) and sent via HTTPS to the Shuttle Management backend. WiFi-based with offline resilience.

### 6.12 IntraWorX Notifications Service

| Component | Details |
|-----------|---------|
| **SNS Topic** | `intraworx-cs-notifications` |
| **SQS Queue** | 60s visibility timeout, 4-day retention, long polling |
| **Dead Letter Queue** | 3 max receive, 14-day retention |
| **Lambda Processor** | Node.js — processes SQS messages, sends email via SES |
| **From Address** | `notifications@intraworx.cloud` |

---

## 7. Data Layer

### 7.1 PostgreSQL (RDS)

A single RDS instance hosts databases for all services via a **Lambda database provisioner**.

| Setting | Value |
|---------|-------|
| Identifier | `intraworx-db` |
| Engine | PostgreSQL 16.8 |
| Instance | `db.t4g.small` |
| Storage | 20 GB gp3, KMS-encrypted |
| Multi-AZ | No |
| Backup Retention | 7 days |
| Maintenance Window | Sun 04:00–05:00 UTC |
| Parameter Tuning | `shared_buffers=256MB`, `max_connections=100`, force SSL |

#### Per-Service Databases

Each service gets an isolated database with a dedicated app user (least-privilege):

| Database | Service |
|----------|---------|
| `employees` | Employees Service |
| `wellness` | Wellness Center |
| `clinic` | Wellness Center Clinic |
| `intraworx_backend` | IntraWorX Backend (Fastify/Prisma) |
| `shuttle_management` | Shuttle Management |
| `dailyflash` | DailyFlash |
| `seatingmap` | Automated Seating Map |
| `grind` | The Grind |
| `zimstat` | ZimStat |

#### Database Provisioner Lambda

A VPC-deployed Lambda function automatically:
1. Connects to the RDS master instance
2. Creates the database if it doesn't exist
3. Creates a dedicated app user with a random password
4. Grants appropriate privileges
5. Stores credentials in AWS Secrets Manager (KMS-encrypted)

### 7.2 Redis (ElastiCache)

| Setting | Value |
|---------|-------|
| Engine | Redis 7.0 |
| Node Type | `cache.t4g.micro` |
| Nodes | 1 (single node) |
| Auth Token | 32-char random, stored in Secrets Manager |
| Transit Encryption | TLS (`rediss://` scheme) |
| At-Rest Encryption | Enabled |
| Eviction Policy | `allkeys-lru` |
| Timeout | 300 seconds |

Used for session caching, Celery task queues, and application-level caching.

### 7.3 S3 Storage

| Bucket | Purpose | Encryption |
|--------|---------|------------|
| `intraworx-storage` | Application file storage (uploads, documents) | KMS with key rotation |
| `intraworx-terraform-state` | Terraform state files | AES-256 |
| `intraworx-cloudtrail-logs` | CloudTrail audit trail | KMS |
| `intraworx-cloudwatch-logs` | CloudWatch Logs export | KMS |
| `intraworx-cost-usage-reports` | AWS Cost & Usage Reports | SSE-S3 |

**Storage lifecycle:**
- Old versions expire after 30 days
- All buckets have public access fully blocked
- Versioning enabled on application storage

### 7.4 External Databases

Some services use external database providers:

| Service | Provider | Details |
|---------|----------|---------|
| BusyBee | Supabase | External PostgreSQL + Realtime + Auth |

---

## 8. Content Delivery & DNS

### 8.1 CloudFront Distribution

A single CloudFront distribution serves all IntraWorX services with 26+ domain aliases.

| Setting | Value |
|---------|-------|
| Price Class | `PriceClass_100` (US, Canada, Europe) |
| HTTP/2 | Enabled |
| SSL Certificate | ACM (us-east-1) wildcard for `*.intraworx.cloud` |
| WAF | Regional (ALB-level) |
| Viewer Protocol | HTTPS only (HTTP → 301 redirect) |

**Security Headers (via Response Headers Policy):**
- `Strict-Transport-Security: max-age=31536000; includeSubDomains; preload`
- `X-Content-Type-Options: nosniff`
- `X-Frame-Options: SAMEORIGIN`
- `X-XSS-Protection: 1; mode=block`
- `Referrer-Policy: strict-origin-when-cross-origin`
- `Cross-Origin-Opener-Policy: same-origin`

**Cache Behaviors:**
- `/auth/*` — bypass cache entirely (TTL 0) for authentication flows
- Default — 1 hour TTL with cache on query strings, headers, cookies

### 8.2 SSL/TLS Certificates (ACM)

| Certificate | Domains | Validation |
|-------------|---------|------------|
| IntraWorX Main | `intraworx.cloud`, `*.intraworx.cloud`, `*.staging.intraworx.cloud` + service-specific | DNS (Route 53 — automated) |
| IntraWorX APIs | `api.thegrind.intraworx.cloud`, clinic, dailyflash, wellness APIs | DNS (Route 53 — automated) |
| CloudFront Cert | Same domains (must be in us-east-1) | DNS (Route 53 — automated) |

TLS Policy: `ELBSecurityPolicy-TLS13-1-2-2021-06` (TLS 1.3 preferred, 1.2 minimum)

### 8.3 ALB Host-Based Routing

The ALB routes traffic by hostname to the appropriate ECS service target group:

| Priority | Host | Target Service |
|----------|------|----------------|
| 8 | `intraworx.cloud` | Auth Portal |
| 10 | `employees.intraworx.cloud` | Employees Service |
| 115 | `api.shuttle.intraworx.cloud` | Shuttle Backend |
| 116 | `shuttle.intraworx.cloud` | Shuttle Frontend |
| 117 | `api.clinic.intraworx.cloud` | Clinic Microservice |
| 120 | `api.seatingmap.intraworx.cloud` | Seating Map Backend |
| 121 | `seatingmap.intraworx.cloud` | Seating Map Frontend |
| 122 | `api.thegrind.intraworx.cloud` | Grind Backend |
| 123 | `thegrind.intraworx.cloud` | Grind Frontend |
| 125 | `zimstat.intraworx.cloud` | ZimStat |

### 8.4 DNS (Route 53)

The `intraworx.cloud` zone is fully managed in AWS Route 53 with automated certificate validation. All subdomains point to either:
- **CloudFront** — for frontend applications (cached, with security headers)
- **ALB** — for backend APIs (dynamic, no caching)

SES domain verification and DKIM records are also managed in Route 53.

---

## 9. Security Architecture

### 9.1 WAF (Web Application Firewall)

| Rule | Details |
|------|---------|
| Rate Limiting | 2,000 requests / 5 minutes per IP |
| AWS Managed: Common Rule Set | OWASP Top 10 protections |
| AWS Managed: Known Bad Inputs | Blocks known exploit patterns |
| AWS Managed: SQL Injection | SQL injection prevention |

### 9.2 Encryption

| Layer | Method |
|-------|--------|
| Data in Transit | TLS 1.2/1.3 everywhere (ALB, CloudFront, Redis, RDS) |
| Data at Rest (RDS) | KMS encryption with dedicated key |
| Data at Rest (S3) | KMS encryption with key rotation |
| Data at Rest (Redis) | Encryption enabled |
| Secrets | AWS Secrets Manager with KMS encryption |
| HIPAA Fields | Application-level AES encryption key for PHI/PII |

### 9.3 Secrets Management

All sensitive credentials are stored in **AWS Secrets Manager** with KMS encryption:

| Secret | Contents |
|--------|----------|
| GHCR Credentials | GitHub Container Registry auth for image pulls |
| DB Master Password | RDS master credentials + connection URL |
| Per-Service DB Credentials | Individual app user + password + URL per database |
| App Secrets | JWT secret, refresh JWT, cookie secrets |
| Service API Key | Inter-service authentication key |
| Encryption Keys | PHI/PII field encryption key |
| SES SMTP Credentials | Email sending auth |
| Redis Auth Token | ElastiCache auth + URL |
| Shuttle Secrets | ESP32 API key, hash salt |
| BusyBee Supabase | Supabase URL + anon key |

### 9.4 GuardDuty

Intelligent threat detection enabled across:
- VPC Flow Logs
- DNS query logs
- CloudTrail events
- S3 data events
- EBS malware protection

High/Critical findings → SNS → email alerts to `devops@zimworx.com` via EventBridge.

### 9.5 CloudTrail

- **Scope:** Multi-region, all management events + S3 data events
- **Storage:** Encrypted S3 bucket with lifecycle (Glacier archival)
- **Log validation:** Enabled
- **CloudWatch integration:** Real-time alerting capability

### 9.6 SSM Patch Management

- **Baseline:** Amazon Linux 2 — Security + Critical patches
- **Schedule:** Weekly maintenance window
- **Auto-patch:** State Manager patches new spot instances on launch
- **Targeting:** Tag-based (`PatchGroup`)

---

## 10. Monitoring, Logging & Alerting

### 10.1 CloudWatch Alarms

| Alarm | Metric | Threshold |
|-------|--------|-----------|
| ALB High Response Time | TargetResponseTime | > 1.0 second |
| ALB Unhealthy Hosts | UnHealthyHostCount | > 0 |
| ALB 5xx Errors | HTTPCode_Target_5XX_Count | > 10 |
| ECS High CPU | CPUUtilization | > 80% |
| ECS High Memory | MemoryUtilization | > 80% |
| RDS High CPU | CPUUtilization | > 80% |
| RDS Low Storage | FreeStorageSpace | < 1 GB |
| RDS High Connections | DatabaseConnections | > 80 |
| Redis High CPU | EngineCPUUtilization | > 75% |
| Redis High Memory | DatabaseMemoryUsagePercentage | > 85% |

### 10.2 Alert Pipeline

```
CloudWatch Alarm → SNS Topic (KMS-encrypted) → devops@zimworx.com
GuardDuty Finding → EventBridge → SNS → devops@zimworx.com
```

### 10.3 Container Insights

ECS Container Insights is enabled on the cluster, providing:
- Per-container CPU, memory, network, and disk metrics
- Service-level aggregate metrics
- Task-level performance data

### 10.4 Log Architecture

| Log Source | Destination | Retention |
|------------|-------------|-----------|
| ECS Service Logs | CloudWatch `/ecs/{service}` | 7 days |
| VPC Flow Logs | CloudWatch `/aws/vpc/intraworx` | 7 days |
| ALB Access Logs | S3 | Per S3 lifecycle |
| CloudTrail | S3 + CloudWatch | Glacier archival |
| Redis Slow Logs | CloudWatch | Default |
| Redis Engine Logs | CloudWatch | Default |

---

## 11. CI/CD & Deployment

### 11.1 Container Registry

All images are hosted on **GitHub Container Registry (GHCR)** under `ghcr.io/devops-zimworx/`.

- Private images pulled via Secrets Manager credentials stored in ECS task definitions
- Images tagged with SHA256 digests for reproducibility

### 11.2 Deployment Pipeline

```
Developer Push → GitHub Actions
  → Build Docker Image
  → Push to GHCR
  → Update ECS Task Definition
  → Deploy to ECS Service (rolling update)
```

### 11.3 Infrastructure Changes

```
Terraform Plan → PR Review → Terraform Apply
  → State stored in S3 (intraworx-terraform-state)
  → Locking via DynamoDB (intraworx-terraform-locks)
```

---

## 12. External Integrations

| Integration | Used By | Purpose |
|-------------|---------|---------|
| **Google OAuth** | Cognito | Identity provider for all users |
| **HubSpot CRM** | BusyBee, ZimStat, Employees, DailyFlash | Client relationship data |
| **Freshservice** | Employees | IT ticketing and knowledge base |
| **Supabase** | BusyBee | External database + auth + realtime |
| **Calendly** | DailyFlash | Meeting scheduling |
| **AWS SES** | Platform-wide | Transactional email |

---

## 13. Disaster Recovery & Business Continuity

The platform has a **designed but currently disabled** DR configuration (for cost optimization):

### When Enabled:

| Component | Primary | DR Target |
|-----------|---------|-----------|
| Region | us-west-2 (Oregon) | eu-west-1 (Ireland) |
| RDS | Source instance | Cross-region read replica |
| S3 | Primary bucket | Cross-region replication bucket |
| KMS | Primary key | DR-region key |

### Current State:
- `enable_disaster_recovery = false` (disabled for cost savings)
- RDS automated backups: 7-day retention
- S3 versioning: Enabled
- Infrastructure reproducible via Terraform

---

## 14. Inter-Service Communication Map

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
│                      EMPLOYEES SERVICE (Foundation Hub)                              │
│                      employees.intraworx.cloud                                      │
│                                                                                     │
│  Single source of truth for TEAM MEMBER data:                                       │
│  employee records, org structure, departments, skills, documents, countries          │
│                                                                                     │
│  Integrations: HubSpot CRM, Freshservice, Cognito JWT                               │
│                                                                                     │
│  Consumed by: Auth Portal (all modules), Wellness Center, Shuttle,                  │
│  The Grind, Seating Map, BusyBee                                          │
└──┬─────────┬─────────┬─────────┬─────────┬──────────────┬──────────────────────────-┘
   │         │         │         │         │              │
   ▼         ▼         ▼         ▼         ▼              ▼
┌────────┐┌────────┐┌────────┐┌────────┐┌──────────────┐┌──────────────────────┐
│Wellness││Shuttle ││The     ││Seating ││ BusyBee ERP  ||                      │
│Center  ││Mgmt    ││Grind   ││Map     ││              ││ Auth: Supabase       │
│        ││        ││        ││        ││ Submodules:  ││                      │
│Integr: ││Integr: ││Integr: ││Integr: ││ • Onboarding ││                      │
│• Emp Sv││• Emp Sv││• Emp Sv││• Emp Sv││ • TM Data    ││ Submodules:          │
│• SES   ││• RFID  ││• Cognit││• Cognit││   Management ││ • Client & TM Mgmt   │
│• Cognit││• SES   ││• Promet││        ││ • Offboarding││ • SME Hub            │
│• Promet││• Cognit││        ││        ││              ││ • Offboarding        │
│• Celery││• WS    ││        ││        ││ Integr:      ││ • Reporting          │
│________││________││________││________││ • HubSpot CRM │______________________|
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

┌────────────────┐ ┌────────────────┐ ┌────────────────┐
│ DailyFlash     │ │ ZimStat        │ │ PayrollGuard   │
│ (Reports)      │ │ (Demographics) │ │ (Backup)       │
│                │ │                │ │                │
│ Integrations:  │ │ Integrations:  │ │ Integrations:  │
│ • HubSpot CRM  │ │ • HubSpot CRM  │ │ • AWS S3 (KMS) │
│ • Google Sheets│ │ • MapLibre GL  │ │ • CloudFront   │
│ • Celery/Redis │ │ • Cognito JWT  │ │ • Cognito SSO  │
│ • SES Email    │ │                │ │ • PowerShell   │
│                │ └────────────────┘ │   Sync Client  │
└────────────────┘                    └────────────────┘
```

### Data Flow Summary

| Source | Target | Protocol | Purpose |
|--------|--------|----------|---------|
| Auth Portal | All Services | JWT (URL param) | SSO token passing |
| Auth Portal (all internal modules) | Employees Service | REST API (via IntraWorX Backend) | Team member data for CS, FunWorX, Wellness, Philanthropy, Facilities |
| Wellness Center, Shuttle, Grind, Seating Map | Employees Service | REST API + Cognito JWT | Team member data lookup |
| BusyBee | Employees Service | REST API (Supabase Auth) | Team member data lookup (not SSO — uses Supabase Auth) |
| TapCard ESP32 | Shuttle Backend | HTTPS (SHA256 hash) | RFID boarding data |
| Lambda Processor | SES | AWS SDK | Email notifications |
| Services | RDS | PostgreSQL (TLS) | Data persistence |
| Services | Redis | Redis (TLS) | Caching / task queues |
| Services | S3 | AWS SDK (HTTPS) | File storage |
| DailyFlash | Google Sheets, HubSpot | HTTPS API | Report data aggregation |
| ZimStat | HubSpot CRM | HTTPS API | Client demographics data |
| DailyFlash, ZimStat | HubSpot | HTTPS API | CRM data sync |

---

**Document created by:** Tinashe Zvihwati
