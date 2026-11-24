# Event Loop

## 1. Skenario Sistem Nyata

Pada aplikasi kasir (Point of Sale / POS) berbasis web di jaringan supermarket, aplikasi bertugas mencetak struk belanja dan menghitung ringkasan pajak untuk 1.000 barang belanjaan.

Developer menulis kalkulasi diskon dan manipulasi data secara sinkron di main thread:

```javascript
// Kode bermasalah: Menghitung 100.000 kalkulasi berat secara sinkron
function calculateDiscountAndTax(cartItems) {
  const startTime = Date.now();
  // Simulasi komputasi berat yang memakan waktu 2 detik
  while (Date.now() - startTime < 2000) {
    // Memproses data intensif CPU
  }
  updateReceiptUI();
}
```

Dampak pada sistem:
- Karena JavaScript di browser bersifat **Single-Threaded**, eksekusi loop `while` selama 2 detik tersebut memonopoli Call Stack.
- Akibatnya, Event Loop tidak dapat memproses antrean klik tombol, input barcode scanner, animasi loading spinner, maupun proses rendering layar.
- Layar browser membeku total (*unresponsive*), kursor mouse berputar (*spinning wheel*), dan browser memunculkan dialog peringatan sistem operasi: *"Page Unresponsive: Do you want to wait or kill the page?"*.

Solusi: Memahami cara kerja **Event Loop**, memecah pekerjaan komputasi panjang menjadi potongan-potongan asinkron (*chunking*) menggunakan `setTimeout` / `scheduler.yield()`, atau memindahkannya ke latar belakang menggunakan **Web Workers**.

## 2. Apa Itu dan Buat Apa?

Event Loop adalah mekanisme internal dalam lingkungan runtime JavaScript (seperti V8 di Chrome atau SpiderMonkey di Firefox) yang bertugas mengoordinasikan eksekusi kode, pengumpulan dan pemrosesan event, serta eksekusi sub-tugas dalam antrean (*queues*).

JavaScript memiliki satu **Call Stack** (eksekusi satu hal pada satu waktu). Namun, berkat Event Loop dan Web APIs browser, JavaScript dapat melakukan operasi asinkron non-blocking (seperti fetch HTTP, timer, event klik pengguna).

Komponen utama ekosistem Event Loop:
- **Call Stack**: Tempat frame fungsi yang sedang dieksekusi ditumpuk.
- **Web APIs**: Lingkungan multi-threaded browser yang menangani timer (`setTimeout`), request jaringan (`fetch`), dan DOM events.
- **Microtask Queue**: Antrean dengan prioritas tinggi untuk callback Promise (`then/catch/finally`), `queueMicrotask`, dan `MutationObserver`.
- **Macrotask Queue (Task Queue)**: Antrean tugas reguler untuk callback `setTimeout`, `setInterval`, `setImmediate`, dan I/O event.

## 3. Kenapa Harus Pakai Ini?

Memahami Event Loop adalah pembeda mendasar antara developer junior dan senior. Tanpa pemahaman ini:
1. Anda tidak dapat memprediksi urutan eksekusi kode asinkron (misal urutan console log antara `setTimeout`, `Promise.resolve()`, dan kode sinkron).
2. Anda akan sering membuat aplikasi macet (*freezing*) karena menaruh komputasi berat di Call Stack.
3. Anda tidak dapat mendiagnosis *race condition* pada pemanggilan data API di frontend.

## 4. Cara Kerjanya (Diagram Alur Prioritas Event Loop)

Aturan emas Event Loop:
> **Selesaikan seluruh Call Stack sampai kosong -> Habiskan seluruh isi Microtask Queue sampai tuntas -> Jalankan Render Update (jika perlu) -> Ambil SATU Macrotask terdepan -> Ulangi siklus.**

```
+---------------------------------------------------------------+
|                         CALL STACK                            |
|             (Mengeksekusi kode sinkron saat ini)              |
+---------------------------------------------------------------+
                               |
               Ketika Call Stack benar-benar KOSONG
                               v
+---------------------------------------------------------------+
|                    MICROTASK QUEUE (Prioritas 1)              |
|  - Promise.then / catch / finally                             |
|  - queueMicrotask()                                           |
|  - MutationObserver                                           |
|  *DIEKSEKUSI SEMUANYA HINGGA BENAR-BENAR KOSONG!              |
+---------------------------------------------------------------+
                               |
                               v
+---------------------------------------------------------------+
|                     RENDER OPPORTUNITY                        |
|  - requestAnimationFrame()                                    |
|  - Recalculate Style -> Layout -> Paint                       |
|  *Dijalankan browser jika tiba jadwal frame (misal 16.6ms)    |
+---------------------------------------------------------------+
                               |
                               v
+---------------------------------------------------------------+
|                     MACROTASK QUEUE (Prioritas 2)             |
|  - setTimeout / setInterval callbacks                         |
|  - DOM Event callbacks (click, keypress, input)               |
|  - Network I/O callbacks                                      |
|  *HANYA DIAMBIL 1 TUGAS TERDEPAN, LALU KEMBALI KE ATAS!       |
+---------------------------------------------------------------+
```

## 5. Cuplikan Logika Inti

### Memecah Tugas Berat (*Time-Slicing*) Menggunakan Macrotask

Agar UI tetap responsif (60 FPS) saat memproses 100.000 data array di main-thread:

```javascript
// Memproses array besar tanpa memblokir input pengguna
function processLargeDataChunked(items, onProgress, onComplete) {
  let index = 0;
  const CHUNK_SIZE = 500; // Jumlah item per potongan tugas

  function processChunk() {
    const chunkEnd = Math.min(index + CHUNK_SIZE, items.length);

    // Proses sebagian kecil data
    for (; index < chunkEnd; index++) {
      heavyCalculation(items[index]);
    }

    onProgress((index / items.length) * 100);

    // Jika masih ada data tersisa, jadwalkan potongan berikutnya di Macrotask Queue
    if (index < items.length) {
      // Memberi kesempatan kepada browser untuk merender UI dan merespons klik
      setTimeout(processChunk, 0);
    } else {
      onComplete();
    }
  }

  // Mulai eksekusi potongan pertama
  processChunk();
}

function heavyCalculation(item) {
  // Simulasi komputasi
  return Math.sqrt(item) * 42;
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan Arsitektur Event Loop
- **Bebas Masalah Concurrency Deadlock**: Karena eksekusi kode berada di satu thread utama, tidak ada risiko *race condition* di tingkat memori seperti pada thread locking / mutex di bahasa multi-threaded (C++/Java/Go).
- **Model Mental Sederhana**: Model pemrograman berbasis event dan non-blocking I/O membuat penulisan interaksi antarmuka pengguna menjadi intuitif.

### Kekurangan
- **Rentan Pemblokiran Total (*Single Point of Failure*)**: Satu fungsi sinkron yang berjalan terlalu lama (misal loop berat atau enkripsi kriptografi) akan melumpuhkan seluruh antarmuka aplikasi.
- **Microtask Starvation**: Jika ada fungsi yang menjadwalkan Microtask baru secara terus-menerus tanpa henti (misal recursive Promise chain), browser tidak akan pernah mencapai Macrotask Queue atau Render phase. Halaman akan hang total!

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Memanfaatkan Microtask (`queueMicrotask` / `Promise`)
- Ketika Anda ingin menjalankan kode tepat setelah fungsi saat ini selesai, tetapi **sebelum** browser melakukan rendering tampilan atau menangani event klik baru.
- Menjamin status data sudah konsisten sebelum event berikutnya diproses.

### Kapan Menggunakan Macrotask (`setTimeout(fn, 0)` atau `requestIdleCallback`)
- Memecah tugas berat (*chunking*) agar main thread memiliki jeda untuk menggambar frame visual.
- Menunda pekerjaan yang prioritasnya rendah (misal: mengirim analitik pelacakan klik pengguna).

## 8. Metrik Kunci & Observabilitas

- **Long Tasks**: Tugas apa pun di main thread yang memakan waktu **> 50 milidetik**. Tugas ini diidentifikasi oleh Chrome DevTools Performance tab dengan segitiga merah di pojok atas bar tugas.
- **Total Blocking Time (TBT)**: Akumulasi waktu dari seluruh Long Task antara First Contentful Paint (FCP) dan Time to Interactive (TTI). Target sehat: **TBT < 200 ms**.
- **Interaction to Next Paint (INP)**: Jika event listener klik tertahan di Macrotask queue karena ada long task yang sedang berjalan, skor INP akan memburuk (> 500 ms = interaksi macet).

## 9. Studi Kasus Industri Nyata

### Optimasi Input Responsiveness pada Google Search

- **Tantangan Rekayasa Riil**:
  Saat pengguna mengetik di kotak pencarian Google, sistem harus melakukan beberapa hal: menyorot teks ketik, merender daftar rekomendasi autocomplete, dan menghitung relevansi lokal. Pada perangkat Android berspesifikasi rendah, eksekusi gabungan ini memicu Long Task sebesar 180 ms, membuat kursor mengetik terasa tertinggal di belakang jari pengguna.
- **Implementasi Solusi Industri**:
  Tim Google memperkenalkan dan mengadopsi API `scheduler.yield()` (dan fallback `scheduler.postTask()`). Alih-alih menjalankan pencarian rekomendasi dalam satu siklus stack tunggal, mereka membagi pekerjaan menjadi prioritas:
  1. *User-Blocking Task*: Memperbarui teks input di layar (prioritas tertinggi).
  2. *Yield ke Event Loop*: Membiarkan browser merender frame baru.
  3. *User-Visible Task*: Memproses dan menampilkan dropdown autocomplete.
- **Dampak & Hasil Sistem**:
  - Skor INP pada perangkat low-end membaik sebesar 40%.
  - Kursor input tidak pernah lagi mengalami frame drop saat mengetik cepat.

## 10. Fakta Menarik & Gotchas Production

1. **Jebakan Urutan Eksekusi (Tebak Output!)**:
   ```javascript
   console.log("1");
   setTimeout(() => console.log("2"), 0);
   Promise.resolve().then(() => console.log("3"));
   console.log("4");
   ```
   *Urutan output*: `1 -> 4 -> 3 -> 2`.
   *Alasan*: `1` dan `4` berjalan sinkron di Call Stack. Callback Promise (`3`) masuk ke **Microtask Queue**, yang dieksekusi langsung begitu Call Stack kosong, mendahului callback `setTimeout` (`2`) yang berada di **Macrotask Queue**.
2. **`setTimeout(fn, 0)` Tidak Benar-benar 0 Milidetik**:
   Sesuai spesifikasi HTML5 standar, browser memberlakukan batas waktu minimal (*minimum clamping timeout*) sebesar **4 milidetik** setelah 5 nesting level pemanggilan timer berturut-turut.

## 11. Rangkuman Singkat

1. **Satu Call Stack**: JavaScript mengeksekusi kode secara sinkron pada satu thread utama.
2. **Kasta Antrean**: Microtask (Promise) selalu diselesaikan terlebih dahulu hingga kosong sebelum browser melayani satu Macrotask (setTimeout).
3. **Hindari Long Tasks**: Komputasi di atas 50 milidetik memblokir rendering dan interaksi pengguna.
4. **Strategi Time-Slicing**: Pecah komputasi masif menjadi potongan-potongan asinkron atau alihkan komputasi murni ke Web Workers.
