# Lazy Loading

## 1. Skenario Sistem Nyata

Pada aplikasi web marketplace belanja online, halaman detail produk memuat banyak elemen berat:
- Galeri foto produk beresolusi tinggi (20 foto @ 1 MB = 20 MB).
- Widget ulasan pelanggan yang memuat 100 foto testimoni pengguna.
- Modul peta lokasi toko offline berbasis Google Maps SDK (ukuran skrip 450 KB).
- Tab spesifikasi teknis dan tabel perbandingan produk.

Developer junior memuat seluruh aset dan pustaka tersebut sekaligus di awal pemuatan halaman:

```html
<!-- Memuat semua aset secara langsung di awal halaman -->
<script src="https://maps.googleapis.com/maps/api/js?key=..."></script>
<img src="/foto-produk-1.jpg" />
<img src="/foto-produk-2.jpg" />
<!-- ... hingga foto ke-20 ... -->
```

Dampak pada sistem:
- Browser harus mengunduh total payload sebesar **25 Megabyte** di awal sebelum halaman dapat digunakan.
- Pada koneksi seluler 4G biasa, kuota data pengguna terkuras habis hanya untuk membuka 1 halaman produk.
- Lebih buruk lagi, 70% pengguna sebenarnya hanya melihat foto pertama dan langsung menekan tombol beli, tanpa pernah menggulir ke bawah untuk melihat peta atau 100 ulasan tersebut. Seluruh 24 MB data yang diunduh terbuang sia-sia!

Solusi: Menerapkan **Lazy Loading (Pemuatan Malas)**. Hanya aset dan modul yang langsung tampak di layar (*above-the-fold*) yang diunduh di awal. Sisanya ditunda sampai pengguna menggulir mendekati elemen tersebut atau melakukan klik interaksi.

## 2. Apa Itu dan Buat Apa?

Lazy Loading adalah strategi optimasi di mana pemuatan sumber daya sistem yang berat—seperti gambar, video, iframe, atau modul kode JavaScript—**ditunda pelaksanaannya sampai sumber daya tersebut benar-benar dibutuhkan** oleh pengguna.

Kebalikan dari Lazy Loading adalah *Eager Loading* (memuat segalanya di muka sejak awal).

Fungsi utama Lazy Loading:
- **Menghemat Bandwidth Jaringan**: Mengurangi volume data yang harus diunduh oleh client dan kuota egress server/CDN.
- **Mempercepat Waktu Muat Awal (*Initial Load Time*)**: Memangkas waktu tunggu FCP dan LCP karena antrean jaringan browser (*network waterfall*) tidak tersumbat oleh file-file sekunder.
- **Menghemat Memori Perangkat**: Mengurangi beban alokasi memori RAM grafis untuk gambar-gambar yang belum tampak di layar.

## 3. Kenapa Harus Pakai Ini?

Browser memiliki batas maksimal koneksi HTTP simultan (pada HTTP/1.1 hanya 6 koneksi per domain). Jika halaman awal Anda memaksa browser mengunduh 30 gambar secara bersamaan, berkas JavaScript dan CSS kritis yang menentukan tampilan visual utama akan mengantre di belakang gambar-gambar tersebut.

Dengan Lazy Loading, kita memprioritaskan jalur pipa jaringan hanya untuk sumber daya yang berdampak langsung pada pengalaman visual pertama pengguna.

## 4. Cara Kerjanya (Diagram Alur Viewport Trigger)

```
+---------------------------------------------------------------+
|                      LAYAR PENGGUNA (Viewport)                |
|                                                               |
|  [ Gambar Hero Utama ]  ---> DIMUAT SEGERA (Eager Loading)    |
|  [ Judul & Harga     ]                                        |
+---------------------------------------------------------------+
                               |
                               | (Pengguna menggulir layar ke bawah)
                               v
-------------------- BATAS PENGAMATAN AMBANG (Margin 200px) ----
                               |
                               v
+---------------------------------------------------------------+
|                  DI BAWAH LAYAR (Below-the-fold)              |
|                                                               |
|  [ Gambar Ulasan 1 ]    ---> TERTUNDA (Lazy Loading)          |
|  [ Widget Google Maps]  ---> SKRIP BELUM DIUNDUH SAMA SEKALI  |
|                                                               |
|  *Browser mendeteksi elemen menyentuh batas ambang 200px*     |
|  --> Trigger fetch image / import() script!                   |
+---------------------------------------------------------------+
```

## 5. Cuplikan Logika Inti

### Penerapan Lazy Loading pada Media dan Skrip Peta

**1. Gambar dan Iframe Native Browser (Nol JavaScript):**
```html
<!-- Cukup tambahkan atribut loading="lazy" standar HTML modern -->
<img 
  src="/ulasan-produk-1.jpg" 
  loading="lazy" 
  alt="Foto ulasan pembeli" 
  width="400" 
  height="300"
/>

<iframe 
  src="https://maps.google.com/..." 
  loading="lazy" 
  width="600" 
  height="450">
</iframe>
```

**2. Lazy Loading Modul JavaScript Berat Berdasarkan Interaksi Klik:**
```javascript
// Memuat SDK peta 450 KB HANYA jika pengguna mengklik tombol "Lihat Lokasi Toko"
const openMapBtn = document.getElementById("btn-open-map");

openMapBtn.addEventListener("click", async () => {
  openMapBtn.textContent = "Memuat peta...";
  
  // Dynamic import: berkas JS peta baru diunduh detik ini juga!
  const { initMapSDK } = await import("./heavy-map-module.js");
  
  initMapSDK(document.getElementById("map-container"));
  openMapBtn.style.display = "none";
});
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan
- **Pengurangan Ukuran Awal Drastis**: Halaman yang semula berukuran 20 MB dapat menyusut menjadi hanya 800 KB untuk pemuatan pertama.
- **Skor Core Web Vitals Meningkat Pesat**: LCP membaik signifikan karena bandwidth terfokus pada aset utama.
- **Penghematan Finansial**: Memangkas biaya transfer bandwidth CDN cloud hingga 40-60%.

### Kekurangan & Jebakan
- **Keterlambatan Saat Scroll Cepat (*Blank Space / Pop-in*)**: Jika pengguna menggulir layar ponsel dengan sangat cepat, mereka mungkin akan melihat kotak abu-abu kosong sesaat sebelum gambar selesai diunduh. (Mitigasi: Pasang `rootMargin: 300px` agar gambar dimuat sebelum benar-benar masuk layar).
- **Potensi Pergeseran Tata Letak (CLS)**: Jika elemen gambar yang di-lazy load tidak diberi atribut `width` dan `height` atau aspek rasio CSS, kemunculan gambar secara tiba-tiba akan mendorong konten di bawahnya, merusak skor **CLS**.

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Pakai
- Semua gambar dan video yang terletak di bawah layar (*below-the-fold*).
- Komponen berat yang disembunyikan di balik interaksi (Modal pop-up, Tab kedua dan ketiga, Drawer menu navigasi, Date Picker kompleks).
- Library eksternal pihak ketiga (Google Maps, Chart.js, PDF Viewer, reCAPTCHA).

### Kapan JANGAN PERNAH Pakai
- **Gambar Banner Hero / LCP Image**: Jangan pernah menambahkan `loading="lazy"` pada gambar utama paling atas di halaman web! Hal ini akan menunda browser mengunduh gambar tersebut sampai browser selesai menghitung layout, yang justru menghancurkan skor LCP Anda. (Gunakan `loading="eager"` atau `<link rel="preload">` untuk gambar utama).

## 8. Metrik Kunci & Observabilitas

- **Largest Contentful Paint (LCP)**: Target **< 2.5 detik**. Pastikan gambar elemen LCP tidak sengaja terpasang atribut `loading="lazy"`.
- **Total Transferred Bytes**: Bandingkan ukuran total unduhan pada tab Network antara saat pertama kali halaman terbuka vs saat halaman digulir sampai ke bawah.
- **Cumulative Layout Shift (CLS)**: Pastikan elemen placeholder gambar memiliki dimensi cadangan agar CLS tetap **< 0.1**.

## 9. Studi Kasus Industri Nyata

### Optimasi Bandwidth dan LCP pada Medium.com

- **Tantangan Rekayasa Riil**:
  Artikel panjang di Medium sering kali memuat 20 hingga 50 gambar ilustrasi beresolusi tinggi. Mengunduh semua gambar ini di awal membuat waktu muat artikel lambat dan boros kuota pada pembaca seluler.
- **Implementasi Solusi Industri**:
  Medium mempelopori teknik **Blurry Image Placeholder (LQIP / Progressive Lazy Loading)**:
  1. Server menyajikan thumbnail gambar berukuran mikro (hanya 20x10 piksel berukuran < 1 KB) yang di-encode sebagai string Base64 langsung di dalam HTML.
  2. Thumbnail tersebut di-render dengan efek CSS blur.
  3. Menggunakan `IntersectionObserver`, gambar resolusi tinggi penuh baru diunduh ketika paragraf artikel mendekati layar, lalu bertransisi secara halus (*fade-in*) menggantikan gambar blur.
- **Dampak & Hasil Sistem**:
  - Ukuran transfer data awal artikel terpangkas sebesar 80%.
  - Pengalaman visual pembaca sangat elegan tanpa ruang kosong yang mengganggu.

## 10. Fakta Menarik & Gotchas Production

1. **Atribut `loading="lazy"` Bawaan Browser**:
   Sejak tahun 2020, seluruh browser modern (Chrome, Safari, Firefox, Edge) telah mendukung atribut native `<img loading="lazy">`. Anda tidak lagi membutuhkan library JavaScript pihak ketiga berukuran 10 KB hanya untuk melakukan lazy loading gambar standar!
2. **Perilaku Default Threshold Browser**:
   Browser modern menentukan jarak kapan harus mulai mengunduh gambar lazy secara otomatis berdasarkan kecepatan koneksi internet pengguna. Di jaringan 4G cepat, browser mungkin mulai memuat gambar saat jaraknya 1.250 piksel dari layar; sedangkan di jaringan 2G lambat, browser akan mulai memuat pada jarak 2.500 piksel.

## 11. Rangkuman Singkat

1. **Muat Saat Dibutuhkan**: Lazy loading menunda pemuatan sumber daya berat sampai mendekati atau diminta oleh pengguna.
2. **Hemat Bandwidth & Memori**: Mengurangi beban awal jaringan dan alokasi memori RAM secara signifikan.
3. **Wajib Dimensi**: Selalu berikan atribut `width` dan `height` pada media lazy load untuk mencegah layout shift (CLS).
4. **Haram untuk Elemen LCP**: Jangan pernah pasang `loading="lazy"` pada gambar utama/hero di bagian atas halaman web.
