# Type Safety

## 1. Skenario Sistem Nyata

Sebuah aplikasi fintech investasi saham menerima data harga portofolio dari backend REST API:
```typescript
interface PortfolioResponse {
  balance: number;
  currency: string;
}
```

Frontend merender saldo pengguna dengan pemformatan mata uang:
`const formatted = data.balance.toFixed(2);`

Suatu hari, tim backend melakukan perubahan struktur respons tanpa pemberitahuan. Nilai `balance` yang semula bertipe angka (`number`), kini diubah menjadi format objek bersarang demi mendukung fitur multi-rekening:
`{ balance: { amount: 1500000, locked: 200000 } }`

TypeScript di frontend sama sekali tidak mengeluarkan error saat build karena tipe data `PortfolioResponse` di-hardcode secara statis tanpa validasi di gerbang data.

Ketika pengguna membuka aplikasi di production, eksekusi kode mencapai `data.balance.toFixed(2)` dan langsung memicu error fatal:
`Uncaught TypeError: data.balance.toFixed is not a function`

Seluruh halaman portofolio hancur (*crash*). Peristiwa ini membuktikan bahwa tipe statis saat build-time saja tidak cukup tanpa **Type Safety Menyeluruh (Compile-time + Runtime Schema Validation)**.

## 2. Apa Itu dan Buat Apa?

Type Safety (Keamanan Tipe) adalah jaminan pada bahasa pemrograman bahwa program tidak akan mengeksekusi operasi yang tidak sah terhadap suatu tipe data (misal: memanggil method string pada data bertipe boolean, atau membaca properti dari nilai `null`/`undefined`).

Di ekosistem frontend modern, Type Safety mencakup dua pilar:
1. **Compile-Time Type Safety (TypeScript)**: Pemeriksaan tipe statis saat developer mengetik kode di editor dan saat proses build. Mencegah kesalahan pengetikan properti (*typos*), memastikan pemanggilan fungsi dengan parameter yang benar, dan memudahkan autocompletion (*IntelliSense*).
2. **Runtime Type Safety (Schema Validation - Zod / Valibot)**: Pemeriksaan tipe saat aplikasi sedang aktif berjalan di browser pengguna. Bertindak sebagai "satpam" di perbatasan eksternal aplikasi (input formulir, parameter URL, dan respons API jaringan) untuk memastikan data yang masuk cocok persis dengan skema yang diharapkan.

## 3. Kenapa Harus Pakai Ini?

- **Menghilangkan Bug Klasik Terbesar di JavaScript**: Sebagian besar bug JavaScript di production adalah varian dari `Cannot read properties of undefined` atau tipe data tak terduga. Type safety melenyapkan kategori bug ini sejak dini.
- **Refactoring Skala Besar Tanpa Cemas**: Ketika Anda mengubah nama properti pada model data (misal `userId` menjadi `accountId`), compiler TypeScript akan langsung menandai setiap baris kode di seluruh repositori yang masih menggunakan nama lama.
- **Dokumentasi Kontrak yang Akurat**: Interface dan Type bertindak sebagai kontrak pasti antar modul dan antar developer dalam tim tanpa perlu membaca berkas implementasi yang panjang.

## 4. Cara Kerjanya

Berikut adalah diagram bagaimana arsitektur Type Safety terintegrasi melindungi aplikasi dari gerbang data jaringan hingga ke komponen antarmuka pengguna:

```
[ Backend REST API / External Source ]
                   |
     (Mengembalikan payload JSON mentah)
                   |
                   v
[ RUNTIME VALIDATION BOUNDARY (Zod Schema) ]
UserSchema.safeParse(jsonResponse)
                   |
       +-----------+-----------+
       |                       |
 (Gagal Validasi)       (Lolos Validasi)
       |                       |
       v                       v
 Catat error log &        Data dipastikan aman 100%
 Tampilkan fallback UI    Tipe otomatis diinferensikan:
 (Cegah crash aplikasi)   type User = z.infer<typeof UserSchema>
                               |
                               v
               [ COMPILE-TIME TYPESCRIPT CHECK ]
               Komponen React: <UserProfile user={user} />
               - Compiler menjamin properti user.name ada
               - Autocomplete aktif di editor developer
                               |
                               v
                     [ APLIKASI AMAN 100% ]
```

## 5. Cuplikan Logika Inti

### Validasi Runtime dan Inferensi Tipe Otomatis (Zod + TypeScript)

Jangan pernah menggunakan *Type Assertion* palsu (`const data = await res.json() as User`). Gunakan parser skema runtime:

```typescript
// src/schemas/portfolioSchema.ts
import { z } from 'zod';

// 1. Definisikan Schema Validasi Runtime
export const PortfolioSchema = z.object({
  id: z.string().uuid(),
  balance: z.number().positive(),
  currency: z.enum(['IDR', 'USD', 'SGD']),
  lastUpdated: z.string().datetime(),
  isVerified: z.boolean().default(false),
});

// 2. Inferensikan Tipe Statis TypeScript secara Otomatis dari Schema
// (Single Source of Truth: Tidak perlu mengetik ulang interface manual!)
export type Portfolio = z.infer<typeof PortfolioSchema>;
```

### Fetching Data Aman dengan Penanganan Error Terkendali

```typescript
// src/api/portfolioApi.ts
import { PortfolioSchema, Portfolio } from '../schemas/portfolioSchema';

export async function fetchUserPortfolio(userId: string): Promise<Portfolio> {
  const response = await fetch(`/api/v1/users/${userId}/portfolio`);

  if (!response.ok) {
    throw new Error(`Gagal memuat API: HTTP ${response.status}`);
  }

  const rawJson = await response.json();

  // 3. safeParse memverifikasi apakah bentuk data sesuai kontrak skema
  const parseResult = PortfolioSchema.safeParse(rawJson);

  if (!parseResult.success) {
    // Data backend korup / berubah bentuk!
    console.error('Penyimpangan Kontrak API:', parseResult.error.format());
    throw new Error('Data portofolio yang diterima dari server tidak valid.');
  }

  // parseResult.data dijamin 100% bertipe Portfolio yang aman
  return parseResult.data;
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

| Kelebihan | Kekurangan / Risiko |
| :--- | :--- |
| Mencegah error runtime tipe data (*TypeError*) sebelum mencapai browser pengguna | Menambah sedikit overhead komputasi parsing data di sisi client browser |
| Memberikan pengalaman coding yang luar biasa dengan autocompletion dan refactor tools | Memerlukan kurva belajar konsep lanjutan (Generics, Discriminated Unions, Type Narrowing) |
| Menjadi jembatan kontrak data yang valid antara tim frontend dan tim backend | Menambah sedikit ukuran bundle library (~12KB untuk Zod, atau ~1KB untuk Valibot) |

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Wajib Menggunakan Type Safety Penuh
- Setiap kali membaca data dari batas luar aplikasi: respons HTTP API, input formulir pengguna, parameter query URL, data dari `localStorage` atau WebSockets.
- Basis kode enterprise berskala menengah hingga besar dengan banyak kontributor.
- Fitur transaksi finansial, checkout, atau manipulasi data penting.

### Kapan Boleh Menggunakan Tipe Sederhana
- Script utilitas satu kali pakai (*one-off scripts*) atau prototipe eksperimen UI kilat yang akan dibuang dalam beberapa jam.

## 8. Metrik Kunci & Observabilitas

1. **Strict Mode Enablement**: Pastikan `tsconfig.json` menyetel `"strict": true`, `"noImplicitAny": true`, dan `"strictNullChecks": true`. Jangan pernah mematikan mode strict di proyek baru!
2. **`any` Usage Count**: Jalankan aturan linter `@typescript-eslint/no-explicit-any` untuk melarang atau meminimalkan penggunaan tipe `any`. Targetkan angka `any` bernilai 0 di production.
3. **Runtime Schema Violation Rate**: Kirim metrik telemetri ke monitoring (Sentry) ketika `safeParse` gagal memvalidasi data API. Lonjakan metrik ini menjadi peringatan dini bahwa backend telah merilis breaking change pada kontrak API.

## 9. Studi Kasus Industri Nyata

Sebuah platform pemesanan tiket bioskop mengalami insiden saat peluncuran film blockbuster. 

Pada halaman pemilihan kursi bioskop, developer menggunakan type assertion mentah:
`const seats = (await res.json()) as Seat[];`

Namun saat traffic membludak, server backend mengalami kelebihan beban dan mengembalikan respons error berformat HTML: `<html><body>502 Bad Gateway</body></html>`.

Karena tidak ada validasi runtime, frontend memperlakukan string HTML tersebut sebagai objek JSON. Saat kode mencoba menjalankan `seats.filter(...)`, aplikasi langsung melempar unhandled exception dan layar pengguna menjadi blank putih tanpa petunjuk tombol reload.

Perbaikan arsitektural:
1. Mengganti seluruh type assertion dengan validasi skema runtime menggunakan `Zod`.
2. Ketika respons bukan JSON atau tidak sesuai skema kursi, `safeParse` menangkap kegagalan tersebut secara elegan dan merender komponen fallback: *"Server pemesanan sedang sibuk, silakan klik tombol Coba Lagi"*.
3. Pengalaman pengguna terselamatkan dari crash fatal.

## 10. Fakta Menarik & Gotchas Production

- **Tipe Hilang Saat Runtime**: Ingat bahwa seluruh kode TypeScript (`interface`, `type`, `as`) **dihapus total (*erased*)** saat proses build ke JavaScript. Di browser, kode TypeScript murni tidak bisa mengecek tipe data. Inilah alasan mengapa library validasi runtime seperti Zod mutlak dibutuhkan.
- **Bahaya Type Assertion (`as Type`)**: Menggunakan kata kunci `as` pada dasarnya adalah memberi tahu compiler: *"Percayalah padaku, aku tahu apa yang kulakukan"*. Seringkali developer salah, dan compiler berhenti memberikan perlindungan. Hindari `as` sebisa mungkin; gunakan *Type Narrowing* atau *Type Guards* (`typeof`, `instanceof`, atau `in`).
- **Discriminated Unions**: Pola terbaik untuk memodelkan state antarmuka yang kompleks:
  ```typescript
  type RequestState =
    | { status: 'idle' }
    | { status: 'loading' }
    | { status: 'success'; data: Product[] }
    | { status: 'error'; error: string };
  ```
  TypeScript akan otomatis mencegah Anda membaca `state.data` jika `state.status` masih bernilai `'loading'`.

## 11. Rangkuman Singkat

Type Safety modern adalah integrasi harmonis antara pemeriksaan statis saat build-time via TypeScript dan validasi gerbang data runtime via skema Zod/Valibot. Kombinasi ini menjamin bahwa seluruh data yang mengalir di dalam aplikasi terbukti valid, melenyapkan error tipe data tak terduga, dan memungkinkan refactoring besar-besaran dengan percaya diri.
