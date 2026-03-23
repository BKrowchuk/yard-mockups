# Offline & Sync Strategy

## Problem Context

The yard has severe connectivity challenges:
- Poor Wi-Fi coverage with dead zones between buildings and in concrete-heavy areas
- Cellular service is inconsistent (Rogers network)
- Forklifts move through coverage gaps constantly
- Current app freezes/crashes on disconnection, forcing paper workarounds
- ~20 users across yard workers, QCs, and planners

The connectivity team is testing Wi-Fi improvements, 5G forklift antennas, and Starlink, but the app must work regardless of infrastructure state.

> **From meetings (Yard connectivity discussion, 2026-03-06):** "The app data set is large, but caching only essential info like element IDs and locations can keep offline operation manageable in megabytes." Hot zones with guaranteed connectivity will exist at key locations, but the app must handle 15-30 minute offline periods between zones.

## Architecture Overview

```
┌──────────────────────────────────────────────┐
│                   PWA Client                  │
│                                               │
│  ┌─────────┐  ┌──────────┐  ┌─────────────┐ │
│  │ UI Layer │──│ Cache Mgr│──│  IndexedDB   │ │
│  └────┬─────┘  └────┬─────┘  └─────────────┘ │
│       │              │                         │
│  ┌────┴─────┐  ┌────┴──────┐                  │
│  │ API Client│  │ Sync Queue│ ← writes queued  │
│  └────┬─────┘  └────┬──────┘   when offline   │
│       │              │                         │
│  ┌────┴──────────────┴──────┐                  │
│  │      Service Worker       │                  │
│  │  (Workbox Background Sync)│                  │
│  └────────────┬──────────────┘                  │
└───────────────┼──────────────────────────────────┘
                │
        ════════╪════════  Network boundary
                │
┌───────────────┼──────────────────────────────────┐
│  ┌────────────┴───────────┐                       │
│  │   Compass API Server    │                       │
│  │   /api/yard/v1/...      │                       │
│  └────────────┬───────────┘                        │
│  ┌────────────┴───────────┐                        │
│  │    PostgreSQL + sync_   │                        │
│  │    queue / sync_attempts│                        │
│  └────────────────────────┘                        │
└───────────────────────────────────────────────────┘
```

## Sync Queue (Client-Side)

### IndexedDB Schema

```typescript
interface SyncQueueEntry {
  id: string;              // UUID generated on device
  operationType: string;   // 'MOVE_BATCH_SUBMIT' | 'SCAN_UPLOAD' | etc.
  payload: object;         // Full request body
  dedupeKey: string;       // device_id + timestamp (prevents duplicate processing)
  status: 'QUEUED' | 'SENDING' | 'ACKED' | 'FAILED' | 'DEAD_LETTER';
  retryCount: number;
  maxRetries: number;      // Default: 5
  nextRetryAt: number;     // Epoch ms, exponential backoff
  lastError: string | null;
  createdAt: number;       // Epoch ms
  ackedAt: number | null;
}
```

### Queue Operations

```typescript
// When user performs a write action while offline (or online)
async function enqueueOperation(op: QueueableOperation): Promise<void> {
  const entry: SyncQueueEntry = {
    id: crypto.randomUUID(),
    operationType: op.type,
    payload: op.payload,
    dedupeKey: `${deviceId}-${Date.now()}`,
    status: 'QUEUED',
    retryCount: 0,
    maxRetries: 5,
    nextRetryAt: Date.now(),
    lastError: null,
    createdAt: Date.now(),
    ackedAt: null,
  };
  await db.syncQueue.add(entry);

  // If online, trigger immediate sync
  if (navigator.onLine) {
    syncEngine.processQueue();
  }
}
```

### Queueable Operation Types

| Operation | Payload | Priority |
|---|---|---|
| `MOVE_BATCH_SUBMIT` | Move batch with element IDs, expected/target containers | High |
| `SCAN_UPLOAD` | Scan event data (code, type, device, timestamp) | Medium |
| `STATUS_UPDATE` | Element status change (e.g., mark as loaded) | High |
| `LOADSHEET_ACTION` | Load sheet item selection/action | Medium |
| `LOCATION_UPDATE` | Activate/deactivate/rename location | Low |

## Sync Engine

### Processing Flow

```
On reconnect OR periodic check (every 30s when online):

1. Check navigator.onLine
2. GET /sync/status?device_id={id} → confirm server reachable
3. Process queue in FIFO order:
   a. Set entry.status = 'SENDING'
   b. POST /sync/batch with up to 10 operations
   c. For each result:
      - ACKED → mark entry ACKED, update local cache with server response
      - CONFLICT → mark FAILED, surface to user for resolution
      - RETRYABLE_ERROR → increment retryCount, set nextRetryAt with backoff
      - FATAL_ERROR → mark DEAD_LETTER, notify user
4. After queue drained: GET /sync/delta?since={last_sync} → update local cache
5. Invalidate stale UI state
```

### Retry Strategy

```
Attempt 1: immediate
Attempt 2: 5 seconds
Attempt 3: 30 seconds
Attempt 4: 2 minutes
Attempt 5: 10 minutes
After 5 failures: DEAD_LETTER (requires manual intervention)
```

Backoff formula: `min(baseDelay * 2^(retryCount - 1), maxDelay)`
- `baseDelay`: 5000ms
- `maxDelay`: 600000ms (10 min)

### Forced Sync on Reconnect

When the device transitions from offline to online:

1. **Block new operations** until queue is drained
2. Show "Syncing..." overlay with progress bar
3. Process all QUEUED and FAILED (retryable) entries
4. Pull delta updates from server
5. Clear overlay, resume normal operation

> **From meetings (Yard Swapping process, 2026-03-10):** "Develop app features supporting offline functionality with forced sync upon re-connection." The forced sync is non-negotiable - stale data causes real operational errors like double rebooking.

## Conflict Detection & Resolution

### How Conflicts Happen

```
Timeline:
  t0: User A selects Element X at Row A (offline)
  t1: User B moves Element X to Row B (online)
  t2: User A reconnects, submits move of Element X to Row C
  t3: Server detects: expected=Row A, actual=Row B → CONFLICT
```

### Server-Side Detection

The `expected_container_id` sent with each move item enables comparison:

```sql
-- For each move_batch_item:
SELECT current_container_id
FROM elements
WHERE id = :element_id;

-- If current_container_id != expected_container_id → CONFLICT
```

### Client-Side Resolution UI

```
┌──────────────────────────────────┐
│  ⚠ Sync Conflicts (2 items)     │
│──────────────────────────────────│
│  HC-001                          │
│  Expected: Row A, Lane 3         │
│  Actually: Row B, Lane 1         │
│  [Skip] [Move to original dest]  │
│                                  │
│  HC-002                          │
│  Expected: Staging Slot 08       │
│  Actually: Loaded on Trailer 7   │
│  [Skip] [Cannot move - loaded]   │
│                                  │
│  [Skip All] [Retry Remaining]    │
└──────────────────────────────────┘
```

### Resolution Options

| Scenario | Options |
|---|---|
| Element moved to different location | Skip, or re-confirm move from new location |
| Element loaded on trailer | Skip only (cannot move loaded element) |
| Element marked as delivered | Skip only |
| Element not found | Skip, flag for investigation |
| Capacity exceeded at destination | Skip, choose different destination |

## Delta Sync (Server to Client)

### How It Works

```
GET /sync/delta?since=2026-03-23T10:00:00Z&tables=elements,containers,racks

Response:
{
  "server_time": "2026-03-23T10:15:00Z",
  "changes": {
    "elements": {
      "updated": [ { ...element with all fields... }, ... ],
      "deleted_ids": [ "uuid", ... ]
    },
    "containers": {
      "updated": [ { ...container... }, ... ],
      "deleted_ids": []
    }
  }
}
```

### Sync Triggers

| Trigger | Action |
|---|---|
| App foreground (resume) | Delta sync if last sync > 1 min ago |
| Online transition | Forced sync (queue + delta) |
| Every 60 seconds (online) | Background delta sync |
| After any write operation | Delta sync for affected tables |
| User pull-to-refresh | Full delta sync |

### What Gets Synced

| Table | Direction | Frequency |
|---|---|---|
| `elements` | Server → Client | Every delta sync |
| `containers` | Server → Client | Every delta sync |
| `racks` | Server → Client | Every delta sync |
| `areas/rows/lanes` | Server → Client | Location tree refresh (hourly) |
| `loadsheets` (today+tomorrow) | Server → Client | Every delta sync |
| `move_batches` (user's own) | Bidirectional | On submit + delta |
| `sync_queue` | Client → Server | On reconnect/manual |

## Offline Duration Limits

| Duration | Behavior |
|---|---|
| **0-5 min** | Full functionality, all writes queued |
| **5-15 min** | Warning banner, writes still queued |
| **15-30 min** | "Data may be stale" warning, new rebook/moves disabled, read-only browse + existing queue |
| **30+ min** | Persistent prompt to reconnect, only cached read-only browsing, queue preserved |

> **From meetings (Yard connectivity discussion):** "Offline operation can't last all day due to data volume and update needs, so syncing must happen regularly to avoid stale data." The 15-minute threshold for write restrictions reflects the balance between usability and data integrity.

## Data Integrity Guarantees

| Guarantee | Mechanism |
|---|---|
| No duplicate operations | `dedupe_key` on `sync_queue` (unique constraint) |
| Operation ordering preserved | FIFO processing, `created_at` ordering |
| Conflict detection | `expected_container_id` comparison at submit |
| No data loss on crash | IndexedDB persistence (survives app close/crash) |
| Audit trail | All mutations logged in `element_movements` + `sync_attempts` |
| Stale data protection | Offline duration limits + forced sync on reconnect |

## Monitoring & Diagnostics

### Sync Status Dashboard (in-app)

```
┌──────────────────────────────────┐
│  Sync Status                     │
│──────────────────────────────────│
│  Connection: Online (Wi-Fi)      │
│  Last sync: 2 min ago            │
│  Queue: 0 pending                │
│                                  │
│  Recent Syncs:                   │
│  ✓ 10:15 - 3 moves synced       │
│  ✓ 10:10 - delta sync (12 updates)|
│  ✗ 10:05 - failed (timeout)     │
│    → retried at 10:06 ✓         │
│                                  │
│  Dead Letter: 0                  │
│  [Force Sync Now]                │
└──────────────────────────────────┘
```

### Server-Side Monitoring

- Track `sync_attempts` for error rate by device
- Alert on devices with high retry counts
- Monitor `sync_queue` entries stuck in `SENDING` state
- Dashboard for dead letter entries requiring manual resolution

> **From meetings (Yard Swapping process):** A monitoring dashboard with red/green status indicators is needed to track Wi-Fi access point health. Early alerts help address outages before users are affected. The sync monitoring extends this concept to the application layer.
