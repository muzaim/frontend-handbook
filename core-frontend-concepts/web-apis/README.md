# Web APIs

## 1. Skenario Sistem Nyata

Pada aplikasi e-commerce katalog produk dengan 1.000 gambar produk di halaman utama, developer ingin menerapkan teknik *Lazy Loading* (memuat gambar hanya saat gambar tersebut mulai terlihat di layar pengguna saat digulir).

Developer junior membuat implementasi berbasis event scroll manual:

```javascript
// Kode bermasalah: Menghitung posisi seluruh gambar di setiap frame scroll
window.addEventListener("scroll", () => {
  const images = document.querySelectorAll("img[data-src]");
  images.forEach((img) => {
    const rect = img.getBoundingClientRect(); // Memicu Forced Synchronous Layout!
    if (rect.top < window.innerHeight && rect.bottom >= 0) {
      img.src = img.dataset.src;
      img.removeAttribute("data-src");
    }
  });
});
```

Dampak pada sistem:
- Memanggil `getBoundingClientRect()` pada ratusan elemen di setiap event scroll memicu puluhan kali **Layout Thrashing** per detik.
- Halaman tersendat parah (*scrolling stuttering*), baterai smartphone cepat panas dan terkuras drastis hanya untuk memeriksa apakah sebuah gambar sudah terlihat di layar.

Solusi: Menggunakan **Web APIs bawaan browser modern**, yaitu **`IntersectionObserver`**. Browser mengelola deteksi visibilitas secara asinkron di tingkat C++ mesin rendering (Compositor Thread) dengan konsumsi CPU dan baterai mendekati nol.

## 2. Apa Itu dan Buat Apa?

Web APIs (Web Application Programming Interfaces) adalah sekumpulan antarmuka, objek, dan metode bawaan yang disediakan oleh lingkungan peramban (browser runtime) untuk dieksekusi melalui kode JavaScript.

Banyak developer pemula mengira bahwa fitur-fitur seperti `fetch`, `setTimeout`, `localStorage`, kanvas grafis, atau kamera ponsel adalah bagian dari sintaks bahasa pemrograman JavaScript (ECMAScript). **Faktanya bukan!** ECMAScript murni hanya mendefinisikan tipe data, perulangan, objek, dan logika dasar. Segala interaksi dengan perangkat keras, jaringan internet, audio, penyimpanan lokal, dan jendela peramban disediakan oleh **Web APIs standar W3C / WHATWG**.

Contoh Web APIs esensial modern:
- **Jaringan & Komunikasi**: `Fetch API`, `WebSocket API`, `Server-Sent Events (SSE)`.
- **Pengamat Performa & DOM**: `IntersectionObserver`, `ResizeObserver`, `MutationObserver`, `PerformanceObserver`.
- **Penyimpanan Lokal**: `IndexedDB`, `Cache API`, `Web Storage (localStorage / sessionStorage)`.
- **Perangkat Keras & Sensor**: `Geolocation API`, `Web Bluetooth`, `Web Audio API`, `Camera/Microphone (MediaDevices)`.

## 3. Kenapa Harus Pakai Ini?

Menggunakan Web APIs modern yang tepat memberikan keuntungan besar:
1. **Performa Native Asinkron**: API modern seperti Observer berjalan di thread internal browser di luar main-thread JavaScript, mengeliminasi layout thrashing.
2. **Mengurangi Ukuran Bundle (Zero-Dependency)**: Alih-alih mengunduh library NPM berukuran puluhan kilobyte untuk fungsi sederhana (seperti library deteksi scroll atau polyfill penyimpanan), kita dapat memanfaatkan fitur yang sudah tertanam di mesin browser pengguna.
3. **Akses Fitur Perangkat Keras (*Progressive Web Apps*)**: Memungkinkan aplikasi web memberikan pengalaman layaknya aplikasi native ponsel (akses GPS, notifikasi push, mode offline).

## 4. Cara Kerjanya (Diagram Alur Ekosistem Browser vs JavaScript)

```
+---------------------------------------------------------------+
|                       MESIN PERAMBAN                          |
|                                                               |
|  +-----------------------+       +-------------------------+  |
|  |     JavaScript        |       |         WEB APIs        |  |
|  |   Runtime (V8)        |       |   (Disediakan Browser)  |  |
|  |                       |       |                         |  |
|  | - Call Stack          |       | - DOM & Window          |  |
|  | - Memory Heap         | <===> | - Fetch / XHR           |  |
|  | - ECMAScript core     |       | - Timers (setTimeout)   |  |
|  |   (Array, Object, fn) |       | - Observers (Intersect) |  |
|  +-----------------------+       | - IndexedDB / Storage   |  |
|                                  | - Hardware (GPS, Audio) |  |
|                                  +-------------------------+  |
|                                              |                |
|                                              v                |
|  +---------------------------------------------------------+  |
|  |                     PERANGKAT KERAS                     |  |
|  |      (Network Card, Storage SSD, GPS, Layar Monitor)    |  |
|  +---------------------------------------------------------+  |
+---------------------------------------------------------------+
```

## 5. Cuplikan Logika Inti

### Deteksi Visibilitas Elemen Modern Menggunakan `IntersectionObserver`

Menggantikan ribuan baris kode event scroll lama dengan 15 baris kode native yang sangat efisien:

```javascript
// 1. Inisialisasi IntersectionObserver
const lazyImageObserver = new IntersectionObserver(
  (entries, observer) => {
    entries.forEach((entry) => {
      // Periksa apakah elemen target sudah masuk ke dalam viewport
      if (entry.isIntersecting) {
        const image = entry.target;
        // Ganti src gambar dengan url asli dari data-src
        image.src = image.dataset.src;
        image.classList.add("fade-in-loaded");

        // Hentikan pengamatan pada elemen ini karena sudah selesai dimuat
        observer.unobserve(image);
      }
    });
  },
  {
    root: null, // Mengamati relatif terhadap viewport layar
    rootMargin: "200px 0px", // Mulai muat 200px SEBELUM gambar menyentuh layar
    threshold: 0.01 // Terpicu saat 1% bagian gambar masuk
  }
);

// 2. Terapkan pengamatan ke seluruh gambar lazy
document.querySelectorAll("img.lazy-load").forEach((img) => {
  lazyImageObserver.observe(img);
});
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan
- **Nol Ukuran Bundle**: Fitur sudah tersedia langsung di dalam binary browser (0 KB biaya unduhan).
- **Performa Teroptimasi Mesin**: Diimplementasikan menggunakan bahasa C++ / Rust berkecepatan tinggi di dalam inti peramban.
- **Konsistensi Standar Terbuka**: Diatur oleh konsorsium web internasional (W3C dan WHATWG).

### Kekurangan & Tantangan
- **Fragmentasi Dukungan Browser (*Compatibility Matrix*)**: Fitur Web API mutakhir (seperti WebGPU atau File System Access API) mungkin belum didukung oleh semua browser (terutama Safari iOS vs Chromium).
- **Keamanan & Izin Pengguna (*Permission Prompt*)**: Banyak Web API canggih (Geolocation, Notifikasi, Kamera) mewajibkan koneksi HTTPS aman dan persetujuan eksplisit dari pengguna. Jika pengguna menekan "Block", aplikasi harus memiliki strategi fallback yang anggun (*graceful degradation*).

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Memilih Web APIs Bawaan
- Deteksi elemen masuk/keluar layar: Selalu gunakan `IntersectionObserver`.
- Deteksi perubahan ukuran dimensi kontainer: Gunakan `ResizeObserver` (jauh lebih baik daripada event `window.onresize`).
- Deteksi mutasi struktur DOM: Gunakan `MutationObserver`.
- Penyimpanan data terstruktur berkapasitas besar di browser (> 5 MB): Gunakan `IndexedDB` alih-alih `localStorage` yang memblokir main-thread.

### Kapan Membutuhkan Pustaka Abstraksi
- Menggunakan `IndexedDB` mentah sangat berbelit-belit (*verbose*). Dianjurkan membungkusnya dengan library pembungkus tipis berbasis Promise (seperti `idb` atau `Dexie.js`).

## 8. Metrik Kunci & Observabilitas

- **Feature Detection Verification**: Selalu verifikasi ketersediaan API sebelum memanggilnya di production:
  ```javascript
  if ("IntersectionObserver" in window) {
    // Gunakan Web API native
  } else {
    // Muat polyfill fallback secara dinamis
  }
  ```
- **Console Warnings (Deprecations)**: Perhatikan tab Console di DevTools. Browser akan memberikan peringatan kuning jika Anda menggunakan Web API yang sudah usang (*deprecated*, seperti `document.write` atau sinkron `XMLHttpRequest`).

## 9. Studi Kasus Industri Nyata

### Migrasi Pinterest ke Web APIs Modern

- **Tantangan Rekayasa Riil**:
  Halaman feed Pinterest terdiri dari ribuan pin gambar dengan tata letak susunan bata (*masonry layout*). Dulunya, Pinterest menggunakan library JavaScript pihak ketiga yang tebal untuk mendeteksi scroll dan menghitung ukuran kolom kanvas. Hal ini menyebabkan waktu muat lambat dan memakan memori besar pada ponsel pengguna di negara berkembang.
- **Implementasi Solusi Industri**:
  Tim teknik Pinterest menghapus library pihak ketiga tersebut dan beralih menggunakan kombinasi Web APIs native:
  1. **`IntersectionObserver`** untuk lazy loading pin gambar.
  2. **`ResizeObserver`** untuk menghitung ulang tata letak kolom saat orientasi layar berganti.
  3. **`Network Information API`** (`navigator.connection.saveData`) untuk secara otomatis menyajikan gambar beresolusi lebih rendah jika pengguna sedang berada di jaringan seluler lambat atau mengaktifkan mode hemat data.
- **Dampak & Hasil Sistem**:
  - Ukuran bundle JavaScript tereduksi sebesar 23%.
  - Time to Interactive (TTI) di jaringan 3G membaik hingga 40%.

## 10. Fakta Menarik & Gotchas Production

1. **`localStorage` adalah Operasi Sinkron Pemblokir Main-Thread**:
   Banyak developer tidak menyadari bahwa `localStorage.getItem()` dan `localStorage.setItem()` berjalan secara **sinkron**. Jika Anda menyimpan JSON besar (misal 4 MB) ke `localStorage`, main-thread akan macet selama puluhan milidetik untuk operasi disk I/O. Untuk data besar, selalu gunakan `IndexedDB` yang bersifat asinkron sepenuhnya!
2. **Wajib Konteks Aman (*Secure Context / HTTPS*)**:
   Hampir seluruh Web APIs modern yang berkaitan dengan privasi dan keamanan (Service Worker, Geolocation, Clipboard write, Web Crypto, Web Share) hanya akan aktif jika website disajikan melalui protokol **HTTPS** (atau `localhost` untuk pengembangan lokal).

## 11. Rangkuman Singkat

1. **Bukan Bagian dari JS Core**: Web APIs adalah antarmuka bawaan yang disediakan oleh lingkungan browser, bukan sintaks resmi JavaScript.
2. **Prioritaskan Observers Modern**: Gantikan listener scroll dan resize manual dengan `IntersectionObserver` dan `ResizeObserver`.
3. **Waspadai localStorage**: Hindari menyimpan data besar di `localStorage` karena bersifat sinkron dan memblokir main-thread; gunakan `IndexedDB`.
4. **Periksa Kompatibilitas**: Selalu terapkan *feature detection* untuk memastikan API didukung oleh browser pengguna sebelum dieksekusi.
