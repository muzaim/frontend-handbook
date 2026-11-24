# Code Splitting

## 1. Skenario Sistem Nyata

Sebuah platform SaaS akuntansi dan penggajian karyawan (*payroll*) memiliki aplikasi frontend monolitik. Di dalam aplikasi tersebut terdapat:
- Halaman Login sederhana.
- Dashboard ringkasan keuangan mingguan.
- Modul pembuat laporan pajak yang memuat pustaka grafik kompleks (Highcharts / D3.js seberat 600 KB).
- Modul ekspor PDF formulir SPT pajak yang menyertakan pustaka parser PDF dan canvas font (pdfmake / jsPDF seberat 850 KB).

Sebelum optimasi dilakukan, bundler (Webpack / Vite) menggabungkan seluruh kode aplikasi ke dalam satu file tunggal: `bundle.js` sebesar **4,8 Megabyte**.

Dampak pada sistem:
- Setiap kali seorang karyawan biasa hanya ingin membuka halaman Login untuk melakukan absensi pagi, peramban ponselnya terpaksa mengunduh seluruh 4,8 MB file JavaScript tersebut!
- Proses kompilasi dan eksekusi skrip di CPU ponsel memakan waktu 4 detik.
- Karyawan mengeluh aplikasi absensi perusahaan sangat lambat, padahal mereka tidak pernah menggunakan modul laporan pajak atau ekspor PDF sama sekali.

Solusi: Menerapkan **Code Splitting**. Memecah satu file raksasa tersebut menjadi puluhan potongan file kecil (*chunks*). Halaman login hanya mengunduh 80 KB kode yang dibutuhkannya. Modul pajak dan PDF hanya akan diunduh jika dan hanya jika pengguna menavigasi ke menu tersebut.

## 2. Apa Itu dan Buat Apa?

Code Splitting adalah teknik dalam proses bundling aplikasi frontend modern di mana satu file bundle JavaScript raksasa dipecah menjadi beberapa berkas bundle yang lebih kecil (*chunks*), yang kemudian dapat dimuat secara paralel atau dimuat secara dinamis saat dibutuhkan (*on-demand*).

Teknik ini didukung secara bawaan oleh bundler modern seperti Vite (Rollup), Webpack, Turbopack, dan esbuild.

Tiga strategi utama Code Splitting:
1. **Entry Points Splitting**: Memisahkan bundle berdasarkan halaman entri yang berbeda (misal bundle halaman publik vs bundle panel admin).
2. **Route-Based Splitting**: Memecah chunk berdasarkan rute URL aplikasi (setiap rute halaman memiliki file chunk-nya sendiri).
3. **Component / Vendor Splitting**: Memisahkan pustaka pihak ketiga (*node_modules*) yang jarang berubah dari kode aplikasi bisnis yang sering berubah.

## 3. Kenapa Harus Pakai Ini?

Seiring berkembangnya fitur aplikasi, ukuran kode JavaScript akan terus bertambah (*code bloat*). Tanpa Code Splitting, pertumbuhan kode berbanding lurus dengan penurunan performa aplikasi: semakin banyak fitur baru yang ditambahkan, semakin lambat halaman login awal dimuat.

Code Splitting memastikan bahwa waktu muat awal aplikasi (*initial load time*) tetap stabil dan cepat terlepas dari seberapa masif skala keseluruhan aplikasi Anda.

## 4. Cara Kerjanya (Diagram Alur Bundling Monolitik vs Chunks)

```
SEBELUM CODE SPLITTING (Monolitik):
[Kode Login] + [Dashboard] + [Modul Pajak 600KB] + [Library PDF 850KB]
                                 |
                                 v
                     [ main.bundle.js (4.8 MB) ]
            *PENGGUNA WAJIB MENGUNDUH SEMUANYA SEJAK AWAL!*

----------------------------------------------------------------------

SESUDAH CODE SPLITTING (Route & Vendor Chunks):
              [ Aplikasi Monolitik ]
                        |
        +---------------+---------------+---------------+
        |               |               |               |
        v               v               v               v
[vendor.js]     [login.chunk.js] [dashboard.js]  [tax-report.js]
  (React, lodash)   (Hanya 80 KB)   (Hanya 150 KB)   (Pustaka Grafis 600KB)
  (Cache Abadi)     *Diunduh Awal*  *Diunduh Nanti*  *Diunduh Nanti*
```

## 5. Cuplikan Logika Inti

### Route-Based Code Splitting di React Modern

Menggabungkan `React.lazy` dan `<Suspense>` dengan React Router:

```javascript
import React, { Suspense, lazy } from "react";
import { BrowserRouter, Routes, Route, Link } from "react-router-dom";

// 1. Modul halaman diimpor secara malas (Lazy / Code-Split)
// Bundler akan memisahkan kedua halaman ini ke dalam file chunk terpisah:
// -> Dashboard.chunk.js dan TaxReport.chunk.js
const Dashboard = lazy(() => import("./pages/Dashboard"));
const TaxReport = lazy(() => import("./pages/TaxReport"));

function App() {
  return (
    <BrowserRouter>
      <nav>
        <Link to="/">Dashboard</Link> | <Link to="/tax">Laporan Pajak</Link>
      </nav>

      {/* Suspense menampilkan fallback UI sementara file chunk sedang diunduh via jaringan */}
      <Suspense fallback={<div className="loading-spinner">Memuat modul...</div>}>
        <Routes>
          <Route path="/" element={<Dashboard />} />
          <Route path="/tax" element={<TaxReport />} />
        </Routes>
      </Suspense>
    </BrowserRouter>
  );
}

export default App;
```

### Konfigurasi Vendor Chunk Splitting di Vite (`vite.config.js`)

Memisahkan library pihak ketiga agar memiliki masa simpan cache peramban yang panjang:

```javascript
// vite.config.js
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";

export default defineConfig({
  plugins: [react()],
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          // Seluruh dependency React dipisahkan ke vendor chunk terpisah
          "vendor-react": ["react", "react-dom", "react-router-dom"],
          // Library analitik grafik berat dipisahkan ke chunk mandiri
          "vendor-charts": ["chart.js", "d3"],
        },
      },
    },
  },
});
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan
- **Initial Load Time Terjaga Cepat**: Pengguna baru hanya mengunduh kode yang benar-benar mereka buka saat itu.
- **Efisiensi Caching HTTP**: Ketika Anda mengubah satu baris logika di halaman login, pengguna tidak perlu mengunduh ulang vendor chunk (React/library) yang tidak berubah.
- **Pemanfaatan Paralelisasi HTTP/2**: Mengunduh 4 file chunk kecil secara paralel melalui satu koneksi multiplexing HTTP/2 sering kali lebih efisien daripada 1 file raksasa.

### Kekurangan & Tantangan
- **Jeda Navigasi Antarhalaman (*Navigation Latency*)**: Saat pengguna mengklik menu rute baru, ada jeda beberapa ratus milidetik untuk mengunduh chunk halaman tersebut melalui jaringan. (Solusi: Terapkan *Prefetching* saat link di-hover).
- **Potensi Chunk Load Error**: Jika deployment baru dirilis ke server dan hash nama file chunk lama dihapus dari CDN, pengguna yang sedang membuka aplikasi lama akan mengalami error saat berpindah halaman: *"Loading chunk failed"*.

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Pakai
- Semua aplikasi Single Page Application (SPA) yang memiliki lebih dari 3 rute navigasi.
- Komponen atau fitur yang memuat library pihak ketiga dengan ukuran > 50 KB (Rich Text Editor, PDF generator, 3D Canvas visualizer, Chart builder).
- Area aplikasi yang terpisah jelas berdasarkan hak akses peran pengguna (misal area `/admin` vs area `/user`).

### Kapan Jangan Terlalu Agresif (*Over-Splitting*)
- Jangan memecah komponen-komponen kecil yang ukurannya hanya 2-5 KB menjadi chunk terpisah. Memecah aplikasi menjadi ratusan file mikro justru akan menimbulkan overhead *HTTP request round-trip* dan kompresi Gzip/Brotli yang kurang efisien.

## 8. Metrik Kunci & Observabilitas

- **Initial JS Bundle Size**: Target di dashboard CI/CD: bundle rute awal **< 150 KB (gzipped)**.
- **Total Blocking Time (TBT)**: Memastikan proses parsing JavaScript di awal halaman tidak melebihi 200 milidetik.
- **Visualisasi Bundle Analyzer**: Gunakan plugin analyzer (seperti `rollup-plugin-visualizer` atau `webpack-bundle-analyzer`) untuk melihat peta pohon (*treemap*) ukuran modul secara visual sebelum rilis ke production.

## 9. Studi Kasus Industri Nyata

### Optimasi Navigasi dan Pemuatan Awal pada Spotify Web Player

- **Tantangan Rekayasa Riil**:
  Aplikasi web pemutar musik Spotify memiliki ratusan fitur: lirik lagu sinkron waktu, antrean lagu, profil artis, pengaturan akun, hingga visualisasi audio canvas. Bundel awal yang terlalu besar membuat waktu pembukaan pemutar musik lambat di laptop lawas.
- **Implementasi Solusi Industri**:
  Spotify menerapkan **Predictive Route-Based Code Splitting**:
  1. Halaman utama hanya memuat mesin pemutar audio inti dan layout navbar.
  2. Tampilan lirik lagu (`LyricsModal`) dipisahkan ke chunk dinamis tersendiri yang baru dimuat saat pengguna menekan tombol mikrofon lirik.
  3. Modul profil artis di-prefetch secara otomatis di latar belakang saat kursor mouse pengguna mengambang (*hover*) di atas foto atau nama artis.
- **Dampak & Hasil Sistem**:
  - Ukuran initial payload berkurang hingga 60%.
  - Transisi menuju halaman lirik dan profil terasa instan tanpa terasa adanya jeda unduhan.

## 10. Fakta Menarik & Gotchas Production

1. **Jebakan "Failed to fetch dynamically imported module"**:
   Saat Anda melakukan deployment versi baru ke production, bundler akan menghasilkan nama hash baru (misal `dashboard.c8d2a.js`). Jika ada pengguna lama yang masih membuka tab browser dan mengklik menu dashboard, browser mencoba mengunduh nama file versi lama (`dashboard.a1b2c.js`) yang sudah dihapus dari server!  
   *Solusi*: Bungkus komponen `<Suspense>` dengan **Error Boundary** yang otomatis memanggil `window.location.reload()` secara cerdas jika terjadi error chunk loading.
2. **Gzip Lebih Efektif pada File Lebih Besar**:
   Menggabungkan 50 file kecil menjadi 1 file sering kali menghasilkan ukuran file terkompresi Gzip/Brotli yang sedikit lebih kecil daripada total 50 file individual. Inilah alasan mengapa *vendor chunking* yang seimbang lebih baik daripada memecah setiap modul individu.

## 11. Rangkuman Singkat

1. **Bagi dan Taklukkan**: Code splitting memecah satu bundle JS raksasa menjadi potongan-potongan chunk kecil.
2. **Pemuatan Berdasarkan Kebutuhan**: Hanya unduh kode yang dibutuhkan oleh halaman yang sedang aktif dibuka pengguna.
3. **Optimasi Route-Based**: Terapkan pemecahan kode pada level rute halaman menggunakan `React.lazy` atau dynamic imports.
4. **Waspadai Cache Miss Deployment**: Siapkan penanganan error boundary saat file chunk lama kedaluwarsa setelah proses rilis baru.
