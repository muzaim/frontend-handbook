# CI/CD for Frontend

## 1. Skenario Sistem Nyata

Sebuah tim produk memiliki kebiasaan manual deployment:
Setiap hari Jumat sore sebelum rilis, lead developer menjalankan `npm run build` di laptop pribadinya. Lalu, folder hasil build `dist/` diunggah ke server hosting AWS S3 / Nginx menggunakan aplikasi FTP FileZilla.

Suatu hari, laptop sang lead developer memiliki berkas `.env.local` sisa pengujian lokal yang belum dibersihkan. Akibatnya, bundle yang terunggah ke production mengarahkan ribuan transaksi pengguna ke database lokal `http://localhost:8080` milik sang developer. Aplikasi production mati total selama akhir pekan karena sang developer sedang dalam perjalanan dan laptopnya mati.

Selain itu, tidak ada jaminan bahwa kode yang diunggah telah lulus uji linter dan unit test.

Bencana human-error dan dependensi laptop individu ini diselesaikan secara tuntas oleh pipeline otomatisasi **CI/CD (Continuous Integration / Continuous Deployment)**.

## 2. Apa Itu dan Buat Apa?

CI/CD adalah metodologi otomasi alur kerja pengembangan perangkat lunak modern yang membagi siklus hidup rilis menjadi dua pilar:

1. **Continuous Integration (CI)**:
   - Praktik di mana setiap developer secara berkala menggabungkan (*merge*) perubahan kode mereka ke repositori pusat (Git).
   - Setiap kali ada *Pull Request (PR)* atau commit baru, server CI otomatis menjalankan serangkaian tahapan validasi: menginstal dependensi, memeriksa format kode (Lint), memverifikasi tipe data (Type-check), dan mengeksekusi seluruh suite pengujian otomatis (Unit, Integration, Component Tests).
2. **Continuous Deployment / Delivery (CD)**:
   - Otomatisasi proses perilisan kode yang telah terbukti lolos validasi CI ke lingkungan staging atau production secara otomatis tanpa intervensi manual.
   - Menyediakan fitur *Preview Deployments* (lingkungan demo sementara untuk setiap PR) dan *Zero-Downtime Deployment* di edge network (Vercel, Cloudflare Pages, Netlify, atau AWS CloudFront).

## 3. Kenapa Harus Pakai Ini?

- **Eliminasi Human Error**: Proses build dan rilis dijalankan oleh server terisolasi yang bersih (*clean runner environment*), bukan dari laptop developer yang sarat dependensi lokal tersembunyi.
- **Umpan Balik Cepat pada Pull Request**: Reviewer kode tidak perlu mengunduh branch ke laptop lokal untuk mengecek apakah kodenya bisa di-build. Server CI langsung memberikan tanda centang hijau atau silang merah di halaman GitHub PR dalam 2-3 menit.
- **Preview Deployments untuk Kolaborasi Tim**: Desainer UI/UX, Product Manager, dan QA bisa langsung mencoba fitur baru melalui link URL demo sementara (`https://pr-142.app-preview.com`) sebelum kode di-merge ke branch utama.
- **Rollback Seketika (Instant Rollback)**: Jika rilis baru memiliki masalah performa mendadak di production, tim dapat mengembalikan (*rollback*) versi rilis ke commit sebelumnya hanya dengan 1 klik tombol dalam hitungan detik.

## 4. Cara Kerjanya

Berikut adalah diagram tahapan pipeline CI/CD lengkap untuk aplikasi frontend menggunakan GitHub Actions:

```
[ Developer Melakukan Push / Buka Pull Request ]
                       |
                       v
====================== TAHAP 1: CONTINUOUS INTEGRATION (CI) ======================
[ Runner GitHub Actions (Ubuntu VM) ]
   |
   |-- 1. Checkout Repository Kode
   |-- 2. Setup Node.js + Restore Pnpm/Npm Cache (Hemat waktu install)
   |-- 3. Parallel Jobs:
   |      +---------------------+---------------------+---------------------+
   |      |                     |                     |                     |
   |      v                     v                     v                     v
   |   JOB LINT              JOB TYPES             JOB TESTS            JOB AUDIT
   |   npm run lint          tsc --noEmit          npm test             audit deps
   |      |                     |                     |                     |
   |      +---------------------+---------------------+---------------------+
   |                                 |
   |                        (Semua Job Lolos)
   |                                 v
   |-- 4. JOB BUILD: npm run build (Menghasilkan folder dist/)
   |-- 5. Laporan Lighthouse CI: Skor Web Vitals & Aksesibilitas
                       |
====================== TAHAP 2: CONTINUOUS DEPLOYMENT (CD) ======================
                       |
         +-------------+-------------+
         |                           |
  (Kondisi: Pull Request)     (Kondisi: Merge ke 'main')
         |                           |
         v                           v
  [ PREVIEW DEPLOYMENT ]      [ PRODUCTION RELEASE ]
  Deploy ke subdomain unik:   Deploy ke CDN Global (Edge)
  pr-42.preview.app.com       app.perusahaan.com
  (Kirim link bot ke PR)      (Zero Downtime + Invalidation)
```

## 5. Cuplikan Logika Inti

### Workflow GitHub Actions Lengkap (`.github/workflows/frontend-ci-cd.yml`)

```yaml
name: Frontend CI/CD Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  validate-and-test:
    runs-on: ubuntu-latest
    steps:
      - name: 1. Checkout Kode
        uses: actions/checkout@v4

      - name: 2. Setup Node.js Runtime
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm' # Otomatis caching dependensi node_modules

      - name: 3. Install Dependensi Bersih
        run: npm ci

      - name: 4. Pemeriksaan Linter & Format
        run: npm run lint

      - name: 5. Pemeriksaan Type-Safety TypeScript
        run: npx tsc --noEmit

      - name: 6. Eksekusi Unit & Integration Tests dengan Coverage
        run: npm test -- --coverage

      - name: 7. Verifikasi Produksi Build
        run: npm run build
        env:
          VITE_API_BASE_URL: ${{ secrets.VITE_API_BASE_URL_STAGING }}

      - name: 8. Upload Build Artifacts
        uses: actions/upload-artifact@v4
        with:
          name: production-build
          path: dist/
          retention-days: 7

  deploy-production:
    needs: validate-and-test
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    runs-on: ubuntu-latest
    steps:
      - name: 1. Download Build Artifacts
        uses: actions/download-artifact@v4
        with:
          name: production-build
          path: dist/

      - name: 2. Deploy ke Cloudflare Pages / AWS S3
        run: |
          echo "Mengunggah aset statis ke CDN Edge Network..."
          # Contoh: npx wrangler pages deploy dist/ --project-name=my-app
        env:
          CLOUDFLARE_API_TOKEN: ${{ secrets.CF_API_TOKEN }}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

| Kelebihan | Kekurangan / Risiko |
| :--- | :--- |
| Standarisasi pengujian dan perilisan tanpa ada ruang untuk human-error | Memerlukan waktu pemeliharaan konfigurasi pipeline YAML dan secret management |
| Umpan balik kualitas kode tersedia dalam hitungan menit untuk setiap PR | Pipeline yang terlalu lambat (>15 menit) dapat menghambat produktivitas merger |
| Menjamin rilis zero-downtime dengan kemampuan rollback seketika | Penggunaan resource runner komputasi CI berbayar jika batas kuota gratis terlampaui |

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Selalu Pakai
- Seluruh aplikasi web yang dikembangkan oleh tim lebih dari 1 orang.
- Aplikasi produksi yang melayani pengguna nyata dan membutuhkan kepastian stabilitas rilis.
- Proyek open-source untuk memverifikasi kontribusi PR dari komunitas luar.

### Optimasi Menghindari Pipeline Lambat
- Gunakan fitur **Caching** pada paket package manager (`actions/setup-node` cache `npm` atau `pnpm`).
- Jalankan job `lint`, `typecheck`, dan `test` secara paralel di GitHub Actions, bukan secara berurutan (*sequential*), untuk memangkas total durasi build hingga 50%.

## 8. Metrik Kunci & Observabilitas

1. **Mean Time to CI Feedback**: Waktu dari saat developer menjalankan `git push` hingga notifikasi sukses/gagal muncul di PR. Targetkan di bawah 4 menit.
2. **Deployment Frequency**: Frekuensi rilis ke production per hari/minggu. Tim elit dengan CI/CD yang matang mampu melakukan deploy beberapa kali dalam sehari.
3. **Change Failure Rate (DORA Metric)**: Persentase deployment yang menyebabkan insiden di production dan membutuhkan rollback/hotfix. Targetkan di bawah 5%.

## 9. Studi Kasus Industri Nyata

Sebuah platform edukasi daring berskala nasional memiliki tim frontend yang terdiri dari 20 developer. Sebelumnya, proses deploy dilakukan manual setiap hari Selasa malam. Rilis sering tertunda berjam-jam karena ditemukan konflik kode dan kegagalan tipe data saat digabungkan di server staging.

Transformasi arsitektur CI/CD:
1. Membangun pipeline GitHub Actions otomatis: Linting + TypeScript type-check + Vitest + Preview Deploy di Cloudflare Pages untuk setiap branch PR.
2. Setiap fitur yang dibuat developer langsung memiliki link preview aktif. Tim QA dan Product Manager melakukan verifikasi acceptance test langsung di link preview tersebut.
3. Tombol merge ke `main` otomatis memicu deploy ke production via global CDN dalam waktu 90 detik.
4. Frekuensi rilis meningkat dari sekali seminggu menjadi rata-rata 8 kali sehari tanpa ada insiden server down.

## 10. Fakta Menarik & Gotchas Production

- **`npm ci` vs `npm install`**: Di server CI, **selalu gunakan `npm ci`**, bukan `npm install`. `npm ci` menghapus folder `node_modules` lama dan menginstal dependensi persis sesuai dengan yang terkunci di `package-lock.json`. Ini mencegah inkonsistensi versi dependensi minor yang bisa merusak build.
- **Cache Invalidation di CDN**: Setelah mengunggah file HTML baru ke CDN/S3 di tahap CD, pastikan pipeline memicu *Purge Cache* atau *Invalidation* khusus untuk berkas `index.html`. Berkas JavaScript dan CSS berekstensi hash (`app-a1b2c3.js`) tidak perlu di-purge karena sudah memiliki nama unik (*immutable cache*).
- **Branch Protection Rules**: Lindungi branch `main` di GitHub dengan aturan *Require status checks to pass before merging*. Ini secara mutlak melarang siapa pun menggabungkan kode yang gagal di pipeline CI.

## 11. Rangkuman Singkat

CI/CD untuk frontend mengotomasi validasi kualitas kode (lint, type-check, test, build) pada setiap commit dan mempublikasikan aset statis ke CDN global secara otomatis. Alur kerja ini melenyapkan kesalahan manual, mempercepat integrasi fitur via preview deployments, dan menjamin rilis production yang stabil dan dapat di-rollback kapan saja.
