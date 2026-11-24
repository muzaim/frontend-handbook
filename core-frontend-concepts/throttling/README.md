# Throttling

## 1. Skenario Sistem Nyata

Pada aplikasi media sosial dengan fitur *Infinite Scroll* (seperti Instagram atau TikTok Web), aplikasi perlu mendeteksi kapan posisi scroll pengguna sudah mendekati bagian bawah halaman (misal sisa 300px dari dasar) untuk segera mengunduh 20 postingan berikutnya:

```javascript
// Kode bermasalah: Memeriksa posisi scroll pada setiap event scroll browser
window.addEventListener("scroll", () => {
  const scrollHeight = document.documentElement.scrollHeight;
  const scrollTop = window.scrollY;
  const clientHeight = window.innerHeight;

  // Operasi pembacaan geometri DOM yang memicu Forced Reflow
  if (scrollTop + clientHeight >= scrollHeight - 300) {
    loadNextPage();
  }
});
```

Dampak pada sistem:
- Saat pengguna menggeser mouse wheel atau menyentuh layar sentuh smartphone dengan cepat, browser memicu event `scroll` sebanyak **60 hingga 120 kali per detik**!
- Mengakses `scrollHeight` dan `scrollTop` di setiap event memaksa browser menghitung ulang geometri halaman (*Layout Thrashing*).
- Main-thread tersedot 100% hanya untuk mengecek angka scroll. Animasi video macet, scrolling terasa berat dan tersendat (*jank*), dan fungsi `loadNextPage()` terpanggil berulang kali sebelum request pertama selesai.

Solusi: Menggunakan teknik **Throttling**. Membatasi eksekusi pengecekan posisi scroll maksimal hanya **1 kali setiap 200 milidetik**, tidak peduli berapa ratus kali event scroll dipicu oleh browser.

## 2. Apa Itu dan Buat Apa?

Throttling adalah teknik optimasi pemrograman yang memastikan suatu fungsi dieksekusi **maksimal satu kali dalam jendela rentang waktu tertentu (interval teratur)**, tidak peduli seberapa sering event pemicunya terjadi.

Jika Debouncing mengumpulkan rentetan event dan hanya mengeksekusi *setelah* pengguna diam, maka Throttling menjamin bahwa fungsi akan dieksekusi secara *berkala dan stabil* sepanjang aksi pengguna berlangsung terus-menerus.

Perbedaan fundamental:
- **Debouncing**: "Tunggu sampai pengguna selesai diam selama 300ms, baru jalankan sekali."
- **Throttling**: "Jalankan fungsi ini sekarang, lalu abaikan panggilan berikutnya selama 200ms. Setelah 200ms lewat, jalankan lagi jika event masih berlanjut."

## 3. Kenapa Harus Pakai Ini?

Event-event berbasis gerakan berkelanjutan di browser—seperti `scroll`, `mousemove`, `touchmove`, dan pemantauan posisi kursor mouse pada game web—dihasilkan dengan frekuensi yang sangat tinggi.

Mengeksekusi logika berat pada setiap mikro-detik event tersebut adalah tindakan yang tidak perlu karena mata manusia dan refresh rate layar fisik hanya sanggup menampilkan perubahan pada rentang 16,6 ms (60 Hz) atau 8,3 ms (120 Hz). Throttling menyelaraskan pemrosesan logika aplikasi dengan batas kemampuan persepsi manusia dan layar.

## 4. Cara Kerjanya (Diagram Alur Interval Berirama)

Visualisasi throttling dengan batas interval 200ms saat scrolling terus-menerus:

```
Event Scroll:  * * * * * * * * * * * * * * * * * * * * * * * * *
Waktu (ms):    0ms       100ms     200ms     300ms     400ms
                |                   |                   |
Eksekusi:     RUN!                RUN!                RUN!
              (Panggilan 1)       (Panggilan 2)       (Panggilan 3)
              
Keterangan:   Event di antara 0-200ms diabaikan sepenuhnya oleh Throttler!
```

## 5. Cuplikan Logika Inti

### Implementasi Throttling Berbasis Timestamp (JavaScript Modern)

Pola paling ringan dan efisien menggunakan perbandingan waktu (*timestamp delta*):

```javascript
function throttle(func, interval = 200) {
  let lastExecutionTime = 0;
  let timerId = null;

  return function throttled(...args) {
    const currentTime = Date.now();
    const timeRemaining = interval - (currentTime - lastExecutionTime);

    // Kasus 1: Waktu interval sudah terlewati, jalankan seketika!
    if (timeRemaining <= 0) {
      if (timerId !== null) {
        clearTimeout(timerId);
        timerId = null;
      }
      lastExecutionTime = currentTime;
      func.apply(this, args);
    } 
    // Kasus 2: Masih dalam jendela interval, jadwalkan eksekusi penutup (trailing call)
    else if (timerId === null) {
      timerId = setTimeout(() => {
        lastExecutionTime = Date.now();
        timerId = null;
        func.apply(this, args);
      }, timeRemaining);
    }
  };
}

// Cara Penggunaan:
const handleScrollCheck = throttle(() => {
  console.log("Mengecek batas scroll pada:", window.scrollY);
}, 200);

window.addEventListener("scroll", handleScrollCheck);
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan
- **Umpan Balik Berkala yang Terjamin**: Menghasilkan respons sistem yang teratur sepanjang aksi berkelanjutan berlangsung (berbeda dari debounce yang diam total sampai aksi berhenti).
- **Mencegah Overload Main-Thread**: Membatasi frekuensi kalkulasi geometri atau rendering ke batas aman (misal 5 kali per detik alih-alih 120 kali per detik).

### Kekurangan
- **Kehilangan Data Presisi Mikro**: Data koordinat antara (misal titik koordinat mouse di antara jendela 200ms) dilewati. Tidak cocok untuk aplikasi tanda tangan digital presisi (*digital signature canvas*) di mana setiap lekukan goresan harus direkam utuh.
- **Pembaruan Akhir Tertinggal**: Jika implementasi tidak menyertakan *trailing edge execution*, posisi terakhir pengguna saat berhenti menggeser layar bisa terlewatkan.

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Pakai
- Pengecekan posisi scroll untuk infinite scrolling, progress bar baca artikel, atau tombol "Back to Top".
- Event `mousemove` saat membuat efek custom cursor, tooltip yang mengikuti posisi mouse, atau grafik interaktif.
- Pembatasan tombol aksi game web (misal: menahan tombol tembak hanya boleh memicu tembakan setiap 500ms).

### Kapan Jangan Pakai
- Input ketik formulir pencarian (gunakan **Debouncing**).
- Kebutuhan modern untuk mendeteksi elemen masuk viewport: Jangan gunakan scroll listener ber-throttle! Gunakan browser API modern **`IntersectionObserver`**, yang berjalan asinkron dan jauh lebih hemat baterai.

## 8. Metrik Kunci & Observabilitas

- **Frame Rate (FPS)**: Buka Chrome DevTools -> *Rendering -> Frame Rendering Stats*. Saat pengguna melakukan scrolling cepat dengan throttling, FPS harus bertahan stabil di **55 - 60 FPS**.
- **CPU Time Spent in Scripting**: Bandingkan grafik CPU di Performance panel saat scrolling selama 10 detik. Throttling yang tepat akan memangkas waktu eksekusi script dari 800 ms menjadi < 40 ms.

## 9. Studi Kasus Industri Nyata

### Optimasi Header sticky dan Parallax pada Airbnb Web

- **Tantangan Rekayasa Riil**:
  Halaman listing properti Airbnb memiliki bilah navigasi pencarian yang mengecil (*sticky header*) dan efek gambar parallax saat digulir. Awalnya, script mendengarkan event scroll secara mentah dan menghitung posisi elemen di setiap piksel scroll. Pengujian menunjukkan bahwa pada perangkat MacBook lama dan laptop Windows dengan layar 120Hz, scroll mengalami *stuttering* parah.
- **Implementasi Solusi Industri**:
  Airbnb menggabungkan dua teknik:
  1. Melakukan **Throttling** pada evaluasi status logika header.
  2. Memindahkan interpolasi visual posisi ke API bawaan browser: `window.requestAnimationFrame()` (rAF-based throttling) yang menyelaraskan kalkulasi tepat sebelum browser menggambar frame visual berikutnya.
- **Dampak & Hasil Sistem**:
  - Waktu eksekusi script scroll turun sebesar 75%.
  - Efek parallax berjalan mulus 60 FPS di seluruh perangkat yang didukung.

## 10. Fakta Menarik & Gotchas Production

1. **`requestAnimationFrame` sebagai Throttler Grafis Terbaik**:
   Jika tujuan throttling Anda murni untuk memperbarui tampilan visual di layar (bukan memanggil API), Anda tidak perlu menghitung milidetik manual dengan `Date.now()`. Gunakan `requestAnimationFrame` sebagai throttler alami:
   ```javascript
   let isTicking = false;
   window.addEventListener('scroll', () => {
     if (!isTicking) {
       window.requestAnimationFrame(() => {
         updateVisualPositions();
         isTicking = false;
       });
       isTicking = true;
     }
   });
   ```
2. **Kombinasi Throttling & Passive Event Listeners**:
   Selalu tambahkan opsi `{ passive: true }` pada listener scroll/touch: `window.addEventListener('scroll', fn, { passive: true })`. Opsi ini memberitahu browser bahwa fungsi Anda tidak akan memanggil `preventDefault()`, sehingga browser dapat melakukan scrolling instan di Compositor Thread tanpa menunggu JavaScript Anda selesai.

## 11. Rangkuman Singkat

1. **Jaga Ritme Konstan**: Throttling membatasi pemanggilan fungsi maksimal sekali dalam rentang interval waktu yang ditentukan.
2. **Scrolling & Dragging**: Pilihan utama untuk event gerakan berkelanjutan seperti scroll, resize, dan mouse tracking.
3. **Pilih Alternatif Modern**: Pertimbangkan `IntersectionObserver` untuk deteksi viewport dan `requestAnimationFrame` untuk pembaruan animasi visual.
4. **Gunakan Passive Flag**: Selalu pasangkan `{ passive: true }` pada event listener gerakan untuk menjaga kelancaran scrolling.
