/* ══════════════════════════════════════════════
   MUFASA STRENGTH EQUIPMENT — Service Worker
   sw.js  |  Cache-first + network fallback
   ══════════════════════════════════════════════ */

const CACHE_NAME    = 'mufasa-v1';
const GUIDE_CACHE   = 'mufasa-guide-v1';

/* ── Assets to pre-cache on install ── */
const PRECACHE_URLS = [
  '/',
  '/index.html',
  '/mufasa-guide.html',
  '/manifest.json',
  '/mufasa-header-logo.png',
  '/hero-bg.png',
  'https://fonts.googleapis.com/css2?family=Anton&family=Barlow+Condensed:ital,wght@0,300;0,400;0,600;0,700;1,400&family=Barlow:wght@300;400;500;600&display=swap'
];

/* ── Offline fallback page (injected as a string) ── */
const OFFLINE_HTML = `<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width,initial-scale=1">
<title>MUFASA — Offline</title>
<style>
  :root{--g:#D4942A;--gl:#F0B84A;--bg:#050403;--w:#F4EFE4;--grey:#A09890;}
  *{margin:0;padding:0;box-sizing:border-box;}
  body{font-family:'Barlow',system-ui,sans-serif;background:var(--bg);color:var(--w);
    display:flex;align-items:center;justify-content:center;min-height:100svh;text-align:center;padding:24px;}
  .wrap{max-width:360px;}
  .lion{font-size:5rem;margin-bottom:16px;display:block;opacity:.6;}
  h1{font-family:'Anton',system-ui,sans-serif;font-size:2.4rem;color:var(--g);letter-spacing:.06em;margin-bottom:10px;}
  p{color:var(--grey);font-size:.95rem;line-height:1.6;margin-bottom:24px;}
  button{background:linear-gradient(135deg,#A8741F,#F0B84A);color:#050403;border:none;
    padding:12px 28px;border-radius:6px;font-family:'Barlow Condensed',system-ui,sans-serif;
    font-size:.9rem;font-weight:700;letter-spacing:.14em;text-transform:uppercase;cursor:pointer;}
  button:hover{opacity:.88;}
</style>
</head>
<body>
<div class="wrap">
  <span class="lion">🦁</span>
  <h1>YOU'RE OFFLINE</h1>
  <p>No connection detected. Check your internet and try again — the Pride awaits.</p>
  <button onclick="location.reload()">⚡ Retry</button>
</div>
</body>
</html>`;

/* ══════════════════════════════════════════════
   INSTALL — pre-cache core assets
   ══════════════════════════════════════════════ */
self.addEventListener('install', event => {
  event.waitUntil(
    (async () => {
      const cache = await caches.open(CACHE_NAME);

      /* Cache each URL individually so one failure doesn't block the rest */
      await Promise.allSettled(
        PRECACHE_URLS.map(url =>
          cache.add(url).catch(err => console.warn('[SW] Pre-cache skip:', url, err))
        )
      );

      /* Store offline fallback */
      await cache.put(
        new Request('/__offline__'),
        new Response(OFFLINE_HTML, { headers: { 'Content-Type': 'text/html; charset=utf-8' } })
      );

      await self.skipWaiting();
    })()
  );
});

/* ══════════════════════════════════════════════
   ACTIVATE — delete old caches
   ══════════════════════════════════════════════ */
self.addEventListener('activate', event => {
  event.waitUntil(
    (async () => {
      const keys = await caches.keys();
      await Promise.all(
        keys
          .filter(k => k !== CACHE_NAME && k !== GUIDE_CACHE)
          .map(k => caches.delete(k))
      );
      await self.clients.claim();
    })()
  );
});

/* ══════════════════════════════════════════════
   FETCH — strategy per request type
   ══════════════════════════════════════════════ */
self.addEventListener('fetch', event => {
  const { request } = event;
  const url = new URL(request.url);

  /* ── Skip non-GET, Chrome extensions, Firebase, WhatsApp ── */
  if (request.method !== 'GET') return;
  if (!url.protocol.startsWith('http')) return;
  if (url.hostname.includes('firebaseio.com'))  return;
  if (url.hostname.includes('firebaseapp.com')) return;
  if (url.hostname.includes('googleapis.com') && url.pathname.includes('/identitytoolkit')) return;
  if (url.hostname.includes('wa.me')) return;

  /* ── Guide file → cache-first (guide has its own cache slot) ── */
  if (url.pathname.endsWith('mufasa-guide.html')) {
    event.respondWith(cacheFirst(request, GUIDE_CACHE));
    return;
  }

  /* ── Google Fonts → stale-while-revalidate ── */
  if (url.hostname.includes('fonts.googleapis.com') || url.hostname.includes('fonts.gstatic.com')) {
    event.respondWith(staleWhileRevalidate(request, CACHE_NAME));
    return;
  }

  /* ── Static assets (images, CSS, JS, JSON) → cache-first ── */
  const isStatic = /\.(png|jpg|jpeg|webp|svg|ico|gif|woff2?|css|js|json)(\?.*)?$/.test(url.pathname);
  if (isStatic) {
    event.respondWith(cacheFirst(request, CACHE_NAME));
    return;
  }

  /* ── HTML pages → network-first with offline fallback ── */
  if (request.headers.get('accept')?.includes('text/html')) {
    event.respondWith(networkFirstWithFallback(request));
    return;
  }

  /* ── Everything else → network-first ── */
  event.respondWith(networkFirst(request));
});

/* ══════════════════════════════════════════════
   STRATEGY HELPERS
   ══════════════════════════════════════════════ */

/** Cache-first: serve from cache; fetch & update if missing */
async function cacheFirst(request, cacheName) {
  const cache    = await caches.open(cacheName);
  const cached   = await cache.match(request);
  if (cached) return cached;

  try {
    const response = await fetch(request);
    if (response.ok) cache.put(request, response.clone());
    return response;
  } catch {
    return fallbackResponse(request);
  }
}

/** Network-first: try network, fall back to cache */
async function networkFirst(request) {
  try {
    const response = await fetch(request);
    if (response.ok) {
      const cache = await caches.open(CACHE_NAME);
      cache.put(request, response.clone());
    }
    return response;
  } catch {
    const cache  = await caches.open(CACHE_NAME);
    const cached = await cache.match(request);
    return cached || fallbackResponse(request);
  }
}

/** Network-first for HTML with friendly offline fallback */
async function networkFirstWithFallback(request) {
  try {
    const response = await fetch(request);
    if (response.ok) {
      const cache = await caches.open(CACHE_NAME);
      cache.put(request, response.clone());
    }
    return response;
  } catch {
    const cache  = await caches.open(CACHE_NAME);
    const cached = await cache.match(request);
    if (cached) return cached;

    /* Return the baked-in offline page */
    const offlinePage = await cache.match('/__offline__');
    return offlinePage || new Response(OFFLINE_HTML, {
      headers: { 'Content-Type': 'text/html; charset=utf-8' }
    });
  }
}

/** Stale-while-revalidate: serve cache immediately, update in background */
async function staleWhileRevalidate(request, cacheName) {
  const cache  = await caches.open(cacheName);
  const cached = await cache.match(request);

  const fetchPromise = fetch(request).then(response => {
    if (response.ok) cache.put(request, response.clone());
    return response;
  }).catch(() => null);

  return cached || await fetchPromise || fallbackResponse(request);
}

/** Last-resort fallback */
function fallbackResponse(request) {
  if (request.headers.get('accept')?.includes('text/html')) {
    return new Response(OFFLINE_HTML, {
      headers: { 'Content-Type': 'text/html; charset=utf-8' }
    });
  }
  return new Response('', { status: 408, statusText: 'Offline' });
}

/* ══════════════════════════════════════════════
   BACKGROUND SYNC — retry failed WhatsApp opens
   (fires when connectivity is restored)
   ══════════════════════════════════════════════ */
self.addEventListener('sync', event => {
  if (event.tag === 'mufasa-order-retry') {
    event.waitUntil(handleOrderRetry());
  }
});

async function handleOrderRetry() {
  /* Orders are opened via window.open() on the client side,
     so this is a no-op placeholder for future push/background use. */
}

/* ══════════════════════════════════════════════
   PUSH NOTIFICATIONS (future-ready stub)
   ══════════════════════════════════════════════ */
self.addEventListener('push', event => {
  const data = event.data?.json() ?? {};
  const title   = data.title   || 'MUFASA STRENGTH';
  const options = {
    body:    data.body    || 'New update from the Pride 🦁',
    icon:    data.icon    || '/mufasa-header-logo.png',
    badge:   data.badge   || '/mufasa-header-logo.png',
    vibrate: [200, 100, 200],
    data:    { url: data.url || '/' },
    actions: [
      { action: 'open',    title: 'View Now' },
      { action: 'dismiss', title: 'Dismiss'  }
    ]
  };
  event.waitUntil(self.registration.showNotification(title, options));
});

self.addEventListener('notificationclick', event => {
  event.notification.close();
  if (event.action === 'dismiss') return;
  const url = event.notification.data?.url || '/';
  event.waitUntil(
    clients.matchAll({ type: 'window', includeUncontrolled: true }).then(list => {
      for (const client of list) {
        if (client.url === url && 'focus' in client) return client.focus();
      }
      if (clients.openWindow) return clients.openWindow(url);
    })
  );
});
