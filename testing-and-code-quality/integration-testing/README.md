# Integration Testing

## 1. Skenario Sistem Nyata

Sebuah aplikasi booking tiket pesawat memiliki modul pencarian penerbangan yang terdiri dari beberapa bagian independen:
1. Input formulir pencarian (kota asal, tujuan, tanggal).
2. Custom hook pencarian yang mengelola state filter dan caching.
3. Komponen daftar hasil penerbangan yang merender kartu tiket.

Setiap unit fungsi telah lulus 100% unit test secara terpisah. Namun saat user mengetik kota tujuan dan menekan tombol "Cari Tiket", daftar tiket tidak pernah muncul di layar.

Penyebabnya adalah ketidakcocokan format data: formulir mengirim tanggal dalam format ISO `YYYY-MM-DD`, sedangkan custom hook mengharapkan Unix timestamp integer milidetik. Unit test masing-masing komponen lolos karena mock data mereka dibuat sendiri-sendiri, namun saat dihubungkan bersama, sistem gagal total.

Kesenjangan komunikasi antar modul ini hanya dapat dideteksi melalui **Integration Testing**.

## 2. Apa Itu dan Buat Apa?

Integration Testing adalah level pengujian perangkat lunak yang berfokus pada pengujian bagaimana beberapa modul atau unit kode yang berbeda bekerja sama sebagai satu kesatuan subsistem.

Di dunia frontend modern, integration testing biasanya menguji kolaborasi antara:
- Komponen antarmuka pengguna (UI) dengan State Management (Zustand/Redux).
- Komponen formulir dengan validasi schema (Zod) dan layer HTTP client.
- Custom hook yang berinteraksi dengan browser storage atau cache query (TanStack Query).

Tujuannya adalah memastikan bahwa antarmuka kontrak (*interface contract*) dan aliran data antar modul yang terhubung berfungsi secara harmonis tanpa cacat integrasi.

## 3. Kenapa Harus Pakai Ini?

- **Mendeteksi Bug di Perbatasan Modul (Contract Mismatches)**: Sebagian besar bug production terjadi bukan pada logika internal satu fungsi terisolasi, melainkan saat data berpindah tangan dari modul A ke modul B.
- **Rasio Kepercayaan terhadap Biaya yang Optimal**: Berada tepat di tengah piramida pengujian (*Testing Trophy*). Integration test memberikan tingkat keyakinan (*confidence*) yang jauh lebih tinggi daripada unit test murni, namun dieksekusi jauh lebih cepat dan lebih murah dipelihara dibandingkan E2E test yang memuat browser penuh.
- **Menguji Alur Nyata Pengguna Tanpa Overhead Browser Nyata**: Menguji apakah interaksi user (klik tombol) berhasil memperbarui store global dan memicu pemanggilan request network yang diharapkan.

## 4. Cara Kerjanya

Berikut adalah diagram alur bagaimana integration test memverifikasi kolaborasi antar komponen, store, dan network mock:

```
+-----------------------------------------------------------------------+
|  INTEGRATION TEST ENVIRONMENT (jsdom / happy-dom via Vitest)          |
|                                                                       |
|  [ Komponen UI: SearchForm ]                                          |
|         |                                                             |
|         |-- 1. Simulasi User Input: ketik "Jakarta" & klik "Cari"     |
|         v                                                             |
|  [ State Layer: useFlightStore / Zustand ]                            |
|         |                                                             |
|         |-- 2. Store memperbarui state { destination: "CGK" }         |
|         |-- 3. Memicu async fetch action                              |
|         v                                                             |
|  [ Network Layer: Mock Service Worker (MSW) ]                         |
|         |                                                             |
|         |-- 4. MSW menangkap request /api/flights?dest=CGK            |
|         |-- 5. Mengembalikan respon tiruan: [{ flightId: "GA-123" }]  |
|         v                                                             |
|  [ Komponen UI: FlightList ]                                          |
|         |                                                             |
|         v                                                             |
|  [ ASSERTION: Layar menampilkan kartu "Garuda Indonesia GA-123" ]     |
+-----------------------------------------------------------------------+
```

## 5. Cuplikan Logika Inti

### Pengujian Integrasi Form, Store, dan API Mock (React Testing Library + Vitest)

```tsx
// src/features/checkout/CheckoutFlow.test.tsx
import { render, screen, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { describe, it, expect, beforeEach } from 'vitest';
import { CheckoutPage } from './CheckoutPage';
import { useCartStore } from '../../store/cartStore';

describe('Integration Test: Checkout Flow', () => {
  beforeEach(() => {
    // Reset state global cart sebelum setiap tes dimulai
    useCartStore.setState({
      items: [{ id: 'prod-1', name: 'Keyboard Mekanikal', price: 500000, quantity: 1 }],
      shippingCost: 0,
    });
  });

  it('harus menghitung ongkir dan mengaktifkan tombol bayar setelah memilih kurir', async () => {
    const user = userEvent.setup();
    render(<CheckoutPage />);

    // 1. Verifikasi subtotal awal dari store terender di UI
    expect(screen.getByText('Keyboard Mekanikal')).toBeInTheDocument();
    expect(screen.getByTestId('subtotal')).toHaveTextContent('Rp 500.000');

    // Tombol bayar harus disabled sebelum kurir dipilih
    const submitBtn = screen.getByRole('button', { name: /Bayar Sekarang/i });
    expect(submitBtn).toBeDisabled();

    // 2. Interaksi User: Pilih kurir ekspres dari dropdown select
    const courierSelect = screen.getByRole('combobox', { name: /Pilih Kurir/i });
    await user.selectOptions(courierSelect, 'express');

    // 3. Verifikasi integrasi: State store terupdate dan UI merender total baru
    await waitFor(() => {
      // Ongkir express 30.000 ditambahkan ke subtotal 500.000 -> 530.000
      expect(screen.getByTestId('grand-total')).toHaveTextContent('Rp 530.000');
    });

    // 4. Tombol bayar sekarang harus sudah aktif
    expect(submitBtn).toBeEnabled();

    // 5. User submit form bayar
    await user.click(submitBtn);

    // 6. Verifikasi pesan sukses muncul di layar
    expect(await screen.findByText(/Pesanan Berhasil Dibuat/i)).toBeInTheDocument();
  });
});
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

| Kelebihan | Kekurangan / Risiko |
| :--- | :--- |
| Menangkap bug integrasi kontrak data antar modul yang lolos dari unit test | Waktu eksekusi sedikit lebih lambat dibanding unit test murni karena merender pohon DOM virtual |
| Menguji perilaku aplikasi dari sudut pandang nyata pengguna (user-centric) | Setup awal lingkungan uji lebih kompleks (memerlukan store provider, router, dan API mocking) |
| Menghindari pengujian detail implementasi internal komponen | Jika struktur hierarki komponen berubah drastis, tes integrasi mungkin perlu penyesuaian query |

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Pakai
- Alur transaksi multi-langkah: form pendaftaran dengan validasi schema, checkout belanja, filter pencarian dengan pagination.
- Interaksi antara komponen presentasi dengan Global State Store (Redux, Zustand) atau Server Cache (TanStack Query).
- Fitur kompleks yang melibatkan beberapa custom hook dan context provider yang saling bergantung.

### Kapan Menggunakan Level Lain
- Jika hanya ingin menguji algoritma matematika murni atau regex formatter tanpa UI (gunakan **Unit Testing**).
- Jika ingin menguji cross-browser layout visual, animasi CSS kompleks, atau web hosting production di browser Chromium/Safari nyata (gunakan **E2E Testing**).

## 8. Metrik Kunci & Observabilitas

1. **Integration Test Suite Duration**: Pantau durasi suite integrasi di pipeline CI. Pertahankan di kisaran 1-3 menit dengan memanfaatkan konkurensi thread Vitest.
2. **Mock Leakage Detection**: Pastikan state global dan mock server selalu direset bersih di blok `beforeEach`/`afterEach` agar tidak ada kebocoran state antar berkas tes.
3. **User Action Coverage**: Pantau apakah interaksi pengguna yang paling krusial (*happy path* dan error state submit form) telah tercakup dalam tes integrasi.

## 9. Studi Kasus Industri Nyata

Sebuah platform perbankan digital memiliki form transfer dana. Formulir tersebut memisahkan komponen input rekening tujuan dan komponen konfirmasi nominal transfer.

Dalam implementasi baru, tim developer memperbarui store Zustand agar menggunakan middleware `persist` ke `sessionStorage`. Namun, serialisasi JSON pada middleware tersebut mengubah nilai tipe `BigInt` (yang digunakan untuk ID rekening perbankan) menjadi string biasa.

Unit test komponen input rekening tetap lulus 100% karena hanya mengecek event `onChange`. Namun saat integrasi dijalankan, komponen konfirmasi transfer menerima string dan gagal mencocokkannya dengan schema validasi transfer, menyebabkan tombol submit macet tanpa pesan error.

Tim QA mengidentifikasi masalah ini melalui pengujian integrasi otomatis yang mensimulasikan alur penuh: mengetik input -> store -> verifikasi konfirmasi. Bug serialisasi berhasil diperbaiki sebelum kode sempat masuk ke pipeline rilis produksi.

## 10. Fakta Menarik & Gotchas Production

- **Hindari Menguji State Internal**: Jangan menguji nilai variabel `state` komponen secara langsung (`expect(wrapper.state('count')).toBe(1)`). Uji efeknya pada apa yang dilihat pengguna di layar (`expect(screen.getByText('1')).toBeInTheDocument()`).
- **Gunakan `findBy*` untuk Asinkron**: Jika elemen muncul setelah proses fetch API tiruan selesai, gunakan query `screen.findByText()` yang otomatis menunggu secara asinkron, bukan `screen.getByText()` yang langsung gagal jika elemen belum terender seketika.
- **Pembersihan Bersama**: Jika Anda menggunakan library mock network seperti MSW, selalu panggil `server.resetHandlers()` di `afterEach` agar konfigurasi handler di satu tes tidak merusak skenario tes berikutnya.

## 11. Rangkuman Singkat

Integration Testing memverifikasi bahwa modul-modul yang berbeda (komponen antarmuka, store state manajemen, dan layer data network) bekerja sama secara mulus sesuai kontrak. Ini adalah pilar utama dalam piramida pengujian frontend yang memberikan tingkat keyakinan tinggi dengan biaya pemeliharaan yang terukur.
