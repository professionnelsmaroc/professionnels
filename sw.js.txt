/* Service Worker — Professionnels Maroc
   Estrategia: network-first con fallback a caché.
   IMPORTANTE: solo intercepta peticiones GET del MISMO origen.
   NUNCA toca llamadas externas (Supabase, fuentes, flags, analytics). */
const CACHE = "pm-cache-v1";
const CORE = [
  "/",
  "/index.html",
  "/manifest.webmanifest",
  "/icon.svg",
  "/icon-192.png",
  "/icon-512.png"
];

self.addEventListener("install", (e) => {
  e.waitUntil(
    caches.open(CACHE).then((c) => c.addAll(CORE)).catch(() => {})
  );
  self.skipWaiting();
});

self.addEventListener("activate", (e) => {
  e.waitUntil(
    caches.keys()
      .then((keys) => Promise.all(keys.filter((k) => k !== CACHE).map((k) => caches.delete(k))))
      .then(() => self.clients.claim())
  );
});

self.addEventListener("fetch", (e) => {
  const req = e.request;
  if (req.method !== "GET") return;                    // no tocar POST/PUT (APIs)
  const url = new URL(req.url);
  if (url.origin !== self.location.origin) return;     // no tocar dominios externos (Supabase, etc.)

  e.respondWith(
    fetch(req)
      .then((res) => {
        if (res && res.ok) {
          const copy = res.clone();
          caches.open(CACHE).then((c) => c.put(req, copy)).catch(() => {});
        }
        return res;
      })
      .catch(() =>
        caches.match(req).then((hit) => {
          if (hit) return hit;
          if (req.mode === "navigate") {
            return caches.match("/index.html").then((m) => m || new Response("Hors ligne", { status: 503, headers: { "Content-Type": "text/plain" } }));
          }
          return new Response("Hors ligne", { status: 503, headers: { "Content-Type": "text/plain" } });
        })
      )
  );
});
