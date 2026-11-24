# Incremental Static Regeneration (ISR)

## 1. Skenario Sistem Nyata

Sebuah platform e-commerce memiliki katalog produk berisi **2.000.000 (dua juta) halaman produk**. 

Jika tim menggunakan arsitektur SSG murni:
- Proses kompilasi build (`npm run build`) untuk 2 juta halaman membutuhkan waktu lebih dari 12 jam di server CI/CD.
- Jika ada satu kesalahan ketik harga pada 1 produk, seluruh situs harus di-build ulang selama 12 jam sebelum koreksi harga tersebut muncul di web!

Jika tim menggunakan arsitektur SSR murni:
- Setiap ada jutaan pengunjung yang datang saat promo Flash Sale, server web Node.js tercekik karena harus merender jutaan string HTML dan melakukan query database secara terus-menerus. Biaya server melonjak gila-gilaan.

Solusi: Menggunakan arsitektur **Incremental Static Regeneration (ISR)** (diperkenalkan oleh Next.js). Halaman disajikan secepat file statis (SSG) dari CDN cache. Namun, ketika ada data yang diperbarui di database, server akan memperbarui file statis halaman tersebut **di latar belakang secara bertahap (*per-page on demand / background revalidation*)**, tanpa perlu melakukan build ulang seluruh situs!

## 2. Apa Itu dan Buat Apa?

Incremental Static Regeneration (ISR) adalah pola arsitektur hibrida modern yang menggabungkan kecepatan kilat Static Site Generation (SSG) dengan fleksibilitas data terkini dari Server-Side Rendering (SSR).

Dengan ISR, developer dapat:
1. Menghasilkan sebagian halaman populer secara statis saat build time.
2. Menghasilkan halaman lain secara *on-demand* saat pertama kali dikunjungi pengguna (*fallback generation*).
3. Memperbarui halaman statis yang sudah ada secara otomatis di latar belakang menggunakan mekanisme **Stale-While-Revalidate (SWR)** berdasarkan durasi waktu (*time-based revalidation*) atau pemicu webhook (*on-demand revalidation*).

Fungsi utama ISR:
- **Skalabilitas Jutaan Halaman**: Memungkinkan jutaan halaman memiliki performa secepat CDN statis tanpa terbentur masalah durasi build time yang membengkak.
- **Pembaruan Data Otomatis**: Memastikan data statis di CDN tidak pernah basi selamanya tanpa perlu redeploy aplikasi.

## 3. Kenapa Harus Pakai Ini?

ISR memecahkan dilema klasik dunia rekayasa web: *"Haruskah kita mengorbankan kecepatan demi data baru (SSR), atau mengorbankan data baru demi kecepatan (SSG)?"*.

Dengan ISR, pengunjung web **selalu mendapatkan respons instan berkecepatan CDN statis (< 50 ms)**, sementara regenerasi halaman baru diserahkan sepenuhnya ke proses worker di latar belakang.

## 4. Cara Kerjanya (Diagram Alur Stale-While-Revalidate)

Visualisasi alur ISR dengan interval validasi ulang 60 detik (`revalidate: 60`):

```
Waktu: 00:00 (Halaman versi v1 tersimpan di CDN Cache)
Pengunjung A ---> GET /produk/sepatu ---> Menerima v1 dari CDN (Instan 30ms)

Waktu: 00:30 (Masih dalam jendela 60 detik)
Pengunjung B ---> GET /produk/sepatu ---> Menerima v1 dari CDN (Instan 30ms)

----------------------------------------------------------------------
Waktu: 01:05 (Batas 60 detik SUDAH KEDALUWARSA / Stale!)
Pengunjung C ---> GET /produk/sepatu ---> Tetap Menerima v1 (Instan, tidak menunggu!)
                         |
                         | (Server mendeteksi cache basi, memicu regenerasi di background)
                         v
             +----------------------------------------+
             |         WORKER BACKGROUND (Node.js)     |
             | 1. Query database harga terbaru        |
             | 2. Render ulang komponen ke HTML baru  |
             | 3. Gantikan file v1 dengan v2 di CDN!  |
             +----------------------------------------+

Waktu: 01:10 (Halaman v2 berhasil diperbarui)
Pengunjung D ---> GET /produk/sepatu ---> Menerima v2 dengan data harga terbaru!
```

## 5. Cuplikan Logika Inti

### Implementasi ISR Berbasis Waktu & On-Demand di Next.js

**1. Time-Based Revalidation (Pages Router):**
```javascript
// pages/products/[id].js
export async function getStaticProps({ params }) {
  const product = await fetchProductFromDB(params.id);

  return {
    props: { product },
    // Revalidasi halaman ini di latar belakang maksimal sekali setiap 60 detik
    revalidate: 60,
  };
}

export async function getStaticPaths() {
  // Hanya build 100 produk terlaris saat build time untuk menghemat waktu build
  const topProducts = await getTop100ProductIds();
  const paths = topProducts.map((id) => ({ params: { id } }));

  // fallback: 'blocking' -> jika produk ke-101 dibuka, render seperti SSR sekali,
  // lalu simpan sebagai file statis baru untuk pengunjung berikutnya!
  return { paths, fallback: "blocking" };
}
```

**2. On-Demand Revalidation via Webhook (App Router):**
```javascript
// app/api/revalidate/route.js (Dipanggil oleh CMS saat admin menekan tombol Simpan)
import { revalidatePath, revalidateTag } from "next/cache";

export async function POST(request) {
  const secret = request.headers.get("x-secret-token");
  if (secret !== process.env.CMS_WEBHOOK_SECRET) {
    return new Response("Unauthorized", { status: 401 });
  }

  const { path, tag } = await request.json();

  if (tag) {
    // Menghapus cache semua komponen dengan tag data tertentu secara presisi
    revalidateTag(tag);
  } else if (path) {
    // Menghapus cache file statis pada path URL spesifik
    revalidatePath(path);
  }

  return Response.json({ revalidated: true, now: Date.now() });
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan
- **Waktu Build Singkat**: Anda hanya perlu membangun segelintir halaman utama saat deployment; jutaan halaman sisanya di-generate secara malas (*lazy-generated on-demand*).
- **Kecepatan Setara SSG**: 99% request dilayani langsung dari cache file statis edge CDN.
- **Biaya Server Sangat Rendah**: Server hanya merender halaman saat ada perubahan data atau saat masa revalidate terlewati, bukan di setiap request pengunjung.

### Kekurangan & Risiko
- **Stale Data Sementara**: Pengunjung pertama setelah masa kedaluwarsa akan tetap melihat data lama (*stale data*) sementara background worker sedang membuat versi baru.
- **Kompleksitas Multi-Region Caching**: Jika aplikasi di-deploy di beberapa region independen tanpa storage terpusat, satu region mungkin menyajikan halaman yang lebih baru daripada region lain.

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Pakai
- E-commerce bervolume katalog besar (ratusan ribu hingga jutaan produk).
- Portal konten berskala masif yang sering diperbarui (portal berita dengan ribuan artikel, direktori lowongan kerja, review film/buku).
- Website yang ingin memangkas durasi build pipeline CI/CD tanpa kehilangan manfaat kecepatan SSG.

### Kapan Jangan Pakai
- Halaman yang menampilkan data privat atau sangat sensitif terhadap perubahan per milidetik (misal: saldo dompet digital atau portofolio saham).
- Halaman yang kontennya berubah secara radikal tergantung siapa pengguna yang sedang login (gunakan **CSR** atau **SSR** dengan cookie header).

## 8. Metrik Kunci & Observabilitas

- **Cache Status Header (`x-nextjs-cache`)**:
  - `HIT`: Request dilayani langsung dari cache statis (waktu respons ~20ms).
  - `STALE`: Versi lama disajikan, regenerasi background sedang dipicu.
  - `MISS`: Halaman belum pernah di-generate sebelumnya dan sedang di-render on-demand.
- **Revalidation Duration**: Pantau waktu yang dibutuhkan background worker untuk menyelesaikan fungsi `getStaticProps` atau `generateStaticParams`.
- **Edge CDN Latency**: Pastikan latensi TTFB tetap di bawah 100 ms untuk status `HIT`.

## 9. Studi Kasus Industri Nyata

### Penanganan Jutaan Halaman Listing Real Estate di Zillow / Trulia

- **Tantangan Rekayasa Riil**:
  Situs pencarian properti rumah memiliki lebih dari 100 juta halaman rincian rumah di seluruh negara bagian. Melakukan build SSG untuk 100 juta halaman adalah hal yang mustahil secara komputasi. Di sisi lain, menggunakan SSR murni untuk 100 juta halaman membebani database PostgreSQL secara ekstrem.
- **Implementasi Solusi Industri**:
  Arsitektur dimigrasikan ke **Incremental Static Regeneration**.
  1. Hanya halaman kota-kota besar terpopuler yang di-build saat deployment awal.
  2. Ketika ada pengguna mencari rumah di kota kecil terpencil, halaman di-generate secara on-demand saat itu juga (`fallback: 'blocking'`) dan disimpan permanen di edge CDN.
  3. Setiap kali ada perubahan status properti (misal: "Rumah Terjual"), webhook database memicu *On-Demand Path Revalidation* untuk memperbarui halaman tersebut seketika.
- **Dampak & Hasil Sistem**:
  - Waktu build deployment berkurang dari berjam-jam menjadi hanya 5 menit.
  - Beban query database turun hingga 70%.

## 10. Fakta Menarik & Gotchas Production

1. **Pengunjung Pertama Menjadi Korban Relawan (*The Sacrificial User*)**:
   Pada model time-based ISR, pengunjung yang tiba setelah masa waktu revalidasi habis tetap menerima konten lama. Pengunjung tersebut bertindak sebagai "pemicu" yang menyuruh server membuat versi baru untuk pengunjung *berikutnya*. Jika website sangat jarang dikunjungi, konten lama bisa bertahan lama di CDN sampai ada orang yang membukanya.
2. **Error di Background Tidak Merusak Halaman**:
   Salah satu fitur paling hebat dari ISR adalah **Fault Tolerance**: Jika proses regenerasi di background gagal (misal database backend tiba-tiba error/timeout), server tidak akan menampilkan halaman 500 ke pengguna! Server akan tetap menyajikan halaman statis lama yang masih sehat sampai database pulih kembali.

## 11. Rangkuman Singkat

1. **Jembatan Terbaik**: ISR menyatukan kecepatan CDN dari SSG dengan kesegaran data dari SSR.
2. **Stale-While-Revalidate**: Menyajikan konten cache instan ke pengguna sembari memperbarui HTML di latar belakang.
3. **On-Demand Revalidation**: Manfaatkan pemicuan webhook untuk memperbarui cache tepat saat data di database berubah, tanpa perlu menunggu timer kedaluwarsa.
4. **Tahan Banting**: Jika backend database down, ISR tetap menyajikan halaman statis lama tanpa memunculkan pesan error ke pengunjung.
