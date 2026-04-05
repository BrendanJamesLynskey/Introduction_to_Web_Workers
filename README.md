# ⚙ Introduction to Web Workers

An interactive Reveal.js presentation covering Web Workers — dedicated workers, shared workers, service workers, the Cache API, offline-first architecture, push notifications, and Progressive Web Apps.

## ▶ [Open the Presentation](https://brendanjameslynskey.github.io/Introduction_to_Web_Workers/)

---

## Contents

| # | Topic | Description |
|---|-------|-------------|
| 01 | Title | Introduction to Web Workers |
| 02 | Agenda | Four-part overview: Foundations, Dedicated Workers, Service Workers, Advanced |
| 03 | The Single-Threaded Problem | Main thread bottleneck, jank, the 16ms frame budget |
| 04 | What Are Web Workers? | Types overview: Dedicated, Shared, Service Workers |
| 05 | Dedicated Workers | Creating workers, postMessage, terminating |
| 06 | Message Passing | Structured clone algorithm, transferable objects, ArrayBuffer |
| 07 | Worker Scope & Limitations | No DOM access, importScripts, available APIs, ES modules |
| 08 | Real-World: Heavy Computation | Image processing, CSV parsing, cryptography in workers |
| 09 | Shared Workers | Multiple tabs sharing one worker, ports, MessagePort |
| 10 | Service Workers — Lifecycle | Install, activate, fetch events and state transitions |
| 11 | Service Worker Registration | register(), scope, update flow, skipWaiting |
| 12 | Cache API | Cache Storage, core methods, pre-caching and dynamic caching |
| 13 | Caching Strategies | Cache-first, network-first, stale-while-revalidate |
| 14 | Offline-First Architecture | App shell model, background sync API |
| 15 | Push Notifications | Push API, Notification API, VAPID keys, subscription flow |
| 16 | Progressive Web Apps | manifest.json, installability criteria, display modes |
| 17 | Workbox | Google's service worker toolkit, route-based strategies |
| 18 | Debugging Workers | Chrome DevTools, logging strategies, common pitfalls |
| 19 | Performance Patterns | Worker pools, Comlink, OffscreenCanvas, SharedArrayBuffer |
| 20 | Summary & Next Steps | Key takeaways and further reading |

---

## Slide Controls

| Action | Key |
|--------|-----|
| Next / Previous | `→` `←` or swipe |
| Overview | `Esc` |
| Fullscreen | `F` |
| Export to PDF | Append `?print-pdf` to URL, then print |

## Technology

[Reveal.js 4.6](https://revealjs.com) · [highlight.js](https://highlightjs.org) · Playfair Display + DM Sans + JetBrains Mono

Single self-contained `index.html` — no build step, no npm, no dependencies to install.

## References

- [MDN — Web Workers API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API)
- [MDN — Service Worker API](https://developer.mozilla.org/en-US/docs/Web/API/Service_Worker_API)
- [web.dev — Learn PWA](https://web.dev/learn/pwa)
- [Chrome Developers — Workbox](https://developer.chrome.com/docs/workbox/)
- [MDN — Cache API](https://developer.mozilla.org/en-US/docs/Web/API/Cache)
- [MDN — Push API](https://developer.mozilla.org/en-US/docs/Web/API/Push_API)

## License

Educational use. Code examples provided as-is.
