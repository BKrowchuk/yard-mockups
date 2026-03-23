# Backend API

## Overview

The Yard App backend is served by the existing **Compass** platform. The Yard App PWA connects to Compass API endpoints for all data operations. The backend must support:

- RESTful endpoints for CRUD and search operations
- Bulk operations for multi-element rebook/move
- Offline queue acceptance (idempotent writes with dedupe keys)
- Real-time sync with i2 and IPBS systems
- Audit logging on all mutations

## Base URL & Auth

```
Base: https://{compass-host}/api/yard/v1
Auth: Bearer token (SSO - existing Compass auth)
```

All endpoints return JSON. Mutations accept JSON request bodies. Pagination uses `?page=1&per_page=50` with `Link` headers.

---

## Endpoint Groups

### 1. Location Hierarchy

Manage the area > row > lane > container tree.

| Method | Path | Description |
|---|---|---|
| `GET` | `/areas` | List all areas (optionally `?active=true`) |
| `POST` | `/areas` | Create area *(restricted - admin/ticketing)* |
| `PATCH` | `/areas/:id` | Update area (rename, activate/deactivate) |
| `GET` | `/areas/:id/rows` | List rows in area |
| `POST` | `/rows` | Create row |
| `PATCH` | `/rows/:id` | Update row |
| `GET` | `/rows/:id/lanes` | List lanes in row |
| `POST` | `/lanes` | Create lane |
| `PATCH` | `/lanes/:id` | Update lane |
| `GET` | `/lanes/:id/containers` | List containers in lane |
| `POST` | `/containers` | Create container |
| `PATCH` | `/containers/:id` | Update container |
| `GET` | `/locations/tree` | Full hierarchy tree (for offline cache prefetch) |
| `GET` | `/locations/tree?updated_since=ISO` | Delta sync - only changed nodes since timestamp |

> **From meetings:** Only designated users (e.g., Martin) can create locations. The API must enforce role-based access on POST endpoints. Location creation flows through a ticketing system in production, but the API should support direct creation for authorized users.

#### Location Tree Response (for offline cache)

```json
{
  "updated_at": "2026-03-23T12:00:00Z",
  "areas": [
    {
      "id": "uuid",
      "name": "Yard West",
      "active": true,
      "rows": [
        {
          "id": "uuid",
          "code": "A",
          "active": true,
          "lanes": [
            {
              "id": "uuid",
              "code": "1",
              "containers": [
                {
                  "id": "uuid",
                  "name": "Rack R-B3-12",
                  "container_type": "RACK",
                  "capacity": 20,
                  "occupancy": 14,
                  "active": true
                }
              ]
            }
          ]
        }
      ]
    }
  ]
}
```

### 2. Search

Unified global search across elements, racks, containers, and projects.

| Method | Path | Description |
|---|---|---|
| `GET` | `/search` | Global search: `?q=term&type=element,rack,project,location&project_id=&status=&limit=` |
| `GET` | `/elements` | List/filter elements: `?project_id=&status=&container_id=&rack_id=&q=&page=` |
| `GET` | `/elements/:id` | Element detail with full location path and movement history |
| `GET` | `/racks` | List/filter racks: `?project_id=&status=&container_id=&q=` |
| `GET` | `/racks/:id` | Rack detail with elements and location |
| `GET` | `/racks/:id/elements` | Elements on a specific rack |

> **From meetings:** Search must be fast - the current app searches millions of lines causing major performance issues. The API should support:
> - Pre-filtering by store location before loading data (reduces reload delays)
> - Filtering by next-day loads and project/job code
> - Collapsing duplicate underscore numbers with location details
> - Full-text search across element_code and description

#### Search Response

```json
{
  "results": [
    {
      "type": "element",
      "id": "uuid",
      "code": "24-10345-001",
      "description": "HC Panel 8\" x 4'",
      "status": "CONFIRMED",
      "project": { "id": "uuid", "code": "PRJ-101", "name": "Site Alpha" },
      "location": {
        "area": "Yard West",
        "row": "A",
        "lane": "3",
        "container": "Rack R-B3-12",
        "container_type": "RACK"
      },
      "rack": { "id": "uuid", "rack_code": "R-B3-12" }
    }
  ],
  "total": 142,
  "page": 1,
  "per_page": 50
}
```

### 3. Rebook / Move Operations

Multi-step workflow for moving elements between locations.

| Method | Path | Description |
|---|---|---|
| `POST` | `/moves` | Create move batch (draft or submit) |
| `GET` | `/moves/:id` | Get move batch status and per-item results |
| `POST` | `/moves/:id/submit` | Submit a draft batch for processing |
| `POST` | `/moves/:id/retry` | Retry failed items in a batch |

#### Move Request Body

```json
{
  "move_type": "REBOOK",
  "target_container_id": "uuid",
  "items": [
    {
      "element_id": "uuid",
      "expected_container_id": "uuid"
    }
  ],
  "device_id": "uuid",
  "dedupe_key": "device-uuid-timestamp"
}
```

#### Move Response (after processing)

```json
{
  "id": "uuid",
  "state": "PARTIAL_CONFLICT",
  "items": [
    { "element_id": "uuid", "result": "MOVED" },
    { "element_id": "uuid", "result": "CONFLICT", "error_message": "Element moved to Row B since selection" }
  ],
  "conflict_summary": "1 of 2 elements had conflicts"
}
```

> **From meetings:** Conflict detection is critical for offline scenarios. When a user selects elements while offline, their locations may change before the batch syncs. The `expected_container_id` enables server-side comparison. Swapping should only occur when pieces are in different physical locations/rows.

### 4. Scanning

Barcode/QR code scan resolution.

| Method | Path | Description |
|---|---|---|
| `POST` | `/scans` | Resolve a scanned code |
| `GET` | `/scans/history` | Recent scan events for device/user |

#### Scan Request

```json
{
  "scan_code": "R-B3-12",
  "scan_type": "RACK",
  "device_id": "uuid"
}
```

#### Scan Response

```json
{
  "scan_result": "MATCH",
  "resolved_entity_type": "RACK",
  "resolved_entity_id": "uuid",
  "entity": {
    "rack_code": "R-B3-12",
    "status": "ACTIVE",
    "location": { "area": "Yard West", "row": "A", "lane": "3" },
    "element_count": 8
  }
}
```

> **From meetings:** Scanning a load sheet barcode should immediately show all panels and their locations. The scanner should auto-launch from the main screen. Support scanning in any orientation. Provide confirmation feedback (success/failure) after every scan.

### 5. Load Sheets

Load sheet management and triage.

| Method | Path | Description |
|---|---|---|
| `GET` | `/loadsheets` | List load sheets: `?status=&load_date=&delivery_date=&project_id=` |
| `GET` | `/loadsheets/:id` | Load sheet detail with items and yard match status |
| `POST` | `/loadsheets` | Create/upload load sheet |
| `PATCH` | `/loadsheets/:id` | Update load sheet status |
| `GET` | `/loadsheets/:id/items` | Items with yard match and location data |
| `PATCH` | `/loadsheets/:id/items/:item_id` | Update item selection or status |
| `POST` | `/loadsheets/:id/items/select-all` | Bulk select/deselect items |

> **From meetings:** Load sheets should display piece locations clearly. The team wants to replace paper load sheets with digital versions in-app. Load sheets should show: element code, location (area/row/lane), status, and planned load/delivery dates. Color coding: blue dots for active loads, orange for patch ready, red for issues.

### 6. Dispatch & Delivery

What to load/deliver next.

| Method | Path | Description |
|---|---|---|
| `GET` | `/dispatch-plans` | List plans: `?project_id=&status=&departure_after=` |
| `GET` | `/dispatch-plans/:id` | Plan detail with ordered items |
| `POST` | `/dispatch-plans` | Create plan |
| `PATCH` | `/dispatch-plans/:id/items/:item_id` | Update item status (stage, load, deliver, skip) |
| `POST` | `/dispatch-plans/:id/complete` | Mark plan complete (with partial load support) |

> **From meetings:** Force complete options allow marking loads finished even if some panels remain staged. The system should prompt: "Load not full - confirm complete?" Communication between yard staff and load scheduler is critical for on-the-fly adjustments.

### 7. Offline Sync Queue

Accept queued operations from offline devices.

| Method | Path | Description |
|---|---|---|
| `POST` | `/sync/batch` | Submit batch of queued operations (idempotent via dedupe_key) |
| `GET` | `/sync/status` | Get sync status for device: `?device_id=&since=` |
| `GET` | `/sync/delta` | Delta data sync: `?since=ISO&tables=elements,containers` |

#### Batch Sync Request

```json
{
  "device_id": "uuid",
  "operations": [
    {
      "operation_type": "MOVE_BATCH_SUBMIT",
      "dedupe_key": "dev-123-1679500000",
      "payload": { "...move batch payload..." },
      "queued_at": "2026-03-23T10:15:00Z"
    },
    {
      "operation_type": "SCAN_UPLOAD",
      "dedupe_key": "dev-123-1679500030",
      "payload": { "...scan payload..." },
      "queued_at": "2026-03-23T10:15:30Z"
    }
  ]
}
```

#### Batch Sync Response

```json
{
  "results": [
    { "dedupe_key": "dev-123-1679500000", "status": "ACKED", "server_id": "uuid" },
    { "dedupe_key": "dev-123-1679500030", "status": "ACKED", "server_id": "uuid" }
  ],
  "server_time": "2026-03-23T10:16:00Z"
}
```

> **From meetings:** All mutations must be idempotent. The `dedupe_key` prevents duplicate processing when devices retry on reconnect. The forced sync on reconnection is mandatory - the app must sync all queued operations before allowing new ones.

### 8. Audit

| Method | Path | Description |
|---|---|---|
| `GET` | `/elements/:id/history` | Full movement history for an element |
| `GET` | `/audit/changes` | Audit log: `?entity_type=&entity_id=&since=&user_id=` |

> **From meetings:** Audit tracing tracks per-column changes. The audit system supports accountability and data quality recovery over the expected ~1-year data cleanup period.

---

## Integration Points

| System | Direction | Mechanism | Purpose |
|---|---|---|---|
| **Compass** | Bidirectional | Direct DB / internal API | Source of truth for projects, elements, racks |
| **i2 (legacy)** | Read, deprecating | Sync every 5 min | Legacy yard system - being retired |
| **IPBS** | Read | Sync trigger on project inserts | Production milestones, status automation |
| **NetSuite** | Read | Project sync | Financial/project data |

> **From meetings:** The sync between Compass and i2 runs every 5 minutes, causing lag and status conflicts. The new app should be the single source of truth for yard operations, with i2 being retired. IPBS integration will automate status flipping when elements hit milestone 450 (production finished).

## Error Handling

All error responses follow:

```json
{
  "error": {
    "code": "CONFLICT_DETECTED",
    "message": "Element 24-10345-001 has moved since selection",
    "details": {
      "element_id": "uuid",
      "expected_container": "Row A, Lane 3",
      "actual_container": "Row B, Lane 1"
    }
  }
}
```

Standard HTTP status codes: `400` validation, `401` auth, `403` forbidden, `404` not found, `409` conflict, `422` unprocessable, `503` service unavailable.

> **From meetings:** Error reporting must include the panel element AND the rebooking target location on failure. Users need to know exactly what went wrong and where the element actually is now.

## Rate Limiting & Performance

- Pagination required for list endpoints (max 100 per page)
- `/locations/tree` response should be cached aggressively (ETag/Last-Modified)
- Delta sync (`?updated_since=`) minimizes transfer for offline reconnection
- Database indexing per schema design (see [Database Schema](./database-schema.md))
- Element search limited to project scope by default to avoid scanning millions of rows
