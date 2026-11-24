# Frontend Monitoring & Error Tracking

## 1. Skenario Sistem Nyata

Sebuah platform perbankan digital meluncurkan pembaruan antarmuka pada hari Senin. Di dashboard server backend, seluruh metrik sistem tampak normal: CPU server 20%, status response HTTP 200 OK, dan tidak ada database error sama sekali. Tim merasa peluncuran sukses besar.

Namun di media sosial Twitter/X dan Google Play Store, ratusan pengguna marah dan memberikan bintang 1:
"Aplikasi kalian rusak! Tombol konfirmasi transfer uang di HP Samsung Galaxy saya tidak bisa dipencet sama sekali!"

Tim backend tidak melihat masalah apa pun karena request memang tidak pernah sampai ke server. Kesalahan terjadi di browser pengguna: script JavaScript melempar `Uncaught TypeError: window.crypto.randomUUID is not a function` pada browser Android WebView versi lama.

Tanpa sistem **Frontend Monitoring & Error Tracking (RUM)**, tim engineering seperti terbang buta di malam hari tanpa instrumen radar, baru menyadari masalah setelah reputasi perusahaan hancur di mata publik.

## 2. Apa Itu dan Buat Apa?

Frontend Monitoring & Error Tracking adalah praktik instrumentasi otomatis di sisi klien untuk mendeteksi, menangkap, mengagregasikan, dan menganalisis kesalahan JavaScript (crash/exceptions), penurunan performa web (Web Vitals), serta interaksi pengguna secara *real-time* dari perangkat pengguna nyata di seluruh dunia (*Real User Monitoring / RUM*).

Alat pemantau frontend modern (seperti Sentry, Datadog RUM, LogRocket, atau OpenTelemetry) mengumpulkan:
1. **Crash Reports & Stack Traces**: Pesan error lengkap yang dipetakan kembali ke baris kode asli developer menggunakan *Source Maps*.
2. **Breadcrumbs (Jejak Rekam Interaksi)**: Rekaman urutan aksi pengguna sesaat sebelum crash terjadi (misal: klik tombol -> navigasi rute -> panggilan HTTP fetch -> crash).
3. **Session Replay**: Rekaman visual seperti video yang merekonstruksi pergerakan mouse dan klik pengguna di DOM untuk mereplikasi bug yang sulit ditiru.
4. **User Context & Environment**: Informasi tipe browser, sistem operasi, resolusi layar, kecepatan koneksi internet, dan ID pengguna.

## 3. Kenapa Harus Pakai Ini?

- **Menghilangkan Titik Buta Sisi Klien**: Server backend hanya tahu apa yang sampai ke gerbangnya. 60% kegagalan web modern terjadi sepenuhnya di browser (kegagalan rendering CSS, unhandled promise, script pihak ketiga yang crash, DOM mutation error).
- **Waktu Penyelesaian Bug Lebih Cepat (Lower MTTR)**: Developer tidak perlu menebak-nebak atau meminta pengguna mengirimkan screenshot. Sentry langsung menyajikan file, nomor baris, dan alur klik yang memicu error.
- **Deteksi Masalah Segmentasi Perangkat**: Mengetahui dengan cepat jika ada bug yang hanya terjadi pada tipe browser tertentu (misal: hanya di Safari iOS 16 atau browser bawaan Xiaomi).

## 4. Cara Kerjanya

Berikut adalah diagram alur bagaimana SDK monitoring (seperti Sentry) menangkap error di browser pengguna dan merekonstruksinya untuk developer:

```
[ PENGGUNA DI PRODUCTION: CHROME BROWSER ]
                  |
     Terjadi Unhandled Error di JavaScript!
                  |
                  v
[ Sentry SDK Client Listener ]
- window.onerror & window.onunhandledrejection
- Kumpulkan Breadcrumbs (klik terakhir, rute, log)
- Samarkan data sensitif pengguna (PII Masking)
                  |
                  |-- Kirim Payload JSON via Beacon / Fetch
                  v
[ SENTRY INGESTION SERVER (CLOUD) ]
                  |
                  |-- Mencocokkan file minifikasi (chunk-a1b2.js:1:3400)
                  |   dengan Private Source Maps yang diunggah saat CI/CD build
                  v
[ SENTRY DASHBOARD DEVELOPER ]
Menampilkan Stack Trace Asli yang Terbaca:
  File: src/features/transfer/TransferForm.tsx, line 42
  Error: "Pilihan rekening tidak ditemukan"
                  |
                  v
[ Notifikasi Otomatis ke Slack / Jira Developer ]
```

## 5. Cuplikan Logika Inti

### 1. Inisialisasi Sentry SDK di Frontend (React + TypeScript)

```typescript
// src/monitoring/sentry.ts
import * as Sentry from '@sentry/react';

export function initMonitoring() {
  if (process.env.NODE_ENV !== 'production') {
    return; // Matikan monitoring di lingkungan development lokal
  }

  Sentry.init({
    dsn: import.meta.env.VITE_SENTRY_DSN,
    integrations: [
      // Pelacakan performa navigasi halaman
      Sentry.browserTracingIntegration(),
      // Session Replay: rekam visual interaksi DOM saat terjadi error
      Sentry.replayIntegration({
        maskAllText: true,       // Samarkan seluruh teks untuk privasi
        blockAllMedia: true,      // Blokir gambar dan video
      }),
    ],
    // Konfigurasi sampling rate agar hemat kuota kuota event
    tracesSampleRate: 0.1,        // Rekam performa 10% dari total sesi pengguna
    replaysOnErrorSampleRate: 1.0, // Rekam 100% video sesi jika terjadi crash!
    replaysSessionSampleRate: 0.01, // Rekam 1% sesi normal

    // Sanitasi data sensitif sebelum dikirim ke server monitoring
    beforeSend(event) {
      if (event.request?.headers) {
        delete event.request.headers['Authorization'];
        delete event.request.headers['Cookie'];
      }
      return event;
    },
  });
}
```

### 2. Pengelolaan Source Maps Aman di CI/CD (Vite Plugin)

Source maps wajib diunggah ke Sentry agar error bisa dibaca oleh developer, namun **DILARANG KERAS** dibiarkan publik di server hosting production karena hacker bisa melihat seluruh kode sumber asli Anda:

```typescript
// vite.config.ts
import { defineConfig } from 'vite';
import { sentryVitePlugin } from '@sentry/vite-plugin';

export default defineConfig({
  build: {
    sourcemap: true, // Wajib menghasilkan source map saat build
  },
  plugins: [
    sentryVitePlugin({
      org: 'perusahaan-fintech',
      project: 'web-app',
      authToken: process.env.SENTRY_AUTH_TOKEN,
      sourcemaps: {
        // Otomatis mengunggah source maps ke Sentry saat CI build,
        // lalu menghapus file *.map dari folder dist/ sebelum di-deploy ke publik!
        filesToDeleteAfterUpload: ['./dist/**/*.map'],
      },
    }),
  ],
});
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

| Kelebihan | Kekurangan / Risiko |
| :--- | :--- |
| Visibilitas real-time terhadap seluruh masalah dan performa di tangan pengguna asli | Menambah sedikit ukuran bundle JavaScript aplikasi (~20-30KB gzipped) |
| Mengetahui akar masalah bug (*root cause*) dalam hitungan menit tanpa tebak-tebakan | Risiko kebocoran data privasi pengguna (PII) jika penyaringan masking tidak ketat |
| Fitur Session Replay memangkas waktu reproduksi bug oleh tim developer | Bahaya *Alert Fatigue* (kebanjiran notifikasi spam) jika filter alert tidak disetel dengan benar |

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Wajib Menggunakan RUM Monitoring
- Seluruh aplikasi web production publik maupun internal korporat.
- Aplikasi transaksi keuangan, kesehatan, perbelanjaan, atau SaaS B2B.

### Kebijakan Perlindungan Privasi (PII Sanitization)
- **Aturan Privasi Nomor Satu**: Jangan pernah merekam data pribadi sensitif (Personally Identifiable Information / PII). Pastikan seluruh kolom input password, nomor KTP, nomor kartu kredit, dan PIN disetel dengan atribut `data-sentry-mask` atau `maskAllInputs: true`.

## 8. Metrik Kunci & Observabilitas

1. **Crash-Free Users Percentage**: Persentase pengguna unik yang tidak pernah mengalami crash sama sekali dalam periode tertentu. Targetkan standar industri minimal **99.5% crash-free users**.
2. **Unhandled Rejection Volume**: Jumlah error asinkron yang tidak ditangkap oleh blok catch. Tren kenaikan metrik ini adalah tanda regresi rilis baru.
3. **Session Replay Error Rate**: Frekuensi sesi pengguna yang memicu perekaman replay error.

## 9. Studi Kasus Industri Nyata

Sebuah platform pemesanan tiket penerbangan merilis fitur pemilih kursi baru. Rata-rata transaksi checkout tiba-tiba anjlok 25%, namun tim QA internal yang mencoba di laptop Mac kantor tidak menemukan masalah apa pun.

Tim memeriksa Sentry Dashboard:
1. Terdeteksi lonjakan 12.000 error dalam 1 jam pertama: `ReferenceError: Intl.Segmenter is not defined`.
2. Dari panel filtering Sentry, 100% korban adalah pengguna browser Firefox versi lama dan browser Android tertentu yang belum mendukung API internasionalisasi teks baru tersebut.
3. Developer melihat source map baris kode yang rusak di Sentry, menulis polyfill fallback untuk browser tersebut dalam 15 menit, dan merilis hotfix.
4. Metrik transaksi langsung pulih ke kondisi normal dalam waktu kurang dari 1 jam sejak deteksi.

## 10. Fakta Menarik & Gotchas Production

- **Mitigasi Alert Fatigue**: Jika Sentry mengirim notifikasi ke Slack setiap kali ada 1 error muncul, tim developer akan cepat merasa lelah dan mulai mengabaikan saluran Slack tersebut (*alert fatigue*). Atur aturan alert di Sentry: hanya kirim notifikasi jika suatu error terjadi lebih dari 100 kali dalam 5 menit, atau memengaruhi lebih dari 1% pengguna aktif.
- **Bahaya Browser Extensions Pihak Ketiga**: Sebagian besar error yang ditangkap di browser pengguna berasal dari ekstensi nakal (ad-blocker rusak, translator, spell-checker). Gunakan fitur *Inbound Filters* di Sentry untuk memfilter error yang berasal dari skrip `chrome-extension://` agar dashboard Anda tidak dipenuhi error sampah.
- **Navigator Beacon**: SDK monitoring modern menggunakan API browser `navigator.sendBeacon()` untuk mengirim data error saat pengguna menutup tab, menjamin data telemetri tetap terkirim tanpa menahan browser saat navigasi.

## 11. Rangkuman Singkat

Frontend Monitoring dan Error Tracking memberikan visibilitas penuh terhadap kesehatan aplikasi di browser pengguna nyata (RUM). Menggunakan platform seperti Sentry dengan pengelolaan source maps yang aman, penyaringan data privasi PII, dan Session Replay memungkinkan tim pengembang mendeteksi serta menyelesaikan bug produksi sebelum disadari oleh sebagian besar pelanggan.
