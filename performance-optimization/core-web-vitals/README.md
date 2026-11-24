# Core Web Vitals

## 1. Skenario Sistem Nyata

Sebuah portal e-commerce ritel besar mengamati penurunan drastis peringkat pencarian organik di Google Search serta penurunan konversi penjualan sebesar 18% dalam tiga bulan terakhir. 

Setelah tim melakukan audit mendalam, ditemukan beberapa gejala pada halaman produk:
- Gambar banner utama (*Hero Banner*) berukuran 4 MB membutuhkan waktu 4,8 detik untuk tampil di layar ponsel.
- Ketika pengguna menekan tombol "Beli Sekarang", browser terdiam selama 450 milidetik tanpa respons karena CPU main thread sedang sibuk mengeksekusi script pelacak analitik pihak ketiga.
- Tepat saat pengguna hendak menekan tombol beli, banner iklan promosi di bagian atas selesai dimuat dan mendorong tombol ke bawah, menyebabkan pengguna salah mengklik tombol "Syarat & Ketentuan".

Dampak pada sistem:
- Algoritma Google Page Experience secara otomatis mendemosikan peringkat situs karena nilai metrik performa pengguna berada di zona merah (*Poor*).
- Calon pembeli merasa frustrasi akibat interaksi yang macet dan antarmuka yang bergeser tiba-tiba.

Solusi: Mengimplementasikan standardisasi pemantauan dan optimasi **Core Web Vitals** (LCP, INP, dan CLS), mengintegrasikannya ke dalam pipeline CI/CD, serta memantau skor Real User Monitoring (RUM) secara real-time.

## 2. Apa Itu dan Buat Apa?

Core Web Vitals (CWV) adalah inisiatif resmi dari Google yang menetapkan sekumpulan metrik terstandarisasi untuk mengukur kualitas pengalaman pengguna nyata (*Real User Experience*) di web. 

Tiga pilar utama Core Web Vitals:
1. **LCP (Largest Contentful Paint)**: Mengukur kecepatan pemuatan konten visual utama. Menghitung waktu dari awal navigasi hingga elemen teks atau gambar terbesar di dalam viewport selesai digambar di layar.
2. **INP (Interaction to Next Paint)**: Mengukur tingkat responsivitas interaksi halaman. Menghitung latensi terpanjang antara saat pengguna berinteraksi (klik, ketuk, ketik) hingga browser berhasil merender frame visual pembaruan berikutnya di layar (menggantikan metrik lama FID sejak Maret 2024).
3. **CLS (Cumulative Layout Shift)**: Mengukur stabilitas visual halaman. Menghitung skor pergeseran tata letak elemen visual yang tidak terduga selama masa hidup halaman.

Fungsi utama Core Web Vitals:
- Memberikan acuan objektif berbasis angka kuantitatif untuk mengevaluasi performa web.
- Menjadi faktor penentu resmi dalam pemeringkatan algoritma pencarian Google (*SEO Ranking Factor*).

## 3. Kenapa Harus Pakai Ini?

Sebelum adanya Core Web Vitals, developer sering terjebak mengukur metrik sintetis seperti `window.onload` atau `DOMContentLoaded`. Metrik-metrik tersebut tidak mencerminkan apa yang dirasakan manusia: sebuah halaman bisa saja sudah memicu event `load`, namun layarnya masih kosong karena data masih diambil melalui AJAX.

Core Web Vitals berfokus 100% pada perspektif pengguna:
1. Kapan pengguna melihat konten penting? (LCP)
2. Apakah tombol langsung merespons saat ditekan? (INP)
3. Apakah elemen melompat-lompat dan menipu klik pengguna? (CLS)

## 4. Cara Kerjanya (Diagram Ambang Batas Metrik Standar Google)

```
1. LARGEST CONTENTFUL PAINT (LCP) - Kecepatan Muat
[ 0s ----------- 2.5s ] ----------- 4.0s -------------> [ Waktu ]
      BAIK (Hijau)       PERLU PERBAIKAN        BURUK (Merah)

2. INTERACTION TO NEXT PAINT (INP) - Responsivitas Interaksi
[ 0ms ---------- 200ms ] ---------- 500ms ------------> [ Latensi ]
      BAIK (Hijau)       PERLU PERBAIKAN        BURUK (Merah)

3. CUMULATIVE LAYOUT SHIFT (CLS) - Stabilitas Visual
[ 0 ------------ 0.10 ] ----------- 0.25 -------------> [ Skor ]
      BAIK (Hijau)       PERLU PERBAIKAN        BURUK (Merah)
```

## 5. Cuplikan Logika Inti

### Mengukur Core Web Vitals di Sisi Klien Menggunakan Pustaka Resmi `web-vitals`

Mengirim data metrik pengguna nyata (*Real User Monitoring*) ke endpoint telemetri analitik internal:

```javascript
import { onLCP, onINP, onCLS } from "web-vitals";

function sendToAnalytics(metric) {
  const body = JSON.stringify({
    name: metric.name,          // 'LCP', 'INP', atau 'CLS'
    value: metric.value,        // Nilai terukur (ms atau score)
    rating: metric.rating,      // 'good', 'needs-improvement', atau 'poor'
    delta: metric.delta,        // Selisih nilai sejak laporan terakhir
    id: metric.id,              // ID unik navigasi halaman
    navigationType: metric.navigationType, // 'navigate', 'reload', 'back-forward'
  });

  // Gunakan navigator.sendBeacon agar transmisi data tidak terputus saat tab ditutup
  if (navigator.sendBeacon) {
    navigator.sendBeacon("/api/analytics/vitals", body);
  } else {
    fetch("/api/analytics/vitals", { body, method: "POST", keepalive: true });
  }
}

// Inisialisasi pengamatan metrik
onLCP(sendToAnalytics);
onINP(sendToAnalytics);
onCLS(sendToAnalytics);
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan
- **Metrik Standar Lintas Industri**: Diakui secara universal oleh seluruh mesin peramban dan komunitas teknologi global.
- **Dampak Langsung ke Bisnis**: Peningkatan skor berkorelasi langsung dengan kenaikan rasio konversi (*conversion rate*) dan durasi retensi pengunjung.
- **Identifikasi Masalah Presisi**: DevTools memetakan secara tepat elemen mana yang menjadi biang keladi LCP lambat, pembuat shift CLS, atau pemicu delay INP.

### Kekurangan & Tantangan
- **Perbedaan Lab Data vs Field Data**: Nilai pengujian lokal di Lighthouse (*Lab Data*) sering kali berbeda dengan data pengguna asli di lapangan (*Field Data / CrUX*) karena variasi kecepatan prosesor ponsel dan kualitas sinyal seluler pengunjung.
- **Kompromi Bisnis vs Teknis**: Menghapus script analitik pihak ketiga atau iklan dapat memperbaiki skor INP/CLS secara signifikan, tetapi sering kali bertentangan dengan kebutuhan tim monetisasi.

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Menjadi Prioritas Utama
- Seluruh website publik komersial yang mengandalkan trafik pencarian organik Google (E-commerce, portal media berita, landing page SaaS, direktori publik).
- Aplikasi web yang ingin meningkatkan pengalaman pengguna pada perangkat seluler berspesifikasi rendah (*low-end devices*).

### Kapan Bisa Ditoleransi
- Aplikasi web internal perusahaan di balik VPN lokal (*intranet tools*), di mana seluruh karyawan menggunakan laptop berspesifikasi tinggi dan peringkat SEO Google tidak memiliki pengaruh apa pun pada kelangsungan bisnis.

## 8. Metrik Kunci & Observabilitas

- **LCP Element Identification**: Cari elemen dengan atribut `element timing` atau amati panel *Performance* di DevTools untuk menemukan elemen LCP (gambar hero, blok teks `<h1>`, atau video poster).
- **Chrome User Experience Report (CrUX)**: Pantau agregasi performa 28 hari terakhir domain Anda di Google Search Console (menu *Page Experience*).
- **Long Animation Frames API (LoAF)**: Gunakan API modern ini untuk mendiagnosis script spesifik yang memblokir eksekusi frame dan merusak skor INP.

## 9. Studi Kasus Industri Nyata

### Optimasi Web Vitals pada Toko Ritel Internasional Vodafone

- **Tantangan Rekayasa Riil**:
  Vodafone menemukan bahwa peningkatan LCP sebesar beberapa detik menurunkan tingkat penjualan paket internet mereka di Italia. Banyak halaman produk memiliki gambar promosi berat tanpa kompresi dan widget JavaScript yang memblokir proses rendering awal.
- **Implementasi Solusi Industri**:
  Vodafone melakukan optimasi terarah pada metrik LCP:
  1. Menerapkan `<link rel="preload">` pada gambar LCP utama.
  2. Mengonversi format gambar ke WebP modern dengan ukuran responsif.
  3. Memindahkan seluruh script analitik pihak ketiga ke Web Worker menggunakan Partytown.
- **Dampak & Hasil Sistem**:
  - Skor LCP membaik sebesar 31%.
  - Peningkatan penjualan (*sales conversion*) sebesar 8%.
  - Peningkatan jumlah prospek (*lead generation*) sebesar 15%.

## 10. Fakta Menarik & Gotchas Production

1. **INP Menggantikan FID Sejak Maret 2024**:
   Banyak developer masih mengira FID (*First Input Delay*) adalah metrik resmi. FID memiliki kelemahan fatal: ia hanya mengukur keterlambatan interaksi *pertama kali* saat halaman baru terbuka. Sebaliknya, **INP mengukur seluruh interaksi sepanjang pengguna berada di halaman tersebut** dan mengambil nilai latensi terburuk.
2. **Koin LCP Berpindah-pindah**:
   Elemen LCP tidak statis! Jika di awal halaman terdapat teks judul `<h1>`, browser mencatat teks tersebut sebagai kandidat LCP. Namun, jika 2 detik kemudian gambar banner besar selesai diunduh dan muncul di layar, browser memperbarui elemen LCP ke gambar tersebut.

## 11. Rangkuman Singkat

1. **Tiga Pilar Pengalaman**: LCP mengukur kecepatan muat konten utama, INP mengukur kelincahan respons interaksi, dan CLS mengukur kestabilan visual.
2. **Ambang Batas Sehat**: Targetkan LCP < 2.5 detik, INP < 200 milidetik, dan CLS < 0.10.
3. **Pantau Data Lapangan**: Gunakan pustaka `web-vitals` untuk mengirimkan data pengguna nyata (*Field Data*) ke backend analitik.
4. **Faktor Penentu SEO**: Mempertahankan skor Core Web Vitals di zona hijau adalah syarat mutlak untuk memenangkan peringkat teratas di Google Search.
