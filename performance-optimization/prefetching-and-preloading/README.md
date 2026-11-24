# Prefetching & Preloading

## 1. Skenario Sistem Nyata

Pada sebuah portal e-commerce, seorang pembeli sedang berada di halaman keranjang belanja (*Shopping Cart*). Pengguna meninjau daftar barang belanjaannya selama 30 detik sebelum akhirnya menekan tombol biru besar bertuliskan: **"Lanjut ke Pembayaran (Checkout)"**.

Tanpa optimasi prefetching:
- Ketika tombol checkout diklik, browser baru mulai meminta berkas chunk JavaScript halaman checkout (`checkout.chunk.js` seberat 450 KB), CSS form pembayaran, dan memuat SDK gateway pembayaran (Stripe / Midtrans).
- Di koneksi seluler 4G yang padat, proses pengunduhan ini membutuhkan waktu 2,8 detik. Pengguna disajikan layar loading berputar (*spinner*) yang lama, menimbulkan keraguan (*checkout hesitation*) dan meningkatkan rasio pembatalan transaksi (*cart abandonment*).

Padahal, browser sebenarnya menganggur (*idle*) selama 30 detik saat pengguna sedang membaca daftar keranjang!

Solusi: Menggunakan teknik **Prefetching & Preloading**. Saat pengguna berada di halaman keranjang, browser memanfaatkan waktu senggangnya untuk mengunduh modul halaman checkout di latar belakang secara hening (**Prefetch**). Ketika tombol "Lanjut ke Pembayaran" ditekan, seluruh aset sudah tersedia di memory/disk cache lokal; transisi ke halaman checkout terjadi secara **instan dalam waktu 0 milidetik**!

## 2. Apa Itu dan Buat Apa?

Prefetching dan Preloading adalah bagian dari **Resource Hints (Petunjuk Sumber Daya)** standar peramban web yang memungkinkan pengembang memberitahu mesin peramban tentang sumber daya apa saja yang kritis saat ini atau yang kemungkinan besar akan dibutuhkan di masa depan, sehingga peramban dapat mengunduh atau menginisialisasi koneksi lebih awal.

Pembeda utama antar tipe Resource Hints:
1. **`<link rel="preload">`**: Menginstruksikan peramban: *"Aset ini SANGAT KRITIS untuk halaman SAAT INI (misal font utama atau gambar banner LCP). Unduh sekarang juga dengan prioritas tinggi!"*.
2. **`<link rel="prefetch">`**: Menginstruksikan peramban: *"Aset ini kemungkinan besar akan dibutuhkan untuk NAVIGASI BERIKUTNYA (halaman lain). Unduh di waktu luang (idle) dengan prioritas rendah!"*.
3. **`<link rel="modulepreload">`**: Versi khusus untuk mengunduh, mem-parsing, dan mengompilasi berkas **ES Modules (`import`)** lebih awal.
4. **`<link rel="preconnect">`**: Melakukan jabat tangan awal (*DNS lookup, TCP handshake, dan TLS negotiation*) ke domain pihak ketiga sebelum request data sebenarnya dikirimkan.
5. **`<link rel="dns-prefetch">`**: Hanya melakukan resolusi DNS awal untuk domain eksternal.

Fungsi utama Resource Hints:
- Menghilangkan latensi jaringan pada navigasi antarhalaman (*near-instant page transitions*).
- Mempercepat kemunculan elemen LCP dan mencegah efek font berkedip (*Flash of Unstyled Text / FOUT*).

## 3. Kenapa Harus Pakai Ini?

Browser secara alami bersifat "reaktif": browser baru mengunduh berkas font web setelah selesai mem-parsing berkas CSS dan menemukan aturan `@font-face`. Hal ini menciptakan rantai ketergantungan jaringan yang panjang (*network waterfall delay*).

Dengan Preload dan Prefetch, kita beralih menjadi "proaktif": kita memberitahu browser apa yang harus diunduh sebelum browser sendiri menyadarinya.

## 4. Cara Kerjanya (Diagram Alur Preload vs Prefetch)

```
KASUS 1: PRELOAD (Untuk Halaman Saat Ini - Prioritas TINGGI)
[ Dokumen HTML mulai di-parse ]
             |
  Mendeteksi: <link rel="preload" href="/fonts/inter.woff2" as="font" crossorigin>
             |
             v
Peramban LANGSUNG menyisipkan unduhan font di antrean teratas jaringan!
(Font selesai diunduh SEBELUM parser CSS selesai membaca @font-face)
HASIL: Teks langsung tampil dengan font yang indah tanpa kedipan FOUT!

----------------------------------------------------------------------

KASUS 2: PREFETCH (Untuk Halaman Masa Depan - Prioritas RENDAH)
[ Pengguna sedang membaca Halaman Keranjang Belanja ]
             |
  Mendeteksi: <link rel="prefetch" href="/js/checkout.chunk.js">
             |
             v
Peramban menunggu sampai CPU dan Jaringan berada dalam status SENGGANG (Idle)
             |
             v
Peramban mengunduh checkout.chunk.js di background dan menyimpannya di HTTP Disk Cache
             |
             v
Pengguna mengklik tombol "Checkout" ---> DIJAWAB INSTAN DARI DISK CACHE (0ms)!
```

## 5. Cuplikan Logika Inti

### Deklarasi Resource Hints Lengkap di Tag `<head>` HTML

```html
<head>
  <!-- 1. PRECONNECT: Selesaikan handshake TLS ke domain API & CDN font lebih awal -->
  <link rel="preconnect" href="https://api.example.com" />
  <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin />

  <!-- 2. PRELOAD: Unduh aset LCP kritis untuk halaman saat ini dengan prioritas tinggi -->
  <!-- Wajib sertakan atribut 'as' agar browser menetapkan prioritas yang tepat! -->
  <link 
    rel="preload" 
    href="/fonts/inter-bold.woff2" 
    as="font" 
    type="font/woff2" 
    crossorigin 
  />
  <link 
    rel="preload" 
    href="/images/hero-banner.avif" 
    as="image" 
    type="image/avif" 
    fetchpriority="high" 
  />

  <!-- 3. PREFETCH: Unduh chunk rute berikutnya di waktu luang peramban -->
  <link rel="prefetch" href="/assets/checkout-page.chunk.js" as="script" />
</head>
```

### Implementasi Prefetching Cerdas Berbasis Hover Tautan (Vanilla JS)

Mengunduh rute hanya jika kursor mouse pengguna mengambang (*hover*) di atas tautan menu selama minimal 65 milidetik:

```javascript
// Mengunduh chunk halaman berikutnya tepat saat ada niat klik (intent to click)
function setupLinkPrefetching() {
  const links = document.querySelectorAll("a[data-prefetch]");

  links.forEach((link) => {
    let hoverTimer = null;

    link.addEventListener("mouseenter", () => {
      // Tunggu 65ms untuk memastikan bukan gerakan kursor mouse yang tidak sengaja lewat
      hoverTimer = setTimeout(() => {
        const targetUrl = link.getAttribute("href");
        prefetchRoute(targetUrl);
      }, 65);
    });

    link.addEventListener("mouseleave", () => {
      if (hoverTimer) clearTimeout(hoverTimer);
    });
  });
}

function prefetchRoute(url) {
  // Cek apakah link prefetch sudah pernah dibuat sebelumnya
  if (document.querySelector(`link[rel="prefetch"][href="${url}"]`)) return;

  const prefetchLink = document.createElement("link");
  prefetchLink.rel = "prefetch";
  prefetchLink.href = url;
  document.head.appendChild(prefetchLink);
  console.log(`Modul untuk ${url} sedang di-prefetch di latar belakang...`);
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan
- **Navigasi Sekejap Mata**: Navigasi halaman terasa secepat aplikasi native desktop tanpa layar putih kosong.
- **Optimasi Metrik LCP & FCP**: Menghilangkan waktu tunggu antrean jaringan pada aset visual utama.
- **Pemanfaatan Waktu Idle**: Mengubah waktu senggang peramban yang terbuang menjadi persiapan produktif.

### Kekurangan & Risiko Over-Prefetching
- **Pemborosan Kuota Bandwidth (*Wasted Bytes*)**: Jika Anda melakukan prefetch pada 10 halaman sekaligus dan pengguna ternyata langsung menutup website, kuota internet seluler pengguna terbuang sia-sia untuk aset yang tidak pernah dibuka.
- **Perebutan Bandwidth Jaringan (*Bandwidth Contention*)**: Jika Anda terlalu banyak memasang `<link rel="preload">`, aset preload tersebut akan berebut bandwidth dengan berkas JavaScript kritis halaman saat ini, yang justru **memperlambat** waktu muat awal!

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Memakai Preload
- Gambar Hero LCP di atas lipatan layar.
- Berkas font tipografi kustom utama yang digunakan oleh judul dan teks paragraf.
- File CSS kritis.

### Kapan Harus Memakai Prefetch
- Halaman berikutnya dalam alur corong konversi pasti (*Linear Funnel*, misal: Langkah 1 Form Keranjang -> Langkah 2 Form Pembayaran).
- Tautan navigasi utama yang di-hover oleh kursor pengguna.

### Kapan JANGAN Menggunakan Prefetch
- Pengguna yang mengaktifkan mode hemat data seluler: Periksa selalu flag `navigator.connection.saveData`. Jika `saveData === true`, matikan seluruh fitur prefetch otomatis!

## 8. Metrik Kunci & Observabilitas

- **Largest Contentful Paint (LCP)**: Memasang preload pada gambar LCP yang benar biasanya memangkas waktu LCP sebesar **500 ms hingga 1.500 ms**.
- **Console Warning (Unused Preload)**: Perhatikan tab Console di Chrome DevTools. Jika ada peringatan kuning: *"The resource was preloaded using link preload but not used within a few seconds"*, artinya Anda salah memasang preload pada aset yang tidak dibutuhkan!
- **Network Tab Priority Column**: Pastikan berkas preload berstatus prioritas `High` / `Very High`, dan berkas prefetch berstatus prioritas `Lowest`.

## 9. Studi Kasus Industri Nyata

### Optimasi Navigasi Instan pada Quicklink di Google

- **Tantangan Rekayasa Riil**:
  Banyak situs web berita memiliki ratusan tautan artikel di beranda. Mengunduh semua artikel tersebut di muka akan mematikan koneksi internet pengguna ponsel di negara berkembang.
- **Implementasi Solusi Industri**:
  Addy Osmani dan tim Google merilis pustaka **Quicklink**:
  1. Menggunakan `IntersectionObserver` untuk memantau tautan mana saja yang sedang terlihat di layar ponsel.
  2. Memeriksa status koneksi pengguna: jika berada di jaringan 2G/3G lambat atau `saveData: true`, prefetch dibatalkan.
  3. Jika pengguna berada di jaringan 4G/WiFi cepat dan browser dalam status senggang (`requestIdleCallback`), Quicklink secara otomatis melakukan prefetch pada tautan yang terlihat di layar.
- **Dampak & Hasil Sistem**:
  - Halaman artikel berita terbuka secara instan dalam waktu kurang dari 50 milidetik saat diklik.
  - Pengurangan drastis tingkat pentalan (*bounce rate*) pembaca media.

## 10. Fakta Menarik & Gotchas Production

1. **Wajib Atribut `as` dan `crossorigin` pada Font**:
   Jika Anda melakukan preload pada font:
   ```html
   <!-- BENCANA: Browser akan mengunduh font DUA KALI! -->
   <link rel="preload" href="/font.woff2" as="font">
   ```
   Sesuai spesifikasi web, font WAJIB menyertakan atribut `crossorigin` meskipun di-hosting di domain yang sama: `<link rel="preload" href="/font.woff2" as="font" crossorigin>`. Jika lupa menulis `crossorigin`, peramban akan mengabaikan hasil preload dan mengunduh ulang font tersebut untuk kedua kalinya!
2. **Atribut `fetchpriority` Modern**:
   HTML modern kini mendukung atribut `fetchpriority="high"` dan `fetchpriority="low"` langsung pada tag `<img>` atau `<script>`, memberikan kendali granular tanpa harus membuat tag `<link>` terpisah di head.

## 11. Rangkuman Singkat

1. **Preload untuk Sekarang**: Gunakan `<link rel="preload">` untuk aset super kritis pada halaman yang sedang dibuka saat ini.
2. **Prefetch untuk Nanti**: Gunakan `<link rel="prefetch">` di waktu idle untuk mengantisipasi navigasi halaman berikutnya.
3. **Preconnect untuk Domain Asing**: Gunakan `<link rel="preconnect">` untuk menyelesaikan koneksi TCP/TLS lebih awal ke domain API pihak ketiga.
4. **Hormati Kuota Pengguna**: Jangan melakukan prefetch berlebih pada koneksi seluler lambat atau saat mode hemat kuota (*Save-Data*) aktif.
