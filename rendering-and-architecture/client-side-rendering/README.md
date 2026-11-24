# Client-Side Rendering (CSR)

## 1. Skenario Sistem Nyata

Sebuah startup teknologi membangun portal e-commerce publik menggunakan Single Page Application (SPA) berbasis Create React App (CSR murni). Saat aplikasi dirilis, tim pemasaran meluncurkan kampanye iklan Google Ads dan optimasi SEO untuk kata kunci "sepatu lari pria".

Setelah satu bulan berjalan, ditemukan dua masalah besar:
1. **Peringkat SEO Terpuruk**: Bot mesin pencari (web crawler) yang mengindeks website hanya menerima berkas HTML kosong: `<div id="root"></div>`. Meskipun Googlebot modern mengeksekusi JavaScript, bot memiliki jatah waktu komputasi (*crawl budget*). Karena bundle JS aplikasi berukuran 3,5 MB dan butuh waktu 6 detik untuk render, sebagian besar halaman produk tidak sempat diindeks dan dianggap halaman kosong (*empty page*).
2. **Pengguna Ponsel Membatalkan Kunjungan (*High Bounce Rate*)**: Pengguna di jaringan 4G/3G dengan smartphone kelas menengah melihat layar putih kosong (*blank white screen*) selama 4 hingga 7 detik sebelum halaman muncul. Lebih dari 55% calon pembeli menutup tab sebelum halaman sempat selesai dimuat.

Solusi: Memahami batasan arsitektur CSR murni, dan mengetahui kapan harus beralih ke Server-Side Rendering (SSR) atau Static Site Generation (SSG) untuk halaman publik yang membutuhkan SEO dan First Contentful Paint cepat.

## 2. Apa Itu dan Buat Apa?

Client-Side Rendering (CSR) adalah pola arsitektur rendering web di mana server hanya mengirimkan sebuah dokumen HTML minimalis (biasanya hanya satu tag kontainer `<div id="root"></div>`) beserta skrip JavaScript yang dibundel (*bundle JS*).

Seluruh proses rendering—mulai dari pembuatan elemen DOM, routing navigasi antarhalaman, pengambilan data dari REST/GraphQL API, hingga penataan layout visual—dilakukan secara lokal oleh peramban pengguna (**di sisi Client**) menggunakan JavaScript.

Fungsi utama CSR:
- **Pengalaman Interaksi Mirip Aplikasi Native**: Navigasi antarhalaman terjadi secara instan tanpa perlu memuat ulang seluruh halaman dari server (*no full-page reload*).
- **Efisiensi Biaya Server**: Server web hanya bertindak sebagai penyaji berkas statis (HTML, JS, CSS melalui CDN) dan penyedia API data mentah (JSON). Komputasi render dibebankan ke perangkat pengguna.

## 3. Kenapa Harus Pakai Ini?

Untuk aplikasi web berbasis dashboard internal, portal manajemen data (*back-office / SaaS portal*), atau jejaring sosial yang memerlukan autentikasi login pengguna:
1. SEO tidak diperlukan karena konten berada di balik gerbang login (*authentication wall*).
2. Transisi antarmuka yang sangat dinamis, animasi transisi halaman yang kaya, dan caching state client lokal (seperti Redux/Zustand) bekerja paling alami dalam model CSR.
3. Arsitektur tim terpisah sangat jelas: tim backend fokus membangun REST/gRPC API murni, sementara tim frontend fokus membangun UI client.

## 4. Cara Kerjanya (Diagram Alur Beban CSR)

```
Pengguna Membuka URL (Contoh: https://app.example.com)
                         |
                         v
[Browser] ---------- GET / ------------> [Server / CDN]
[Browser] <--- 1. HTML Kosong (<div id="root"></div>) --- [Server]
     |
     | (Layar masih PUTIH KOSONG / Blank Screen)
     v
[Browser] ------- GET /bundle.js ------> [Server / CDN]
[Browser] <--- 2. Berkas JS Raksasa (3 MB) -------------- [Server]
     |
     | (Browser mem-parsing & mengeksekusi JS di Main Thread)
     v
[Browser] mengeksekusi: React.createElement(...)
     |
     v
[Browser] ------- GET /api/v1/user-data -> [API Backend]
[Browser] <--- 3. Response JSON Data ---- [API Backend]
     |
     v
[Browser] Merender DOM Lengkap & Menampilkan UI ke Pengguna
     |
     v
*PENGGUNA BARU BISA MELIHAT & BERINTERAKSI PADA DETIK KE-4!*
```

## 5. Cuplikan Logika Inti

### Struktur Standar Titik Masuk CSR (Vanilla / React)

HTML dasar yang diterima dari server web:

```html
<!DOCTYPE html>
<html lang="id">
<head>
  <meta charset="UTF-8" />
  <title>Aplikasi Dashboard CSR</title>
  <!-- Browser mengunduh bundle JS sebelum merender apa pun -->
  <script defer src="/assets/main.bundle.js"></script>
</head>
<body>
  <!-- Titik jangkar kosong tempat JavaScript akan menyuntikkan seluruh pohon UI -->
  <div id="root"></div>
</body>
</html>
```

File entry point JavaScript (`main.js`):

```javascript
import React, { useEffect, useState } from "react";
import ReactDOM from "react-dom/client";

function App() {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    // Pengambilan data dilakukan di browser setelah bundle JS selesai dimuat
    fetch("/api/dashboard-summary")
      .then((res) => res.json())
      .then((json) => {
        setData(json);
        setLoading(false);
      });
  }, []);

  if (loading) return <div class="skeleton-loader">Memuat data...</div>;

  return <main><h1>Ringkasan Penjualan: Rp {data.totalRevenue}</h1></main>;
}

const rootElement = document.getElementById("root");
ReactDOM.createRoot(rootElement).render(<App />);
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan
- **Navigasi Halaman Cepat (*Instant Transition*)**: Setelah bundle awal diunduh, berpindah dari halaman A ke B hanya memperbarui komponen yang relevan tanpa re-fetch HTML.
- **Serverless & CDN Friendly**: Dokumen HTML dan bundle JS dapat di-hosting secara murah dan scalable di Object Storage (S3, Cloudflare Pages, Firebase Hosting).
- **Pengurangan Beban Server Komputasi**: Server tidak perlu merender string HTML untuk jutaan pengguna secara bersamaan.

### Kekurangan
- **First Contentful Paint (FCP) Lambat**: Pengguna harus menunggu HTML diunduh, JS diunduh, JS dieksekusi, dan API di-fetch sebelum melihat konten bermakna.
- **Buruk untuk Mesin Pencari (SEO)**: Crawler web selain Google sering kali gagal mengeksekusi JavaScript, sehingga metadata og-image dan konten tidak terbaca.
- **Tergantung Performa Hardware Pengguna**: Smartphone murah dengan prosesor lambat akan mengalami waktu parsing JavaScript yang sangat lama (*CPU bottleneck*).

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Pakai
- Dashboard admin, ERP, CRM, portal perbankan, dan aplikasi internal perusahaan.
- Aplikasi web interaktif kompleks di balik login (misal: Trello, Figma Web, Google Docs).
- Aplikasi yang murni mengandalkan komunikasi WebSocket real-time lokal.

### Kapan Jangan Pakai
- Toko online (e-commerce), katalog produk publik, landing page pemasaran, portal berita/blog publik di mana SEO dan kecepatan muat awal adalah penentu konversi penjualan. (Gunakan **SSR** atau **SSG**).

## 8. Metrik Kunci & Observabilitas

- **First Contentful Paint (FCP)**: Waktu dari saat navigasi dimulai hingga browser merender teks/gambar pertama. Pada CSR, target FCP: **< 1.8 detik**.
- **Time to Interactive (TTI)**: Waktu saat halaman sudah sepenuhnya responsif terhadap klik pengguna. Pada CSR, TTI sering kali tertinggal jauh di belakang FCP jika ukuran bundle sangat besar.
- **JavaScript Bundle Size**: Pantau ukuran berkas JavaScript terkompresi (Gzip/Brotli). Aturan produksi: **Total initial JS bundle < 200 KB** (sebelum kompresi < 800 KB).

## 9. Studi Kasus Industri Nyata

### Refactoring Dashboard Finansial Stripe

- **Tantangan Rekayasa Riil**:
  Dashboard pengguna Stripe adalah salah satu aplikasi web terkaya di dunia, memuat grafik analitik transaksi real-time, manajemen sengketa (*disputes*), dan pengaturan API keys. Pengguna berpindah menu puluhan kali sehari.
- **Implementasi Solusi Industri**:
  Stripe mempertahankan arsitektur **CSR (Single Page Application)** untuk area dashboard karena pengguna selalu terautentikasi dan SEO bernilai nol di area tersebut. Namun, untuk mengatasi kelemahan FCP lambat, Stripe menerapkan:
  1. *Route-based Code Splitting*: Hanya modul rute yang sedang dibuka yang diunduh.
  2. *Aggressive Local Caching*: Menyimpan skema form dan layout di IndexedDB lokal.
- **Dampak & Hasil Sistem**:
  - Pengalaman navigasi antar tab pembayaran terasa instan (< 100ms) tanpa ada kedipan layar (*full reload*).
  - Server Stripe terlindungi dari beban rendering HTML yang tidak perlu.

## 10. Fakta Menarik & Gotchas Production

1. **Ilusi Status HTTP 200**:
   Pada CSR, jika pengguna membuka URL yang tidak ada (misal `https://site.com/produk-hantu-999`), web server statis (Nginx/S3) tetap mengembalikan status **HTTP 200 OK** bersama berkas `index.html`. Baru setelah JavaScript berjalan di browser, aplikasi menyadari bahwa ID produk tidak ada dan merender tampilan 404. Bot crawler menganggap halaman tersebut ada (soft 404), yang merusak reputasi SEO domain.
2. **Ketergantungan Ekstrem Jaringan**:
   Jika koneksi internet pengguna terputus tepat saat file `main.bundle.js` sedang diunduh di tengah jalan, aplikasi akan mati total dan hanya menampilkan layar putih selamanya tanpa pesan error yang ramah, kecuali developer menyuntikkan script fallback manual di HTML statis.

## 11. Rangkuman Singkat

1. **Rendering di Browser**: CSR membebankan seluruh proses pembuatan HTML dan fetching data ke perangkat pengguna.
2. **Sempurna untuk Dashboard**: Pilihan terbaik untuk aplikasi berbasis web dinamis di balik autentikasi login di mana SEO tidak diperlukan.
3. **Kelemahan Awal**: Memiliki skor FCP dan SEO yang rentan buruk jika ukuran bundle JavaScript tidak dikontrol secara ketat.
4. **Optimasi Wajib**: Selalu terapkan code-splitting, lazy-loading, dan skeleton loader saat menggunakan arsitektur CSR.
