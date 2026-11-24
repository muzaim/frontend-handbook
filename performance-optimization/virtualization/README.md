# Virtualization (Windowing)

## 1. Skenario Sistem Nyata

Pada aplikasi perbankan enterprise atau sistem audit log keamanan siber, pengguna membuka tabel mutasi rekening atau log server yang memuat **50.000 baris data transaksi**.

Developer junior me-render seluruh 50.000 data tersebut secara langsung ke dalam tabel HTML:

```javascript
// Kode bermasalah: Me-render 50.000 elemen DOM secara bersamaan
function LogTable({ logs }) {
  return (
    <table>
      <tbody>
        {logs.map((log) => (
          <tr key={log.id}>
            <td>{log.timestamp}</td>
            <td>{log.ipAddress}</td>
            <td>{log.message}</td>
          </tr>
        ))}
      </tbody>
    </table>
  );
}
```

Dampak pada sistem:
- Browser terpaksa membuat lebih dari **200.000 node DOM baru** (1 baris x 4 tag node x 50.000 baris) di memori RAM C++ browser.
- Konsumsi memori peramban melonjak hingga melebihi **1,2 Gigabyte**.
- Waktu render awal memakan waktu 4 detik di mana layar membeku total (*frozen*).
- Setiap kali pengguna menggulir mouse wheel (*scrolling*), peramban tercekik menghitung ulang geometri layout 200.000 elemen. Frame rate anjlok ke 4 FPS dan pada perangkat smartphone, tab browser langsung crash (*Out of Memory*).

Solusi: Menggunakan teknik **Virtualization (Windowing)** (seperti *TanStack Virtual* atau *react-window*). Layar fisik pengguna sebenarnya hanya sanggup menampilkan 20 baris dalam satu waktu. Virtualisasi hanya merender **20-30 baris yang sedang berada di dalam viewport layar**, ditambah beberapa baris buffer di atas dan bawahnya. Puluhan ribu baris lainnya hanya ada sebagai data mentah di memori JavaScript tanpa pernah dibuatkan elemen fisik DOM-nya.

## 2. Apa Itu dan Buat Apa?

Virtualization (atau *Windowing*) adalah teknik optimasi rendering daftar data panjang (*long lists/grids*) di mana peramban **hanya membuat dan merender elemen DOM yang saat ini sedang tampak di dalam jendela pandang layar pengguna (*viewport window*)**.

Elemen yang digulir keluar dari layar akan dicopot dari pohon DOM dan didaur ulang, sementara elemen baru yang memasuki layar akan disuntikkan secara dinamis.

Fungsi utama Virtualization:
- Menjaga jumlah simpul DOM (*DOM Node Count*) tetap konstan dan sangat rendah (misal stabil di bawah 40 node) terlepas dari apakah data berjumlah 1.000 item atau 1.000.000 item.
- Menjamin performa scrolling tetap berjalan mulus mentega pada 60 FPS konstan.
- Menghemat konsumsi memori RAM browser hingga 95%.

## 3. Kenapa Harus Pakai Ini?

Batas kemampuan mesin peramban web untuk menangani elemen DOM aktif secara efisien umumnya berada di kisaran **1.000 hingga 1.500 node**. Di atas angka tersebut, setiap kali ada reflow atau recalculate style, browser membutuhkan waktu komputasi eksponensial.

Virtualisasi memisahkan antara **volume data bisnis** (yang bisa berjumlah jutaan baris) dari **volume elemen visual DOM** (yang dibatasi hanya sebatas ukuran tinggi layar monitor).

## 4. Cara Kerjanya (Diagram Alur Mekanisme Kotak Siluman Phantom Spacer)

Bagaimana scrollbar tetap terasa panjang padahal elemen fisik di DOM hanya ada 5 baris:

```
+---------------------------------------------------------------+
|  KONTAINER SCROLL UTAMA (Tinggi Total Siluman: 100.000 px)    |
|  style="height: 100000px; position: relative;"                |
+---------------------------------------------------------------+
|                                                               |
|  [ Ruang Kosong Atas (Offset Y: 12.000 px) ]                  |
|                                                               |
|  ----------------- BATAS VIEWPORT LAYAR FISIK --------------- |
|  |                                                          | |
|  |  Baris 120 (Offset Y: 12.000 px)  ---> DOM AKTIF         | |
|  |  Baris 121 (Offset Y: 12.050 px)  ---> DOM AKTIF         | |
|  |  Baris 122 (Offset Y: 12.100 px)  ---> DOM AKTIF         | |
|  |  Baris 123 (Offset Y: 12.150 px)  ---> DOM AKTIF         | |
|  |                                                          | |
|  ----------------- BATAS VIEWPORT LAYAR FISIK --------------- |
|                                                               |
|  [ Ruang Kosong Bawah (Sisa 87.800 px) ]                      |
|                                                               |
+---------------------------------------------------------------+
*HANYA 4 BARIS YANG DIBUATKAN NODE DOM! SCROLLBAR TETAP PRESISI!*
```

## 5. Cuplikan Logika Inti

### Implementasi Virtualisasi List Menggunakan `@tanstack/react-virtual`

Pola standar industri modern untuk merender 10.000 baris dengan performa 60 FPS:

```tsx
import React, { useRef } from "react";
import { useVirtualizer } from "@tanstack/react-virtual";

interface LogEntry {
  id: number;
  message: string;
}

// Simulasi 10.000 data log di memori
const allLogs: LogEntry[] = Array.from({ length: 10000 }, (_, i) => ({
  id: i,
  message: `Log Transaksi #${i + 1} - Status: Berhasil diverifikasi`,
}));

export function VirtualizedLogList() {
  // 1. Referensi ke elemen pembungkus scroll fisik
  const parentRef = useRef<HTMLDivElement>(null);

  // 2. Inisialisasi virtualizer
  const rowVirtualizer = useVirtualizer({
    count: allLogs.length, // Total seluruh item (10.000)
    getScrollElement: () => parentRef.current,
    estimateSize: () => 40, // Estimasi tinggi per baris dalam piksel (40px)
    overscan: 5, // Render ekstra 5 item di atas & bawah layar sebagai buffer halus
  });

  return (
    <div
      ref={parentRef}
      className="scroll-container"
      style={{ height: "400px", overflowY: "auto", border: "1px solid #cbd5e1" }}
    >
      {/* 3. Kontainer siluman dengan tinggi total asli: 10.000 x 40px = 400.000px */}
      <div
        style={{
          height: `${rowVirtualizer.getTotalSize()}px`,
          width: "100%",
          position: "relative",
        }}
      >
        {/* 4. HANYA render item virtual yang sedang tampak di layar! */}
        {rowVirtualizer.getVirtualItems().map((virtualRow) => {
          const log = allLogs[virtualRow.index];
          return (
            <div
              key={virtualRow.key}
              style={{
                position: "absolute",
                top: 0,
                left: 0,
                width: "100%",
                height: `${virtualRow.size}px`,
                // Transform posisi vertikal Y sesuai posisi scroll
                transform: `translateY(${virtualRow.start}px)`,
              }}
              className="p-2 border-b flex items-center bg-white"
            >
              <span>{log.message}</span>
            </div>
          );
        })}
      </div>
    </div>
  );
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan
- **Skalabilitas Data Tanpa Batas**: Anda dapat memuat 100.000 item tanpa penurunan performa sedikit pun.
- **Konsumsi Memori Tetap Rendah**: Jumlah node DOM terjaga stabil di bawah 50 elemen.
- **Waktu Inisialisasi Instan**: Tidak ada jeda loading layar karena browser hanya me-render 20 baris pertama.

### Kekurangan & Tantangan
- **Fitur Pencarian Browser Rusak (*Ctrl + F broken*)**: Pengguna tidak dapat mencari kata di halaman menggunakan fungsi Find bawaan browser (`Ctrl + F`) untuk item yang sedang berada di luar layar karena elemen tersebut memang belum ada di DOM!
- **Kompleksitas Item dengan Tinggi Dinamis (*Dynamic Row Heights*)**: Jika isi teks setiap baris memiliki panjang yang bervariasi (misal status postingan twitter dengan jumlah kata yang berbeda-beda), virtualizer harus mengukur tinggi elemen nyata (*measurement cache*) secara dinamis, yang menambah kompleksitas kode.

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Memakai Virtualisasi
- Daftar data tabular atau daftar list dengan panjang lebih dari **100 hingga 200 item** (Tabel analitik, mutasi rekening bank, log aktivitas sistem).
- Grid galeri foto dengan ratusan baris.
- Kotak dropdown seleksi (*autocomplete / combobox*) yang memuat ribuan opsi data (misal daftar seluruh kode pos atau kota di dunia).

### Kapan Tidak Perlu Virtualisasi
- Daftar pendek yang jumlah itemnya terjamin kurang dari 50 item (gunakan perulangan `.map()` biasa).
- Dokumen artikel teks biasa yang mengandalkan pembacaan sekuensial penuh.

## 8. Metrik Kunci & Observabilitas

- **Total DOM Nodes Count**: Periksa di Console DevTools: `document.querySelectorAll('*').length`. Pada tabel bervirtualisasi 10.000 baris, jumlah node DOM total harus tetap berada di bawah **1.000 node**.
- **Scroll Frame Rate (FPS)**: Pantau via Chrome DevTools *Rendering -> Frame Rendering Stats*. Saat scrolling cepat dilakukan, FPS harus bertahan stabil di **55 - 60 FPS**.

## 9. Studi Kasus Industri Nyata

### Optimasi Timeline Chat pada Discord Web

- **Tantangan Rekayasa Riil**:
  Pada server Discord komunitas besar dengan jutaan pesan teks, pengguna dapat melakukan scroll ke atas membaca riwayat percakapan bertahun-tahun yang lalu. Di awal pengembangannya, Discord merender pesan teks secara naif ke DOM. Setelah pengguna melakukan scroll beberapa menit, tab browser Google Chrome kehabisan memori RAM (*OOM crash*) akibat ratusan ribu elemen chat bersarang.
- **Implementasi Solusi Industri**:
  Tim teknik Discord membangun pustaka virtualisasi internal khusus (berbasis React virtualized windowing):
  1. Hanya pesan teks di viewport layar aktif yang dipertahankan di dalam DOM.
  2. Menggunakan sistem pengukuran ketinggian dinamis (*Dynamic Height Measurement*) untuk mengakomodasi pesan dengan lampiran gambar atau emoji kustom.
  3. Mempertahankan posisi scroll jangkar (*Scroll Anchoring*) saat pesan baru masuk dari pengguna lain di bagian bawah.
- **Dampak & Hasil Sistem**:
  - Jumlah node DOM terjaga konstan di bawah 250 node terlepas dari seberapa jauh pengguna menelusuri riwayat pesan lama.
  - Memori RAM peramban tetap stabil di bawah 150 MB sepanjang hari pemakaian.

## 10. Fakta Menarik & Gotchas Production

1. **Pentingnya Buffer (Overscan)**:
   Jika Anda tidak memasang parameter `overscan` (misal `overscan: 5`), pengguna yang menggulir layar dengan sangat cepat akan melihat kilatan ruang putih kosong (*blank flashing*) selama beberapa milidetik sebelum elemen sempat dihitung posisinya oleh JavaScript. Buffer 5 item di atas dan bawah layar bertindak sebagai bantalan visual yang menyamarkan komputasi tersebut.
2. **Kunci Transform TranslateY vs Top**:
   Selalu gunakan properti `transform: translateY(...)` untuk mengatur posisi elemen virtual, **JANGAN gunakan `top: ...px`**! Mengubah properti `top` memicu proses Layout/Reflow yang mahal, sedangkan `transform` diproses secara mulus di GPU Compositor Thread.

## 11. Rangkuman Singkat

1. **Jendela Pandang Terbatas**: Virtualisasi hanya merender elemen yang sedang tampak di layar pengguna saat itu juga.
2. **Kekebalan Skala Masif**: Memungkinkan manipulasi 100.000 baris data dengan performa scroll 60 FPS yang stabil.
3. **Penyelamat Memori**: Menjaga jumlah simpul DOM tetap konstan dan mencegah crash kehabisan memori RAM peramban.
4. **Perhatikan Kompromi**: Fitur pencarian bawaan browser `Ctrl + F` tidak dapat membaca elemen yang belum di-mount ke DOM; sediakan fitur pencarian filter internal aplikasi.
