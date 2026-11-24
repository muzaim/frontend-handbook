# Tree Shaking

## 1. Skenario Sistem Nyata

Pada aplikasi formulir kalkulator pinjaman bank, developer membutuhkan sebuah fungsi utilitas sederhana untuk memformat mata uang dan memotong string teks: `capitalize('jakarta')`.

Developer menginstal pustaka utilitas populer `lodash` via NPM dan mengimpor fungsi tersebut seperti berikut:

```javascript
// Kode bermasalah: Mengimpor fungsi dari pustaka berbasis CommonJS monolitik
import { capitalize } from "lodash";

console.log(capitalize("pinjaman mikro"));
```

Ketika proses kompilasi bundler (Webpack / Vite) selesai:
- Ukuran berkas output JavaScript membengkak sebesar **72 Kilobyte (gzipped)** hanya untuk satu fungsi kapitalisasi huruf!
- Mengapa ini terjadi? Karena pustaka `lodash` lama didistribusikan dalam format **CommonJS (`require` / `module.exports`)**. 
- Bundler tidak dapat menganalisis secara statis fungsi mana saja yang tidak terpakai. Akibatnya, seluruh 300+ fungsi lodash lain (yang tidak pernah dipanggil sama sekali) ikut terangkut masuk ke dalam bundle produksi.

Solusi: Memahami cara kerja **Tree Shaking**. Menggunakan pustaka yang didistribusikan dalam format **ES Modules (ESM)** (seperti `lodash-es`), memastikan konfigurasi `"sideEffects": false` terpasang di `package.json`, dan mengimpor modul secara spesifik agar kode yang tidak terpakai (*dead code*) dicoret sepenuhnya oleh bundler.

## 2. Apa Itu dan Buat Apa?

Tree Shaking adalah istilah di dunia kompilasi bundler frontend modern (Rollup, Vite, Webpack, esbuild) untuk menyebut proses **Dead-Code Elimination (Pembersihan Kode Mati)**.

Analogi visualnya: bayangkan seluruh kode aplikasi dan dependensi Anda seperti sebuah pohon (*tree*). Kode yang benar-benar dieksekusi dan digunakan oleh aplikasi diibaratkan seperti daun-daun hijau yang masih hidup. Tree Shaking adalah tindakan "mengguncang pohon tersebut secara kencang" agar seluruh daun-daun cokelat yang sudah mati (*unused code*) berjatuhan dan disingkirkan dari bundel akhir.

Fungsi utama Tree Shaking:
- Memangkas fungsi, kelas, atau variabel dari pustaka pihak ketiga yang tidak pernah dipanggil di dalam kode aplikasi.
- Menjaga ukuran berkas bundle JavaScript tetap ramping dan cepat diunduh oleh peramban pengguna.

## 3. Kenapa Harus Pakai Ini?

Tanpa Tree Shaking, setiap kali Anda menginstal pustaka utilitas atau pustaka komponen UI (seperti Material UI, Ant Design, atau Lucide Icons):
1. Mengimpor 1 buah icon panah akan menarik 1.500 icon lainnya ke dalam bundle Anda.
2. Mengimpor 1 helper tanggal akan menarik ratusan helper zona waktu yang tidak dibutuhkan.

Tree Shaking memastikan bahwa Anda **hanya membayar apa yang benar-benar Anda gunakan (*pay only for what you use*)**.

## 4. Cara Kerjanya (Diagram Alur Static Analysis ES Modules)

Tree Shaking **HANYA bisa bekerja berkat sifat struktur statis dari ES Modules (`import` dan `export`)**:

```
KODE SUMBER APLIKASI:
// mathUtils.js (ES Module)
export function add(a, b) { return a + b; }      <--- DIGUNAKAN
export function subtract(a, b) { return a - b; } <--- TIDAK PERNAH DIPANGGIL
export function multiply(a, b) { return a * b; } <--- TIDAK PERNAH DIPANGGIL

----------------------------------------------------------------------

// main.js
import { add } from './mathUtils.js';
console.log(add(5, 10));

----------------------------------------------------------------------

PROSES TREE SHAKING (Saat npm run build dijalankan):
1. Bundler menganalisis Abstract Syntax Tree (AST) secara statis.
2. Menemukan: 'subtract' dan 'multiply' tidak memiliki referensi pemanggil.
3. Menandai kedua fungsi tersebut sebagai "Dead Code".

Keluaran Berkas bundle.min.js:
function add(a,b){return a+b}console.log(add(5,10));
*FUNGSI subtract DAN multiply DIBUANG TOTAL DARI BUNDLE!*
```

## 5. Cuplikan Logika Inti

### Perbedaan Fatal: CommonJS vs ES Modules pada Tree Shaking

**1. Format CommonJS (GAGAL Tree Shaking):**
```javascript
// mathCommonJS.js
module.exports = {
  add: (a, b) => a + b,
  subtract: (a, b) => a - b
};

// main.js
const { add } = require('./mathCommonJS');
// Bundler TIDAK BISA membuang subtract karena require() bersifat dinamis runtime!
```

**2. Format ES Modules Modern (BERHASIL Tree Shaking):**
```javascript
// mathESM.js
export const add = (a, b) => a + b;
export const subtract = (a, b) => a - b;

// main.js
import { add } from './mathESM';
// Bundler 100% membuang subtract dari bundle akhir!
```

### Konfigurasi Kritis: Properti `"sideEffects"` di `package.json`

Memberitahu bundler bahwa berkas di dalam proyek Anda murni bebas dari efek samping global sehingga aman untuk di-shake:

```json
{
  "name": "my-frontend-library",
  "version": "1.0.0",
  "type": "module",
  "sideEffects": [
    "*.css",
    "*.scss",
    "./src/polyfills.js"
  ]
}
```
*Keterangan*: Nilai array di atas memberitahu bundler: "Hanya file CSS dan polyfills yang memiliki efek samping. Seluruh file JavaScript lainnya aman untuk dibuang jika fungsi di dalamnya tidak diimpor!".

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan
- **Otomatis di Bundler Modern**: Vite, Rollup, dan Webpack 5 menjalankan tree shaking secara default saat mode production (`NODE_ENV=production`).
- **Pengurangan Ukuran Berkas Drastis**: Mengurangi ukuran dependensi pustaka besar hingga 80%.

### Kekurangan & Tantangan
- **Tergantung pada Penulis Pustaka (Library Authors)**: Jika pembuat library tidak mengekspor kodenya dalam format ESM murni atau salah mengonfigurasi `sideEffects`, tree shaking akan gagal total.
- **Waktu Kompilasi Build Lebih Lama**: Analisis pohon sintaksis abstrak (AST) menyeluruh membutuhkan daya CPU lebih saat proses build di CI/CD.

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Memastikan Tree Shaking Berjalan
- Menggunakan pustaka utilitas (pilih `lodash-es` atau `radash` alih-alih `lodash` biasa).
- Menggunakan pustaka ikon (pilih `lucide-react` yang mendukung ESM alih-alih mengimpor font ikon monolitik).
- Menggunakan pustaka animasi atau komponen UI besar.

### Kapan Kode Tidak Boleh Di-Tree Shake (*Side Effects*)
- Berkas CSS global (misal `import './global.css'`). Jika CSS di-tree shake, seluruh gaya visual halaman akan hilang!
- Modul polyfill yang memodifikasi prototype global (misal `import 'core-js/actual/structured-clone'`).

## 8. Metrik Kunci & Observabilitas

- **Bundle Visualizer Analysis**: Gunakan `rollup-plugin-visualizer` (di Vite) atau `webpack-bundle-analyzer`. Periksa apakah ada kotak modul raksasa yang masuk ke bundle padahal Anda hanya menggunakan segelintir fungsinya.
- **Production JS Artifact Size**: Amati ukuran file `.js` di folder `dist/` sebelum dan sesudah mengganti import pustaka CommonJS ke ESM.

## 9. Studi Kasus Industri Nyata

### Optimasi Pustaka Tanggal Moment.js ke Date-fns di Twitter Web

- **Tantangan Rekayasa Riil**:
  Twitter Web dulunya menggunakan pustaka manipulasi tanggal `moment.js`. Pustaka Moment.js dirancang sebelum era ES Modules populer dan secara otomatis memuat seluruh ratusan paket bahasa dunia (*locale files*) ke dalam bundle, memakan ukuran lebih dari 300 KB tanpa bisa di-tree shake.
- **Implementasi Solusi Industri**:
  Twitter memigrasikan seluruh manipulasi tanggal ke pustaka modern **`date-fns`** (atau native `Intl` API):
  1. `date-fns` dibangun dengan arsitektur fungsi murni (*pure functions*) berbasis ES Modules.
  2. Komponen yang hanya membutuhkan format tanggal cukup mengimpor: `import { format } from 'date-fns'`.
- **Dampak & Hasil Sistem**:
  - Ukuran bundle JavaScript terkait manipulasi tanggal terpangkas dari 300 KB menjadi hanya 12 KB.
  - Waktu eksekusi parsing JavaScript awal di ponsel membaik secara nyata.

## 10. Fakta Menarik & Gotchas Production

1. **Efek Samping Eksekusi Global (*Side Effect Trap*)**:
   Jika Anda menulis kode yang memodifikasi objek global di tingkat teratas berkas:
   ```javascript
   // util.js
   window.myTracker = true; // SIDE EFFECT!
   export function doSomething() {}
   ```
   Bundler akan menolak membuang berkas `util.js` meskipun fungsi `doSomething` tidak pernah diimpor, karena bundler takut membuang baris `window.myTracker = true` yang berpotensi merusak jalannya aplikasi!
2. **Ketergantungan Melingkar (*Circular Dependencies*)**:
   Jika Modul A mengimpor Modul B, dan Modul B mengimpor kembali Modul A, bundler sering kali gagal menganalisis pohon ketergantungan dan terpaksa menyertakan seluruh kode dari kedua modul tersebut.

## 11. Rangkuman Singkat

1. **Pembersih Kode Mati**: Tree shaking membuang fungsi dan modul yang tidak pernah dipanggil dari bundel akhir.
2. **Wajib ES Modules**: Hanya bekerja pada format modul `import` dan `export` statis; tidak dapat bekerja optimal pada CommonJS `require()`.
3. **Pilih Library ESM**: Selalu utamakan pustaka modern yang mendukung ESM (misal `lodash-es` daripada `lodash`).
4. **Pasang sideEffects: false**: Pasang deklarasi `sideEffects: false` pada `package.json` untuk membantu bundler bekerja maksimal.
