# Dynamic Imports

## 1. Skenario Sistem Nyata

Pada aplikasi formulir pendaftaran asuransi kesehatan, terdapat satu langkah opsional: "Unggah Dokumen KTP / Paspor". Jika dokumen miring, pengguna dapat menekan tombol "Edit Gambar" untuk memotong (*crop*) atau memutar foto.

Fitur pengedit foto ini mengandalkan pustaka editor gambar (*Cropper.js* dan *Canvas Filter SDK*) yang memiliki ukuran berkas total **750 Kilobyte**.

Developer junior mengimpor pustaka tersebut secara statis di bagian paling atas berkas komponen:

```javascript
// Kode bermasalah: Mengimpor modul berat secara statis di level teratas
import Cropper from "cropperjs";
import { applyFilter } from "./heavy-image-filters";
import "cropperjs/dist/cropper.css";

export function RegistrationForm() {
  // 95% pengguna hanya mengunggah foto normal tanpa pernah menekan tombol Edit Gambar!
  // Tapi seluruh 750 KB pustaka cropper tetap dipaksa diunduh di awal halaman.
}
```

Dampak pada sistem:
- Data analitik menunjukkan bahwa **95% pengguna tidak pernah membuka editor gambar**.
- Namun, 100% pengguna dipaksa menanggung beban unduhan 750 KB kode tersebut di awal, memperlambat proses pendaftaran di daerah dengan jaringan internet minim.

Solusi: Menggunakan **Dynamic Imports (`import()`)**. Memindahkan pengunduhan pustaka Cropper agar baru diunduh dari server **hanya ketika pengguna benar-benar menekan tombol "Edit Gambar"**.

## 2. Apa Itu dan Buat Apa?

Dynamic Import adalah fitur bawaan standar JavaScript modern (ES2020) berupa fungsi bawaan sintaksis `import(modulePath)` yang mengembalikan sebuah **Promise**.

Berbeda dengan *Static Import* (`import x from 'y'`) yang wajib ditaruh di baris teratas berkas dan dieksekusi secara sinkron saat pemuatan awal modul, Dynamic Import dapat dipanggil **di mana saja secara kondisional dan asinkron**: di dalam tubuh fungsi, di dalam blok `if/else`, atau di dalam event handler klik tombol.

Fungsi utama Dynamic Imports:
- **Pemuatan Berdasarkan Permintaan (*On-Demand / Lazy Loading*)**: Mengunduh modul JavaScript hanya saat dibutuhkan oleh logika eksekusi runtime.
- **Fondasi Teknis Code Splitting**: Menjadi mekanisme dasar yang digunakan oleh bundler (Webpack, Vite, Rollup) untuk memotong kode menjadi chunk terpisah secara otomatis.
- **Kompilasi Modul Kondisional**: Memuat pustaka yang berbeda tergantung pada platform browser atau bahasa lokal pengguna (misal memuat kamus bahasa Indonesia vs bahasa Jepang).

## 3. Kenapa Harus Pakai Ini?

Dalam rekayasa perangkat lunak frontend, tidak semua baris kode memiliki prioritas yang sama. Fitur seperti:
- Export data ke Excel / PDF.
- Pemutar video modal popup.
- Modul analitik pelacakan perilaku pengguna.
- Paket bahasa penerjemahan (*i18n locale files*).

Sangat jarang atau tidak langsung digunakan pada detik pertama halaman terbuka. Menggunakan Dynamic Imports memungkinkan kita menggeser biaya unduhan dari waktu muat awal (*initial load*) ke waktu interaksi pengguna (*interaction time*).

## 4. Cara Kerjanya (Diagram Alur Static vs Dynamic Import)

```
STATIC IMPORT (Sinkron di awal):
File: form.js
  |-- import { Cropper } from 'cropperjs'; (Baris 1)
  |-- import { DatePicker } from 'datepicker'; (Baris 2)
  |
  v
Browser mengunduh form.js + cropperjs + datepicker SEKALIGUS di awal!

----------------------------------------------------------------------

DYNAMIC IMPORT (Asinkron sesuai interaksi):
File: form.js (Hanya berukuran 15 KB di awal!)
  |
  +---> Pengguna mengisi nama, email, nomor HP... (Cepat & Ringan)
  |
  +---> Pengguna menekan tombol "Edit Foto KTP"
          |
          v
        Panggil: const { Cropper } = await import('cropperjs');
          |
          v
        Browser mengirim request HTTP untuk file: cropperjs.chunk.js
          |
          v
        Eksekusi cropper di kanvas!
```

## 5. Cuplikan Logika Inti

### Memuat Modul Berat Secara Dinamis Saat Tombol Diklik

Contoh implementasi ekspor tabel data ke Excel menggunakan pustaka `xlsx`:

```javascript
// Tombol Ekspor di Dashboard
const exportButton = document.getElementById("btn-export-excel");

exportButton.addEventListener("click", async () => {
  exportButton.disabled = true;
  exportButton.textContent = "Menyiapkan modul Excel...";

  try {
    // 1. Modul XLSX seberat 400 KB baru diunduh saat tombol diklik
    const { utils, writeFile } = await import("xlsx");

    exportButton.textContent = "Membuat berkas...";

    // 2. Gunakan fungsi dari modul yang berhasil diunduh
    const tableData = getTableDataFromDOM();
    const worksheet = utils.json_to_sheet(tableData);
    const workbook = utils.book_new();
    utils.book_append_sheet(workbook, worksheet, "Laporan Keuangan");

    // 3. Simpan berkas ke komputer pengguna
    writeFile(workbook, "Laporan_2026.xlsx");
  } catch (error) {
    console.error("Gagal mengunduh modul Excel:", error);
    alert("Gagal memproses ekspor. Periksa koneksi internet Anda.");
  } finally {
    exportButton.disabled = false;
    exportButton.textContent = "Ekspor ke Excel";
  }
});
```

### Memuat Berkas Bahasa Internasionalisasi (i18n) Berdasarkan Pilihan Pengguna

```javascript
async function loadLanguageDictionary(languageCode) {
  // Bundler akan memisahkan setiap file JSON ke chunk terpisah
  const dictionary = await import(`./locales/${languageCode}.json`);
  applyTranslationsToUI(dictionary.default);
}

// Dipanggil saat pengguna memilih dropdown bahasa "id" atau "en"
dropdownSelect.addEventListener("change", (e) => {
  loadLanguageDictionary(e.target.value);
});
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan
- **Standar Bahasa Resmi**: Bagian dari spesifikasi ECMAScript standar, didukung oleh semua browser modern dan bundler tanpa memerlukan plugin khusus.
- **Menghemat Jaringan & Memori**: Komponen dan library berat hanya dikompilasi di RAM peramban jika memang dipanggil.
- **Sintaksis Berbasis Promise**: Sangat elegan dipadukan dengan kata kunci `async / await` dan penanganan error `try / catch`.

### Kekurangan & Risiko
- **Keterlambatan Interaksi (*Click Delay*)**: Pengguna yang memiliki koneksi internet lambat akan merasakan jeda 1-2 detik setelah mengklik tombol sebelum modul selesai diunduh dan terbuka. (Mitigasi: Tampilkan indikator status loading spinner yang jelas pada tombol).
- **Rentan Gangguan Jaringan**: Jika koneksi terputus sesaat saat modul sedang diunduh, Promise akan berstatus *rejected*. Developer wajib menulis penanganan blok `catch` agar aplikasi tidak membisu (*silent crash*).

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Pakai
- Modul atau library yang hanya digunakan pada aksi tertentu (ekspor Excel/PDF, share ke sosial media, scan QR code webcam).
- Dialog modal atau panel pop-up yang berat (misal panel pengaturan profil pengguna yang kompleks).
- Fitur eksperimental atau A/B testing yang hanya ditampilkan kepada sebagian pengguna.

### Kapan Jangan Pakai
- Komponen atau fungsi kritis yang dibutuhkan untuk tampilan awal halaman (seperti komponen header navigasi utama, layout grid, atau tombol navigasi penting). Memuat komponen kritis dengan dynamic import akan membuat halaman berkedip (*flash of missing content*).

## 8. Metrik Kunci & Observabilitas

- **Interaction to Next Paint (INP)**: Pastikan saat dynamic import sedang berjalan, tombol langsung memberikan umpan balik visual (misal berubah menjadi spinner) dalam waktu < 200 milidetik agar tidak merusak skor INP.
- **Module Download Latency**: Pantau durasi unduhan chunk modul di tab Network DevTools. Pastikan chunk hasil dynamic import berukuran wajar (< 100 KB gzipped) agar selesai diunduh dalam < 500 ms di jaringan seluler.

## 9. Studi Kasus Industri Nyata

### Optimasi Web Editor Canva untuk Filter dan Efek Teks

- **Tantangan Rekayasa Riil**:
  Canva menyediakan ratusan efek teks, filter foto, dan generator grafik. Jika seluruh modul efek ini dibundel ke dalam kode kanvas editor utama, ukuran aplikasi akan melebihi 20 Megabyte, membuat desainer menunggu terlalu lama saat membuka proyek desain mereka.
- **Implementasi Solusi Industri**:
  Tim Canva memecah setiap efek visual ke dalam berkas chunk terpisah menggunakan **Dynamic Imports**:
  1. Kanvas utama hanya memuat mesin rendering geometri dasar.
  2. Ketika pengguna mengklik tab "Efek Teks Bayangan (Shadow)", skrip efek shadow baru diimpor secara dinamis melalui jaringan.
  3. Modul yang sudah pernah diimpor disimpan di dalam memory cache JavaScript sehingga pemanggilan kedua berlangsung instan tanpa request jaringan ulang.
- **Dampak & Hasil Sistem**:
  - Waktu inisialisasi awal editor Canva menjadi 4x lebih cepat.
  - Memori RAM browser berkurang hingga 45% untuk dokumen desain standar.

## 10. Fakta Menarik & Gotchas Production

1. **Magic Comments di Webpack**:
   Saat menggunakan bundler Webpack, Anda dapat menyuntikkan komentar khusus di dalam dynamic import untuk mengontrol nama chunk dan strategi *prefetching*:
   ```javascript
   const Chart = await import(
     /* webpackChunkName: "analitik-chart" */
     /* webpackPrefetch: true */
     './ChartComponent'
   );
   ```
   Komentar `webpackPrefetch: true` memberitahu browser untuk secara otomatis mengunduh modul tersebut di waktu senggang (*browser idle time*) setelah halaman utama selesai dimuat!
2. **Caching Alami Peramban**:
   Begitu sebuah modul berhasil diimpor melalui `import('./module.js')`, browser dan runtime JavaScript akan menyimpan referensi modul tersebut di memori internal. Pemanggilan `import('./module.js')` untuk kedua kalinya tidak akan mengirimkan request HTTP baru ke server, melainkan langsung mengembalikan instance modul yang sama.

## 11. Rangkuman Singkat

1. **Impor Sesuai Kebutuhan**: Dynamic import (`import()`) memuat modul JavaScript secara asinkron berbasis Promise pada saat runtime.
2. **Singkirkan Beban Awal**: Alihkan library berat yang jarang digunakan (seperti ekspor PDF/Excel) agar tidak membebani pemuatan awal halaman.
3. **Wajib Beri Umpan Balik**: Selalu pasang indikator loading visual pada antarmuka saat modul sedang diunduh melalui jaringan.
4. **Siapkan Penanganan Error**: Bungkus selalu dengan `try / catch` untuk menangani kemungkinan kegagalan jaringan saat mengunduh chunk modul.
