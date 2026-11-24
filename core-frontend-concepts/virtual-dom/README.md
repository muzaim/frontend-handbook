# Virtual DOM

## 1. Skenario Sistem Nyata

Pada aplikasi spreadsheet kolaboratif berbasis web (seperti Google Sheets atau Airtable), satu tabel dapat memuat 50 kolom dan 2.000 baris. Ketika seorang pengguna mengubah format angka pada satu sel tertentu (misal menambahkan simbol mata uang `$`), aplikasi perlu memperbarui UI.

Jika aplikasi menggunakan manipulasi DOM langsung tanpa abstraksi cerdas:
- Aplikasi tidak tahu secara pasti elemen mana saja yang terpengaruh tanpa logika manual yang rumit.
- Developer yang malas cenderung memanggil fungsi render ulang secara global: merender ulang seluruh tabel 100.000 sel.
- Dampaknya, browser menghitung ulang layout seluruh tabel, input pengguna macet selama 400 milidetik, kursor mengetik terputus, dan pengalaman pengguna menjadi sangat buruk.

Solusi: Menggunakan **Virtual DOM**. Aplikasi menyimpan representasi ringan seluruh struktur UI di memori JavaScript (RAM). Saat data berubah, representasi baru dibuat di memori, dibandingkan dengan representasi lama (*diffing*), dan hanya 1 sel yang perubahannya diterapkan ke Real DOM (*patching*).

## 2. Apa Itu dan Buat Apa?

Virtual DOM (VDOM) adalah pola arsitektur di mana representasi ideal atau "virtual" dari UI dipertahankan dalam memori JavaScript murni sebagai objek biasa (Plain Old JavaScript Object / POJO), lalu disinkronkan dengan "Real DOM" browser melalui proses yang disebut **Reconciliation**.

Pola ini dipopulerkan oleh React dan kemudian diadopsi dalam berbagai varian oleh Vue (VDOM Snabbdom-based), Preact, dan Inferno.

Struktur objek VDOM biasanya berupa pohon objek sederhana:
```javascript
{
  type: 'div',
  props: { className: 'alert' },
  children: [
    { type: 'span', props: {}, children: ['Data berhasil disimpan'] }
  ]
}
```

Fungsi utama VDOM:
- **Deklaratif**: Memungkinkan developer menulis UI sebagai fungsi dari status data (`UI = f(state)`), tanpa perlu memikirkan instruksi imperatif seperti `appendChild` atau `removeChild`.
- **Batching & Diffing**: Mengelompokkan mutasi data dan menghitung delta perubahan terkecil sebelum menyentuh Real DOM.

## 3. Kenapa Harus Pakai Ini?

Banyak orang salah kaprah dengan mengira Virtual DOM "lebih cepat daripada manipulasi DOM langsung". Realitanya, **Virtual DOM tidak lebih cepat daripada manipulasi DOM manual yang ditulis dengan sangat teliti dan sempurna**.

Lantas mengapa harus pakai Virtual DOM?
1. **Developer Experience (DX) & Skalabilitas Kode**: Menulis kode manipulasi DOM imperatif secara manual pada aplikasi ribuan baris dengan ratusan interaksi sangat rawan bug dan sulit dirawat.
2. **Kinerja yang Cukup Baik Secara Konsisten (*Predictable Performance*)**: VDOM memberikan performa "cukup cepat secara default" tanpa menuntut developer menjadi pakar mesin browser di setiap baris kode.
3. **Abstraksi Lintas Platform (*Cross-Platform Rendering*)**: Karena VDOM hanyalah objek JavaScript, ia dapat dikonversi ke target selain web browser, seperti mobile native (React Native), terminal CLI (Ink), atau PDF.

## 4. Cara Kerjanya (Diagram Alur Reconciliation)

Proses VDOM terbagi menjadi 3 fase utama: Render (Snapshot Baru) -> Diffing (Perbandingan Algoritmik) -> Commit (Patching ke Real DOM).

```
+---------------------------------------------------------------+
|                    State Aplikasi Berubah                     |
|                      count: 1  -->  count: 2                  |
+---------------------------------------------------------------+
                               |
                               v
+---------------------------------------------------------------+
|                  Buat VDOM Tree Baru (Memori)                 |
|       { type: 'button', children: ['Klik: 2'] }              |
+---------------------------------------------------------------+
                               |
                               v
+---------------------------------------------------------------+
|               Diffing Algorithm (Reconciliation)              |
|                                                               |
|    VDOM Lama (RAM)                     VDOM Baru (RAM)        |
|    <div>                               <div>                  |
|      <p>Halo</p>        Bandingkan       <p>Halo</p> (Sama)   |
|      <span>1</span>     =========>       <span>2</span> (Ubah)|
|                                                               |
|    Hasil: Hanya teks <span> yang berubah!                     |
+---------------------------------------------------------------+
                               |
                               v
+---------------------------------------------------------------+
|                      Commit Phase (Patching)                  |
|           spanNode.textContent = "2"; (Sangat Cepat)          |
+---------------------------------------------------------------+
                               |
                               v
+---------------------------------------------------------------+
|                           REAL DOM                            |
|             Browser hanya merender perubahan teks             |
+---------------------------------------------------------------+
```

## 5. Cuplikan Logika Inti

### Implementasi Minimal Mesin Diffing Virtual DOM

Berikut adalah inti cara kerja algoritma rekonsiliasi VDOM dalam JavaScript murni:

```javascript
// 1. Fungsi pembuat elemen VDOM (Virtual Node)
function h(type, props, ...children) {
  return { type, props: props || {}, children };
}

// 2. Mengonversi VNode menjadi Real DOM Node
function createElement(vnode) {
  if (typeof vnode === "string" || typeof vnode === "number") {
    return document.createTextNode(String(vnode));
  }
  const el = document.createElement(vnode.type);
  Object.keys(vnode.props).forEach((key) => {
    el.setAttribute(key, vnode.props[key]);
  });
  vnode.children.map(createElement).forEach(el.appendChild.bind(el));
  return el;
}

// 3. Algoritma Diffing & Patching
function updateElement(parent, newNode, oldNode, index = 0) {
  // Kasus 1: Node lama dihapus
  if (!newNode) {
    parent.removeChild(parent.childNodes[index]);
    return;
  }
  // Kasus 2: Node baru ditambahkan
  if (!oldNode) {
    parent.appendChild(createElement(newNode));
    return;
  }
  // Kasus 3: Tipe node berubah (misal <div> menjadi <span>)
  if (
    newNode.type !== oldNode.type ||
    typeof newNode !== typeof oldNode ||
    (typeof newNode === "string" && newNode !== oldNode)
  ) {
    parent.replaceChild(createElement(newNode), parent.childNodes[index]);
    return;
  }
  // Kasus 4: Node sama, bandingkan children-nya secara rekursif
  if (newNode.type) {
    const max = Math.max(newNode.children.length, oldNode.children.length);
    for (let i = 0; i < max; i++) {
      updateElement(
        parent.childNodes[index],
        newNode.children[i],
        oldNode.children[i],
        i
      );
    }
  }
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan
- **Pemrograman Deklaratif**: Developer cukup fokus pada bagaimana tampilan UI berdasarkan status data saat ini, bukan memikirkan urutan manipulasi DOM.
- **Batching Otomatis**: Beberapa pembaruan state dalam satu siklus event loop digabungkan menjadi satu kali pembaruan Real DOM.
- **Isolasi Keamanan XSS**: Pustaka VDOM secara bawaan menyaring (*escaping*) string sebelum memasukkannya ke Real DOM.

### Kekurangan
- **Overhead Alokasi Memori**: Membuat ratusan ribu objek JavaScript setiap kali render membebani Garbage Collector (GC), yang dapat memicu *GC pause* (jank).
- **Biaya Komputasi Diffing**: Menelusuri pohon VDOM dua kali (lama vs baru) membutuhkan siklus CPU. Jika pohon komponen sangat besar, diffing bisa memakan waktu belasan milidetik.
- **Ukuran Bundle Tambahan**: Aplikasi membutuhkan runtime library (misal react-dom ~40KB minified+gzip) hanya untuk menjalankan rekonsiliasi.

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Pakai
- Membangun Single Page Application (SPA) modern yang interaktif dengan perubahan status data yang dinamis.
- Tim developer berskala menengah-besar di mana konsistensi pola arsitektur kode menjadi prioritas utama.
- Proyek yang menuntut portabilitas kode ke platform lain (seperti web ke mobile native).

### Kapan Jangan Pakai
- Halaman web berbasis konten statis (Blog, Portal Berita, Landing Page dokumentasi). Pilihan SSG murni atau framework tanpa VDOM (Astro, Svelte) jauh lebih efisien.
- Sistem dengan keterbatasan memori ekstrem (perangkat IoT atau Smart TV low-end) di mana overhead Garbage Collection VDOM dapat membuat sistem macet.

## 8. Metrik Kunci & Observabilitas

- **Total Blocking Time (TBT)**: Memantau berapa lama main thread terblokir oleh eksekusi JavaScript (termasuk fase render & diffing VDOM) selama pemuatan halaman. Ambang ideal: **< 200 ms**.
- **React/Vue DevTools Profiler**:
  - Gunakan fitur *Record* untuk melihat durasi render tiap komponen.
  - Cari komponen yang ditandai warna kuning/merah (*wasted renders*) — komponen yang di-render ulang padahal prop dan state-nya tidak berubah.
- **Garbage Collection Frequency**: Pantau tab Memory di DevTools. Jika grafik memori berbentuk gergaji tajam (*sawtooth pattern*) yang berulang cepat, itu indikasi alokasi objek VDOM berlebih yang memicu GC thrashing.

## 9. Studi Kasus Industri Nyata

### Evolusi Mesin Render Svelte vs VDOM di Rekayasa Modern

- **Tantangan Rekayasa Riil**:
  Pada aplikasi analitik real-time yang menerima ribuan mutasi data per detik, eksekusi pohon VDOM di React sering memicu pemanggilan ratusan fungsi komponen dan diffing objek memori berulang, meskipun sebagian besar UI tidak berubah.
- **Implementasi Solusi Industri**:
  Rich Harris menciptakan **Svelte** dengan premis radikal: *"Virtual DOM is pure overhead"*. Alih-alih melakukan diffing di browser pada saat runtime, Svelte memindahkan pekerjaan tersebut ke tahap kompilasi (*compile-time*). Kompiler Svelte menganalisis kode dan menghasilkan kode JavaScript imperatif presisi tinggi: `p.textContent = count`.
- **Dampak & Hasil Sistem**:
  - Tidak ada runtime VDOM library yang perlu diunduh browser (mengurangi bundle size hingga puluhan kilobyte).
  - Konsumsi memori RAM turun signifikan dan kecepatan eksekusi meningkat drastis pada aplikasi dengan frekuensi mutasi data tinggi.

## 10. Fakta Menarik & Gotchas Production

1. **Jebakan Penggunaan Index sebagai `key`**:
   Pada list dinamis di React/Vue, memberikan index array sebagai `key` (`items.map((item, index) => <Item key={index} />)`) adalah bencana saat item diurutkan ulang (*reordering*) atau disisipkan di awal. VDOM akan mengira elemen tidak berpindah dan hanya mengubah teksnya, yang dapat menyebabkan input form mempertahankan state lama dan animasi transisi rusak fatal.
2. **VDOM Bukan Fitur Browser**:
   Banyak developer junior mengira Virtual DOM adalah fitur resmi dari W3C atau API bawaan browser Chrome/Firefox. Faktanya, VDOM adalah 100% pola desain perangkat lunak (*software design pattern*) yang ditulis murni dalam JavaScript di tingkat library pengguna.

## 11. Rangkuman Singkat

1. **Abstraksi Memori**: VDOM adalah representasi pohon elemen UI dalam bentuk objek JavaScript di memori RAM.
2. **Rekonsiliasi**: Melalui proses *render -> diff -> commit*, framework hanya memutasi bagian Real DOM yang benar-benar berubah.
3. **Bukan Soal Kecepatan Murni**: VDOM diciptakan untuk kemudahan perawatan (*maintainability*) dan performa yang stabil secara default, bukan untuk mengalahkan kecepatan manipulasi DOM manual murni.
4. **Waspadai Render Berlebih**: Optimalkan re-render komponen yang tidak perlu menggunakan `memo`, pemilihan state yang terisolasi, atau framework berbasis reaktivitas kompilasi.
