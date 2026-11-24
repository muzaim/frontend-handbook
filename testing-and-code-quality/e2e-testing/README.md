# E2E Testing (End-to-End Testing)

## 1. Skenario Sistem Nyata

Sebuah platform SaaS bursa kerja merilis pembaruan besar pada sistem autentikasi dan billing langganan. Seluruh unit test dan integration test di CI lulus 100% dengan status hijau.

Namun, beberapa menit setelah kode dirilis ke domain production (`https://app.jobboard.com`), para pengguna membanjiri kanal bantuan:
"Saya tidak bisa login sama sekali! Setelah memasukkan password, layar hanya blank putih."

Setelah diinvestigasi darurat, penyebabnya adalah konfigurasi bundler Vite di production memuat polyfill HTTP cookie yang tidak kompatibel dengan browser Safari iOS di perangkat mobile nyata. Selain itu, ada aturan firewall Cloudflare yang memblokir header CORS pada subdomain otentikasi tertentu.

Lingkungan unit test berbasis `jsdom` di komputer lokal tidak dapat mendeteksi kegagalan jaringan riil, engine layout browser asli, maupun konflik konfigurasi server edge ini. Masalah fatal ini hanya bisa ditangkap sebelum rilis oleh **E2E Testing (End-to-End Testing)**.

## 2. Apa Itu dan Buat Apa?

E2E Testing (End-to-End Testing) adalah metodologi pengujian perangkat lunak komprehensif yang menguji seluruh alur kerja aplikasi secara lengkap dari awal hingga akhir (*end-to-end*), menggunakan mesin browser nyata (Chromium, Firefox, WebKit/Safari) yang digerakkan secara otomatis melalui otomasi browser.

Karakteristik utama E2E testing:
- **Lingkungan Nyata**: Berjalan pada URL aplikasi yang benar-benar aktif (local dev server, staging environment, atau preview deploy).
- **Interaksi Browser Nyata**: Mensimulasikan klik mouse, pengetikan keyboard, navigasi halaman URL, pengunggahan berkas, dan penyimpanan cookie browser sesungguhnya.
- **Tingkat Kepercayaan Tertinggi**: Menguji tumpukan teknologi secara menyeluruh: Frontend UI, Backend API, Database, Network Latency, dan Browser Security Policies.

Tujuannya adalah memastikan bahwa seluruh sistem terintegrasi berfungsi dengan benar dari perspektif pengguna nyata sebelum kode diluncurkan ke production.

## 3. Kenapa Harus Pakai Ini?

- **Melindungi Alur Bisnis Paling Kritis (Golden Paths)**: Alur seperti *User Registration*, *Login & 2FA*, *Checkout Belanja*, dan *Pembayaran Langganan* adalah urat nadi bisnis perusahaan. Kegagalan pada alur ini berarti kehilangan pendapatan seketika. E2E test menjamin alur-alur ini tidak pernah rusak.
- **Deteksi Masalah Lintas Browser (Cross-Browser Quirks)**: Memverifikasi bahwa tampilan visual, event listener, dan API browser (seperti Date picker, Clipboard API, Service Worker) bekerja konsisten di Chrome, Firefox, dan Safari.
- **Verifikasi Infrastruktur Lengkap**: Menguji interaksi nyata dengan reverse proxy, header CORS, cookie SameSite/Secure, dan SSL certificate yang tidak ada di lingkungan testing berbasis memori.

## 4. Cara Kerjanya

Berikut adalah diagram alur bagaimana runner pengujian E2E modern (seperti Playwright) mengendalikan browser nyata melalui protokol DevTools:

```
[ Playwright Test Runner (Node.js) ]
                |
                |-- 1. Luncurkan Browser Nyata via Chrome DevTools Protocol (CDP)
                v
[ Headless Chromium / WebKit Instance ]
                |
                |-- 2. page.goto('https://staging.app.com/login')
                v
[ Web Server Staging + Backend Database ]
                |
                |<-- 3. Muat HTML, CSS, JS, dan Aset Asli
                v
[ Browser Merender Halaman Penuh ]
                |
                |-- 4. page.fill('input[type="email"]', 'user@test.com')
                |-- 5. page.fill('input[type="password"]', 'Rahasia123')
                |-- 6. page.click('button:has-text("Masuk")')
                v
[ Browser Mengirim Request Nyata ke Backend Auth ]
                |<-- Backend mengembalikan Cookie Sesi Nyata
                v
[ Browser Redirect ke /dashboard ]
                |
                v
[ ASSERTION: expect(page).toHaveURL(/.*dashboard/) ]
  - Ambil screenshot visual jika gagal
  - Rekam video tracing untuk analisis debugging
```

## 5. Cuplikan Logika Inti

### Pengujian Alur Checkout Kritis (Menggunakan Playwright di TypeScript)

```typescript
// e2e/checkout.spec.ts
import { test, expect } from '@playwright/test';

test.describe('E2E Alur Pembelian Pengguna', () => {
  test.beforeEach(async ({ page }) => {
    // Mulai dari halaman katalog produk
    await page.goto('/products');
  });

  test('pengguna dapat memilih barang, mengisi alamat, dan menyelesaikan pesanan', async ({ page }) => {
    // 1. Pilih produk pertama dan klik tombol tambah ke keranjang
    const productCard = page.locator('.product-card').first();
    await expect(productCard).toBeVisible();
    
    await productCard.locator('button:has-text("Beli Sekarang")').click();

    // 2. Buka drawer keranjang belanja
    const cartBadge = page.locator('[data-testid="cart-badge"]');
    await expect(cartBadge).toHaveText('1');
    await cartBadge.click();

    // 3. Lanjutkan ke halaman checkout
    await page.locator('button:has-text("Lanjut ke Pembayaran")').click();
    await expect(page).toHaveURL(/\/checkout/);

    // 4. Isi formulir pengiriman
    await page.fill('#shipping-name', 'Ahmad Dani');
    await page.fill('#shipping-address', 'Jl. Sudirman No. 45, Jakarta Selatan');
    await page.selectOption('#shipping-courier', 'jne-express');

    // 5. Pilih metode pembayaran Transfer Bank
    await page.check('input[value="bank_transfer"]');

    // 6. Submit pesanan
    const payButton = page.locator('button:has-text("Konfirmasi & Bayar")');
    await expect(payButton).toBeEnabled();
    await payButton.click();

    // 7. Verifikasi halaman sukses dan nomor invoice
    await expect(page).toHaveURL(/\/order\/success/);
    const successBanner = page.locator('h1');
    await expect(successBanner).toHaveText(/Pesanan Anda Telah Diterima/i);

    const invoiceText = page.locator('[data-testid="invoice-number"]');
    await expect(invoiceText).toBeVisible();
  });
});
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

| Kelebihan | Kekurangan / Risiko |
| :--- | :--- |
| Memberikan tingkat keyakinan (*confidence*) paling tinggi bahwa aplikasi bekerja normal di tangan user | Waktu eksekusi jauh lebih lambat (menit hingga puluhan menit untuk satu suite penuh) |
| Menangkap masalah lingkungan nyata: CORS, cookie, SSL, DNS, dan browser quirks | Membutuhkan biaya infrastruktur CI yang lebih besar (CPU & RAM untuk menjalankan headless browser) |
| Menguji seluruh arsitektur secara terintegrasi (frontend + backend + database) | Rentan terhadap kegagalan palsu (*flaky tests*) jika ada latency jaringan tidak stabil |

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Menggunakan E2E Testing
- Alur perjalanan pengguna paling bernilai (*high-value customer journeys*): registrasi, login, reset kata sandi, transaksi pembayaran, checkout pesanan.
- Alur integrasi pihak ketiga: login via Google/GitHub OAuth, webhook pembayaran simulator.
- Uji kompatibilitas cross-browser berkala (misal: verifikasi Safari WebKit di Mac/iOS).

### Kapan Jangan Menggunakan E2E Testing
- Menguji seluruh kemungkinan kombinasi logika matematika atau edge case string (gunakan **Unit Testing**).
- Menguji setiap variasi state visual komponen sederhana seperti warna tombol disabled atau loading spinner (gunakan **Component Testing**).

## 8. Metrik Kunci & Observabilitas

1. **Flaky Test Rate**: Persentase tes yang gagal pada run pertama namun lulus saat di-*retry*. Flakiness harus ditekan di bawah 2% dengan menggunakan fitur *auto-waiting* Playwright dan menghindari pemanggilan `sleep(5000)`.
2. **E2E Pipeline Run Duration**: Waktu yang dihabiskan di CI untuk menjalankan suite E2E. Jika melebihi 15 menit, lakukan paralelisasi menggunakan beberapa worker mesin (*sharding*).
3. **Artifact Retention (Trace & Video)**: Pastikan CI otomatis menyimpan rekaman trace (*Playwright Trace Viewer*) dan video rekaman hanya pada tes yang berstatus gagal, untuk memudahkan investigasi tanpa menghabiskan kuota storage CI.

## 9. Studi Kasus Industri Nyata

Sebuah platform marketplace properti besar memiliki siklus rilis harian. Di masa lalu, tim QA manual membutuhkan waktu 4 jam setiap sore untuk mengklik alur pencarian rumah, kalkulator simulasi KPR, dan formulir pengajuan kontak agen di berbagai browser.

Seringkali, karena kelelahan manusia, formulir kontak agen di browser Safari tidak sengaja terlewat untuk dites. Suatu hari, bug CSS `pointer-events: none` menutupi tombol kontak agen di Safari mobile, menyebabkan penurunan 40% leads agen selama 3 hari berturut-turut.

Transformasi arsitektur testing:
1. Mengotomasi 25 skenario *critical user journeys* menggunakan Playwright dengan eksekusi paralel di 4 container CI.
2. Suite E2E berjalan otomatis setiap kali ada Pull Request ke branch `main`, selesai dalam waktu 6 menit (menggantikan 4 jam QA manual).
3. Bug regresi antarmuka di Safari terdeteksi seketika di pipeline PR sebelum kode di-merge ke production.

## 10. Fakta Menarik & Gotchas Production

- **Jangan Gunakan Hardcoded Sleep**: Menggunakan kode seperti `await page.waitForTimeout(5000)` adalah penyebab utama flaky test di production. Browser modern seperti Playwright secara otomatis menunggu elemen siap diklik (*web-first assertions* dan *actionability checks*).
- **Strategi Data Seeding**: E2E test yang baik tidak bergantung pada data statis yang bisa diubah oleh tes lain. Gunakan API internal untuk membuat (*seed*) user dan data belanja baru yang unik untuk setiap sesi pengujian, lalu bersihkan setelah selesai.
- **Autentikasi Ulang Berulang**: Jangan login manual melalui form login pada setiap berkas tes karena akan memperlambat suite. Gunakan fitur *storageState* Playwright untuk menyimpan session cookie sekali saja, lalu pakai kembali di seluruh rangkaian tes.

## 11. Rangkuman Singkat

E2E Testing menguji aplikasi web secara menyeluruh di atas browser nyata untuk memverifikasi alur bisnis paling bernilai dari kacamata pengguna. Menggunakan framework modern seperti Playwright dengan auto-waiting dan data seeding yang bersih menjamin stabilitas sistem tanpa menimbulkan masalah tes yang rapuh (*flaky*).
