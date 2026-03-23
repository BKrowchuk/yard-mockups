# Database Schema

> **Canonical schema definition:** [`../location-management-db-table-design.md`](../location-management-db-table-design.md)

## Overview

The Yard App database is a PostgreSQL schema with **17 tables** organized into four domains:

| Domain | Tables | Purpose |
|---|---|---|
| **Location Hierarchy** | `areas`, `rows`, `lanes`, `containers` | Normalized yard geography: area > row > lane > container |
| **Inventory** | `projects`, `elements`, `racks` | Master records for trackable items and their current locations |
| **Operations** | `move_batches`, `move_batch_items`, `element_movements`, `scan_events`, `loadsheets`, `loadsheet_items`, `dispatch_plans`, `dispatch_plan_items` | Movement history, scanning, load sheets, and delivery planning |
| **Offline Sync** | `sync_queue`, `sync_attempts` | Queued offline operations with retry telemetry |

## Entity Relationship Diagram (Simplified)

```
projects
  |
  +--< elements >--+-- current_container_id --> containers
  |                 +-- current_rack_id ------> racks
  |
  +--< racks >-----+-- current_container_id --> containers

areas
  +--< rows
        +--< lanes
              +--< containers
                    +--< elements (via current_container_id)
                    +--< racks (via current_container_id)

elements --< element_movements   (immutable history)
elements --< move_batch_items --< move_batches  (rebook/move operations)
elements --< loadsheet_items --< loadsheets  (scan/review/selection)
elements --< dispatch_plan_items --< dispatch_plans  (load/deliver ordering)

sync_queue --< sync_attempts  (offline retry telemetry)
```

## Location Hierarchy

The yard is modeled as a strict 4-level tree:

1. **Area** - top-level yard region (e.g., "Yard West", "Staging West", "LD Front", "LD Back")
2. **Row** - row within area (e.g., "A", "B", "Staging")
3. **Lane** - lane within row (e.g., "1", "Main") - segments of ~50-70 ft for easier navigation
4. **Container** - final assignable slot (rack, pre-stack, staging slot, trailer)

Each level has an `active` flag so unused locations can be hidden from pickers without deletion. Containers have a `container_type` enum (`RACK`, `PRE_STACK`, `STAGING_SLOT`, etc.) and optional `capacity` and `tag_code` (QR/barcode).

> **From meetings:** Users need the ability to dynamically create, rename, activate/deactivate locations. Only designated users (e.g., Martin) should have location creation access, managed through a ticketing system. The "Laid Down" area must be split into "LD Front" and "LD Back" to reflect Plant 3's physical layout.

## Inventory: Elements & Racks

### Elements
- `element_code` is the unique identifier (e.g., `24-10345-001`)
- `barcode` for scan lookup
- `status` tracks lifecycle: `CONFIRMED` > `QUEUED` > `LOADED` > `DELIVERED` (plus `CONFLICT`, `MISSING`, `NOT_IN_YARD`)
- `current_container_id` gives instant "where is it now" (null = not in yard)
- `current_rack_id` links to rack if element is on one
- Full-text search index on `element_code + description`

> **From meetings:** Elements share underscore numbers across duplicates. The system must handle duplicate identifiers gracefully - collapsing duplicates in search, allowing users to select by location. Production IDs must remain immutable during swaps.

### Racks
- Separate from container concept - a rack is a trackable, movable asset
- `rack_code` and `barcode` for identification
- `leave_at` supports "what leaves next" queries
- `status`: `ACTIVE`, `IN_TRANSIT`, `LOADED`, etc.
- Shipping racks vs. pay racks must be distinguished

> **From meetings:** Thousands of pieces remain untracked on racks. Mandatory scanning before racks leave/return is needed. QR codes on racks failed due to physical damage - alternative tracking (tags, air tags) being explored.

## Operations Tables

### Move Batches & Items
Models the multi-select rebook/move workflow with conflict detection:
- A `move_batch` represents one user action (rebook, store, move)
- Each `move_batch_item` records per-element result: `MOVED`, `CONFLICT`, `SKIPPED`, `FAILED`
- `expected_container_id` vs `actual_container_id_at_submit` enables conflict detection for offline scenarios
- Device and user tracking for audit

### Element Movements
Immutable audit trail of every location transition. Types: `MOVE`, `REBOOK`, `LOAD`, `UNLOAD`, `DELIVER`.

> **From meetings:** Audit tracing is critical. The team plans to enable audit trace systems within 1-2 weeks. Multi-column change tracking (per-column rather than per-row) is the chosen strategy.

### Load Sheets
- `loadsheets` header: code, dates (load/delivery), status (`NEW` > `PARSED` > `PARTIAL` > `CLOSED`)
- `loadsheet_items` per line: element matching, yard match status (`OK`, `MISSING`, `NOT_IN_YARD`, `UNKNOWN`), sequence, selection state
- Supports barcode scanning a load sheet to see all pieces and their locations

### Dispatch Plans
- Outbound delivery/loading ordering
- `dispatch_plan_items.sequence_no` is source of truth for "load/deliver next"
- Status progression: `PENDING` > `STAGED` > `LOADED` > `DELIVERED` > `SKIPPED`

## Offline Sync Tables

See [Offline & Sync Strategy](./offline-sync.md) for full architecture.

- `sync_queue`: Operations queued on-device with `dedupe_key` to prevent duplicates
- `sync_attempts`: Per-attempt telemetry for diagnostics
- Status flow: `QUEUED` > `SENDING` > `ACKED` or `FAILED` > `DEAD_LETTER`

## Key Indexes & Query Patterns

| Query | How |
|---|---|
| Find element by scan | `barcode` or `element_code` unique index, join to location path |
| Find rack + leave order | `rack_code`/`barcode`, ordered by `leave_at` |
| Find pre-stack | Filter `containers.container_type = 'PRE_STACK'` |
| Grid path search | Join `areas > rows > lanes > containers > elements` |
| Project search | Filter by `project_id` on elements/racks |
| Location occupancy | Count elements per container vs `capacity` |
| Move conflict detection | Compare `expected_container_id` with current at submit |
| Load sheet triage | Group `loadsheet_items` by `yard_match_status` |
| What loads next | First `dispatch_plan_items` with `load_status = 'PENDING'` ordered by `sequence_no` |
| Full-text search | GIN index on `element_code || description` |

## MVP Phasing

| Phase | Tables |
|---|---|
| **Phase 1** | `projects`, `areas`, `rows`, `lanes`, `containers`, `elements`, `element_movements` |
| **Phase 2** | `move_batches`, `move_batch_items`, `loadsheets`, `loadsheet_items` |
| **Phase 3** | `racks`, `dispatch_plans`, `dispatch_plan_items`, `scan_events`, `sync_queue`, `sync_attempts` |

## Enums / Controlled Text

| Enum | Values |
|---|---|
| `element_status` | `CONFIRMED`, `QUEUED`, `CONFLICT`, `MISSING`, `NOT_IN_YARD`, `LOADED`, `DELIVERED` |
| `move_state` | `DRAFT`, `QUEUED`, `SUBMITTED`, `COMPLETED`, `FAILED`, `PARTIAL_CONFLICT` |
| `batch_item_result` | `MOVED`, `CONFLICT`, `SKIPPED`, `FAILED` |
| `yard_match_status` | `OK`, `MISSING`, `NOT_IN_YARD`, `UNKNOWN` |
| `dispatch_item_status` | `PENDING`, `STAGED`, `LOADED`, `DELIVERED`, `SKIPPED` |
| `sync_attempt_result` | `SUCCESS`, `RETRYABLE_ERROR`, `FATAL_ERROR` |
| `container_type` | `RACK`, `PRE_STACK`, `STAGING_SLOT`, `TRAILER`, etc. |

> **From meetings:** Additional statuses discussed: "ready for patch", "ready for windows" (post-paint, pre-load). Color codes: blue dots = active loads, red dots = issues, orange = patch ready.
