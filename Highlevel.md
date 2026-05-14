# IntraWorX Platform Overview
## A Plain-Language Guide for Domain Experts

---

## What is IntraWorX?

**IntraWorX is a unified digital workplace and operations platform**—the central hub where employees, managers, and teams access, manage, and monitor all business services, workflows, and data in one place.

 It provides real-time stats, role-based dashboards, and direct access to every operational module—facilities, client success, wellness, philanthropy, admin, and more. Instead of 15 disconnected systems, IntraWorX is **one connected ecosystem** where everything works together, and every user sees what matters most to them.

---

## The Big Picture

```
                         ┌──────────────────────────────────────────────┐
                         │   IntraWorX Auth Portal & Operations Hub     │
                         │   (Your main dashboard for everything)       │
                         └──────────────────────────────────────────────┘
                                        │
        ┌───────────────────────────────┼───────────────────────────────┐
        │                               │                               │
        ▼                               ▼                               ▼
┌───────────────┐             ┌─────────────────┐             ┌─────────────────┐
│   PEOPLE      │             │   WORKPLACE     │             │   BUSINESS      │
│   OPERATIONS  │             │   SERVICES      │             │   INTELLIGENCE  │
├───────────────┤             ├─────────────────┤             ├─────────────────┤
│ • Employee    │             │ • Facilities    │             │ • Sales Metrics │
│   Directory   │             │ • Shuttles      │             │ • Client Maps   │
│ • Onboarding  │             │ • Cafeteria     │             │ • Dashboards    │
│ • Attendance  │             │ • Seating       │             │ • AI Support    │
│ • Wellness    │             │ • Documents     │             │                 │
│ • Philanthropy│             │                 │             │                 │
└───────────────┘             └─────────────────┘             └─────────────────┘
```

---

## What Each System Does (In Plain English)

### 🏢 INTRAWORX AUTH PORTAL & OPERATIONS HUB

| System | What It Does | Who Uses It |
|--------|--------------|-------------|
| **IntraWorX Auth Portal** | The main dashboard and operations hub. Provides single sign-on, but also lets users access, manage, and monitor all business modules: facilities, client success, wellness, philanthropy, admin, and more. Real-time stats, role-based dashboards, and direct service access. | Everyone |

### 👥 PEOPLE OPERATIONS

> **Note:** IntraWorX does not include a core HRIS (like Workday or BambooHR). These systems handle operational workflows and employee data—not compensation, benefits, performance reviews, or core HR records.

| System | What It Does | Who Uses It |
|--------|--------------|-------------|
| **Employees Service** | The employee directory and data service. Stores names, departments, managers, skills, equipment issued, documents. Every other system pulls employee data from here—but it's not a full HRIS. | HR, Managers, IT |
| **BusyBee** | Operational HR workflows—attendance tracking, PTO requests, client management, sales reporting, and exit workflows. Handles day-to-day HR operations, not core HR records. | HR, Operations, Sales |
| **TeamEase** | Handles onboarding and offboarding checklists. When someone joins, it coordinates IT setup, facilities access, equipment, and orientation. When they leave, it manages returns and access removal. | HR, Facilities, IT |
| **PayrollGuard** | Secure backup and audit system for payroll data. Creates tamper-proof records of who accessed what, when—for compliance and peace of mind. | Finance, HR, Compliance |
| **Wellness Center Clinic (WCC)** | The company clinic portal—appointment scheduling, medical records, prescriptions, sick notes. Built to healthcare data standards (HIPAA-compliant). | Clinic Staff, Employees |

### 🏢 WORKPLACE SERVICES

| System | What It Does | Who Uses It |
|--------|--------------|-------------|
| **Shuttle Management** | Manages company transportation—routes, schedules, drivers, vehicles, who boarded, and billing. Employees tap their card, the system records their ride. | Facilities, Drivers, Finance |
| **TapCard** | The physical card readers on shuttles and at entry points. When employees tap their badge, this system captures it and sends it to the appropriate service. | Automatic (IoT devices) |
| **The Grind** | Cafeteria point-of-sale and inventory system. Employees order food, pay (or charge to their account), and management tracks inventory, waste, and costs. | Cafeteria Staff, Employees |
| **Automated Seating Map** | Visual floor plans showing who sits where. Tracks vacant desks, equipment locations, and helps with workspace planning and moves. | Facilities, HR, Managers |
| **CheetahHub** | Secure document portal connected to company Google Drive. Find and preview documents without leaving the platform. | Everyone |

### 📊 BUSINESS INTELLIGENCE

| System | What It Does | Who Uses It |
|--------|--------------|-------------|
| **DailyFlash** | Sales dashboard that tracks seats sold, revenue targets, sales rep performance, and forecasts. Syncs with HubSpot for real-time data. | Sales, Leadership |
| **ZimStat** | Geographic visualization of clients and team members on a map. See at a glance where your business footprint is and where opportunities exist. | Leadership, Client Success |
| **Buzz AI** | AI-powered support assistant that answers questions by searching your company's knowledge base (Freshservice articles). Employees get instant answers 24/7. | Everyone |

---

## How It All Connects

### The Employee Journey Example

**Day 1 - Sarah joins the company:**
1. **HR creates her record** in Employees Service
2. **TeamEase kicks off onboarding**—IT gets notified to set up her laptop, Facilities assigns her a desk
3. **Automated Seating Map** shows her assigned to Desk 7B on Floor 8
4. **Shuttle Management** gives her access to the company bus route
5. **The Grind** creates her cafeteria account with a monthly food stipend
6. **Auth Portal** gives her one login that works for everything

**Day 30 - Sarah's daily routine:**
- She **taps her card** (TapCard) to board the shuttle → recorded in Shuttle Management
- She **checks the map** to find a meeting room → Automated Seating Map
- She **buys lunch** at the cafeteria → The Grind deducts from her account
- She **asks a question** about leave policy → Buzz AI answers instantly
- She **requests PTO** → BusyBee routes it to her manager

**Day 365 - Sarah needs the clinic:**
- She **books an appointment** → Wellness Center Clinic
- Her **medical record** stays private and audited → WCC (HIPAA-compliant)
- She gets a **sick note** emailed automatically

---

## How It's Built (Without the Tech Jargon)

### Think of it like a city:

| Concept | What It Means |
|---------|---------------|
| **Cloud-Based** | Everything runs on secure servers managed by Amazon (AWS). No physical servers in the office to maintain. |
| **Modular Design** | Each system (shuttle, cafeteria, HR) is independent but connected. If one needs an update, the others keep running. |
| **Single Sign-On** | One username and password gets you into everything. No remembering 15 different passwords. |
| **Real-Time Sync** | When data changes in one system, others see it immediately. Add someone to HR → they appear in every system. |
| **Mobile-Ready** | Works on phones, tablets, and computers. No special software to install. |
| **Audit Trails** | Every action is logged. We can always answer "who did what, when" for compliance. |
| **Secure by Design** | Sensitive data (payroll, medical) is encrypted. Access is role-based—you only see what you need. |

### The Foundation (Infrastructure):

```
┌────────────────────────────────────────────────────────────┐
│                    AMAZON WEB SERVICES (AWS)               │
├────────────────────────────────────────────────────────────┤
│  • Databases: Where all the data lives (encrypted)        │
│  • Compute: The "brains" that run the applications        │
│  • Storage: Where documents and files are kept            │
│  • Security: Firewalls, threat detection, access control  │
│  • Monitoring: 24/7 watching for problems                 │
│  • Backup: Daily copies for disaster recovery             │
└────────────────────────────────────────────────────────────┘
```

---

## Why This Matters


### With IntraWorX:
- ✅ One connected platform
- ✅ Single login for everything
- ✅ Data flows automatically
- ✅ Real-time dashboards and reports
- ✅ Digital, auditable processes

---

## Common Questions

**Q: Is this like Workday or BambooHR?**
> No—IntraWorX is **not an HRIS**. It doesn't manage compensation, benefits, performance reviews, or core HR records. It's an **operations platform** that handles workplace services, workflows, and employee data sharing. Think of it as the layer that makes day-to-day work easier, while your HRIS (once available) manages the official HR records.

**Q: What if one system goes down?**
> Each system is independent. If the the system has issues, HR and shuttles keep running.

**Q: Who can see what?**
> Role-based access. HR sees HR data. Finance sees finance data. Employees see their own records. Everything is logged.

**Q: Is our data safe?**
> Enterprise-grade security: encryption, backups, access controls, audit logs, threat detection. Hosted on AWS (same infrastructure banks use).

**Q: Can we add new features?**
> Yes. The modular design means we can add new systems or features without rebuilding everything.

---

## System Ownership Quick Reference

| Area | Systems | Primary Stakeholders |
|------|---------|---------------------|
| People Operations | Employees Service, BusyBee, TeamEase, PayrollGuard | HR, People Ops |
| *Note: No core HRIS* | *Compensation, benefits, performance reviews managed elsewhere* | *—* |
| Facilities | Shuttle, TapCard, Seating Map, CheetahHub | Facilities, Operations |
| Food Services | The Grind | Cafeteria, Finance |
| Health | Wellness Center Clinic | Clinic Staff, HR |
| Sales & BI | DailyFlash, ZimStat | Sales, Leadership |
| Support | Buzz AI | IT, HR, Everyone |

---

## Summary

**IntraWorX is your organization's digital operating system.** It connects people, places, and processes into one seamless experience—making work easier for employees and operations more visible for managers.

Every system was built to solve a real problem, and together they create a workplace where:
- Information flows automatically
- Processes are digital and auditable
- People can focus on their work, not on navigating systems

---
