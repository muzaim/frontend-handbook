# Browser Caching

## 1. Skenario Sistem Nyata

Seorang pengguna membuka aplikasi web portal berita setiap pagi sebelum berangkat kerja. Halaman memuat logo portal berita, berkas CSS tema global, font tipografi kustom, dan pustaka JavaScript dasar (total ukuran berkas sekitar **2,5 Megabyte**).

Server web perusahaan dikonfigurasi secara keliru tanpa menyertakan header cache HTTP sama sekali:

```http
# Response header yang buruk: Tidak ada instruksi caching
HTTP/1.1 200 OK
Content-Type: application/javascript
Content-Length: 1204850
# Tanpa Cache-Control atau ETag!
```

Dampak pada sistem:
- Setiap kali pengguna membuka halaman berita baru atau me-refresh halaman, browser terpaksa **mengunduh ulang seluruh 2,5 MB aset statis yang sama persis dari nol**.
- Pengguna menghabiskan waktu tunggu 2 detik di setiap perpindahan halaman, kuota internet pengguna terkuras sia-sia, dan server perusahaan menerima jutaan request unduhan aset statis yang sebenarnya tidak pernah berubah selama 6 bulan terakhir.

Solusi: Menguasai mekanisme **Browser Caching (Penyimpanan Cache Peramban)**: mengonfigurasi header HTTP `Cache-Control` yang presisi, memanfaatkan mekanisme validasi bersyarat **`ETag` (HTTP 304 Not Modified)**, serta memanfaatkan **Service Worker Cache API** untuk pengalaman instan bahkan saat offline.

## 2. Apa Itu dan Buat Apa?

Browser Caching adalah mekanisme penyimpanan lokal di dalam perangkat peramban web (pada memori RAM dan disk storage lokal komputer/ponsel) yang menyimpan salinan berkas sumber daya web (dokumen HTML, skrip JavaScript, stylesheet CSS, gambar, dan respons API) agar dapat **digunakan kembali pada kunjungan berikutnya tanpa perlu mengunduh ulang melalui jaringan internet**.

Tiga pilar mekanisme Browser Caching:
1. **Strong Caching (Cache-Control: max-age)**: Browser langsung membaca berkas dari disk lokal dalam waktu 0 milidetik tanpa mengirimkan request HTTP sama sekali ke server (*Memory Cache / Disk Cache*).
2. **Conditional Validation (ETag & Last-Modified)**: Browser mengirimkan request ringan untuk menanyakan: *"Apakah berkas ini sudah berubah sejak terakhir kali saya simpan?"*. Jika belum berubah, server membalas dengan status **HTTP 304 Not Modified** tanpa mengirimkan ulang isi berkas, menghemat 99% bandwidth!
3. **Programmatic Caching (Service Worker Cache API)**: Penyimpanan cache yang dikendalikan sepenuhnya melalui kode JavaScript untuk arsitektur Progressive Web Apps (PWA) dan kemampuan akses luring (*offline-first*).

Fungsi utama Browser Caching:
- Menghadirkan navigasi antarhalaman secepat kilat (*sub-second load time*) untuk pengunjung setia (*repeat visitors*).
- Memangkas volume bandwidth server web hingga lebih dari 80%.

## 3. Kenapa Harus Pakai Ini?

Koneksi internet seluler tidak pernah stabil. Permintaan jaringan tercepat adalah **permintaan yang tidak pernah dikirimkan ke jaringan (*the fastest request is a request not made*)**.

Browser Caching memungkinkan aplikasi Anda memanfaatkan ruang penyimpanan lokal di laptop atau ponsel pengguna untuk mengubah web aplikasi menjadi secepat aplikasi desktop terinstal.

## 4. Cara Kerjanya (Diagram Alur Keputusan Pohon Cache HTTP)

Alur peramban saat mengevaluasi cache berkas:

```
[ Browser Membutuhkan File: /assets/style.css ]
                         |
                         v
+---------------------------------------------------------------+
|                      CEK MEMORI CACHE LOKAL                   |
+---------------------------------------------------------------+
|  Apakah berkas ada di cache dan max-age masih berlaku?        |
|                                                               |
|  [YA] -------> AMBIL DARI DISK CACHE!                         |
|                Status: 200 OK (from disk cache)               |
|                *WAKTU: 0 MILIDETIK (NOL JARINGAN)!*           |
|                                                               |
|  [TIDAK] ----> Apakah ada header ETag (misal "v1.2")?         |
|                                                               |
|                [YA] ---> Kirim Request Bersyarat ke Server:   |
|                          Header: If-None-Match: "v1.2"        |
|                                     |                         |
|                    +----------------+----------------+        |
|                    |                                 |        |
|           (Belum Berubah)                     (Sudah Berubah) |
|           Server kirim:                       Server kirim:   |
|           304 Not Modified                    200 OK          |
|           (Hanya 100 byte!)                   (Kirim file baru|
|           Pakai cache lokal!                  & update cache) |
+---------------------------------------------------------------+
```

## 5. Cuplikan Logika Inti

### Perbedaan Krusial Direktif `Cache-Control` di Lingkungan Produksi

**1. Untuk Berkas Statis Ber-Hash Unik (JS, CSS, Font, Gambar Produk):**
```http
# Berkas ini memiliki nama unik seperti 'main.a9f1b2.js'.
# Jika kode berubah, nama filenya pasti berubah. Maka aman di-cache 1 tahun!
Cache-Control: public, max-age=31536000, immutable
```

**2. Untuk Berkas Titik Masuk HTML (`index.html`):**
```http
# BENCANA jika file HTML di-cache 1 tahun, karena pengguna tidak akan pernah
# menerima update nama file JS baru!
# 'no-cache' artinya: Simpan di disk, TAPI wajib tanyakan ke server (ETag check)
# sebelum menggunakannya!
Cache-Control: no-cache
```

**3. Untuk Data Sensitif Perbankan / Dokumen Privat:**
```http
# 'no-store' artinya: DILARANG menyimpan berkas ini di media penyimpanan apa pun!
# Browser dan proxy tidak boleh membuat salinan di disk.
Cache-Control: private, no-store
```

### Pemanfaatan Service Worker Cache API untuk Mode Offline

```javascript
// service-worker.js
const CACHE_NAME = "v1-core-assets";
const ASSETS_TO_CACHE = [
  "/",
  "/offline.html",
  "/styles/main.css",
  "/scripts/app.js",
];

// 1. Simpan aset inti saat Service Worker diinstal
self.addEventListener("install", (event) => {
  event.waitUntil(
    caches.open(CACHE_NAME).then((cache) => cache.addAll(ASSETS_TO_CACHE))
  );
});

// 2. Cegat request jaringan: Cek cache dulu, fallback ke jaringan, fallback ke offline.html
self.addEventListener("fetch", (event) => {
  event.respondWith(
    caches.match(event.request).then((cachedResponse) => {
      // Jika ada di cache, sajikan instan!
      if (cachedResponse) {
        return cachedResponse;
      }
      // Jika tidak ada di cache, ambil dari internet
      return fetch(event.request).catch(() => {
        // Jika internet putus, sajikan halaman ramah offline!
        return caches.match("/offline.html");
      });
    })
  );
});
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan
- **Kecepatan Akses Sub-Milidetik**: Pengunjung setia menikmati pemuatan halaman seketika.
- **Penghematan Beban Server Ekstrem**: Server terbebas dari jutaan request pembacaan aset statis berulang.
- **Ketahanan Luring (Offline First)**: Pengguna tetap dapat membuka dashboard dasar saat berada di dalam pesawat atau terowongan tanpa sinyal internet.

### Kekurangan & Jebakan Klasik
- **Jebakan Cache Abadi (*Stale Cache Lockout*)**: Jika Anda secara keliru memasang `Cache-Control: max-age=31536000` pada berkas `index.html`, pengguna tidak akan pernah bisa melihat pembaruan fitur baru Anda sampai mereka membersihkan riwayat browser secara manual!
- **Keterbatasan Ruang Disk Browser**: Browser memiliki kuota penyimpanan lokal. Jika disk perangkat pengguna penuh, peramban dapat membersihkan data Cache API secara otomatis (*cache eviction*).

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Menggunakan `max-age` Panjang + `immutable`
- Semua aset kompilasi yang memiliki hash acak pada nama filenya (`.chunk.js`, `.bundle.css`, web fonts `.woff2`).

### Kapan Menggunakan `no-cache` (Revalidate with ETag)
- Berkas `index.html` Single Page Application.
- Berkas manifes versi aplikasi (`version.json`).

### Kapan Menggunakan `no-store`
- Endpoint API mutasi keuangan, dokumen medis pasien, data otorisasi kartu kredit, dan respons API yang memuat data rahasia pribadi (*Personally Identifiable Information / PII*).

## 8. Metrik Kunci & Observabilitas

- **Status Code 304 Not Modified Ratio**: Pantau log server web. Persentase status 304 yang tinggi menunjukkan bahwa mekanisme validasi ETag bekerja sehat menghemat bandwidth.
- **Disk Cache Size di DevTools**: Buka tab *Application -> Storage* di Chrome DevTools untuk memeriksa ukuran memori Cache Storage yang digunakan oleh Service Worker.

## 9. Studi Kasus Industri Nyata

### Transformasi Pengalaman Navigasi di Wikipedia Web

- **Tantangan Rekayasa Riil**:
  Wikipedia memiliki miliaran pembaca per bulan. Sebagian besar pembaca berpindah dari satu artikel ensiklopedia ke artikel lainnya melalui tautan internal (*hyperlinks*).
- **Implementasi Solusi Industri**:
  Wikimedia Foundation mengimplementasikan arsitektur HTTP caching yang sangat disiplin:
  1. Seluruh aset gaya tampilan CSS dan skrip JavaScript diberi hash unik dan di-cache selama 1 tahun di browser pembaca.
  2. Artikel HTML diberi header `Cache-Control: public, s-maxage=86400, max-age=3600` yang dipadukan dengan header validasi `ETag`.
  3. Ketika pembaca menekan tombol "Back" atau "Forward", peramban menyajikan halaman dari **Back-Forward Cache (bfcache)** lokal secara seketika dalam waktu 0 milidetik.
- **Dampak & Hasil Sistem**:
  - 85% request aset di Wikipedia dilayani langsung dari cache browser lokal tanpa menyentuh server Wikimedia.
  - Mempertahankan operasional situs dengan efisiensi biaya server filantropis yang sangat terjangkau.

## 10. Fakta Menarik & Gotchas Production

1. **Arti Sebenarnya dari `no-cache`**:
   Banyak developer mengira `no-cache` berarti "jangan simpan di cache". **Ini 100% keliru!** `no-cache` artinya: *"Simpan berkas ini di cache, tetapi kamu WAJIB menanyakan ke server terlebih dahulu sebelum menggunakannya!"*. Jika Anda benar-benar tidak ingin berkas disimpan sama sekali, gunakan direktif **`no-store`**!
2. **Back-Forward Cache (bfcache)**:
   Browser modern memiliki cache khusus bernama `bfcache` yang membekukan seluruh snapshot memori halaman di RAM. Saat pengguna menekan tombol kembali (*Back*), browser tidak me-refresh HTML, melainkan mencairkan kembali snapshot memori tersebut secara instan. Menghindari event listener `beforeunload` yang tidak perlu memastikan halaman Anda dapat masuk ke dalam bfcache.

## 11. Rangkuman Singkat

1. **Permintaan Tercepat**: Akses ke disk cache lokal 0 milidetik adalah cara tercepat melayani halaman web.
2. **Kuasai Tiga Perintah**: Gunakan `max-age=31536000, immutable` untuk aset ber-hash, gunakan `no-cache` untuk `index.html`, dan gunakan `no-store` untuk data privat rahasia.
3. **Manfaatkan ETag**: Header `ETag` memungkinkan server mengembalikan status ringan `304 Not Modified` tanpa mengirim ulang payload data.
4. **Offline First dengan Service Worker**: Manfaatkan Cache API untuk menyediakan halaman fallback ramah saat jaringan internet terputus.
