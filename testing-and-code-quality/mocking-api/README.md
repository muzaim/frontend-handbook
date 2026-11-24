# Mocking API

## 1. Skenario Sistem Nyata

Tim frontend dan backend sedang membangun fitur dashboard analytics inventaris barang secara paralel (*sprint concurrent*). Backend API belum selesai dibangun karena skema database masih dalam tahap migrasi.

Developer frontend memutuskan untuk membuat fungsi tiruan sederhana dengan memodifikasi kode produksi:
```typescript
// src/api/inventory.ts
export async function getInventory() {
  // TODO: Hapus kode hardcode ini sebelum rilis ke production!
  return [{ id: 1, name: 'Barang Dummy', stock: 100 }];
  // return fetch('/api/v1/inventory').then(r => r.json());
}
```

Dua minggu kemudian, fitur diluncurkan ke production. Developer lupa menghapus baris mock di kode produksi tersebut. Akibatnya, dashboard seluruh pelanggan perusahaan di production menampilkan data palsu "Barang Dummy", dan data stok gudang asli tidak pernah dimuat dari server.

Pendekatan mock manual di dalam kode produksi adalah bom waktu. Solusi arsitektural yang bersih dan aman adalah **Mocking API di Level Jaringan (Network-Level Mocking)** menggunakan Mock Service Worker (MSW).

## 2. Apa Itu dan Buat Apa?

Mocking API adalah teknik mensimulasikan respons dari backend REST API atau GraphQL yang realistis selama masa pengembangan (development) dan pengujian (testing) frontend, tanpa perlu bergantung pada server backend nyata yang sedang aktif.

Perbedaan pendekatan mocking modern:
- **Module Mocking Lawas (`vi.mock('axios')` / `global.fetch = vi.fn()`)**: Mengganti modul client HTTP di memori JavaScript. Kode pengujian tidak lagi menguji implementasi fetch asli, dan konfigurasi header atau interceptor sering terlewatkan.
- **Network-Level Mocking Modern (MSW - Mock Service Worker)**: Bekerja di tingkat protokol jaringan HTTP. Kode produksi menggunakan `fetch()` atau `axios` resmi tanpa diubah sama sekali. Permintaan HTTP ditangkap oleh Service Worker di browser atau interceptor network di Node.js, lalu dikembalikan respons tiruan yang menyerupai server asli.

## 3. Kenapa Harus Pakai Ini?

- **Pengembangan Paralel Independen**: Tim frontend bisa mulai membangun seluruh UI, state management, dan error handling sejak hari pertama sprint menggunakan kontrak API (OpenAPI/Swagger), meskipun backend belum selesai dibuat.
- **Simulasi Skenario Ekstrem dengan Mudah**: Menguji bagaimana UI bereaksi saat server mengembalikan HTTP 500 Internal Server Error, HTTP 401 Unauthorized, rate limit HTTP 429, atau koneksi timeout lambat (delay 3 detik). Sangat sulit menguji skenario ini secara konsisten pada backend nyata.
- **Konsistensi Testing**: Satu definisi mock handler yang sama dapat digunakan bersama untuk mode development di browser (`npm run dev`) maupun untuk integration test di Vitest/Jest (`npm test`).

## 4. Cara Kerjanya

Berikut adalah diagram alur bagaimana Mock Service Worker (MSW) mencegat request jaringan di browser dan di lingkungan test runner (Node.js):

```
--- 1. MODE PENGEMBANGAN DI BROWSER (npm run dev) ---
[ Kode Aplikasi Frontend: fetch('/api/products') ]
                      |
                      v
         [ Browser Network Layer ]
                      |
             (Dilewatkan ke Service Worker)
                      v
         [ MSW Service Worker (mockServiceWorker.js) ]
                      |
           +----------+----------+
           | Cocok dengan rute?  |
           +----------+----------+
              /               \
        (YA) /                 \ (TIDAK)
            v                   v
     Kembalikan Respon     Teruskan ke Server
     Mock JSON (200 OK)    Backend Nyata di Internet


--- 2. MODE PENGUJIAN OTOMATIS (npm test / Node.js) ---
[ Vitest: render(<ProductCatalog />) ]
                      |
                      v
[ Node.js HTTP Interceptor (msw/node) ]
                      |
                      v
[ Handlers MSW: Mengembalikan data mock ke komponen ]
```

## 5. Cuplikan Logika Inti

### Definisi Handlers Tiruan (MSW v2)

```typescript
// src/mocks/handlers.ts
import { http, HttpResponse, delay } from 'msw';

export interface Product {
  id: string;
  name: string;
  price: number;
}

export const mockProducts: Product[] = [
  { id: 'p1', name: 'MacBook Pro M3', price: 28000000 },
  { id: 'p2', name: 'Magic Mouse', price: 1200000 },
];

export const handlers = [
  // Mock endpoint GET katalog produk
  http.get('/api/products', async () => {
    // Tambahkan simulasi latency jaringan 200ms
    await delay(200);
    return HttpResponse.json(mockProducts, { status: 200 });
  }),

  // Mock endpoint POST pembuatan pesanan
  http.post('/api/orders', async ({ request }) => {
    const body = (await request.json()) as { productId: string; quantity: number };

    // Simulasi validasi bisnis
    if (!body.productId) {
      return HttpResponse.json(
        { message: 'Product ID wajib disertakan' },
        { status: 400 }
      );
    }

    return HttpResponse.json(
      { orderId: 'ord-999', status: 'PENDING' },
      { status: 201 }
    );
  }),
];
```

### Integrasi pada Pengujian Integrasi (Vitest)

Menguji skenario sukses dan skenario server error tanpa mengubah satu baris pun kode produksi:

```typescript
// src/features/products/ProductCatalog.test.tsx
import { render, screen } from '@testing-library/react';
import { describe, it, expect, beforeAll, afterEach, afterAll } from 'vitest';
import { setupServer } from 'msw/node';
import { http, HttpResponse } from 'msw';
import { handlers } from '../../mocks/handlers';
import { ProductCatalog } from './ProductCatalog';

// Siapkan mock server untuk lingkungan Node.js
const server = setupServer(...handlers);

beforeAll(() => server.listen({ onUnhandledRequest: 'error' }));
afterEach(() => server.resetHandlers());
afterAll(() => server.close());

describe('<ProductCatalog /> dengan MSW', () => {
  it('harus merender daftar produk dari API mock', async () => {
    render(<ProductCatalog />);

    // Elemen awal loading
    expect(screen.getByText(/Memuat produk/i)).toBeInTheDocument();

    // Data dari mockProducts terender di layar
    expect(await screen.findByText('MacBook Pro M3')).toBeInTheDocument();
    expect(screen.getByText('Magic Mouse')).toBeInTheDocument();
  });

  it('harus menampilkan pesan error jika backend mengalami HTTP 500', async () => {
    // Override handler khusus untuk skenario kegagalan server ini
    server.use(
      http.get('/api/products', () => {
        return new HttpResponse(null, { status: 500 });
      })
    );

    render(<ProductCatalog />);

    // Verifikasi UI menangani error state dengan baik
    expect(
      await screen.findByText(/Gagal memuat data dari server/i)
    ).toBeInTheDocument();
  });
});
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

| Kelebihan | Kekurangan / Risiko |
| :--- | :--- |
| Kode produksi 100% bersih tanpa ada kode mock yang bocor ke release build | Jika backend mengubah kontrak API tanpa memperbarui mock, tes bisa memberi sinyal hijau palsu |
| Bekerja di tingkat jaringan sesungguhnya (menguji `fetch`, header HTTP, status code) | Membutuhkan setup awal file worker publik (`mockServiceWorker.js`) untuk browser |
| Dapat membagikan kumpulan mock handlers yang sama untuk dev mode dan test suite | Memerlukan waktu pemeliharaan agar data mock tetap sinkron dengan evolusi skema backend |

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Menggunakan MSW Network Mocking
- Seluruh pengujian komponen dan integrasi frontend yang memanggil REST API atau GraphQL.
- Masa pengembangan lokal di mana backend belum siap atau sering mengalami down / maintenance.
- Membuat demo prototipe atau lingkungan Storybook interaktif yang memerlukan data dinamis.

### Kapan Jangan Menggunakan Mocking
- Pengujian E2E Smoke Test di lingkungan staging/pre-production yang bertujuan memverifikasi koneksi database riil.
- Menguji integrasi pihak ketiga yang menyediakan API sandbox resmi (seperti Stripe Test Mode).

## 8. Metrik Kunci & Observabilitas

1. **Unhandled Request Warning Count**: Periksa apakah ada request dari frontend yang bocor ke internet saat pengujian berjalan (`onUnhandledRequest: 'warn'`). Setiap request yang unhandled menandakan rute yang belum dimock.
2. **Mock Drift Ratio**: Jadwalkan audit berkala menggunakan alat seperti OpenAPI TypeScript Generator untuk mencocokkan skema handler MSW dengan dokumen backend Swagger terbaru.
3. **Test Speed Stability**: Pastikan mock handler di lingkungan test tidak menggunakan `delay()` buatan yang memperlambat laju pengujian otomatis.

## 9. Studi Kasus Industri Nyata

Sebuah startup logistik fintech memiliki frontend dashboard pengiriman paket. Setiap kali tim QA menjalankan tes integrasi, mereka memanggil server staging nyata.

Masalah muncul:
- Database staging sering terisi data sampah yang bertabrakan (*race condition* saat dua tes berjalan bersamaan).
- Ketika server staging mati untuk deploy, seluruh pipeline CI pengembang terblokir, menghentikan deployment selama berjam-jam.

Solusi engineering:
1. Mengadopsi MSW untuk seluruh pengujian integrasi frontend dan dev mode.
2. Seluruh dependensi ke server staging nyata pada unit/integration test dihapus 100%.
3. Waktu eksekusi pipeline CI terpangkas dari 18 menit (karena latensi jaringan server) menjadi hanya 45 detik, dengan tingkat keberhasilan tes mencapai 100% tanpa gangguan downtime backend.

## 10. Fakta Menarik & Gotchas Production

- **MSW v2 Perubahan Sintaks**: Di MSW versi 2, namespace `rest.get()` telah digantikan menjadi `http.get()`, dan `res(ctx.json())` telah digantikan dengan standar Web API native `HttpResponse.json()`.
- **Keamanan Bundler**: Jangan pernah mengimpor file inisialisasi browser worker (`enableMocking()`) di production build. Bungkus inisialisasi di dalam blok kondisi environment: `if (process.env.NODE_ENV === 'development')`.
- **Dukungan GraphQL**: MSW mendukung query dan mutasi GraphQL secara native melalui handler `graphql.query()` dan `graphql.mutation()`, memudahkan penanganan payload bersarang (*nested data*).

## 11. Rangkuman Singkat

Mocking API modern dilakukan di tingkat protokol jaringan menggunakan Mock Service Worker (MSW). Pendekatan ini memungkinkan kode aplikasi menggunakan pemanggilan HTTP native secara utuh, memberikan kebebasan bereksperimen dengan berbagai skenario respons server tanpa mengotori kode produksi.
