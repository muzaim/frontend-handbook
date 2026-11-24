# Lazy Loading Images

## 1. Skenario Sistem Nyata

Pada sebuah portal galeri fotografi dan portofolio arsitektur, satu halaman pameran memuat 80 foto karya seni resolusi tinggi dengan ukuran rata-rata 1,5 MB per gambar (total muatan halaman melebihi **120 Megabyte**).

Ketika pengguna dengan smartphone membuka halaman pameran tersebut melalui jaringan seluler:
- Browser secara membabi buta langsung membuka puluhan koneksi jaringan paralel untuk mengunduh seluruh 80 foto sekaligus dari atas sampai paling bawah halaman.
- Antrean jaringan tersumbat total. File CSS dan font tipografi yang menentukan tampilan judul pameran tertahan di antrean jaringan (*network waterfall starvation*).
- Pengguna melihat layar putih kosong selama 8 detik. Kuota data internet pengguna tersedot hingga 120 MB dalam hitungan detik, padahal pengguna baru melihat 3 foto teratas dan langsung menutup tab.

Solusi: Mengimplementasikan strategi **Lazy Loading Gambar Khusus**, memadukan atribut native browser `loading="lazy"`, teknik placeholder visual (*BlurHash / LQIP*), serta optimasi margin ambang batas (*rootMargin*) agar gambar diunduh tepat sebelum pengguna menggulir layar menyentuh gambar tersebut.

## 2. Apa Itu dan Buat Apa?

Lazy Loading Images adalah teknik penundaan pengunduhan berkas media gambar yang berada di luar area pandang layar pengguna (*offscreen / below-the-fold*) sampai pengguna menggulir layar mendekati posisi gambar tersebut.

Evolusi teknik lazy loading gambar di industri web:
1. **Atribut Native HTML (`loading="lazy"`)**: Didukung secara bawaan oleh seluruh peramban modern tanpa membutuhkan satu baris pun pustaka JavaScript tambahan.
2. **IntersectionObserver API**: Pendekatan terprogram untuk mendeteksi kapan elemen gambar berjarak sekian ratus piksel dari viewport dengan kendali transisi animasi yang mulus.
3. **Teknik Placeholder (LQIP & BlurHash)**: Menampilkan versi gambar sangat buram atau representasi string matematis berukuran mikro (< 100 byte) sebagai penahan tata letak sebelum gambar resolusi penuh selesai diunduh.

Fungsi utama Lazy Loading Images:
- Membebaskan bandwidth jaringan kritis untuk sumber daya prioritas tinggi di awal pemuatan.
- Menghemat kuota transfer data pengguna dan memangkas biaya transfer data keluar (*CDN egress bill*) perusahaan.
- Mencegah kelebihan beban alokasi memori grafis (VRAM) pada peramban seluler.

## 3. Kenapa Harus Pakai Ini?

Gambar menyumbang lebih dari **60% total ukuran muatan data (*bytes transferred*)** pada rata-rata halaman web di dunia saat ini.

Mengunduh puluhan gambar yang belum tentu pernah dilihat oleh pengguna adalah pemborosan sumber daya komputasi dan finansial. Dengan lazy loading gambar, aplikasi web berskala masif dapat mempertahankan waktu muat awal yang konsisten cepat terlepas dari apakah halaman memuat 10 gambar atau 10.000 gambar.

## 4. Cara Kerjanya (Diagram Alur Native vs BlurHash Placeholder)

```
Posisi Pengguna Saat Ini:
+---------------------------------------------------------------+
|                      VIEWPORT LAYAR AKTIF                     |
|                                                               |
|  [ Gambar 1 - Banner Atas ] --> EAGER LOAD (Muat Instan!)     |
|                                 loading="eager"               |
+---------------------------------------------------------------+
                               |
                               v (Pengguna mulai scroll ke bawah)
================ Jarak Ambang Batas (Contoh: 300px) ============
                               |
                               v
+---------------------------------------------------------------+
|                   DI BAWAH LAYAR (Below-the-fold)             |
|                                                               |
|  Status Gambar 2:                                             |
|  1. Menampilkan Canvas BlurHash (Hanya 30 byte string!)        |
|  2. Layout tertahan stabil (Width: 800, Height: 600)          |
|                                                               |
|  *Ketiadaan jarak < 300px memicu request jaringan:*           |
|  --> GET /foto-karya-2.webp                                   |
|  --> Animasi Fade-In halus menggantikan BlurHash              |
+---------------------------------------------------------------+
```

## 5. Cuplikan Logika Inti

### Pola Produksi Lengkap: Gambar Native + Decoding Asinkron + Dimensi Anti-CLS

```html
<!-- Praktik Standar Industri Terbaik untuk Gambar Lazy -->
<div class="image-wrapper" style="aspect-ratio: 16 / 9; background-color: #f1f5f9;">
  <img
    src="/assets/portfolio-item-2.webp"
    alt="Desain Interior Minimalis Modern"
    loading="lazy"
    decoding="async"
    width="1200"
    height="675"
    class="lazy-image"
  />
</div>
```

### Implementasi Komponen Placeholder Cerdas (React + CSS Transition)

Menghilangkan efek gambar berkedip putih saat selesai diunduh:

```tsx
import React, { useState } from "react";

interface LazyImageProps {
  src: string;
  alt: string;
  width: number;
  height: number;
  blurDataUrl?: string; // String Base64 LQIP mikro (misal 20x10 px)
}

export const ProgressiveImage: React.FC<LazyImageProps> = ({
  src,
  alt,
  width,
  height,
  blurDataUrl,
}) => {
  const [isLoaded, setIsLoaded] = useState<boolean>(false);

  return (
    <div 
      className="relative overflow-hidden" 
      style={{ aspectRatio: `${width} / ${height}` }}
    >
      {/* 1. Placeholder blur instan (0 milidetik render) */}
      {blurDataUrl && !isLoaded && (
        <img
          src={blurDataUrl}
          alt={alt}
          className="absolute inset-0 w-full h-full object-cover filter blur-md scale-105"
          aria-hidden="true"
        />
      )}

      {/* 2. Gambar asli yang dimuat secara malas */}
      <img
        src={src}
        alt={alt}
        width={width}
        height={height}
        loading="lazy"
        decoding="async"
        onLoad={() => setIsLoaded(true)}
        className={`w-full h-full object-cover transition-opacity duration-500 ${
          isLoaded ? "opacity-100" : "opacity-0"
        }`}
      />
    </div>
  );
};
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan
- **Efisiensi Bandwidth Maksimal**: Mengurangi transfer data awal hingga lebih dari 75% pada halaman kaya media.
- **Nol Pustaka Eksternal**: Atribut `loading="lazy"` native bekerja otomatis di seluruh browser modern tanpa biaya bundle JavaScript.
- **Peningkatan Skor FCP**: Jalur pipa jaringan menjadi lengang sehingga berkas CSS dan JS kritis tiba lebih cepat.

### Kekurangan & Risiko
- **Efek Gambar Kosong Sesaat (*Blank Pop-in*)**: Jika pengguna melakukan scroll secara agresif pada koneksi seluler lemot, gambar mungkin belum selesai diunduh saat masuk ke viewport.
- **Bahaya Terhadap LCP**: Kesalahan memasang lazy loading pada gambar paling atas (*Hero Image*) akan memperlambat skor LCP secara drastis.

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Pakai
- Seluruh gambar produk, thumbnail artikel, avatar komentar, dan ilustrasi yang berada di bawah lipatan layar (*below-the-fold*).
- Galeri foto, grid katalog e-commerce, feed media sosial tak terbatas (*infinite scroll*).

### Kapan JANGAN PERNAH Pakai
- **Gambar Banner Hero / Elemen LCP**: Jangan pernah pasang `loading="lazy"` pada gambar pertama di atas layar! Browser akan menunda pengunduhan gambar tersebut sampai selesai menghitung layout dokumen, menunda LCP hingga beberapa detik. Untuk gambar hero, gunakan selalu: `loading="eager"` dan `fetchpriority="high"`.

## 8. Metrik Kunci & Observabilitas

- **Largest Contentful Paint (LCP)**: Pastikan elemen gambar LCP tidak memiliki atribut `loading="lazy"`. Nilai LCP harus tetap **< 2.5 detik**.
- **Cumulative Layout Shift (CLS)**: Verifikasi bahwa setiap gambar memiliki atribut `width` dan `height` atau `aspect-ratio` CSS eksplisit agar ruang visual dialokasikan sebelum gambar selesai dimuat (target: **CLS < 0.1**).
- **Network Tab (Resource Priority)**: Periksa kolom *Priority* di tab Network Chrome DevTools. Gambar dengan `loading="lazy"` akan diberi prioritas `Low`, sedangkan gambar biasa diberi prioritas `High`.

## 9. Studi Kasus Industri Nyata

### Penurunan Bounce Rate pada Portal E-Commerce Tokopedia

- **Tantangan Rekayasa Riil**:
  Halaman hasil pencarian barang di Tokopedia memuat puluhan kartu produk dengan foto promosi. Pada perangkat Android dengan jaringan 3G, pemuatan seluruh foto secara serentak membuat scrolling terasa berat dan konsumsi baterai melonjak tinggi.
- **Implementasi Solusi Industri**:
  Tim rekayasa menerapkan arsitektur lazy loading bertingkat:
  1. Empat gambar produk teratas dimuat secara langsung (*eager*).
  2. Gambar produk di bawahnya menggunakan **Native Lazy Loading** dengan atribut `decoding="async"`.
  3. Menggunakan placeholder warna dominan (*dominant color placeholder*) sebelum gambar selesai dimuat.
- **Dampak & Hasil Sistem**:
  - Ukuran transfer data awal halaman pencarian berkurang hingga 60%.
  - Rasio pengunjung yang langsung keluar (*bounce rate*) turun sebesar 12%.

## 10. Fakta Menarik & Gotchas Production

1. **Atribut `decoding="async"`**:
   Mendekode gambar resolusi tinggi (mengubah JPEG terkompresi menjadi bitmap memori) membutuhkan daya CPU yang tidak sedikit. Menambahkan atribut `decoding="async"` memberitahu browser untuk melakukan decoding gambar di thread latar belakang, mencegah frame drop pada main thread saat scroll.
2. **Jangan Sembunyikan dengan `display: none`**:
   Jika Anda memasang `loading="lazy"` pada gambar yang berada di dalam kontainer dengan gaya CSS `display: none`, peramban mungkin tidak akan pernah mengunduh gambar tersebut karena menganggap elemen tidak pernah berada di dalam geometri layout aktif.

## 11. Rangkuman Singkat

1. **Tunda Gambar Luar Layar**: Lazy loading menahan pengunduhan gambar di bawah layar sampai mendekati viewport pengguna.
2. **Gunakan Native Standar**: Gunakan tag bawaan `<img loading="lazy" decoding="async">` untuk implementasi performa tinggi tanpa library eksternal.
3. **Kunci Dimensi**: Selalu tetapkan aspek rasio atau atribut `width` dan `height` untuk mencegah pergeseran tata letak visual (CLS).
4. **Haram di Hero Image**: Jangan pernah memasang lazy loading pada gambar LCP utama paling atas di halaman web Anda.
