# Debouncing

## 1. Skenario Sistem Nyata

Pada kotak pencarian (*search bar*) aplikasi e-commerce dengan fitur autocomplete, pengguna mengetik kata pencarian: `"laptop gaming asus"`. Kata tersebut terdiri dari 18 karakter dan diketik dalam rentang waktu sekitar 1,5 detik.

Developer memasang event listener `input` biasa yang langsung menembak API backend setiap kali ada karakter baru:

```javascript
// Kode bermasalah: Memanggil API di setiap ketikan huruf
searchInput.addEventListener("input", async (event) => {
  const query = event.target.value;
  const results = await fetchSearchResultsFromAPI(query);
  renderResults(results);
});
```

Dampak pada sistem:
- **Trafik Meledak Sia-sia**: Backend menerima 18 request HTTP bertubi-tubi dari 1 pengguna dalam 1,5 detik (`"l"`, `"la"`, `"lap"`, ..., `"laptop gaming asus"`). Jika ada 10.000 pengguna aktif yang sedang mengetik bersamaan, backend dibanjiri 180.000 request seketika.
- **Race Condition UI**: Karena latensi jaringan seluler tidak stabil, request untuk `"lap"` (huruf ke-3) bisa saja selesai lebih lambat daripada request untuk `"laptop"` (huruf ke-6). Akibatnya, UI sempat menampilkan hasil pencarian lengkap, tetapi 200ms kemudian tertimpa oleh hasil pencarian usang `"lap"`. Hasil pencarian menjadi keliru dan berkedip (*flickering*).

Solusi: Menggunakan teknik **Debouncing**. Menunda pemanggilan API sampai pengguna **berhenti mengetik** selama durasi tertentu (misal 350 milidetik).

## 2. Apa Itu dan Buat Apa?

Debouncing adalah teknik optimasi pemrograman yang memastikan bahwa suatu fungsi yang berpotensi membebani sistem hanya akan dieksekusi **setelah melewati jeda waktu tertentu sejak terakhir kali fungsi tersebut dipicu**.

Jika event baru datang sebelum batas waktu jeda (delay) habis, timer penghitung mundur akan dibatalkan (*reset*) dan dihitung ulang dari nol.

Fungsi utama Debouncing:
- **Meredam Frekuensi Eksekusi**: Menahan pemanggilan fungsi berulang yang tidak perlu.
- **Mencegah Banjir Request API**: Menghemat bandwidth client dan kuota request backend.
- **Mencegah Layout Thrashing**: Menghindari kalkulasi ulang dimensi elemen saat ukuran jendela browser diubah (*window resize*).

## 3. Kenapa Harus Pakai Ini?

Event input manusia (seperti mengetik di keyboard atau mengubah ukuran jendela browser) menghasilkan puluhan hingga ratusan event dalam waktu sangat singkat. 

Mengeksekusi operasi berat (seperti request HTTP ke server atau kalkulasi geometri DOM) pada setiap milidetik event tersebut adalah pemborosan sumber daya komputasi. Debouncing menjamin aplikasi hanya mengeksekusi aksi final saat pengguna sudah selesai melakukan aksinya.

## 4. Cara Kerjanya (Diagram Alur Timer Reset)

Berikut visualisasi bagaimana timer debounce membatalkan panggilan lama dan hanya mengeksekusi panggilan terakhir setelah pengguna berhenti mengetik:

```
Ketik:        [ l ]      [ a ]        [ p ]
Waktu (ms):    0ms       150ms        280ms
                |          |            |
                v          v            v
Timer:       Mulai 300ms  Reset timer  Reset timer
             (Batal!)     (Batal!)     (Jalan terus tanpa gangguan...)
                                            |
                                            | +300ms tidak ada ketikan baru
                                            v
                                       Waktu 580ms:
                          +------------------------------------+
                          |    EKSEKUSI API: "lap"             |
                          | (Hanya 1 request yang terkirim!)   |
                          +------------------------------------+
```

## 5. Cuplikan Logika Inti

### Implementasi Debounce Murni (TypeScript / JavaScript Modern)

Fungsi tingkat tinggi (*Higher-Order Function*) dengan dukungan pembatalan (*cancel*):

```javascript
function debounce(func, delay = 300) {
  let timerId = null;

  function debounced(...args) {
    // 1. Jika ada timer yang sedang berjalan, batalkan!
    if (timerId !== null) {
      clearTimeout(timerId);
    }

    // 2. Pasang timer baru
    timerId = setTimeout(() => {
      func.apply(this, args);
      timerId = null;
    }, delay);
  }

  // 3. Kemampuan untuk membatalkan timer secara manual (misal saat komponen unmount)
  debounced.cancel = () => {
    if (timerId !== null) {
      clearTimeout(timerId);
      timerId = null;
    }
  };

  return debounced;
}

// Cara Penggunaan:
const handleSearch = debounce((query) => {
  fetchSearchResultsFromAPI(query);
}, 350);

searchInput.addEventListener("input", (e) => handleSearch(e.target.value));
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan
- **Efisiensi Trafik Jaringan**: Memangkas hingga 90% panggilan API pencarian yang redundan.
- **Bebas Race Condition Sederhana**: Mengurangi kemungkinan response lama menimpa response baru karena request yang dikirimkan jauh lebih sedikit.
- **CPU Idle**: Menjaga CPU main-thread tetap dingin saat event terjadi beruntun.

### Kekurangan
- **Latensi Buatan (*Perceived Lag*)**: Menambahkan penundaan buatan (misal 300ms) sebelum aksi dieksekusi. Jika delay disetel terlalu lama (misal 1 detik), pengguna merasa aplikasi lambat atau tidak responsif.
- **Bukan untuk Feedback Instan**: Tidak cocok untuk aksi yang membutuhkan respons visual seketika (seperti animasi tombol saat ditekan atau drag elemen).

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Pakai
- Input pencarian teks (Autocomplete / Typeahead search).
- Pengiriman formulir otomatis (*Auto-save draft* pada editor dokumen atau formulir panjang).
- Event `resize` pada jendela browser untuk menghitung ulang tata letak grid kanvas.
- Validasi input form secara real-time ke backend (misal: mengecek ketersediaan username).

### Kapan Jangan Pakai
- Event scrolling di mana posisi elemen harus diperbarui secara berkala dan konsisten (gunakan **Throttling**).
- Tombol aksi sekali klik (seperti tombol "Bayar Sekarang"). Untuk tombol aksi, gunakan *disable state* atau pembatalan instan, bukan debounce bertimer lama.

## 8. Metrik Kunci & Observabilitas

- **Network Requests Count (DevTools Network Tab)**: Amati jumlah request HTTP saat mengetik 20 karakter di search bar. Tanpa debounce: ~20 request. Dengan debounce: tepat 1 request.
- **Main Thread CPU Usage**: Memantau penurunan aktivitas scripting pada tab Performance saat resize jendela browser dilakukan secara agresif.
- **Interaction to Next Paint (INP)**: Memastikan debounce tidak menunda render teks di dalam input box itu sendiri. (Catatan: teks di input harus tetap muncul instan, hanya pemanggilan API yang di-debounce).

## 9. Studi Kasus Industri Nyata

### Optimasi Sistem Autosave pada Notion Web Editor

- **Tantangan Rekayasa Riil**:
  Pengguna menulis ribuan kata dokumen di editor Notion. Jika setiap karakter yang diketik langsung memicu request sinkronisasi ke server dokumen via WebSocket / HTTP POST, jutaan koneksi aktif akan membanjiri klaster database dokumen Notion, menyebabkan biaya server meledak dan konflik sinkronisasi teks.
- **Implementasi Solusi Industri**:
  Notion menerapkan arsitektur *Debounced Local Mutation Buffer*. Saat pengguna mengetik, perubahan langsung disimpan secara instan ke database lokal browser (IndexedDB). Setelah pengguna berhenti mengetik selama jeda debounce (misal 1.000 ms), kumpulan mutasi data lokal tersebut dikemas (*batched*) dan dikirimkan sekaligus ke server dalam 1 request patch tunggal.
- **Dampak & Hasil Sistem**:
  - Beban request write ke backend berkurang hingga 85%.
  - Pengguna tetap merasakan penulisan teks tanpa lag (karena penyimpanan lokal bersifat instan).

## 10. Fakta Menarik & Gotchas Production

1. **Jebakan Re-creation di Komponen React**:
   Jika Anda membuat fungsi debounce langsung di dalam tubuh fungsi komponen React:
   ```javascript
   // JEBAKAN FATAL DI REACT:
   function SearchComponent() {
     // Fungsi debounce di-recreate SETIAP KALI re-render!
     // Timer lama hilang, efek debounce BATAL TOTAL!
     const debouncedSearch = debounce(fetchData, 300);
     return <input onChange={(e) => debouncedSearch(e.target.value)} />;
   }
   ```
   *Solusi*: Bungkus dengan `useCallback`, `useMemo`, atau simpan instance debounce di dalam `useRef`.
2. **Leading vs Trailing Debounce**:
   Debounce standar mengeksekusi fungsi di *akhir* jeda (*trailing edge*). Namun, ada varian *leading edge* (immediate debounce) yang langsung mengeksekusi fungsi pada klik PERTAMA, lalu mengabaikan klik berikutnya sampai jeda waktu hening terlewati (sangat berguna untuk tombol submit anti *double-click*).

## 11. Rangkuman Singkat

1. **Tunggu Sampai Hening**: Debounce hanya mengeksekusi fungsi setelah tidak ada lagi event baru selama durasi jeda tertentu.
2. **Penyelamat Autocomplete**: Mencegah banjir puluhan request HTTP saat pengguna mengetik cepat di form pencarian.
3. **Pilih Delay yang Manusiawi**: Durasi jeda ideal untuk input teks berkisar antara **300ms hingga 500ms** (cukup cepat bagi manusia, cukup aman bagi server).
4. **Waspadai Siklus Render**: Pastikan referensi fungsi debounce stabil dan tidak dibuat ulang di setiap siklus re-render framework.
