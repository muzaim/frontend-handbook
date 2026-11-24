# Bundle Optimization

## 1. Skenario Sistem Nyata

Pada sebuah aplikasi dashboard SaaS B2B, proses deployment menghasilkan satu berkas output JavaScript: `index-d8e2a1.js` dengan ukuran **5,4 Megabyte**.

Developer mengeluh aplikasi lambat, namun bingung dari mana asal muasal membengkaknya ukuran file tersebut. Ketika tim melakukan inspeksi menggunakan alat visualizer bundle (*bundle analyzer*):
- Ditemukan **3 versi React yang berbeda** terpasang secara bersamaan karena ada pustaka UI pihak ketiga yang belum di-update dependensinya.
- Ditemukan seluruh file ikon set (berisi 2.500 file SVG) ikut terkompilasi ke dalam bundle.
- Berkas disajikan melalui web server Nginx tanpa mengaktifkan kompresi modern **Brotli**, melainkan disajikan mentah (*uncompressed plain text*).

Dampak pada sistem:
- Pengguna dengan koneksi internet kantor biasa membutuhkan waktu 4 detik hanya untuk mengunduh berkas JavaScript tersebut.
- Ponsel karyawan mengalami panas tinggi saat mem-parsing kode 5,4 MB tersebut di main-thread.

Solusi: Menerapkan **Bundle Optimization (Optimasi Bundel Menyeluruh)**: mengaktifkan kompresi modern **Brotli**, melakukan minifikasi dan mangling kode via esbuild/Terser, mengeliminasi paket dependensi ganda (*duplicate packages*), serta menerapkan batasan ukuran bundel (*Performance Budget*) di CI/CD.

## 2. Apa Itu dan Buat Apa?

Bundle Optimization adalah serangkaian teknik dan konfigurasi rekayasa pada tahap kompilasi (*build pipeline*) dan penyajian server (*server delivery*) untuk **meminimalkan ukuran byte berkas JavaScript dan CSS yang dikirimkan melalui jaringan internet ke peramban pengguna**.

Pilar-pilar utama optimasi bundel:
1. **Minifikasi & Mangling**: Menghapus seluruh spasi, enter, baris komentar, serta memendekkan nama variabel panjang (misal `userAccountBalance` disingkat menjadi variabel satu huruf `a`) menggunakan mesin minifier super cepat seperti esbuild atau Terser.
2. **Kompresi Jaringan (Brotli & Gzip)**: Mengompresi berkas teks biner saat transmisi HTTP. Brotli (algoritma kompresi dari Google) menghasilkan ukuran berkas 15-25% lebih kecil dibandingkan Gzip standar.
3. **Pembersihan Dependensi Ganda**: Memastikan hanya ada satu versi tunggal dari pustaka fondasi (React, Vue, lodash) di dalam grafik dependensi proyek.
4. **Performance Budgeting**: Memasang batas ambang maksimal ukuran file di pipeline build CI/CD agar developer tidak dapat melakukan *merge* jika kode baru membuat ukuran bundle melonjak melebihi batas.

## 3. Kenapa Harus Pakai Ini?

Biaya pengiriman JavaScript adalah **biaya paling mahal di peramban web**.

1 Megabyte gambar JPEG yang diunduh hanya perlu didekode oleh GPU. Sebaliknya, **1 Megabyte JavaScript** harus:
1. Diunduh melalui jaringan (*Download*).
2. Diparsing menjadi pohon sintaksis oleh parser browser (*Parse*).
3. Dikompilasi menjadi bytecode mesin (*Compile*).
4. Dieksekusi di CPU main-thread (*Execute*).

Setiap kilobyte JavaScript yang berhasil Anda pangkas secara langsung mempercepat waktu **Total Blocking Time (TBT)** dan **Interaction to Next Paint (INP)** aplikasi Anda.

## 4. Cara Kerjanya (Diagram Alur Komparasi Ukuran Bundle Pipeline)

```
KODE SUMBER AWAL (Mentah sebelum build)
[ Kode Aplikasi ] + [ node_modules ] = 8.5 MB
                 |
                 v
FASE 1: TREE SHAKING & DEAD CODE ELIMINATION
Membuang fungsi yang tidak dipanggil = Sisa 5.2 MB
                 |
                 v
FASE 2: MINIFIKASI & MANGLING (esbuild / Terser)
Menghapus whitespace, komentar, perpendek nama variabel = Sisa 1.8 MB
                 |
                 v
FASE 3: KOMPRESI TRANSMISI (Saat dikirim via HTTP ke Browser)
+---------------------------------------------------------------+
|  Tanpa Kompresi:  1.8 MB  (Waktu unduh: 2.50 detik)           |
|  Kompresi Gzip:    520 KB  (Waktu unduh: 0.80 detik)           |
|  Kompresi Brotli:  390 KB  (Waktu unduh: 0.45 detik) - TERBAIK |
+---------------------------------------------------------------+
```

## 5. Cuplikan Logika Inti

### Konfigurasi Optimasi Bundel di Vite (`vite.config.js`)

Menggabungkan minifikasi modern, visualisasi bundle, dan kompresi Brotli otomatis:

```javascript
// vite.config.js
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";
import compression from "vite-plugin-compression2";
import { visualizer } from "rollup-plugin-visualizer";

export default defineConfig({
  plugins: [
    react(),
    // 1. Otomatis menghasilkan berkas .br (Brotli) saat npm run build
    compression({
      algorithm: "brotliCompress",
      exclude: [/\.(br)$/, /\.(gz)$/],
      threshold: 1024, // Hanya kompres file > 1 KB
    }),
    // 2. Menghasilkan laporan peta ukuran bundle visual (stats.html)
    visualizer({
      filename: "dist/stats.html",
      open: false,
      gzipSize: true,
      brotliSize: true,
    }),
  ],
  build: {
    target: "es2020", // Menghindari polyfill usang yang membuat bundle gemuk
    minify: "esbuild", // Minifikasi super cepat berbasis Go
    chunkSizeWarningLimit: 500, // Batas peringatan 500 KB per chunk
  },
});
```

### Konfigurasi Nginx untuk Menyajikan Pre-Compressed Brotli

```nginx
# nginx.conf
server {
    listen 443 ssl http2;
    server_name app.example.com;

    # Mengaktifkan penyajian berkas .br statis yang sudah dikompresi saat build time
    brotli_static on;
    brotli on;
    brotli_comp_level 6;
    brotli_types text/plain text/css application/javascript application/json image/svg+xml;

    location / {
        root /usr/share/nginx/html;
        try_files $uri $uri/ /index.html;
    }
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan
- **Waktu Muat Awal Instan**: File terkompresi Brotli berukuran kecil tiba di browser dalam hitungan puluhan milidetik.
- **Penghematan Biaya Cloud**: Mengurangi volume data keluar (*egress bandwidth*) hingga lebih dari 60%.
- **CPU Mobile Dingin**: Minifikasi mengurangi jumlah token kode yang harus diparsing oleh mesin V8 browser.

### Kekurangan & Biaya
- **Waktu Kompresi CPU Server**: Mengompresi file secara dinamis menggunakan Brotli level 11 saat runtime dapat membebani CPU web server. (Solusi: Lakukan *Pre-compression* berkas `.br` saat build time di CI/CD).
- **Sulit Membaca Source Code Produksi**: Variabel yang dimangling menjadi satu huruf (`function a(b, c)`) membuat debugging di production mustahil tanpa berkas **Source Maps (`.map`)**.

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Wajib Mengoptimasi Bundel
- Seluruh aplikasi web produksi komersial yang diakses publik.
- Aplikasi dengan target pengguna perangkat seluler dan negara berkembang dengan koneksi terbatas.

### Kapan Boleh Diabaikan
- Lingkungan pengembangan lokal (*Development mode / localhost*). Selama coding, minifikasi dan kompresi sengaja dimatikan agar Hot Module Replacement (HMR) berjalan secepat kilat.

## 8. Metrik Kunci & Observabilitas

- **Total JavaScript Transferred Size**: Pantau di tab Network DevTools. Standar emas industri: total JS awal **< 200 KB (Brotli compressed)**.
- **CI/CD Size Limit Check**: Pasang alat seperti `size-limit` di GitHub Actions. Jika Pull Request baru menambah ukuran bundel sebesar > 10 KB, gagalkan build otomatis!
- **Source Maps Hygiene**: Pastikan file `.map` tidak diunggah ke server publik agar kode sumber asli tidak dicuri kompetitor.

## 9. Studi Kasus Industri Nyata

### Reduksi Bundle Size Ekstrem pada Tinder Web

- **Tantangan Rekayasa Riil**:
  Saat pertama kali meluncurkan Tinder Online PWA, aplikasi memiliki ukuran bundle awal sebesar 2,8 Megabyte. Pengguna di jaringan seluler lambat sering mengalami kegagalan saat membuka halaman swipe kartu jodoh.
- **Implementasi Solusi Industri**:
  Tim Tinder melakukan perombakan optimasi bundel secara agresif:
  1. Mengganti polyfill lama dan menghapus pustaka dependensi duplikat.
  2. Menerapkan Route-based Code Splitting dan Vendor Chunking.
  3. Mengaktifkan kompresi Brotli level tinggi di edge CDN Cloudflare.
- **Dampak & Hasil Sistem**:
  - Ukuran bundle JavaScript terkompresi berhasil dipangkas dari **2.8 MB menjadi hanya 290 KB** (berkurang hampir 90%).
  - Waktu muat awal (*Time to Interactive*) membaik dari 11,9 detik menjadi 4,6 detik di jaringan 4G lambat.

## 10. Fakta Menarik & Gotchas Production

1. **Brotli Membutuhkan HTTPS**:
   Semua peramban modern (Chrome, Safari, Firefox) hanya akan mengirimkan header `Accept-Encoding: br` jika koneksi web disajikan melalui protokol **HTTPS** yang aman. Jika website Anda masih menggunakan HTTP biasa, browser hanya akan menerima kompresi Gzip lama.
2. **Gunakan `npm dedupe` atau `pnpm`**:
   Sering kali dua library yang berbeda membutuhkan library ketiga yang sama tetapi beda patch version (misal library A butuh `debug@4.3.1` dan library B butuh `debug@4.3.4`). Menjalankan perintah `npm dedupe` atau beralih ke package manager modern **pnpm** menyatukan versi-versi tersebut ke satu versi tunggal, menghemat ratusan kilobyte duplikasi kode.

## 11. Rangkuman Singkat

1. **JavaScript Sangat Mahal**: Setiap kilobyte JS membebani jaringan, parser CPU, dan memori RAM peramban.
2. **Gunakan Brotli**: Kompresi Brotli menghemat 15-25% ukuran transfer data dibandingkan Gzip standar.
3. **Audit dengan Visualizer**: Selalu gunakan plugin bundle analyzer untuk mendeteksi paket dependensi raksasa atau duplikat.
4. **Pasang Performance Budget**: Tetapkan batas maksimal ukuran bundle di pipeline CI/CD untuk mencegah regresi kode di masa depan.
