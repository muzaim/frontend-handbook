# Test Coverage

## 1. Skenario Sistem Nyata

Sebuah tim pengembang frontend bangga karena dashboard CI mereka menampilkan angka **Test Coverage: 100%** berwarna hijau cerah. Manajemen merasa sangat tenang dan berasumsi bahwa aplikasi perbankan mereka bebas dari segala bug.

Namun di minggu pertama rilis ke pengguna nyata, terjadi crash masif pada modul kalkulator pinjaman. Pengguna yang memasukkan nominal pinjaman tanpa memilih jangka waktu tenor (*undefined*) mengalami layar putih blank (*Uncaught TypeError: Cannot read properties of undefined*).

Bagaimana mungkin kode dengan cakupan 100% bisa crash di production?

Setelah diaudit, unit test yang ditulis ternyata hanya berupa tes dangkal:
```typescript
test('render kalkulator', () => {
  render(<LoanCalculator />);
  // Tidak ada assertion kondisi edge case atau branch validasi sama sekali!
});
```
Baris kode memang dieksekusi oleh test runner sehingga angka metrik coverage tercatat 100%, namun tidak ada pengujian logika percabangan (*branch conditions*) atau assertion nilai output sama sekali.

Peristiwa ini menyadarkan tim akan bahaya mengejar metrik coverage semu tanpa memahami esensi pengujian kualitas kode.

## 2. Apa Itu dan Buat Apa?

Test Coverage (Cakupan Pengujian) adalah metrik pengukuran kuantitatif yang menunjukkan seberapa banyak proporsi kode sumber aplikasi yang dieksekusi ketika rangkaian pengujian otomatis (*automated test suite*) dijalankan.

Alat coverage (seperti Istanbul atau engine V8 di Vitest/Jest) menginstrumentasi kode dan mengukur 4 dimensi utama:
1. **Statement Coverage**: Persentase pernyataan kode yang dieksekusi.
2. **Branch Coverage**: Persentase percabangan logika (`if/else`, switch case, ternary operator `?:`, default parameter) yang dievaluasi ke kondisi benar (`true`) dan salah (`false`).
3. **Function Coverage**: Persentase fungsi atau metode yang dipanggil selama pengujian.
4. **Line Coverage**: Persentase baris kode fisik yang berhasil dilewati.

Tujuan utama test coverage bukan untuk mencapai angka 100%, melainkan sebagai **peta navigasi (radar)** untuk menemukan bagian kode kritis atau skenario edge case yang belum pernah disentuh oleh pengujian.

## 3. Kenapa Harus Pakai Ini?

- **Menemukan Titik Buta (Blind Spots)**: Mengidentifikasi cabang `else` atau blok `catch` penanganan error yang terlupakan untuk diuji.
- **Menegakkan Standar Kualitas di CI/CD**: Anda dapat menyetel batas ambang minimal (*coverage threshold*), misalnya mewajibkan minimal 80% coverage pada modul logika bisnis. Jika developer mengirim PR dengan kode baru tanpa tes, pipeline CI otomatis menolak merge.
- **Mendeteksi Dead Code**: Menemukan fungsi-fungsi lama warisan yang tidak pernah lagi dipanggil atau tidak bisa dijangkau oleh alur logika mana pun, sehingga aman untuk dihapus.

## 4. Cara Kerjanya

Berikut adalah diagram alur bagaimana runner pengujian menginstrumentasi kode dan menghitung persentase coverage percabangan (*branch*):

```
[ Kode Sumber Asli: authenticateUser() ]
if (password.length >= 8 && hasSpecialChar(password)) {  <-- Ada 2 Percabangan!
  return "VALID";
} else {
  return "INVALID";
}
                    |
                    v
[ Engine V8 / Istanbul Instrumenter: Memasang Pelacak Counters ]
                    |
                    v
[ Eksekusi Test Suite: test('password valid') ]
- Input: "Rahasia#1" -> Memenuhi syarat pertama (true) dan kedua (true)
                    |
                    v
+-------------------------------------------------------------+
| LAPORAN HASIL COVERAGE:                                     |
|                                                             |
| - Statement Coverage: 75%  (Baris "VALID" tereksekusi)      |
| - Branch Coverage:    50%  (Hanya cabang IF yang dites,     |
|                             cabang ELSE belum pernah dites!)|
|                                                             |
| PERINGATAN: Cabang ELSE belum teruji!                       |
+-------------------------------------------------------------+
```

## 5. Cuplikan Logika Inti

### Konfigurasi Batas Ambang Coverage di Vitest (`vitest.config.ts`)

Menetapkan aturan ketat untuk logika bisnis namun mengecualikan file konfigurasi yang tidak perlu diuji:

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    coverage: {
      provider: 'v8', // Engine native Node.js V8 (sangat cepat)
      reporter: ['text', 'json', 'html'], // Cetak tabel di terminal dan file HTML visual
      reportsDirectory: './coverage',
      
      // Berkas yang dikecualikan dari perhitungan coverage
      exclude: [
        'node_modules/**',
        'src/main.tsx',
        'src/**/*.d.ts',
        'src/mocks/**',
        'vite.config.ts',
      ],

      // Menegakkan ambang batas minimal kelulusan CI (Gagal jika di bawah batas)
      thresholds: {
        lines: 80,
        functions: 85,
        branches: 80,
        statements: 80,
        
        // Aturan ekstra ketat khusus untuk modul utilitas keuangan
        'src/utils/finance/**': {
          branches: 100,
          lines: 100,
        },
      },
    },
  },
});
```

### Membaca Laporan Coverage di Terminal

Output tabel yang dihasilkan saat menjalankan perintah `vitest run --coverage`:

```text
----------------------|---------|----------|---------|---------|-------------------
File                  | % Stmts | % Branch | % Funcs | % Lines | Uncovered Line #s 
----------------------|---------|----------|---------|---------|-------------------
All files             |   88.23 |    82.14 |   90.00 |   88.23 |                   
 src/utils            |     100 |      100 |     100 |     100 |                   
  pricing.ts          |     100 |      100 |     100 |     100 |                   
 src/features/auth    |   81.81 |    71.42 |   80.00 |   81.81 |                   
  tokenManager.ts     |   81.81 |    71.42 |   80.00 |   81.81 | 42-45, 58         
----------------------|---------|----------|---------|---------|-------------------
```
*Kolom "Uncovered Line #s" langsung memberi tahu developer baris kode nomor berapa yang belum memiliki skenario uji.*

## 6. Kelebihan dan Kekurangan (Trade-offs)

| Kelebihan | Kekurangan / Risiko |
| :--- | :--- |
| Memberikan visibilitas objektif bagian kode mana yang belum teruji | Menimbulkan rasa aman palsu jika developer menulis tes tanpa assertion bermakna |
| Mencegah penurunan standar kualitas secara otomatis via CI gating | Mengejar target 100% coverage pada kode UI sering kali membuang waktu dan menghasilkan tes rapuh |
| Memotivasi tim menulis kode yang lebih modular dan mudah diuji | Sedikit menambah durasi waktu build/test di CI untuk proses instrumentasi AST |

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Menetapkan Target Coverage Tinggi (>85-90%)
- Modul kalkulasi finansial, harga, diskon, dan perpajakan.
- Parser data dan transformer skema yang menangani data pengguna mentah.
- Library utilitas bersama (*shared utilities*) di arsitektur monorepo.
- Mesin otentikasi, enkripsi, dan penanganan hak akses token.

### Kapan Boleh Mengabaikan / Menurunkan Coverage
- Berkas deklarasi tipe TypeScript (`*.d.ts`).
- Kode integrasi pihak ketiga eksternal yang sulit dimock.
- File konfigurasi build tool atau file boilerplate entry point (`index.tsx`).

## 8. Metrik Kunci & Observabilitas

1. **Branch Coverage Ratio**: Selalu utamakan branch coverage dibanding line coverage. Branch coverage adalah indikator terbaik apakah developer telah menguji kondisi kegagalan (*error path*), bukan hanya alur sukses (*happy path*).
2. **Coverage Delta on PR**: Periksa perubahan selisih coverage pada setiap Pull Request. Pastikan PR tidak menurunkan rata-rata coverage repositori (misal menggunakan tool otomatis GitHub bot seperti Codecov atau Coveralls).
3. **HTML Visual Heatmap**: Buka berkas `coverage/index.html` di browser lokal untuk melihat sorotan warna merah pada baris-baris kode yang tidak tersentuh tes.

## 9. Studi Kasus Industri Nyata

Sebuah platform SaaS formulir survei enterprise memiliki aturan CI ketat yang mewajibkan 95% total line coverage.

Karena tertekan oleh target angka tersebut menjelang deadline rilis, para developer mulai menulis tes-tes instan:
- Merender komponen kompleks hanya dengan `<SurveyBuilder />` tanpa berinteraksi dengan tombol.
- Menguji fungsi parser dengan input kosong hanya agar baris fungsi terpanggil.

Meskipun laporan coverage mencapai 96%, tingkat kebocoran bug ke production justru meningkat 30%. Pengguna sering mengalami error saat menambahkan pertanyaan bertipe tanggal.

Perubahan strategi:
1. Menghapus target global 95% line coverage yang kaku.
2. Menggantinya dengan aturan selektif: Mewajibkan 90% **Branch Coverage** khusus pada folder `src/domain/` dan `src/utils/`.
3. Komponen visual UI diuji berbasis skenario pengguna (React Testing Library) tanpa memaksakan angka coverage baris styling.
4. Kualitas pengujian meningkat signifikan dan tingkat bug production turun hingga 65%.

## 10. Fakta Menarik & Gotchas Production

- **Hukum Goodhart**: *"Ketika suatu ukuran dijadikan target, ukuran tersebut tidak lagi menjadi ukuran yang baik."* Jika Anda memaksa developer mengejar angka 100% coverage, mereka akan menemukan cara paling mudah untuk memuaskan metrik tersebut tanpa benar-benar meningkatkan kualitas kode.
- **Provider V8 vs Istanbul**: Provider coverage `v8` di Vitest memanfaatkan compiler native internal Node.js sehingga 3-5x lebih cepat daripada `istanbul` yang harus membedah dan memodifikasi AST kode sumber secara manual.
- **Ternary Operator Tersembunyi**: Satu baris ternary seperti `const x = a ? b : c;` memiliki 2 branches. Jika Anda hanya menguji kondisi saat `a` bernilai benar, baris tersebut tercatat 100% line coverage, namun branch coverage-nya hanya 50%.

## 11. Rangkuman Singkat

Test Coverage adalah instrumen pengukur untuk mendeteksi bagian kode yang belum tersentuh pengujian otomatis. Fokuskan perhatian pada metrik branch coverage pada logika bisnis inti dan utilitas daripada terjebak dalam mitos mengejar 100% line coverage pada seluruh kode antarmuka.
