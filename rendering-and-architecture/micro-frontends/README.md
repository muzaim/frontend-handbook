# Micro Frontends

## 1. Skenario Sistem Nyata

Pada sebuah perusahaan perbankan digital skala besar (SuperApp), aplikasi web portal nasabah dikerjakan oleh 15 tim rekayasa independen (total lebih dari 120 insinyur perangkat lunak):
- Tim Tabungan & Rekening
- Tim Kartu Kredit & Pinjaman
- Tim Investasi & Reksadana
- Tim Pembayaran Tagihan & PPOB

Awalnya, seluruh aplikasi dibangun di atas satu repositori frontend raksasa (*Monolithic Frontend*).

Dampak pada sistem dan organisasi:
- **Kemacetan Rilis (*Deployment Bottleneck*)**: Untuk merilis perbaikan bug kecil pada fitur Pembayaran Tagihan, seluruh aplikasi harus di-build dan di-deploy bersama-sama. Jika Tim Investasi memiliki bug pada kodenya, rilis seluruh 15 tim lainnya terblokir!
- **Konflik Dependency (*Dependency Hell*)**: Tim Kartu Kredit ingin memperbarui versi React ke versi terbaru untuk menggunakan fitur baru, tetapi Tim Tabungan masih terikat pada pustaka pihak ketiga lama yang tidak kompatibel.
- **Waktu Build Membengkak**: Menjalankan tes otomatis dan build CI/CD membutuhkan waktu 45 menit untuk setiap perubahan kecil (*Pull Request*).

Solusi: Mengadopsi arsitektur **Micro Frontends**. Memecah aplikasi monolitik menjadi aplikasi-aplikasi frontend kecil yang independen secara otonom, masing-masing dimiliki, dikembangkan, diuji, dan di-deploy secara mandiri oleh timnya sendiri ke server atau CDN mereka sendiri, lalu disatukan menjadi satu kesatuan antarmuka yang mulus di browser pengguna.

## 2. Apa Itu dan Buat Apa?

Micro Frontends adalah pola arsitektur perangkat lunak di mana sebuah aplikasi frontend berskala besar didekomposisi menjadi beberapa aplikasi frontend independen yang lebih kecil, yang masing-masing merepresentasikan satu domain bisnis tertentu.

Konsep ini merupakan perpanjangan dari prinsip **Microservices** di dunia backend ke lapisan antarmuka pengguna (UI).

Teknik integrasi Micro Frontends yang populer di industri:
1. **Module Federation (Webpack 5 / Vite)**: Memuat dan berbagi kode komponen JavaScript antar aplikasi secara dinamis saat runtime melalui jaringan tanpa perlu instalasi NPM.
2. **Web Components (Custom Elements)**: Membungkus aplikasi mikro menggunakan standar web native (`<custom-element>`) dengan isolasi gaya CSS Shadow DOM.
3. **Iframes**: Pendekatan isolasi total paling sederhana dan aman, namun memiliki keterbatasan dalam fleksibilitas tata letak dan performa memori.
4. **Server-Side Routing / Gateway Routing**: Nginx atau Cloudflare menyatukan aplikasi mikro berdasarkan awalan rute URL (misal `/app/rekening` diarahkan ke Server A, `/app/investasi` ke Server B).

## 3. Kenapa Harus Pakai Ini?

Micro Frontends **bukanlah solusi teknis untuk performa browser, melainkan solusi organisasi untuk skalabilitas tim (*Organizational Scaling Pattern*)**.

Jika perusahaan Anda memiliki puluhan developer yang sering mengalami konflik merge di Git, perselisihan jadwal rilis, atau keterikatan versi pustaka, Micro Frontends memberikan otonomi penuh:
- Setiap tim bebas merilis kode ke production 10 kali sehari tanpa perlu izin atau koordinasi jadwal dengan tim lain.
- Tim memiliki otonomi memilih teknologi atau meng-upgrade dependensi secara bertahap.

## 4. Cara Kerjanya (Diagram Alur Module Federation Runtime)

Visualisasi penggabungan aplikasi Host (Shell) dengan aplikasi Remote di browser pengguna:

```
[ Browser Pengguna ] membuka: https://bank.example.com
                         |
                         v
+---------------------------------------------------------------+
|                    APLIKASI HOST (APP SHELL)                  |
|  - Menyediakan Navbar, Sidebar, Autentikasi Pengguna          |
|  - Dikelola oleh Tim Fondasi Platform                         |
+---------------------------------------------------------------+
          |                                            |
          | Mengunduh secara asinkron via jaringan     |
          v                                            v
+-----------------------------------+   +-----------------------------------+
|     REMOTE 1: TIM TABUNGAN        |   |     REMOTE 2: TIM INVESTASI       |
|  https://savings.cdn.com/remote.js|   |  https://invest.cdn.com/remote.js |
|  - Menampilkan Saldo & Mutasi     |   |  - Menampilkan Portofolio Saham   |
|  - Deploy mandiri via Server A    |   |  - Deploy mandiri via Server B    |
+-----------------------------------+   +-----------------------------------+
```

## 5. Cuplikan Logika Inti

### Konfigurasi Module Federation di Webpack / Vite

**1. Aplikasi Remote (Tim Investasi - `vite.config.js`):**
```javascript
import { defineConfig } from "vite";
import federation from "@originjs/vite-plugin-federation";

export default defineConfig({
  plugins: [
    federation({
      name: "investment_app",
      filename: "remoteEntry.js",
      // Komponen yang diekspos agar bisa diimpor oleh aplikasi lain melalui jaringan
      exposes: {
        "./PortfolioWidget": "./src/components/PortfolioWidget.jsx",
      },
      // Berbagi dependensi agar browser tidak mengunduh React dua kali!
      shared: ["react", "react-dom"],
    }),
  ],
});
```

**2. Aplikasi Host / App Shell (Tim Platform):**
```javascript
// Shell App mengonsumsi widget dari tim investasi secara dinamis
import React, { Suspense, lazy } from "react";

// Mengimpor komponen dari URL CDN Tim Investasi secara asinkron saat runtime!
const PortfolioWidget = lazy(() => import("investment_app/PortfolioWidget"));

export function DashboardPage() {
  return (
    <div className="dashboard-layout">
      <h1>Selamat Datang di Bank Digital</h1>
      <Suspense fallback={<div>Memuat widget investasi...</div>}>
        <PortfolioWidget />
      </Suspense>
    </div>
  );
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan
- **Otonomi Rilis Penuh**: Tim A dapat melakukan deploy perbaikan tanpa risiko merusak aplikasi Tim B.
- **Batas Domain Kode Bersih**: Mencegah kode spaghetti karena setiap repositori terisolasi ketat.
- **Pembaruan Teknologi Bertahap**: Memungkinkan migrasi teknologi lama ke teknologi baru per modul tanpa perlu merombak (*rewrite*) seluruh sistem sekaligus.

### Kekurangan & Risiko Signifikan
- **Overhead Ukuran Bundle Jaringan**: Jika konfigurasi *shared dependencies* keliru, browser bisa mengunduh 3 versi React yang berbeda atau duplikasi pustaka styling, membuat ukuran halaman membengkak hingga puluhan megabyte.
- **Kompleksitas Lingkungan Operasional**: Membutuhkan infrastruktur CI/CD yang jauh lebih rumit, versioning kontrak antar remote, dan orkestrasi deployment yang matang.
- **Tantangan Konsistensi Desain Visual (Design System)**: Rentan terjadi inkonsistensi antarmuka (warna tombol, ukuran font) jika Design System tidak diterapkan secara ketat antar tim.

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Pakai
- Perusahaan skala enterprise dengan lebih dari 30-50 insinyur frontend yang terbagi dalam beberapa domain bisnis otonom.
- Aplikasi web raksasa yang menggabungkan beberapa produk independen di bawah satu atap portal (SuperApp, dashboard AWS/Google Cloud Console).
- Kasus migrasi bertahap dari aplikasi monolitik warisan (*legacy*) ke framework modern.

### Kapan JANGAN PERNAH Pakai
- Startup tahap awal, tim kecil (< 15 developer), atau proyek dengan skala menengah ke bawah. Menggunakan Micro Frontends pada tim kecil adalah bentuk *Over-Engineering* fatal yang hanya akan menghabiskan waktu tim mengurus konfigurasi build daripada merilis fitur produk.

## 8. Metrik Kunci & Observabilitas

- **Duplicate Dependency Warnings**: Pantau log bundler untuk mendeteksi pustaka kembar yang terunduh ganda di browser.
- **Remote Entry Latency**: Memantau waktu pengunduhan file `remoteEntry.js` dari masing-masing remote CDN.
- **Cross-Application Error Tracking**: Gunakan Sentry dengan tagging `micro_frontend_name` agar error di production langsung terkirim ke channel tim pemilik modul yang bertanggung jawab.

## 9. Studi Kasus Industri Nyata

### Transformasi Arsitektur Marketplace Amazon Web

- **Tantangan Rekayasa Riil**:
  Halaman produk Amazon (`amazon.com`) memiliki ratusan widget: kotak beli (*Buy Box*), ulasan pembeli, rekomendasi produk serupa, navigasi header, dan pelacak pengiriman. Ratusan tim di berbagai belahan dunia mengelola fitur-fitur ini. Monolit frontend membuat Amazon sering mengalami kelambatan rilis di era 2000-an.
- **Implementasi Solusi Industri**:
  Amazon memelopori arsitektur Micro Frontends paling awal di industri (dikenal secara internal sebagai *Gurupa / UI Composition Framework*). Setiap tim memiliki server independen yang menghasilkan fragmen HTML untuk widget masing-masing. Server proxy terdepan menggabungkan fragmen-fragmen tersebut menjadi satu dokumen HTML sebelum dikirimkan ke pembeli.
- **Dampak & Hasil Sistem**:
  - Amazon mampu melakukan deployment ribuan kali per hari tanpa pernah memblokir jadwal rilis tim lain.
  - Jika widget ulasan sedang mengalami gangguan, widget utama tombol "Beli Sekarang" tetap berfungsi 100% normal tanpa terganggu.

## 10. Fakta Menarik & Gotchas Production

1. **Konflik Variabel Global & CSS**:
   Jika Tim A dan Tim B sama-sama mendefinisikan kelas CSS global `.btn { background: red; }`, gaya tampilan tombol Tim B bisa tertimpa secara tidak sengaja!  
   *Solusi*: Terapkan CSS Modules, Tailwind CSS dengan prefix, atau CSS-in-JS dengan scope otomatis pada setiap aplikasi mikro.
2. **Komunikasi Antar Aplikasi Mikro**:
   Hindari mengaitkan state management (seperti menghubungkan satu Redux store raksasa untuk semua tim). Komunikasi antar aplikasi mikro harus seminimal mungkin, gunakan standar terbuka browser seperti **Custom Events (`window.dispatchEvent(new CustomEvent(...))`)** untuk menjaga isolasi domain.

## 11. Rangkuman Singkat

1. **Pemisahan Domain**: Micro frontends memecah monolit frontend menjadi aplikasi-aplikasi mikro independen per domain bisnis.
2. **Solusi Organisasi**: Ditujukan untuk menyelesaikan friksi skala tim besar, bukan untuk mempercepat kinerja rendering browser.
3. **Module Federation**: Standar industri modern untuk memuat dan berbagi dependensi antar aplikasi mikro saat runtime.
4. **Hindari Prematur**: Jangan gunakan pada tim kecil atau aplikasi skala menengah karena beban operasionalnya sangat tinggi.
