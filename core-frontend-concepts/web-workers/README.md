# Web Workers

## 1. Skenario Sistem Nyata

Pada aplikasi web pengolah gambar atau editor video online (seperti Canva atau Photopea di browser), pengguna menerapkan filter efek khusus (*Gaussian Blur* atau pengenalan tepi AI) pada gambar beresolusi 4K (3840 x 2160 piksel = lebih dari 8,2 juta piksel).

Setiap piksel memiliki 4 kanal warna (Red, Green, Blue, Alpha). Algoritma filter harus melakukan iterasi matematika matriks konvolusi pada 32 juta nilai data di memori.

Jika algoritma ini dijalankan langsung di thread utama JavaScript (*main-thread*):
- Komputasi memakan waktu 3 detik penuh.
- Selama 3 detik tersebut, main-thread terkunci total (*frozen*).
- Animasi loading spinner berhenti berputar, tombol batalkan (*Cancel*) tidak dapat diklik, musik latar terputus, dan browser menampilkan peringatan *"Page Not Responding"*.

Solusi: Menggunakan **Web Workers**. Memindahkan seluruh perhitungan matematika pemrosesan piksel ke **thread latar belakang independen (*background thread*)**, sehingga antarmuka pengguna (UI) di main-thread tetap berjalan mulus pada 60 FPS tanpa jeda sedikit pun.

## 2. Apa Itu dan Buat Apa?

Web Workers adalah API standar W3C yang memungkinkan kode JavaScript dijalankan pada **thread latar belakang terpisah**, terisolasi sepenuhnya dari thread eksekusi utama aplikasi web.

Secara default, browser mengeksekusi JavaScript, manipulasi DOM, rendering, styling, dan penanganan event pengguna di **satu thread utama (main-thread)**. Web Workers mematahkan batasan single-thread ini dengan memberikan kemampuan **multithreading sejati** yang memanfaatkan multi-core CPU perangkat secara nyata.

Fungsi utama Web Workers:
- **Offloading Komputasi Berat**: Memindahkan pekerjaan berat (kriptografi, parsing CSV/JSON raksasa, kompresi data, manipulasi audio/video/gambar) keluar dari main-thread.
- **Menjamin Responsivitas UI**: Memastikan interaksi klik, ketik, dan animasi pengguna tidak pernah terganggu oleh beban CPU aplikasi.

## 3. Kenapa Harus Pakai Ini?

Bahkan dengan optimasi Event Loop terbaik (seperti memecah tugas dengan `setTimeout`), pekerjaan komputasi murni yang membutuhkan kalkulasi CPU intensif akan tetap memperlambat main-thread dan menghabiskan jatah waktu frame (16,6 ms).

Web Workers adalah satu-satunya cara di browser web standar untuk mencapai paralelisasi komputasi murni. Worker berjalan di thread OS terpisah, sehingga komputasi 100% pada Worker CPU core 2 tidak akan memengaruhi kelancaran UI yang berjalan di CPU core 1.

## 4. Cara Kerjanya (Diagram Alur Komunikasi Antar-Thread)

Web Workers berjalan di konteks global terpisah (`DedicatedWorkerGlobalScope`), tidak memiliki akses ke objek `window` atau `document` (DOM), dan berkomunikasi dengan main thread murni melalui pengiriman pesan asinkron (**`postMessage`**).

```
+------------------------------------+        +------------------------------------+
|            MAIN THREAD             |        |             WEB WORKER             |
|     (UI, DOM, Event Listeners)     |        |      (Thread Latar Belakang)       |
+------------------------------------+        +------------------------------------+
|                                    |        |                                    |
| 1. Kirim data mentah               |        |                                    |
|    worker.postMessage(imageData);  | =====> | 2. Terima di worker:               |
|                                    |        |    self.onmessage = (e) => { ... } |
|                                    |        |                                    |
| *UI TETAP RESPONSIF 60 FPS!*       |        | 3. Lakukan komputasi berat 3 detik |
| Pengguna bisa klik tombol lain     |        |    (Loop 8 juta piksel matematika) |
| Spinner tetap berputar mulus       |        |                                    |
|                                    |        | 4. Kirim hasil selesai:            |
| 5. Terima hasil:                   | <===== |    self.postMessage(resultData);   |
|    worker.onmessage = (e) => {     |        |                                    |
|      renderResultToCanvas(e.data); |        |                                    |
|    };                              |        |                                    |
+------------------------------------+        +------------------------------------+
```

## 5. Cuplikan Logika Inti

### Implementasi Worker dengan Dukungan Zero-Copy Transferable Objects

Mengirim array data besar (misal 50 MB) via `postMessage` biasa akan melakukan kloning memori (*Structured Clone Algorithm*) yang memakan waktu. Gunakan **Transferable Objects** untuk mentransfer kepemilikan memori secara instan (0 milidetik):

**1. File Worker (`image-filter.worker.js`):**
```javascript
// Berjalan di thread latar belakang (self merepresentasikan DedicatedWorkerGlobalScope)
self.onmessage = (event) => {
  const { buffer, width, height } = event.data;
  
  // Bungkus buffer ke typed array
  const pixels = new Uint8ClampedArray(buffer);

  // Lakukan manipulasi matematika piksel (Grayscale Filter)
  for (let i = 0; i < pixels.length; i += 4) {
    const avg = (pixels[i] + pixels[i + 1] + pixels[i + 2]) / 3;
    pixels[i] = avg;     // R
    pixels[i + 1] = avg; // G
    pixels[i + 2] = avg; // B
    // Alpha pixels[i + 3] tidak diubah
  }

  // Transfer balik kepemilikan buffer ke main-thread (Zero-Copy)
  self.postMessage({ buffer: pixels.buffer }, [pixels.buffer]);
};
```

**2. File Utama (`main.js`):**
```javascript
// Inisialisasi Web Worker
const worker = new Worker(new URL('./image-filter.worker.js', import.meta.url), {
  type: 'module'
});

function applyFilterAsync(imageArrayBuffer, width, height) {
  return new Promise((resolve) => {
    worker.onmessage = (event) => {
      resolve(event.data.buffer);
    };

    // Kirim buffer dan serahkan kepemilikan memori via Transferable List (array kedua)
    worker.postMessage({ buffer: imageArrayBuffer, width, height }, [imageArrayBuffer]);
    
    // imageArrayBuffer sekarang bernilai kosong (neutered) di main thread demi performa!
  });
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan
- **Paralelisasi Sejati**: Memanfaatkan seluruh inti fisik prosesor modern pada laptop atau smartphone.
- **Ketahanan UI Tanpa Hambatan**: Kode macet atau *infinite loop* di dalam Worker tidak akan membuat halaman browser membeku.

### Kekurangan & Batasan
- **NOL Akses ke DOM**: Worker tidak dapat membaca `document.getElementById` atau memanipulasi elemen UI secara langsung.
- **Overhead Serialisasi Pesan**: Mengirim objek JavaScript biasa yang kompleks melalui `postMessage` memerlukan proses duplikasi memori (*deep cloning*), yang dapat menimbulkan jeda milidetik jika data berukuran puluhan megabyte.
- **Memori Tambahan**: Setiap instance Worker membuat proses thread baru dengan alokasi memori tersendiri (sekitar 2 - 5 MB per worker).

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Pakai
- Pengolahan media grafis (Canvas pixel manipulation, WebAssembly audio synthesizer, video transcoding via ffmpeg.wasm).
- Pemrosesan data tabular besar di browser (parsing CSV 500.000 baris, enkripsi/dekripsi file PDF besar di sisi client).
- Simulasi fisika atau kalkulasi AI/ML lokal (misal: visualisasi graf rumit di D3.js atau library TensorFlow.js).

### Kapan Jangan Pakai
- Tugas-tugas ringan (seperti kalkulasi matematika sederhana < 10ms) — overhead pembuatan worker dan pengiriman pesan justru akan lebih lambat daripada mengeksekusinya langsung di main-thread.
- Operasi yang membutuhkan manipulasi DOM secara intensif dan instan.

## 8. Metrik Kunci & Observabilitas

- **Total Blocking Time (TBT)**: Pindahkan tugas berat ke Web Worker dan perhatikan penurunan TBT dari ribuan milidetik menjadi **< 50 ms**.
- **CPU Profiler di Chrome DevTools**:
  - Di panel Performance atau Memory, Chrome menampilkan thread terpisah dengan label nama Worker Anda (misal: `DedicatedWorker Thread`).
  - Anda dapat memantau grafik beban CPU Worker secara terisolasi tanpa mengacaukan timeline Main Thread.

## 9. Studi Kasus Industri Nyata

### Akselerasi Aplikasi Desain Kolaboratif Figma di Browser

- **Tantangan Rekayasa Riil**:
  Figma adalah aplikasi desain grafis vektor profesional yang berjalan 100% di peramban web. Sebuah dokumen desain dapat memuat ratusan frame dan 50.000 elemen vektor rumit. Jika parsing berkas `.fig`, komputasi kurva Bézier, atau ekspor PNG dilakukan di main-thread, desainer tidak akan bisa menggeser kanvas (*pan and zoom*) dengan lancar.
- **Implementasi Solusi Industri**:
  Figma memisahkan arsitektur menjadi dua domain:
  1. Main-thread hanya bertugas merender grafik menggunakan WebGL/WebGPU dan menangani event mouse/keyboard.
  2. Seluruh pemrosesan file, serialisasi data dokumen yang besar, dan kompresi ekspor dialihkan ke **Web Workers** yang dikompilasi dari kode C++ menggunakan WebAssembly (Wasm).
- **Dampak & Hasil Sistem**:
  - Kanvas desain mempertahankan frame rate 60 FPS stabil bahkan saat dokumen berukuran gigabyte sedang diunduh dan diparsing di latar belakang.

## 10. Fakta Menarik & Gotchas Production

1. **Memori yang Dikosongkan (*Neutered ArrayBuffer*)**:
   Saat Anda mentransfer `ArrayBuffer` menggunakan daftar transferabel (`postMessage(data, [data.buffer])`), memori tersebut langsung dipindahkan kepemilikannya ke Worker. Di main thread, `data.byteLength` seketika berubah menjadi `0`. Mengakses data tersebut kembali di main thread akan menghasilkan `undefined` atau error!
2. **Keluarga Worker di Browser**:
   - **Dedicated Worker**: Digunakan oleh 1 tab saja (yang paling umum dibahas di sini).
   - **Shared Worker**: Dapat diakses dan digunakan bersama oleh beberapa tab browser berbeda dari domain yang sama.
   - **Service Worker**: Worker khusus bertindak sebagai proxy jaringan untuk caching offline (PWA) dan push notification.

## 11. Rangkuman Singkat

1. **Multithreading Browser**: Web Workers menghadirkan eksekusi JavaScript multi-thread sejati di latar belakang.
2. **Lindungi Main-Thread**: Gunakan untuk komputasi berat agar UI tetap responsif 60 FPS dan terbebas dari Long Tasks.
3. **Komunikasi Pesan**: Berkomunikasi melalui `postMessage` dan event listener `message`.
4. **Gunakan Transferable**: Manfaatkan `Transferable Objects` untuk memindahkan data biner masif tanpa overhead duplikasi memori.
