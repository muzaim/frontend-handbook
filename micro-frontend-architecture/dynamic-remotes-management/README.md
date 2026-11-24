# Dynamic Remotes Management

## 1. Skenario Sistem Nyata

Sebuah organisasi perbankan enterprise memiliki 10 Micro-App yang dikembangkan oleh 10 tim berbeda. Pada konfigurasi awal Webpack Module Federation, URL setiap remote di-hardcode di dalam file `webpack.config.js` milik Host App:

```javascript
remotes: {
  mfe_cards: 'mfe_cards@https://cards.prod.bank.com/remoteEntry.js',
  mfe_loans: 'mfe_loans@https://loans.prod.bank.com/remoteEntry.js',
}
```

Arsitektur hardcode statis ini segera menjadi mimpi buruk operasional:
1. **Multi-Environment Hell**: Saat diuji di laptop developer (`localhost`), di server Staging, dan di Production, Host App harus di-build ulang sebanyak 3 kali hanya untuk mengganti string domain URL remote.
2. **Keterikatan Rilis**: Jika Tim Kartu Kredit ingin memindahkan server CDN mereka dari AWS CloudFront ke Cloudflare, Tim Host terpaksa harus ikut merilis ulang dan men-deploy aplikasi Shell mereka.
3. **Mustahil Melakukan Canary Release**: Tim Pinjaman ingin merilis versi baru Micro-App (`v2.0.0`) hanya ke 10% nasabah terpilih untuk menguji stabilitas sistem sebelum rilis 100%. Dengan URL statis yang di-hardcode saat build time, hal ini mustahil dilakukan.

Solusi arsitektur modern adalah beralih dari remote statis ke **Dynamic Remotes Management (Runtime Service Discovery & Manifest Loading)**.

## 2. Apa Itu dan Buat Apa?

Dynamic Remotes Management adalah pola arsitektur Micro Frontends di mana lokasi URL, versi, dan metadata dari remote container (`remoteEntry.js`) tidak lagi dikunci saat proses build time, melainkan dimuat, dievaluasi, dan diinisialisasi secara dinamis saat aplikasi Host sedang aktif berjalan di browser pengguna (*runtime*).

Pilar utama arsitektur ini:
- **Frontend Service Discovery (Manifest API)**: Host App memanggil endpoint konfigurasi ringan (misal `GET /api/v1/mfe-manifest`) saat startup untuk mendapatkan daftar URL remote aktif yang disesuaikan dengan lingkungan saat itu (Dev, Staging, Prod) dan profil pengguna.
- **Canary & A/B Deployments**: Backend manifest dapat mengembalikan URL versi baru (`https://cdn.com/v2/remoteEntry.js`) untuk 10% pengguna, dan versi stabil (`https://cdn.com/v1/remoteEntry.js`) untuk 90% pengguna lainnya.
- **Low-Level Container API**: Memanfaatkan antarmuka native Module Federation (`window[scope].init()` dan `window[scope].get()`) untuk menyuntikkan skrip remote ke DOM secara terprogram.

## 3. Kenapa Harus Pakai Ini?

- **Build Sekali, Deploy di Mana Saja (Build Once, Run Anywhere)**: Bundle aplikasi Host yang dihasilkan di CI bersifat agnostik lingkungan. Image docker / bundle statis yang sama dapat diuji di staging lalu langsung dipromosikan ke production tanpa perlu kompilasi ulang.
- **Independensi Rilis Mutlak**: Tim remote bebas mengubah lokasi hosting, mengganti CDN, atau merilis versi baru kapan saja. Host App akan otomatis membaca lokasi terbaru dari manifest API tanpa perlu menyentuh kode Shell.
- **Dukungan Multi-Region & Latensi Rendah**: Pengguna di Asia Tenggara otomatis diarahkan ke URL CDN Singapura, sedangkan pengguna di Eropa diarahkan ke URL CDN Frankfurt melalui respons manifest yang sadar geolokasi (*Geo-DNS aware*).

## 4. Cara Kerjanya

Berikut adalah diagram alur bagaimana Host App memuat remote secara dinamis saat runtime:

```
[ BROWSER MEMBUKA HOST SHELL APP ]
                |
                |-- 1. GET /api/v1/mfe-manifest (Kirim session cookie & user ID)
                v
[ API GATEWAY / EDGE WORKER CONFIG SERVICE ]
Mengevaluasi targeting rules:
- Lingkungan: Production
- User: Segment Beta Tester (Arahkan ke versi v2!)
Mengembalikan JSON:
{
  "mfe_loans": "https://cdn.bank.com/loans/v2/remoteEntry.js",
  "mfe_cards": "https://cdn.bank.com/cards/v1/remoteEntry.js"
}
                |
                v
[ HOST DYNAMIC CONTAINER LOADER ]
2. Suntikkan tag <script src="https://cdn.bank.com/loans/v2/remoteEntry.js"> ke DOM
                |
                v
3. Tunggu skrip selesai diunduh oleh browser
                |
                v
4. Inisialisasi container Module Federation:
   await window.mfe_loans.init(__webpack_share_scopes__.default)
                |
                v
5. Ekstrak komponen:
   const factory = await window.mfe_loans.get('./LoanApplicationWidget')
   const Component = factory()
                |
                v
[ KOMPONEN PINJAMAN V2 BERHASIL TAMPIL DI LAYAR ]
```

## 5. Cuplikan Logika Inti

### Dynamic Script Injector & Container Initializer (TypeScript)

Implementasi fungsi loader universal tanpa keterikatan konfigurasi Webpack statis:

```typescript
// host-app/src/utils/dynamicRemoteLoader.ts

declare const __webpack_init_sharing__: (scope: string) => Promise<void>;
declare const __webpack_share_scopes__: { default: any };

interface LoadComponentOptions {
  url: string;        // URL remoteEntry.js (didapat dari manifest API)
  scope: string;      // Nama container remote (misal: 'mfe_loans')
  module: string;     // Nama path komponen yang diekspos (misal: './LoanWidget')
}

export async function loadDynamicRemoteComponent<T = any>({
  url,
  scope,
  module,
}: LoadComponentOptions): Promise<{ default: React.ComponentType<T> }> {
  // 1. Suntikkan skrip remoteEntry.js ke dalam DOM jika belum ada
  if (!(window as any)[scope]) {
    await new Promise<void>((resolve, reject) => {
      const existingScript = document.querySelector(`script[src="${url}"]`);
      if (existingScript) {
        resolve();
        return;
      }

      const script = document.createElement('script');
      script.src = url;
      script.type = 'text/javascript';
      script.async = true;

      script.onload = () => resolve();
      script.onerror = () => reject(new Error(`Gagal mengunduh remote script: ${url}`));

      document.head.appendChild(script);
    });
  }

  // 2. Inisialisasi Shared Scope Module Federation jika belum diinisialisasi
  await __webpack_init_sharing__('default');

  const container = (window as any)[scope];
  if (!container) {
    throw new Error(`Container scope '${scope}' tidak ditemukan di objek window.`);
  }

  // 3. Sambungkan shared scope Host ke container remote
  await container.init(__webpack_share_scopes__.default);

  // 4. Minta modul komponen dari container
  const factory = await container.get(module);
  const Module = factory();

  return Module;
}
```

### Penggunaan Dinamis di Komponen React Host

```tsx
// host-app/src/pages/LoansPage.tsx
import React, { Suspense, useMemo } from 'react';
import { loadDynamicRemoteComponent } from '../utils/dynamicRemoteLoader';
import { useManifest } from '../hooks/useManifest';

export function LoansPage() {
  // Ambil manifest URL dari API konfigurasi runtime
  const { manifest, isLoading } = useManifest();

  // Lazy load komponen menggunakan URL dinamis hasil manifest API
  const DynamicLoanWidget = useMemo(() => {
    if (!manifest?.mfe_loans_url) return null;

    return React.lazy(() =>
      loadDynamicRemoteComponent({
        url: manifest.mfe_loans_url, // URL dinamis dari API: https://cdn.com/v2/remoteEntry.js
        scope: 'mfe_loans',
        module: './LoanWidget',
      })
    );
  }, [manifest?.mfe_loans_url]);

  if (isLoading || !DynamicLoanWidget) {
    return <div>Menyiapkan portal pinjaman...</div>;
  }

  return (
    <div className="loans-container">
      <h1>Pengajuan Pinjaman</h1>
      <Suspense fallback={<div>Memuat widget pinjaman...</div>}>
        <DynamicLoanWidget />
      </Suspense>
    </div>
  );
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

| Kelebihan | Kekurangan / Risiko |
| :--- | :--- |
| Zero-rebuild Host: Artefak build Host yang sama dapat digunakan di seluruh lingkungan | Menambah 1 kali pemanggilan network request awal untuk mengambil data manifest API |
| Memungkinkan perilisan bertahap (Canary Rollout) dan A/B testing di level Micro-App | Memerlukan infrastruktur backend / Edge Worker untuk mengelola routing manifest |
| Tim remote memiliki otonomi 100% atas alamat hosting dan siklus rilis CDN mereka | Penanganan error handling script injection harus sangat teliti agar tidak memory leak |

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Menggunakan Dynamic Remotes
- Aplikasi enterprise dengan lingkungan bertingkat (Local -> Dev -> QA -> Staging -> UAT -> Prod).
- Tim yang rutin menerapkan strategi rilis canary atau feature flagging di level aplikasi mikro.
- Sistem dengan puluhan Micro-App di mana memelihara daftar URL statis di file config bundler sudah tidak realistis lagi.

### Kapan Cukup Menggunakan Static Remotes (Hardcode Webpack Config)
- Proyek prototipe kecil dengan hanya 2 Micro-App yang dikelola oleh tim yang sama dan tidak memerlukan strategi rilis canary bertingkat.

## 8. Metrik Kunci & Observabilitas

1. **Manifest Service Response Time (ms)**: Waktu respons endpoint `GET /api/v1/mfe-manifest`. Wajib berada di bawah 50-100 milidetik (gunakan edge caching via Cloudflare Workers).
2. **Dynamic Script Injection Failure Rate**: Persentase kegagalan event `script.onerror` saat menyuntikkan URL remote dinamis ke DOM.
3. **Canary Version Distribution Ratio**: Pantau grafik proporsi pengguna yang menerima versi v1 vs versi v2 untuk memastikan pembagian traffic canary bekerja presisi.

## 9. Studi Kasus Industri Nyata

Sebuah platform dompet digital memiliki 8 Micro-App. Di masa lalu, tim DevOps harus menjalankan 8 pipeline build terpisah setiap kali mempromosikan kode dari staging ke production, memakan waktu 1,5 jam.

Selain itu, ketika Tim Merchant merilis pembaruan yang memiliki bug fatal di production, seluruh tim panik karena proses rollback membutuhkan deploy ulang aplikasi Host Shell yang memakan waktu 30 menit.

Solusi transformasi:
1. Mengadopsi arsitektur **Dynamic Remotes Management** dengan Manifest Service yang ditenagai Cloudflare KV di edge.
2. Saat terjadi bug darurat pada Tim Merchant, tim DevOps tidak perlu melakukan rollback kode. Mereka cukup mengubah nilai URL di Cloudflare KV dalam 1 detik: mengembalikan URL remote ke versi kemarin.
3. Seluruh pengguna di seluruh dunia langsung otomatis memuat versi stabil kemarin saat me-refresh halaman. Waktu pemulihan insiden (*MTTR*) terpangkas dari 30 menit menjadi hanya 5 detik.

## 10. Fakta Menarik & Gotchas Production

- **Cache-Control pada Manifest vs RemoteEntry**:
  - Endpoint **Manifest API**: Harus memiliki cache berdurasi sangat pendek (`Cache-Control: max-age=60, s-maxage=60`) atau no-cache agar perubahan canary release langsung berefek instan.
  - File **`remoteEntry.js`**: Juga wajib disetel dengan `no-cache` atau `max-age=0` di CDN agar browser selalu meminta manifest terbaru.
  - File **Chunks komponen (`*.chunk.js`)**: Wajib disetel dengan immutable cache panjang (`max-age=31536000, immutable`) karena nama filenya memuat hash konten unik.
- **Dynamic Scope Collision**: Pastikan nama scope unik di objek `window` (misal `window.mfe_loans_v2`). Jika dua remote app yang berbeda menggunakan nama scope yang sama, remote yang dimuat terakhir akan menimpa remote sebelumnya di memori browser.
- **Module Federation 2.0 Native Support**: Di versi terbaru Module Federation (MF 2.0 / Rsbuild), dynamic remote loading kini didukung secara native melalui fungsi deklaratif `loadRemote('mfe/Component')` tanpa perlu menulis script injector manual.

## 11. Rangkuman Singkat

Dynamic Remotes Management melepaskan keterikatan konfigurasi URL statis saat build time, menggantinya dengan pemuatan remote secara dinamis saat runtime melalui Manifest API. Pendekatan ini memungkinkan prinsip *Build Once Run Anywhere*, mendukung rilis canary bertahap, dan menyediakan kemampuan rollback instan tanpa perlu mendeploy ulang aplikasi Shell.
