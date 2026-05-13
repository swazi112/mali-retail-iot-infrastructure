# Regional IoT & Retail Infrastructure — Mali & Africa

> Digitizing gas station operations and fuel monitoring across remote West African regions through offline-resilient software and low-power IoT telemetry.

---

## Executive Summary

This project delivers a **full-stack digital transformation** for gas station retail networks operating across Mali and West Africa. It addresses two operational pain points that cost the industry millions annually:

1. **Fuel loss visibility** — Underground tank levels were checked manually, often weekly. Theft, leaks, and accounting discrepancies went undetected for days.
2. **Retail inefficiency** — Gas station convenience stores (boutiques) managed pricing, promotions, and stock via paper lists and phone calls, with no centralized oversight.

The platform replaces both with a unified, cloud-managed system: an **IoT Smart Station** backend for continuous remote fuel monitoring, and a **Progressive Web App (PWA)** for retail catalog and promotion management — both engineered to operate reliably on the intermittent 2G/Edge cellular networks typical of the region.

**Key ROI drivers:**
- Fuel shrinkage detected in **minutes** instead of days
- Retail catalog updates propagated to **all stores instantly** instead of per-store manual distribution
- **Zero hardware cost** for the boutique app — installs from the browser on any smartphone
- **No on-premise servers** required — fully managed cloud infrastructure

---

## The Challenge

### Infrastructure Constraints

Gas stations in Mali and across francophone West Africa operate under conditions that invalidate most standard SaaS assumptions:

- **Connectivity** — Rural stations rely on 2G/Edge cellular. Connections drop frequently, bandwidth is metered, and latency is high. A cloud-first, always-online architecture fails here.
- **Power** — Grid electricity is unreliable. IoT sensors run on solar with limited battery reserves. Every byte transmitted and every CPU cycle on the device consumes scarce power.
- **Hardware access** — Staff use low-end Android smartphones. There is no budget for tablets, POS terminals, or dedicated devices. App Store distribution adds friction for non-technical users.
- **Geography** — Stations are spread across hundreds of kilometers. Physical inspection trips are expensive (fuel, vehicles, staff time) and infrequent.
- **Literacy & training** — The solution must be immediately usable with minimal onboarding. Complex multi-step workflows are not viable.

### Business Requirements

| Requirement | Constraint |
|---|---|
| Fuel monitoring | Must work with ultrasonic **and** hydrostatic pressure sensors across different tank geometries |
| Theft detection | Alerts must reach managers within minutes, not hours |
| Boutique catalog | Must support thousands of SKUs with hierarchical categories |
| Promotions | Weekly/monthly/special campaigns with automatic start/end dates |
| Multi-store | Single admin manages pricing and availability across all locations |
| Offline access | Core app functionality must survive network outages |
| Deployment | No app store. No APK sideloading. Browser-based install only |

---

## Technical Architecture

### System Overview

```
┌─────────────────────────────────────────────────────────┐
│                    FIELD LAYER                          │
│                                                         │
│  ┌──────────────┐    ┌──────────────┐                  │
│  │ Tank Sensor  │    │ Tank Sensor  │   ...             │
│  │ (Ultrasonic) │    │ (Pressure)   │                  │
│  └──────┬───────┘    └──────┬───────┘                  │
│         │ GPRS/2G          │ GPRS/2G                   │
└─────────┼──────────────────┼───────────────────────────┘
          │                  │
          ▼                  ▼
┌─────────────────────────────────────────────────────────┐
│                   INGESTION LAYER                       │
│                                                         │
│  ┌──────────────────────────────────────────────┐      │
│  │  NestJS API — /sensors/ingest                │      │
│  │  • Payload validation                        │      │
│  │  • SMA noise filtering                       │      │
│  │  • Gauging engine (volume calculation)        │      │
│  │  • Anomaly detection (delta threshold)        │      │
│  └──────────────────┬───────────────────────────┘      │
│                     │                                   │
│         ┌───────────┼───────────┐                      │
│         ▼           ▼           ▼                      │
│    Socket.io    Web Push    CRON Reports               │
│   (Real-time)   (Alerts)   (Scheduled)                 │
└─────────────────────────────────────────────────────────┘
          │
          ▼
┌─────────────────────────────────────────────────────────┐
│                  APPLICATION LAYER                       │
│                                                         │
│  ┌─────────────────┐    ┌───────────────────────┐      │
│  │  Smart Station  │    │  ColiEnergy Boutique  │      │
│  │  Dashboard      │    │  PWA                  │      │
│  │  (IoT Monitor)  │    │  (Retail Catalog)     │      │
│  │                 │    │                       │      │
│  │  React + Vite   │    │  React + Vite         │      │
│  │  Zustand        │    │  TanStack Query       │      │
│  │  Socket.io      │    │  Supabase             │      │
│  └─────────────────┘    └───────────────────────┘      │
└─────────────────────────────────────────────────────────┘
```

### Smart Station — IoT Monitoring

The Smart Station subsystem continuously monitors underground fuel tanks at remote gas stations.

**Sensor Ingestion Pipeline:**
- Sensors transmit compact telemetry packets over GPRS: `fuel level (mm)`, `RSSI (dBm)`, `battery level (V)`
- A **Simple Moving Average (SMA)** filter smooths readings over the last N samples to suppress turbulence noise during fill and discharge operations
- Volume is calculated from raw level readings using a **Strategy Pattern** engine:
  - *Horizontal Cylinder Strategy* — Circular segment formula for standard underground tanks
  - *Hydrostatic Pressure Strategy* — Converts mBar readings to fluid height for pressure-based probes, then delegates to the cylinder formula
- New tank geometries are supported by adding a strategy class — no pipeline modifications required

**Anomaly Detection:**
- Each new volume reading is compared against the last known value
- If the delta exceeds a configurable threshold (e.g., 50L) within a configurable time window (e.g., 5 minutes), the system triggers:
  - A **real-time WebSocket event** to the monitoring dashboard
  - A **Web Push notification** to the field manager's phone (delivered even when the browser is closed)
- **Scheduled CRON reporting** dispatches periodic summary notifications (all tank volumes + battery status) at user-configured intervals

**Connectivity Resilience:**
- RSSI telemetry per reading enables proactive detection of degraded cellular links
- Battery monitoring prevents silent data outages at solar-powered stations
- The monitoring PWA caches its shell and last-known state for offline access

### Boutique App — Retail Operations

The ColiEnergy PWA digitizes gas station convenience store management.

**Core Capabilities:**
- **Product Catalog** — Hierarchical category/subcategory navigation, full-text search by name, SKU, and barcode
- **Promotion Engine** — Time-bound promotional catalogues (weekly, monthly, special) with automatic activation/expiration. Products display original and discounted pricing with percentage badges
- **Multi-Store Management** — Per-store product availability tracking (`in_stock` / `low_stock` / `out_of_stock`). Staff select their active store; managers switch between locations
- **Bulk CSV Import** — Thousands of products onboarded via a single file upload. Categories and subcategories are auto-created. Idempotent — duplicate SKUs are skipped
- **Admin Dashboard** — Aggregate KPIs across all stores: total products, categories, active stores, live promotions

**Offline-First Architecture:**
- Workbox Service Worker pre-caches the full app shell (HTML, JS, CSS, fonts, icons)
- API responses are cached with a `NetworkFirst` strategy — fresh data when online, cached data when offline
- Cache TTL of 24 hours with a 100-entry cap balances freshness against storage on low-end devices
- PWA installs directly from the browser — no app store required, no download size concerns

**Data Layer:**
- Supabase (managed PostgreSQL) with Row-Level Security enforced at the database layer
- Public read access for catalog browsing; authenticated admin access for data management
- Targeted indexes on high-cardinality columns ensure sub-100ms query times at scale

---

## Impact

### Operational Efficiency

| Area | Before | After |
|---|---|---|
| Fuel inventory checks | Manual, requires travel to site, weekly at best | Continuous automated monitoring, every sensor reading |
| Theft / leak detection | Discovered during monthly reconciliation (days to weeks) | Real-time alert within minutes of anomaly |
| Product pricing updates | Paper lists distributed per store via field staff | Centralized update, instantly available at all locations |
| Promotion management | Verbal communication, inconsistent execution | Automated start/end dates, uniform across all stores |
| New product onboarding | Manual data entry, one product at a time | Bulk CSV import — thousands of products in seconds |
| App deployment to staff | Not applicable (no digital tools existed) | Browser-based PWA install — zero friction, zero cost |
| Order placement to suppliers | Phone calls, paper forms, no tracking | Structured digital workflow with status lifecycle |

### Cost Reduction

- **Eliminated inspection travel** — No more vehicle trips to check tank levels at remote stations
- **Reduced fuel shrinkage** — Minute-level anomaly detection recovers losses that previously went unnoticed for days
- **Zero per-device licensing** — PWA replaces per-seat licensed POS or inventory software
- **No on-premise IT** — Fully managed cloud backend (Supabase + NestJS) removes the cost of local server maintenance, UPS systems, and IT support staff in the field
- **Reduced training overhead** — Familiar mobile-browser interface requires minimal staff onboarding

### Data-Driven Decision Making for Regional Managers

- **Real-time regional dashboards** aggregating fuel levels, battery health, and connectivity status across all stations in a territory
- **Proactive maintenance alerts** — Battery degradation and RSSI signal drops are flagged before they cause data gaps
- **Retail performance visibility** — Promotion adoption, product catalog breadth, and store activity metrics available centrally
- **Configurable reporting cadence** — Managers choose how frequently they receive automated status summaries via push notification

---

## Technology Stack

| Component | Technology | Why |
|---|---|---|
| IoT Backend | **NestJS** + Prisma ORM + Socket.io + Web Push | Type-safe, modular architecture with real-time and notification capabilities built-in |
| Boutique Backend | **Supabase** (PostgreSQL + Auth + Edge Functions) | Managed infrastructure with database-level security policies — zero server ops |
| Boutique Frontend | **React** + Vite + TypeScript + TanStack Query | Fast PWA builds, intelligent client-side caching, type safety |
| IoT Dashboard | **React** + Vite + Zustand + Socket.io Client | Lightweight state management for real-time data streams |
| Supply Chain | **Next.js** + Prisma + SQLite | Server-rendered B2B ordering portal, embeddable database for portable deployment |
| Offline Layer | **Workbox** (Service Worker) | Fine-grained caching strategies tuned for low-bandwidth networks |
| UI Framework | **Tailwind CSS** + shadcn/ui + Radix Primitives | Consistent, accessible, mobile-first component library |

---

## Confidentiality Notice

> **⚠️ This repository is a public case study and does not contain proprietary source code.**
>
> The software described in this document was developed under a Non-Disclosure Agreement (NDA). All source code, database schemas, API specifications, sensor communication protocols, and deployment configurations remain the intellectual property of the commissioning organization and are housed in private repositories.
>
> This case study is published with authorization for the purpose of demonstrating technical architecture, engineering methodology, and business impact at a conceptual level.
>
> **For technical architecture walkthroughs, live demonstrations, or partnership inquiries, please contact:**
>
> 📧 **sassikarim11@gmail.com**
> 🔗 **[Karim Sassi — LinkedIn](https://www.linkedin.com/in/karim-sassi-581663252/)**

---

## License

This case study document is published under [CC BY-NC-ND 4.0](https://creativecommons.org/licenses/by-nc-nd/4.0/). You may share it with attribution, but you may not modify it or use it commercially.

---

<p align="center">
  <strong>Built by ATS</strong> — Connected infrastructure for West Africa
</p>
