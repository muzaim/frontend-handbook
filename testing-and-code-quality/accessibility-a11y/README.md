# Accessibility (a11y)

## 1. Skenario Sistem Nyata

Sebuah instansi perbankan nasional merilis aplikasi pembukaan rekening digital baru. Seorang tunanetra bernama Budi mencoba mendaftar menggunakan teknologi pembaca layar (*screen reader* NVDA di laptop dan VoiceOver di iPhone).

Ketika membuka halaman pendaftaran, pembaca layar hanya membacakan:
"Tombol tidak berlabel... Gambar... Tautan tanpa teks..."

Budi mencoba menekan tombol `Tab` pada keyboard untuk berpindah ke kolom isian nama, namun fokus kursor keyboard terjebak di dalam banner promosi dan tidak bisa keluar (*keyboard focus trap*). Selain itu, tombol "Lanjut Pendaftaran" dibuat menggunakan elemen `<div onClick={...}>` tanpa atribut ARIA dan tanpa dukungan tombol `Enter`, sehingga sama sekali tidak dapat diaktifkan oleh pengguna keyboard.

Budi frustrasi dan membatalkan niatnya. Instansi tersebut tidak hanya kehilangan nasabah potensial, tetapi juga melanggar regulasi inklusivitas hukum disabilitas digital internasional (seperti ADA Title III dan European Accessibility Act) yang mewajibkan kepatuhan aksesibilitas web.

## 2. Apa Itu dan Buat Apa?

Accessibility (sering disingkat sebagai **a11y**, di mana angka 11 melambangkan jumlah huruf antara 'a' dan 'y') adalah disiplin perancangan dan pengembangan aplikasi web agar dapat diakses, dipahami, dinavigasi, dan digunakan secara mandiri oleh semua orang, termasuk mereka yang memiliki disabilitas visual, motorik, pendengaran, maupun kognitif.

Standar internasional aksesibilitas diatur oleh konsorsium W3C melalui **WCAG (Web Content Accessibility Guidelines)** yang berlandaskan 4 prinsip utama (**POUR**):
1. **Perceivable (Dapat Dirasakan)**: Informasi dan elemen UI harus disajikan sehingga dapat ditangkap oleh indera pengguna (teks alternatif untuk gambar, rasio kontras warna memadai).
2. **Operable (Dapat Dioperasikan)**: Antarmuka tidak boleh memerlukan interaksi yang mustahil dilakukan oleh pengguna (seluruh fitur harus bisa dioperasikan via keyboard tanpa mouse).
3. **Understandable (Dapat Dipahami)**: Informasi dan cara kerja antarmuka harus jelas dan konsisten (pesan error formulir deskriptif dan mudah dipahami).
4. **Robust (Kuat / Andal)**: Konten harus dapat diinterpretasikan secara andal oleh berbagai teknologi asistif (screen reader, braille display, software navigasi suara).

## 3. Kenapa Harus Pakai Ini?

- **Inklusivitas Sosial**: Jutaan orang di seluruh dunia mengandalkan teknologi asistif untuk beraktivitas digital sehari-hari. Desain inklusif memastikan tidak ada kelompok masyarakat yang terisolasi.
- **Meningkatkan Pengalaman Seluruh Pengguna**: Praktik aksesibilitas menguntungkan semua orang dalam situasi temporal (misal: pengguna yang tangannya cedera gips, pengguna di bawah terik sinar matahari yang butuh kontras tinggi, atau developer yang gemar navigasi keyboard cepat).
- **SEO yang Lebih Unggul**: Mesin perayap Googlebot membaca halaman web persis seperti cara screen reader membaca: mengutamakan struktur HTML semantik, hierarki judul (`h1`-`h6`), dan atribut `alt` gambar.
- **Kepatuhan Hukum & Standar Industri**: Menghindari sanksi denda hukum regulasi aksesibilitas pada sektor publik, finansial, dan e-commerce global.

## 4. Cara Kerjanya

Berikut adalah diagram perbandingan bagaimana browser menyajikan antarmuka visual ke pengguna biasa vs bagaimana browser membangun **Accessibility Tree (AOM)** untuk screen reader:

```
[ KODE HTML HALAMAN WEB ]
<button aria-label="Tutup Dialog Pembayaran" class="icon-x"></button>
                           |
                           v
              [ MESIN BROWSER: PARSING DOM ]
                           |
            +--------------+--------------+
            |                             |
            v                             v
   [ RENDER TREE (CSSOM) ]     [ ACCESSIBILITY TREE (AOM) ]
            |                             |
      (Visual Piksel)            (Struktur Semantik untuk Asistif)
            |                             |
   Menampilkan ikon silang        Role: button
   berwarna merah di layar        Name: "Tutup Dialog Pembayaran"
            |                     State: focusable, clickable
            v                             |
    [ Pengguna Layar ]                    v
                              [ Screen Reader: NVDA / VoiceOver ]
                              Membacakan lewat suara / braille:
                              "Tombol Tutup Dialog Pembayaran, tekan Enter untuk mengaktifkan"
```

## 5. Cuplikan Logika Inti

### HTML Semantik vs "Div Soup" Non-Semantik

```tsx
// 1. CONTOH KODE BURUK (Div Soup: Tidak Aksesibel)
// JANGAN LAKUKAN INI! Screen reader menganggap ini teks biasa dan tidak bisa difokuskan keyboard
function BadCustomButton({ onClick, label }: { onClick: () => void; label: string }) {
  return (
    <div className="btn-styling" onClick={onClick}>
      {label}
    </div>
  );
}

// 2. CONTOH KODE BAIK (HTML Semantik Native)
// AMAN: Browser otomatis memberikan role="button", keyboard focus (Tab), dan event trigger (Enter/Space)
function GoodButton({ onClick, label }: { onClick: () => void; label: string }) {
  return (
    <button type="button" className="btn-styling" onClick={onClick}>
      {label}
    </button>
  );
}
```

### Aksesibilitas Formulir dan Error Handling (ARIA)

```tsx
// src/components/AccessibleInput.tsx
import React from 'react';

interface AccessibleInputProps {
  id: string;
  label: string;
  value: string;
  error?: string;
  onChange: (val: string) => void;
}

export function AccessibleInput({ id, label, value, error, onChange }: AccessibleInputProps) {
  const errorId = `${id}-error`;
  const hasError = Boolean(error);

  return (
    <div className="form-group mb-4">
      {/* Label wajib dihubungkan ke input via htmlFor */}
      <label htmlFor={id} className="block font-medium mb-1">
        {label}
      </label>

      <input
        id={id}
        type="text"
        value={value}
        onChange={(e) => onChange(e.target.value)}
        // Hubungkan pesan error ke input secara semantik
        aria-describedby={hasError ? errorId : undefined}
        // Beri tahu screen reader jika data tidak valid
        aria-invalid={hasError}
        className={`w-full border p-2 rounded ${hasError ? 'border-red-500' : 'border-gray-300'}`}
      />

      {/* role="alert" membuat screen reader langsung membacakan error saat muncul */}
      {hasError && (
        <p id={errorId} role="alert" className="text-red-600 text-sm mt-1">
          {error}
        </p>
      )}
    </div>
  );
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

| Kelebihan | Kekurangan / Risiko |
| :--- | :--- |
| Membuka pasar ke jutaan pengguna disabilitas dan meningkatkan metrik SEO Google | Membutuhkan pemahaman mendalam tentang standar W3C ARIA dan pengujian manual keyboard |
| Mendorong kode HTML yang jauh lebih terstruktur, bersih, dan mudah dirawat | Menambahkan waktu pengembangan awal untuk merancang navigasi modal trap dan state focus |
| Melindungi perusahaan dari risiko hukum dan tuntutan regulasi inklusivitas | Penggunaan atribut ARIA yang salah justru dapat merusak pengalaman screen reader (*"No ARIA is better than Bad ARIA"*) |

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Selalu Menerapkan a11y
- **Selalu dan di semua halaman aplikasi web publik maupun internal**: Form pendaftaran, dashboard, checkout pembayaran, navigasi menu, dan artikel informasi.
- Komponen interaktif kompleks: Modal Dialog (wajib membatasi fokus keyboard di dalam dialog), Dropdown Menu, Tab panels, dan Accordion.

### Prinsip Utama Penggunaan ARIA
- **Aturan Pertama ARIA**: Gunakan elemen HTML native terlebih dahulu (`<button>`, `<input>`, `<nav>`, `<main>`, `<dialog>`). Hanya gunakan atribut ARIA kustom jika elemen HTML native tidak menyediakan fitur yang dibutuhkan.

## 8. Metrik Kunci & Observabilitas

1. **Lighthouse Accessibility Score**: Pantau audit aksesibilitas di Google Lighthouse DevTools. Targetkan skor minimal 95-100 di lingkungan production.
2. **Automated a11y Linting Violations**: Pasang plugin `eslint-plugin-jsx-a11y` di editor dan pipeline CI untuk mencegah elemen `<img>` tanpa atribut `alt` atau `<button>` kosong.
3. **Color Contrast Ratio**: Pastikan rasio kontras teks terhadap latar belakang memenuhi standar WCAG AA:
   - Minimal **4.5:1** untuk teks ukuran normal.
   - Minimal **3.0:1** untuk teks ukuran besar (>18pt atau >14pt bold).

## 9. Studi Kasus Industri Nyata

Sebuah platform pemesanan hotel internasional mengalami penurunan rasio konversi pada pengguna berusia lanjut dan pengguna keyboard.

Audit pentest aksesibilitas menemukan beberapa kegagalan kritis:
1. Modal pop-up "Pilih Kamar" tidak mengunci fokus keyboard (*focus trap missing*), sehingga saat pengguna menekan tombol Tab, kursor berpindah ke elemen di belakang modal yang tidak terlihat.
2. Kalender pemilih tanggal tidak dapat dioperasikan menggunakan tombol panah keyboard (*Arrow Keys*).
3. Kontras warna teks harga (abu-abu muda di atas latar putih: rasio 2.1:1) sangat sulit dibaca oleh penderita gangguan penglihatan warna.

Solusi engineering:
1. Mengintegrasikan pustaka komponen headless yang telah teruji aksesibilitasnya secara native (seperti Radix UI / Headless UI).
2. Memperbaiki palet warna untuk memastikan seluruh teks memiliki rasio kontras minimal 4.8:1.
3. Menambahkan pengujian otomatis menggunakan `@axe-core/playwright` pada pipeline E2E.
4. Hasilnya, Lighthouse skor naik dari 62 menjadi 98, dan jumlah pemesanan dari pengguna desktop non-mouse meningkat 18%.

## 10. Fakta Menarik & Gotchas Production

- **Bahaya `outline: none`**: Menghilangkan garis outline fokus menggunakan CSS `outline: none;` tanpa memberikan gaya fokus visual pengganti (`:focus-visible`) adalah salah satu dosa terbesar aksesibilitas web. Ini membuat pengguna keyboard menjadi "buta" di mana letak posisi kursor mereka.
- **Tautan Lewati Konten (Skip Links)**: Menyediakan tombol tersembunyi "Lewati ke Konten Utama" di bagian paling atas halaman memungkinkan pengguna keyboard melompati puluhan link menu navbar untuk langsung membaca isi artikel utama.
- **Atribut `alt=""` Kosong pada Gambar Dekoratif**: Gambar yang murni berfungsi sebagai dekorasi visual (garis pemisah, hiasan latar) sebaiknya diberi `alt=""` (string kosong). Jika atribut `alt` dihapus total, screen reader justru akan membacakan URL file gambar yang panjang dan mengganggu.

## 11. Rangkuman Singkat

Aksesibilitas (a11y) adalah fondasi web inklusif yang memastikan aplikasi dapat dinavigasi oleh semua orang, termasuk pengguna dengan teknologi asistif. Terapkan prinsip POUR, prioritaskan elemen HTML semantik native, pastikan navigasi keyboard penuh, dan validasi secara otomatis menggunakan tools linter dan engine axe-core.
