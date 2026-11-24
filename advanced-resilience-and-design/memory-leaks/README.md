# Memory Leaks

## 1. Skenario Sistem Nyata

Sebuah dashboard operasional logistik digunakan oleh operator gudang dari pukul 08.00 pagi hingga 17.00 sore tanpa me-refresh tab browser. Dashboard tersebut menampilkan peta pergerakan armada truk secara live dan daftar paket yang diperbarui setiap beberapa detik.

Pada jam pertama, aplikasi berjalan mulus dan responsif dengan konsumsi memori RAM browser sekitar 120MB. Namun memasuki jam kelima, laptop operator mulai menderu panas, kipas berputar kencang, dan antarmuka web mulai patah-patah (*stuttering*).

Tepat pada pukul 14.30, browser mendadak crash dengan layar tab bergambar tabung reaksi meledak dan kode error Google Chrome:
`Error code: Out of Memory (STATUS_BREAKPOINT / STATUS_STACK_OVERFLOW)`

Penyebabnya adalah komponen peta yang di-mount dan di-unmount secara berkala meninggalkan event listener `window.addEventListener('resize', ...)` dan objek layer peta di memori JavaScript tanpa pernah dibersihkan (*uncleared listener*). Setiap menit, memori RAM membengkak sebesar 15MB hingga menembus batas alokasi V8 engine (~4GB).

Insiden ini adalah contoh klasik dari **Memory Leak pada Single-Page Application (SPA)**.

## 2. Apa Itu dan Buat Apa?

Memory Leak (Kebocoran Memori) adalah kondisi di mana suatu blok memori RAM yang telah dialokasikan oleh program untuk menampung objek, data, atau elemen DOM tidak lagi digunakan oleh aplikasi, namun tidak dapat dibebaskan oleh mekanisme pembersih memori otomatis browser (*Garbage Collector*).

Di mesin JavaScript (seperti V8 di Chrome), Garbage Collector menggunakan algoritma **Mark-and-Sweep**. Objek hanya akan dibersihkan dari RAM jika objek tersebut tidak lagi memiliki referensi aktif yang dapat dijangkau dari *Root* (seperti objek global `window` atau closure aktif).

Jika ada variabel global, event listener, atau closure yang secara tidak sengaja masih memegang referensi ke objek lama, objek tersebut akan terkunci di RAM selamanya.

Memahami dan mencegah memory leak bertujuan menjaga stabilitas performa aplikasi agar tetap ringan, responsif, dan tidak mengalami crash saat digunakan dalam durasi waktu yang lama.

## 3. Kenapa Harus Pakai Ini? (Urgensi Mencegah 4 Penyebab Utama Memory Leak)

1. **Uncleared Event Listeners & Observers**:
   - Menambahkan listener pada objek global (`window.addEventListener('scroll', fn)`) atau instansiasi observer (`ResizeObserver`, `IntersectionObserver`) di dalam komponen, namun lupa memanggil `removeEventListener()` atau `disconnect()` saat komponen di-unmount.
2. **Uncleared Timers (setInterval & setTimeout)**:
   - Menjalankan interval polling data (`setInterval(poll, 3000)`) di dalam komponen tanpa menyimpan ID timer dan tanpa memanggil `clearInterval()` saat komponen ditutup. Closure di dalam interval terus berjalan di background memegang state komponen lama.
3. **Detached DOM Nodes**:
   - Menyimpan referensi elemen HTML di variabel JavaScript (`let btn = document.getElementById('my-btn')`), lalu menghapus elemen tersebut dari layar via `parent.removeChild(btn)`. Elemen memang hilang dari UI, namun tetap hidup di RAM karena variabel `btn` masih memegangnya (*detached from DOM tree*).
4. **Unbounded Global State & Caches**:
   - Menyimpan data riwayat pencarian atau log ke dalam array global tanpa batas maksimal alokasi (*eviction policy*) atau tanpa menggunakan struktur data `WeakMap`/`WeakSet`.

## 4. Cara Kerjanya

Berikut adalah diagram perbandingan antara siklus hidup alokasi memori yang bersih vs terjadinya kebocoran memori (leak):

```
--- SIKLUS NORMAL (BERSIH) ---
Komponen Mount -> Alokasi Objek di Heap RAM (10MB)
       |
Komponen Unmount -> Bersihkan Listener / Hapus Referensi
       |
Garbage Collector Berjalan (Mark-and-Sweep)
Objek tidak memiliki rantai referensi ke GC Root -> MEMORI DIBEBASKAN (0MB)


--- SIKLUS TERJADINYA MEMORY LEAK ---
Komponen Mount -> Alokasi Objek Besar (10MB)
       |
Komponen mendaftarkan: window.addEventListener('resize', handleResize)
       |
Komponen Unmount -> Developer LUPA memanggil removeEventListener!
       |
Garbage Collector Berjalan
Objek window masih memegang handleResize -> handleResize memegang scope Komponen
       |
GC MENGANGGAP OBJEK MASIH DIGUNAKAN! -> MEMORI TIDAK BISA DIHAPUS
       |
Komponen di-mount dan di-unmount 100 kali berturut-turut:
Total RAM terkunci membengkak: 10MB x 100 = 1.000MB (1 GB!) -> BROWSER CRASH
```

## 5. Cuplikan Logika Inti

### Kode Bocor vs Kode Bersih di React Hook (`useEffect`)

```tsx
// 1. CONTOH KODE BOCOR (Memory Leak Trap)
import { useEffect, useState } from 'react';

export function LeakyChatWidget() {
  const [messages, setMessages] = useState<string[]>([]);

  useEffect(() => {
    // BAHAYA: Timer interval berjalan selamanya di background!
    setInterval(() => {
      console.log('Polling pesan baru...');
    }, 2000);

    // BAHAYA: Event listener global tidak pernah dicabut!
    const handleKey = (e: KeyboardEvent) => console.log(e.key);
    window.addEventListener('keydown', handleKey);

    // LUPA RETURN CLEANUP FUNCTION!
  }, []);

  return <div>Chat Widget</div>;
}

// 2. CONTOH KODE AMAN & BERSIH (Proper Cleanup Function)
export function SafeChatWidget() {
  const [messages, setMessages] = useState<string[]>([]);

  useEffect(() => {
    // 1. Simpan identifier timer
    const timerId = setInterval(() => {
      console.log('Polling pesan baru...');
    }, 2000);

    // 2. Buat fungsi handler
    const handleKey = (e: KeyboardEvent) => console.log(e.key);
    window.addEventListener('keydown', handleKey);

    // 3. WAJIB RETURN CLEANUP FUNCTION!
    // Dieksekusi otomatis oleh React saat komponen di-unmount dari DOM
    return () => {
      clearInterval(timerId);
      window.removeEventListener('keydown', handleKey);
      console.log('Seluruh listener dan timer dibersihkan dengan aman.');
    };
  }, []);

  return <div>Chat Widget Aman</div>;
}
```

### Menggunakan `WeakMap` untuk Cache Tanpa Kebocoran Memori

Jika Anda membutuhkan cache objek, gunakan `WeakMap` agar Garbage Collector bebas menghapus entri jika objek referensinya sudah tidak digunakan:

```typescript
// src/utils/cache.ts
// WeakMap memegang referensi kunci secara "lemah" (weak reference)
const metadataCache = new WeakMap<HTMLElement, { clickCount: number }>();

export function trackElement(el: HTMLElement) {
  metadataCache.set(el, { clickCount: 1 });
  // Ketika elemen `el` dihapus dari DOM dan tidak ada variabel lain yang memegangnya,
  // Garbage Collector akan OTOMATIS membersihkan data cache dari RAM!
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

| Strategi Mitigasi | Kelebihan | Kekurangan / Risiko |
| :--- | :--- | :--- |
| Disiplin Cleanup di `useEffect` / Lifecycle | Mencegah kebocoran memori paling umum secara instan tanpa pustaka luar | Menuntut ketelitian developer; mudah terlewat jika tidak dikawal aturan ESLint |
| Penggunaan Struktur Data `WeakMap` / `WeakRef` | Memungkinkan Garbage Collector bekerja alami tanpa perlu pembersihan manual | Kunci WeakMap harus berupa objek (tidak bisa string/number) dan tidak bisa di-iterasi |
| Audit Rutin Chrome Heap Snapshot | Mampu menemukan objek bocor hingga ke nama fungsi dan alur alokasinya | Memerlukan keterampilan analisis grafik memori yang cukup tinggi |

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai (Kapan Harus Waspada)

### Kapan Harus Sangat Waspada terhadap Memory Leak
- Aplikasi berdurasi sesi panjang (*long-lived tabs*): dashboard analitik live, portal perpesanan chat (Slack/WhatsApp web), terminal trading saham, CMS editor dokumen.
- Komponen yang sering dibuka-tutup (*frequently toggled*): Modal dialog besar, tab panel navigasi, tooltip dinamis, dropdown select berisi ribuan item.
- Visualisasi canvas, grafik grafik data (Chart.js / D3 / Three.js), dan pemutar video/audio.

### Kapan Risiko Kebocoran Relatif Rendah
- Website statis atau dokumen berita sederhana (*Multi-Page Application*) di mana setiap kali berpindah halaman, browser me-reload seluruh proses dan mereset RAM secara otomatis.

## 8. Metrik Kunci & Observabilitas (Diagnosa DevTools)

1. **Chrome DevTools Memory Panel - Heap Snapshot Comparison**:
   - Ambil Snapshot 1 saat halaman pertama kali dibuka.
   - Lakukan interaksi (misal buka dan tutup modal 10 kali berturut-turut).
   - Jalankan Garbage Collection paksa (klik ikon tempat sampah di DevTools).
   - Ambil Snapshot 2 dan bandingkan (*Comparison view*). Jika jumlah instansi komponen atau elemen `Detached HTMLDivElement` terus bertambah, Anda memiliki memory leak aktif.
2. **Performance Monitor Tab**: Buka menu *More tools -> Performance monitor* di Chrome DevTools. Pantau grafik `JS heap size` dan `DOM Nodes`. Jika grafiknya membentuk garis tangga naik terus menerus tanpa pernah turun (*sawtooth failure*), itu adalah sinyal kebocoran.
3. **Continuous RUM Memory Telemetry**: Pantau metrik `performance.memory.usedJSHeapSize` (pada browser Chromium) untuk mendeteksi user yang mengalami alokasi RAM abnormal di production.

## 9. Studi Kasus Industri Nyata

Sebuah platform monitoring server cloud memiliki fitur grafik live CPU usage yang me-render grafik canvas setiap 1 detik. Operator NOC membiarkan dashboard terbuka di monitor dinding selama berhari-hari.

Setiap minggu, PC monitor dinding tersebut selalu mengalami crash browser.

Investigasi DevTools:
1. Rekayasa pengembang mengambil Heap Snapshot dan menemukan lebih dari 2.500.000 objek bertipe `CanvasRenderingContext2D` dan callback listener WebSocket yang menumpuk di memori.
2. Ditemukan bahwa library grafik membuat instance buffer baru setiap kali koneksi WebSocket terputus dan melakukan rekoneksi otomatis, namun buffer instance lama tidak pernah di-destroy.
3. Solusi: Menambahkan method `chart.destroy()` dan pembatalan listener socket pada siklus reconnect.
4. Hasilnya, konsumsi RAM dashboard stabil di angka 85MB secara konstan meskipun dibiarkan berjalan nonstop selama 30 hari berturut-turut.

## 10. Fakta Menarik & Gotchas Production

- **Bahaya Konsol Log di Production**: Menjalankan `console.log(hugeDataObject)` di production mencegah objek tersebut dibersihkan oleh Garbage Collector selama tab browser DevTools dibuka, karena console browser memegang referensi aktif ke objek untuk kebutuhan inspeksi.
- **Detached Windows (Popups)**: Jika Anda membuka popup menggunakan `const popup = window.open(...)`, pastikan untuk menyetel `popup = null` setelah ditutup. Menahan referensi window popup yang sudah ditutup akan mengunci seluruh isi dokumen popup tersebut di RAM.
- **Closures yang Menjebak**: Variabel di dalam scope luar yang tidak digunakan oleh fungsi anak tetap bisa terjebak di memori jika fungsi anak lain di scope yang sama memegang closure tersebut.

## 11. Rangkuman Singkat

Memory Leak di frontend terjadi saat objek yang sudah tidak terpakai gagal dibebaskan oleh Garbage Collector akibat referensi yang masih tertahan (event listener, timer, detached DOM). Cegah kebocoran dengan selalu menulis fungsi cleanup pada setiap efek asinkron dan gunakan fitur Heap Snapshot di Chrome DevTools untuk melacak objek yang bocor secara akurat.
