# Server State

## 1. Skenario Sistem Nyata

Pada aplikasi perbankan atau investasi saham, seorang pengguna membuka halaman portofolio saham. Aplikasi mengambil data harga saham saat ini dari backend API dan menyimpannya di Redux Store global: `currentPrice: 150.00`.

Pengguna kemudian meninggalkan tab browser tersebut terbuka selama 45 menit untuk rapat kerja. Saat kembali ke meja:
- Tampilan di layar laptop pengguna **masih menampilkan angka 150.00**, padahal harga saham di pasar bursa sebenarnya sudah anjlok ke 120.00!
- Pengguna yang tidak menyadari bahwa data di layarnya sudah basi (*stale data*) menekan tombol "Beli", menghasilkan transaksi keuangan yang salah dan komplain hukum ke bank.

Kesalahan fundamental developer: Memperlakukan data yang berasal dari server seolah-olah itu adalah data lokal milik frontend. Developer lupa bahwa **frontend tidak pernah memiliki data server; frontend hanya meminjam snapshot data pada waktu tertentu**.

Solusi: Memahami paradigma **Server State**. Menggunakan pustaka pengelola server cache (seperti TanStack Query / SWR / RTK Query) yang memiliki kecerdasan *Auto-Refetch on Window Focus*, *Stale-Time Management*, deduping request, dan sinkronisasi cache otomatis.

## 2. Apa Itu dan Buat Apa?

Server State adalah status atau data yang aslinya **dimiliki, disimpan, dan dikontrol oleh server backend** (database jarak jauh), di mana frontend hanya menyimpan salinan sementara (*cached snapshot*) di memori browser untuk kebutuhan visualisasi.

Karakteristik unik Server State yang membedakannya secara radikal dari Client State biasa:
- **Asinkron Sepenuhnya**: Membutuhkan waktu jaringan untuk membaca dan menulis data (*latency*).
- **Dimiliki oleh Pihak Lain**: Data di server dapat diubah oleh orang lain kapan saja tanpa sepengetahuan browser kita (misal pengguna lain membeli tiket terakhir).
- **Rentan Kedaluwarsa (*Stale*)**: Begitu data selesai diunduh dan tiba di browser, data tersebut sudah berpotensi usang.

Fungsi utama memahami Server State:
- Menggantikan ratusan baris kode manual `isLoading`, `isError`, dan `useEffect` dengan deklarasi cache otomatis.
- Mengelola siklus hidup data: *Fetching*, *Caching*, *Deduplication*, *Background Revalidation*, dan *Garbage Collection*.

## 3. Kenapa Harus Pakai Ini?

Sebelum konsep Server State dipisahkan dari Client State, komunitas developer frontend memasukkan seluruh hasil fetch API ke dalam Redux Store atau Vuex:
- Developer harus menulis Action types, Reducer cases, dan dispatch manual hanya untuk menangani status loading dan error.
- Tidak ada mekanisme bawaan untuk membersihkan cache memori ketika data sudah tidak digunakan.
- Tidak ada sinkronisasi otomatis saat koneksi internet sempat terputus (*reconnect*) atau saat tab browser kembali aktif (*window focus*).

Memisahkan Server State ke library khusus (seperti TanStack Query) memangkas hingga **50% kode boilerplate** pada aplikasi frontend modern.

## 4. Cara Kerjanya (Diagram Alur Siklus Hidup Server State)

Visualisasi penanganan data server dengan status kesegaran cache (*Fresh vs Stale*):

```
Komponen Meminta Data: useQuery(['user-profile'])
                         |
                         v
+---------------------------------------------------------------+
|                      CACHE MANAGER LOKAL                      |
+---------------------------------------------------------------+
|  Apakah data sudah ada di cache RAM?                          |
|                                                               |
|  [TIDAK] --> Tampilkan status isLoading: true                 |
|              Kirim request HTTP ke server via jaringan...     |
|              Simpan response di cache -> Tampilkan ke UI!     |
|                                                               |
|  [YA] -----> Periksa: Apakah data masih FRESH (staleTime)?   |
|                                                               |
|              [FRESH] --> Langsung kembalikan data instan!     |
|                          (Nol request jaringan baru)          |
|                                                               |
|              [STALE] --> Tampilkan data lama SEGERA ke layar  |
|                          (Pengguna tidak melihat spinner!)    |
|                          TETAPI jalankan BACKGROUND FETCH...  |
|                          Begitu data baru tiba, perbarui UI!  |
+---------------------------------------------------------------+
```

## 5. Cuplikan Logika Inti

### Perbandingan Pendekatan Lama vs Paradigma Server State Modern

**Cara Lama yang Melelahkan (Manual Client State):**
```javascript
// Cara lama: Menulis 30 baris boilerplate di setiap komponen
function OldUserProfile() {
  const [data, setData] = useState(null);
  const [isLoading, setIsLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    let isMounted = true;
    fetch("/api/profile")
      .then((res) => res.json())
      .then((json) => {
        if (isMounted) { setData(json); setIsLoading(false); }
      })
      .catch((err) => {
        if (isMounted) { setError(err); setIsLoading(false); }
      });
    return () => { isMounted = false; };
  }, []);

  if (isLoading) return <div>Memuat...</div>;
  if (error) return <div>Error!</div>;
  return <h1>Halo, {data.name}</h1>;
}
```

**Cara Modern Menggunakan Paradigma Server State (TanStack Query):**
```javascript
import { useQuery } from "@tanstack/react-query";

// 3 baris kode tangguh dengan caching, deduping, dan background refetch otomatis!
function ModernUserProfile() {
  const { data, isLoading, isError } = useQuery({
    queryKey: ["profile"],
    queryFn: () => fetch("/api/profile").then((res) => res.json()),
    staleTime: 1000 * 60 * 5, // Data dianggap segar selama 5 menit
  });

  if (isLoading) return <div>Memuat...</div>;
  if (isError) return <div>Error!</div>;
  return <h1>Halo, {data.name}</h1>;
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan
- **Eliminasi Kode Boilerplate**: Menghemat ribuan baris kode state loading, error, dan caching manual.
- **Request Deduplication Otomatis**: Jika 3 komponen di halaman yang sama meminta data `/api/profile` secara bersamaan, library hanya mengirim **1 request HTTP tunggal** ke server.
- **Pengalaman Pengguna Tanpa Spinner (*Optimistic Feel*)**: Pengguna langsung melihat data dari cache saat navigasi halaman, sementara sinkronisasi data baru terjadi hening di latar belakang.

### Kekurangan
- **Kurva Pembelajaran Konsep Caching**: Developer wajib memahami perbedaan antara `staleTime` (kapan data harus di-fetch ulang) dan `gcTime / cacheTime` (kapan data yang tidak dipakai dibuang dari memori).
- **Ukuran Bundle Eksternal**: Membutuhkan pustaka dependensi tambahan (~12 KB gzipped untuk TanStack Query).

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Pakai
- Seluruh pengambilan data dari REST API, GraphQL, atau RPC backend (daftar produk, profil, riwayat pesanan).
- Data yang sering mengalami mutasi dan membutuhkan pembaruan cache otomatis.
- Fitur yang membutuhkan paginasi data atau *infinite scroll* yang mulus.

### Kapan Jangan Pakai
- Status murni antarmuka pengguna lokal (misal: modal buka/tutup, status checklist checkbox — gunakan **Local State**).
- State global yang tidak berasal dari server backend (misal: daftar item yang dipilih untuk diekspor ke PDF — gunakan **Global State**).

## 8. Metrik Kunci & Observabilitas

- **Network Request Waterfall**: Amati tab Network di DevTools. Paradigma server state yang sehat akan mengeliminasi request ganda yang identik dalam satu halaman.
- **TanStack Query DevTools**:
  - Amati status query: `fresh` (hijau), `stale` (kuning), `fetching` (biru), `inactive` (abu-abu).
  - Pastikan query yang sudah tidak digunakan berpindah ke status `inactive` dan terhapus setelah batas waktu `gcTime`.

## 9. Studi Kasus Industri Nyata

### Penurunan Beban Server Backend pada Trello Web

- **Tantangan Rekayasa Riil**:
  Pengguna Trello sering kali membuka belasan tab kartu pekerjaan di browser mereka. Pada arsitektur lama, setiap kali tab dibuka kembali, aplikasi secara buta menembak lusinan API ke backend untuk memuat ulang daftar komentar, label, dan lampiran. Server backend Trello dibanjiri jutaan request read yang redundan.
- **Implementasi Solusi Industri**:
  Trello mengadopsi layer **Server State Cache Terdistribusi**:
  1. Menerapkan `staleTime: 2 Menit` pada data kartu.
  2. Mengaktifkan *Window Focus Refetching* secara terkontrol (hanya jika data sudah benar-benar stale).
  3. Menggunakan deduping cerdas untuk request yang sedang berjalan (*in-flight request sharing*).
- **Dampak & Hasil Sistem**:
  - Trafik request read ke database backend Trello turun sebesar 45%.
  - Transisi pengguna saat berpindah-pindah antar kartu di papan terasa instan tanpa delay loading spinner.

## 10. Fakta Menarik & Gotchas Production

1. **Perbedaan Krusial: `staleTime` vs `gcTime`**:
   - `staleTime`: Menentukan **kapan data dianggap basi**. Selama data masih dalam masa staleTime, komponen baru yang membaca data tersebut TIDAK AKAN mengirim request jaringan baru. Default di React Query: `0` (langsung dianggap stale).
   - `gcTime` (*Garbage Collection Time*): Menentukan **berapa lama data yang sudah tidak digunakan disimpan di memori RAM**. Jika tidak ada komponen yang menggunakan query tersebut selama waktu gcTime, data dihapus permanen dari memori. Default: 5 menit.
2. **Auto-Refetch on Window Focus**:
   Secara default, TanStack Query akan secara otomatis menembak API di background ketika pengguna kembali membuka tab browser dari tab lain. Jika API Anda bertarif berbayar per-request (misal API Google Maps Places), matikan fitur ini: `refetchOnWindowFocus: false`.

## 11. Rangkuman Singkat

1. **Frontend Hanya Meminjam Data**: Server state adalah data milik server backend, frontend hanya menyimpan cache sementaranya.
2. **Karakteristik Asinkron**: Selalu libatkan status loading, error, latensi jaringan, dan potensi data basi.
3. **Pangkas Boilerplate**: Gantikan `useEffect` fetch manual dengan pustaka server state modern (TanStack Query / SWR).
4. **Kuasai Stale vs GC**: Atur `staleTime` untuk mengontrol frekuensi fetch jaringan, dan `gcTime` untuk manajemen memori RAM.
