# Memoization

## 1. Skenario Sistem Nyata

Pada aplikasi dashboard portofolio analitik saham, terdapat komponen induk yang memuat grafik interaktif dan tabel transaksi 1.000 baris. Di dalam komponen induk terdapat sebuah timer yang memperbarui jam digital di sudut layar setiap 1 detik:

```javascript
// Kode bermasalah: Menghitung ulang komputasi berat di setiap siklus re-render
function StockDashboard() {
  const [time, setTime] = useState(new Date());

  useEffect(() => {
    const timer = setInterval(() => setTime(new Date()), 1000);
    return () => clearInterval(timer);
  }, []);

  // Komputasi matematika berat yang memakan waktu 120 milidetik!
  // Fungsi ini dieksekusi ulang SETIAP DETIK hanya karena state time berubah!
  const heavyAnalytics = runComplexStockAlgorithm(rawTransactions);

  return (
    <div>
      <Header time={time} />
      <HeavyStockChart data={heavyAnalytics} />
    </div>
  );
}
```

Dampak pada sistem:
- Setiap kali jam berganti detik, state `time` berubah, memicu `StockDashboard` melakukan re-render.
- Algoritma `runComplexStockAlgorithm` yang memakan waktu 120 milidetik dipaksa berjalan ulang dari nol setiap detik, padahal data `rawTransactions` sama sekali tidak pernah berubah!
- Main-thread terblokir selama 120ms di setiap detik. Layar mengalami *jank*, animasi kursor tersendat, dan kipas laptop berputar kencang.

Solusi: Menggunakan teknik **Memoization**. Menyimpan hasil komputasi mahal di dalam memori cache menggunakan hook `useMemo`. Algoritma berat hanya dihitung ulang jika dan hanya jika data dependensinya (`rawTransactions`) benar-benar berubah; pembaruan state `time` cukup mengambil hasil kalkulasi lama dari cache memori dalam waktu 0 milidetik.

## 2. Apa Itu dan Buat Apa?

Memoization (Memoarisasi) adalah teknik optimasi pemrograman di mana hasil pemanggilan suatu fungsi **disimpan di dalam media penyimpanan cache (RAM)**, sehingga ketika fungsi tersebut dipanggil kembali dengan parameter input (*arguments*) yang sama persis, sistem tidak perlu menghitung ulang logika fungsi tersebut dari nol, melainkan langsung mengembalikan hasil kalkulasi yang sudah tersimpan di cache.

Dalam ekosistem antarmuka pengguna modern (khususnya React):
1. **`useMemo`**: Meng-cache hasil nilai kalkulasi matematika atau transformasi data berat antar siklus render.
2. **`useCallback`**: Meng-cache dan mempertahankan kestabilan referensi memori dari suatu fungsi callback agar tidak dibuat ulang di setiap render.
3. **`React.memo` (Higher-Order Component)**: Mencegah komponen anak melakukan render ulang (*re-render bailout*) jika nilai `props` yang diterimanya tidak mengalami perubahan referensi memori.

Fungsi utama Memoization:
- Menghindari komputasi CPU berulang yang sia-sia (*wasteful computations*).
- Menjaga kestabilan referensi memori objek dan fungsi agar tidak memicu efek samping berantai pada komponen anak.

## 3. Kenapa Harus Pakai Ini?

Secara default, di dalam framework deklaratif seperti React: **ketika sebuah komponen induk me-render ulang, SELURUH fungsi, variabel di dalamnya, dan seluruh komponen anak di bawahnya akan di-render ulang secara default**.

Memoization bertindak sebagai rem tangan pintar:
1. Menahan kalkulasi berat agar tidak dijalankan kecuali inputnya berubah.
2. Memutus rantai render ulang komponen anak yang pasif (*Pure Components*).

## 4. Cara Kerjanya (Diagram Alur Mekanisme Cache Dependencies)

Visualisasi evaluasi dependensi pada `useMemo`:

```
Komponen Me-render Ulang (State Lain Berubah)
                         |
                         v
+---------------------------------------------------------------+
|                      useMemo(() => fn(), [a, b])              |
+---------------------------------------------------------------+
|  Bandingkan nilai dependensi lama vs baru secara dangkal      |
|  (Object.is(oldA, newA) && Object.is(oldB, newB))             |
|                                                               |
|  [SAMA PERSIS] ---> AMBIL DARI CACHE RAM!                     |
|                     (Nol komputasi baru, waktu: 0.001 ms)     |
|                                                               |
|  [BERBEDA] -------> JALANKAN FUNGSI fn() ULANG!              |
|                     Simpan hasil baru ke cache -> Kembalikan! |
+---------------------------------------------------------------+
```

## 5. Cuplikan Logika Inti

### Memadukan `useMemo`, `useCallback`, dan `React.memo` Secara Tepat

```tsx
import React, { useState, useMemo, useCallback } from "react";

interface Item {
  id: number;
  name: string;
  price: number;
}

// 1. Bungkus komponen anak dengan React.memo agar tidak re-render
// jika props 'onItemClick' dan 'items' tidak berganti referensi memori!
const PureItemList = React.memo<{
  items: Item[];
  onItemClick: (id: number) => void;
}>(({ items, onItemClick }) => {
  console.log("PureItemList me-render!");
  return (
    <ul>
      {items.map((item) => (
        <li key={item.id} onClick={() => onItemClick(item.id)}>
          {item.name} - Rp {item.price}
        </li>
      ))}
    </ul>
  );
});

export function ProductCatalog({ rawItems }: { rawItems: Item[] }) {
  const [filterText, setFilterText] = useState("");
  const [theme, setTheme] = useState("dark");

  // 2. useMemo: Meng-cache hasil pemfilteran data array 10.000 item.
  // Hanya dihitung ulang jika rawItems atau filterText berubah!
  // Ganti tema 'theme' TIDAK AKAN memicu loop filter ini!
  const filteredItems = useMemo(() => {
    return rawItems.filter((item) =>
      item.name.toLowerCase().includes(filterText.toLowerCase())
    );
  }, [rawItems, filterText]);

  // 3. useCallback: Mempertahankan referensi memori fungsi callback yang stabil.
  // Mencegah PureItemList mengira prop fungsinya selalu berubah di tiap render!
  const handleItemClick = useCallback((id: number) => {
    console.log("Item diklik:", id);
  }, []);

  return (
    <div>
      <input 
        value={filterText} 
        onChange={(e) => setFilterText(e.target.value)} 
        placeholder="Cari..." 
      />
      <button onClick={() => setTheme(t => t === "dark" ? "light" : "dark")}>
        Ganti Tema (Saat ini: {theme})
      </button>

      <PureItemList items={filteredItems} onItemClick={handleItemClick} />
    </div>
  );
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan
- **Peningkatan FPS Drastis**: Menghilangkan stuttering pada aplikasi interaktif yang memproses dataset besar.
- **Isolasi Beban Komponen**: Komponen anak yang dibungkus `React.memo` terlindungi dari re-render induk yang berisik.

### Kekurangan & Biaya Tersembunyi (*The Cost of Memoization*)
- **Biaya Overhead Komparasi Dangkal**: Menyimpan closure fungsi, mengalokasikan array dependensi di memori, dan membandingkan dependensi di setiap frame membutuhkan waktu komputasi CPU dan alokasi memori RAM tersendiri.
- **Bisa Memperlambat Jika Digunakan Sembarangan**: Jika komputasi yang di-memoize sangat sepele (misal `const sum = useMemo(() => a + b, [a, b])`), biaya menjalankan overhead `useMemo` justru **jauh lebih mahal** daripada sekadar mengeksekusi `a + b` secara langsung!

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Memakai Memoization
- Operasi transformasi atau agregasi data yang kompleks (mengurutkan, memfilter, atau mengelompokkan array dengan > 500 item).
- Menjaga kestabilan referensi objek atau fungsi yang dijadikan dependensi di dalam hook `useEffect` lain.
- Props yang dilewatkan ke komponen anak yang terbukti lambat dan sudah dibungkus dengan `React.memo`.

### Kapan JANGAN Menggunakan Memoization
- Operasi matematika atau manipulasi string sederhana (`a + b`, format mata uang sederhana).
- Komponen anak yang sangat ringan (seperti tag `<button>` atau `<span>` biasa). Jangan membungkus semua komponen dengan `React.memo` secara membabi-buta!

## 8. Metrik Kunci & Observabilitas

- **React DevTools Profiler (Render Reason)**: Buka tab Profiler, klik icon roda gigi (*Settings*), lalu centang **"Record why each component rendered while profiling"**.
- **Wasted Renders Duration**: Amati durasi render komponen di panel Profiler. Jika waktu render < 2 milidetik, memoization kemungkinan besar tidak dibutuhkan.

## 9. Studi Kasus Industri Nyata

### Optimasi Tabel Transaksi Finansial Robinhood Web

- **Tantangan Rekayasa Riil**:
  Halaman portofolio Robinhood menampilkan grafik harga saham yang menerima aliran data real-time via WebSocket beberapa kali per detik. Di bawah grafik terdapat tabel transaksi yang memuat ratusan aset saham. Setiap kali titik grafik baru tiba dari websocket, seluruh baris tabel di bawahnya ikut me-render ulang, menyebabkan scroll tabel terasa patah-patah (*dropped frames*).
- **Implementasi Solusi Industri**:
  Robinhood menerapkan strategi memoization granular:
  1. Komponen baris tabel dibungkus dengan `React.memo` menggunakan fungsi pembanding kustom (*custom arePropsEqual*).
  2. Data ringkasan portofolio dibungkus menggunakan `useMemo` dengan dependensi saldo akun.
  3. Seluruh handler aksi klik tombol beli/jual distabilkan menggunakan `useCallback`.
- **Dampak & Hasil Sistem**:
  - Jumlah komponen yang re-render per detik berkurang dari 350 komponen menjadi hanya 2 komponen (titik grafik dan angka saldo).
  - Frame rate antarmuka stabil di 60 FPS saat jam sibuk pasar bursa.

## 10. Fakta Menarik & Gotchas Production

1. **Jebakan Referensi Objek Baru pada Default Value**:
   ```javascript
   // JEBAKAN FATAL:
   const userSettings = useMemo(() => fetchSettings(), []);
   // Mengirim objek literal kosong {} sebagai fallback mematahkan React.memo anak!
   <ChildComponent options={userSettings || {}} />
   ```
   Objek `{}` literal membuat referensi memori baru di setiap render. Gunakan konstanta global: `const EMPTY_OBJECT = {}`.
2. **React Compiler (React 19+)**:
   Mulai React 19, tim inti React memperkenalkan **React Compiler (Forget)**. Kompiler ini secara otomatis menganalisis kode Anda saat proses build dan menyuntikkan memoization optimal secara otomatis di belakang layar. Di masa depan, developer tidak perlu lagi menulis `useMemo` dan `useCallback` secara manual!

## 11. Rangkuman Singkat

1. **Cache Hasil Komputasi**: Memoization menyimpan hasil kalkulasi mahal di memori RAM dan mengembalikannya secara instan jika inputnya sama.
2. **Trio React**: `useMemo` untuk nilai data, `useCallback` untuk fungsi callback, dan `React.memo` untuk komponen UI.
3. **Bukan Solusi Gratis**: Memoization memiliki overhead memori dan komparasi; jangan gunakan untuk kalkulasi sepele.
4. **Masa Depan Otomatis**: Pahami konsep dasarnya hari ini untuk mempersiapkan era optimasi otomatis dengan React Compiler.
