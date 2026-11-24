# Error Boundaries

## 1. Skenario Sistem Nyata

Seorang pengguna sedang membuka aplikasi dashboard perbankan online yang berisi banyak widget: ringkasan saldo tabungan, formulir mutasi transfer, widget grafik investasi reksadana, dan daftar banner promo diskon.

Widget grafik reksadana mengalami bug rendering JavaScript kecil: backend mengembalikan nilai persentase `null`, dan komponen mencoba membaca `null.toFixed(2)` tanpa optional chaining.

Di aplikasi React standar tanpa penanganan batas kesalahan:
Error rendering yang terjadi pada satu widget kecil grafik tersebut akan **menghancurkan seluruh pohon komponen (*unmounts the entire React component tree*)**. Seluruh layar pengguna mendadak lenyap berganti menjadi layar putih polos tanpa teks (*White Screen of Death*). Pengguna yang sedang mengisi form transfer panik dan tidak bisa bertransaksi.

Dengan menerapkan arsitektur **Error Boundaries**, hanya widget grafik reksadana yang menampilkan kotak pesan error mini: *"Gagal memuat grafik"*, sementara fitur transfer uang dan saldo tabungan tetap berfungsi normal dan dapat digunakan pengguna.

## 2. Apa Itu dan Buat Apa?

Error Boundary adalah komponen khusus dalam arsitektur berbasis komponen (seperti React) yang bertugas menangkap (*catch*) kesalahan JavaScript yang terjadi di mana saja di dalam pohon komponen turunannya (*child components*), mencatat kesalahan tersebut ke layanan monitoring, dan menampilkan antarmuka cadangan (*fallback UI*) alih-alih membiarkan seluruh aplikasi crash.

Karakteristik penting:
- Mengisolasi kegagalan rendering (*failure containment*).
- Memungkinkan aplikasi menerapkan prinsip penurunan fungsionalitas secara anggun (*Graceful Degradation*).
- Menyediakan mekanisme pemulihan (*error recovery / reset state*) bagi pengguna tanpa perlu memuat ulang seluruh halaman website.

## 3. Kenapa Harus Pakai Ini?

- **Mencegah White Screen of Death (WSoD)**: Sejak React 16+, unhandled error yang terjadi selama fase rendering akan menyebabkan seluruh root React di-unmount secara default. Error boundary adalah satu-satunya benteng pertahanan resmi untuk menghentikan efek domino ini.
- **Granularitas Kerusakan UI**: Anda bisa menempatkan Error Boundary di level mikro (di setiap widget independen) sehingga jika widget ulasan rusak, halaman katalog produk dan tombol "Beli" tetap berjalan sempurna.
- **Observabilitas Otomatis**: Menjadi gerbang sentral untuk menangkap metadata error, stack trace, dan *componentStack* untuk dikirimkan secara otomatis ke platform crash monitoring (seperti Sentry atau Datadog).

## 4. Cara Kerjanya

Berikut adalah diagram perbandingan antara aplikasi tanpa Error Boundary vs aplikasi yang dilindungi oleh Error Boundary berlapis:

```
--- 1. TANPA ERROR BOUNDARY (CRASH TOTAL) ---
[ Root Application ]
   +-- [ Navbar ]
   +-- [ Main Content ]
   |      +-- [ Transfer Form ]
   |      +-- [ Crypto Widget ] -> MELEMPAR UNCAUGHT ERROR!
   +-- [ Footer ]
              |
              v (Error merembet ke seluruh pohon komponen)
[ SELURUH APLIKASI DI-UNMOUNT -> LAYAR PUTIH KOSONG / WSOD ]


--- 2. DENGAN ERROR BOUNDARY TERISOLASI ---
[ Root Application ] (Dilindungi Global Error Boundary)
   +-- [ Navbar ] (Berfungsi Normal)
   +-- [ Main Content ]
   |      +-- [ Transfer Form ] (Berfungsi Normal!)
   |      +-- <ErrorBoundary fallback={<WidgetErrorFallback />}>
   |             [ Crypto Widget ] -> MELEMPAR UNCAUGHT ERROR!
   |          </ErrorBoundary>
   |             |
   |             v (Error ditangkap dan diisolasi oleh Boundary)
   |          Menampilkan: [ "Gagal memuat grafik. Coba Lagi" ]
   +-- [ Footer ] (Berfungsi Normal)
```

## 5. Cuplikan Logika Inti

### Implementasi Komponen Error Boundary (React TypeScript)

Di React, Error Boundary wajib ditulis menggunakan Class Component karena membutuhkan lifecycle `getDerivedStateFromError` dan `componentDidCatch` (belum ada padanan Hook resmi untuk menangkap render error):

```tsx
// src/components/ErrorBoundary.tsx
import React, { Component, ErrorInfo, ReactNode } from 'react';

interface Props {
  children: ReactNode;
  fallback?: ReactNode | ((props: { error: Error; resetError: () => void }) => ReactNode);
  onError?: (error: Error, errorInfo: ErrorInfo) => void;
}

interface State {
  hasError: boolean;
  error: Error | null;
}

export class ErrorBoundary extends Component<Props, State> {
  constructor(props: Props) {
    super(props);
    this.state = { hasError: false, error: null };
  }

  // 1. Memperbarui state agar render berikutnya menampilkan fallback UI
  static getDerivedStateFromError(error: Error): State {
    return { hasError: true, error };
  }

  // 2. Mencatat informasi error dan component stack trace
  componentDidCatch(error: Error, errorInfo: ErrorInfo) {
    console.error('Error Boundary menangkap exception:', error, errorInfo);
    
    // Panggil callback pelaporan (misal kirim ke Sentry)
    this.props.onError?.(error, errorInfo);
  }

  // 3. Fungsi reset untuk memulihkan state komponen anak
  resetError = () => {
    this.setState({ hasError: false, error: null });
  };

  render() {
    if (this.state.hasError && this.state.error) {
      // Jika fallback berupa fungsi render props
      if (typeof this.props.fallback === 'function') {
        return this.props.fallback({
          error: this.state.error,
          resetError: this.resetError,
        });
      }

      // Jika fallback berupa elemen JSX statis
      if (this.props.fallback) {
        return this.props.fallback;
      }

      // Fallback bawaan jika tidak disediakan props
      return (
        <div role="alert" className="p-4 border border-red-300 bg-red-50 rounded">
          <h3 className="text-red-800 font-semibold">Komponen Mengalami Kendala</h3>
          <p className="text-red-600 text-sm mt-1">{this.state.error.message}</p>
          <button
            onClick={this.resetError}
            className="mt-3 px-3 py-1 bg-red-600 text-white text-sm rounded hover:bg-red-700"
          >
            Muat Ulang Komponen
          </button>
        </div>
      );
    }

    return this.props.children;
  }
}
```

### Praktik Penerapan Berlapis di Halaman Dashboard

```tsx
// src/pages/DashboardPage.tsx
import { ErrorBoundary } from '../components/ErrorBoundary';
import { BalanceWidget } from './BalanceWidget';
import { AnalyticsChart } from './AnalyticsChart';
import { RecentTransactions } from './RecentTransactions';

export function DashboardPage() {
  return (
    <div className="dashboard-grid">
      {/* 1. Widget Finansial Utama dilindungi boundary mandiri */}
      <ErrorBoundary>
        <BalanceWidget />
      </ErrorBoundary>

      {/* 2. Widget Grafik pihak ketiga yang rentan dilindungi boundary sendiri */}
      <ErrorBoundary
        fallback={({ resetError }) => (
          <div className="chart-error-card">
            <p>Grafik gagal ditampilkan.</p>
            <button onClick={resetError}>Coba Lagi</button>
          </div>
        )}
      >
        <AnalyticsChart />
      </ErrorBoundary>

      {/* 3. Daftar Transaksi */}
      <ErrorBoundary>
        <RecentTransactions />
      </ErrorBoundary>
    </div>
  );
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

| Kelebihan | Kekurangan / Risiko |
| :--- | :--- |
| Mengisolasi crash sehingga fitur aplikasi lainnya tetap dapat digunakan | Tidak menangkap error di dalam Event Handler (`onClick`), asynchronous code (`setTimeout`), atau SSR |
| Memberikan pengalaman UX yang ramah dengan pesan fallback informatif | Memerlukan perencanaan hierarki komponen yang matang agar tampilan fallback tidak merusak layout visual |
| Memungkinkan pengguna memulihkan komponen (*retry*) tanpa me-refresh halaman | Masih membutuhkan penulisan Class Component di era ekosistem React Hooks modern |

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai (Batasan Error Boundary)

Error Boundary secara spesifik **hanya menangkap error yang terjadi selama siklus hidup deklaratif React**:
- Render phase.
- Lifecycle methods (`componentDidMount`, `useEffect`).
- Konstruktor komponen anak.

Error Boundary **TIDAK BISA MENANGKAP**:
1. **Event Handlers (`onClick`, `onSubmit`)**:
   - Jika error terjadi saat tombol diklik, React tidak me-unmount layar. Gunakan blok `try / catch` standar di dalam event handler tersebut.
2. **Asynchronous Code**:
   - Error di dalam callback `setTimeout`, `requestAnimationFrame`, atau unhandled promise rejection.
3. **Server-Side Rendering (SSR)**:
   - Error yang terjadi saat proses HTML rendering di server Node.js.

## 8. Metrik Kunci & Observabilitas

1. **Boundary Catch Rate per Feature**: Pantau metrik berapa kali masing-masing Error Boundary terpicu dalam sehari. Jika Boundary pada fitur "Checkout" sering menangkap error, itu adalah sinyal darurat *high-priority bug*.
2. **User Recovery Rate**: Persentase pengguna yang berhasil melanjutkan aktivitas setelah menekan tombol "Coba Lagi" / *Reset Error* pada fallback UI.
3. **Component Stack Trace Quality**: Pastikan source maps production terunggah ke Sentry agar informasi `componentStack` menunjukkan nama file dan baris komponen React yang asli, bukan kode minifikasi (*minified chunk*).

## 9. Studi Kasus Industri Nyata

Sebuah platform marketplace global mengintegrasikan widget promosi pihak ketiga (Third-party Recommendation Carousel) di halaman detail produk.

Saat kampanye diskon tanggal kembar, vendor pihak ketiga secara tidak sengaja merilis payload JSON yang memiliki format array rusak. Tanpa perlindungan Error Boundary, bug pada carousel rekomendasi tersebut meruntuhkan seluruh halaman detail produk. Ratusan ribu pembeli di seluruh dunia disajikan layar putih kosong, dan tombol "Masukkan Keranjang" tidak bisa diakses selama 40 menit.

Kerugian penjualan ditaksir mencapai jutaan dolar.

Solusi engineering pasca insiden:
1. Membungkus seluruh integrasi komponen pihak ketiga dan widget sekunder ke dalam `<ErrorBoundary>`.
2. Jika widget pihak ketiga melempar exception, fallback yang dirender adalah string kosong (`null`), sehingga halaman tampak normal tanpa carousel tersebut tanpa merusak fungsionalitas pembelian utama.
3. Insiden kegagalan vendor pihak ketiga tidak pernah lagi mengganggu alur checkout pelanggan.

## 10. Fakta Menarik & Gotchas Production

- **Library Komunitas yang Populer**: Anda tidak perlu membuat class component manual dari nol. Di industri, library `react-error-boundary` karya Kent C. Dodds adalah standar de facto yang menyediakan hook `useErrorBoundary()` dan komponen deklaratif yang sangat fleksibel.
- **Trik Menangkap Async Error via Error Boundary**: Anda bisa meneruskan error asinkron (seperti error fetch di dalam event handler) ke Error Boundary terdekat dengan cara memanggil updater state React:
  ```typescript
  const [_, setError] = useState();
  // Memanggil setError di dalam catch akan melempar error ke render phase:
  fetchData().catch((err) => setError(() => { throw err; }));
  ```
- **Bahaya Fallback yang Ikut Error**: Pastikan komponen Fallback UI yang Anda tulis sesederhana mungkin (hindari pemanggilan hooks kompleks atau komputasi rumit). Jika komponen Fallback itu sendiri melempar error, Error Boundary di atasnya yang akan terpicu.

## 11. Rangkuman Singkat

Error Boundary adalah komponen pertahanan arsitektur React yang mengisolasi kegagalan rendering pada level komponen tertentu, mencegah terjadinya White Screen of Death, dan menyajikan UI alternatif yang elegan dengan kemampuan reset mandiri. Terapkan Error Boundary secara berlapis (di level rute halaman dan di level widget independen) untuk menjamin ketahanan aplikasi di production.
