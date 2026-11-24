# Linting and Formatting

## 1. Skenario Sistem Nyata

Sebuah tim pengembang frontend yang terdiri dari 12 insinyur sedang mengembangkan platform e-commerce. Tanpa kesepakatan konfigurasi linter dan formatter otomatis, kekacauan terjadi pada setiap proses Pull Request (PR):
- Developer A menggunakan tab 2 spasi dan titik koma (*semicolon*), sedangkan Developer B menggunakan 4 spasi tanpa titik koma.
- Setiap kali Developer B mengedit 1 baris kode di file milik Developer A, git diff mencatat 300 baris perubahan semu hanya karena pemformatan spasi ulang, menutupi logika bisnis yang sebenarnya diubah.
- Yang lebih berbahaya: seorang developer lupa menyertakan variabel state di dalam array dependency `useEffect`:
  `useEffect(() => { fetchUser(userId); }, []); // Lupa memasukkan userId`
  Akibatnya, profil pengguna tidak pernah diperbarui saat berganti halaman (*stale closure bug*).

Kekacauan gaya kode dan bug closure halus ini seharusnya dieliminasi secara otomatis sebelum kode sempat meninggalkan laptop developer menggunakan kombinasi **Linting (ESLint/Biome)** dan **Formatting (Prettier)** yang dikawal oleh **Git Hooks**.

## 2. Apa Itu dan Buat Apa?

Linting dan Formatting adalah dua alat otomatisasi kualitas kode statis yang memiliki peran berbeda namun saling melengkapi:

1. **Linter (ESLint / Biome)**:
   - **Tujuan**: Menganalisis kode sumber untuk menemukan potensi bug logika, antipattern, pelanggaran aturan framework (seperti aturan React Hooks), masalah aksesibilitas, dan inkonsistensi sintaks.
   - **Pertanyaan yang Dijawab**: *"Apakah kode ini berisiko menimbulkan bug saat dijalankan?"*
2. **Formatter (Prettier / Biome)**:
   - **Tujuan**: Merapikan tata letak visual kode (panjang baris maksimal, indentasi spasi, tanda kutip tunggal vs ganda, trailing commas) secara konsisten dan otomatis tanpa mengubah logika kode.
   - **Pertanyaan yang Dijawab**: *"Bagaimana tata letak kode ini seharusnya ditampilkan agar rapi dan seragam?"*

Tujuan utamanya adalah menjaga standar basis kode yang bersih, mencegah bug logika sedini mungkin, dan menghentikan perdebatan gaya kode yang tidak produktif saat *code review*.

## 3. Kenapa Harus Pakai Ini?

- **Menegakkan Best Practices Framework Secara Otomatis**: Plugin seperti `eslint-plugin-react-hooks` secara otomatis mencegah bug *stale closures* dan aturan *Rules of Hooks* yang sulit dideteksi secara kasat mata.
- **Git Diff yang Bersih**: Seluruh anggota tim menghasilkan format kode yang identik 100%. Reviewer PR dapat fokus meninjau logika bisnis alih-alih mengomentari tanda kurung atau titik koma.
- **Otomasi Terintegrasi (Format on Save)**: Developer tidak perlu membuang waktu merapikan baris kode secara manual; cukup tekan `Cmd+S` di VS Code, kode langsung rapi seketika.
- **Garda Depan Kualitas di Pre-Commit Hook**: Menggunakan `husky` dan `lint-staged` memastikan bahwa kode yang memiliki error sintaks atau gaya berantakan tidak akan pernah bisa di-commit ke Git.

## 4. Cara Kerjanya

Berikut adalah diagram alur bagaimana siklus kerja Linting dan Formatting mengawal kode dari editor lokal hingga ke repositori pusat:

```
[ Developer Mengetik Kode di IDE / VS Code ]
                     |
                     |-- 1. Tekan Simpan (Cmd+S)
                     v
[ Formatter: Prettier / Biome ]
  - Rapikan indentasi (2 spaces)
  - Bungkus baris > 100 karakter
  - Standardisasi tanda kutip tunggal ('...')
                     |
                     v
[ Linter: ESLint ]
  - Periksa aturan hooks (react-hooks/exhaustive-deps)
  - Periksa aturan variabel tak terpakai (no-unused-vars)
  - Tampilkan garis bawah merah jika ada potensi bug
                     |
                     |-- 2. Developer jalankan git commit
                     v
[ Git Pre-Commit Hook: Husky + lint-staged ]
  - Hanya periksa berkas yang masuk area stage (git add)
  - Jalankan eslint --max-warnings=0
                     |
         +-----------+-----------+
         |                       |
   (Ada Error Linter)     (Semua Lolos)
         |                       |
         v                       v
   [ COMMIT DIBLOKIR ]    [ COMMIT BERHASIL ]
   Developer wajib fix           |
                                 v
                     [ CI Pipeline Gating (GitHub Actions) ]
                     Verifikasi akhir sebelum merge ke branch main
```

## 5. Cuplikan Logika Inti

### 1. Konfigurasi ESLint Modern Flat Config (`eslint.config.js`)

Sejak ESLint v9, konfigurasi beralih ke format *Flat Config*:

```javascript
// eslint.config.js
import js from '@eslint/js';
import tseslint from 'typescript-eslint';
import reactPlugin from 'eslint-plugin-react';
import reactHooks from 'eslint-plugin-react-hooks';
import jsxA11y from 'eslint-plugin-jsx-a11y';
import prettierConfig from 'eslint-config-prettier';

export default tseslint.config(
  // Terapkan konfigurasi dasar yang disarankan
  js.configs.recommended,
  ...tseslint.configs.recommendedTypeChecked,
  {
    languageOptions: {
      parserOptions: {
        project: ['./tsconfig.json'],
        tsconfigRootDir: import.meta.dirname,
      },
    },
    plugins: {
      react: reactPlugin,
      'react-hooks': reactHooks,
      'jsx-a11y': jsxA11y,
    },
    rules: {
      // Wajibkan aturan React Hooks (mencegah bug rendering)
      ...reactHooks.configs.recommended.rules,
      'react-hooks/exhaustive-deps': 'error',

      // Larang variabel yang tidak digunakan
      '@typescript-eslint/no-unused-vars': ['error', { argsIgnorePattern: '^_' }],

      // Larang penggunaan tipe 'any' eksplisit
      '@typescript-eslint/no-explicit-any': 'error',
    },
  },
  // Matikan semua aturan ESLint yang berkonflik dengan Prettier
  prettierConfig
);
```

### 2. Konfigurasi Prettier (`.prettierrc`)

```json
{
  "semi": true,
  "singleQuote": true,
  "tabWidth": 2,
  "trailingComma": "es5",
  "printWidth": 100,
  "arrowParens": "always"
}
```

### 3. Konfigurasi Otomasi Git Hook (`package.json`)

Menggunakan `lint-staged` agar linter hanya memeriksa berkas yang baru diubah, bukan memindai seluruh proyek dari awal yang memakan waktu lama:

```json
{
  "scripts": {
    "prepare": "husky",
    "lint": "eslint . --max-warnings=0",
    "format:check": "prettier --check .",
    "format:write": "prettier --write ."
  },
  "lint-staged": {
    "*.{ts,tsx}": [
      "prettier --write",
      "eslint --fix"
    ],
    "*.{json,md,css}": [
      "prettier --write"
    ]
  }
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

| Kelebihan | Kekurangan / Risiko |
| :--- | :--- |
| Mencegah bug fatal (seperti dependency array kosong pada React hooks) sebelum dijalankan | Konfigurasi awal flat config ESLint dan integrasi TypeScript bisa cukup rumit |
| Menghilangkan debat kusir gaya penulisan kode saat code review PR | Linter yang terlalu agresif dapat memperlambat proses commit developer |
| Menjamin riwayat commit git diff yang bersih dan fokus pada logika bisnis | Perlu disiplin tim agar tidak menggunakan komentar `eslint-disable` sembarangan |

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Wajib Digunakan
- **Seluruh proyek perangkat lunak**: Baik proyek solo, startup kecil, maupun repositori enterprise berskala besar.
- Wajib dipasang pada hari pertama inisialisasi repositori agar basis kode tidak terlanjur kotor.

### Kapan Menggunakan Biome alih-alih ESLint + Prettier
- Jika tim menginginkan tool all-in-one yang jauh lebih kencang (10-20x lebih cepat karena ditulis dalam Rust) dan tidak memerlukan plugin ekosistem ESLint yang rumit.

## 8. Metrik Kunci & Observabilitas

1. **Lint Warnings Count**: Targetkan 0 warnings di lingkungan production (`--max-warnings=0`). Warning yang dibiarkan menumpuk lama kelamaan akan menjadi kebiasaan buruk yang diabaikan tim.
2. **Pre-commit Execution Time**: Pastikan proses `lint-staged` pada git commit berjalan di bawah 2 detik. Jika lambat, pastikan linter hanya memeriksa staged files dan tidak menjalankan type-check seluruh project pada git commit hook.
3. **Number of `eslint-disable` Directives**: Pantau jumlah komentar penonaktifan linter di basis kode. Jika melonjak, berarti ada aturan yang terlalu mengekang atau developer sengaja mengabaikan praktik baik.

## 9. Studi Kasus Industri Nyata

Sebuah platform SaaS perpesanan instan mengalami lonjakan komplain baterai boros dan aplikasi sering hang di browser pengguna.

Setelah ditelusuri lewat profiling CPU, ditemukan sebuah komponen percakapan yang memicu infinite loop render:
```typescript
useEffect(() => {
  fetchMessages();
  setLastPolled(new Date());
}, [lastPolled]); // Memodifikasi state yang menjadi dependensinya sendiri!
```

Developer menonaktifkan peringatan editor secara manual karena terburu-buru. Tim engineering kemudian mengambil langkah tegas:
1. Memperbarui aturan `react-hooks/exhaustive-deps` dari level `warn` menjadi `error`.
2. Memasang hook git pre-commit menggunakan Husky yang memblokir commit jika terdeteksi error hooks.
3. Menambahkan pengecekan `npm run lint` pada GitHub Actions yang memblokir tombol merge PR.
4. Sejak aturan ditegakkan, insiden infinite loop render di production turun menjadi 0 kasus.

## 10. Fakta Menarik & Gotchas Production

- **ESLint vs Prettier Conflict**: Aturan nomor satu saat memasang ESLint dan Prettier: matikan seluruh aturan pemformatan di ESLint (seperti indentasi, quotes, spasi) menggunakan `eslint-config-prettier`. Biarkan Prettier yang menangani urusan pemformatan, dan biarkan ESLint fokus menangani kualitas logika kode.
- **`tsc` Tidak Dijalankan oleh ESLint**: ESLint memeriksa sintaks dan gaya, namun secara default tidak memvalidasi tipe data global TypeScript secara menyeluruh. Selalu jalankan `tsc --noEmit` di pipeline CI bersamaan dengan `npm run lint`.
- **Evolusi Biome**: Tool baru seperti `Biome` (fork dari Rome) kini mulai diadopsi luas sebagai pengganti ESLint dan Prettier sekaligus, menyatukan linting dan formatting dalam 1 binary Rust super cepat tanpa overhead node_modules.

## 11. Rangkuman Singkat

Linting (ESLint) berfokus mendeteksi bug logika dan penegakan best practices framework, sedangkan Formatting (Prettier) berfokus pada standardisasi estetika tata letak visual kode. Dipadukan dengan Git Pre-commit Hooks (Husky dan lint-staged), tim developer dijamin menghasilkan basis kode yang bersih, konsisten, dan bebas dari bug keteledoran.
