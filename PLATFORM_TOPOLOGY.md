# IntraWorX Platform — Topology Document

**Version:** 1.0
**Date:** 22 April 2026
**Classification:** Internal — Reference

---

## Table of Contents

1. [Platform Summary](#1-platform-summary)
2. [Global Traffic Flow](#2-global-traffic-flow)
3. [Network Layout](#3-network-layout)
4. [Service Map](#4-service-map)
5. [Domain & Routing Directory](#5-domain--routing-directory)
6. [Security Boundaries](#6-security-boundaries)
7. [Data Stores & Connections](#7-data-stores--connections)
8. [Authentication Topology](#8-authentication-topology)
9. [Notification & Email Pipeline](#9-notification--email-pipeline)
10. [External Integrations](#10-external-integrations)
11. [Disaster Recovery Topology](#11-disaster-recovery-topology)

---

## 1. Platform Summary

IntraWorX is hosted on **Amazon Web Services** in the **us-west-2 (Oregon)** region. All services run as containers on AWS ECS clusters, fronted by a single Application Load Balancer and a CloudFront CDN. The platform domain is `intraworx.cloud`, with a separate product, WorxConnect, running on `worxconnect.com`.

| | |
|---|---|
| **Region** | us-west-2 (Oregon) |
| **Domain** | `intraworx.cloud` (Route 53) |
| **Secondary Domain** | `worxconnect.com` (Cloudflare) |
| **ECS Clusters** | 2 (IntraWorX + WorxConnect) |
| **Total Services** | 20+ containers across both clusters |
| **Databases** | 9 PostgreSQL databases on shared RDS |
| **Cache** | Redis 7.0 (4 logical partitions) |
| **CDN** | 2 CloudFront distributions (20+ aliases) |

---

## 2. Global Traffic Flow

All user traffic follows a consistent path from browser to container:

```
                          ┌──────────────┐
                          │   End Users  │
                          │  (Browsers)  │
                          └──────┬───────┘
                                 │
                          ┌──────┴───────┐
                          │   Route 53   │  DNS resolution
                          │  (DNS)       │  ─────────────────────┐
                          └──────┬───────┘                       │
                                 │                               │
                    ┌────────────┴────────────┐          ┌───────┴────────┐
                    │                         │          │   Cloudflare   │
                    ▼                         │          │  (worxconnect  │
             ┌──────────────┐                 │          │   .com DNS)    │
             │  CloudFront  │                 │          └───────┬────────┘
             │  CDN + WAF   │                 │                  │
             │  (HTTPS)     │                 │                  │
             └──────┬───────┘                 │                  │
                    │                         │                  │
                    └─────────┬───────────────┘                  │
                              │                                  │
                              ▼                                  │
                    ┌──────────────────┐                         │
                    │  Application     │◄────────────────────────┘
                    │  Load Balancer   │
                    │  (ALB — HTTPS)   │
                    │                  │
                    │  Host-based      │
                    │  routing rules   │
                    └────────┬─────────┘
                             │
            ┌────────────────┼────────────────┐
            │                │                │
            ▼                ▼                ▼
     ┌────────────┐  ┌────────────┐  ┌────────────┐
     │ Frontend   │  │ Backend    │  │ Standalone  │
     │ Containers │  │ Containers │  │ Services    │
     │ (Nuxt,     │  │ (Django,   │  │ (ZimStat,  │
     │  React)    │  │  Fastify,  │  │  BusyBee,  │
     │            │  │  FastAPI)  │  │  etc.)     │
     └────────────┘  └──────┬─────┘  └────────────┘
                            │
               ┌────────────┼────────────┐
               │            │            │
               ▼            ▼            ▼
          ┌─────────┐ ┌─────────┐ ┌──────────┐
          │ RDS     │ │ Redis   │ │ S3       │
          │ Postgres│ │ Cache   │ │ Storage  │
          └─────────┘ └─────────┘ └──────────┘
```

**Routing logic:**
- Frontend domains (e.g. `clinic.intraworx.cloud`) → CloudFront → ALB → container
- API domains (e.g. `api.clinic.intraworx.cloud`) → ALB directly → container

---

## 3. Network Layout

The platform runs inside a single VPC spanning **4 Availability Zones**, organized into three subnet tiers.

```
┌──────────────────────────────────────────────────────────────────────────┐
│                          VPC: 10.0.0.0/16                                │
│                          Region: us-west-2                               │
│                                                                          │
│  ┌─── PUBLIC TIER (10.0.1–4.0/24) ─────────────────────────────────┐    │
│  │                                                                   │    │
│  │  Internet Gateway ←→ ALB (load balancing across all services)     │    │
│  │  NAT Gateway (outbound internet for private subnets)              │    │
│  │                                                                   │    │
│  │  AZ-a: 10.0.1.0/24    AZ-b: 10.0.2.0/24                         │    │
│  │  AZ-c: 10.0.3.0/24    AZ-d: 10.0.4.0/24                         │    │
│  └───────────────────────────────────────────────────────────────────┘    │
│           │                                                              │
│  ┌─── APPLICATION TIER (10.0.10–13.0/24) ──────────────────────────┐    │
│  │                                                                   │    │
│  │  ECS Container Instances (EC2 ARM64 — Graviton)                   │    │
│  │  All application containers run here                              │    │
│  │  Outbound internet via NAT Gateway                                │    │
│  │                                                                   │    │
│  │  AZ-a: 10.0.10.0/24   AZ-b: 10.0.11.0/24                        │    │
│  │  AZ-c: 10.0.12.0/24   AZ-d: 10.0.13.0/24                        │    │
│  └───────────────────────────────────────────────────────────────────┘    │
│           │                                                              │
│  ┌─── DATA TIER (10.0.20–23.0/24) ────────────────────────────────┐     │
│  │                                                                   │    │
│  │  PostgreSQL RDS (primary in AZ-a, standby in AZ-b)               │    │
│  │  Redis ElastiCache (AZ-a)                                         │    │
│  │  No direct internet access                                        │    │
│  │                                                                   │    │
│  │  AZ-a: 10.0.20.0/24   AZ-b: 10.0.21.0/24                        │    │
│  │  AZ-c: 10.0.22.0/24   AZ-d: 10.0.23.0/24                        │    │
│  └───────────────────────────────────────────────────────────────────┘    │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

### Subnet Summary

| Tier | CIDR Range | Purpose | Internet Access |
|------|-----------|---------|-----------------|
| Public | `10.0.1–4.0/24` | Load balancer, NAT gateway | Direct (via Internet Gateway) |
| Application | `10.0.10–13.0/24` | All ECS containers | Outbound only (via NAT Gateway) |
| Data | `10.0.20–23.0/24` | Databases and cache | None |

---

## 4. Service Map

### 4.1 ECS Cluster: IntraWorX

The primary cluster runs all IntraWorX platform services on **ARM64 Graviton** instances.

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                     ECS CLUSTER: IntraWorX                                    │
│                                                                              │
│  Core ─────────────────── intraworx.cloud                                     │
│    └─ Hosts: Admin, Client Success, FunWorX,                                 │
│       Wellness Associates, Philanthropy, Facilities                          │
│                                                                              │
│  Employees ──────────── employees.intraworx.cloud                            │
│                                                                              │
│  Wellness Center ────── clinic.intraworx.cloud                               │
│                                                                              │
│  Shuttle Management ─── shuttle.intraworx.cloud                              │
│    └─ Receives data from TapCard ESP32 IoT devices                           │
│                                                                              │
│  The Grind ──────────── thegrind.intraworx.cloud                             │
│                                                                              │
│  Seating Map ────────── seatingmap.intraworx.cloud                           │
│                                                                              │
│  ZimStat ────────────── zimstat.intraworx.cloud                              │
│                                                                              │
│  DailyFlash ─────────── dailyflash.intraworx.cloud                           │
│                                                                              │
│  BusyBee ERP ────────── busybee.intraworx.cloud                              │
│                                                                              │
│  PayrollGuard ────────── payroll.intraworx.cloud                             │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

### 4.2 ECS Cluster: WorxConnect

A separate cluster for the WorxConnect talent marketplace product.

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                     ECS CLUSTER: WorxConnect                                  │
│                                                                              │
│  WorxConnect ────────── worxconnect.com                                       │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

### 4.3 IoT & On-Premise

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                     IoT & ON-PREMISE DEVICES                                 │
│                                                                              │
│  TapCard ESP32 RFID Readers (installed in shuttles)                          │
│    └─ Posts boarding data to Shuttle Management                              │
│    └─ WiFi-based with offline resilience                                     │
│                                                                              │
│  PayrollGuard Sync Client (on-premise Windows)                               │
│    └─ Encrypted upload to AWS S3                                             │
│    └─ Runs on local payroll workstations                                     │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 5. Domain & Routing Directory

### 5.1 CloudFront Distributions

| Distribution | Domains Served | Origin |
|-------------|----------------|--------|
| **IntraWorX** | 20+ aliases — all `*.intraworx.cloud` frontend domains (staging + production) | ALB |
| **WorxConnect** | `worxconnect.com`, `staging.worxconnect.com` | ALB |

### 5.2 Complete Domain Directory

| Service | Domain | Protocol |
|---------|--------|----------|
| Core | `intraworx.cloud` | HTTPS → CloudFront → ALB |
| Employees | `employees.intraworx.cloud` | HTTPS → ALB |
| Wellness Center | `clinic.intraworx.cloud` | HTTPS → CloudFront → ALB |
| Shuttle Management | `shuttle.intraworx.cloud` | HTTPS → CloudFront → ALB |
| The Grind | `thegrind.intraworx.cloud` | HTTPS → CloudFront → ALB |
| Seating Map | `seatingmap.intraworx.cloud` | HTTPS → CloudFront → ALB |
| ZimStat | `zimstat.intraworx.cloud` | HTTPS → CloudFront → ALB |
| DailyFlash | `dailyflash.intraworx.cloud` | HTTPS → CloudFront → ALB |
| BusyBee ERP | `busybee.intraworx.cloud` | HTTPS → ALB |
| PayrollGuard | `payroll.intraworx.cloud` | HTTPS → CloudFront → ALB |
| WorxConnect | `worxconnect.com` | HTTPS → CloudFront → ALB |

### 5.3 DNS Management

| Zone | Provider | Services |
|------|----------|----------|
| `intraworx.cloud` | AWS Route 53 | All IntraWorX services, SES/DKIM records, ACM validation |
| `worxconnect.com` | Cloudflare | WorxConnect frontend and API |

---

## 6. Security Boundaries

### 6.1 Perimeter Controls

```
┌─────────────────────────────────────────────────────────────────────┐
│                      EDGE LAYER                                      │
│                                                                      │
│  CloudFront ─── WAF (rate limiting, OWASP rules, SQLi protection)    │
│  HTTPS only ─── TLS 1.2 / 1.3                                       │
│  Security headers: HSTS, X-Frame-Options, CSP                       │
│                                                                      │
│  Rate Limit: 2,000 requests per 5 minutes per IP                    │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
┌────────────────────────────────┴────────────────────────────────────┐
│                      NETWORK LAYER                                   │
│                                                                      │
│  ALB Security Group ──── accepts TCP 80, 443 from internet           │
│  ECS Security Group ──── accepts all ports from ALB SG only          │
│  RDS Security Group ──── accepts TCP 5432 from ECS SG only           │
│  Redis Security Group ── accepts TCP 6379 from ECS SG only           │
│                                                                      │
│  Private subnets: no inbound internet access                         │
│  Data subnets: no internet access at all                             │
│  VPC Flow Logs: all traffic captured (7-day retention)               │
└─────────────────────────────────────────────────────────────────────┘
```

### 6.2 Encryption

| What | How |
|------|-----|
| Traffic between users and platform | TLS 1.2/1.3 via CloudFront and ALB |
| Traffic between containers and databases | TLS (PostgreSQL SSL enforced, Redis TLS) |
| Data stored in databases | KMS encryption at rest |
| Data stored in S3 | KMS encryption with key rotation |
| Secrets and credentials | AWS Secrets Manager (KMS-encrypted) |
| Sensitive health records | Application-level AES encryption |

### 6.3 Threat Detection

| Service | Coverage |
|---------|----------|
| AWS GuardDuty | VPC flow logs, DNS queries, CloudTrail events, S3 data, EBS |
| AWS CloudTrail | All API calls across the account (multi-region) |
| VPC Flow Logs | All network traffic (accept + reject) |
| WAF Logs | Blocked and allowed HTTP requests |

---

## 7. Data Stores & Connections

### 7.1 PostgreSQL RDS

A single RDS instance (PostgreSQL 16.8) hosts isolated databases for each service. Each service has its own database with dedicated credentials stored in Secrets Manager.

| Database | Service |
|----------|---------|
| `employees` | Employees |
| `clinic` | Wellness Center |
| `intraworx_backend` | Core |
| `shuttle_management` | Shuttle Management |
| `grind` | The Grind |
| `seatingmap` | Seating Map |
| `dailyflash` | DailyFlash |
| `zimstat` | ZimStat |

### 7.2 Redis ElastiCache

Single Redis 7.0 node with logical database partitioning. All connections over TLS.

| DB | Used By | Purpose |
|----|---------|---------|
| 0 | WorxConnect | Session cache, API response cache |
| 1 | Employees, Wellness Center | Cache, session data |
| 2 | Celery Workers | Task results and state |
| 3 | Shuttle | Real-time device tracking |

### 7.3 S3 Storage

| Bucket | Purpose |
|--------|---------|
| `intraworx-storage` | Application file uploads, documents (Client Success, Philanthropy, Facilities) |
| `intraworx-terraform-state` | Terraform state files |
| `intraworx-cloudtrail-logs` | Audit trail |
| `intraworx-cloudwatch-logs` | Log archival |
| `intraworx-cost-usage-reports` | Cost reporting |

### 7.4 External Data Sources

| Service | External Store | Purpose |
|---------|---------------|---------|
| BusyBee ERP | Supabase (external PostgreSQL) | Client and team member data, real-time features |


---

## 8. Authentication Topology

All services authenticate through a single AWS Cognito User Pool with Google as the identity provider.

```
┌──────────┐     ┌──────────────────┐     ┌───────────┐     ┌──────────┐
│  User    │────▶│  Core            │────▶│  AWS      │────▶│  Google  │
│  Browser │     │  intraworx.cloud │     │  Cognito  │     │  OAuth   │
└──────────┘     └──────────────────┘     └─────┬─────┘     └──────────┘
                                                │
                                         JWT Token issued
                                         (contains user roles)
                                                │
                    ┌───────────────────────────┴───────────────────────┐
                    │                                                   │
              Internal Modules                                External Services
              (within Core)                                 (separate containers)
                    │                                                   │
      ┌─────────────┼─────────────┐                    ┌───────────────┼────────────────┐
      │             │             │                    │               │                │
  Client Succ.  FunWorX     Philanthropy          Shuttle        Wellness          The Grind
  Wellness Assoc. Facilities  Admin Console       Seating Map    DailyFlash        ZimStat
      │             │             │                    │               │                │
      └─────────────┼─────────────┘                    └───────────────┼────────────────┘
                    │                                                   │
              Core API                                             JWT validated via
              (platform API)                                     Cognito JWKS endpoint
                    │                                                   │
                    ▼                                                   ▼
            Employees  ◄────────────────────────────────────── REST API calls
            (source of truth                                  with Cognito JWT
             for all team
             member data)
```

**How tokens flow to external services:**
1. User navigates to an external service from Core
2. Core retrieves a fresh JWT token from Cognito
3. Token is passed as a URL parameter to the receiving service
4. The service validates the JWT against Cognito's public keys
5. Cognito groups in the token determine the user's role
6. The service auto-creates a local user record on first access

**Role groups follow the pattern:** `{MODULE}_{Level}` (e.g., `SHUTTLE_Manager`, `WCC_Full`, `GRIND_User`)

**Global admin role:** `Admin` — grants full access to all services

---

## 9. Notification & Email Pipeline

```
┌──────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌──────────┐
│  Application │────▶│  SNS Topic  │────▶│  SQS Queue  │────▶│   Lambda    │────▶│  AWS SES │
│  (ECS Task)  │     │             │     │             │     │  Processor  │     │  (Email) │
└──────────────┘     └─────────────┘     └──────┬──────┘     └─────────────┘     └──────────┘
                                                │
                                         (after 3 failed
                                          attempts)
                                                │
                                         ┌──────▼──────┐
                                         │  Dead Letter │
                                         │  Queue (DLQ) │
                                         │  14-day hold │
                                         └─────────────┘
```

| Component | Details |
|-----------|---------|
| **SNS Topic** | `intraworx-cs-notifications` |
| **SQS Queue** | 7-day retention, 60s visibility timeout, long polling |
| **Dead Letter Queue** | 14-day retention, triggers after 3 failed receives |
| **Lambda Processor** | Node.js 20.x, batch size 10, processes SQS → sends email via SES |
| **SES Domain** | `intraworx.cloud` with DKIM verification |
| **From Addresses** | `noreply@intraworx.cloud`, `notifications@intraworx.cloud`, `busybee@intraworx.cloud` |

---

## 10. External Integrations

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     INTRAWORX PLATFORM                                   │
│                                                                          │
│  Core (all services) ────┬──── Employees Service (team member data)      │
│                          └──── Google OAuth via Cognito                  │
│                                                                          │
│  Employees Service ──────┬──── HubSpot CRM (client data sync)           │
│                          └──── Freshservice (IT ticketing)               │
│                                                                          │
│  DailyFlash ─────────────┬──── Google Sheets (report data)              │
│                          └──── HubSpot CRM (deal pipeline)              │
│                                                                          │
│  ZimStat ────────────────────── HubSpot CRM (client demographics)       │
│                                                                          │
│  BusyBee ERP ────────────┬──── Employees Service (team member data)     │
│                          ├──── Supabase (database + realtime)           │
│                          └──── HubSpot CRM (client sync)                │
│                                                                          │
│  TapCard (IoT) ──────────────── ESP32 → Shuttle Management (RFID taps)  │
│                                                                          │
│  PayrollGuard ───────────────── PowerShell Sync Client → S3             │
│                                                                          │
│  Platform-wide ──────────────── AWS SES (transactional email)           │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 11. Disaster Recovery Topology

DR infrastructure is designed for **eu-west-1 (Ireland)** and can be activated when needed.

```
┌────────────────────────────┐              ┌────────────────────────────┐
│   PRIMARY: us-west-2       │              │   DR TARGET: eu-west-1     │
│   (Oregon)                 │              │   (Ireland)                │
│                            │              │                            │
│  RDS PostgreSQL ───────────┼── replication ──▶ Cross-region read replica│
│                            │              │                            │
│  S3 Buckets ───────────────┼── replication ──▶ Replicated S3 buckets   │
│                            │              │                            │
│  KMS Keys ─────────────────┼─────────────────▶ DR-region KMS keys      │
│                            │              │                            │
│  Terraform code ───────────┼─────────────────▶ Reproducible infra      │
│                            │              │                            │
└────────────────────────────┘              └────────────────────────────┘

Current state: DR replication is designed but disabled for cost optimization.
Active protections: 7-day automated RDS backups, S3 versioning, Terraform reproducibility.
```

---

## Service Connection Summary

| From | To | Method | Purpose |
|------|----|--------|---------|
| All users | Core | HTTPS (browser) | Single sign-on entry point |
| Core | Cognito → Google | OAuth 2.0 | Authentication |
| Core | External services | JWT (URL param) | Token pass-through for SSO |
| Core modules | Core API | REST API | Business logic and data |
| Core API | Employees | REST API | Team member data |
| Wellness Center, Shuttle, The Grind, Seating Map | Employees | REST API + JWT | Team member lookup |
| BusyBee | Employees | REST API | Team member data |
| BusyBee | Supabase | HTTPS | Database + real-time |
| TapCard ESP32 | Shuttle Management | HTTPS (SHA256 hash) | RFID boarding events |
| PayrollGuard Sync Client | AWS S3 | HTTPS (AES-256) | Payroll backup uploads |
| All services | RDS PostgreSQL | TLS (port 5432) | Data persistence |
| Multiple services | Redis | TLS (port 6379) | Cache + task queues |
| All services | S3 | HTTPS (AWS SDK) | File storage |
| Lambda Processor | AWS SES | AWS SDK | Email delivery |
| DailyFlash | Google Sheets, HubSpot | HTTPS API | Report data ingestion |
| ZimStat, BusyBee, Employees | HubSpot CRM | HTTPS API | Client data sync |

| CloudWatch, GuardDuty | SNS → Email | AWS internal | Alerts to `devops@zimworx.com` |

---

**Document created by:** Tinashe Zvihwati
**Source:** AWS Terraform configurations, service repositories, and platform architecture documentation
