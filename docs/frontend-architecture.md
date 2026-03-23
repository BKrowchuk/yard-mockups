# Frontend Architecture

## Overview

The Yard App frontend is a **Progressive Web App (PWA)** built as an offline-first mobile application. It connects to the Compass backend API and is optimized for use on phones and tablets in a harsh industrial yard environment with unreliable connectivity.

## Technology Stack

| Layer | Technology | Rationale |
|---|---|---|
| **Framework** | React or Vue (TBD) | Component-based, PWA ecosystem support |
| **State Management** | IndexedDB (Dexie.js) + in-memory store | Offline-first persistent cache |
| **Networking** | Service Worker + Workbox | Background sync, cache strategies |
| **Scanning** | Web Barcode Detection API / ZXing-js | Camera-based barcode/QR scanning |
| **Bundler** | Vite | Fast builds, PWA plugin |
| **Styling** | Tailwind CSS | Rapid mobile-first UI |

> **From meetings:** The app will be a PWA installable via Safari/Chrome to bypass App Store restrictions. Users prefer phones over tablets. The app must be fast, intuitive, and visually clear with standardized color codes for statuses.

## Application Structure

```
src/
  app/
    App.tsx                 # Root component, router, online/offline provider
    router.ts               # Route definitions
  pages/
    Home/                   # Dashboard - quick actions, connectivity status
    Search/                 # Global search with filters
    ElementDetail/          # Element info, location path, movement history
    RackDetail/             # Rack info with element list
    Rebook/                 # Multi-step rebook/move flow
    LoadSheet/              # Load sheet viewer with triage
    LoadSheetScan/          # Barcode scan entry to load sheet
    LocationGrid/           # Visual grid of area > row > lane > container
    LocationManagement/     # Admin: create/edit/activate locations
    Scanner/                # Barcode/QR scanner overlay
    SyncStatus/             # Offline queue status and diagnostics
  components/
    layout/
      AppShell.tsx          # Nav bar, status bar, connectivity indicator
      BottomNav.tsx         # Mobile bottom navigation
    search/
      GlobalSearch.tsx       # Unified search bar with type filters
      SearchResults.tsx      # Result cards with location badges
      ElementCard.tsx        # Element summary card
      RackCard.tsx           # Rack summary card
    location/
      LocationTree.tsx       # Hierarchical location browser
      LocationPicker.tsx     # Drill-down picker for rebook destination
      OccupancyBadge.tsx     # Capacity vs occupancy indicator
    rebook/
      ElementSelector.tsx    # Multi-select element list with filters
      DestinationPicker.tsx  # Choose yard/rack/trailer destination
      ConflictResolver.tsx   # Show conflicts, offer retry/skip
      MoveConfirmation.tsx   # Summary before submit
    loadsheet/
      LoadSheetList.tsx      # Load sheets with date/status filters
      LoadSheetItems.tsx     # Item list with yard match badges
      TriagePanel.tsx        # Group items by OK/MISSING/NOT_IN_YARD
    scanner/
      BarcodeScanner.tsx     # Camera scanner with auto-detect
      ScanFeedback.tsx       # Success/failure toast after scan
    sync/
      ConnectivityBanner.tsx # Online/offline/syncing indicator
      SyncQueueViewer.tsx    # Pending operations list
      SyncProgress.tsx       # Upload progress during reconnect
    common/
      StatusBadge.tsx        # Color-coded status indicators
      FilterBar.tsx          # Reusable filter controls
      PullToRefresh.tsx      # Mobile pull-to-refresh
      EmptyState.tsx         # No results / loading states
  services/
    api/
      client.ts             # HTTP client with auth, retry, offline detection
      endpoints.ts          # Typed API endpoint functions
    cache/
      CacheManager.ts       # IndexedDB read/write operations
      DeltaSync.ts          # Incremental sync using updated_since
      CacheInvalidation.ts  # Staleness checks and eviction
    sync/
      SyncQueue.ts          # Offline operation queue (IndexedDB-backed)
      SyncEngine.ts         # Background sync processor
      ConflictDetector.ts   # Compare expected vs actual on submit
    scanner/
      BarcodeService.ts     # Camera access and code detection
  hooks/
    useOnlineStatus.ts      # Reactive online/offline state
    useSearch.ts            # Debounced search with cache fallback
    useSyncQueue.ts         # Queue operations and monitor status
    useLocationPicker.ts    # Hierarchical location selection
  db/
    schema.ts               # Dexie.js IndexedDB schema definition
    migrations.ts           # DB version migrations
  workers/
    sw.ts                   # Service worker (Workbox)
    sync-worker.ts          # Background sync web worker
  types/
    api.ts                  # API response types
    domain.ts               # Domain model types
    cache.ts                # Cache entry types
```

## Core User Flows

### 1. Home / Dashboard

The landing page provides quick access to primary actions and system status.

```
+----------------------------------+
|  YARD APP            [signal] [] |
|----------------------------------|
|  [Scan]  [Search]  [Rebook]     |
|                                  |
|  Today's Loads          3 pending|
|  Sync Status         All synced  |
|  Pending Moves              0    |
|                                  |
|  Recent Activity                 |
|  > HC-001 moved to Row A-3      |
|  > LS-2026-184 parsed (2 miss)  |
|  > R-B3-12 loaded on Trailer 7  |
|                                  |
|  [Home] [Search] [Loads] [More]  |
+----------------------------------+
```

### 2. Global Search

Unified search across elements, racks, projects, and locations.

- Type-ahead with debounce (300ms)
- Filter chips: Element | Rack | Project | Location
- Pre-filter by project/job code to reduce data volume
- Results show: code, status badge, location path, project
- Tap result to view detail or start rebook
- Duplicate underscore numbers collapsed with location disambiguation

> **From meetings:** Search is the #1 pain point. Current app searches millions of rows. The new app must pre-filter by project or location, paginate results, and provide instant feedback. Sorting by three-letter code + numeric panel number must be restored.

### 3. Rebook / Move Flow

Multi-step wizard:

```
Step 1: Select source (scan barcode, search, or browse location grid)
Step 2: Select elements (multi-select with select-all, filter by status)
Step 3: Choose destination (yard location, rack, or trailer)
Step 4: Review & confirm (show from > to, element count)
Step 5: Submit (or queue if offline)
Step 6: Result (success/conflict/partial with per-item status)
```

- If offline: batch is queued in IndexedDB, submitted on reconnect
- Conflict resolution: show which elements moved since selection, offer skip/retry
- Loading a trailer = special move type that marks elements as delivered
- "Load" action type is separate from "store" and "rebook"

> **From meetings:** Booking destinations are three types: yard location, half rack, or trailer. Booking a rack updates all its elements simultaneously. The system should show live status with success/failure notifications and retry options.

### 4. Load Sheet Viewer

- Scan load sheet barcode to open
- Items grouped by yard match status: OK (green), MISSING (red), NOT_IN_YARD (orange), UNKNOWN (grey)
- Each item shows: element code, location path, sequence number
- Select/deselect individual items or bulk select
- Action: rebook selected items, mark as loaded

### 5. Location Grid

Visual representation of the location hierarchy for browsing and occupancy overview.

```
Yard West
  Row A  [||||||||..] 80%    Row B  [||||||....] 60%
  Row C  [||||......] 40%    Row D  [||........] 20%

Staging West
  Staging [|||.......] 30%
```

### 6. Scanner

- Auto-launches from home screen tap
- Supports barcode and QR in any orientation
- Resolves to element, rack, container, or load sheet
- Shows confirmation toast with entity summary
- On success: navigates to detail page or adds to current selection

## Data Flow & Caching Strategy

### Cache Layers

```
┌─────────────────┐
│   UI Components  │  in-memory reactive state
├─────────────────┤
│   Cache Manager  │  read-through to IndexedDB
├─────────────────┤
│    IndexedDB     │  persistent offline store (Dexie.js)
├─────────────────┤
│   Service Worker │  HTTP cache (Workbox)
├─────────────────┤
│   Compass API    │  source of truth
└─────────────────┘
```

### What Gets Cached Offline

| Data | Cache Strategy | TTL | Size Estimate |
|---|---|---|---|
| Location tree | Cache-first, delta sync | 1 hour | ~500 KB |
| Elements (active project) | Stale-while-revalidate | 15 min | ~2-5 MB |
| Racks | Stale-while-revalidate | 15 min | ~200 KB |
| Load sheets (today/tomorrow) | Network-first | 5 min | ~100 KB |
| App shell / assets | Cache-first (precached) | Until deploy | ~2 MB |
| Scan history | Local-only | 7 days | ~50 KB |
| Sync queue | Local-only | Until acked | Variable |

> **From meetings:** The app should cache 5-10 minutes of essential data (element IDs, locations) for offline use. No large text blobs or images. Total offline cache should stay in the low megabytes. The app should restrict functionality when offline beyond set limits to prevent stale operations.

### Delta Sync Strategy

On reconnect or periodic refresh:
1. Call `GET /sync/delta?since={last_sync_timestamp}&tables=elements,containers`
2. Merge updated records into IndexedDB
3. Invalidate stale in-memory state
4. Process any queued operations via `POST /sync/batch`

### Cache Invalidation Rules

- Location tree: refresh when user opens location management or every 1 hour
- Elements: refresh when user searches or opens detail (background revalidation)
- After a move/rebook: immediately invalidate affected elements and containers
- On reconnect: full delta sync before allowing new operations

## Connectivity Handling

### States

| State | UI Indicator | Behavior |
|---|---|---|
| **Online** | Green dot | Normal API calls, background delta sync |
| **Slow** | Yellow dot | Longer timeouts, show stale data with warning |
| **Offline** | Red dot + banner | Read from cache, queue writes, limit to 15-20 min |
| **Syncing** | Pulsing blue | Processing queued operations, show progress |

> **From meetings:** Users must be trained on connectivity limits. The app will restrict functionality offline beyond ~15 minutes. Hot zones with strong Wi-Fi at critical areas ensure key operations stay online. The PWA must handle seamless switching between Wi-Fi and 5G networks.

### Offline Restrictions

When offline longer than the configured threshold (default 15 min):
- Show warning: "Data may be stale. Connect to sync."
- Disable new move/rebook operations (queued data too old for reliable conflict detection)
- Allow read-only browsing of cached data
- Keep existing queue intact for future sync

## UI/UX Guidelines

### Color Coding (Status System)

| Status | Color | Badge |
|---|---|---|
| Confirmed / OK | Green | Solid circle |
| Queued / Pending | Blue | Hollow circle |
| Loaded | Blue | Solid square |
| Conflict | Red | Exclamation |
| Missing | Red | X mark |
| Not In Yard | Grey | Dash |
| Ready for Patch | Orange | Half circle |
| Ready for Windows | Purple | Diamond |
| Delivered | Dark green | Checkmark |

### Mobile-First Design Principles

- Touch targets minimum 44px
- Bottom navigation for primary actions (Home, Search, Loads, More)
- Scanner accessible from every screen via floating action button
- Pull-to-refresh on all list views
- Skeleton loading states (never blank screens)
- Toast notifications for scan results and sync events
- Confirmation dialogs for destructive actions (mark as delivered, force complete)

### Performance Targets

| Metric | Target |
|---|---|
| First Contentful Paint | < 1.5s |
| Time to Interactive | < 3s |
| Search results (cached) | < 200ms |
| Search results (network) | < 1s |
| Barcode scan to result | < 500ms |
| Offline page load | < 500ms |

> **From meetings:** Speed is the #1 user complaint. The current app reloads data at every click. The new app must feel instant through aggressive caching and optimistic UI updates. Fast loading and responsiveness are prerequisites for user adoption.
