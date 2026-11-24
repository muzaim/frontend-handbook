# Component-Based Architecture

## 1. Skenario Sistem Nyata

Pada sebuah portal web layanan kesehatan, terdapat elemen tombol konfirmasi (*Call to Action Button*) yang tersebar di 40 halaman berbeda (halaman janji temu dokter, pendaftaran pasien, pembayaran obat, hasil lab, dll).

Developer generasi lama membangun tombol-tombol tersebut dengan menyalin dan menempelkan (*copy-paste*) kode HTML dan CSS mentah di setiap halaman:

```html
<!-- Ditempel berulang kali di 40 file berbeda -->
<button class="btn-blue-primary text-bold" style="padding: 10px 20px; border-radius: 4px;">
  <span>Konfirmasi Janji Temu</span>
</button>
```

Dampak pada sistem:
- Ketika tim desain grafis memutuskan untuk memperbarui identitas visual perusahaan (mengubah warna biru ke hijau tosca dan membuat sudut tombol lebih melengkung), developer terpaksa mencari dan mengubah kode di **40 file yang berbeda secara manual**.
- Tiga file terlewatkan, menghasilkan tampilan tombol yang belang-belang di berbagai menu.
- Tidak ada konsistensi dalam menangani status pemuatan (*loading state*) atau disabilitas tombol saat transaksi sedang diproses, memicu bug pembayaran ganda karena tombol bisa diklik berulang kali.

Solusi: Mengadopsi **Component-Based Architecture (Arsitektur Berbasis Komponen)**. Membungkus elemen tombol menjadi satu komponen independen yang dapat digunakan kembali (*reusable component*) bernama `<Button />`. Komponen ini merangkum struktur HTML, logika status pemuatan, aksesibilitas, dan tampilan CSS di satu tempat terpusat.

## 2. Apa Itu dan Buat Apa?

Component-Based Architecture adalah paradigma desain perangkat lunak di mana antarmuka pengguna (UI) dibangun dari balok-balok penyusun kecil yang mandiri, terisolasi, dan dapat digunakan kembali (*reusable*) yang disebut **Komponen**.

Setiap komponen bertanggung jawab atas tampilan visualnya sendiri (*view*), perilakunya (*logic*), dan status datanya (*state*). Komponen-komponen kecil (seperti `Avatar`, `Badge`, `Button`) dapat disusun (*composed*) menjadi komponen yang lebih besar (seperti `UserProfileCard`), yang kemudian disusun kembali menjadi satu halaman penuh (`DashboardPage`).

Paradigma ini merupakan fondasi utama seluruh framework dan library frontend modern: React, Vue, Svelte, Angular, dan Web Components standar.

Fungsi utama Arsitektur Berbasis Komponen:
- **Prinsip DRY (Don't Repeat Yourself)**: Meniadakan duplikasi kode UI di seluruh aplikasi.
- **Enkapsulasi & Isolasi**: Perubahan logika di dalam satu komponen tidak akan merusak komponen lain di sekitarnya.
- **Konsistensi Antarmuka (*Design System*)**: Menjamin seluruh elemen tombol, form, dan kartu memiliki perilaku dan estetika seragam.

## 3. Kenapa Harus Pakai Ini?

Membangun aplikasi web berskala besar dengan ratusan halaman menggunakan file HTML statis monolitik adalah mimpi buruk pemeliharaan (*maintenance nightmare*).

Arsitektur Berbasis Komponen memperlakukan UI seperti mainan balok Lego:
1. **Dapat Diuji Secara Terisolasi (*Unit Testing*)**: Anda dapat menguji apakah tombol menampilkan spinner saat `isLoading={true}` tanpa perlu menjalankan seluruh halaman dashboard.
2. **Pengembangan Paralel Efisien**: Developer A fokus membuat komponen `DatePicker`, Developer B membuat komponen `DataTable`, dan keduanya dapat digabungkan secara instan.

## 4. Cara Kerjanya (Diagram Alur Komposisi Komponen)

```
+---------------------------------------------------------------+
|                      HALAMAN PROFIL PASIEN                    |
|                        <PatientProfile />                     |
+---------------------------------------------------------------+
|                                                               |
|   +-------------------------------------------------------+   |
|   |                 KARTU INFORMASI PASIEN                |   |
|   |                   <PatientCard />                     |   |
|   +-------------------------------------------------------+   |
|   |  [ <Avatar /> ]                                       |   |
|   |  Nama: Syafri Surya                                   |   |
|   |  Status: [ <Badge variant="success">Aktif</Badge> ]   |   |
|   |                                                       |   |
|   |  +-------------------------------------------------+  |   |
|   |  |            KOMPONEN TOMBOL REUSABLE             |  |   |
|   |  |    <Button variant="primary" isLoading={false}> |  |   |
|   |  |      Jadwalkan Konsultasi                       |  |   |
|   |  |    </Button>                                    |  |   |
|   |  +-------------------------------------------------+  |   |
|   +-------------------------------------------------------+   |
|                                                               |
+---------------------------------------------------------------+
```

## 5. Cuplikan Logika Inti

### Membangun Komponen Reusable yang Tangguh (TypeScript / React)

Prinsip enkapsulasi logika, varian visual, dan penanganan aksesibilitas dalam satu berkas:

```typescript
// components/ui/Button.tsx
import React from "react";

interface ButtonProps extends React.ButtonHTMLAttributes<HTMLButtonElement> {
  variant?: "primary" | "secondary" | "danger";
  size?: "sm" | "md" | "lg";
  isLoading?: boolean;
  children: React.ReactNode;
}

export const Button: React.FC<ButtonProps> = ({
  variant = "primary",
  size = "md",
  isLoading = false,
  disabled,
  children,
  className = "",
  ...restProps
}) => {
  // 1. Tentukan gaya visual berdasarkan props varian
  const baseStyles = "font-medium rounded-lg transition-all flex items-center justify-center";
  const variantStyles = {
    primary: "bg-blue-600 text-white hover:bg-blue-700 disabled:bg-blue-300",
    secondary: "bg-gray-200 text-gray-800 hover:bg-gray-300 disabled:bg-gray-100",
    danger: "bg-red-600 text-white hover:bg-red-700 disabled:bg-red-300",
  }[variant];

  const sizeStyles = {
    sm: "px-3 py-1.5 text-xs",
    md: "px-4 py-2 text-sm",
    lg: "px-6 py-3 text-base",
  }[size];

  return (
    <button
      className={`${baseStyles} ${variantStyles} ${sizeStyles} ${className}`}
      disabled={disabled || isLoading}
      aria-busy={isLoading}
      {...restProps}
    >
      {isLoading ? (
        <span className="flex items-center gap-2">
          <svg className="animate-spin h-4 w-4" viewBox="0 0 24 24">...</svg>
          Memproses...
        </span>
      ) : (
        children
      )}
    </button>
  );
};
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan
- **Maintainability Superior**: Perubahan gaya visual atau perbaikan bug di satu berkas komponen otomatis langsung diterapkan di seluruh aplikasi.
- **Keterbacaan Kode Deklaratif**: Struktur tampilan halaman menjadi sangat mudah dibaca: `<Header /><ProductGrid /><Footer />`.
- **Kolaborasi Tim Mulus Melalui Storybook**: Komponen UI dapat didokumentasikan dan dipratinjau dalam katalog interaktif independen.

### Kekurangan & Risiko
- **Prop Drilling Problem**: Pada pohon komponen yang bersarang dalam, mengirimkan data dari kakek ke cicit (*grandparent to grandchild*) melalui props menjadi melelahkan jika tidak menggunakan state management yang tepat.
- **Bahaya Over-Abstraction**: Terlalu banyak memecah komponen menjadi bagian mikro yang tidak perlu (misal membuat komponen terpisah hanya untuk satu tag `<p>`) dapat membuat arsitektur direktori berantakan dan sulit ditelusuri oleh developer baru.

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Memecah Menjadi Komponen Baru
- Kode UI yang berulang lebih dari dua kali di tempat berbeda (*Rule of Three*).
- Komponen yang memiliki kompleksitas logika internal tersendiri (misal form input dengan validasi regex otomatis dan autocomplete).
- Bagian UI yang sering di-render ulang secara independen untuk mencegah re-render pada komponen induknya.

### Kapan Sebaiknya Tetap Menjadi Inline Elements
- Elemen tata letak statis yang hanya muncul sekali seumur hidup aplikasi dan tidak memiliki logika interaktif (misal halaman 404 sederhana).

## 8. Metrik Kunci & Observabilitas

- **Component Re-render Count**: Gunakan ekstensi *React Developer Tools -> Profiler* dan aktifkan opsi *"Highlight updates when components render"*. Jika komponen tombol berkedip hijau saat pengguna mengetik di input teks lain, artinya ada masalah re-render yang bocor dari induk ke anak.
- **Storybook Coverage**: Pantau persentase komponen UI yang terdokumentasi dan memiliki tes visual regresi di Storybook / Chromatic.

## 9. Studi Kasus Industri Nyata

### Evolusi Design System Uber Web (Base Web)

- **Tantangan Rekayasa Riil**:
  Uber memiliki puluhan aplikasi web: Uber Driver portal, Uber Eats merchant dashboard, Uber Freight, dan platform internal perusahaan. Ratusan developer membuat formulir input dan tabel data dengan cara mereka sendiri, menghasilkan pengalaman visual yang berantakan dan rawan bug aksesibilitas (*a11y*).
- **Implementasi Solusi Industri**:
  Uber merancang dan merilis **Base Web Design System**, arsitektur berbasis komponen terpusat yang bersifat open-source.
  1. Seluruh komponen fondasi (Button, Modal, DatePicker, Select) dibuat dengan standar aksesibilitas keyboard dan screen-reader WCAG AAA tertinggi.
  2. Komponen mendukung sistem tema (*theming tokens*) yang fleksibel sehingga Uber Eats (warna hijau) dan Uber Driver (warna hitam) dapat menggunakan basis komponen yang sama persis.
- **Dampak & Hasil Sistem**:
  - Kecepatan rilis fitur antarmuka baru oleh tim produk meningkat sebesar 35%.
  - Kasus bug aksesibilitas hukum turun hingga nol di lingkungan produksi.

## 10. Fakta Menarik & Gotchas Production

1. **Pemisahan Smart (Container) vs Dumb (Presentational) Components**:
   Pola klasik industri yang tetap relevan:
   - *Presentational Component*: Komponen murni yang hanya menerima `props` dan merender tampilan visual (tidak melakukan fetch data API atau manipulasi global store). Sangat mudah diuji dan digunakan kembali.
   - *Container Component*: Komponen yang bertugas mengambil data dari server atau state store, lalu menyalurkan data tersebut ke komponen presentational.
2. **Bahaya Mendefinisikan Komponen di Dalam Komponen**:
   ```javascript
   // JEBAKAN FATAL:
   function ParentComponent() {
     // BENCANA: Komponen anak dibuat ulang setiap kali Parent re-render!
     // State internal Child akan selalu ter-reset dan input kehilangan fokus!
     function ChildComponent() {
       return <input />;
     }
     return <div><ChildComponent /></div>;
   }
   ```
   *Aturan*: Selalu definisikan fungsi komponen di luar komponen lain!

## 11. Rangkuman Singkat

1. **Balok Lego UI**: Arsitektur berbasis komponen memecah tampilan antarmuka menjadi unit-unit kecil yang mandiri dan terenkapsulasi.
2. **Pola DRY**: Memusatkan logika dan tampilan visual di satu tempat untuk pemeliharaan yang mudah.
3. **Komposisi Alami**: Komponen kecil disusun menjadi komponen yang lebih besar secara deklaratif.
4. **Hindari Abstraksi Berlebih**: Pecah komponen saat ada pengulangan atau kompleksitas nyata, jangan membuat abstraksi prematur untuk elemen yang hanya dipakai sekali.
