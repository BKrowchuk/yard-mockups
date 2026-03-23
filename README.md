# Yard Management App

A Progressive Web App (PWA) for tracking precast elements, racks, and loadsheets across the yard. Replaces the legacy i2 yard system with an offline-first, mobile-optimized application built on the Compass backend.

## Problem

Yard workers at Stubbe's precast plants manage thousands of elements across rows, racks, and staging areas. The current system suffers from:

- **Slow, unreliable app** - reloads data on every click, frequent crashes
- **Poor connectivity** - Wi-Fi dead zones, cellular gaps, concrete-heavy buildings blocking signal
- **No offline support** - workers revert to paper notes and phone calls when disconnected
- **Data fragmentation** - multiple systems (i2, Compass, spreadsheets) with manual data entry
- **Low adoption** - speed and usability issues cause workers to avoid the app entirely

This results in ~$35,000/week in wasted panels from tracking errors and operational inefficiencies.

## Solution

An offline-first PWA that:
- Caches essential data locally for 15-30 min offline operation
- Queues writes (rebook, scan, status changes) and syncs on reconnect
- Provides fast search with project/location pre-filtering
- Supports barcode/QR scanning for instant element and loadsheet lookup
- Runs on phones (preferred by workers) without App Store dependency
- Connects to Compass backend as the single source of truth

## Users

| Role | Count | Primary Actions |
|---|---|---|
| Yard Workers | ~8-10 | Rebook/move elements, scan barcodes, find panels |
| QC Inspectors | ~6-7 | Check statuses, verify locations, update element state |
| Planners (Jake, etc.) | 2-3 | Load sheet management, delivery planning, search |
| Supervisors | 2-3 | Location management, reporting, oversight |
| **Total** | **~20** | |

## Architecture Overview

```
┌─────────────────────────────┐     ┌──────────────────────┐
│     PWA (Phone/Tablet)       │     │   Compass Backend    │
│                              │     │                      │
│  React/Vue + Tailwind        │────▶│  REST API            │
│  IndexedDB (Dexie.js)        │◀────│  PostgreSQL          │
│  Service Worker (Workbox)    │     │  17 tables           │
│  Barcode Scanner             │     │                      │
│  Sync Queue                  │     │  Integrations:       │
│                              │     │  - i2 (retiring)     │
└─────────────────────────────┘     │  - IPBS              │
                                     │  - NetSuite          │
                                     └──────────────────────┘
```

### Key Design Decisions

| Decision | Rationale |
|---|---|
| **PWA over native** | Bypass App Store, single codebase, instant updates |
| **Offline-first** | Connectivity is unreliable; app must work in dead zones |
| **Compass backend** | Existing platform, shared auth (SSO), project/element data |
| **IndexedDB + Service Worker** | Persistent cache survives app close, background sync |
| **Forced sync on reconnect** | Prevents stale data from causing double-bookings |
| **Conflict detection via expected_container_id** | Catches moves that happened while user was offline |

## Core Features

### Phase 1 (MVP)
- Global search (element, rack, project, location)
- Element detail with location path and status
- Rebook/move single and multi-element
- Location hierarchy browsing (area > row > lane > container)
- Barcode/QR scanning
- Offline data caching and sync queue
- Connectivity status indicators

### Phase 2
- Load sheet scanning and triage
- Move batch conflict detection and resolution
- Load sheet digital replacement (paper elimination)
- Location management (create, rename, activate/deactivate)
- Rack tracking with element association

### Phase 3
- Dispatch planning ("what loads next" ordering)
- Delivery confirmation and status automation
- IPBS integration for production milestone sync
- Rack return tracking
- Audit log viewer
- Advanced reporting and analytics

## Documentation

| Document | Description |
|---|---|
| [Database Schema](docs/database-schema.md) | 17-table PostgreSQL schema: location hierarchy, inventory, operations, offline sync |
| [Canonical Schema Definition](location-management-db-table-design.md) | Full table definitions with columns, types, constraints, and indexes |
| [Backend API](docs/backend-api.md) | REST API endpoints for locations, search, moves, scanning, loadsheets, sync |
| [Frontend Architecture](docs/frontend-architecture.md) | Component structure, data flow, caching layers, user flows, UI/UX guidelines |
| [Offline & Sync Strategy](docs/offline-sync.md) | Sync queue, conflict detection, retry logic, delta sync, duration limits |
| [PWA Requirements](docs/pwa-requirements.md) | Service worker, manifest, install experience, camera, network handling |

## File Structure (Current: Wireframes)

```
wireframes/
├── README.md                              # This file
├── location-management-db-table-design.md # Canonical DB schema
├── docs/
│   ├── database-schema.md                 # DB architecture overview
│   ├── backend-api.md                     # API endpoint design
│   ├── frontend-architecture.md           # Frontend components & data flow
│   ├── offline-sync.md                    # Offline queue & sync strategy
│   └── pwa-requirements.md               # PWA technical requirements
├── index.html                             # Entry point (redirects to light mode)
├── dark/                                  # Dark mode wireframes
│   ├── home-dark.html
│   ├── element-grid-dark.html
│   ├── loadsheet-scan-dark.html
│   ├── location-management-dark.html
│   ├── rebook-dark.html
│   ├── search-dark.html
│   └── wireframes-dark.html
├── light/                                 # Light mode wireframes
│   ├── light-mode-ui-mockups.html         # Consolidated light mode mockups
│   └── archive/                           # Individual light mode pages
├── screenshots/
│   └── Yard Mockups/                      # UI screenshot captures
├── home.html                              # Home page wireframe
├── element-grid.html                      # Element grid wireframe
├── loadsheet-scan.html                    # Load sheet scan wireframe
├── location-management.html               # Location management wireframe
├── rebook.html                            # Rebook flow wireframe
├── search.html                            # Search wireframe
└── wireframes.html                        # Combined wireframes page
```

## File Structure (Target: Production App)

```
yard-app/
├── public/
│   ├── manifest.json                      # PWA manifest
│   ├── sw.js                              # Service worker (generated)
│   └── icons/                             # App icons (192, 512, maskable)
├── src/
│   ├── app/                               # Root app, router, providers
│   ├── pages/                             # Route-level page components
│   │   ├── Home/
│   │   ├── Search/
│   │   ├── ElementDetail/
│   │   ├── RackDetail/
│   │   ├── Rebook/
│   │   ├── LoadSheet/
│   │   ├── LocationGrid/
│   │   ├── LocationManagement/
│   │   ├── Scanner/
│   │   └── SyncStatus/
│   ├── components/                        # Reusable UI components
│   │   ├── layout/                        # App shell, nav
│   │   ├── search/                        # Search bar, results, cards
│   │   ├── location/                      # Tree, picker, occupancy
│   │   ├── rebook/                        # Element selector, destination picker
│   │   ├── loadsheet/                     # List, items, triage
│   │   ├── scanner/                       # Barcode scanner, feedback
│   │   ├── sync/                          # Connectivity banner, queue viewer
│   │   └── common/                        # Status badges, filters, empty states
│   ├── services/
│   │   ├── api/                           # HTTP client, endpoint functions
│   │   ├── cache/                         # IndexedDB manager, delta sync
│   │   └── sync/                          # Sync queue, engine, conflict detector
│   ├── hooks/                             # React hooks (online status, search, sync)
│   ├── db/                                # IndexedDB schema (Dexie.js)
│   ├── workers/                           # Service worker, sync worker
│   └── types/                             # TypeScript types
├── docs/                                  # Architecture documentation
└── wireframes/                            # UI mockups (this repo)
```

## Database Overview

17 PostgreSQL tables in 4 domains:

| Domain | Tables | Purpose |
|---|---|---|
| **Location Hierarchy** | `areas` > `rows` > `lanes` > `containers` | Normalized yard geography |
| **Inventory** | `projects`, `elements`, `racks` | Trackable items + current location |
| **Operations** | `move_batches`, `move_batch_items`, `element_movements`, `scan_events`, `loadsheets`, `loadsheet_items`, `dispatch_plans`, `dispatch_plan_items` | Movement history, scanning, load sheets, delivery |
| **Offline Sync** | `sync_queue`, `sync_attempts` | Queued operations with retry telemetry |

See [Database Schema](docs/database-schema.md) for full details and [canonical definitions](location-management-db-table-design.md) for table DDL.

## Offline Strategy (Summary)

```
Online → Normal API calls, background delta sync every 60s
Slow   → Longer timeouts, show cached data with freshness warning
Offline (0-15 min) → Full functionality, writes queued in IndexedDB
Offline (15-30 min) → Read-only, warning banner, queue preserved
Reconnect → Forced sync: drain queue, pull delta updates, then resume
```

Conflict detection uses `expected_container_id` to catch elements that moved while the user was offline. See [Offline & Sync Strategy](docs/offline-sync.md) for complete architecture.

## Connectivity Context

The yard has known Wi-Fi dead zones. The infrastructure team is testing:
- Directional Wi-Fi access points (U7 Outdoor)
- 5G cellular antennas mounted on forklifts
- Starlink satellite internet
- Wi-Fi "hot zones" at critical operational points

The app is designed to work independently of connectivity improvements. Even with perfect coverage, offline support remains essential for resilience.

## Timeline

| Phase | Target | Focus |
|---|---|---|
| Design & wireframes | Feb-Mar 2026 | UI mockups, DB schema, user feedback sessions |
| Development start | Mid-April 2026 | PWA scaffold, core search, rebook flow |
| MVP (Phase 1) | June 2026 | Search, rebook, scan, offline sync |
| Phase 2 | July-Aug 2026 | Load sheets, conflict resolution, location management |
| Phase 3 | Sep-Dec 2026 | Dispatch planning, IPBS integration, analytics |
| i2 retirement | Post-Phase 2 | Migrate remaining users to new app |

## Meeting Context

This architecture is informed by requirements gathered across multiple stakeholder sessions recorded in Fireflies:

| Date | Meeting | Key Topics |
|---|---|---|
| 2026-02-03 | Initial Yard App Discussion (Jake Teichroeb) | PWA decision, offline needs, user count, pain points |
| 2026-02-06 | Wireless Survey Review | Wi-Fi roaming issues, AP placement, directional antennas |
| 2026-02-19 | PL3 User Session | Connectivity issues, barcode scanning, status codes, rack tracking |
| 2026-02-19 | Yard User Session | Speed complaints, filtering needs, data sync conflicts |
| 2026-03-06 | Yard-Rielly Sprint Review | Search/rebook workflows, PWA + Compass backend, phased plan |
| 2026-03-06 | Stacks & Racks Discussion | Hierarchy (stacks/racks/trailers), load sheet integration |
| 2026-03-06 | Yard Connectivity Discussion | Wi-Fi vs 5G vs Starlink, offline duration, PWA network switching |
| 2026-03-10 | Yard Swapping Process | Swap rules, location-based swapping, load sheet visibility |
| 2026-03-18 | Inventory Data Quality | Live tracking, null locations, audit tracing, data cleanup timeline |
| 2026-03-18 | Yard Locations (Joseph) | Location access control, LD front/back split, ticketing system |
