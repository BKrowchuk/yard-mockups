# PWA Requirements

## Why PWA

The team chose a Progressive Web App over a native app for several reasons identified across multiple planning meetings:

1. **Bypass App Store restrictions** - iOS deployment requires developer licensing and review cycles. A PWA installable via Safari/Chrome eliminates this friction.
2. **Offline-first by design** - Service workers provide caching and background sync natively.
3. **Cross-platform** - Single codebase for Android and iOS phones/tablets.
4. **Instant updates** - No app store review delay; deploy updates server-side.
5. **URL-based sharing** - Deep links to specific elements, racks, or load sheets.
6. **Low device footprint** - ~20 users on mixed devices (mostly phones, some tablets).

> **From meetings (Initial Yard App Discussion, 2026-02-03):** "Plan to redesign the app as a PWA with caching for offline use and fast loading." Jake emphasized offline capabilities to queue data and reduce user frustration from connectivity drops.

## Web App Manifest

```json
{
  "name": "Yard Management App",
  "short_name": "Yard App",
  "description": "Track elements, racks, and loadsheets across the yard",
  "start_url": "/",
  "display": "standalone",
  "orientation": "portrait",
  "background_color": "#1a1a2e",
  "theme_color": "#1a1a2e",
  "icons": [
    { "src": "/icons/icon-192.png", "sizes": "192x192", "type": "image/png" },
    { "src": "/icons/icon-512.png", "sizes": "512x512", "type": "image/png" },
    { "src": "/icons/icon-maskable-512.png", "sizes": "512x512", "type": "image/png", "purpose": "maskable" }
  ],
  "categories": ["business", "productivity"],
  "shortcuts": [
    {
      "name": "Scan",
      "url": "/scanner",
      "icons": [{ "src": "/icons/scan-96.png", "sizes": "96x96" }]
    },
    {
      "name": "Search",
      "url": "/search",
      "icons": [{ "src": "/icons/search-96.png", "sizes": "96x96" }]
    }
  ]
}
```

## Service Worker Strategy

### Caching Strategies (Workbox)

| Resource | Strategy | Rationale |
|---|---|---|
| App shell (HTML, JS, CSS) | **Precache** (build-time) | Instant offline load |
| Static assets (icons, fonts) | **Cache-first** | Rarely change |
| API: `/locations/tree` | **Stale-while-revalidate** | Stable data, OK to be slightly stale |
| API: `/elements`, `/racks` | **Network-first**, fallback cache | Fresh data preferred, cache for offline |
| API: `/search` | **Network-only**, manual cache | Results cached in IndexedDB by app logic |
| API: `/sync/*` | **Network-only** | Must always hit server |
| API: mutations (POST/PATCH) | **Network-only** + sync queue | Queued in IndexedDB if offline |

### Service Worker Registration

```typescript
// sw.ts (Workbox)
import { precacheAndRoute } from 'workbox-precaching';
import { registerRoute } from 'workbox-routing';
import { StaleWhileRevalidate, NetworkFirst, CacheFirst } from 'workbox-strategies';
import { BackgroundSyncPlugin } from 'workbox-background-sync';
import { ExpirationPlugin } from 'workbox-expiration';

// Precache app shell
precacheAndRoute(self.__WB_MANIFEST);

// Location tree - stale while revalidate
registerRoute(
  ({ url }) => url.pathname.includes('/locations/tree'),
  new StaleWhileRevalidate({
    cacheName: 'location-tree',
    plugins: [new ExpirationPlugin({ maxAgeSeconds: 3600 })]
  })
);

// API data - network first with cache fallback
registerRoute(
  ({ url }) => url.pathname.startsWith('/api/yard/v1/') &&
               !url.pathname.includes('/sync/'),
  new NetworkFirst({
    cacheName: 'api-data',
    networkTimeoutSeconds: 5,
    plugins: [new ExpirationPlugin({ maxAgeSeconds: 900 })] // 15 min
  })
);

// Static assets - cache first
registerRoute(
  ({ request }) => request.destination === 'image' ||
                   request.destination === 'font',
  new CacheFirst({
    cacheName: 'static-assets',
    plugins: [new ExpirationPlugin({ maxEntries: 60, maxAgeSeconds: 30 * 24 * 3600 })]
  })
);
```

### Background Sync

The Service Worker handles background sync for queued mutations when connectivity returns:

```typescript
// Register background sync for offline mutations
const bgSyncPlugin = new BackgroundSyncPlugin('yard-sync-queue', {
  maxRetentionTime: 24 * 60, // 24 hours
  onSync: async ({ queue }) => {
    // Process queue entries through SyncEngine
    // See offline-sync.md for full sync flow
  }
});
```

## Install Experience

### Install Prompt

The app should show a custom install banner after the user's second visit:

```
┌──────────────────────────────────┐
│  Install Yard App for offline    │
│  access and faster loading       │
│                                  │
│  [Install]          [Not now]    │
└──────────────────────────────────┘
```

### iOS Safari Install

Since iOS doesn't support the `beforeinstallprompt` event, show instructions:

```
┌──────────────────────────────────┐
│  To install on iPhone:           │
│  1. Tap the Share button  ↑      │
│  2. Tap "Add to Home Screen"    │
│  3. Tap "Add"                    │
└──────────────────────────────────┘
```

## Network Handling

### Online/Offline Detection

```typescript
// useOnlineStatus.ts
function useOnlineStatus() {
  const [status, setStatus] = useState<'online' | 'slow' | 'offline'>('online');

  useEffect(() => {
    const checkConnectivity = async () => {
      if (!navigator.onLine) {
        setStatus('offline');
        return;
      }
      try {
        const start = Date.now();
        await fetch('/api/yard/v1/health', { method: 'HEAD', cache: 'no-store' });
        const latency = Date.now() - start;
        setStatus(latency > 3000 ? 'slow' : 'online');
      } catch {
        setStatus('offline');
      }
    };

    window.addEventListener('online', checkConnectivity);
    window.addEventListener('offline', () => setStatus('offline'));
    const interval = setInterval(checkConnectivity, 30000);

    checkConnectivity();
    return () => {
      window.removeEventListener('online', checkConnectivity);
      window.removeEventListener('offline', () => setStatus('offline'));
      clearInterval(interval);
    };
  }, []);

  return status;
}
```

### Network Switching (Wi-Fi <> 5G)

> **From meetings (Yard connectivity discussion):** The PWA must handle "seamless network switching" between Wi-Fi and 5G. Devices on forklifts move between Wi-Fi hot zones and cellular coverage.

The `navigator.connection` API (Network Information API) can detect network type changes:

```typescript
if ('connection' in navigator) {
  navigator.connection.addEventListener('change', () => {
    // Network type changed (wifi → cellular or vice versa)
    // Trigger connectivity check and delta sync
    syncEngine.checkAndSync();
  });
}
```

Key behaviors on network switch:
- Re-check connectivity immediately
- If sync queue has pending items, attempt processing
- Adjust timeout thresholds based on connection type (Wi-Fi: 5s, cellular: 10s)

## Camera & Scanner Requirements

### Barcode Scanning

The app needs camera access for barcode/QR code scanning:

```json
// Required permissions
{
  "permissions": ["camera"]
}
```

Implementation options:
1. **Web Barcode Detection API** (Chrome 83+, not Safari) - native, fast
2. **ZXing-js library** - cross-browser fallback, supports all formats

Scanner requirements:
- Auto-launch from home screen and floating action button
- Scan in any orientation (users in tight spaces)
- Support Code 128, Code 39, QR Code formats
- Provide audio/haptic feedback on successful scan
- Auto-navigate to resolved entity after scan

> **From meetings (260219-PL3):** "The app's barcode scanner will auto-launch from the main screen for quick scanning without extra taps. It will support scanning barcodes in any orientation."

## Storage Budget

| Storage | Limit | Usage |
|---|---|---|
| Service Worker cache | 50 MB | App shell + API responses |
| IndexedDB | 100 MB | Offline data + sync queue |
| Total per origin | ~150 MB | Well within browser limits |

Estimated actual usage:
- App shell: ~2 MB
- Location tree cache: ~500 KB
- Active project elements: ~2-5 MB
- Racks: ~200 KB
- Load sheets: ~100 KB
- Sync queue: ~50 KB per pending operation
- **Total: ~5-10 MB typical**

## Update Strategy

### App Updates

```typescript
// Check for service worker updates
if ('serviceWorker' in navigator) {
  navigator.serviceWorker.register('/sw.js').then(reg => {
    reg.addEventListener('updatefound', () => {
      const newWorker = reg.installing;
      newWorker.addEventListener('statechange', () => {
        if (newWorker.state === 'activated') {
          // Show update banner
          showUpdateBanner('New version available. Tap to update.');
        }
      });
    });
  });
}
```

Update flow:
1. New service worker installs in background
2. User sees "Update available" banner
3. User taps to reload with new version
4. IndexedDB data preserved across updates

## Device Requirements

| Requirement | Minimum |
|---|---|
| iOS | Safari 15+ (iOS 15+) |
| Android | Chrome 90+ |
| Screen | 375px width minimum (iPhone SE) |
| Camera | Required for scanning |
| Storage | 50 MB available |
| RAM | 2 GB+ |

> **From meetings:** Users primarily use phones (not tablets). Devices include iPhones and various Android phones. The app must work on whatever devices yard workers already carry. Forklift-mounted devices may use docking stations with wired connections.

## PWA Checklist

- [ ] Web App Manifest with icons (192px, 512px, maskable)
- [ ] Service Worker with precaching and runtime caching
- [ ] Offline fallback page
- [ ] HTTPS (required for service workers)
- [ ] Responsive design (375px - 1024px)
- [ ] App shell architecture (instant load)
- [ ] Background sync for offline mutations
- [ ] IndexedDB for persistent offline data
- [ ] Install prompt (Android) and instructions (iOS)
- [ ] Camera access for barcode scanning
- [ ] Network status detection and indicators
- [ ] Push notifications (future: sync complete, conflicts)
- [ ] Splash screen
- [ ] Standalone display mode
- [ ] Touch target sizes (44px minimum)
- [ ] Performance: LCP < 2.5s, FID < 100ms, CLS < 0.1
