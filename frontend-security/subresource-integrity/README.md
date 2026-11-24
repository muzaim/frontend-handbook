# Subresource Integrity (SRI)

## 1. Skenario Sistem Nyata

Sebuah portal berita dan e-commerce besar memuat library JavaScript jQuery dan font awesome dari CDN publik pihak ketiga:
`<script src="https://code-cdn-gratis.com/jquery-3.6.0.min.js"></script>`

Suatu malam, server CDN publik tersebut berhasil diretas oleh kelompok peretas. Peretas tidak menghapus file jQuery tersebut, melainkan menyisipkan 10 baris kode Trojan di bagian akhir file yang bertugas membelokkan nomor kartu kredit nasabah saat checkout belanja.

Karena URL file tetap sama dan status respons tetap HTTP 200 OK, pengembang e-commerce sama sekali tidak menyadari adanya perubahan. Browser ribuan pengunjung mengunduh file yang telah diracuni tersebut dan mengeksekusinya tanpa curiga.

Bencana *Supply Chain Attack* semacam ini dapat dicegah secara otomatis oleh browser menggunakan fitur **Subresource Integrity (SRI)**.

## 2. Apa Itu dan Buat Apa?

Subresource Integrity (SRI) adalah fitur keamanan bawaan browser modern yang memungkinkan pengembang web memverifikasi bahwa file eksternal yang diunduh (terutama file JavaScript dan stylesheet CSS dari CDN atau pihak ketiga) belum dimodifikasi atau dirusak oleh pihak yang tidak berwenang.

Mekanisme ini bekerja dengan menyematkan nilai hash kriptografis (*cryptographic digest*) yang telah dihitung sebelumnya ke dalam atribut `integrity` pada elemen `<script>` atau `<link>`.

Saat browser mengunduh file dari CDN, browser akan menghitung ulang nilai hash dari konten file tersebut secara mandiri di memori. Jika hash hasil hitungan browser tidak cocok persis dengan atribut `integrity`, browser akan memblokir eksekusi file tersebut seketika.

## 3. Kenapa Harus Pakai Ini?

CDN publik (seperti cdnjs, unpkg, jsdelivr) sangat populer karena mempercepat pengiriman aset dan menghemat bandwidth hosting sendiri. Namun, mengandalkan CDN pihak ketiga membawa risiko besar:
- **CDN Kompromi**: Server CDN bisa diretas oleh hacker.
- **DNS Hijacking**: Penyerang memalsukan catatan DNS sehingga traffic dialihkan ke server tiruan berisi malware.
- **Man-in-the-Middle (MitM)**: Di jaringan Wi-Fi publik tanpa enkripsi yang sempurna, proxy nakal bisa menyuntikkan iklan atau skrip berbahaya ke dalam file JS yang sedang diunduh.

Dengan SRI:
- Browser bertindak sebagai verifikator integritas data yang independen. Meskipun CDN berhasil dibobol hacker dan file diganti, browser pengunjung menolak menjalankan file tersebut karena tanda tangan hash-nya berubah.

## 4. Cara Kerjanya

Berikut adalah diagram alur bagaimana browser memvalidasi integritas file sebelum mengeksekusinya:

```
[ Developer Saat Build Time ]
1. Hitung Hash File: sha384(konten jquery.js) = "vny34uoiwg...=="
2. Tambahkan atribut ke HTML:
   <script src="https://cdn.com/jquery.js" 
           integrity="sha384-vny34uoiwg...==" 
           crossorigin="anonymous"></script>

                                 |
                                 v
[ Pengunjung Membuka Website di Browser ]
                                 |
                                 +---------------------------------------+
                                 |                                       |
                   (Skenario Normal: File Asli)           (Skenario Serangan: CDN Dihack)
                                 |                                       |
                 Browser unduh file dari CDN               Browser unduh file dari CDN
                                 |                                       |
                 Browser hitung hash di memori             Browser hitung hash di memori
                 Hasil: sha384-vny34uoiwg...==             Hasil: sha384-XYZ_MALWARE...==
                                 |                                       |
                 Bandingkan dengan atribut HTML            Bandingkan dengan atribut HTML
                                 |                                       |
                         [ COCOK PERSIS ]                         [ TIDAK COCOK! ]
                                 |                                       |
                                 v                                       v
                    FILE DIEKSEKUSI BROWSER                     BROWSER BLOKIR EKSEKUSI
                                                           (Cetak Network Integrity Error,
                                                            Sistem Terlindungi 100%)
```

## 5. Cuplikan Logika Inti

### Menghitung Nilai Hash SRI via Terminal CLI

Anda bisa menggunakan utility OpenSSL di terminal untuk menghasilkan string atribut integrity secara instan:

```bash
# Menghitung hash SHA-384 dari file lokal atau hasil download
cat jquery-3.6.0.min.js | openssl dgst -sha384 -binary | openssl base64 -A
# Output: vny34uoiwg... (salin nilai ini ke atribut integrity)
```

### Implementasi pada HTML Tag

```html
<!DOCTYPE html>
<html lang="id">
<head>
  <meta charset="UTF-8">
  <title>Aplikasi Aman dengan SRI</title>

  <!-- Verifikasi Stylesheet CSS dari CDN -->
  <link 
    rel="stylesheet" 
    href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" 
    integrity="sha384-9ndCyUaIbzAi2FUVXJi0CjmCapSmO7SnpJef0486qhLnuZ2cdeRhO02iuK6FUUVM" 
    crossorigin="anonymous"
  >
</head>
<body>
  <h1>Portal Transaksi</h1>

  <!-- Verifikasi JavaScript Script dari CDN -->
  <script 
    src="https://code.jquery.com/jquery-3.7.0.min.js" 
    integrity="sha256-2Pmvv0kuTBOenSvLm6bvfBSSHrUJ+3A7x6P5Ebd07/g=" 
    crossorigin="anonymous"
  ></script>
</body>
</html>
```

*Catatan: Atribut `crossorigin="anonymous"` wajib disertakan agar browser meminta resource menggunakan mekanisme CORS. Tanpa header CORS dari CDN, browser akan menolak memeriksa hash demi privasi lintas origin.*

### Otomasi SRI pada Modern Bundler (Vite / Webpack)

Pada proyek modern, Anda tidak perlu menghitung hash secara manual. Gunakan plugin bundler otomatis:

```typescript
// vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import { subresourceIntegrity } from 'vite-plugin-subresource-integrity';

export default defineConfig({
  plugins: [
    react(),
    // Otomatis menghitung hash SHA-384 untuk semua file JS & CSS hasil build
    subresourceIntegrity(),
  ],
  build: {
    sourcemap: false,
  },
});
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

| Kelebihan | Kekurangan / Risiko |
| :--- | :--- |
| Memberikan perlindungan mutlak terhadap pembobolan CDN atau manipulasi MitM | Jika file di CDN di-update tanpa sengaja (misal URL `jquery-latest.js`), halaman akan langsung rusak (*broken*) |
| Performa verifikasi sangat cepat (kriptografi dihitung langsung oleh browser engine C++) | Wajib menyertakan atribut `crossorigin="anonymous"` dan CDN tujuan wajib mengaktifkan CORS header |
| Standar W3C resmi yang didukung oleh semua browser modern | Menambah langkah ekstra pada pipeline CI/CD build jika tidak diotomasi |

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Pakai
- Setiap kali Anda memuat library JavaScript atau stylesheet CSS pihak ketiga dari CDN publik eksternal (Google Fonts, jsDelivr, cdnjs, unpkg, Cloudflare).
- Proyek single-page application berkeamanan tinggi yang menyajikan aset static bundle miliknya sendiri dari domain multi-region CDN.

### Kapan Jangan Pakai
- File yang kontennya berubah-ubah secara dinamis pada URL yang sama (misal skrip bertarget `bundle.js?v=latest`). Menggunakan SRI pada skrip dinamis akan menyebabkan pemblokiran seketika saat konten diperbarui.
- Resource gambar biasa (`<img>`), video, atau font, karena browser membatasi pemeriksaan SRI utama pada tag `<script>` dan `<link rel="stylesheet">`.

## 8. Metrik Kunci & Observabilitas

1. **SRI Integrity Check Failure Rate**: Pantau log browser error console via Sentry atau Datadog RUM. Jika error `Failed to find a valid digest in the 'integrity' attribute` melonjak tiba-tiba, segera periksa apakah ada rilis deploy baru yang menghasilkan hash berbeda dari yang tertulis di HTML.
2. **CDN Delivery Latency**: Pastikan verifikasi hash di sisi client tidak menambah waktu proses rendering UI yang signifikan (pada file berukuran wajar, overhead komputasi SHA browser < 2ms).

## 9. Studi Kasus Industri Nyata

Sebuah platform analytics web populer bernama *Browsealoud* (yang menyediakan fitur text-to-speech untuk ribuan website pemerintahan dan perbankan di Inggris dan Amerika Serikat) pernah diretas.

Hacker menyusupkan skrip penambang kripto *Coinhive* ke dalam file JavaScript resmi Browsealoud di CDN mereka. Akibatnya, lebih dari 4.000 situs web penting (termasuk situs pengadilan UK dan otoritas medis NHS) mendadak mengeksekusi skrip penambang kripto di browser para pengunjungnya.

Namun, segelintir situs instansi yang telah menerapkan Subresource Integrity (SRI) pada tag script Browsealoud mereka sama sekali tidak terdampak. Browser pengunjung situs-situs tersebut mendeteksi perubahan hash SHA pada file CDN dan langsung memblokir pemuatan skrip, menyelamatkan pengunjung dari eksploitasi CPU.

## 10. Fakta Menarik & Gotchas Production

- **Wajib CORS**: Jika Anda menyematkan atribut `integrity` tanpa `crossorigin="anonymous"`, atau jika server CDN tidak mengirimkan header `Access-Control-Allow-Origin: *`, browser akan memblokir resource tersebut secara total demi keamanan privasi.
- **Algoritma yang Didukung**: Standar W3C mendukung awalan algoritma `sha256-`, `sha384-`, dan `sha512-`. Di industri modern, `sha384-` adalah algoritma yang paling banyak direkomendasikan karena memberikan rasio keamanan dan panjang karakter optimal.
- **Multi-Hash Support**: Anda dapat mendefinisikan beberapa hash sekaligus yang dipisahkan oleh spasi di atribut `integrity`. Browser akan memilih algoritma terkuat yang didukungnya.

## 11. Rangkuman Singkat

Subresource Integrity (SRI) adalah fitur browser yang memastikan file skrip eksternal dari CDN belum mengalami manipulasi dengan cara mencocokkan hash kriptografis konten file terhadap nilai atribut `integrity`. SRI adalah pertahanan garis terdepan terhadap serangan rantai pasok (*supply chain attacks*) pada dependensi web pihak ketiga.
