# Design Systems

## 1. Skenario Sistem Nyata

Sebuah perusahaan teknologi berskala unicorn memiliki 5 produk web yang dikembangkan oleh 5 tim terpisah: Toko Online, Dompet Digital, Portal Mitra Penjual, Aplikasi Logistik, dan Pusat Bantuan Pelanggan.

Tanpa adanya sistem desain terpusat, inkonsistensi visual dan arsitektur merajalela:
- Terdapat 14 variasi warna tombol biru primer yang berbeda-beda di seluruh situs (`#0052cc`, `#1877f2`, `#0070f3`, dll).
- Setiap tim membangun komponen Modal Dialog dan Dropdown sendiri dari nol. Sebagian modal tidak bisa ditutup dengan tombol keyboard `Escape`, sebagian lagi tidak bisa dibuka di perangkat mobile.
- Setiap kali tim branding memutuskan untuk memperbarui font resmi perusahaan atau palet warna brand, butuh waktu 6 bulan bagi seluruh tim rekayasa untuk menyisir dan mengubah ribuan baris file CSS secara manual.

Untuk mengatasi inefisiensi raksasa ini, perusahaan membentuk tim khusus untuk membangun dan memelihara **Design System** terpadu.

## 2. Apa Itu dan Buat Apa?

Design System adalah kumpulan standar desain, prinsip panduan, dokumentasi terstruktur, serta pustaka komponen kode antarmuka yang dapat digunakan kembali (*reusable UI components*) untuk membangun produk digital yang konsisten, mudah diakses, dan efisien dalam skala besar.

Tiga pilar utama Design System:
1. **Design Tokens**: Nilai-nilai atomik dasar desain (seperti kode warna, ukuran tipografi, skala jarak *spacing*, radius border, dan efek bayangan) yang didefinisikan dalam format agnostik platform (biasanya JSON) dan diubah menjadi variabel CSS / objek JavaScript.
2. **Component Library**: Komponen kode antarmuka nyata (seperti `<Button>`, `<Input>`, `<Modal>`, `<Toast>`) yang telah terintegrasi dengan aksesibilitas standar (WCAG), dukungan keyboard, dan tema dinamis (Dark Mode).
3. **Dokumentasi & Playground (Storybook)**: Katalog interaktif yang menjadi satu-satunya sumber kebenaran (*Single Source of Truth*) tempat desainer, developer, dan product manager melihat, mencoba, dan menguji komponen UI secara terisolasi.

## 3. Kenapa Harus Pakai Ini?

- **Kecepatan Pengembangan Fitur (Time-to-Market)**: Tim pengembang fitur tidak perlu lagi membuang waktu menulis styling CSS tombol atau merancang logika modal dialog. Mereka cukup menyusun komponen yang sudah ada seperti menyusun balok lego.
- **Konsistensi Visual & Pengalaman Pengguna (Brand Cohesion)**: Pengguna merasakan pengalaman antarmuka yang seragam dan profesional saat berpindah-pindah antar berbagai produk di dalam ekosistem perusahaan.
- **Aksesibilitas Terpusat (Accessible by Default)**: Ketika tim Design System memperbaiki bug aksesibilitas pada komponen `<Select>` (misal menambahkan navigasi tombol panah keyboard), seluruh ratusan halaman di seluruh aplikasi perusahaan langsung otomatis menjadi aksesibel.
- **Dukungan Multi-Tema Tanpa Beban**: Memudahkan implementasi fitur Dark Mode, tema musiman, atau *White-labeling* untuk mitra korporat hanya dengan menukar nilai Design Tokens.

## 4. Cara Kerjanya

Berikut adalah diagram arsitektur aliran data Design System dari desain Figma hingga menjadi komponen yang dikonsumsi aplikasi web:

```
[ Tim Desain UI/UX di Figma ]
              |
              |-- Ekspor Desain Tokens ke JSON
              v
[ DESIGN TOKENS (tokens.json) ]
{
  "color": { "primary": { "500": "#2563eb" } },
  "spacing": { "4": "16px" },
  "radius": { "md": "8px" }
}
              |
              |-- Style Dictionary (Compiler Otomatis)
              v
[ VARIABEL CSS / TAILWIND THEME ]
:root {
  --color-primary-500: #2563eb;
  --spacing-4: 16px;
  --radius-md: 8px;
}
              |
              v
[ PUSTAKA KOMPONEN (@repo/ui) ]
Komponen Headless (Radix UI / React Aria) + Desain Tokens
- <Button variant="primary">
- <Modal isOpen={...}>
              |
              +-------------------------------+-------------------------------+
              |                               |                               |
              v                               v                               v
     [ Toko Online Web ]            [ Admin Dashboard ]             [ Storybook Catalog ]
     (Konsumsi @repo/ui)            (Konsumsi @repo/ui)             (Dokumentasi Interaktif)
```

## 5. Cuplikan Logika Inti

### 1. Definisi Design Tokens (CSS Custom Properties)

```css
/* packages/ui/src/tokens.css */
:root {
  /* Skala Warna Semantik */
  --color-brand-primary: #0284c7;
  --color-brand-primary-hover: #0369a1;
  --color-bg-surface: #ffffff;
  --color-text-main: #0f172a;
  --color-border-subtle: #e2e8f0;

  /* Skala Spacing (Kelipatan 4px) */
  --space-1: 4px;
  --space-2: 8px;
  --space-4: 16px;
  --space-6: 24px;

  /* Skala Radii */
  --radius-sm: 4px;
  --radius-md: 8px;
  --radius-full: 9999px;
}

/* Dukungan Dark Mode Otomatis via Token */
[data-theme='dark'] {
  --color-bg-surface: #0f172a;
  --color-text-main: #f8fafc;
  --color-border-subtle: #334155;
}
```

### 2. Komponen Polimorfik Varian (Button Component di TypeScript)

Menggunakan pola varian komponen yang terstruktur (misal dengan `class-variance-authority`):

```tsx
// packages/ui/src/components/Button.tsx
import React, { ButtonHTMLAttributes } from 'react';
import { cva, type VariantProps } from 'class-variance-authority';

const buttonStyles = cva(
  'inline-flex items-center justify-center font-medium transition-colors focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-offset-2 disabled:opacity-50 disabled:pointer-events-none',
  {
    variants: {
      variant: {
        primary: 'bg-[var(--color-brand-primary)] text-white hover:bg-[var(--color-brand-primary-hover)]',
        outline: 'border border-[var(--color-border-subtle)] bg-transparent hover:bg-gray-100 dark:hover:bg-gray-800',
        danger: 'bg-red-600 text-white hover:bg-red-700',
      },
      size: {
        sm: 'h-8 px-3 text-xs rounded-[var(--radius-sm)]',
        md: 'h-10 px-4 text-sm rounded-[var(--radius-md)]',
        lg: 'h-12 px-6 text-base rounded-[var(--radius-md)]',
      },
    },
    defaultVariants: {
      variant: 'primary',
      size: 'md',
    },
  }
);

export interface ButtonProps
  extends ButtonHTMLAttributes<HTMLButtonElement>,
    VariantProps<typeof buttonStyles> {
  isLoading?: boolean;
}

export function Button({ variant, size, className, isLoading, children, disabled, ...props }: ButtonProps) {
  return (
    <button
      className={buttonStyles({ variant, size, className })}
      disabled={disabled || isLoading}
      aria-busy={isLoading}
      {...props}
    >
      {isLoading ? <span>Memuat...</span> : children}
    </button>
  );
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

| Kelebihan | Kekurangan / Risiko |
| :--- | :--- |
| Konsistensi brand dan visual 100% terjamin di seluruh produk digital perusahaan | Membutuhkan investasi waktu dan alokasi tim khusus untuk pemeliharaan awal |
| Efisiensi rekayasa melonjak: penulisan fitur baru tidak perlu merancang UI dari nol | Jika komponen terlalu kaku (*inflexible*), tim fitur akan tergoda melakukan bypass styling |
| Kepatuhan aksesibilitas web terpusat dan mudah diaudit secara menyeluruh | Perubahan kecil pada komponen dasar (seperti Button) dapat memicu breaking changes di puluhan aplikasi |

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Membangun Design System
- Perusahaan yang memiliki beberapa produk digital atau basis kode web multi-aplikasi.
- Tim produk yang memiliki lebih dari 8-10 developer frontend dan desainer UI/UX berdedikasi.
- Aplikasi enterprise dengan standar branding yang ketat dan persyaratan aksesibilitas regulasi.

### Kapan Cukup Menggunakan UI Library Siap Pakai (Shadcn UI / Chakra UI / MUI)
- Tim kecil, proyek solo, atau startup tahap awal (*early stage MVP*). Membangun design system custom dari nol saat model bisnis belum terbukti adalah bentuk *premature optimization*.

## 8. Metrik Kunci & Observabilitas

1. **Design System Adoption Rate**: Persentase komponen di aplikasi web yang diimpor dari `@repo/ui` dibandingkan elemen HTML biasa atau styling kustom liar. Targetkan > 85% adopsi.
2. **Visual Regression Test Coverage**: Jumlah komponen di Storybook yang dilindungi oleh pengujian regresi visual otomatis (Chromatic / Playwright Screenshot comparison) pada setiap PR.
3. **Accessibility Audit on Component Suite**: Pastikan seluruh cerita (*stories*) di Storybook lulus audit `@storybook/addon-a11y` dengan 0 pelanggaran WCAG.

## 9. Studi Kasus Industri Nyata

Sebuah platform SaaS perbankan B2B memiliki 3 portal web. Setiap portal memiliki implementasi dropdown menu custom sendiri.

Sebuah survei kepuasan pelanggan enterprise mengungkap bahwa 35% nasabah korporat mengeluhkan pengalaman navigasi yang membingungkan karena perilaku interaksi tombol dan modal berbeda di setiap portal. Selain itu, migrasi Dark Mode yang diminta klien tertunda selama 1 tahun karena kode styling tersebar acak di ribuan file CSS.

Solusi engineering:
1. Membangun Design System terpusat bernama *"Pulse UI"* di dalam Monorepo.
2. Mengadopsi prinsip **Headless UI primitives** (menggunakan Radix UI) yang dibungkus dengan Design Tokens perusahaan.
3. Menggunakan Storybook sebagai platform komunikasi antara tim desainer produk dan pengembang.
4. Hasil implementasi:
   - Fitur Dark Mode berhasil dirilis hanya dalam waktu 3 minggu di seluruh 3 portal.
   - Waktu pembangunan halaman fitur baru (*page creation velocity*) meningkat 2,5 kali lipat.
   - Inkonsistensi bug antarmuka di tim QA turun sebesar 70%.

## 10. Fakta Menarik & Gotchas Production

- **Komponen Headless adalah Kunci**: Tren design system modern beralih ke arsitektur *Headless UI* (seperti Radix UI, Headless UI, atau React Aria). Pustaka headless menangani seluruh logika aksesibilitas, fokus keyboard, dan state ARIA yang rumit tanpa opini styling, memberikan kebebasan mutlak bagi tim internal untuk menerapkan styling visual sendiri.
- **Bahaya "Mega-Component"**: Hindari membuat satu komponen dengan terlalu banyak props konfigurasi (misal `<Button isSuperCompact isRoundedWithSpecialIcon hasDropShadowVariant3 />`). Lebih baik pecah menjadi pola komposisi (*Compound Components*): `<Dialog.Root><Dialog.Trigger /><Dialog.Content /></Dialog.Root>`.
- **Atomic Design Methodology**: Konsep pengelompokan yang dicetuskan oleh Brad Frost: *Atoms* (Label, Input, Button) -> *Molecules* (SearchBar = Input + Button) -> *Organisms* (Navbar = Logo + Navigation Links + SearchBar) -> *Templates* -> *Pages*.

## 11. Rangkuman Singkat

Design System adalah fondasi skalabilitas frontend yang menyatukan desainer dan developer melalui Design Tokens, pustaka komponen yang aksesibel, dan dokumentasi Storybook yang interaktif. Dengan memperlakukan komponen UI sebagai produk internal bersama, perusahaan menjamin konsistensi pengalaman pengguna dan mempercepat siklus rilis fitur secara signifikan.
