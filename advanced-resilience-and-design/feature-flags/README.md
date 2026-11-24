# Feature Flags (Feature Toggles)

## 1. Skenario Sistem Nyata

Tim pengembang e-commerce sedang membangun alur *One-Click Checkout* baru yang sangat kompleks. Fitur ini membutuhkan waktu pengerjaan selama 6 minggu.

Di masa lalu, tim membuat branch Git terpisah yang berumur panjang: `feat/one-click-checkout`. Selama 6 minggu, developer lain terus melakukan merge puluhan fitur lain ke branch `main`. Ketika tiba saatnya branch checkout digabungkan ke `main`, terjadi bencana *Merge Conflict Hell*:
- Ratusan baris kode bertabrakan.
- Membutuhkan waktu 4 hari hanya untuk menyelesaikan konflik git secara manual.
- Saat akhirnya di-deploy ke production pada hari Jumat malam, terjadi bug konversi mata uang tak terduga yang menyebabkan seluruh transaksi gagal.
- Tim panik melakukan rollback darurat yang memakan waktu 30 menit, merugikan penjualan miliaran rupiah.

Dengan menerapkan arsitektur **Feature Flags (Feature Toggles)**:
Fitur dikembangkan secara inkremental dan di-merge ke branch `main` setiap hari (*Trunk-Based Development*). Kode baru tetap aman berada di production namun disembunyikan di balik toggle *OFF*. Ketika fitur siap, tim cukup menyalakan toggle ke *ON* via dashboard kendali dalam 1 detik. Dan jika terjadi bug, fitur dapat dimatikan seketika (*Kill Switch*) tanpa perlu melakukan deploy ulang aplikasi.

## 2. Apa Itu dan Buat Apa?

Feature Flags (juga dikenal sebagai Feature Toggles atau Feature Switches) adalah teknik rekayasa perangkat lunak yang memungkinkan tim mengubah fungsionalitas aplikasi, mengaktifkan fitur baru, atau memodifikasi alur pengguna saat *runtime* tanpa perlu mengubah atau men-deploy ulang kode sumber.

Tujuan utama Feature Flags adalah **memisahkan proses Deployment dari proses Release**:
- **Deployment**: Tindakan teknis mengunggah dan mendistribusikan kode baru ke server/CDN.
- **Release**: Tindakan bisnis mengaktifkan fitur tersebut agar dapat dilihat dan digunakan oleh pengguna.

## 3. Kenapa Harus Pakai Ini? (Empat Kategori Utama Feature Flags)

1. **Release Toggles**:
   - Memungkinkan pengembang melakukan *Trunk-Based Development* dengan mengintegrasikan kode fitur yang belum selesai (*work-in-progress*) ke branch utama tanpa mengganggu pengguna akhir. Berumur pendek (dihapus setelah fitur 100% stabil).
2. **Experimentation Toggles (A/B Testing)**:
   - Menampilkan versi antarmuka yang berbeda kepada dua kelompok pengguna yang dipilih secara acak (misal: 50% melihat tombol hijau, 50% melihat tombol biru) untuk mengukur metrik bisnis (tingkat konversi penjualan).
3. **Ops Toggles (Kill Switches)**:
   - Saklar darurat operasional. Jika fitur tertentu yang memakan komputasi berat (seperti fitur auto-complete pencarian pintar) membuat server backend kelebihan beban saat flash sale, tim devops dapat mematikan fitur tersebut seketika dari dashboard kendali untuk menyelamatkan sistem.
4. **Permission / Targeting Toggles**:
   - Membatasi akses fitur tertentu berdasarkan profil pengguna (misal: hanya pengguna tier Enterprise, karyawan internal perusahaan *dogfooding*, atau pengguna di wilayah geografis tertentu).

## 4. Cara Kerjanya

Berikut adalah diagram arsitektur bagaimana Feature Flag dievaluasi di browser pengguna:

```
[ Dashboard Konfigurasi Admin / LaunchDarkly / Unleash ]
                        |
       (Admin mengubah status toggle: "new-checkout" = ON)
                        |
                        v
[ CDN / Feature Flag Edge Evaluation Service ]
                        |
                        |-- Server-Sent Events (SSE) / Polling
                        v
[ Frontend Client Browser ]
  1. SDK mengidentifikasi pengguna: { userId: "user-123", country: "ID", tier: "VIP" }
  2. Evaluasi aturan Flag di memori lokal browser:
     - Apakah flag "new-checkout" aktif untuk user ini? -> TRUE
                        |
                        v
[ Komponen React: <CheckoutPage /> ]
  const isNewCheckoutEnabled = useFeatureFlag('new-checkout');

  if (isNewCheckoutEnabled) {
    return <OneClickCheckoutModal />; // Rute Baru Aktif!
  } else {
    return <TraditionalCheckoutForm />; // Rute Klasik
  }
```

## 5. Cuplikan Logika Inti

### 1. Custom Hook Feature Flag di React (TypeScript)

```tsx
// src/features/flags/useFeatureFlag.ts
import { createContext, useContext, useEffect, useState, ReactNode } from 'react';

type FlagMap = Record<string, boolean>;

const FeatureFlagContext = createContext<FlagMap>({});

export function FeatureFlagProvider({ children, initialFlags = {} }: { children: ReactNode; initialFlags?: FlagMap }) {
  const [flags, setFlags] = useState<FlagMap>(initialFlags);

  useEffect(() => {
    // Ambil konfigurasi flags terbaru dari Edge API saat aplikasi dimuat
    fetch('/api/v1/feature-flags', { credentials: 'include' })
      .then((res) => res.json())
      .then((data: FlagMap) => setFlags(data))
      .catch((err) => console.warn('Gagal memuat feature flags, gunakan fallback default', err));
  }, []);

  return (
    <FeatureFlagContext.Provider value={flags}>
      {children}
    </FeatureFlagContext.Provider>
  );
}

export function useFeatureFlag(flagKey: string, defaultValue: boolean = false): boolean {
  const flags = useContext(FeatureFlagContext);
  return flags[flagKey] ?? defaultValue;
}
```

### 2. Komponen Deklaratif Bersyarat

```tsx
// src/components/FeatureGate.tsx
import React, { ReactNode } from 'react';
import { useFeatureFlag } from '../features/flags/useFeatureFlag';

interface FeatureGateProps {
  flag: string;
  children: ReactNode;
  fallback?: ReactNode;
}

export function FeatureGate({ flag, children, fallback = null }: FeatureGateProps) {
  const isEnabled = useFeatureFlag(flag);

  if (isEnabled) {
    return <>{children}</>;
  }

  return <>{fallback}</>;
}
```

### 3. Penggunaan pada Komponen Bisnis

```tsx
// src/pages/CheckoutPage.tsx
import { FeatureGate } from '../components/FeatureGate';
import { NewOneClickCheckout } from './NewOneClickCheckout';
import { LegacyCheckout } from './LegacyCheckout';

export function CheckoutPage() {
  return (
    <div className="checkout-container">
      <h1>Halaman Pembayaran</h1>
      
      {/* Fitur baru dilindungi feature gate */}
      <FeatureGate
        flag="enable_one_click_checkout"
        fallback={<LegacyCheckout />}
      >
        <NewOneClickCheckout />
      </FeatureGate>
    </div>
  );
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

| Kelebihan | Kekurangan / Risiko |
| :--- | :--- |
| Memungkinkan *Trunk-Based Development* tanpa takut merusak production | Menumpuknya utang teknis (*technical debt*) jika bendera kadaluwarsa tidak segera dibersihkan |
| Kemampuan mematikan fitur bermasalah dalam 1 detik (*Instant Kill Switch*) | Meningkatkan kompleksitas pengujian karena kombinasi state percabangan kode bertambah |
| Rollout bertahap (Canary Release: 5% -> 20% -> 100% pengguna) meminimalkan risiko bisnis | Kedua versi kode (lama dan baru) harus dikemas ke dalam bundle, sedikit menambah ukuran file |

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Menggunakan Feature Flags
- Peluncuran fitur baru berskala menengah hingga besar yang berisiko tinggi terhadap alur bisnis.
- Pengujian A/B Testing untuk mengukur respons pasar terhadap perubahan antarmuka atau copy teks.
- Fitur operasional yang mungkin perlu dimatikan saat beban server melonjak (*graceful feature degradation*).
- Rilis fitur beta untuk kelompok pengguna khusus (*early adopters / internal testers*).

### Kapan Jangan Menggunakan Feature Flags
- Perbaikan bug kecil (*bug fix*) atau perbaikan salah ketik teks (*typo*) yang jelas tidak memerlukan toggle.
- Refactoring arsitektur internal menyeluruh yang tidak mengubah perilaku fungsionalitas UI.

## 8. Metrik Kunci & Observabilitas

1. **Flag Evaluation Latency**: Waktu yang dibutuhkan SDK untuk mengevaluasi status flag. Harus berada di bawah 2ms di memori client agar tidak memicu layout shift atau blocking rendering.
2. **Stale Flag Age (Days)**: Umur rata-rata feature flag di repositori. Standar industri yang sehat mengharuskan Release Toggle dihapus maksimal **30 hari** setelah fitur 100% dirilis ke publik.
3. **Kill Switch Activation Rate**: Jumlah insiden di mana tim operasional harus mematikan fitur secara darurat di production.

## 9. Studi Kasus Industri Nyata

Sebuah platform streaming musik terbesar di dunia mengadopsi budaya *Release Toggles* secara masif. Ratusan engineer melakukan merge ratusan commit ke branch `main` setiap hari, dan seluruh kode otomatis di-deploy ke production setiap beberapa jam.

Saat merilis desain antarmuka pemutar lagu baru:
1. Fitur dikunci di balik flag: `desktop_player_redesign_v2`.
2. Minggu 1: Flag diaktifkan hanya untuk 100% karyawan internal perusahaan (*Dogfooding*). Ditemukan puluhan bug kecil yang diperbaiki langsung.
3. Minggu 2: Rollout 5% pengguna di Australia dan Selandia Baru. Metrik durasi pemutaran lagu dipantau secara ketat.
4. Minggu 3: Ditemukan bug memori leak pada kartu grafis tertentu. Tim dengan tenang menurunkan rollout kembali ke 0% dalam 2 detik melalui dashboard tanpa perlu rollback deploy kode.
5. Setelah bug diperbaiki, rollout dinaikkan bertahap: 10% -> 25% -> 50% -> 100% ke seluruh dunia.
6. Risiko kerugian bisnis dan kepanikan rilis berhasil dieliminasi total.

## 10. Fakta Menarik & Gotchas Production

- **Bahaya "Zombie Flags" (Utang Teknis)**: Salah satu insiden perbankan terkenal di Amerika (Knight Capital) yang merugi \$440 juta dalam 45 menit disebabkan oleh kode feature flag usang berusia 8 tahun yang secara tidak sengaja terpicu kembali. Selalu jadwalkan tiket *cleanup* untuk menghapus kode flag dan percabangan `if/else` lama setelah fitur stabil.
- **Flicker Effect (FOOC - Flash of Original Content)**: Jika flag diambil secara asinkron dari API setelah halaman dirender, pengguna mungkin melihat komponen lama selama 200ms sebelum tiba-tiba berkedip berubah menjadi komponen baru. Atasi masalah ini dengan melakukan evaluasi flag di server (SSR / Edge Middleware) atau menyertakan default state yang tepat.
- **Batasan Akses Keamanan**: Feature Flag di sisi frontend murni bersifat kontrol presentasi UI, **bukan sistem keamanan otorisasi**. Pengguna yang cerdas tetap bisa mengubah variabel flag di JavaScript console mereka. Pastikan backend API tetap memvalidasi hak akses data secara ketat!

## 11. Rangkuman Singkat

Feature Flags memisahkan proses deployment teknis dari perilisan bisnis, memungkinkan developer mengintegrasikan kode secara berkelanjutan ke branch utama melalui Trunk-Based Development. Dengan kemampuan rollout bertahap, eksperimen A/B testing, dan saklar darurat instan (*Kill Switch*), Feature Flags meningkatkan kelincahan tim sambil menekan risiko kegagalan rilis di production.
