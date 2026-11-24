# Event Bubbling & Capturing

## 1. Skenario Sistem Nyata

Pada aplikasi perbankan online, terdapat sebuah jendela Modal Dialog untuk konfirmasi transfer dana. Di dalam dialog tersebut terdapat formulir rincian rekening dan tombol "Kirim Dana". 

Developer ingin agar pengguna dapat menutup dialog tersebut dengan cara mengklik area abu-abu di luar modal (latar belakang / *backdrop*):

```javascript
// Kode bermasalah: Menutup modal saat backdrop diklik
const backdrop = document.getElementById("modal-backdrop");
const modalContent = document.getElementById("modal-content");

backdrop.addEventListener("click", () => {
  closeModal(); // Menutup dialog
});
```

Dampak pada sistem:
- Ketika pengguna mengklik input formulir di dalam `modal-content` untuk mengetik nomor rekening, atau mengklik tombol "Kirim", secara misterius jendela modal tiba-tiba tertutup sendiri!
- Mengapa ini terjadi? Karena event klik pada input atau tombol di dalam kotak modal **merambat naik (*bubbles up*)** ke atas hingga menyentuh `#modal-backdrop`. Listener pada backdrop mendeteksi klik tersebut dan langsung mengeksekusi `closeModal()`.
- Transaksi pengguna gagal dan pengguna frustrasi berat karena modal tertutup di tengah proses transfer.

Solusi: Memahami fase perambatan event browser (**Capturing, Target, dan Bubbling**), serta mengetahui kapan harus menggunakan `event.stopPropagation()` atau mengecek `event.target === event.currentTarget`.

## 2. Apa Itu dan Buat Apa?

Event Bubbling dan Capturing adalah dua fase dari **Event Flow (Pola Perambatan Event)** standar W3C di browser yang menentukan urutan pemanggilan event handler saat sebuah interaksi terjadi pada elemen DOM yang bersarang (*nested elements*).

Siklus hidup perambatan event di browser terdiri dari 3 fase berurutan:
1. **Capturing Phase (Trickling)**: Event bergerak turun dari puncak hierarki dokumen (`window -> document -> <html> -> <body>`) menuju ke elemen target terdalam.
2. **Target Phase**: Event tiba di elemen fisik yang menjadi sasaran interaksi pengguna.
3. **Bubbling Phase**: Setelah target selesai, event memantul dan bergerak merambat naik kembali dari elemen target ke puncak dokumen (`target -> parent -> ... -> window`).

Secara default, metode `addEventListener(type, listener)` mendengarkan event pada **Fase Bubbling**.

## 3. Kenapa Harus Pakai Ini?

Memahami kedua fase ini sangat penting untuk:
1. **Mencegah Efek Samping Tak Terduga**: Menghindari aksi elemen induk terpanggil saat elemen anak berinteraksi (seperti kasus modal backdrop atau tombol hapus di dalam kartu yang dapat diklik).
2. **Intersepsi Tingkat Tinggi (*Security & Analytics Tracking*)**: Menggunakan fase Capturing untuk mencegat interaksi sebelum sampai ke komponen anak (misal: sistem otorisasi klik atau pelacakan telemetri analitik).

## 4. Cara Kerjanya (Diagram Alur 3 Fase Event W3C)

```
                       [ Window ]
                      /          ^
         1. CAPTURE  |            |  3. BUBBLE
        (Turun ke    v            | (Naik ke atas)
          bawah)   [ Document ]   |
                     |            |
                     v            |
                   [ <html> ]     |
                     |            |
                     v            |
                   [ <body> ]     |
                     |            |
                     v            |
                [ <div id="card"> ]
                     |            ^
                     v            |
              +----------------------+
              |   2. TARGET PHASE    |
              |  [ <button id="btn">]|
              |  (Elemen yang diklik)|
              +----------------------+
```

## 5. Cuplikan Logika Inti

### Mengendalikan Alur Perambatan Event

Contoh penanganan kasus modal backdrop dan penggunaan fase capturing vs bubbling:

```javascript
const backdrop = document.getElementById("modal-backdrop");
const card = document.getElementById("modal-card");
const deleteBtn = document.getElementById("btn-delete");

// 1. Solusi Elegan untuk Backdrop Modal (Tanpa stopPropagation)
backdrop.addEventListener("click", (event) => {
  // Hanya tutup modal jika pengguna mengklik area backdrop murni,
  // BUKAN hasil rambatan klik dari dalam kotak konten modal!
  if (event.target === event.currentTarget) {
    closeModal();
  }
});

// 2. Mencegah klik tombol memicu klik pada kartu induk
deleteBtn.addEventListener("click", (event) => {
  // Menghentikan perambatan event agar tidak naik ke elemen induk (card)
  event.stopPropagation();
  executeDeleteAction();
});

// 3. Mengaktifkan Listener pada FASE CAPTURING (Parameter ketiga bernilai true)
// Berguna untuk logger atau analytics global sebelum dicegat anak
window.addEventListener(
  "click",
  (event) => {
    console.log("Analytics: Klik terdeteksi pada fase capture:", event.target);
  },
  { capture: true } // Mendengarkan saat event sedang turun ke bawah!
);
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan
- **Fleksibilitas Intersepsi**: Fase Capturing memungkinkan kita mencegat aksi sebelum dieksekusi oleh elemen target.
- **Pondasi Event Delegation**: Tanpa mekanisme Bubbling, pola Event Delegation tidak mungkin bisa berjalan.

### Kekurangan & Bahaya `stopPropagation()`
- **Mematahkan Analitik dan Telemetri Pihak Ketiga**: Jika developer secara sembarangan memanggil `event.stopPropagation()`, pustaka pihak ketiga (seperti Google Analytics, Hotjar, atau Mixpanel) yang memasang listener di `document` atau `window` tidak akan pernah menerima event tersebut.
- **Menyulitkan Pelacakan Bug (*Hidden Control Flow*)**: Logika aplikasi menjadi sulit dipahami ketika event tiba-tiba berhenti merambat di tengah hierarki komponen yang dalam.

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Menggunakan Bubbling (Default)
- 99% kebutuhan interaksi aplikasi umum (klik tombol, input form, perubahan checkbox).
- Seluruh implementasi pola Event Delegation.

### Kapan Menggunakan Capturing (`{ capture: true }`)
- Mencegat event sebelum komponen anak memprosesnya (misal: memeriksa apakah pengguna masih memiliki sesi login aktif sebelum tombol di dalam formulir dijalankan).
- Menangani event yang tidak melakukan bubbling secara andal (seperti menangani event `focus` dan `blur` pada elemen formulir yang dinamis).

### Kapan Harus Menghindari `event.stopPropagation()`
- Hindari penggunaan membabi-buta hanya untuk "mengatasi bug cepat". Sebagai gantinya, gunakan pengecekan selektor: `if (event.target !== event.currentTarget) return;`.

## 8. Metrik Kunci & Observabilitas

- **Event Propagation Latency**: Memantau waktu yang dibutuhkan event dari saat dipicu hingga seluruh handler di rantai DOM selesai dipanggil. Gunakan DevTools Performance panel:
  - Cari blok **`Event: click`**.
  - Periksa apakah durasi eksekusi handler melebihi 16 ms.
- **Console Debugging**: Browser Chrome menyediakan perintah debug praktis:
  ```javascript
  // Memonitor semua event yang terjadi pada sebuah node secara langsung di console
  monitorEvents(document.getElementById("modal-card"), "click");
  // Untuk menghentikan: unmonitorEvents(...)
  ```

## 9. Studi Kasus Industri Nyata

### Penanganan Klik Iklan Fraud pada Sistem Monetisasi Web

- **Tantangan Rekayasa Riil**:
  Sebuah platform periklanan menemukan bahwa script widget pihak ketiga yang nakal memasang tombol tak kasat mata (*clickjacking/invisible overlay*) dan memanggil `event.stopPropagation()` untuk mencegah sistem anti-fraud mendeteksi koordinat klik asli pengguna.
- **Implementasi Solusi Industri**:
  Tim keamanan periklanan memasang pendengar event klik pada simpul tertinggi (`window`) dengan memanfaatkan **Fase Capturing** (`window.addEventListener('click', auditClickCoordinate, { capture: true })`).
- **Dampak & Hasil Sistem**:
  - Karena fase Capturing dieksekusi **sebelum** event mencapai target anak di DOM tree, fungsi audit berhasil membaca dan memverifikasi keaslian koordinat klik sebelum skrip nakal sempat memanggil `event.stopPropagation()`.
  - Tingkat deteksi klik palsu (*fraudulent clicks*) meningkat hingga 99,8%.

## 10. Fakta Menarik & Gotchas Production

1. **`stopPropagation()` vs `stopImmediatePropagation()`**:
   - `event.stopPropagation()`: Menghentikan event merambat ke elemen induk berikutnya, tetapi listener LAIN pada elemen yang SAMA tetap dieksekusi.
   - `event.stopImmediatePropagation()`: Menghentikan perambatan ke elemen induk DAN langsung membatalkan eksekusi listener lain yang terpasang pada elemen yang sama saat itu juga.
2. **Event yang Tidak Melakukan Bubbling**:
   Beberapa event bawaan browser dirancang untuk tidak merambat ke atas (*bubbles: false*), di antaranya: `focus`, `blur`, `load`, `unload`, `mouseenter`, `mouseleave`, dan `scroll` (pada elemen non-document).

## 11. Rangkuman Singkat

1. **Tiga Fase Alur**: Alur event selalu melalui Capturing (turun dari window) -> Target (di elemen fisik) -> Bubbling (naik kembali ke window).
2. **Default Bubbling**: Listener standar `addEventListener` mendengarkan pada fase Bubbling.
3. **Tangani Backdrop Bersih**: Gunakan pengecekan `event.target === event.currentTarget` alih-alih merusak perambatan dengan `stopPropagation()`.
4. **Hati-hati dengan stopPropagation**: Penggunaan sembarangan dapat merusak library analytics, custom tooltip, dan dropdown global.
