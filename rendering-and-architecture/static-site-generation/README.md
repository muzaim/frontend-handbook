# Static Site Generation (SSG)

## 1. Skenario Sistem Nyata

Sebuah perusahaan teknologi open-source memiliki situs web dokumentasi teknis (seperti dokumentasi React atau Kubernetes) yang memuat lebih dari 800 artikel panduan. Awalnya, situs tersebut menggunakan Server-Side Rendering (SSR) dinamis yang terhubung ke database CMS.

Saat rilis versi mayor baru diumumkan, situs tersebut dikunjungi oleh 500.000 developer dalam 3 jam pertama:
- Server Node.js SSR kewalahan memproses rendering string HTML berulang-ulang untuk konten panduan yang isinya sama persis bagi setiap pengunjung.
- Kapasitas CPU server menyentuh 100%, batas koneksi database terlampaui, dan situs dokumentasi tumbang (*HTTP 502 Bad Gateway* / *504 Gateway Timeout*).
- Biaya tagihan server cloud melonjak ribuan dolar hanya untuk melayani dokumen teks statis.

Solusi: Mengubah arsitektur menjadi **Static Site Generation (SSG)** (seperti Astro, Next.js SSG, Docusaurus, atau Hugo). Seluruh 800 halaman HTML di-generate sekali saja pada saat proses kompilasi/build (`npm run build`), lalu seluruh file HTML mentah tersebut disebarkan ke jaringan Content Delivery Network (CDN) global. Jutaan pengunjung dilayani langsung dari edge server CDN dalam waktu sub-100 milidetik dengan biaya server mendekati nol dolar.

## 2. Apa Itu dan Buat Apa?

Static Site Generation (SSG) adalah metode rendering di mana seluruh dokumen HTML, berkas CSS, dan JavaScript aplikasi dihasilkan secara menyeluruh **pada saat kompilasi aplikasi (*Build Time*)**, bukan pada saat ada request pengguna (*Request Time*).

Hasil keluaran dari proses build SSG hanyalah kumpulan berkas statis murni (`index.html`, `about.html`, `style.css`, dll) yang dapat langsung disimpan dan disajikan langsung melalui web server statis sederhana atau CDN (seperti Cloudflare Pages, Vercel, Netlify, atau AWS S3 + CloudFront).

Fungsi utama SSG:
- **Kecepatan TTFB Maksimal**: Berkas HTML disajikan langsung dari edge CDN terdekat dengan lokasi geografis pengguna tanpa perlu komputasi server atau query database.
- **Ketahanan Sistem Tanpa Batas (*Infinite Scalability*)**: Mampu melayani lonjakan trafik jutaan request per detik tanpa risiko server backend tumbang.
- **Keamanan Ekstrem**: Karena tidak ada server runtime atau database yang berjalan saat request dilayani, celah keamanan injeksi server (seperti SQL Injection atau Node.js RCE) tereliminasi secara alami.

## 3. Kenapa Harus Pakai Ini?

Jika konten halaman web Anda tidak berubah setiap menit (misal: halaman landing page pemasaran, artikel blog, dokumentasi API, halaman syarat & ketentuan, portofolio):
1. **Biaya Infrastruktur Terendah**: Anda tidak perlu menyewa cluster server Node.js atau instance database yang menyala 24/7.
2. **Kinerja Web Vitals Tertinggi**: SSG secara konsisten menghasilkan skor Google Lighthouse sempurna (100/100) karena ketiadaan latensi pemrosesan server saat runtime.

## 4. Cara Kerjanya (Diagram Alur Build Time vs Request Time)

```
FASE 1: BUILD TIME (Hanya dijalankan sekali di CI/CD Pipeline saat rilis)
+-------------------+      +--------------------+
|  File Markdown /  |      | Template Komponen  |
|  Headless CMS     |      | (Astro / Next.js)  |
+-------------------+      +--------------------+
          \                          /
           \                        /
            v                      v
     +------------------------------------+
     |      PROSES GENERASI (BUILD)       |
     |        npm run build               |
     +------------------------------------+
                      |
                      v
     +------------------------------------+
     |     Keluaran Berkas Statis:        |
     |  - dist/index.html                 |
     |  - dist/blog/panduan-pemula.html   |
     |  - dist/docs/api-reference.html    |
     +------------------------------------+
                      |
                      v
     +------------------------------------+
     |   Deploy ke CDN Global (Edge)      |
     +------------------------------------+

----------------------------------------------------------------------

FASE 2: REQUEST TIME (Saat jutaan pengguna membuka website)
[Pengguna di Jakarta] ---> GET /docs/api-reference ---> [Edge CDN Singapura]
                                                              |
                                           Membaca file statis dari SSD Edge
                                                              |
[Pengguna di Jakarta] <--- 200 OK (HTML Siap Tayang) <--------+
*WAKTU RESPONS HANYA 25 MILIDETIK! (NOL QUERY DATABASE)*
```

## 5. Cuplikan Logika Inti

### Pola Static Site Generation di Next.js (Pages Router / App Router)

Mendefinisikan daftar halaman statis yang harus dibuat saat build time:

```javascript
// pages/blog/[slug].js (Konsep SSG di Next.js Pages Router)

// 1. Tentukan URL apa saja yang harus di-generate saat proses build
export async function getStaticPaths() {
  const posts = await fetchAllPostSlugsFromCMS();
  
  const paths = posts.map((post) => ({
    params: { slug: post.slug }
  }));

  // fallback: false artinya URL di luar daftar ini akan menghasilkan halaman 404
  return { paths, fallback: false };
}

// 2. Ambil konten untuk setiap halaman saat proses build dijalankan
export async function getStaticProps({ params }) {
  const post = await fetchPostDataBySlug(params.slug);

  return {
    props: { post } // Data disuntikkan ke komponen saat kompilasi
  };
}

// 3. Komponen UI murni
export default function BlogPost({ post }) {
  return (
    <article>
      <h1>{post.title}</h1>
      <div dangerouslySetInnerHTML={{ __html: post.contentHtml }} />
    </article>
  );
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan
- **Kecepatan Paling Superior**: Waktu TTFB sub-milidetik saat dilayani melalui jaringan CDN modern.
- **Skalabilitas Tak Terbatas**: Mampu menahan lonjakan trafik viral tanpa perlu menambah kapasitas server.
- **Keamanan Tingkat Tinggi**: Tidak ada database atau server-side script yang dieksekusi secara aktif saat pengguna mengakses halaman.

### Kekurangan
- **Waktu Build Membengkak (*Build Time Bottleneck*)**: Jika website memiliki 50.000 artikel produk, proses build di CI/CD bisa memakan waktu berjam-jam untuk menghasilkan 50.000 file HTML.
- **Data Tidak Real-Time**: Pembaruan konten di database/CMS mengharuskan pemicuan proses build ulang (*re-deploy*) seluruh situs sebelum perubahan dapat dilihat oleh pengunjung web.
- **Kurang Cocok untuk Konten Personalisasi**: Tidak dapat menyajikan konten yang berbeda-beda untuk tiap pengguna unik (seperti nama akun pengguna yang sedang login) pada level HTML awal.

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Pakai
- Blog pribadi, portal dokumentasi pengembang, situs panduan pengetahuan (*knowledge base*).
- Halaman profil perusahaan (*company profile*), landing page kampanye pemasaran, situs portofolio.
- Halaman konten publik yang frekuensi pembaruannya hanya hitungan hari atau minggu.

### Kapan Jangan Pakai
- Dashboard aplikasi keuangan atau platform trading dengan pergerakan angka real-time per detik (gunakan **CSR**).
- Marketplace raksasa dengan ratusan ribu produk di mana stok dan harga berubah dinamis setiap detik (gunakan **SSR** atau **ISR**).

## 8. Metrik Kunci & Observabilitas

- **Time to First Byte (TTFB)**: Pada SSG yang di-hosting di CDN, TTFB umumnya konsisten berada di bawah **100 ms** di seluruh penjuru dunia.
- **Build Duration**: Pantau durasi proses build di pipeline CI/CD (GitHub Actions / GitLab CI). Jika durasi build melampaui 15 menit, pertimbangkan beralih ke strategi **ISR (Incremental Static Regeneration)**.
- **Cache Hit Ratio di CDN**: Persentase request yang berhasil dilayani langsung dari cache edge CDN. Standar sehat untuk SSG adalah **> 98%**.

## 9. Studi Kasus Industri Nyata

### Migrasi Dokumentasi Smashing Magazine ke Arsitektur JAMstack (SSG)

- **Tantangan Rekayasa Riil**:
  Smashing Magazine dulunya beroperasi menggunakan platform WordPress monolitik dinamis. Setiap kali ada artikel desain viral yang terbit, jutaan pembaca membanjiri server LAMP (Linux, Apache, MySQL, PHP), menyebabkan server database sering crash dan biaya hosting melonjak tinggi.
- **Implementasi Solusi Industri**:
  Tim Smashing Magazine memigrasikan seluruh situs ke arsitektur JAMstack menggunakan **Static Site Generation**. Konten disimpan dalam file Markdown terkelola di Git. Setiap kali ada commit artikel baru, webhook memicu generator situs statis untuk membuat HTML murni dan mendistribusikannya ke ribuan node CDN Netlify.
- **Dampak & Hasil Sistem**:
  - Biaya hosting bulanan terpangkas drastis hingga 80%.
  - Waktu muat halaman meningkat 6x lebih cepat secara global.
  - Situs tidak pernah mengalami downtime lagi akibat lonjakan pembaca.

## 10. Fakta Menarik & Gotchas Production

1. **Jebakan `Date.now()` saat Build Time**:
   Jika Anda menulis `<footer>Hak Cipta © {new Date().getFullYear()}</footer>` atau menampilkan waktu terbit artikel menggunakan `Date.now()` di komponen SSG, waktu tersebut dibekukan pada tanggal dan jam saat proses build dijalankan di server CI/CD, BUKAN waktu saat pengunjung membuka web!
2. **Kombinasi SSG + Client-Side Fetching**:
   Banyak developer junior mengira situs SSG tidak bisa menampilkan data dinamis. Kenyataannya, kita bisa membuat kerangka halaman utama secara statis (SSG), lalu menyuntikkan satu komponen interaktif di dalamnya (misal widget kotak komentar atau status login) yang mengambil data secara dinamis via `fetch()` di browser pengguna.

## 11. Rangkuman Singkat

1. **Paling Cepat & Murah**: SSG menghasilkan seluruh halaman HTML saat build time, menyajikan file statis murni lewat CDN.
2. **Nol Komputasi Runtime**: Menghilangkan kebutuhan server dinamis dan beban query database saat request pengguna masuk.
3. **Kendala Build Skala Masif**: Kurang ideal untuk situs dengan puluhan ribu halaman yang sering berubah karena waktu build menjadi sangat lama.
4. **Gunakan untuk Konten Statis**: Pilihan mutlak terbaik untuk dokumentasi, blog, dan landing page pemasaran.
