# CSS and Asset Isolation

## 1. Skenario Sistem Nyata

Sebuah portal SuperApp menggabungkan dua Micro-App independen di halaman yang sama:
1. `mfe-dashboard` (Tim Internal): Menggunakan library UI berbasis Bootstrap dengan styling tabel kustom.
2. `mfe-recommendation` (Tim Data Science): Menggunakan Tailwind CSS versi terbaru.

Ketika kedua aplikasi di-mount ke layar, bencana visual terjadi:
Styling *Preflight* bawaan milik Tailwind CSS di Micro-App Rekomendasi me-reset seluruh elemen HTML global:
```css
/* Tailwind Preflight yang bocor ke global scope */
h1, h2, h3 { font-size: inherit; font-weight: inherit; }
button { background-color: transparent; border-color: inherit; }
```

Akibatnya, seluruh tombol biru dan heading tebal milik Bootstrap di Micro-App Dashboard mendadak kehilangan warnanya, menjadi transparan, dan teks judul menyusut menjadi ukuran font normal. Tim Dashboard panik mengira kode mereka dirusak hacker.

Bencana tabrakan styling lintas aplikasi ini dikenal sebagai **CSS Bleeding / CSS Collision**. Masalah ini diselesaikan melalui strategi **CSS & Asset Isolation**.

## 2. Apa Itu dan Buat Apa?

CSS & Asset Isolation adalah kumpulan teknik arsitektur dan perkakas kompilasi untuk mengisolasi aturan gaya CSS, font, dan aset statis (gambar/ikon) milik suatu Micro-App agar tidak bocor (*bleed*) dan merusak tampilan Micro-App lain yang berjalan berdampingan di halaman yang sama.

Masalah mendasar CSS di browser:
Secara alamiah, tag `<style>` dan `<link rel="stylesheet">` yang disuntikkan ke dalam `<head>` bersifat **Global Scope (Cacat Kaskade Global)**. Kelas bernama `.btn` di Micro-App A akan menimpa kelas `.btn` di Micro-App B jika spesifisitas selektornya lebih tinggi atau urutan pemuatannya lebih akhir.

Tujuan isolasi aset adalah menjamin bahwa styling dan path aset suatu tim murni berlaku di dalam kotak pasir (*sandbox*) mereka sendiri.

## 3. Kenapa Harus Pakai Ini? (Empat Strategi Utama Isolasi CSS)

1. **Shadow DOM (Web Components)**:
   - Fitur native browser yang menyediakan batas isolasi DOM dan CSS mutlak (*Encapsulation Boundary*). Gaya CSS di dalam Shadow Root sama sekali tidak bisa keluar ke halaman utama, dan gaya dari luar tidak bisa masuk ke dalam.
2. **PostCSS Namespace Prefixing (`postcss-prefix-selector`)**:
   - Menambahkan pembungkus kelas unik secara otomatis pada saat build time (misal: `#mfe-recommendation-root .btn`), sehingga styling hanya aktif di dalam container elemen tersebut.
3. **CSS Modules & BEM Scoping**:
   - Mengubah nama kelas menjadi hash unik secara otomatis (misal: `.btn_a1b2c_3`), mencegah tabrakan nama kelas secara instan.
4. **Tailwind CSS Prefixes & Preflight Disabling**:
   - Mematikan reset global (`preflight: false`) dan menambahkan prefix kelas khusus (`prefix: 'rec-'` -> `rec-flex rec-text-blue-500`).

## 4. Cara Kerjanya

Berikut adalah diagram perbandingan antara CSS bocor (global namespace) vs CSS terisolasi menggunakan **PostCSS Namespace Prefixing & Shadow DOM**:

```
--- 1. TANPA ISOLASI (CSS BLEEDING) ---
[ MFE-A (Tailwind) ] -> inject: button { background: transparent; }
                               |
                               v
               [ GLOBAL BROWSER DOCUMENT HEAD ]
                               |
                               v
[ MFE-B (Bootstrap) ] -> Tombol Bootstrap mendadak ikut transparan! (RUSAK)


--- 2. DENGAN ISOLASI NAMESPACE POSTCSS ---
Saat Build Time:
PostCSS otomatis mengubah setiap selektor:
  button { background: transparent; }
              |
              v
  #mfe-a-root button { background: transparent; }
              |
              v
[ BROWSER RUNTIME ]
- Hanya tombol di dalam <div id="mfe-a-root"> yang transparan.
- Tombol di <div id="mfe-b-root"> TETAP BIRU AMAN! (ISOLASI BERHASIL)
```

## 5. Cuplikan Logika Inti

### 1. Konfigurasi PostCSS Prefixing (`postcss.config.js`)

Solusi paling praktis di industri tanpa perlu merombak kode komponen:

```javascript
// remote-app/postcss.config.js
module.exports = {
  plugins: [
    require('tailwindcss'),
    require('autoprefixer'),
    // Otomatis bungkus seluruh selektor dengan ID container aplikasi
    require('postcss-prefix-selector')({
      prefix: '#mfe-recommendation-root',
      // Kecualikan elemen html/body agar tidak rusak
      transform(prefix, selector, prefixedSelector) {
        if (selector.startsWith('html') || selector.startsWith('body')) {
          return selector;
        }
        return prefixedSelector;
      },
    }),
  ],
};
```

### 2. Mengatasi Masalah Path Aset Gambar (`publicPath: 'auto'`)

Masalah umum lain di Micro Frontends: saat Remote App mencoba memuat gambar `<img src="/logo.png">`, browser secara default mencarinya di domain Host (`https://host.com/logo.png`), padahal gambar tersebut tersimpan di CDN Remote (`https://remote-cdn.com/logo.png`).

Solusinya adalah mengonfigurasi Webpack `output.publicPath: 'auto'`:

```javascript
// remote-app/webpack.config.js
module.exports = {
  output: {
    // 'auto' membuat Webpack menghitung path aset secara dinamis berdasarkan
    // lokasi skrip remoteEntry.js diunduh saat runtime!
    publicPath: 'auto',
  },
};
```

### 3. Menggunakan Shadow DOM untuk Isolasi Mutlak

```tsx
// remote-app/src/mountShadowApp.tsx
import React from 'react';
import ReactDOM from 'react-dom/client';
import App from './App';
// Impor file CSS mentah sebagai string teks
// @ts-ignore
import cssText from './index.css?inline';

export function mountInShadow(container: HTMLElement) {
  // Buat Shadow Root mode 'open'
  const shadowRoot = container.attachShadow({ mode: 'open' });

  // Suntikkan stylesheet khusus ke dalam Shadow Root
  const styleTag = document.createElement('style');
  styleTag.textContent = cssText;
  shadowRoot.appendChild(styleTag);

  // Mount React ke dalam Shadow DOM
  const reactRootContainer = document.createElement('div');
  shadowRoot.appendChild(reactRootContainer);

  const root = ReactDOM.createRoot(reactRootContainer);
  root.render(<App />);

  return () => root.unmount();
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

| Metode Isolasi | Kelebihan | Kekurangan / Risiko |
| :--- | :--- | :--- |
| **PostCSS Namespace Prefixing** | Sangat mudah dipasang, modal/popover tetap bisa keluar ke body | Tidak mengisolasi font-face atau keyframe animation names |
| **Shadow DOM** | Isolasi gaya 100% mutlak dan didukung engine browser native | Masalah pada elemen Portal/Modal yang harus menempel di `document.body` |
| **CSS Modules** | Ringan, zero-collision nama kelas di level komponen | Tidak memecahkan masalah jika salah satu tim membawa CSS reset global liar |

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Memilih PostCSS Prefixing
- Kasus paling umum: Tim menggunakan Tailwind CSS atau library komponen UI biasa (Ant Design / Bootstrap) dan ingin mencegah reset global bocor ke aplikasi tetangga.

### Kapan Menggunakan Shadow DOM
- Ketika Micro-App Anda adalah widget pihak ketiga yang akan disematkan di website pelanggan eksternal yang styling-nya tidak bisa Anda kontrol (misal: widget Chatbot Intercom atau Zendesk).

## 8. Metrik Kunci & Observabilitas

1. **Unscoped Global CSS Rules Count**: Jalankan linter CSS (Stylelint) di pipeline CI untuk melarang selektor elemen telanjang tanpa kelas (misal melarang aturan `h1 { ... }` atau `* { box-sizing: border-box; }` di level remote app).
2. **Missing Asset 404 Rate**: Pantau log HTTP 404 pada aset gambar/font di CDN. Jika ada 404, hampir pasti remote app lupa menyetel `publicPath: 'auto'`.
3. **Shadow DOM Portal Leaks**: Pastikan modal popover yang dibuat di dalam Shadow DOM tetap mewarisi styling yang tepat jika menggunakan React Portal.

## 9. Studi Kasus Industri Nyata

Sebuah portal perbankan investasi memiliki aplikasi Shell yang mengintegrasikan Micro-App Kartu Kredit dan Micro-App Reksa Dana.

Ketika Tim Reksa Dana mengadopsi Tailwind CSS, tombol konfirmasi transaksi di Tim Kartu Kredit mendadak kehilangan padding dan border-nya, berubah menjadi teks biru polos yang membingungkan nasabah.

Analisis akar masalah:
Tailwind Preflight menyertakan aturan `button { border-style: solid; border-width: 0; }` yang disuntikkan langsung ke `<head>` browser global.

Perbaikan arsitektural:
1. Mematikan preflight Tailwind di tim Reksa Dana: `corePlugins: { preflight: false }`.
2. Menerapkan `postcss-prefix-selector` pada seluruh bundler remote dengan pembungkus `#mfe-mutual-funds`.
3. Menyetel Webpack `output.publicPath = 'auto'` untuk menyelesaikan masalah gambar ikon bank yang sebelumnya 404.
4. Tampilan antarmuka kedua tim kembali stabil 100% berdampingan secara damai.

## 10. Fakta Menarik & Gotchas Production

- **Dilema Modal & Tooltip di Shadow DOM**: Jika Anda menggunakan library UI seperti Material UI atau Radix di dalam Shadow DOM, komponen Modal secara default merender dirinya di `document.body` via React Portal. Karena `document.body` berada di **luar** Shadow Root, modal tersebut akan kehilangan seluruh gaya CSS-nya dan tampil hancur! Solusinya adalah mengonfigurasi prop `container` pada Portal agar merender di dalam Shadow Root lokal.
- **Font-Family Kaskade**: Shadow DOM mengisolasi selektor CSS, namun nilai properti seperti `font-family`, `color`, dan `line-height` **tetap diwariskan secara alami (*inherited*)** dari dokumen induk luar ke dalam Shadow DOM.
- **`@keyframes` Collisions**: Jika dua remote app mendefinisikan nama animasi yang sama (misal `@keyframes spin { ... }`), animasi yang dimuat terakhir di `<head>` akan menimpa animasi sebelumnya. Gunakan penamaan animasi ber-prefix (misal `@keyframes mfe-cards-spin`).

## 11. Rangkuman Singkat

CSS & Asset Isolation adalah perlindungan mutlak terhadap benturan visual di arsitektur Micro Frontends. Mengombinasikan penonaktifan reset global liar, penerapan PostCSS Namespace Prefixing pada kontainer unik, dan pengaturan `publicPath: 'auto'` menjamin bahwa styling dan aset masing-masing tim tetap terisolasi dan tampil sempurna.
