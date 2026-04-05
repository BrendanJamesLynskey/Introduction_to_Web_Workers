# Introduction to Web Workers

**Background Threads, Service Workers & Offline-First Web Apps**

Dedicated Workers - Shared Workers - Service Workers - Cache API - PWA

---

## Table of Contents

1. [Title](#slide-01--title)
2. [Agenda](#slide-02--agenda)
3. [The Single-Threaded Problem](#slide-03--the-single-threaded-problem)
4. [What Are Web Workers?](#slide-04--what-are-web-workers)
5. [Dedicated Workers](#slide-05--dedicated-workers)
6. [Message Passing](#slide-06--message-passing)
7. [Worker Scope & Limitations](#slide-07--worker-scope--limitations)
8. [Real-World: Heavy Computation](#slide-08--real-world-heavy-computation)
9. [Shared Workers](#slide-09--shared-workers)
10. [Service Workers -- Lifecycle](#slide-10--service-workers--lifecycle)
11. [Service Worker Registration](#slide-11--service-worker-registration)
12. [Cache API](#slide-12--cache-api)
13. [Caching Strategies](#slide-13--caching-strategies)
14. [Offline-First Architecture](#slide-14--offline-first-architecture)
15. [Push Notifications](#slide-15--push-notifications)
16. [Progressive Web Apps](#slide-16--progressive-web-apps)
17. [Workbox -- Google's SW Toolkit](#slide-17--workbox--googles-sw-toolkit)
18. [Debugging Workers](#slide-18--debugging-workers)
19. [Performance Patterns](#slide-19--performance-patterns)
20. [Summary & Next Steps](#slide-20--summary--next-steps)

---

## Slide 01 -- Title

# Introduction to Web Workers

**Background Threads, Service Workers & Offline-First Web Apps**

Dedicated Workers - Shared Workers - Service Workers - Cache API - PWA

---

## Slide 02 -- Agenda

### Part I -- Foundations

- The single-threaded problem
- What are Web Workers?
- Worker types overview

### Part II -- Dedicated Workers

- Creating & messaging workers
- Structured clone & transferables
- Scope, limitations & real-world use

### Part III -- Service Workers

- Lifecycle: install, activate, fetch
- Cache API & caching strategies
- Offline-first & push notifications

### Part IV -- Advanced

- Progressive Web Apps
- Workbox library
- Debugging & performance patterns

> 20 slides - Covers dedicated, shared & service workers with live code examples, SVG diagrams & best practices.

---

## Slide 03 -- The Single-Threaded Problem

JavaScript runs on a **single main thread**. Every task -- DOM updates, event handlers, network callbacks, layout, painting -- shares that one thread.

### The 16ms Budget

To hit **60 fps**, each frame must complete in ~16.6ms. A heavy computation blocks the entire UI -- scroll jank, frozen inputs, dropped frames.

```javascript
// Blocks UI for seconds!
function fibonacci(n) {
  if (n <= 1) return n;
  return fibonacci(n - 1) + fibonacci(n - 2);
}
fibonacci(45); // main thread frozen
```

### Main Thread Timeline

**Normal frame:**

| JS Event | Layout | Paint | ... idle to 16ms ... |

**With heavy computation:**

| fibonacci(45) -- 3200ms BLOCKED | No layout | No paint | User input lost |

**With Web Worker:**

| JS Event | Layout | Paint |
| Worker: fibonacci(45) -- separate thread (runs in parallel) |

---

## Slide 04 -- What Are Web Workers?

Web Workers run JavaScript in **background threads**, separate from the main UI thread. They communicate via **message passing** and cannot access the DOM directly.

| Type | Scope | Lifetime | Use Case |
|------|-------|----------|----------|
| **Dedicated Worker** | Single page | While page is open | Heavy computation, parsing, image processing |
| **Shared Worker** | Multiple pages (same origin) | While any tab is open | Shared state across tabs, WebSocket pooling |
| **Service Worker** | Origin-wide proxy | Event-driven (outlives tabs) | Offline caching, push notifications, background sync |

### Browser Support

All modern browsers. Dedicated Workers since IE10. Service Workers since 2018 across all major engines.

### Thread Model

Each worker gets its own event loop, global scope (`self`), and JS execution context. No shared memory by default.

### Security

Workers obey **same-origin policy**. Service workers require HTTPS (except localhost). No `eval()` from strings in some contexts.

---

## Slide 05 -- Dedicated Workers

### Main Thread (index.js)

```javascript
// Create a dedicated worker
const worker = new Worker('worker.js');

// Send data to the worker
worker.postMessage({
  task: 'fibonacci',
  value: 45
});

// Receive results back
worker.onmessage = (event) => {
  console.log('Result:', event.data);
  // UI remains responsive!
};

// Handle errors
worker.onerror = (err) => {
  console.error('Worker error:', err.message);
};

// Terminate when done
worker.terminate();
```

### Worker Thread (worker.js)

```javascript
// Worker has its own global scope: `self`
self.onmessage = function(event) {
  const { task, value } = event.data;

  if (task === 'fibonacci') {
    const result = fibonacci(value);
    // Send result back to main thread
    self.postMessage(result);
  }
};

function fibonacci(n) {
  if (n <= 1) return n;
  return fibonacci(n - 1) + fibonacci(n - 2);
}

// Workers can also close themselves
// self.close();
```

### Inline Workers (Blob URL)

```javascript
const blob = new Blob([`
  self.onmessage = (e) => {
    self.postMessage(e.data * 2);
  };
`], { type: 'application/javascript' });
const w = new Worker(URL.createObjectURL(blob));
```

---

## Slide 06 -- Message Passing

### Structured Clone Algorithm

Data sent via `postMessage` is deep-copied using the structured clone algorithm. Supports objects, arrays, Maps, Sets, Dates, Blobs, ArrayBuffers, and more.

- Functions, DOM nodes, and Error objects **cannot** be cloned
- Prototype chains are **not preserved**
- Cloning large data has a **performance cost**

```javascript
// Structured clone — data is COPIED
worker.postMessage({
  pixels: new Uint8Array(4_000_000),
  metadata: { width: 1000, height: 1000 }
});
// Both threads have separate copies
```

### Transferable Objects

**Zero-copy** transfer of ownership. The sending context loses access to the data. Orders of magnitude faster for large buffers.

```javascript
// Transfer — data is MOVED (zero-copy)
const buffer = new ArrayBuffer(32_000_000);

worker.postMessage(buffer, [buffer]);
// buffer.byteLength === 0 now!

// Transfer multiple objects
const ab1 = new ArrayBuffer(1024);
const ab2 = new ArrayBuffer(2048);
worker.postMessage(
  { a: ab1, b: ab2 },
  [ab1, ab2]  // transfer list
);
```

### Transferable Types

- `ArrayBuffer` -- raw binary data
- `MessagePort` -- channel endpoints
- `OffscreenCanvas` -- canvas rendering
- `ImageBitmap` -- decoded images
- `ReadableStream` / `WritableStream`

---

## Slide 07 -- Worker Scope & Limitations

| Available in Workers | NOT Available |
|----------------------|---------------|
| `fetch()`, `XMLHttpRequest` | `document`, `window` |
| `setTimeout` / `setInterval` | DOM APIs (`querySelector`, etc.) |
| `IndexedDB` | `localStorage` / `sessionStorage` |
| `WebSocket` | `alert()` / `confirm()` |
| `crypto`, `TextEncoder` | `document.cookie` |
| `importScripts()` | Parent scope variables |
| `navigator` (subset) | `window.getComputedStyle` |
| `Cache` API | Any rendering / layout APIs |

### importScripts()

```javascript
// Classic workers: sync imports
importScripts(
  'utils.js',
  'parser.js'
);
```

### ES Modules in Workers

```javascript
// Module workers (modern)
const w = new Worker('worker.js', {
  type: 'module'
});

// Inside worker.js:
import { parse } from './parser.js';
import _ from 'https://esm.sh/lodash';
```

### WorkerGlobalScope

`self` refers to the worker's own global. Key properties: `self.name`, `self.location`, `self.navigator`, `self.performance`.

---

## Slide 08 -- Real-World: Heavy Computation

### Image Processing

```javascript
// worker-image.js
self.onmessage = ({ data }) => {
  const { pixels, width, height } = data;
  const d = new Uint8ClampedArray(pixels);

  // Grayscale filter
  for (let i = 0; i < d.length; i += 4) {
    const avg = (d[i] + d[i+1] + d[i+2]) / 3;
    d[i] = d[i+1] = d[i+2] = avg;
  }

  self.postMessage(
    d.buffer, [d.buffer]  // transfer
  );
};
```

### CSV Parsing

```javascript
// worker-csv.js
self.onmessage = ({ data }) => {
  const rows = data.csv.split('\n');
  const headers = rows[0].split(',');
  const records = [];

  for (let i = 1; i < rows.length; i++) {
    const vals = rows[i].split(',');
    const obj = {};
    headers.forEach((h, j) => {
      obj[h.trim()] = vals[j]?.trim();
    });
    records.push(obj);
  }

  self.postMessage({ records });
};
```

### Cryptography

```javascript
// worker-crypto.js
self.onmessage = async ({ data }) => {
  const encoder = new TextEncoder();
  const msgBuf = encoder.encode(data.text);

  // SHA-256 hashing
  const hashBuf = await crypto.subtle
    .digest('SHA-256', msgBuf);

  const hashArr = Array.from(
    new Uint8Array(hashBuf)
  );
  const hex = hashArr
    .map(b => b.toString(16).padStart(2,'0'))
    .join('');

  self.postMessage({ hash: hex });
};
```

### Pattern: Progress Reporting

```javascript
// Worker sends periodic progress updates
for (let i = 0; i < total; i++) {
  processItem(items[i]);
  if (i % 1000 === 0) self.postMessage({ type: 'progress', done: i, total });
}
self.postMessage({ type: 'complete', result: output });
```

---

## Slide 09 -- Shared Workers

A **SharedWorker** is shared across all pages/tabs/iframes of the **same origin**. Each connection gets a unique `MessagePort`.

### Connecting from Pages

```javascript
// page-a.js (Tab 1) and page-b.js (Tab 2)
const shared = new SharedWorker('shared.js');
const port = shared.port;

port.onmessage = (e) => {
  console.log('Received:', e.data);
};

port.postMessage({ user: 'Tab1', action: 'join' });
port.start(); // required if using addEventListener
```

### Shared Worker Script

```javascript
// shared.js
const ports = [];

self.onconnect = (event) => {
  const port = event.ports[0];
  ports.push(port);

  port.onmessage = (e) => {
    // Broadcast to all connected tabs
    ports.forEach(p => {
      p.postMessage({
        from: e.data.user,
        msg: e.data.action,
        connections: ports.length
      });
    });
  };
};
```

### Use Cases

- WebSocket connection pooling
- Cross-tab state synchronisation

---

## Slide 10 -- Service Workers -- Lifecycle

### State Transitions

```
register() --> installing --> waiting --> activating --> activated
```

| State | Event | Description |
|-------|-------|-------------|
| **installing** | `install` | Pre-cache assets |
| **waiting** | -- | Old SW still active. `skipWaiting()` to bypass |
| **activating** | `activate` | Clean old caches |
| **activated** | -- | Intercepts fetch, push, sync events |

### Key Concepts

- Service workers act as a **network proxy** between the browser and server
- They run in a **separate thread** and have no DOM access
- They are **event-driven** -- the browser starts/stops them as needed
- Require **HTTPS** in production (localhost exempt)
- Can outlive the page -- respond to push/sync even when tab is closed

### Events a Service Worker Handles

- `install` -- Pre-cache critical assets
- `activate` -- Clean up old cache versions
- `fetch` -- Intercept network requests
- `push` -- Receive push messages from server
- `sync` -- Background sync when connectivity returns
- `message` -- Communicate with controlled pages

---

## Slide 11 -- Service Worker Registration

### Registering the Service Worker

```javascript
// main.js — register on page load
if ('serviceWorker' in navigator) {
  window.addEventListener('load', async () => {
    try {
      const reg = await navigator.serviceWorker.register(
        '/sw.js',
        { scope: '/' }  // controls which URLs it intercepts
      );

      console.log('SW registered, scope:', reg.scope);

      // Check for updates
      reg.addEventListener('updatefound', () => {
        const newSW = reg.installing;
        newSW.addEventListener('statechange', () => {
          if (newSW.state === 'activated') {
            // New version ready — prompt reload?
            showUpdateBanner();
          }
        });
      });
    } catch (err) {
      console.error('SW registration failed:', err);
    }
  });
}
```

### Service Worker Shell

```javascript
// sw.js
const CACHE_NAME = 'app-v2';

self.addEventListener('install', (event) => {
  console.log('SW installing...');
  // Skip waiting to activate immediately
  self.skipWaiting();
});

self.addEventListener('activate', (event) => {
  console.log('SW activating...');
  // Claim all open clients
  event.waitUntil(
    clients.claim()
  );
});

self.addEventListener('fetch', (event) => {
  // Intercept every network request
  event.respondWith(
    handleFetch(event.request)
  );
});
```

### Scope Gotcha

A SW at `/app/sw.js` can only control URLs under `/app/`. Place `sw.js` at the root for full-site control, or use the `Service-Worker-Allowed` header.

---

## Slide 12 -- Cache API

### Cache Storage Basics

The Cache API provides a persistent key-value store where keys are `Request` objects and values are `Response` objects. Independent of HTTP cache.

### Pre-cache during install

```javascript
self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open('static-v1').then(cache => {
      return cache.addAll([
        '/',
        '/index.html',
        '/styles.css',
        '/app.js',
        '/logo.png',
        '/offline.html'
      ]);
    })
  );
});
```

### Clean old caches during activate

```javascript
self.addEventListener('activate', (event) => {
  const keep = ['static-v2', 'dynamic-v1'];
  event.waitUntil(
    caches.keys().then(keys =>
      Promise.all(
        keys.filter(k => !keep.includes(k))
            .map(k => caches.delete(k))
      )
    )
  );
});
```

### Core Cache Methods

| Method | Description |
|--------|-------------|
| `caches.open(name)` | Open/create a named cache |
| `cache.add(url)` | Fetch & store a response |
| `cache.addAll(urls)` | Fetch & store multiple |
| `cache.put(req, res)` | Manually store a pair |
| `cache.match(req)` | Find matching response |
| `cache.delete(req)` | Remove a cached item |
| `cache.keys()` | List all cached requests |

### Dynamic caching during fetch

```javascript
self.addEventListener('fetch', (event) => {
  event.respondWith(
    caches.match(event.request).then(cached => {
      if (cached) return cached;

      return fetch(event.request).then(response => {
        const clone = response.clone();
        caches.open('dynamic-v1').then(cache => {
          cache.put(event.request, clone);
        });
        return response;
      });
    })
  );
});
```

---

## Slide 13 -- Caching Strategies

### Cache First (Cache Falling Back to Network)

Best for: static assets, fonts, images. Fast loads from cache; network only if cache miss.

```javascript
async function cacheFirst(request) {
  const cached = await caches.match(request);
  if (cached) return cached;
  const response = await fetch(request);
  const cache = await caches.open('static-v1');
  cache.put(request, response.clone());
  return response;
}
```

### Network First (Network Falling Back to Cache)

Best for: API calls, frequently updated content. Fresh data when online; cached fallback offline.

```javascript
async function networkFirst(request) {
  try {
    const response = await fetch(request);
    const cache = await caches.open('dynamic-v1');
    cache.put(request, response.clone());
    return response;
  } catch (err) {
    return caches.match(request);
  }
}
```

### Stale-While-Revalidate

Best for: semi-dynamic content (avatars, articles). Instant response from cache + background update.

```javascript
async function staleWhileRevalidate(request) {
  const cache = await caches.open('swr-v1');
  const cached = await cache.match(request);
  const fetchPromise = fetch(request).then(res => {
    cache.put(request, res.clone());
    return res;
  });
  return cached || fetchPromise;
}
```

### Cache Only / Network Only

**Cache Only**: pre-cached assets that never change. **Network Only**: non-GET requests, analytics pings.

```javascript
// Cache only
const cacheOnly = (req) => caches.match(req);

// Network only
const networkOnly = (req) => fetch(req);
```

---

## Slide 14 -- Offline-First Architecture

### App Shell Model

Pre-cache the minimal HTML, CSS, JS **"shell"** at install time. Dynamic content loaded separately. Shell renders instantly even offline.

```javascript
// sw.js — App Shell caching
const SHELL_ASSETS = [
  '/shell.html',
  '/css/app.css',
  '/js/app.js',
  '/js/router.js',
  '/images/logo.svg'
];

self.addEventListener('install', (e) => {
  e.waitUntil(
    caches.open('shell-v3')
      .then(c => c.addAll(SHELL_ASSETS))
  );
});

// Navigate requests → shell
self.addEventListener('fetch', (e) => {
  if (e.request.mode === 'navigate') {
    e.respondWith(
      caches.match('/shell.html')
    );
    return;
  }
  // Other requests: strategy per type...
});
```

### Background Sync API

Queue operations when offline, replay when connectivity returns. User never sees a failure.

```javascript
// Register a sync event
async function saveComment(data) {
  // Save to IndexedDB first
  await db.comments.add(data);

  // Request background sync
  const reg = await navigator
    .serviceWorker.ready;
  await reg.sync.register('sync-comments');
}

// sw.js — handle sync
self.addEventListener('sync', (event) => {
  if (event.tag === 'sync-comments') {
    event.waitUntil(
      pushPendingComments()
    );
  }
});
```

### Offline Fallback Pattern

```javascript
// Serve offline page for nav failures
e.respondWith(
  fetch(e.request).catch(() =>
    caches.match('/offline.html')
  )
);
```

---

## Slide 15 -- Push Notifications

### 1. Subscribe (Client-Side)

```javascript
// Request notification permission
const permission = await Notification.requestPermission();
if (permission !== 'granted') return;

// Subscribe to push
const reg = await navigator.serviceWorker.ready;
const subscription = await reg.pushManager.subscribe({
  userVisibleNotification: true,
  applicationServerKey: urlBase64ToUint8Array(
    'BEl62iUYgUivxIkv69yViE...' // VAPID public key
  )
});

// Send subscription to your server
await fetch('/api/push/subscribe', {
  method: 'POST',
  body: JSON.stringify(subscription),
  headers: { 'Content-Type': 'application/json' }
});
```

### VAPID Keys

Voluntary Application Server Identification. A key pair that identifies your server to the push service. Generate with `web-push generate-vapid-keys`.

### 2. Handle Push (Service Worker)

```javascript
// sw.js — receive push event
self.addEventListener('push', (event) => {
  const data = event.data?.json() ?? {};

  const options = {
    body: data.body || 'New notification',
    icon: '/icons/icon-192.png',
    badge: '/icons/badge-72.png',
    image: data.image,
    vibrate: [200, 100, 200],
    tag: data.tag || 'default',  // group
    renotify: true,
    actions: [
      { action: 'open', title: 'Open App' },
      { action: 'dismiss', title: 'Dismiss' }
    ],
    data: { url: data.url || '/' }
  };

  event.waitUntil(
    self.registration.showNotification(
      data.title || 'App Update', options
    )
  );
});
```

### 3. Handle Click

```javascript
self.addEventListener('notificationclick', (e) => {
  e.notification.close();
  if (e.action === 'open' || !e.action) {
    e.waitUntil(
      clients.openWindow(e.notification.data.url)
    );
  }
});
```

---

## Slide 16 -- Progressive Web Apps

### manifest.json

```json
{
  "name": "My Awesome App",
  "short_name": "AwesomeApp",
  "description": "A full-featured PWA",
  "start_url": "/",
  "display": "standalone",
  "background_color": "#0a0a0f",
  "theme_color": "#f5a623",
  "orientation": "portrait",
  "icons": [
    {
      "src": "/icons/icon-192.png",
      "sizes": "192x192",
      "type": "image/png",
      "purpose": "any maskable"
    },
    {
      "src": "/icons/icon-512.png",
      "sizes": "512x512",
      "type": "image/png"
    }
  ]
}
```

```html
<!-- Link in HTML head -->
<link rel="manifest" href="/manifest.json">
<meta name="theme-color" content="#f5a623">
```

### PWA Installability Criteria

- Served over **HTTPS**
- Has a **web app manifest** with required fields
- Registers a **service worker** with a fetch handler
- Icons at 192px and 512px
- `start_url` is cacheable / works offline

### Install Prompt

```javascript
let deferredPrompt;
window.addEventListener(
  'beforeinstallprompt', (e) => {
    e.preventDefault();
    deferredPrompt = e;
    showInstallButton();
  }
);

installBtn.onclick = async () => {
  deferredPrompt.prompt();
  const { outcome } = await
    deferredPrompt.userChoice;
  console.log(outcome); // 'accepted'
};
```

### Display Modes

`fullscreen` - `standalone` - `minimal-ui` - `browser`

---

## Slide 17 -- Workbox -- Google's SW Toolkit

**Workbox** provides production-ready modules for common service worker patterns -- routing, caching strategies, precaching, background sync, and more.

### Using Workbox via CDN

```javascript
// sw.js — import Workbox from CDN
importScripts(
  'https://storage.googleapis.com/workbox-cdn' +
  '/releases/7.0.0/workbox-sw.js'
);

const { registerRoute } = workbox.routing;
const { CacheFirst, StaleWhileRevalidate, NetworkFirst }
  = workbox.strategies;
const { precacheAndRoute } = workbox.precaching;
const { ExpirationPlugin } = workbox.expiration;

// Precache app shell (generated by build tool)
precacheAndRoute(self.__WB_MANIFEST || []);

// Cache images: cache-first, max 60, 30 days
registerRoute(
  ({ request }) => request.destination === 'image',
  new CacheFirst({
    cacheName: 'images',
    plugins: [
      new ExpirationPlugin({
        maxEntries: 60,
        maxAgeSeconds: 30 * 24 * 60 * 60,
      }),
    ],
  })
);
```

### Route-Based Strategies

```javascript
// CSS/JS: stale-while-revalidate
registerRoute(
  ({ request }) =>
    request.destination === 'style' ||
    request.destination === 'script',
  new StaleWhileRevalidate({
    cacheName: 'static-resources',
  })
);

// API calls: network-first with timeout
registerRoute(
  ({ url }) => url.pathname.startsWith('/api/'),
  new NetworkFirst({
    cacheName: 'api-cache',
    networkTimeoutSeconds: 3,
  })
);

// Navigation: network-first with offline fallback
import { NavigationRoute } from 'workbox-routing';
import { NetworkFirst } from 'workbox-strategies';

const navHandler = new NetworkFirst({
  cacheName: 'navigations',
});
registerRoute(new NavigationRoute(navHandler));
```

### Build Integration

`workbox-cli`, `workbox-webpack-plugin`, and `workbox-build` generate the precache manifest automatically at build time.

---

## Slide 18 -- Debugging Workers

### Chrome DevTools

- **Application > Service Workers** -- view status, skip waiting, unregister
- **Application > Cache Storage** -- inspect cached items
- **Sources panel** -- set breakpoints in worker scripts
- **Network panel** -- see which requests served from SW (gear icon: "from ServiceWorker")
- `chrome://inspect/#service-workers` -- all registered SWs
- `chrome://serviceworker-internals` -- detailed internal state

### DevTools Tips

- Check **"Update on reload"** during development
- Use **"Offline"** checkbox to test offline behavior
- Application > Storage > **"Clear site data"** for fresh start
- Lighthouse PWA audit checks installability

### Logging Strategies

```javascript
// Conditional logging for workers
const DEBUG = true;
const log = (...args) => {
  if (DEBUG) console.log('[SW]', ...args);
};

self.addEventListener('fetch', (e) => {
  log('Fetch:', e.request.url);
  // ...
});
```

### Dedicated Worker Debugging

```javascript
// Workers appear in Sources panel
// under "Threads" section

// Use console.log in worker
self.onmessage = (e) => {
  console.log('Worker received:', e.data);
  console.time('processing');
  // ... work ...
  console.timeEnd('processing');
};
```

### Common Pitfalls

- SW cached an old version -- use versioned cache names
- Scope mismatch -- check `reg.scope` vs expected
- `event.waitUntil()` not used -- SW killed mid-operation
- Not cloning responses before caching (`response.clone()`)

---

## Slide 19 -- Performance Patterns

### Worker Pools

Spin up N workers to parallelize work across CPU cores.

```javascript
class WorkerPool {
  constructor(script, size = navigator.hardwareConcurrency) {
    this.workers = Array.from({ length: size },
      () => new Worker(script));
    this.queue = [];
    this.free = [...this.workers];
  }

  exec(data) {
    return new Promise((resolve) => {
      const task = { data, resolve };
      const w = this.free.pop();
      if (w) this._run(w, task);
      else this.queue.push(task);
    });
  }

  _run(worker, task) {
    worker.onmessage = (e) => {
      task.resolve(e.data);
      const next = this.queue.shift();
      if (next) this._run(worker, next);
      else this.free.push(worker);
    };
    worker.postMessage(task.data);
  }
}
```

### Comlink -- RPC for Workers

Google's library wraps `postMessage` with Proxy-based RPC. Call worker functions as if they were local async functions.

```javascript
// worker.js
import * as Comlink from 'comlink';

const api = {
  fibonacci(n) {
    if (n <= 1) return n;
    return api.fibonacci(n-1) + api.fibonacci(n-2);
  },
  async processImage(data) { /* ... */ }
};
Comlink.expose(api);

// main.js
import * as Comlink from 'comlink';
const api = Comlink.wrap(
  new Worker('worker.js', { type: 'module' })
);
const result = await api.fibonacci(42); // magic!
```

### OffscreenCanvas

Render canvas graphics in a worker -- no main thread cost.

```javascript
// main.js
const canvas = document.getElementById('c');
const offscreen = canvas.transferControlToOffscreen();
worker.postMessage({ canvas: offscreen }, [offscreen]);

// worker.js — render loop off main thread
self.onmessage = ({ data }) => {
  const ctx = data.canvas.getContext('2d');
  function draw() { /* ... */ requestAnimationFrame(draw); }
  draw();
};
```

### SharedArrayBuffer

True shared memory between threads. Requires `Cross-Origin-Isolation` headers. Use `Atomics` for synchronization.

```javascript
// Shared memory between main + worker
const sab = new SharedArrayBuffer(1024);
const arr = new Int32Array(sab);
worker.postMessage({ buffer: sab });

// Worker: Atomics for thread-safe ops
Atomics.add(arr, 0, 1);
Atomics.wait(arr, 0, expectedVal);
Atomics.notify(arr, 0);
```

---

## Slide 20 -- Summary & Next Steps

### Dedicated Workers

- Offload CPU-heavy tasks
- Message passing via `postMessage`
- Transfer large data with transferables
- Use worker pools for parallelism
- Comlink for ergonomic RPC

### Service Workers

- Proxy between browser & network
- Lifecycle: install -> activate -> fetch
- Cache API for offline support
- Strategies: cache-first, network-first, SWR
- Push notifications & background sync

### PWA Ecosystem

- manifest.json for installability
- App shell model for instant loads
- Workbox for production-ready SW
- Lighthouse for PWA auditing
- OffscreenCanvas & SharedArrayBuffer

### Further Reading & Resources

- [MDN -- Web Workers API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API)
- [MDN -- Service Worker API](https://developer.mozilla.org/en-US/docs/Web/API/Service_Worker_API)
- [web.dev -- Learn PWA](https://web.dev/learn/pwa)
- [Chrome Developers -- Workbox](https://developer.chrome.com/docs/workbox/)
- [TC39 Shared Structs Proposal](https://github.com/nicolo-ribaudo/tc39-proposal-structs)
