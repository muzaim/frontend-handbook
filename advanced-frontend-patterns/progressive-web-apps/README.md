# Progressive Web Apps (PWA)

## 1. Skenario Sistem Nyata

Bayangkan seorang kurir logistik sedang menggunakan aplikasi web inventaris pengiriman barang di ponselnya. Saat memasuki area basement gedung parkir atau pedalaman terpencil, sinyal seluler hilang total (*dead zone*). Pada website konvensional, setiap klik tombol atau perpindahan tab seketika memicu tampilan layar error browser ("No Internet / Dinosaur Screen"), seluruh input formulir yang belum sempat tersimpan hilang, dan alur kerja operasional lumpuh total.

Dengan arsitektur Progressive Web App (PWA), aplikasi web tetap dapat dibuka secara instan dari homescreen perangkat layaknya aplikasi native, halaman shell dan katalog aset statis dimuat langsung dari cache lokal, data yang diinput kurir disimpan ke database lokal peramban (IndexedDB), dan antrean pengiriman secara otomatis terkirim ke server (*background sync*) begitu perangkat kembali mendeteksi sinyal internet.

## 2. Apa Itu dan Buat Apa?

Progressive Web Apps (PWA) adalah metodologi arsitektur pengembangan web yang memanfaatkan rangkaian Web APIs modern (terutama Service Worker, CacheStorage, Web App Manifest, dan IndexedDB) untuk menghadirkan pengalaman pengguna setara aplikasi native pada platform web terbuka.

Tujuan utama PWA:
- Menghilangkan ketergantungan mutlak pada ketersediaan jaringan internet (*offline-capable*).
- Memangkas waktu muat awal hingga mendekati 0 milidetik pada kunjungan berulang (*instant loading* via App Shell).
- Memberikan kemampuan instalasi aplikasi ke beranda (*homescreen*) sistem operasi tanpa harus melalui App Store atau Play Store.
- Mendukung interaksi latar belakang seperti sinkronisasi data mutasi tertunda (*Background Sync*) dan notifikasi dorong (*Web Push Notifications*).

## 3. Kenapa Harus Pakai Ini?

1. **Jaminan Ketersediaan Bisnis (Business Continuity)**: Aplikasi tidak runtuh saat jaringan bermasalah (*flaky network*). Pengguna tetap dapat melihat data yang sebelumnya telah dibuka dan tetap dapat berinteraksi dengan aplikasi.
2. **Peningkatan Retensi dan Konversi Pengguna**: Ikon aplikasi yang terinstal di homescreen dan kemampuan push notification memicu interaksi berulang tanpa perlu biaya akuisisi ulang pengguna.
3. **Efisiensi Biaya dan Ukuran Bundel**: Ukuran instalasi PWA sering kali di bawah 2 MB, berbanding terbalik dengan aplikasi native (Android/iOS) yang sering kali berukuran 50 MB hingga 200 MB. Satu basis kode web melayani semua platform secara bersamaan.
4. **Bypass Hambatan App Store**: Rilis fitur baru dan perbaikan bug dapat langsung didistribusikan ke pengguna saat Service Worker diperbarui, tanpa harus menunggu review berhari-hari dari pengelola toko aplikasi.

## 4. Cara Kerjanya

Arsitektur PWA berpusat pada **Service Worker**, sebuah script JavaScript independen yang berjalan di background thread peramban, terpisah sepenuhnya dari main thread DOM halaman web. Service Worker bertindak sebagai programmable network proxy di antara browser dan server.

```
+-------------------------------------------------------------------------+
|                  ARSITEKTUR INTERSEPSI REQUEST PWA                      |
+-------------------------------------------------------------------------+
|                                                                         |
|  [ Komponen UI / Main Thread ]                                          |
|        |                                                                |
|   (1) fetch('/api/katalog' atau '/app.js')                              |
|        v                                                                |
|  +-------------------------------------------------------------------+  |
|  | SERVICE WORKER (Background Thread / Network Proxy)                |  |
|  | - Menerima event 'fetch'                                          |  |
|  | - Menentukan rute strategi caching                                |  |
|  +-------------------------------------------------------------------+  |
|        |                                        |                       |
|   (2a) Evaluasi Cache                     (2b) Network Fallback / SWR   |
|        v                                        v                       |
|  +---------------------------+            +--------------------------+  |
|  | CacheStorage (Cache API)  |            | Jaringan Eksternal       |  |
|  | - HTML Shell, CSS, JS     |            | - API Server             |  |
|  | - Gambar Statis           |            | - CDN Aset               |  |
|  +---------------------------+            +--------------------------+  |
|        |                                        |                       |
|   (3a) Cache Hit: Return Response          (3b) Response Sukses: Put    |
|        v                                        v   ke CacheStorage     |
|  +-------------------------------------------------------------------+  |
|  | Mengembalikan Response ke UI (Transparan bagi fetch client)       |  |
|  +-------------------------------------------------------------------+  |
|                                                                         |
+-------------------------------------------------------------------------+
```

### 1. Siklus Hidup Service Worker (Lifecycle)
Service Worker memiliki siklus hidup yang ketat:
- **Registration**: Halaman web mendaftarkan file `sw.js` melalui `navigator.serviceWorker.register()`.
- **Installation (`install`)**: Service Worker mengunduh dan menyimpan aset kritis (*App Shell pre-caching*).
- **Waiting**: Jika ada Service Worker versi baru, worker baru masuk status *waiting* sampai semua tab yang membuka worker lama ditutup oleh pengguna.
- **Activation (`activate`)**: Worker baru mengambil alih kendali dan membersihkan cache lama yang sudah usang (*cache cleanup*).

### 2. Lima Strategi Caching Utama
- **Cache-First (Fallback to Network)**: Cocok untuk aset ber-hash (CSS, JS, Font, logo). Cek cache dulu; jika tidak ada, baru ambil ke jaringan dan simpan ke cache.
- **Network-First (Fallback to Cache)**: Cocok untuk data realtime penting (misal saldo pengguna). Coba jaringan dulu; jika offline/timeout, ambil data terakhir dari cache.
- **Stale-While-Revalidate**: Cocok untuk avatar, feed berita, dan profil. Kembalikan data lama dari cache secara instan ke UI, lalu di latar belakang lakukan fetch jaringan untuk memperbarui cache untuk kunjungan berikutnya.
- **Network-Only**: Cocok untuk transaksi sensitif non-idempotent (misal otentikasi login, pembayaran kartu kredit).
- **Cache-Only**: Cocok untuk aset statis internal yang sudah dijamin ada sejak proses instalasi awal.

## 5. Cuplikan Logika Inti

### 1. Mendaftarkan Service Worker dan Menangani Update UX (Main Thread)

```javascript
// index.js (Main Thread)
if ('serviceWorker' in navigator) {
  window.addEventListener('load', async () => {
    try {
      const reg = await navigator.serviceWorker.register('/sw.js');

      // Deteksi jika ada Service Worker baru yang sedang menunggu aktivasi
      reg.addEventListener('updatefound', () => {
        const newWorker = reg.installing;
        newWorker.addEventListener('statechange', () => {
          if (newWorker.state === 'installed' && navigator.serviceWorker.controller) {
            // Tampilkan UI Toast/Banner kepada pengguna
            showUpdatePrompt(() => {
              newWorker.postMessage({ type: 'SKIP_WAITING' });
            });
          }
        });
      });
    } catch (err) {
      console.error('Gagal meregistrasi Service Worker:', err);
    }
  });

  // Reload halaman secara otomatis saat worker baru mengambil kendali
  let refreshing = false;
  navigator.serviceWorker.addEventListener('controllerchange', () => {
    if (!refreshing) {
      refreshing = true;
      window.location.reload();
    }
  });
}

function showUpdatePrompt(onConfirm) {
  const banner = document.createElement('div');
  banner.textContent = 'Versi baru tersedia. ';
  const btn = document.createElement('button');
  btn.textContent = 'Perbarui Sekarang';
  btn.onclick = onConfirm;
  banner.appendChild(btn);
  document.body.appendChild(banner);
}
```

### 2. Implementasi Service Worker: Lifecycle dan Strategi Cache (sw.js)

```javascript
// sw.js (Background Thread)
const CACHE_NAME = 'app-shell-v2';
const STATIC_ASSETS = [
  '/',
  '/index.html',
  '/styles.css',
  '/app.js',
  '/offline.html'
];

// 1. Install Event: Pre-cache App Shell
self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open(CACHE_NAME).then((cache) => {
      return cache.addAll(STATIC_ASSETS);
    })
  );
});

// 2. Activate Event: Bersihkan cache versi lama
self.addEventListener('activate', (event) => {
  event.waitUntil(
    caches.keys().then((keys) => {
      return Promise.all(
        keys.map((key) => {
          if (key !== CACHE_NAME) {
            return caches.delete(key);
          }
        })
      );
    }).then(() => self.clients.claim())
  );
});

// 3. Listener untuk paksa aktivasi tanpa menunggu tab ditutup
self.addEventListener('message', (event) => {
  if (event.data && event.data.type === 'SKIP_WAITING') {
    self.skipWaiting();
  }
});

// 4. Fetch Event: Strategi Campuran (Stale-While-Revalidate untuk Aset, Network-First untuk Navigasi)
self.addEventListener('fetch', (event) => {
  const { request } = event;
  const url = new URL(request.url);

  // Jangan intersep request non-GET atau request otentikasi eksternal
  if (request.method !== 'GET' || url.origin !== location.origin) {
    return;
  }

  // Strategi Navigasi Halaman (HTML): Network-First dengan Fallback ke offline.html
  if (request.mode === 'navigate') {
    event.respondWith(
      fetch(request).catch(() => {
        return caches.match('/offline.html');
      })
    );
    return;
  }

  // Strategi Aset Statis: Stale-While-Revalidate
  event.respondWith(
    caches.open(CACHE_NAME).then(async (cache) => {
      const cachedResponse = await cache.match(request);
      const networkFetchPromise = fetch(request).then((networkResponse) => {
        if (networkResponse.status === 200) {
          cache.put(request, networkResponse.clone());
        }
        return networkResponse;
      }).catch(() => null);

      return cachedResponse || networkFetchPromise;
    })
  );
});
```

### 3. Menangkap Custom Install Prompt (`beforeinstallprompt`)

```javascript
// install-prompt.js (Main Thread)
let deferredPrompt = null;
const installButton = document.getElementById('btn-install');

window.addEventListener('beforeinstallprompt', (e) => {
  // Mencegah mini-infobar default browser agar tidak muncul tiba-tiba
  e.preventDefault();
  deferredPrompt = e;
  // Munculkan tombol install buatan kita sendiri di UI
  installButton.style.display = 'block';
});

installButton.addEventListener('click', async () => {
  if (!deferredPrompt) return;
  // Munculkan prompt dialog instalasi bawaan OS
  deferredPrompt.prompt();
  const { outcome } = await deferredPrompt.userChoice;
  if (outcome === 'accepted') {
    console.log('Pengguna menyetujui instalasi PWA');
  }
  deferredPrompt = null;
  installButton.style.display = 'none';
});
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

| Kelebihan | Kekurangan / Batasan |
| :--- | :--- |
| **Aksesibilitas Universal**: Berjalan di semua peramban modern tanpa instalasi wajib | **Batasan Akses Hardware iOS/Safari**: Fitur seperti Bluetooth, WebUSB, dan Background Sync dibatasi di ekosistem Apple WebKit |
| **Performa Instan**: CacheStorage memotong latency jaringan hingga mendekati 0ms | **Storage Quota & Eviction**: Browser sewaktu-waktu dapat menghapus cache jika disk memori perangkat penuh |
| **Hemat Kuota & Bandwidth**: Hanya mengunduh aset yang benar-benar mengalami pembaruan | **Kompleksitas Debugging Cache**: Bug pada logika Service Worker dapat menyebabkan pengguna terkunci pada versi web usang |
| **Bisa Offline**: Aplikasi tetap beroperasi saat koneksi internet putus | **Tidak Memiliki Validasi Toko Resmi**: Tidak ada endorsement review resmi dari toko aplikasi untuk verifikasi kepercayaan pengguna |

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Pakai
- **Aplikasi Lapangan (Field Service / Operational Tool)**: Driver pengiriman, teknisi lapangan, atau pencatat data gudang yang sering berada di zona minim sinyal.
- **E-Commerce & Media Publikasi**: Toko online atau portal berita di mana kecepatan muat awal dan retensi pembaca sangat berkorelasi langsung dengan omzet/iklan.
- **Aplikasi Produktivitas (Notetaking / Dashboard)**: Aplikasi seperti Trello, Notion, atau Spotify Web di mana pengguna menginginkan akses instan tanpa jeda pemuatan halaman.

### Kapan Jangan Pakai
- **Website Statis Sederhana**: Blog portofolio pribadi atau landing page promosi singkat yang jarang dikunjungi berulang. Overhead pembuatan dan maintenance Service Worker tidak sebanding dengan manfaatnya.
- **Aplikasi yang Membutuhkan Deep Hardware Integration**: Aplikasi yang membutuhkan background location tracking berkelanjutan, pemindaian NFC intensif, atau integrasi sensor hardware low-level yang hanya didukung oleh native SDK.

## 8. Metrik Kunci & Observabilitas

Untuk mengukur efektivitas dan kesehatan PWA di lingkungan produksi:
1. **Cache Hit Ratio**: Persentase request aset yang berhasil dilayani dari CacheStorage tanpa harus menyentuh jaringan server. Target ideal: di atas 85% untuk aset statis.
2. **First Contentful Paint (FCP) & Time to Interactive (TTI)**: Perbandingan waktu muat antara kunjungan pertama (*cold load*) vs kunjungan kedua dengan Service Worker aktif (*warm load*). Pada warm load, FCP idealnya berada di bawah 0.8 detik.
3. **PWA Installation Rate**: Rasio persentase pengguna yang memicu `beforeinstallprompt` dan mengklik "Install" ke homescreen mereka.
4. **Service Worker Error Tracking**: Monitoring error pada event `install`, `activate`, atau error parsing JavaScript di dalam worker melalui pelaporan Sentry khusus scope Service Worker.

## 9. Studi Kasus Industri Nyata

- **Twitter Lite**: Twitter mendesain ulang arsitektur mobile web mereka menjadi PWA dengan pendekatan App Shell dan Service Worker caching. Hasilnya: konsumsi data berkurang hingga 70%, ukuran aplikasi hanya 600 KB (dibandingkan native Android 23 MB), waktu interaksi turun 30%, dan terjadi peningkatan 65% pada pages per session.
- **Starbucks PWA**: Starbucks meluncurkan PWA untuk pasar global yang mencakup negara berkembang dengan koneksi seluler lambat. PWA ini berukuran 99% lebih kecil dibanding native app iOS mereka, memungkinkan pengguna menjelajahi menu, mengustomisasi pesanan, dan menambahkan barang ke keranjang belanja sepenuhnya secara offline. Hasilnya, jumlah pesanan via web desktop dan mobile melonjak 2 kali lipat.

## 10. Fakta Menarik & Gotchas Production

1. **Jebakan `Cache-Control` pada Berkas `sw.js`**: Jika web server memberikan header `Cache-Control: max-age=31536000` pada file `sw.js`, browser mungkin tidak akan pernah memeriksa pembaruan Service Worker baru selama satu tahun! Pastikan server selalu menyajikan `sw.js` dengan header `Cache-Control: no-cache, no-store, must-revalidate`.
2. **Scope Directory Service Worker**: Lokasi file Service Worker menentukan cakupan intercept-nya. Jika ditaruh di `/assets/sw.js`, ia hanya bisa mencegat request di bawah `/assets/*`. Agar dapat mengontrol seluruh domain, `sw.js` wajib berada di root web (`/sw.js`) atau menggunakan response header `Service-Worker-Allowed: /`.
3. **Hati-Hati dengan Pola Pre-Caching Terlalu Agresif**: Jangan memasukkan ratusan gambar produk atau rute dinamis ke dalam array `addAll()` pada saat event `install`. Jika ada **satu saja** file di array tersebut yang gagal diunduh (misal 404), seluruh proses instalasi Service Worker akan dibatalkan (*fail-to-install*).

## 11. Rangkuman Singkat

Progressive Web Apps (PWA) mengubah website konvensional menjadi aplikasi tangguh berkinerja tinggi yang mampu berjalan offline, dapat diinstal langsung ke homescreen, dan memuat aset secara instan via CacheStorage. Komponen intinya adalah **Service Worker** yang bertindak sebagai network proxy di background. Dengan menerapkan strategi caching yang terukur (seperti *Cache-First* untuk bundel statis dan *Stale-While-Revalidate* untuk data semi-dinamis) serta memperhatikan siklus update UX yang aman, aplikasi web dapat memberikan kenyamanan dan keandalan yang setara dengan aplikasi native.
