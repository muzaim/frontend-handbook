# Monorepo Architecture

## 1. Skenario Sistem Nyata

Sebuah perusahaan fintech memiliki 4 aplikasi web terpisah:
1. `customer-portal` (web nasabah).
2. `admin-dashboard` (web internal operasional).
3. `merchant-checkout` (web gateway pembayaran toko).
4. `documentation-site` (portal dokumentasi API).

Semula, keempat aplikasi ini disimpan di 4 repositori Git yang terpisah (*Polyrepo*). Setiap kali tim desain memperbarui komponen tombol perbankan atau skema validasi tipe data transfer, tim harus:
- Melakukan perubahan di repositori library komponen UI tersendiri.
- Menjalankan rilis paket npm internal, menunggu proses build dan publish ke private npm registry.
- Membuka 4 Pull Request terpisah di 4 repositori aplikasi untuk menaikkan versi dependensi `npm update @fintech/ui-kit`.
- Menguji satu per satu aplikasi untuk memastikan tidak ada breaking changes.

Siklus pembaruan komponen kecil memakan waktu 3 hari kerja penuh. Tim beralih ke **Monorepo Architecture** yang ditenagai oleh **pnpm workspaces** dan **Turborepo**. Seluruh aplikasi dan paket bersama disatukan ke dalam satu repositori Git terpadu, memungkinkan pembaruan komponen dan verifikasi langsung seketika (*instant cross-package updates*).

## 2. Apa Itu dan Buat Apa?

Monorepo (singkatan dari *Monolithic Repository*) adalah strategi arsitektur pengelolaan kode sumber di mana beberapa proyek, aplikasi, dan paket pustaka bersama (*shared packages*) disimpan di dalam satu repositori Git yang sama.

Penting untuk dipahami bahwa **Monorepo BUKAN Monolitik Aplikasi**:
- Aplikasi di dalam monorepo tetap terisolasi dan dapat di-deploy secara independen ke server yang berbeda.
- Yang disatukan adalah repositori kodenya, perkakas tooling (linter, tsconfig, testing), dan mekanisme berbagi kode (*code sharing*).

Teknologi pendukung monorepo modern:
- **Workspaces Manager (pnpm / npm / yarn)**: Menangani instalasi dependensi secara efisien tanpa duplikasi storage melalui *symlinks / hard links*.
- **Build System Orchestrator (Turborepo / Nx)**: Memetakan grafik dependensi antar paket, menjalankan task build/test secara paralel di multi-core CPU, dan menyimpan hasil cache kompilasi secara lokal maupun remote (*Remote Caching*).

## 3. Kenapa Harus Pakai Ini?

- **Berbagi Kode Tanpa Overhead Publikasi Npm**: Aplikasi web dapat mengimpor komponen UI, utilitas kalkulasi, atau skema validasi Zod langsung dari folder lokal (`@repo/ui` atau `@repo/schemas`) tanpa perlu mempublikasikannya ke npm registry.
- **Atomic Commits & Refactoring Lintas Proyek**: Anda dapat mengubah kontrak API pada modul shared package dan langsung memperbarui seluruh 4 aplikasi yang mengonsumsinya dalam **satu commit Git tunggal**. Jika ada kode aplikasi yang rusak, compiler langsung memberitahu seketika.
- **Standarisasi Tooling Tunggal**: Satu konfigurasi linter (ESLint), formatter (Prettier), dan TypeScript base config yang diwarisi oleh seluruh aplikasi di dalam perusahaan.
- **Incremental Builds & Remote Caching**: Turborepo tidak akan pernah mengkompilasi ulang kode yang belum pernah diubah (*"Never do the same work twice"*). Jika aplikasi A tidak diubah, proses build selesai dalam 100 milidetik berkat cache.

## 4. Cara Kerjanya

Berikut adalah diagram hierarki struktur direktori monorepo dan aliran dependensi antar aplikasi dan shared packages:

```
[ AKAR MONOREPO: / ]
+-- apps/
|   +-- web-customer/        --> (Aplikasi Next.js / Vite)
|   |   +-- package.json     --> Bergantung pada: "@repo/ui", "@repo/utils"
|   +-- admin-dashboard/     --> (Aplikasi Dashboard Vite)
|   |   +-- package.json     --> Bergantung pada: "@repo/ui", "@repo/utils"
|   +-- merchant-checkout/   --> (Aplikasi Checkout)
|       +-- package.json     --> Bergantung pada: "@repo/ui"
|
+-- packages/
|   +-- ui/                  --> (Shared Component Library: Button, Modal)
|   +-- utils/               --> (Shared Pure Functions: Currency, Date)
|   +-- typescript-config/   --> (Shared tsconfig.base.json)
|   +-- eslint-config/       --> (Shared ESLint Rules)
|
+-- turbo.json               --> (Grafik Orkestrasi Pipeline Turborepo)
```

Aliran Eksekusi Turborepo:
```
apps/web-customer build
       ^
       |-- Wajib menunggu paket "@repo/ui" selesai di-build terlebih dahulu
packages/ui build
```
Turborepo secara otomatis menyusun antrean dan mengeksekusi paket-paket yang independen secara paralel.

## 5. Cuplikan Logika Inti

### 1. Konfigurasi Workspace (`pnpm-workspace.yaml`)

Mendefinisikan direktori mana saja yang menjadi bagian dari workspace monorepo:

```yaml
packages:
  - 'apps/*'
  - 'packages/*'
```

### 2. Mengonsumsi Paket Lokal di Aplikasi (`apps/web-customer/package.json`)

Menggunakan protokol `workspace:*` untuk menghubungkan dependensi lokal:

```json
{
  "name": "web-customer",
  "version": "1.0.0",
  "private": true,
  "dependencies": {
    "react": "^18.3.1",
    "react-dom": "^18.3.1",
    "@repo/ui": "workspace:*",
    "@repo/utils": "workspace:*"
  }
}
```

### 3. Konfigurasi Orkestrasi Pipeline (`turbo.json`)

Mendefinisikan hubungan dependensi antar task build dan test:

```json
{
  "$schema": "https://turbo.build/schema.json",
  "tasks": {
    "build": {
      // Task build suatu aplikasi membutuhkan seluruh paket dependensinya selesai build
      "dependsOn": ["^build"],
      "outputs": ["dist/**", ".next/**", "!.next/cache/**"]
    },
    "lint": {
      "dependsOn": ["^lint"]
    },
    "test": {
      "dependsOn": ["^build"],
      "outputs": ["coverage/**"]
    },
    "dev": {
      "cache": false,
      "persistent": true
    }
  }
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

| Kelebihan | Kekurangan / Risiko |
| :--- | :--- |
| Kemudahan berbagi kode tanpa proses rilis dan versi npm yang melelahkan | Ukuran repositori Git membesar seiring bertambahnya aplikasi dan riwayat commit |
| Refactoring skala besar lintas aplikasi dapat diselesaikan dalam 1 commit atomik | Setup awal tooling dan pipeline CI/CD lebih kompleks dibanding single-repo sederhana |
| Konsistensi styling dan design system terjamin di seluruh aplikasi perusahaan | Memerlukan disiplin dependency management agar paket tidak saling mengimpor secara siklis |

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Memilih Monorepo
- Organisasi yang memiliki lebih dari satu aplikasi web yang berbagi komponen UI, skema tipe data, atau utilitas yang sama.
- Tim yang mengembangkan Design System internal untuk dikonsumsi oleh beberapa produk digital.
- Aplikasi yang memiliki arsitektur Micro Frontends atau aplikasi multi-platform (Web + Mobile React Native).

### Kapan Cukup Menggunakan Single-Repo (Polyrepo)
- Proyek web tunggal independen yang tidak memiliki hubungan kode dengan proyek lain di perusahaan.
- Tim yang sangat kecil (1-2 developer) dengan aplikasi sederhana di mana overhead konfigurasi monorepo belum memberikan manfaat signifikan.

## 8. Metrik Kunci & Observabilitas

1. **Remote Cache Hit Rate**: Persentase task build/test di CI yang menggunakan hasil cache tersimpan di cloud (Turborepo Remote Cache / Vercel). Targetkan cache hit rate > 75% untuk menghemat biaya dan waktu komputasi CI.
2. **Monorepo Clone & Install Time**: Waktu yang dibutuhkan untuk menjalankan `git clone` dan `pnpm install` di mesin baru. Penggunaan `pnpm` sangat krusial di monorepo untuk menghemat disk space via hard linking.
3. **Circular Dependency Detection**: Jalankan alat deteksi grafik dependensi (seperti `madge` atau `dpdm`) untuk memastikan tidak ada siklus impor memutar antar paket (misal: Paket A mengimpor Paket B, dan Paket B mengimpor Paket A).

## 9. Studi Kasus Industri Nyata

Sebuah unicorn e-commerce memiliki 6 aplikasi web publik (Web Konsumen, Web Mitra Toko, Web Kurir, Web Admin, Portal Afiliasi, dan Blog Tech). Sebelumnya, mereka mengadopsi arsitektur Polyrepo (6 repositori terpisah).

Masalah muncul saat tim keamanan siber mengeluarkan instruksi wajib: memperbarui library autentikasi karena ditemukannya celah CVE tingkat tinggi.
- Butuh waktu 2 minggu bagi para pimpinan tim untuk mengoordinasikan pembaruan di 6 repositori berbeda.
- Dua aplikasi tertinggal dan lupa diperbarui, membiarkan celah keamanan terbuka selama beberapa minggu di production.

Solusi engineering:
1. Memigrasikan seluruh 6 aplikasi dan paket utilitas bersama ke dalam satu Monorepo berbasis **pnpm + Turborepo**.
2. Modul autentikasi disatukan ke dalam satu shared package: `@repo/auth`.
3. Ketika ada pembaruan keamanan, developer cukup mengubah kode di `@repo/auth` dalam satu PR. Seluruh 6 aplikasi langsung terverifikasi lulus tes secara otomatis di pipeline CI monorepo.
4. Waktu penyebaran patch keamanan terpangkas dari 2 minggu menjadi hanya 2 jam.

## 10. Fakta Menarik & Gotchas Production

- **Keunggulan Pnpm atas Npm di Monorepo**: Npm tradisional menyalin file dependensi yang sama ke setiap folder `node_modules` aplikasi, menghabiskan puluhan gigabyte disk space. `pnpm` menggunakan sistem *Content-Addressable Storage*, di mana satu versi library (misal React 18.3.1) hanya disimpan satu kali di hard disk komputer, lalu dihubungkan menggunakan hard links.
- **Konsep Remote Caching**: Turborepo memungkinkan developer A memanfaatkan hasil kompilasi yang telah dikerjakan oleh developer B atau mesin CI di belahan dunia lain. Jika developer B sudah meng-compile `@repo/ui`, developer A cukup mengunduh hasil binary kompilasinya dari cloud cache tanpa perlu menjalankan kompilasi lokal ulang.
- **Bahaya Ghost Dependencies**: Pastikan workspace manager Anda memblokir *phantom dependencies* (mengimpor library yang tidak tertulis di `package.json` lokal paket tersebut, namun ada di root). `pnpm` secara default mengisolasi dependensi dengan ketat untuk mencegah masalah ini.

## 11. Rangkuman Singkat

Monorepo Architecture menyatukan pengelolaan berbagai aplikasi dan pustaka kode bersama di dalam satu repositori Git terpadu. Dikombinasikan dengan package manager modern (pnpm workspaces) dan build orchestrator cerdas (Turborepo), monorepo menyederhanakan kolaborasi lintas tim, memangkas duplikasi kode, dan mempercepat waktu build melalui remote caching.
