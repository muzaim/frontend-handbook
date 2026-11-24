# Local State

## 1. Skenario Sistem Nyata

Pada sebuah aplikasi dashboard analitik keuangan, terdapat komponen accordion daftar transaksi kartu kredit yang memuat 100 baris transaksi. Setiap baris accordion dapat dibuka (*expanded*) atau ditutup (*collapsed*) oleh pengguna untuk melihat rincian biaya.

Developer junior memutuskan untuk menyimpan status buka-tutup accordion di dalam Global State Store (Redux / Zustand):

```javascript
// Kode bermasalah: Menyimpan status UI lokal di Global Store
const toggleRow = (rowId) => {
  dispatch({ type: "TOGGLE_ROW_ACCORDION", payload: { rowId } });
};
```

Dampak pada sistem:
- Setiap kali pengguna mengklik satu panah accordion baris ke-42, Global State Store termutasi.
- Karena state global berubah, **seluruh pohon komponen dashboard** (termasuk komponen grafik penjualan, widget cuaca, navbar, dan 99 baris accordion lainnya) memicu proses render ulang (*re-render cascade*).
- Frame rate browser anjlok dari 60 FPS ke 14 FPS. Animasi buka-tutup accordion terasa patah-patah dan lambat hanya untuk membalikkan nilai boolean `true` / `false`.

Solusi: Menggunakan **Local State (State Lokal)** di tingkat komponen individu (misal `useState` di React atau `ref()` di Vue). Status buka-tutup accordion hanya menjadi urusan baris komponen itu sendiri, mengisolasi render ulang tepat pada 1 baris tersebut tanpa mengganggu komponen lain di sekitarnya.

## 2. Apa Itu dan Buat Apa?

Local State (Status Lokal atau *Component State*) adalah data dinamis yang disimpan, dikelola, dan hanya digunakan secara eksklusif di dalam **satu komponen tertentu** atau komponen anak langsungnya.

Siklus hidup Local State terikat erat dengan siklus hidup komponen tersebut di layar: ketika komponen di-*mount* ke DOM, state lokal diinisialisasi; dan ketika komponen di-*unmount* (dihapus dari DOM), memori state lokal otomatis dibersihkan oleh Garbage Collector.

Fungsi utama Local State:
- **Mengisolasi Pembaruan Antarmuka**: Mencegah efek domino render ulang pada komponen yang tidak berkepentingan.
- **Mengelola Interaksi UI Sesaat (*Ephemeral UI State*)**: Menyimpan nilai ketikan input teks form, status visibilitas modal dialog, tab navigasi aktif, atau status hover tooltip.
- **Mempertahankan Enkapsulasi Komponen**: Menjadikan komponen dapat digunakan kembali (*reusable*) di tempat lain tanpa ketergantungan pada store luar.

## 3. Kenapa Harus Pakai Ini?

Banyak developer pemula jatuh ke dalam jebakan menaruh seluruh data aplikasi ke dalam Global Store (Redux/Zustand).

Menggunakan Local State untuk data UI adalah prinsip rekayasa yang sangat penting:
1. **Performa Tinggi Tanpa Konfigurasi Rumit**: Hanya komponen yang statusnya berubah yang menghitung ulang Virtual DOM.
2. **Kemudahan Penghapusan Kode (*Refactoring*)**: Jika suatu fitur dihapus, Anda cukup menghapus berkas komponen tersebut beserta state lokal di dalamnya tanpa meninggalkan "kode sampah" di global store.

## 4. Cara Kerjanya (Diagram Alur Isolasi Render)

Perbandingan dampak render antara Local State vs Global State:

```
DENGAN GLOBAL STATE (Semua Komponen Terkena Dampak):
[ Global Store: { isRowOpen: true } ]
                 |
  +--------------+--------------+--------------+
  |              |              |              |
  v              v              v              v
[ Navbar ]   [ Charts ]   [ Row 1..99 ]   [ Row 42 ]
(Re-render)  (Re-render)  (Re-render)    (Re-render)
*BOROS CPU & BIKIN LAG!*

----------------------------------------------------------------------

DENGAN LOCAL STATE (Terisolasi Sempurna):
[ Global Store: (Hanya data transaksi penting) ]
  |
  +---> [ Navbar ]    (Tenang, tidak re-render)
  +---> [ Charts ]    (Tenang, tidak re-render)
  +---> [ Row 42 ]    [ Local State: isOpen = true ]
                           |
                           v
              HANYA Row 42 yang Re-render!
              *MULUS 60 FPS!*
```

## 5. Cuplikan Logika Inti

### Implementasi Local State Terisolasi (React `useState` / `useReducer`)

```typescript
// components/TransactionRow.tsx
import React, { useState } from "react";

interface Transaction {
  id: string;
  merchant: string;
  amount: number;
  details: string;
}

export const TransactionRow: React.FC<{ transaction: Transaction }> = ({ transaction }) => {
  // Local State: Hanya mengontrol baris ini sendiri!
  const [isOpen, setIsOpen] = useState<boolean>(false);

  return (
    <div className="border-b py-3">
      <div 
        className="flex justify-between cursor-pointer"
        onClick={() => setIsOpen((prev) => !prev)}
      >
        <span className="font-semibold">{transaction.merchant}</span>
        <span>Rp {transaction.amount.toLocaleString()}</span>
        <span>{isOpen ? "[-] Tutup" : "[+] Rincian"}</span>
      </div>

      {/* Rincian hanya muncul jika isOpen true */}
      {isOpen && (
        <div className="mt-2 text-sm text-gray-600 bg-gray-50 p-3 rounded">
          <p>{transaction.details}</p>
        </div>
      )}
    </div>
  );
};
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan
- **Performa Tercepat**: Render ulang terlokalisasi secara ketat pada node terkecil pohon DOM.
- **Nol Boilerplate**: Tidak membutuhkan pembuatan context provider, reducer, action creator, atau selector.
- **Portabilitas Sempurna**: Komponen dapat dipindahkan ke proyek atau repositori lain secara mandiri (*self-contained*).

### Kekurangan & Tantangan
- **Masalah Prop Drilling**: Jika komponen cicit (*grandchild*) membutuhkan state ini, state harus dinaikkan (*lifted up*) ke induk dan disalurkan ke bawah melalui props lapis demi lapis.
- **State Hilang Saat Unmount**: Jika komponen terhapus dari DOM (misal pengguna berpindah tab formulir), seluruh nilai Local State akan musnah dan kembali ke nilai default saat dibuka kembali.

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Pakai (Gunakan Sebagai Default Pertama!)
- Status formulir interaktif (nilai teks input, pesan validasi error spesifik input).
- Status visual sementara (apakah modal terbuka, apakah dropdown aktif, tab terpilih saat ini).
- Data sementara yang tidak dibutuhkan oleh komponen lain di luar hierarki keluarga tersebut.

### Kapan Harus Beralih ke Global State atau URL State
- Data yang dibutuhkan oleh komponen yang berada di cabang pohon DOM yang berjauhan (misal data profil user login di Navbar dan di Halaman Pengaturan).
- State yang harus bertahan saat halaman di-refresh atau dibagikan ke orang lain (misal filter pencarian barang atau nomor paginasi — gunakan **URL Search Params**).

## 8. Metrik Kunci & Observabilitas

- **Wasted Renders Counter (React Profiler)**: Pastikan saat komponen berinteraksi, jumlah komponen yang dirender ulang pada tab Profiler hanya berjumlah 1.
- **Memory Footprint**: Amati apakah memori komponen berhasil dibebaskan setelah komponen di-unmount dari DOM (bebas dari penahanan referensi closure).

## 9. Studi Kasus Industri Nyata

### Migrasi Twitter Web dari Redux Berat ke Local State Komponen

- **Tantangan Rekayasa Riil**:
  Di era awal rilis Twitter Lite (PWA), tim insinyur menyimpan hampir seluruh status UI—termasuk status apakah tombol like pada tweet tertentu sedang mengalami animasi hati meletup—ke dalam satu Redux Store global. Akibatnya, saat pengguna menekan tombol like pada koneksi ponsel murah, seluruh timeline tweet memicu perbandingan selector Redux, menimbulkan jeda terasa (*input lag*).
- **Implementasi Solusi Industri**:
  Tim Twitter merombak arsitektur dengan prinsip: *"Push state as low as possible"*. Status animasi tombol like, kotak balasan tweet yang terbuka, dan hover profil pengguna dipindahkan sepenuhnya menjadi **Local State komponen**.
- **Dampak & Hasil Sistem**:
  - Waktu respons interaksi klik like menjadi instan (< 16 ms).
  - Konsumsi CPU scripting berkurang hingga 40% saat scrolling feed.

## 10. Fakta Menarik & Gotchas Production

1. **Aturan "Lift State Up"**:
   Jika dua komponen saudara (*sibling components*) tiba-tiba membutuhkan data yang sama, jangan langsung meloncat ke Redux/Zustand! Cukup naikkan (*lift up*) state tersebut satu tingkat ke komponen induk bersama mereka (*common parent*), lalu salurkan via props.
2. **Stale Closure pada Event Listener**:
   Jika Anda membaca nilai local state di dalam fungsi callback `setTimeout` atau listener DOM manual yang dibuat di `useEffect` tanpa array dependensi yang benar, fungsi tersebut akan "menangkap" nilai state lama (*stale state snapshot*).

## 11. Rangkuman Singkat

1. **Mulai dari Lokal**: Jadikan Local State sebagai pilihan pertama dan utama untuk setiap data baru sebelum mempertimbangkan global store.
2. **Kendalikan Render**: Mengisolasi state di dalam komponen adalah cara termudah menjaga aplikasi tetap berjalan 60 FPS.
3. **Siklus Hidup Terikat**: State lokal hidup dan mati bersama komponennya di DOM.
4. **Gunakan URL untuk Filter**: Simpan status filter dan tab di URL parameters jika state tersebut perlu bertahan saat di-refresh.
