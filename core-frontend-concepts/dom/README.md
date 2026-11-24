# DOM (Document Object Model)

## 1. Skenario Sistem Nyata

Pada aplikasi dashboard log analitik enterprise, terdapat tabel pemantauan transaksi yang menerima aliran data streaming via WebSocket sebanyak 50 data per detik. Developer pemula mengimplementasikan pembaruan tabel dengan cara menghapus dan membuat ulang elemen DOM secara langsung:

```javascript
// Kode bermasalah: Memanipulasi DOM secara naif setiap kali ada pesan masuk
socket.onmessage = (event) => {
  const container = document.getElementById("log-table-body");
  container.innerHTML = ""; // Menghapus seluruh baris
  allLogs.push(JSON.parse(event.data));
  allLogs.forEach((log) => {
    container.innerHTML += `<tr><td>${log.id}</td><td>${log.message}</td></tr>`;
  });
};
```

Dampak pada sistem:
- Setiap kali `container.innerHTML += ...` dipanggil di dalam loop, browser dipaksa mem-parsing string HTML ulang, menghancurkan node DOM lama, mengalokasikan memori node baru, dan memicu kalkulasi ulang tata letak (*Forced Synchronous Layout / Layout Thrashing*).
- Frame rate browser anjlok dari 60 FPS ke 8 FPS. Halaman menjadi tersendat-sendat (*jank*), scroll macet, kipas laptop pengguna berputar kencang akibat CPU main-thread 100%, dan input ketik pengguna di form pencarian mengalami penundaan (delay) hingga 1,5 detik.

Solusi mendasar: Memahami bahwa DOM adalah struktur data pohon di memori C++ browser yang mahal untuk dimutasi. Alih-alih merombak pohon secara berulang, mutasi harus dibungkus menggunakan `DocumentFragment`, membatasi pembacaan/penulisan berulang, atau memanfaatkan teknik virtualisasi list.

## 2. Apa Itu dan Buat Apa?

DOM (Document Object Model) adalah antarmuka pemrograman (*Application Programming Interface*) berbasis objek yang merepresentasikan struktur dokumen HTML atau XML sebagai pohon simpul (*node tree*).

Ketika browser mengunduh berkas HTML mentah (berupa baris teks biner/karakter), mesin rendering browser mem-parsing karakter tersebut menjadi token, lalu mengonversinya menjadi objek-objek C++ (Node) yang saling terhubung dalam hierarki induk-anak (*parent-child*). Objek-objek ini kemudian diekspos ke lingkungan JavaScript melalui objek global `document`.

Fungsi utama DOM:
- **Jembatan Antara Dokumen dan Kode**: Memungkinkan JavaScript membaca, memanipulasi, menambah, menghapus elemen, atribut, dan gaya CSS secara dinamis setelah halaman dimuat.
- **Manajemen Event**: Menjadi fondasi interaksi pengguna (klik, ketik, scroll) yang dapat didengarkan (*event listeners*) di setiap node.

## 3. Kenapa Harus Pakai Ini?

Tanpa DOM, halaman web hanya akan menjadi dokumen teks statis seperti berkas PDF atau koran cetak. Browser tidak akan memiliki cara standar bagi bahasa skrip (JavaScript) untuk mengetahui elemen apa saja yang ada di layar, di mana posisinya, atau bagaimana cara mengubah teks tombol saat pengguna menekannya.

Namun, yang harus dipahami oleh junior developer: **DOM itu sendiri sebenarnya tidak lambat; yang lambat adalah efek samping mutasi DOM terhadap proses rendering browser** (Recalculate Style, Layout/Reflow, dan Repaint).

## 4. Cara Kerjanya (Diagram Alur)

Ketika browser menerima file HTML mentah dari jaringan, inilah tahapan terbentuknya DOM tree:

```
+---------------------------------------------------------------+
|                      Byte HTML Mentah                         |
|  3c 21 44 4f 43 54 59 50 45 20 68 74 6d 6c 3e ...            |
+---------------------------------------------------------------+
                               |
                               v
+---------------------------------------------------------------+
|             Karakter Teks (Character Encoding)                |
|  <html><head><title>Kamus</title></head><body>...             |
+---------------------------------------------------------------+
                               |
                               v
+---------------------------------------------------------------+
|               Tokens (HTML Lexer / Tokenizer)                 |
|  StartTag: html, StartTag: head, StartTag: title, etc.        |
+---------------------------------------------------------------+
                               |
                               v
+---------------------------------------------------------------+
|                Nodes (Objek C++ dengan Properti)              |
|  HTMLHtmlElement, HTMLHeadElement, HTMLTitleElement           |
+---------------------------------------------------------------+
                               |
                               v
+---------------------------------------------------------------+
|                           DOM TREE                            |
|                                                               |
|                         [document]                            |
|                             |                                 |
|                          [<html>]                             |
|                         /        \                            |
|                     [<head>]     [<body>]                     |
|                        |          /    \                      |
|                     [<title>]  [<h1>]  [<div>]                |
|                                           |                   |
|                                         [<p>]                 |
+---------------------------------------------------------------+
```

## 5. Cuplikan Logika Inti

### Optimasi Mutasi DOM dengan `DocumentFragment` dan `requestAnimationFrame`

Menghindari layout thrashing dengan mengumpulkan perubahan di memori sebelum menempelkannya ke dokumen utama:

```javascript
// Memasukkan 10.000 elemen tanpa memicu 10.000 reflow
function renderLargeListOptimized(items, targetContainer) {
  // 1. Buat kontainer memori ringan yang tidak menempel ke pohon DOM aktif
  const fragment = document.createDocumentFragment();

  items.forEach((item) => {
    const tr = document.createElement("tr");
    tr.className = "log-row";

    const tdId = document.createElement("td");
    tdId.textContent = item.id;

    const tdMsg = document.createElement("td");
    tdMsg.textContent = item.message;

    tr.appendChild(tdId);
    tr.appendChild(tdMsg);
    
    // Tempelkan baris ke fragment (hanya operasi memori, nol reflow)
    fragment.appendChild(tr);
  });

  // 2. Tempelkan seluruh fragment sekaligus ke DOM aktif dalam 1 kali reflow
  window.requestAnimationFrame(() => {
    targetContainer.innerHTML = "";
    targetContainer.appendChild(fragment);
  });
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan
- **Standar Terbuka W3C**: Didukung oleh seluruh mesin peramban (Blink, Gecko, WebKit) tanpa membutuhkan library pihak ketiga.
- **Kendali Penuh Granular**: Memberikan kontrol langsung ke level atribut, properti CSS, kelas, dan event handling tanpa abstraksi framework.
- **Ekosistem Web API yang Kaya**: Terintegrasi langsung dengan API browser modern (IntersectionObserver, ResizeObserver, MutationObserver).

### Kekurangan
- **Biaya Mutasi Mahal**: Perubahan pada dimensi atau posisi node memicu reflow dan repaint pada node di sekitarnya.
- **State Management Manual Rawan Bug**: Pada manipulasi DOM langsung, status UI (state) sering kali tersebar di atribut HTML (`data-*`), menyulitkan pelacakan sumber kebenaran data (*single source of truth*).
- **Rentan terhadap Keamanan XSS**: Menggunakan properti seperti `innerHTML` tanpa sanitasi ketat membuka celah injeksi skrip jahat (*Cross-Site Scripting*).

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Pakai
- Mengakses elemen untuk operasi langsung yang terisolasi (misal: fokus otomatis input form dengan `inputElement.focus()`, mengukur posisi koordinat elemen dengan `getBoundingClientRect()`).
- Membangun komponen UI ultra-ringan tanpa dependensi framework (Vanilla JS Widget, SDK pihak ketiga seperti script tracking / chat widget).
- Mengintegrasikan pustaka visualisasi eksternal yang mengelola kanvas sendiri (misal: D3.js, Chart.js, Three.js).

### Kapan Jangan Pakai
- Membangun aplikasi web skala menengah hingga besar dengan interaksi data yang rumit. Gunakan pendekatan deklaratif (React, Vue, Svelte) untuk menghindari ketidaksinkronan antara data dan tampilan.
- Melakukan mutasi massal ratusan item per detik secara naif di dalam loop sinkron.

## 8. Metrik Kunci & Observabilitas

Untuk mengidentifikasi inefisiensi DOM di production:

- **DOM Node Count**: Jumlah total elemen di halaman. Rekomendasi Lighthouse: maksimal **< 800 node** untuk performa optimal, dan **< 1.400 node** batas peringatan. Periksa via console: `document.getElementsByTagName('*').length`.
- **Layout Duration (Chrome DevTools Performance Tab)**: Pantau bar berwarna ungu (*Layout / Recalculate Style*). Jika layout memakan waktu > 16.6ms dalam satu frame, halaman akan mengalami *frame drop* (< 60 FPS).
- **Interaction to Next Paint (INP)**: Mengukur kelambatan respons visual setelah interaksi pengguna. Manipulasi DOM yang lambat pada event click/input langsung merusak skor INP (> 200ms menandakan *Poor*).

## 9. Studi Kasus Industri Nyata

### Penurunan Memori dan DOM Node pada Twitter Web (X)

- **Tantangan Rekayasa Riil**:
  Ketika pengguna Twitter melakukan *infinite scrolling* berjam-jam membaca ribuan tweet, timeline terus menambahkan elemen `<div>` tweet baru ke dalam DOM. Setelah 1.000 tweet termuat, jumlah DOM node melonjak hingga lebih dari 35.000 node. Konsumsi memori browser di perangkat mobile melonjak hingga > 700 MB, menyebabkan tab browser crash (*Out of Memory*) dan scroll menjadi sangat lambat.
- **Implementasi Solusi Industri**:
  Tim rekayasa web Twitter menerapkan teknik **DOM Virtualization (Windowing)**. Hanya tweet yang sedang berada di dalam viewport layar (plus 3-5 tweet di atas dan bawah viewport) yang dipertahankan di dalam DOM (sekitar 30-50 node aktif). Tweet yang telah tergulung keluar dari layar dicopot dari DOM pohon aktif dan digantikan oleh elemen kosong dengan tinggi (*height*) yang setara untuk menjaga posisi scroll bar.
- **Dampak & Hasil Sistem**:
  - Jumlah node DOM terjaga konstan di bawah 500 node terlepas dari seberapa jauh pengguna melakukan scroll.
  - Penggunaan memori turun hingga 65% dan frame rate scroll stabil di 60 FPS pada perangkat low-end.

## 10. Fakta Menarik & Gotchas Production

1. **Forced Synchronous Layout (Layout Thrashing)**:
   Membaca properti geometri (seperti `offsetWidth`, `clientHeight`, `offsetTop`) tepat setelah menulis ke DOM (seperti `element.style.width = '100px'`) memaksa browser menghentikan eksekusi JavaScript dan langsung menghitung ulang geometri layout saat itu juga.
   
   ```javascript
   // JEBAKAN: Layout Thrashing
   for (let i = 0; i < boxes.length; i++) {
     boxes[i].style.width = '100px';       // Write
     const width = boxes[i].offsetWidth;   // Read -> Browser dipaksa reflow seketika!
   }
   ```
2. **Perbedaan `NodeList` Hidup (*Live*) vs Mati (*Static*)**:
   `document.getElementsByClassName()` menghasilkan *Live HTMLCollection*, artinya jika elemen baru ditambahkan, koleksi tersebut langsung berubah secara otomatis. Sedangkan `document.querySelectorAll()` menghasilkan *Static NodeList*, yang snapshot-nya tidak berubah meskipun DOM bertambah. Ketidaktahuan akan hal ini sering menyebabkan infinite loop saat melakukan iterasi.

## 11. Rangkuman Singkat

1. **Pohon Representasi**: DOM adalah representasi hierarki objek C++ di memori browser dari dokumen HTML mentah.
2. **Biaya Reflow**: Manipulasi DOM murah di memori, tetapi mahal pada tahap kalkulasi style, reflow layout, dan repaint visual.
3. **Kendalikan Mutasi**: Gunakan `DocumentFragment`, `requestAnimationFrame`, atau DOM Virtualization untuk meminimalkan rendering thrashing.
4. **Pantau Node Count**: Jaga jumlah node DOM di bawah 800 node agar alokasi memori dan waktu traversal browser tetap optimal.
