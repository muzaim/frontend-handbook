# Module Federation Internals

## 1. Skenario Sistem Nyata

Sebuah portal perbankan enterprise mengadopsi arsitektur Micro Frontends. Aplikasi Host (Shell) memuat dua remote aplikasi independen: `mfe-cards` (Tim Kartu Kredit) dan `mfe-loans` (Tim Pinjaman).

Saat rilis baru di-deploy ke staging, console browser mendadak dibanjiri error fatal:
`Uncaught Error: Invalid hook call. Hooks can only be called inside the body of a function component. (Multiple copies of React in the same application)`

Setelah ditelusuri lewat tab Network DevTools, browser mengunduh paket `react.js` sebanyak 3 kali dari 3 origin yang berbeda: Host mengunduh React 18.2.0, `mfe-cards` mengunduh React 18.2.0 terpisah, dan `mfe-loans` mengunduh React 18.1.0.

Karena masing-masing bundle membawa salinan React sendiri di memori JavaScript (*multiple instances*), dispatcher internal React Context dan Hooks menjadi kacau dan menyebabkan crash total.

Masalah duplikasi library dan konflik runtime ini diselesaikan di level bundler melalui pemahaman **Module Federation Internals & Shared Dependencies Negotiation**.

## 2. Apa Itu dan Buat Apa?

Module Federation (diperkenalkan di Webpack 5 dan kini hadir di ekosistem Vite / Rsbuild) adalah mekanisme arsitektur kompilasi dan runtime yang memungkinkan sebuah aplikasi JavaScript mengeksekusi kode dari aplikasi JavaScript lain secara dinamis di browser tanpa perlu menginstal dependensi tersebut saat build time.

Pilar internal terpenting dari Module Federation adalah **Shared Scope Management (Manajemen Dependensi Bersama)**:
Sistem negosiasi cerdas yang bertugas memastikan bahwa pustaka-pustaka inti yang bersifat stateful (seperti React, ReactDOM, Vue, atau RxJS) hanya diinisialisasi **satu kali sebagai Singleton** di memori browser, terlepas dari berapa banyak Micro-App yang dimuat di halaman tersebut.

## 3. Kenapa Harus Pakai Ini?

- **Mencegah Ledakan Ukuran Bundle (Bundle Bloat)**: Tanpa negosiasi shared scope, jika Anda memiliki 10 Micro-App di satu halaman dan masing-masing membawa vendor React + Lodash sendiri, browser akan mengunduh ratusan kilobyte kode identik secara berulang-ulang, menghancurkan metrik LCP.
- **Menjamin Keutuhan State Hook & Context**: Pustaka seperti React, Redux, atau TanStack Query bergantung pada instance memori tunggal (*singleton instance*). Jika instance terbelah menjadi dua, React Context tidak bisa menyalurkan state dari Host ke Remote.
- **Resolusi Versi Semantik Otomatis (SemVer Negotiation)**: Jika Host membutuhkan React `^18.2.0` dan Remote membutuhkan React `^18.1.0`, Module Federation runtime secara otomatis memilih versi tertinggi yang kompatibel (`18.2.0`) untuk digunakan bersama oleh kedua aplikasi.

## 4. Cara Kerjanya

Berikut adalah diagram alur bagaimana Module Federation Runtime menginisialisasi **Shared Scope** saat Host memuat `remoteEntry.js`:

```
[ BROWSER MEMUAT HOST APP ]
            |
            |-- 1. Inisialisasi __webpack_share_scopes__.default di window
            |      Host mendaftarkan versinya: React 18.2.0 (Loaded: YES)
            v
[ HOST MEMINTA REMOTE APP: mfe-loans ]
            |
            |-- 2. Unduh remoteEntry.js dari CDN Remote
            v
[ REMOTE ENTRY INITIALIZATION: mfe_loans.init(shareScope) ]
            |
            |-- 3. Remote menyerahkan daftar dependensinya:
            |      - "Saya butuh React ^18.0.0 (singleton: true)"
            v
+---------------------------------------------------------------+
|  NEGOSIASI RUNTIME MODULE FEDERATION                          |
|                                                               |
|  - Apakah React sudah tersedia di shared scope? -> YA (18.2.0)|
|  - Apakah 18.2.0 memenuhi syarat ^18.0.0?       -> YA         |
|                                                               |
|  KEPUTUSAN:                                                   |
|  Gunakan React 18.2.0 milik Host!                             |
|  Batalkan pengunduhan chunk React milik Remote!               |
+---------------------------------------------------------------+
            |
            v
[ BROWSER MENGEKSEKUSI REMOTE COMPONENT ]
Remote berjalan menggunakan React milik Host. RAM aman, 0 duplikasi!
```

## 5. Cuplikan Logika Inti

### Konfigurasi Module Federation di Sisi Host (`webpack.config.js`)

```javascript
// host-app/webpack.config.js
const { ModuleFederationPlugin } = require('webpack').container;
const deps = require('./package.json').dependencies;

module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: 'shell_host',
      // Daftarkan container remote statis
      remotes: {
        mfe_cards: 'mfe_cards@https://cards.perusahaan.com/remoteEntry.js',
        mfe_loans: 'mfe_loans@https://loans.perusahaan.com/remoteEntry.js',
      },
      // Konfigurasi negosiasi pustaka bersama
      shared: {
        react: {
          singleton: true,          // WAJIB: Hanya izinkan 1 instance React di memori
          requiredVersion: deps.react, // Validasi versi semantik (^18.2.0)
          strictVersion: true,      // Berikan error jika remote memaksa versi yang tidak kompatibel
        },
        'react-dom': {
          singleton: true,
          requiredVersion: deps['react-dom'],
          strictVersion: true,
        },
      },
    }),
  ],
};
```

### Konfigurasi di Sisi Remote App (`webpack.config.js`)

```javascript
// remote-loans/webpack.config.js
const { ModuleFederationPlugin } = require('webpack').container;
const deps = require('./package.json').dependencies;

module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: 'mfe_loans',
      filename: 'remoteEntry.js', // Manifest entry point yang diunduh oleh Host
      exposes: {
        // Komponen yang diekspos ke publik
        './LoanApplicationWidget': './src/widgets/LoanApplicationWidget.tsx',
      },
      shared: {
        react: {
          singleton: true,
          requiredVersion: deps.react,
        },
        'react-dom': {
          singleton: true,
          requiredVersion: deps['react-dom'],
        },
      },
    }),
  ],
};
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

| Kelebihan | Kekurangan / Risiko |
| :--- | :--- |
| Mengeliminasi duplikasi library vendor besar secara dinamis di runtime browser | Jika versi dependensi antar tim terlampau jauh (misal React 17 vs React 18), resolusi singleton akan gagal |
| Proses instalasi npm lokal antar tim terpisah 100% tanpa tight-coupling repositori | Menambah titik kegagalan jaringan: jika file `remoteEntry.js` gagal diunduh, Host harus menangani error boundary |
| Mendukung pembaruan fitur independen tanpa perlu re-build dan re-deploy aplikasi Host | Debugging error runtime di balik abstraction layer Module Federation lebih rumit dibanding bundler monolit |

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Mengonfigurasi `singleton: true`
- Pustaka-pustaka inti berbasis global state atau memory context: `react`, `react-dom`, `vue`, `pinia`, `styled-components`, `@tanstack/react-query`.

### Kapan Boleh Mengizinkan Duplikasi (`singleton: false`)
- Pustaka utilitas stateless murni tanpa side effect: `lodash-es`, `date-fns`, `numeral`. Jika Host dan Remote menggunakan versi minor yang berbeda, membiarkan keduanya membawa salinan sendiri jauh lebih aman daripada memaksa singleton yang bisa menimbulkan breaking change logika utilitas.

## 8. Metrik Kunci & Observabilitas

1. **Shared Scope Duplication Warnings**: Pantau console log di browser. Jika muncul pesan warning `[webpack-dev-server] Shared module react has multiple versions in scope`, segera periksa `package.json` tim yang menyimpang.
2. **`remoteEntry.js` File Size**: File manifest `remoteEntry.js` harus berukuran sangat kecil (< 5-10 KB) karena hanya berisi tabel pemetaan (*manifest pointer*) dan shared scope logic, bukan kode komponen sebenarnya.
3. **Chunk Loading Latency**: Pantau waktu unduhan chunk remote saat komponen di-load secara dinamis via `React.lazy()`.

## 9. Studi Kasus Industri Nyata

Sebuah platform e-commerce retail memiliki aplikasi Shell yang mengintegrasikan Micro-App Katalog Produk dan Micro-App Ulasan Pengguna.

Awalnya, tim tidak menyetel konfigurasi `shared` di Module Federation. Akibatnya, setiap halaman memuat React, ReactDOM, dan library icon (Lucide) sebanyak 3 kali dari masing-masing container. Total ukuran unduhan JavaScript halaman produk mencapai 1.8 MB dengan waktu First Contentful Paint (FCP) mencapai 4.2 detik di jaringan 4G.

Langkah optimasi arsitektural:
1. Mengonfigurasi shared scope terpusat di Host dan seluruh Remote: `react`, `react-dom`, dan library styling `lucide-react` disetel sebagai `singleton: true`.
2. Hasilnya, ukuran unduhan JavaScript terpangkas 55% dari 1.8 MB menjadi 800 KB.
3. Waktu FCP turun drastis menjadi 1.9 detik, dan bug *Invalid Hook Call* di lingkungan produksi teratasi sepenuhnya.

## 10. Fakta Menarik & Gotchas Production

- **Bootstrap Async Boundary**: Pada aplikasi Module Federation, file entry point (`index.ts`) biasanya hanya berisi satu baris: `import('./bootstrap');`. Ini adalah trik wajib agar Webpack memiliki jeda waktu asinkron untuk menegosiasikan shared scope dan mengunduh dependensi bersama sebelum kode aplikasi utama dieksekusi.
- **StrictVersion Error Trap**: Jika Anda menyetel `strictVersion: true` dan salah satu remote menggunakan versi React yang berbeda minor (misal Host 18.2.0, Remote 18.3.0), Module Federation akan melempar error dan menolak menjalankan remote. Di production, gunakan `strictVersion: false` dengan `singleton: true` agar bundler memilih versi tertinggi yang tersedia tanpa mematikan aplikasi.
- **Ekosistem Non-Webpack**: Module Federation kini bukan lagi milik eksklusif Webpack. Plugin `@originjs/vite-plugin-federation` memungkinkan aplikasi Vite saling berbagi remote dengan aplikasi Webpack secara lintas bundler.

## 11. Rangkuman Singkat

Module Federation Internals mengandalkan Shared Scope Runtime untuk menegosiasikan dan menyatukan dependensi library (seperti React) sebagai singleton tunggal di memori browser. Dengan konfigurasi `singleton: true` dan pemahaman siklus `remoteEntry.js`, aplikasi Micro Frontends terbebas dari duplikasi bundle dan konflik runtime instansi.
