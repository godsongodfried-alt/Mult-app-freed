// Service Worker pour Mult - PWA
// Auteur: Ing JOHNSON JOSIAS - EPAC_UAC
// Version: 1.0.0

const CACHE_NAME = 'mult-v1.0.0';
const urlsToCache = [
  '/',
  '/index.html',
  '/manifest.json',
  '/icon-72.png',
  '/icon-96.png',
  '/icon-128.png',
  '/icon-144.png',
  '/icon-152.png',
  '/icon-192.png',
  '/icon-384.png',
  '/icon-512.png'
];

// Installation du Service Worker
self.addEventListener('install', (event) => {
  console.log('[Service Worker] Installation en cours...');
  event.waitUntil(
    caches.open(CACHE_NAME)
      .then((cache) => {
        console.log('[Service Worker] Mise en cache des fichiers');
        return cache.addAll(urlsToCache);
      })
      .then(() => {
        console.log('[Service Worker] Installation terminée');
      })
      .catch((error) => {
        console.error('[Service Worker] Erreur installation:', error);
      })
  );
  // Force l'activation immédiate
  self.skipWaiting();
});

// Activation du Service Worker
self.addEventListener('activate', (event) => {
  console.log('[Service Worker] Activation en cours...');
  event.waitUntil(
    caches.keys().then((cacheNames) => {
      return Promise.all(
        cacheNames.map((cacheName) => {
          if (cacheName !== CACHE_NAME) {
            console.log('[Service Worker] Suppression ancien cache:', cacheName);
            return caches.delete(cacheName);
          }
        })
      );
    }).then(() => {
      console.log('[Service Worker] Activation terminée');
    })
  );
  // Prend le contrôle immédiatement
  self.clients.claim();
});

// Stratégie de mise en cache: Cache First, fallback Network
self.addEventListener('fetch', (event) => {
  event.respondWith(
    caches.match(event.request)
      .then((cachedResponse) => {
        // Si trouvé dans le cache, retourner la version en cache
        if (cachedResponse) {
          console.log('[Service Worker] Chargé depuis cache:', event.request.url);
          return cachedResponse;
        }

        // Sinon, faire une requête réseau
        console.log('[Service Worker] Chargé depuis réseau:', event.request.url);
        
        // Cloner la requête car elle ne peut être utilisée qu'une fois
        const fetchRequest = event.request.clone();

        return fetch(fetchRequest).then((response) => {
          // Vérifier si réponse valide
          if (!response || response.status !== 200 || response.type !== 'basic') {
            return response;
          }

          // Cloner la réponse car elle ne peut être utilisée qu'une fois
          const responseToCache = response.clone();

          // Mettre en cache pour utilisation future
          caches.open(CACHE_NAME)
            .then((cache) => {
              cache.put(event.request, responseToCache);
              console.log('[Service Worker] Mis en cache:', event.request.url);
            });

          return response;
        }).catch((error) => {
          // En cas d'erreur réseau, essayer de retourner la page principale
          console.log('[Service Worker] Erreur réseau, mode hors ligne activé');
          return caches.match('/index.html');
        });
      })
  );
});

// Gestion des messages du client
self.addEventListener('message', (event) => {
  if (event.data && event.data.type === 'SKIP_WAITING') {
    self.skipWaiting();
  }
  
  if (event.data && event.data.type === 'GET_VERSION') {
    event.ports[0].postMessage({ version: CACHE_NAME });
  }
});

// Notification de mise à jour disponible
self.addEventListener('updatefound', () => {
  console.log('[Service Worker] Mise à jour détectée');
});