# Browser Rendering Process

## 1. Skenario Sistem Nyata

Pada sebuah portal e-commerce, developer membuat animasi kartu produk yang membesar (*zoom in*) ketika di-hover oleh kursor mouse:

```css
/* Kode bermasalah: Memanipulasi properti geometri secara berkala */
.product-card {
  transition: width 0.3s ease, height 0.3s ease, top 0.3s ease;
}
.product-card:hover {
  width: 320px;
  height: 420px;
  top: -10px;
}
```

Dampak pada sistem:
- Mengubah properti `width`, `height`, dan `top` memaksa browser menjalankan seluruh pipa rendering: **Recalculate Style -> Layout (Reflow) -> Paint -> Composite** untuk setiap frame (60 kali per detik).
- Karena layout menghitung ulang posisi dan ukuran seluruh elemen lain di sekeliling kartu produk di halaman, CPU main-thread terbebani hingga 100%.
- Animasi patah-patah (*dropped frames* / FPS turun ke 20 FPS). Di perangkat smartphone kelas menengah ke bawah, halaman terasa membeku (*freeze*) selama 200 milidetik saat pengguna menggulir atau menyentuh layar.

Solusi: Mengubah animasi menggunakan properti **Transform** (`transform: scale(1.05) translateY(-10px)`). Transform dilewatkan langsung ke GPU melalui tahap **Composite**, melewati tahap Layout dan Paint sama sekali, menghasilkan animasi 60 FPS mulus mentega tanpa membebani CPU main-thread.

## 2. Apa Itu dan Buat Apa?

Browser Rendering Process (Pipa Rendering Kritis / *Critical Rendering Path*) adalah serangkaian tahapan yang dilalui browser untuk mengubah berkas teks mentah (HTML, CSS, JavaScript) menjadi piksel-piksel berwarna di layar monitor pengguna.

Pipa ini terdiri dari 5 fase utama:
1. **DOM Construction**: Parsing HTML menjadi Document Object Model.
2. **CSSOM Construction**: Parsing CSS menjadi CSS Object Model.
3. **Render Tree**: Menggabungkan DOM dan CSSOM untuk menentukan node apa saja yang benar-benar tampak di layar.
4. **Layout (Reflow)**: Menghitung geometri presisi (koordinat X, Y, lebar, tinggi) dari setiap elemen yang tampak.
5. **Paint & Composite**: Menggambar piksel elemen ke berbagai layer (Paint) lalu menyatukan layer-layer tersebut di GPU (Composite).

Fungsi utama memahami proses ini: Menulis kode CSS dan JavaScript yang selaras dengan arsitektur hardware browser (*hardware-accelerated rendering*), bukan melawannya.

## 3. Kenapa Harus Pakai Ini?

Layar modern umumnya me-refresh gambar sebanyak 60 kali per detik (60 Hz) atau 120 kali per detik (120 Hz). Artinya, untuk menghasilkan tampilan animasi atau scrolling yang mulus tanpa patah, browser hanya memiliki waktu **16,6 milidetik** (pada 60 Hz) untuk menyelesaikan seluruh pekerjaan JavaScript, styling, layout, dan painting dalam satu frame.

Jika JavaScript memblokir main-thread atau kode CSS memaksa browser melakukan layout berulang kali yang melebihi batas 16,6 ms, frame tersebut akan terbuang (*dropped frame*), menyebabkan efek tersentak (*jank*) yang langsung merusak kenyamanan pengguna.

## 4. Cara Kerjanya (Diagram Alur Pipa Rendering)

```
+----------------+      +----------------+
|   HTML Bytes   |      |   CSS Bytes    |
+----------------+      +----------------+
        |                       |
        v                       v
+----------------+      +----------------+
|    DOM Tree    |      |  CSSOM Tree    |
| (Struktur Teks)|      |  (Gaya Visual) |
+----------------+      +----------------+
        \                       /
         \                     /
          v                   v
+----------------------------------------+
|              RENDER TREE               |
|  (Hanya elemen yang tampak di layar;   |
|   mengabaikan <head>, display: none)   |
+----------------------------------------+
                    |
                    v
+----------------------------------------+
|            LAYOUT (REFLOW)             |
|  (Menghitung geometri elemen:          |
|   koordinat X, Y, width, height)       |
|  *Operasi CPU Main Thread yang mahal!  |
+----------------------------------------+
                    |
                    v
+----------------------------------------+
|                 PAINT                  |
|  (Mengisi warna, border, teks, shadow, |
|   membagi elemen ke dalam layers)      |
+----------------------------------------+
                    |
                    v
+----------------------------------------+
|               COMPOSITE                |
|  (Layer dikirim ke GPU;                |
|   GPU menyatukan layer ke layar fisik) |
|  *Sangat cepat, 60-120 FPS!            |
+----------------------------------------+
```

## 5. Cuplikan Logika Inti

### Membandingkan Biaya Mutasi CSS: Reflow vs Repaint vs Composite

Tiga jalur mutasi visual di browser memiliki biaya komputasi yang sangat berbeda:

```css
/* Jalur 1: layout (PALING MAHAL - Memicu Layout -> Paint -> Composite) */
/* Mengubah dimensi atau posisi relatif terhadap dokumen */
.expensive-box {
  left: 100px;
  width: 200px;
  margin-top: 10px;
}

/* Jalur 2: Repaint (SEDANG - Melewati Layout, memicu Paint -> Composite) */
/* Mengubah tampilan visual tanpa memengaruhi posisi elemen lain */
.medium-box {
  background-color: #2563eb;
  color: #ffffff;
  outline: 2px solid red;
}

/* Jalur 3: Composite (PALING MURAH & CEPAT - Hanya Composite via GPU) */
/* Dikelola langsung oleh GPU compositor thread tanpa mengganggu CPU */
.fast-gpu-box {
  transform: translate3d(100px, 0, 0) scale(1.1);
  opacity: 0.9;
  will-change: transform, opacity; /* Memberitahu browser untuk membuat layer GPU baru */
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan Optimasi Jalur GPU (Composite Only)
- **Bebas Macet (*Jank-Free*)**: Animasi transform dan opacity berjalan di Compositor Thread independen, tetap mulus bahkan saat Main Thread CPU sedang sibuk mengeksekusi JavaScript.
- **Konsumsi Baterai Efisien**: GPU dirancang secara spesifik untuk memanipulasi matriks transformasi grafis secara paralel.

### Kekurangan & Risiko Over-Optimization
- **Konsumsi VRAM Melonjak (*GPU Memory Overhead*)**: Menggunakan `will-change: transform` pada terlalu banyak elemen (misal 500 elemen sekaligus) memaksa browser membuat layer grafis terpisah untuk setiap elemen. Hal ini menghabiskan memori GPU dan dapat memicu crash pada perangkat low-end.
- **Font Blurring**: Elemen yang dipromosikan ke layer GPU kadang mengalami masalah rendering subpixel font, membuat teks tampak sedikit buram (*blurry*).

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Mengoptimalkan ke Composite Thread
- Semua jenis animasi UI interaktif (Drawer slide-in, modal pop-up, tooltip fade-in, carousel/slider scroll).
- Halaman dengan interaksi berfrekuensi tinggi (misal: visualisasi drag-and-drop, gesture pinch-to-zoom).

### Kapan Jangan Memaksa Layer GPU
- Jangan pasang `will-change` secara statis di stylesheet global untuk semua elemen. Pasang `will-change` hanya saat elemen akan dianimasikan (misal via JavaScript sebelum animasi mulai) dan copot kembali setelah animasi selesai.
- Elemen teks statis yang tidak pernah berpindah posisi.

## 8. Metrik Kunci & Observabilitas

- **Cumulative Layout Shift (CLS)**: Mengukur pergeseran tata letak visual tak terduga selama pemuatan halaman. Standar Google Web Vitals: **CLS < 0.1**.
  *Penyebab utama CLS*: Gambar atau iframe tanpa atribut `width` dan `height` eksplisit, atau font web yang berganti (*FOUT/FOIT*).
- **Chrome DevTools Performance Panel**:
  - Buka *More Tools -> Rendering* -> Centang **Layout Shift Regions** (area yang bergeser akan diarsir biru).
  - Centang **Paint Flashing** (elemen yang di-repaint akan berkedip hijau). Jika seluruh halaman berkedip hijau saat scroll, ada masalah repaint yang parah.
- **Frame Rate (FPS Meter)**: Memantau stabilitas FPS saat animasi berlangsung (target: konsisten 60 FPS).

## 9. Studi Kasus Industri Nyata

### Perbaikan CLS pada Halaman Artikel Media Online Besar

- **Tantangan Rekayasa Riil**:
  Sebuah portal berita online memiliki skor CLS buruk sebesar **0.58** (kategori merah/buruk). Pengguna sering kali salah mengklik tautan karena ketika artikel sedang dibaca, banner iklan di bagian atas tiba-tiba termuat, mendorong seluruh teks artikel ke bawah sejauh 250 piksel.
- **Implementasi Solusi Industri**:
  1. Mengalokasikan ruang kosong (*aspect-ratio container*) pada kontainer iklan CSS sebelum iklan termuat: `aspect-ratio: 16 / 9; min-height: 250px;`.
  2. Memberikan atribut dimensi eksplisit pada semua tag gambar: `<img width="800" height="450" ... />`.
  3. Menggunakan properti `font-display: optional` pada Web Fonts untuk mencegah teks bergeser saat custom font selesai diunduh.
- **Dampak & Hasil Sistem**:
  - Skor CLS anjlok drastis dari **0.58 menjadi 0.02** (kategori hijau/sempurna).
  - Bounce rate pembaca menurun sebesar 18% dan komplain salah klik iklan berkurang hingga 80%.

## 10. Fakta Menarik & Gotchas Production

1. **`display: none` vs `visibility: hidden` pada Render Tree**:
   Elemen dengan `display: none` tidak akan dimasukkan ke dalam Render Tree sama sekali (tidak memakan ruang dan tidak di-paint). Sedangkan elemen dengan `visibility: hidden` tetap dimasukkan ke dalam Render Tree dan diproses pada tahap Layout (memakan ruang kosong di layar), tetapi dilewati pada tahap Paint.
2. **CSS Memblokir Rendering (*Render-Blocking*)**:
   Secara default, browser tidak akan merender konten apa pun ke layar sampai seluruh berkas CSS eksternal selesai diunduh dan diparsing menjadi CSSOM. Browser melakukan hal ini secara sengaja untuk mencegah fenomena *Flash of Unstyled Content (FOUC)*.

## 11. Rangkuman Singkat

1. **Pipa Bertahap**: Proses rendering berjalan melalui fase DOM + CSSOM -> Render Tree -> Layout (Reflow) -> Paint -> Composite.
2. **Hierarki Biaya**: Layout adalah operasi termahal (mengubah geometri), Paint adalah operasi visual (warna/shadow), dan Composite adalah operasi tercepat (dikelola oleh GPU).
3. **Gunakan Transform & Opacity**: Untuk animasi performa tinggi 60 FPS, batasi mutasi visual hanya pada properti `transform` dan `opacity`.
4. **Cegah CLS**: Selalu tetapkan aspek rasio atau dimensi eksplisit pada media dinamis untuk mencegah pergeseran layout mendadak.
