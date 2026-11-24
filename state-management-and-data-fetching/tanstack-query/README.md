# TanStack Query (React Query)

## 1. Skenario Sistem Nyata

Pada aplikasi dashboard telemedisin rumah sakit, dokter membuka rekam medis pasien. Di dalam halaman tersebut terdapat 3 widget independen:
1. Widget **Ringkasan Riwayat Pasien** di sebelah kiri.
2. Widget **Daftar Resep Obat Aktif** di sebelah kanan.
3. Widget **Jadwal Janji Temu Terakhir** di bagian atas.

Setiap widget dibangun oleh komponen yang berbeda dan masing-masing komponen membutuhkan data profil dasar pasien dari endpoint yang sama: `GET /api/patients/101`.

Tanpa TanStack Query:
- Ketiga komponen secara serentak mengeksekusi `fetch('/api/patients/101')` di dalam hook `useEffect` masing-masing saat halaman terbuka.
- Browser mengirimkan **3 request HTTP identik secara bersamaan** ke server backend untuk mengambil data yang sama persis!
- Jika dokter berpindah tab dan kembali lagi, 3 request tersebut ditembakkan ulang dari nol, membebani server backend dan kuota API.

Solusi: Menggunakan **TanStack Query (React Query)**. TanStack Query bertindak sebagai pengatur lalu lintas data cerdas (*Asynchronous State Manager & Cache*). Ketiga komponen cukup mendeklarasikan `useQuery({ queryKey: ['patient', 101], ... })`. TanStack Query secara otomatis melakukan **Request Deduplication** (hanya 1 request HTTP yang dikirimkan ke server, hasilnya dibagikan ke ketiga komponen), menyimpan data di memory cache, dan mengelola sinkronisasi latar belakang secara otomatis.

## 2. Apa Itu dan Buat Apa?

TanStack Query (sebelumnya dikenal sebagai *React Query*) adalah pustaka pengelola data asinkron (*data-fetching and state management library*) yang bertindak sebagai mesin caching tangguh untuk aplikasi web (tersedia untuk React, Vue, Svelte, Solid, dan Angular).

Alih-alih memaksa developer mengelola siklus hidup data asinkron secara manual menggunakan `useState` dan `useEffect`, TanStack Query menyediakan pendekatan deklaratif yang menangani:
- **Pemuatan Data Otomatis**: Menyediakan flag `isLoading`, `isPending`, `isError`, `isSuccess`.
- **Deduplikasi Request**: Menggabungkan pemanggilan endpoint yang sama menjadi 1 request tunggal (*in-flight deduplication*).
- **Auto Caching & Garbage Collection**: Menyimpan data di memori dan membersihkannya secara otomatis saat tidak terpakai.
- **Background Revalidation**: Memperbarui data secara hening saat pengguna kembali membuka tab browser (*Window Focus*) atau saat koneksi internet pulih (*Network Reconnect*).
- **Pagination & Infinite Scrolling**: Dukungan bawaan untuk data bertahap.

## 3. Kenapa Harus Pakai Ini?

TanStack Query adalah revolusi terbesar dalam arsitektur data-fetching frontend modern:
1. **Mengeliminasi Ratusan Baris Kode `useEffect`**: Tidak ada lagi kode boilerplate `let isMounted = true` untuk mencegah memory leak.
2. **Pengalaman Pengguna Instan Tanpa Spinner (*Stale-While-Revalidate*)**: Menampilkan data lama dari cache terlebih dahulu dalam 0 milidetik, lalu memperbarui data di latar belakang.
3. **Penyelarasan Server & Client Mulus**: Menyediakan API *Mutations* dengan fitur pembatalan cache otomatis (*Invalidation*) dan *Optimistic Updates*.

## 4. Cara Kerjanya (Diagram Alur Request Deduplication & Cache Lifecycle)

```
Komponen A           Komponen B           Komponen C
    |                    |                    |
    +---- useQuery ------>+---- useQuery ------>+---- useQuery
          ['patient', 101]      ['patient', 101]      ['patient', 101]
                         |
                         v
       +------------------------------------+
       |        TANSTACK QUERY ENGINE       |
       +------------------------------------+
                         |
         1. Deteksi: Ada 3 pemanggil kunci identik!
         2. Eksekusi HANYA 1 REQUEST JARINGAN:
            GET /api/patients/101
                         |
                         v
                  [ SERVER BACKEND ]
                         |
                         | 200 OK: Data Pasien
                         v
       +------------------------------------+
       |   Simpan Data di Memory Cache RAM  |
       |   Key: ['patient', 101]            |
       +------------------------------------+
             |           |           |
             |           |           | Bagikan hasil yang sama
             v           v           v
        Komponen A  Komponen B  Komponen C
        *KETIGANYA MENERIMA DATA SEKALIGUS DALAM 1 REQUEST TUNGGAL!*
```

## 5. Cuplikan Logika Inti

### Penggunaan Query dan Mutasi Data

**1. Mengambil Data dengan `useQuery`:**
```typescript
import { useQuery } from "@tanstack/react-query";

interface Patient {
  id: number;
  name: string;
  condition: string;
}

export function usePatientDetails(patientId: number) {
  return useQuery({
    queryKey: ["patient", patientId],
    queryFn: async (): Promise<Patient> => {
      const response = await fetch(`/api/patients/${patientId}`);
      if (!response.ok) {
        throw new Error("Gagal mengambil data rekam medis");
      }
      return response.json();
    },
    staleTime: 1000 * 60 * 3, // Data dianggap segar selama 3 menit
  });
}
```

**2. Mengubah Data dan Membatalkan Cache dengan `useMutation`:**
```typescript
import { useMutation, useQueryClient } from "@tanstack/react-query";

export function useUpdatePatientCondition() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: async ({ id, condition }: { id: number; condition: string }) => {
      const res = await fetch(`/api/patients/${id}`, {
        method: "PATCH",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ condition }),
      });
      return res.json();
    },
    // Saat mutasi di server berhasil:
    onSuccess: (updatedData, variables) => {
      // BATALKAN CACHE: Suruh TanStack Query untuk me-refetch data pasien terkait secara otomatis!
      queryClient.invalidateQueries({ queryKey: ["patient", variables.id] });
    },
  });
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan
- **Pengurangan Beban Jaringan Ekstrem**: Request deduplication dan caching memangkas beban server hingga 60%.
- **Penanganan Status Lengkap**: Mengeliminasi kebutuhan membuat variabel boolean status error dan loading manual.
- **Mendukung Mode Offline**: Dapat dipadukan dengan persistensi IndexedDB untuk aplikasi yang tetap dapat dibuka saat tidak ada sinyal internet.

### Kekurangan & Tantangan
- **Perlu Desain Query Key yang Rapi**: Jika struktur `queryKey` Anda berantakan (misal lupa menyertakan ID atau parameter paginasi), data yang disajikan bisa tertukar atau gagal ter-refresh.
- **Ukuran Bundle**: Menambahkan dependensi eksternal sekitar 12 KB (gzipped) ke dalam aplikasi.

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Menggunakan TanStack Query
- Seluruh aplikasi web yang berkomunikasi dengan API backend (REST, GraphQL, tRPC, Supabase, Firebase).
- Aplikasi dengan banyak halaman yang menampilkan data server yang sama.
- Fitur pencarian data dinamis, filter katalog, dan infinite scrolling.

### Kapan Jangan Menggunakan TanStack Query
- Mengelola status murni antarmuka lokal (misal: modal dialog terbuka/tutup, status checklist — gunakan **useState** / **Zustand**).
- Komunikasi WebSocket real-time murni dua arah di mana server melakukan push data terus-menerus (meskipun TanStack Query dapat dipadukan dengan WebSocket via cache manipulation manual).

## 8. Metrik Kunci & Observabilitas

- **TanStack Query DevTools (Floating Widget)**:
  - Buka widget DevTools di browser: Anda dapat melihat seluruh query yang aktif di memori.
  - Periksa status warna: Hijau (*Fresh*), Kuning (*Stale*), Biru (*Fetching*), Abu-abu (*Inactive*).
  - Anda dapat mengklik tombol "Refetch" atau "Invalidate" secara manual untuk menguji reaksi UI.
- **Network Tab (Fetch Count)**: Pastikan saat pengguna bernavigasi bolak-balik antara halaman A dan B, request HTTP tidak ditembakkan berulang kali jika data masih dalam jendela `staleTime`.

## 9. Studi Kasus Industri Nyata

### Transformasi Pengalaman Belanja di GitHub Web (Pull Requests View)

- **Tantangan Rekayasa Riil**:
  Halaman Pull Request di GitHub memuat tab percakapan (*Conversation*), tab perubahan kode (*Files Changed*), dan status checks CI/CD. Pengguna sering berpindah-pindah antara tab kode dan percakapan puluhan kali. Pada sistem lama, setiap kali tab diklik, GitHub menampilkan skeleton loading dan mengunduh ulang data komentar.
- **Implementasi Solusi Industri**:
  GitHub mengadopsi prinsip **Query Caching Stale-While-Revalidate**:
  1. Data percakapan disimpan di memory query cache.
  2. Saat pengguna kembali dari tab kode ke tab percakapan, komentar lama langsung muncul seketika (0 milidetik).
  3. Query engine memeriksa pembaruan komentar baru di latar belakang tanpa mengganggu pembaca.
- **Dampak & Hasil Sistem**:
  - Waktu tunggu visual (*Perceived Latency*) turun ke nol milidetik saat berpindah tab.
  - Beban server API GitHub berkurang hingga 35%.

## 10. Fakta Menarik & Gotchas Production

1. **Jebakan Default `staleTime: 0`**:
   Banyak pemula kaget: *"Saya sudah pakai TanStack Query, kenapa setiap komponen di-mount tetap ada request ke server?"*. Jawabannya: secara default, `staleTime` bernilai `0`. Artinya, data langsung dianggap *stale* (basi) begitu selesai diunduh. Jika Anda ingin data di-cache selama 5 menit tanpa ada refetch baru saat berpindah halaman, atur eksplisit: `staleTime: 1000 * 60 * 5`.
2. **Query Keys Bersifat Hierarkis**:
   Array query key bekerja seperti folder: `['todos', 'list', { filter: 'completed' }]`. Jika Anda memanggil `queryClient.invalidateQueries({ queryKey: ['todos'] })`, maka SELURUH query yang diawali dengan kata `'todos'` (baik list, detail, maupun filter) akan otomatis ikut dibatalkan dan diperbarui!

## 11. Rangkuman Singkat

1. **Raja Server State**: TanStack Query adalah pustaka manajemen data asinkron dan cache API standar industri modern.
2. **Deduplikasi Bawaan**: Mencegah request HTTP ganda ke endpoint yang sama dari komponen-komponen berbeda.
3. **Stale-While-Revalidate**: Menyajikan data cache seketika sembari menyinkronkan data baru di latar belakang.
4. **Kuasai Query Key**: Rancang susunan array `queryKey` secara hierarkis untuk mempermudah invalidasi data yang presisi.
