# Server-Side Rendering (SSR)

## 1. Skenario Sistem Nyata

Pada sebuah portal berita nasional bervolume tinggi, sebuah berita penting (*breaking news*) diterbitkan: "Gempa Bumi Terjadi di Wilayah Pesisir". Dalam waktu 2 menit, ratusan ribu pengguna mengklik tautan berita tersebut dari aplikasi Twitter/X, WhatsApp, dan Google Search.

Jika portal menggunakan arsitektur CSR:
- Server mengirim berkas HTML kosong.
- Tautan yang dibagikan ke WhatsApp tidak memunculkan gambar thumbnail atau ringkasan berita (*Open Graph meta tags* kosong karena bot WhatsApp scraper tidak mengeksekusi JavaScript).
- Ratusan ribu ponsel pengguna secara serentak menembak API backend untuk mengambil isi berita yang sama persis, membuat server database tumbang.

Solusi: Menggunakan **Server-Side Rendering (SSR)** (seperti Next.js, Nuxt, atau Remix). Server web mengeksekusi komponen UI di server Node.js, mengambil data berita dari database, menyuntikkan metadata SEO lengkap, dan langsung mengirimkan dokumen HTML matang (*fully rendered HTML*) dalam hitungan milidetik. Pengguna dan web crawler langsung melihat teks dan gambar seketika pada saat HTML tiba.

## 2. Apa Itu dan Buat Apa?

Server-Side Rendering (SSR) adalah teknik di mana setiap kali ada request halaman dari browser, server web mengeksekusi kode aplikasi frontend (biasanya di lingkungan Node.js/Bun) untuk menghasilkan berkas HTML lengkap yang sudah terisi dengan data dinamis terkini, lalu mengirimkan HTML tersebut langsung ke client.

Setelah HTML matang diterima dan ditampilkan oleh browser, browser kemudian mengunduh berkas JavaScript aplikasi untuk menyambungkan event listener interaktif melalui proses yang disebut **Hydration**.

Fungsi utama SSR:
- **First Contentful Paint (FCP) Instan**: Pengguna langsung membaca konten teks tanpa harus menunggu JavaScript diunduh dan dieksekusi.
- **SEO & Social Sharing Optimal**: Mesin pencari dan bot scraper menerima konten teks dan meta tag Open Graph (`<meta property="og:title">`) secara lengkap sejak byte pertama.
- **Akses Data Terkini**: Konten halaman selalu mencerminkan data terbaru dari database pada detik request diterima.

## 3. Kenapa Harus Pakai Ini?

Untuk halaman-halaman yang menjadi pintu masuk utama bisnis (*acquisition funnel*):
1. **Kecepatan Konversi Pengguna**: Penelitian Google membuktikan bahwa setiap penundaan 1 detik pada waktu muat halaman seluler menurunkan tingkat konversi sebesar 20%. SSR menghilangkan fenomena layar putih (*blank screen*).
2. **Keterbacaan Mesin Pencari Mutlak**: Mengamankan peringkat teratas pada pencarian organik Google dengan menyajikan konten HTML semantik lengkap.
3. **Kerahasiaan Kredensial**: Pemanggilan API backend internal yang membutuhkan API Secret Key dapat dilakukan secara aman di server SSR tanpa pernah membocorkan key tersebut ke browser pengguna.

## 4. Cara Kerjanya (Diagram Alur Request SSR)

```
Pengguna Membuka Halaman (Contoh: GET /berita/gempa-terkini)
                         |
                         v
[Browser] -----------------------------------------------------> [Server Web (Node.js)]
                                                                           |
                                                    1. Ambil data database internal
                                                       const data = await getNews();
                                                                           |
                                                    2. Render komponen ke HTML:
                                                       renderToString(<News data={data} />)
                                                                           |
[Browser] <--- 3. Menerima HTML Lengkap & CSS Matang <--------------------+
     |
     v
*PENGGUNA SUDAH BISA MEMBACA TEKS & MELIHAT GAMBAR!* (FCP Sangat Cepat)
     |
     | (Latar belakang: Browser mengunduh bundle.js)
     v
[Browser] Menjalankan Proses HYDRATION:
          Menghubungkan event handler (klik tombol bookmark, komentar)
     |
     v
*HALAMAN SEPENUHNYA INTERAKTIF!*
```

## 5. Cuplikan Logika Inti

### Konsep Server-Side Rendering Minimal di Node.js

Bagaimana server mengeksekusi komponen React menjadi string HTML:

```javascript
// server.js (Lingkungan Node.js Server)
import express from "express";
import React from "react";
import { renderToString } from "react-dom/server";

const app = express();

function ArticlePage({ title, content }) {
  return (
    <article>
      <h1>{title}</h1>
      <p>{content}</p>
      <button onClick={() => alert("Disimpan!")}>Simpan Artikel</button>
    </article>
  );
}

app.get("/artikel/:id", async (req, res) => {
  // 1. Ambil data dinamis dari database di sisi server
  const articleData = await db.articles.findById(req.params.id);

  // 2. Render komponen React menjadi string HTML murni
  const appHtml = renderToString(<ArticlePage {...articleData} />);

  // 3. Sisipkan hasil render ke dalam dokumen HTML lengkap
  const fullHtml = `
    <!DOCTYPE html>
    <html lang="id">
    <head>
      <title>${articleData.title}</title>
      <meta property="og:title" content="${articleData.title}" />
      <script defer src="/client-bundle.js"></script>
    </head>
    <body>
      <div id="root">${appHtml}</div>
    </body>
    </html>
  `;

  // 4. Kirimkan langsung ke browser dengan status 200 OK
  res.setHeader("Content-Type", "text/html");
  res.send(fullHtml);
});

app.listen(3000);
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan
- **First Contentful Paint (FCP) Super Cepat**: Layar langsung terisi konten begitu paket data HTML pertama selesai di-parse.
- **SEO & Social Share Preview Sempurna**: Dukungan penuh untuk bot pencari dan kartu pratinjau media sosial (Twitter Card, Facebook Share).
- **Keamanan Komunikasi Backend**: Query database dan token rahasia tidak pernah terekspos ke sisi client.

### Kekurangan & Tantangan
- **Time to First Byte (TTFB) Lebih Lambat**: Server harus menunggu query database selesai dan merender string HTML sebelum dapat mengirimkan byte pertama ke browser.
- **Beban CPU Server Melonjak (*High Server Load*)**: Rendering string HTML ribuan kali per detik menguras CPU server Node.js. Diperlukan autoscaling dan caching reverse proxy (Nginx/Cloudflare).
- **Kesenjangan Interaktivitas (*Uncanny Valley*)**: Pengguna sudah melihat tombol di layar, tetapi jika pengguna mengklik tombol tersebut sebelum proses *Hydration* JavaScript selesai, tombol tersebut tidak merespons apa pun.

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Pakai
- Halaman katalog produk e-commerce dengan harga dan stok yang sering berubah.
- Portal berita, media publikasi artikel, dan forum diskusi publik.
- Halaman profil pengguna publik yang membutuhkan indexing mesin pencari.

### Kapan Jangan Pakai
- Halaman dashboard internal perusahaan yang berada di balik form login (gunakan **CSR**).
- Halaman yang kontennya jarang sekali berubah, seperti dokumentasi teknis atau blog statis (gunakan **SSG** karena jauh lebih murah dan cepat tanpa komputasi server runtime).

## 8. Metrik Kunci & Observabilitas

- **Time to First Byte (TTFB)**: Waktu dari pengiriman request hingga byte data pertama diterima browser. Target sehat SSR: **TTFB < 600 ms**. Jika TTFB > 1.5 detik, query database di sisi server harus dioptimalkan atau di-cache.
- **Largest Contentful Paint (LCP)**: Waktu muat elemen konten terbesar (misal gambar hero artikel). Pada SSR, LCP biasanya sangat baik (**< 2.5 detik**).
- **Node.js CPU Utilization**: Memantau kapasitas CPU server rendering SSR. Siapkan alert jika pemakaian CPU menyentuh 75%.

## 9. Studi Kasus Industri Nyata

### Migrasi Halaman Produk TikTok Shop ke Next.js SSR

- **Tantangan Rekayasa Riil**:
  Awalnya antarmuka web TikTok Shop menggunakan arsitektur CSR murni. Ketika pengguna mengklik tautan produk dari video viral, banyak pengguna mengalami layar putih hingga 5 detik di koneksi seluler lambat. Googlebot juga kesulitan mengindeks varian produk baru secara real-time.
- **Implementasi Solusi Industri**:
  Tim TikTok memigrasikan halaman rincian produk (*Product Detail Page*) ke Next.js SSR yang dipadukan dengan layer CDN Edge Caching. Server me-render HTML awal lengkap dengan informasi harga promosi dan ulasan pembeli terkini.
- **Dampak & Hasil Sistem**:
  - Skor LCP membaik sebesar 45% secara global.
  - Trafik organik dari Google Search meningkat sebesar 30% dalam kuartal pertama.

## 10. Fakta Menarik & Gotchas Production

1. **Window / Document is Not Defined**:
   Kesalahan paling umum yang dibuat oleh junior developer saat pertama kali menggunakan framework SSR (Next.js/Remix) adalah mengakses objek peramban seperti `window.innerWidth` atau `localStorage` di dalam siklus hidup awal komponen. Karena kode tersebut dieksekusi di server Node.js (di mana objek `window` tidak ada), aplikasi akan mengalami crash fatal dengan error: `ReferenceError: window is not defined`.  
   *Solusi*: Jalankan kode yang mengakses browser API hanya di dalam hook `useEffect()` atau setelah komponen dipastikan *mounted* di browser.
2. **Ketergantungan Kuat pada Node.js Event Loop**:
   Satu operasi sinkron berat di server SSR (misal melakukan regex kompleks atau serialisasi JSON raksasa) akan memblokir Event Loop Node.js, memperlambat respon request SSR untuk seluruh pengguna lain yang datang di detik yang sama.

## 11. Rangkuman Singkat

1. **Rendering di Server**: SSR menghasilkan dokumen HTML lengkap di server sebelum dikirimkan ke peramban.
2. **FCP & SEO Unggul**: Menghilangkan layar putih kosong dan menjamin keterbacaan sempurna oleh mesin pencari.
3. **TTFB vs FCP Trade-off**: TTFB sedikit lebih lambat karena komputasi server, tetapi FCP menjadi jauh lebih cepat bagi pengguna akhir.
4. **Waspadai Lingkungan Eksekusi**: Pisahkan kode yang hanya boleh berjalan di server dari kode yang membutuhkan API browser.
