# Event Delegation

## 1. Skenario Sistem Nyata

Pada aplikasi e-commerce katalog produk, terdapat grid tak terbatas (*infinite scrolling*) yang memuat 5.000 kartu produk. Setiap kartu produk memiliki tombol "Tambah ke Keranjang", tombol "Wishlist", dan tombol "Bagikan".

Developer junior memasang event listener secara imperatif pada setiap tombol produk baru yang dimuat:

```javascript
// Kode bermasalah: Memasang ribuan listener terpisah
function attachListenersToNewProducts(products) {
  products.forEach((product) => {
    const btn = document.getElementById(`btn-cart-${product.id}`);
    btn.addEventListener("click", () => addToCart(product.id));
  });
}
```

Dampak pada sistem:
- Ketika 5.000 produk termuat, browser terpaksa mengalokasikan **15.000 objek event listener terpisah** di dalam memori heap C++ browser.
- Konsumsi memori melonjak tajam (hingga ratusan megabyte).
- Jika ada elemen produk yang dihapus dari layar tanpa membersihkan listener (`removeEventListener`), objek listener tersebut tertinggal di memori, memicu **Memory Leak** parah yang membuat browser semakin lambat seiring berjalannya waktu.
- Setiap kali elemen baru ditambahkan secara dinamis lewat AJAX, developer harus ingat untuk memanggil fungsi penempelan listener lagi.

Solusi: Menggunakan teknik **Event Delegation**. Alih-alih memasang 15.000 listener pada masing-masing tombol, pasang **1 listener tunggal** pada kontainer induk teratas (`#product-grid`), lalu manfaatkan sifat perambatan event (*Event Bubbling*) untuk mendeteksi tombol mana yang diklik.

## 2. Apa Itu dan Buat Apa?

Event Delegation adalah pola desain di mana kita memasang satu event handler pada elemen induk (*parent element*) untuk mengelola event yang dipicu oleh anak-anaknya (*child elements*), baik elemen anak yang sudah ada saat ini maupun yang baru akan dibuat secara dinamis di masa depan.

Pola ini bekerja murni berkat mekanisme **Event Bubbling**: ketika pengguna mengklik sebuah elemen anak (misal `<button>`), event tersebut tidak berhenti di tombol itu saja, melainkan merambat naik ke atas pohon DOM melewati elemen induknya (`<div>`, `<section>`, `<body>`, hingga `document`).

Fungsi utama Event Delegation:
- **Penghematan Memori Drastis**: Mengurangi ribuan objek listener di memori heap menjadi 1 listener tunggal.
- **Dukungan Elemen Dinamis**: Elemen yang ditambahkan ke DOM di kemudian hari otomatis dapat berinteraksi tanpa perlu inisialisasi ulang.
- **Pembersihan Kode**: Menyederhanakan siklus hidup pembersihan event listener (*lifecycle cleanup*).

## 3. Kenapa Harus Pakai Ini?

Dalam aplikasi web modern yang berbasis data dinamis (daftar pesan chat, tabel transaksi dengan paginasi, timeline media sosial), elemen DOM datang dan pergi setiap detik.

Memasang listener satu per satu pada elemen dinamis mengharuskan kita mengelola siklus hidup listener secara manual. Jika lupa menghapus listener saat elemen anak dibuang, browser mempertahankan referensi elemen tersebut di memori (*Detached DOM Tree*), yang menjadi biang keladi crash browser di perangkat dengan RAM terbatas.

Dengan Event Delegation, Anda hanya perlu mengelola satu listener di kontainer yang stabil.

## 4. Cara Kerjanya (Diagram Alur Event Bubbling ke Induk)

```
Pengguna Mengklik Tombol Keranjang (<button class="btn-cart">)
                         |
                         v
+---------------------------------------------------------------+
|                       FASE TARGET (Asal Event)                |
|               event.target = <button class="btn-cart">        |
+---------------------------------------------------------------+
                         |
                         |  (Event merambat naik ke atas - BUBBLING)
                         v
+---------------------------------------------------------------+
|                      <div class="product-card">               |
|                                |                              |
|                                v                              |
|                      <div class="product-row">                |
|                                |                              |
|                                v                              |
+---------------------------------------------------------------+
|                 ELEMEN INDUK DENGAN LISTENER                  |
|                   <div id="product-grid">                     |
|                                                               |
|  event.currentTarget = <div id="product-grid">                |
|  1. Periksa apakah target cocok: event.target.closest(...)    |
|  2. Ambil data: event.target.dataset.productId                |
|  3. Eksekusi fungsi addToCart(productId)!                     |
+---------------------------------------------------------------+
```

## 5. Cuplikan Logika Inti

### Implementasi Event Delegation Modern Menggunakan `closest()`

Pola paling tangguh di era web modern untuk menangani klik tombol yang mungkin memiliki ikon atau teks di dalamnya:

```javascript
const productGrid = document.getElementById("product-grid");

// Pasang HANYA 1 listener untuk seluruh kartu produk
productGrid.addEventListener("click", (event) => {
  // Gunakan .closest() untuk menangani kasus jika user mengklik ikon di dalam tombol
  const cartButton = event.target.closest(".btn-add-cart");
  const wishlistButton = event.target.closest(".btn-wishlist");

  if (cartButton && productGrid.contains(cartButton)) {
    const productId = cartButton.dataset.productId;
    handleAddToCart(productId);
    return;
  }

  if (wishlistButton && productGrid.contains(wishlistButton)) {
    const productId = wishlistButton.dataset.productId;
    handleToggleWishlist(productId);
    return;
  }
});

function handleAddToCart(id) {
  console.log(`Produk ${id} ditambahkan ke keranjang.`);
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan
- **Efisiensi Memori Maksimal**: Hanya 1 fungsi listener yang dialokasikan di RAM.
- **Nol Setup untuk Elemen Baru**: Ketika item baru disuntikkan via AJAX/infinite scroll, tombol langsung aktif tanpa kode tambahan.
- **Arsitektur Rapi**: Seluruh penanganan logika interaksi satu modul terpusat di satu tempat.

### Kekurangan
- **Overhead Pengecekan Target**: Setiap kali ada klik di mana pun di dalam kontainer induk, listener akan terpicu dan harus menjalankan pengecekan selektor (`matches` atau `closest`).
- **Tidak Semua Event Melakukan Bubbling**: Beberapa event browser penting tidak merambat ke atas secara default (misal: `focus`, `blur`, `mouseenter`, `mouseleave`).
- **Risiko `stopPropagation()`**: Jika ada elemen anak yang memanggil `event.stopPropagation()`, event akan dicegat di tengah jalan dan tidak akan pernah mencapai elemen induk.

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Pakai
- Menangani daftar item yang panjang atau jumlahnya tidak pasti (List, Table, Feed, Carousel).
- Elemen yang sering dihapus dan ditambah secara dinamis (misal Todo item, notifikasi banner).
- Navigasi menu atau tab bar dengan banyak tombol seragam.

### Kapan Jangan Pakai
- Event yang tidak melakukan bubbling (`focus`, `blur`). Sebagai solusi, gunakan event versi capture-nya atau padanannya yang melakukan bubbling (`focusin`, `focusout`).
- Kontainer tingkat tinggi yang sangat besar (seperti `document.body`) untuk event dengan frekuensi tinggi seperti `mousemove` atau `scroll`, karena pengecekan selektor akan membebani CPU secara sia-sia.

## 8. Metrik Kunci & Observabilitas

- **JS Heap Memory Size**: Gunakan tab Memory di Chrome DevTools untuk mengambil *Heap Snapshot*. Bandingkan ukuran memori antara pendekatan per-elemen vs event delegation.
- **Event Listener Count**: Periksa jumlah listener aktif di console browser:
  ```javascript
  // Memeriksa jumlah listener yang menempel pada sebuah elemen di DevTools
  getEventListeners(document.getElementById("product-grid"));
  ```
- **Total Detached DOM Elements**: Cari kata kunci *"Detached"* di Heap Snapshot. Jika jumlahnya ribuan, kemungkinan besar ada listener yang menahan referensi elemen yang sudah dihapus.

## 9. Studi Kasus Industri Nyata

### Optimasi Sistem Event pada React SyntheticEvent System

- **Tantangan Rekayasa Riil**:
  Pada aplikasi React skala besar dengan ribuan komponen bersarang, jika React memasang listener DOM asli pada setiap tag `<button onClick={...}>`, memori browser akan cepat membengkak dan performa inisialisasi komponen menjadi lambat.
- **Implementasi Solusi Industri**:
  Dari versi awal hingga React 16, React menerapkan **Event Delegation Tingkat Dokumen**. React tidak menempelkan listener ke Real DOM node komponen Anda. Sebaliknya, React hanya memasang **satu listener tunggal** untuk setiap tipe event di simpul `document`. Di React 17+, delegasi ini dipindahkan dari `document` ke simpul root kontainer aplikasi (`rootNode` tempat `ReactDOM.render` dipasang) untuk mencegah konflik antar aplikasi mikro-frontend yang berjalan di halaman yang sama.
- **Dampak & Hasil Sistem**:
  - Pengurangan konsumsi memori listener hingga 90% pada aplikasi berskala masif.
  - Waktu mounting komponen UI menjadi jauh lebih singkat.

## 10. Fakta Menarik & Gotchas Production

1. **Jebakan Ikon SVG di Dalam Tombol (`event.target`)**:
   Jika di dalam `<button class="btn-cart"><svg><path>...</path></svg></button>` ada elemen SVG, saat pengguna mengklik tombol tepat di atas ikon, `event.target` bernilai `<path>` atau `<svg>`, bukan `<button>`! Jika Anda mengecek `event.target.classList.contains('btn-cart')`, logika Anda akan gagal.  
   *Solusi*: Gunakan selalu `event.target.closest('.btn-cart')` atau tambahkan CSS `pointer-events: none;` pada elemen icon di dalam tombol.
2. **Perbedaan `event.target` vs `event.currentTarget`**:
   - `event.target`: Elemen fisik paling dalam yang benar-benar diklik oleh kursor pengguna.
   - `event.currentTarget`: Elemen tempat event listener tersebut terpasang (elemen induk yang mendelegasikan).

## 11. Rangkuman Singkat

1. **Satu untuk Semua**: Pasang satu event handler pada induk untuk melayani seluruh elemen anak di dalamnya.
2. **Ditenagai Bubbling**: Pola ini memanfaatkan sifat alami event yang merambat naik dari elemen target ke induk dokumen.
3. **Pemberantas Memory Leak**: Meniadakan kebutuhan mengalokasikan ribuan objek listener terpisah pada daftar data dinamis.
4. **Gunakan `.closest()`**: Selalu gunakan metode `event.target.closest(selector)` untuk menangkap elemen target secara aman dan presisi.
