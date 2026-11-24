# Image Optimization

## 1. Skenario Sistem Nyata

Sebuah portal media gaya hidup dan majalah digital memuat foto sampul (*cover article*) beresolusi 4000 x 3000 piksel dalam format file JPEG mentah berukuran **6,2 Megabyte**. Foto tersebut diunggah langsung oleh editor tanpa proses kompresi atau transformasi ukuran.

Ketika seorang pengguna membuka artikel tersebut menggunakan ponsel dengan lebar layar hanya 390 piksel di jaringan seluler 4G:
- Ponsel pengguna terpaksa mengunduh seluruh berkas 6,2 MB data mentah tersebut.
- Layar ponsel yang hanya berdimensi 390 piksel tidak memerlukan resolusi 4000 piksel. Lebih dari 90% piksel gambar terbuang sia-sia saat diperkecil oleh browser!
- Proses decoding gambar 6,2 MB memakan memori RAM ponsel sebesar 48 MB, menyebabkan tab browser terasa lambat dan gambar membutuhkan waktu 5,5 detik untuk tampil (merusak skor LCP).

Solusi: Mengimplementasikan strategi **Image Optimization (Optimasi Gambar Menyeluruh)**: mengonversi format lama ke format generasi baru (**WebP** dan **AVIF**), menyajikan resolusi gambar responsif menggunakan tag `<picture>` dan atribut `srcset` / `sizes`, serta memanfaatkan kompresi otomatis berbasis Image CDN.

## 2. Apa Itu dan Buat Apa?

Image Optimization adalah serangkaian proses rekayasa untuk menghasilkan gambar web dengan **ukuran berkas (*file size*) sekecil mungkin tanpa menurunkan kualitas persepsi visual mata manusia secara nyata (*perceptual quality*)**, serta menyajikan dimensi gambar yang tepat sesuai dengan resolusi layar perangkat pengguna.

Pilar-pilar utama optimasi gambar modern:
1. **Format Generasi Baru (Next-Gen Formats)**:
   - **WebP**: Mengurangi ukuran berkas sebesar 25-35% dibandingkan JPEG/PNG pada kualitas visual setara.
   - **AVIF (AV1 Image File Format)**: Format paling mutakhir berbasis kompresi video AV1, memangkas hingga 50% ukuran berkas dibanding JPEG dengan ketajaman warna luar biasa.
2. **Gambar Responsif (*Responsive Images*)**: Menyediakan variasi ukuran gambar yang berbeda menggunakan atribut `srcset` dan `sizes`, membiarkan browser memilih ukuran yang paling pas.
3. **Kompresi Visual (*Lossy & Lossless Compression*)**: Menghilangkan metadata yang tidak perlu (informasi kamera EXIF, profil warna berlebih) dan mereduksi detail warna mikro yang tidak dapat dibedakan oleh mata manusia.

## 3. Kenapa Harus Pakai Ini?

Pengguna web mengakses situs Anda melalui spektrum perangkat yang sangat bervariasi: mulai dari monitor desktop 4K 32 inci hingga smartphone murah berlayar 5 inci.

Menyajikan satu file gambar ukuran raksasa untuk semua perangkat adalah inefisiensi arsitektur yang fatal. Optimasi gambar:
1. Memangkas waktu muat LCP (*Largest Contentful Paint*) dari hitungan detik ke sub-detik.
2. Menghemat bandwidth CDN dan kuota internet seluler pengguna secara masif.
3. Menjaga ketajaman grafis tetap tajam pada layar berkepadatan piksel tinggi (*Retina / High-DPI screens*).

## 4. Cara Kerjanya (Diagram Alur Negosiasi Format Browser)

Bagaimana tag `<picture>` menegosiasikan format terbaik yang didukung oleh browser:

```
[ Browser Pengguna ] meminta render gambar:
             |
             v
+---------------------------------------------------------------+
|                      TAG <picture> HTML                       |
+---------------------------------------------------------------+
|  1. Apakah browser mendukung format modern AVIF?              |
|     <source type="image/avif" srcset="foto.avif" />           |
|                                                               |
|     [YA] -----> UNDUH AVIF! (Hanya 45 KB - Paling Cepat!)     |
|                                                               |
|     [TIDAK] --> 2. Apakah browser mendukung format WebP?     |
|                 <source type="image/webp" srcset="foto.webp" />|
|                                                               |
|                 [YA] -----> UNDUH WebP! (Hanya 75 KB)         |
|                                                               |
|                 [TIDAK] --> 3. Fallback Format Warisan        |
|                             <img src="foto.jpg" /> (180 KB)   |
+---------------------------------------------------------------+
```

## 5. Cuplikan Logika Inti

### Implementasi Gambar Responsif Modern dengan Tag `<picture>` dan `srcset`

Menyajikan format AVIF/WebP dengan beberapa variasi dimensi layar:

```html
<!-- Menggabungkan Next-Gen Formats dan Resolusi Responsif -->
<picture>
  <!-- Opsi 1: Format AVIF untuk browser modern (Chrome, Firefox, Safari terbaru) -->
  <source
    type="image/avif"
    srcset="
      /images/hero-400.avif   400w,
      /images/hero-800.avif   800w,
      /images/hero-1200.avif 1200w
    "
    sizes="(max-width: 640px) 100vw, (max-width: 1024px) 50vw, 1200px"
  />

  <!-- Opsi 2: Format WebP sebagai fallback kedua yang didukung sangat luas -->
  <source
    type="image/webp"
    srcset="
      /images/hero-400.webp   400w,
      /images/hero-800.webp   800w,
      /images/hero-1200.webp 1200w
    "
    sizes="(max-width: 640px) 100vw, (max-width: 1024px) 50vw, 1200px"
  />

  <!-- Opsi 3: Fallback gambar JPEG standar dengan dimensi presisi anti-CLS -->
  <img
    src="/images/hero-800.jpg"
    alt="Pemandangan Kantor Pusat Modern"
    width="1200"
    height="675"
    fetchpriority="high"
    decoding="async"
    class="hero-banner"
  />
</picture>
```

### Parameter Otomatis Image CDN (Cloudinary / Cloudflare Images / Imgix)

Jika menggunakan Image CDN, Anda cukup mengubah parameter URL secara dinamis tanpa perlu menyimpan puluhan file di server:

```javascript
// Mengonversi URL gambar asli menjadi versi teroptimasi otomatis via Image CDN
function getOptimizedImageUrl(originalUrl, targetWidth) {
  // f_auto: browser otomatis menerima AVIF atau WebP sesuai kemampuan
  // q_auto: kompresi kualitas visual pintar secara otomatis
  // w_{targetWidth}: potong gambar sesuai lebar yang diminta
  return `https://res.cloudinary.com/demo/image/fetch/f_auto,q_auto,w_${targetWidth}/${encodeURIComponent(
    originalUrl
  )}`;
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan
- **Reduksi Bobot Halaman Ekstrem**: Ukuran total halaman web dapat berkurang hingga 70-80%.
- **Peningkatan Skor LCP Signifikan**: Berkas AVIF yang berukuran hanya puluhan kilobyte selesai diunduh dalam sekejap di jaringan 4G.
- **Kualitas Visual Retina Terjaga**: Perangkat dengan Device Pixel Ratio (DPR) tinggi menerima varian gambar 2x secara otomatis.

### Kekurangan & Biaya
- **Waktu Kompresi AVIF Lambat (*Encoding Overhead*)**: Proses konversi ke format AVIF membutuhkan daya CPU yang cukup intensif saat build time atau pada server CDN.
- **Biaya Tambahan Image CDN**: Menggunakan layanan Image CDN pihak ketiga berbayar jika volume trafik jutaan gambar per bulan.

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Memakai Optimasi Menyeluruh
- Semua jenis gambar fotografi, banner promosi, katalog produk e-commerce, dan aset visual pemasaran.
- Halaman artikel berita, blog, atau portal media massa dengan banyak gambar konten.

### Kapan Memilih Format Lain (SVG / CSS)
- Ikon antarmuka, logo perusahaan, ilustrasi vektor sederhana: **Gunakan SVG murni**, bukan WebP atau AVIF! SVG berbasis matematika vektor sehingga ukurannya hanya 1-2 KB dan tidak pernah pecah di resolusi mana pun.
- Gradasi warna latar belakang atau bayangan: **Gunakan CSS murni**, jangan gunakan gambar bitmap.

## 8. Metrik Kunci & Observabilitas

- **Total Image Payload Size**: Pantau di tab Network DevTools dengan memfilter tab *Img*. Total ukuran gambar di atas lipatan layar (*above-the-fold*) disarankan **< 300 KB**.
- **Lighthouse Image Audit**:
  - *"Serve images in next-gen formats"* (Harus Lolos).
  - *"Properly size images"* (Memastikan gambar tidak disajikan dalam ukuran 2000px untuk kontainer 300px).
  - *"Efficiently encode images"* (Memastikan kompresi kualitas visual diatur di level optimal, biasanya quality 80-85%).

## 9. Studi Kasus Industri Nyata

### Transformasi Pengiriman Gambar di Netflix Web

- **Tantangan Rekayasa Riil**:
  Halaman beranda Netflix menampilkan poster ribuan film dan serial TV. Pengguna mengakses Netflix dari berbagai penjuru dunia dengan kualitas koneksi internet seluler yang sangat beragam di negara berkembang.
- **Implementasi Solusi Industri**:
  Netflix beralih dari format JPEG lama ke format **AVIF**:
  1. Melakukan kompresi otomatis seluruh katalog poster film ke format AVIF.
  2. Menerapkan parameter algoritma *Structural Similarity Index Measure (SSIM)* untuk menjamin teks judul film pada poster tetap tajam terbaca tanpa artefak blur.
- **Dampak & Hasil Sistem**:
  - Ukuran poster berkurang rata-rata sebesar 50% dibandingkan JPEG dan 20% lebih kecil dibanding WebP.
  - Waktu muat gambar pada koneksi lambat meningkat pesat dan tingkat kegagalan pemuatan gambar (*image load failure*) turun drastis.

## 10. Fakta Menarik & Gotchas Production

1. **Aturan Kualitas 80-85%**:
   Menyimpan gambar dengan kualitas kompresi 100% (*quality: 100*) menghasilkan ukuran file 4x lebih besar dibanding kualitas 85%, padahal mata manusia normal tidak dapat membedakan selisih kualitas tersebut pada layar monitor biasa. Standar emas industri web adalah menggunakan level kualitas **80% hingga 85%**.
2. **Kepadatan Piksel (DPR) pada Atribut `sizes`**:
   Banyak developer salah paham mengira `100vw` selalu berarti gambar diunduh pada resolusi lebar layar. Pada layar iPhone dengan Retina Display (DPR: 3), browser sebenarnya mengalikan kebutuhan lebar gambar dengan 3x untuk menjaga ketajaman piksel!

## 11. Rangkuman Singkat

1. **Format Generasi Baru**: Utamakan format AVIF dan WebP untuk menghemat 30-50% ukuran berkas dibanding JPEG/PNG.
2. **Gunakan Tag Picture**: Terapkan tag `<picture>` untuk menyediakan format modern dengan fallback aman ke format standar.
3. **Resolusi Responsif**: Selalu pasangkan atribut `srcset` dan `sizes` agar ponsel tidak mengunduh gambar resolusi desktop 4K.
4. **SVG untuk Vektor**: Gunakan SVG untuk logo dan ikon antarmuka; gunakan WebP/AVIF untuk foto dan grafik kompleks.
