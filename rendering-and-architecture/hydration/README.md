# Hydration

## 1. Skenario Sistem Nyata

Pengguna membuka situs web reservasi tiket pesawat yang menggunakan Server-Side Rendering (Next.js). Halaman jadwal penerbangan muncul di layar dalam waktu 800 milidetik (FCP sangat cepat). Pengguna langsung melihat tombol "Pilih Kursi" berwarna biru cerah dan segera mengkliknya.

Namun, sesuatu yang aneh terjadi:
- Pengguna mengklik tombol tersebut 3 kali berturut-turut, tetapi **sama sekali tidak ada respons**. Panel pemilihan kursi tidak terbuka.
- Baru setelah detik ke-3, tombol tiba-tiba merespons dan membuka 3 jendela pemilihan kursi sekaligus secara kacau.
- Pengguna merasa aplikasi "rusak", "aneh", atau "lambat merespons sentuhan".

Fenomena ini dikenal di dunia industri rekayasa web sebagai **The Uncanny Valley of Hydration**. Tampilan visual HTML sudah matang dirender oleh server, tetapi JavaScript di browser belum selesai diunduh, diparsing, dan dipasangkan listener-nya ke tombol fisik tersebut.

Solusi: Memahami proses **Hydration**, meminimalkan ukuran bundle hidrasi, menerapkan arsitektur *Progressive / Selective Hydration*, atau beralih ke pola *Islands Architecture*.

## 2. Apa Itu dan Buat Apa?

Hydration (atau *Rehydration*) adalah proses di sisi client (browser) di mana pustaka UI berbasis komponen (seperti React atau Vue) membaca struktur HTML statis yang sebelumnya sudah dirender oleh server (melalui SSR atau SSG), lalu:
1. Membangun pohon Virtual DOM di memori browser.
2. Mencocokkan simpul Virtual DOM dengan simpul Real DOM yang sudah ada di layar.
3. Memasang seluruh penangan event (*event listeners* seperti `onClick`, `onChange`, `onSubmit`) dan menginisialisasi reaktivitas state.

Ibarat menyiram air ke tanaman kering (*dehydrated* HTML) agar kembali hidup dan mekar (*hydrated* interaktif).

Fungsi utama Hydration:
- Menjadikan dokumen HTML server yang semula statis dan kaku menjadi aplikasi web modern yang dinamis dan sepenuhnya interaktif.

## 3. Kenapa Harus Pakai Ini?

Ketika server mengirimkan HTML matang (SSR/SSG), browser hanya menerima string teks statis. Tombol `<button>` di HTML tidak memiliki kecerdasan logika apa pun selain tampilan visualnya.

Agar tombol tersebut dapat memicu fungsi JavaScript saat diklik, framework perlu melakukan rekonsiliasi dan "menghidupkan" pohon DOM tersebut. Tanpa proses hidrasi, aplikasi Anda hanya akan berfungsi sebagai brosur dokumen teks tanpa kemampuan interaksi modern.

## 4. Cara Kerjanya (Diagram Alur Dua Dunia Hydration)

```
FASE 1: SERVER
[Komponen React di Node.js]
             |
             v
renderToString(<App />)
             |
             v
Kirim HTML Statis Mentah:
<div id="root"><button class="btn">Pilih Kursi</button></div>
             |
             v
FASE 2: BROWSER (Saat halaman pertama kali tampil)
[Browser menampilkan tombol secara instan]
*TOMBOL BISA DILIHAT, TAPI BELUM BISA DIKLIK!* (Uncanny Valley)
             |
             | Browser mengunduh bundle.js
             v
FASE 3: EKSEKUSI HIDRASI
[hydrateRoot(root, <App />)]
             |
   1. Baca DOM yang sudah ada di layar
   2. Bangun VDOM di memori JavaScript
   3. Cocokkan: VDOM <button> == Real DOM <button> (Match!)
   4. Pasang: buttonNode.addEventListener('click', handleSelectSeat)
             |
             v
*TOMBOL SEKARANG RESMI HIDUP & INTERAKTIF!*
```

## 5. Cuplikan Logika Inti

### Entry Point Hydration di React 18 / 19

Perbedaan mendasar antara render client murni vs hidrasi server:

```javascript
// client-entry.js
import React from "react";
import { hydrateRoot } from "react-dom/client";
import App from "./App";

const domNode = document.getElementById("root");

// Alih-alih createRoot().render(), kita gunakan hydrateRoot!
// hydrateRoot menginstruksikan React: "Jangan buang HTML yang ada di layar,
// cukup sambungkan event listener dan state ke elemen tersebut!"
hydrateRoot(domNode, <App />);
```

### Jebakan Hydration Mismatch (Error Klasik Production)

Jika hasil render di server berbeda dengan hasil render di browser, browser akan memunculkan error fatal: *"Hydration failed because the initial UI does not match that was rendered on the server."*

```javascript
// CONTOH KODE RUSAK YANG MEMICU HYDRATION ERROR:
function UserHeader() {
  // Bencana: Di server jam 14:00 (UTC), di browser jam 21:00 (WIB)
  // Server merender teks "14:00", browser mengharapkan "21:00" -> MISMATCH!
  return <div>Waktu login: {new Date().toLocaleTimeString()}</div>;
}

// SOLUSI YANG BENAR:
function SafeUserHeader() {
  const [time, setTime] = useState(null);

  useEffect(() => {
    // Jalankan kode yang tergantung lingkungan client hanya setelah komponen mounted!
    setTime(new Date().toLocaleTimeString());
  }, []);

  return <div>Waktu login: {time || "Memuat waktu..."}</div>;
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan
- **Penyatuan Dua Keunggulan**: Menyatukan kecepatan tampilan awal server (FCP) dengan keluwesan navigasi aplikasi SPA di browser.
- **Konsistensi Kode Tunggal (*Isomorphic / Universal JavaScript*)**: Developer menulis komponen yang sama untuk dijalankan di server dan di client.

### Kekurangan & Biaya Tersembunyi (*The Cost of Hydration*)
- **Pekerjaan Ganda (*Double Computation*)**: Pohon komponen dieksekusi **dua kali**: sekali di server untuk menghasilkan string HTML, dan sekali lagi di browser untuk membangun pohon VDOM saat hidrasi.
- **Konsumsi Baterai dan CPU Smartphone**: Pada aplikasi masif, proses hidrasi di browser dapat memakan waktu hingga 1,5 detik CPU main-thread, merusak skor **Total Blocking Time (TBT)** dan **Interaction to Next Paint (INP)**.
- **Mengunduh Kode yang Sama Dua Kali**: Data halaman dikirimkan dua kali: sekali sebagai teks di dalam HTML, dan sekali lagi sebagai payload JSON di dalam tag `<script id="__NEXT_DATA__">` agar state hidrasi sinkron.

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Membutuhkan Full Hydration
- Halaman formulir multi-langkah interaktif, panel editor, keranjang belanja dinamis.
- Komponen dengan state lokal yang rumit dan interaksi pengguna yang instan.

### Kapan Menggunakan Partial Hydration / Zero-JS (Astro / Islands)
- Halaman yang didominasi teks bacaan (artikel blog, berita, dokumentasi, landing page produk). Tidak ada gunanya menghidrasi ribuan paragraf teks statis! Cukup hidrasi tombol interaktif kecil (misal tombol *Like* atau kolom komentar) menggunakan pola *Islands Architecture*.

## 8. Metrik Kunci & Observabilitas

- **Interaction to Next Paint (INP)**: Mengukur keterlambatan respons visual saat pengguna mengklik tombol di tengah-tengah proses hidrasi yang sedang berjalan. Target: **INP < 200 ms**.
- **Hydration Duration (React Profiler)**: Pantau waktu yang dihabiskan pada fase `commit` dan `layout effect` saat aplikasi pertama kali melakukan hidrasi.
- **Hydration Mismatch Warnings**: Pantau monitoring error frontend (Sentry/Datadog). Peringatan *Hydration Mismatch* harus disetel sebagai prioritas P1 karena memicu React membuang seluruh HTML server dan melakukan render ulang dari nol secara darurat.

## 9. Studi Kasus Industri Nyata

### Inovasi Islands Architecture pada Astro Framework

- **Tantangan Rekayasa Riil**:
  Situs web e-commerce atau berita tradisional yang dibangun dengan Next.js sering kali mengunduh ratusan kilobyte JavaScript hanya untuk menghidrasi header, footer, dan paragraf artikel yang sebenarnya tidak memiliki interaksi apa pun.
- **Implementasi Solusi Industri**:
  Arsitektur **Astro Islands (Component-Level Partial Hydration)** membalik paradigma:
  1. Secara default, seluruh halaman di-render sebagai HTML statis murni dengan **0 KB JavaScript client**.
  2. Hanya komponen yang benar-benar membutuhkan interaksi (misal: widget `ImageCarousel` atau `AddToCartButton`) yang dihidrasi secara terisolasi menggunakan direktif: `<Carousel client:visible />`. Komponen tersebut baru dihidrasi ketika pengguna menggulir layar hingga komponen tersebut terlihat.
- **Dampak & Hasil Sistem**:
  - Ukuran bundle JavaScript yang diunduh browser terpangkas hingga 90%.
  - TBT turun ke 0 milidetik dan skor performa Lighthouse melonjak ke 100.

## 10. Fakta Menarik & Gotchas Production

1. **Atribut `suppressHydrationWarning`**:
   Jika Anda memang sengaja memiliki perbedaan kecil antara server dan client (misal format tanggal atau tema dark/light mode bawaan sistem), Anda dapat menambahkan properti `suppressHydrationWarning={true}` pada tag HTML terkait di React untuk membungkam peringatan error di console.
2. **Streaming SSR dengan Selective Hydration (React 18)**:
   React 18 memperkenalkan kemampuan memecah hidrasi menggunakan `<Suspense>`. Browser dapat mulai menghidrasi komponen navbar di atas layar tanpa perlu menunggu komponen komentar di bagian bawah selesai di-render oleh server!

## 11. Rangkuman Singkat

1. **Menghidupkan HTML**: Hydration adalah proses menyambungkan Virtual DOM dan event listener JavaScript ke HTML statis hasil render server.
2. **The Uncanny Valley**: Waspadai celah waktu di mana tombol sudah tampak di layar tetapi belum bisa merespons klik pengguna.
3. **Cegah Mismatch**: Pastikan state dan data awal di server sama persis dengan state awal di browser saat komponen pertama kali dimuat.
4. **Masa Depan Parsial**: Kurangi beban hidrasi dengan mengadopsi Selective Hydration (React Suspense) atau Islands Architecture (Astro).
