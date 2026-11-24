# Cache Invalidation

## 1. Skenario Sistem Nyata

Pada aplikasi e-commerce admin dashboard, seorang staf gudang mengedit harga produk sepatu dari **Rp 500.000** menjadi **Rp 450.000** pada form edit produk dan menekan tombol "Simpan Perubahan". Notifikasi pop-up hijau bertuliskan *"Produk Berhasil Diperbarui"* muncul di layar.

Namun, ketika staf tersebut mengklik menu kembali ke halaman **Daftar Tabel Produk**:
- Di tabel daftar produk, harga sepatu tersebut **masih tertulis Rp 500.000**!
- Staf mengira tombol simpan tadi belum bekerja, lalu menekan tombol simpan ulang berkali-kali.
- Masalahnya bukan di backend, melainkan di frontend: data daftar tabel produk disimpan di dalam memory cache peramban, dan frontend **lupa membatalkan (*invalidate*) cache daftar produk** setelah operasi edit selesai!

Kasus ini mencerminkan kutipan legendaris ilmuwan komputer Phil Karlton: *"There are only two hard things in Computer Science: cache invalidation and naming things."*

Solusi: Menguasai teknik **Cache Invalidation (Pembatalan Cache)** di frontend. Menghapus atau menandai data di memory cache sebagai "kedaluwarsa" secara terkoordinasi segera setelah mutasi data berhasil dilakukan, sehingga query berikutnya otomatis mengambil data terbaru dari database.

## 2. Apa Itu dan Buat Apa?

Cache Invalidation adalah proses menghapus, memperbarui, atau menandai entri data tertentu di dalam media penyimpanan sementara (Memory RAM frontend, HTTP Cache, IndexedDB) sebagai **kedaluwarsa (*stale / invalid*)**, memaksa sistem untuk membaca kembali data segar dari sumber aslinya (server database).

Jika caching adalah teknik mempercepat pembacaan data, maka Cache Invalidation adalah **disiplin menjaga agar data yang di-cache tersebut tidak menyajikan kebohongan (data basi) kepada pengguna**.

Tiga pola utama Cache Invalidation di frontend:
1. **Time-To-Live (TTL / Expiration-based)**: Cache otomatis dianggap basi setelah durasi waktu tertentu berlalu (misal `staleTime: 5 menit`).
2. **Event-Driven / Mutation Invalidation**: Cache dihapus atau ditandai basi tepat saat sebuah aksi mutasi berhasil diselesaikan (misal setelah `POST /api/products`, batalkan query `['products']`).
3. **Pessimistic vs Optimistic Invalidation**: Membatalkan cache dan menunggu konfirmasi server vs langsung menyuntikkan data baru ke cache seketika.

Fungsi utama Cache Invalidation:
- Menjamin konsistensi data antarmuka setelah pengguna melakukan aksi tambah, ubah, atau hapus (*C-U-D*).
- Menghindari bug visual di mana data lama masih terpampang di layar setelah diperbarui.

## 3. Kenapa Harus Pakai Ini?

Tanpa strategi invalidasi cache yang presisi:
1. Pengguna akan melihat data yang bertentangan di berbagai halaman aplikasi.
2. Pengguna terpaksa melakukan *Hard Refresh* manual (`Ctrl + F5`) agar dapat melihat perubahan yang baru saja mereka simpan.
3. Sebaliknya, jika Anda menghapus seluruh cache secara membabi-buta (*Nuke the Cache*), seluruh manfaat performa caching akan hilang karena browser harus mengunduh ulang seluruh data aplikasi dari nol.

## 4. Cara Kerjanya (Diagram Alur Targeted Mutation Invalidation)

Visualisasi invalidasi cache terarah menggunakan hierarki Query Keys:

```
Pohon Cache di Memori Frontend:
- ['products', 'list']               <--- Cache Daftar Tabel Produk
- ['products', 'detail', 42]         <--- Cache Detail Produk ID 42
- ['users', 'profile']               <--- Cache Profil User (Tidak Berhubungan)

----------------------------------------------------------------------

Pengguna Menyimpan Perubahan pada Produk ID 42 (PATCH /api/products/42)
                         |
                         v
[ Server Backend Membalas 200 OK: Berhasil Diperbarui ]
                         |
                         v
       +------------------------------------+
       |   TARGETED INVALIDATION DISPATCH   |
       |   invalidateQueries(['products'])  |
       +------------------------------------+
                         |
      +------------------+------------------+
      | Menargetkan semua key               | Mengabaikan data
      | berawalan ['products']              | yang tidak berhubungan
      v                                     v
['products', 'list']         --> STATUS: STALE!  ['users', 'profile']
['products', 'detail', 42]   --> STATUS: STALE!  (Tetap aman di cache,
                                                  TIDAK di-refetch!)
      |
      v
Browser secara otomatis me-refetch HANYA data produk yang terpengaruh!
Tabel produk seketika menampilkan harga baru: Rp 450.000!
```

## 5. Cuplikan Logika Inti

### Strategi Invalidasi Terarah vs Pembaruan Cache Langsung (TanStack Query)

Dua pendekatan mutasi cache di production:

```typescript
import { useMutation, useQueryClient } from "@tanstack/react-query";

export function useEditProductPrice() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: async ({ id, newPrice }: { id: number; newPrice: number }) => {
      const res = await fetch(`/api/products/${id}`, {
        method: "PATCH",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ price: newPrice }),
      });
      return res.json();
    },

    onSuccess: (updatedProduct) => {
      // PENDEKATAN 1: Invalidasi Terarah (Sederhana & Aman)
      // Menandai seluruh query produk sebagai stale agar di-fetch ulang di background:
      queryClient.invalidateQueries({
        queryKey: ["products"],
        exact: false, // Membatalkan ['products', 'list'], ['products', 'detail', id], dll.
      });

      // PENDEKATAN 2: Direct Cache Update (Efisien - Nol Network Re-fetch!)
      // Langsung perbarui data di dalam cache detail secara presisi tanpa request HTTP baru:
      queryClient.setQueryData(["products", "detail", updatedProduct.id], updatedProduct);
      
      // Perbarui juga item di dalam daftar tabel jika sudah ada di memori:
      queryClient.setQueryData<Product[]>(["products", "list"], (oldList) =>
        oldList?.map((p) => (p.id === updatedProduct.id ? updatedProduct : p))
      );
    },
  });
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan
- **Integritas UI Terjamin**: Menghilangkan fenomena data hantu yang membingungkan pengguna.
- **Efisiensi Jaringan Terarah**: Hanya data yang terpengaruh yang diunduh ulang, membiarkan 95% data cache lain tetap aman di RAM.

### Kekurangan & Tantangan
- **Tantangan Query Key Mismatch**: Jika key yang dibatalkan tidak sama persis atau strukturnya berbeda (misal Anda membatalkan `['product']` dengan huruf tunggal padahal query memakai `['products']` jamak), invalidasi gagal tanpa ada pesan error di console.
- **Thundering Herd di Frontend**: Jika sebuah mutasi membatalkan 20 query sekaligus, browser akan menembakkan 20 request HTTP secara bersamaan ke backend dalam hitungan milidetik.

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Menggunakan Invalidation (`invalidateQueries`)
- Setelah operasi Create, Update, Delete (CUD) yang memengaruhi banyak tempat di aplikasi (misal menambah transaksi baru memengaruhi tabel transaksi, grafik pengeluaran, dan sisa limit kartu).
- Setelah operasi autentikasi (Logout membatalkan seluruh cache profil dan hak akses).

### Kapan Menggunakan Direct Cache Set (`setQueryData`)
- Pembaruan nilai tunggal yang sederhana di mana backend mengembalikan objek utuh hasil update. Ini menghemat 1 request HTTP round-trip tambahan.

## 8. Metrik Kunci & Observabilitas

- **TanStack Query DevTools (Invalidate State)**: Amati perubahan warna label query key saat aksi mutasi selesai. Label harus langsung berubah dari hijau (*Fresh*) menjadi kuning (*Stale*) atau biru (*Fetching*).
- **Post-Mutation Network Calls**: Periksa tab Network DevTools setelah menekan tombol simpan form. Pastikan hanya endpoint yang relevan yang di-refetch, bukan seluruh endpoint aplikasi.

## 9. Studi Kasus Industri Nyata

### Penanganan Cache Stale pada Keranjang Belanja Shopify Checkout

- **Tantangan Rekayasa Riil**:
  Saat pembeli menambahkan kupon diskon (misal: "DISKON50") di halaman checkout, total harga harus langsung berkurang. Namun, jika pembeli kemudian menghapus salah satu barang dari keranjang belanja di tab lain, subtotal kupon diskon menjadi tidak valid karena syarat minimum belanja tidak terpenuhi. Jika cache harga checkout tidak dibatalkan seketika, pembeli dapat membayar dengan harga diskon yang keliru.
- **Implementasi Solusi Industri**:
  Shopify menerapkan pola **Coordinated Cache Invalidation Matrix**:
  1. Setiap mutasi pada keranjang (`cart_updated`) memancarkan event invalidasi terpusat.
  2. Cache kupon, ongkos kirim, dan subtotal secara otomatis dibatalkan secara bersamaan.
  3. Sistem mewajibkan revalidasi harga final ke server sebelum tombol "Bayar Sekarang" diizinkan untuk diklik.
- **Dampak & Hasil Sistem**:
  - Kerugian finansial akibat transaksi diskon tidak sah tereliminasi sepenuhnya.
  - Pengalaman pembeli tetap mulus dengan kalkulasi rincian biaya yang selalu akurat.

## 10. Fakta Menarik & Gotchas Production

1. **Jebakan `exact: true` vs `exact: false`**:
   Secara default, di TanStack Query, `queryClient.invalidateQueries({ queryKey: ['todos'] })` bersifat fuzzy match (`exact: false`). Perintah ini akan membatalkan SEMUA query yang diawali dengan `'todos'`. Jika Anda HANYA ingin membatalkan query daftar tanpa menyentuh query detail, gunakan opsi: `{ queryKey: ['todos'], exact: true }`.
2. **Invalidasi Menghormati Komponen yang Sedang Aktif**:
   TanStack Query sangat cerdas: jika Anda membatalkan query yang komponennya saat ini sedang TIDAK tampak di layar (status `inactive`), TanStack Query TIDAK akan langsung mengirimkan request jaringan saat itu juga. Ia hanya menandai data tersebut sebagai *stale*, dan baru akan menembak request jaringan saat komponen tersebut dibuka kembali oleh pengguna di kemudian hari.

## 11. Rangkuman Singkat

1. **Disiplin Kebenaran Data**: Cache invalidation memastikan data di layar tidak menyajikan data usang setelah mutasi terjadi.
2. **Kutipan Klasik**: Menjadi salah satu tantangan tersulit dalam ilmu komputer karena memerlukan koordinasi presisi antara kapan data berubah dan apa saja yang terpengaruh.
3. **Invalidasi Terarah**: Rancang susunan `queryKey` yang hierarkis untuk membatalkan cache secara spesifik tanpa perlu menghapus seluruh memori aplikasi.
4. **Pilih Invalidate vs Set**: Gunakan `invalidateQueries` untuk konsistensi aman, atau gunakan `setQueryData` untuk efisiensi jaringan nol round-trip.
