# MFE Resilience and Fallbacks

## 1. Skenario Sistem Nyata

Hari Jumat malam saat kampanye belanja Harbolnas (11.11), traffic pengguna ke portal e-commerce melonjak 10 kali lipat.

Server CDN milik Tim Rekomendasi Produk mengalami lonjakan beban dan down total. Setiap permintaan browser untuk mengunduh skrip `https://cdn-recs.shop.com/remoteEntry.js` mengalami **HTTP 504 Gateway Timeout (menunggu 30 detik sebelum gagal)**.

Pada aplikasi Host yang dirancang buruk:
Karena komponen remote rekomendasi diimpor langsung tanpa penanganan batas kesalahan dan batas waktu:
- Browser membeku (*freeze*) selama 30 detik menunggu skrip yang mati.
- Ketika skrip akhirnya gagal (*Script load error*), aplikasi melempar unhandled promise exception yang meruntuhkan seluruh pohon React Host.
- Pengguna disajikan layar putih kosong (*White Screen of Death*). Pengguna tidak bisa melihat keranjang belanja, tidak bisa membayar, dan tidak bisa checkout barang!

Seluruh bisnis perusahaan lumpuh hanya karena 1 remote sekunder mati. Kerugian mencapai miliaran rupiah.

Prinsip **Blast Radius Containment (Isolasi Dampak Kerusakan)** dan **Resilience Architecture** menjamin bahwa kegagalan satu Micro-App tidak akan pernah meruntuhkan aplikasi lainnya.

## 2. Apa Itu dan Buat Apa?

MFE Resilience and Fallbacks adalah arsitektur pertahanan di sisi klien yang dirancang untuk mengisolasi titik-titik kegagalan jaringan, timeout, dan runtime exception pada aplikasi remote independen, memastikan bahwa aplikasi penampung utama (Shell / Host) tetap berjalan stabil dan fungsional.

Tujuan utamanya adalah menerapkan **Graceful Degradation (Penurunan Fungsionalitas yang Anggun)**:
Jika modul sekunder (seperti Banner Iklan, Rekomendasi Produk, atau Widget Chat Bantuan) mengalami down, modul tersebut cukup digantikan dengan kartu pesan alternatif atau disembunyikan sama sekali (*hidden silently*), sementara alur transaksi utama (katalog, keranjang, pembayaran) tetap beroperasi 100%.

## 3. Kenapa Harus Pakai Ini? (Empat Pilar Ketahanan Micro Frontends)

1. **Independent Error Boundaries per Container**:
   - Setiap Micro-App remote wajib dibungkus di dalam Error Boundary tersendiri di level Host.
2. **Network Timeout & Retry Mechanism**:
   - Pemuatan skrip remote jaringan (`remoteEntry.js`) dibatasi dengan batas waktu maksimal (misal 5 detik). Jika melebihi batas waktu, eksekusi diputus dan dilakukan percobaan ulang (*retry with exponential backoff*).
3. **Frontend Circuit Breaker Pattern**:
   - Jika remote tertentu gagal dimuat 3 kali berturut-turut untuk pengguna, sistem "membuka sirkuit" (*trip the circuit breaker*) dan berhenti meminta remote tersebut selama 5 menit, langsung menyajikan UI fallback tanpa membebani jaringan pengguna.
4. **Offline / Static Cache Fallback**:
   - Menyimpan versi stabil terakhir komponen remote di Service Worker Cache API sebagai cadangan darurat jika CDN remote mati.

## 4. Cara Kerjanya

Berikut adalah diagram alur bagaimana Host App mengisolasi kegagalan remote menggunakan **Timeout Wrapper, Circuit Breaker, dan Error Boundary**:

```
[ HOST APP MERENDER HALAMAN PRODUK ]
                 |
                 +-----------------------------------------------+
                 |                                               |
                 v                                               v
       [ MFE-Checkout: Stabil ]                      [ MFE-Promo: CDN Timeout / 504 ]
                 |                                               |
                 |                                    (Menunggu hingga batas waktu 3 detik)
                 v                                               |
       (Berhasil dimuat 200 OK)                                  v
                 |                                    [ TIMEOUT CIRCUIT BREAKER ]
                 v                                    - Batalkan request jaringan
         [ LAYAR CHECKOUT AKTIF ]                     - Buka sirkuit kegagalan
         (Pengguna bisa bayar)                                   |
                                                                 v
                                                      [ MFE ERROR BOUNDARY ]
                                                      - Tangkap script load error
                                                      - Render Fallback UI:
                                                        [ "Promo tidak tersedia sementara" ]
                                                                 |
                                                                 v
                                             [ SISTEM TETAP STABIL 100% ]
                                             (Transaksi belanja sama sekali tidak terganggu!)
```

## 5. Cuplikan Logika Inti

### Dynamic Remote Loader dengan Timeout dan Retry (TypeScript)

Jangan gunakan `import('remote/Component')` mentah tanpa perlindungan timeout. Gunakan wrapper promise berbatas waktu:

```typescript
// host-app/src/utils/federationLoader.ts
import React from 'react';

interface LoadRemoteOptions {
  timeoutMs?: number;
  retries?: number;
}

// Fungsi pembungkus impor dinamis berbatas waktu
export function loadRemoteWithTimeout<T>(
  importFn: () => Promise<T>,
  options: LoadRemoteOptions = {}
): Promise<T> {
  const { timeoutMs = 4000, retries = 2 } = options;

  return new Promise((resolve, reject) => {
    // Buat timer timeout
    const timer = setTimeout(() => {
      reject(new Error(`Timeout: Remote container gagal merespons dalam ${timeoutMs}ms`));
    }, timeoutMs);

    importFn()
      .then((module) => {
        clearTimeout(timer);
        resolve(module);
      })
      .catch((error) => {
        clearTimeout(timer);
        if (retries > 0) {
          console.warn(`Gagal memuat remote, mencoba ulang... (${retries} tersisa)`);
          resolve(loadRemoteWithTimeout(importFn, { timeoutMs, retries: retries - 1 }));
        } else {
          reject(error);
        }
      });
  });
}
```

### Mengintegrasikan Fallback di Komponen Host

```tsx
// host-app/src/pages/ProductPage.tsx
import React, { Suspense } from 'react';
import { ErrorBoundary } from '../components/ErrorBoundary';
import { loadRemoteWithTimeout } from '../utils/federationLoader';

// Muat komponen remote dengan pelindung batas waktu 3 detik
const RemoteRecommendationWidget = React.lazy(() =>
  loadRemoteWithTimeout(() => import('mfe_recs/RecommendationWidget'), {
    timeoutMs: 3000,
    retries: 1,
  })
);

export function ProductPage() {
  return (
    <div className="product-layout">
      {/* 1. Komponen Inti Produk (Wajib Ada) */}
      <main className="product-main">
        <h1>Kamera Mirrorless Pro</h1>
        <button className="btn-buy">Beli Sekarang</button>
      </main>

      {/* 2. Komponen Sekunder Dilindungi Boundary & Timeout */}
      <aside className="product-sidebar">
        <ErrorBoundary
          fallback={
            <div className="rec-fallback-card">
              <p className="text-gray-500 text-sm">Rekomendasi produk saat ini tidak tersedia.</p>
            </div>
          }
        >
          <Suspense fallback={<div className="skeleton-loader">Memuat rekomendasi...</div>}>
            <RemoteRecommendationWidget />
          </Suspense>
        </ErrorBoundary>
      </aside>
    </div>
  );
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

| Kelebihan | Kekurangan / Risiko |
| :--- | :--- |
| Mengisolasi dampak kerusakan (*blast radius*): kegagalan modul non-kritis tidak mematikan sistem | Memerlukan penulisan komponen fallback dan skeleton loading untuk setiap Micro-App |
| Mencegah halaman utama hang akibat latency jaringan CDN remote yang lambat | Logika retry dan timeout wrapper menambah sedikit kompleksitas arsitektur pemuatan Host |
| Memberikan perlindungan ketersediaan (*High Availability*) tingkat enterprise | Jika terlalu toleran terhadap fallback, tim mungkin lambat menyadari adanya remote yang rusak |

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai (Fallback vs Block)

### Kapan Boleh Graceful Fallback (Komponen Sekunder)
- Komponen rekomendasi barang, banner promosi, ulasan nasabah, widget chatbot, integrasi media sosial.
- *Aturan*: Jika ketiadaan komponen ini tidak menghalangi pengguna untuk menyelesaikan misi utama, selalu gunakan fallback senyap atau pesan alternatif.

### Kapan Wajib Block / Full Error Screen (Komponen Kritis)
- Jika modul yang gagal adalah modul inti yang tidak tergantikan: modul Form Pembayaran pada halaman Checkout, atau modul Otentikasi pada halaman Login. Jika modul ini gagal, tampilkan pesan error layar penuh dengan tombol *Reload*.

## 8. Metrik Kunci & Observabilitas

1. **Remote Container Availability Ratio**: Persentase keberhasilan pengunduhan `remoteEntry.js` dari CDN. Targetkan ketersediaan minimal **99.9% (Three Nines)**.
2. **Circuit Breaker Trip Count**: Frekuensi terpicunya circuit breaker pada masing-masing remote app. Jika remote Tim Promo sering terpicu, lakukan audit kapasitas CDN mereka.
3. **Fallback Impression Rate**: Persentase sesi pengguna yang disajikan tampilan fallback UI dibandingkan tampilan komponen remote asli.

## 9. Studi Kasus Industri Nyata

Sebuah platform streaming olahraga internasional memiliki jutaan penonton bersamaan saat final kejuaraan sepak bola. Aplikasi Shell menampilkan pemutar video utama dan widget live chat interaktif di samping video.

Server WebSocket dan CDN milik Tim Live Chat mengalami crash akibat kelebihan beban pesan. Skrip remote chat macet dan melempar exception fatal.

Di sistem lama tanpa isolasi:
Bug pada chat membuat pemutar video ikut mati, memicu kemarahan ratusan ribu penonton yang sedang menonton siaran langsung.

Solusi rekayasa:
1. Membungkus remote chat ke dalam `<ErrorBoundary>` mandiri dengan batas waktu timeout 2 detik.
2. Saat chat gagal, Error Boundary langsung merender fallback sederhana: *"Obrolan langsung dinonaktifkan sementara untuk menjaga stabilitas video"*.
3. Pemutar video utama tetap berjalan mulus tanpa gangguan satu detik pun. Kredibilitas platform streaming terselamatkan.

## 10. Fakta Menarik & Gotchas Production

- **Masalah Versi Tertinggal (Version Skew)**: Jika Host App masih menyimpan cache HTML lama yang merujuk ke chunk hash remote yang sudah dihapus dari CDN tim remote, browser akan menerima error 404. Mitigasinya: Tim remote wajib mempertahankan file hash lama di CDN minimal selama 24-48 jam setelah rilis versi baru (*backward asset retention*).
- **Retry dengan Jitter**: Jangan mencoba ulang (*retry*) pada detik yang sama persis untuk ribuan pengguna saat CDN down, karena ini akan menciptakan badai request (*Thundering Herd Problem*). Tambahkan acakan jeda waktu (*exponential backoff with jitter*).
- **Service Worker Stale Fallback**: Anda dapat mengonfigurasi Service Worker untuk menyajikan versi cache `remoteEntry.js` kemarin jika request jaringan hari ini mengembalikan error 500.

## 11. Rangkuman Singkat

MFE Resilience menjamin bahwa batas kerusakan kegagalan jaringan atau crash runtime pada sub-aplikasi terisolasi secara lokal melalui kombinasi Error Boundary, timeout wrapper, dan circuit breaker. Pola penurunan fungsionalitas anggun (*Graceful Degradation*) memastikan alur bisnis utama tetap terlindungi meskipun layanan pihak ketiga atau tim lain mengalami down.
