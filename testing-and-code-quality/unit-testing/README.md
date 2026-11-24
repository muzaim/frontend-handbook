# Unit Testing

## 1. Skenario Sistem Nyata

Sebuah platform e-commerce memiliki fungsi utilitas untuk menghitung diskon bertingkat, ongkos kirim, dan pajak PPN pada halaman keranjang belanja:
`calculateOrderTotal(items, discountCode, userTier)`

Seorang developer junior diminta menambahkan aturan baru: "Pengguna dengan tier VIP mendapat potongan tambahan 5% jika total belanja di atas Rp 500.000". Developer mengubah fungsi tersebut dan langsung mengunggahnya ke server staging.

Ternyata, perubahan logika kondisi `if (subtotal > 500000)` secara tidak sengaja merusak perhitungan untuk pengguna non-VIP yang menggunakan voucher kupon musiman. Akibatnya, ribuan transaksi mendapatkan diskon ganda yang merugikan perusahaan ratusan juta rupiah sebelum bug disadari.

Jika fungsi matematika tersebut memiliki rangkaian **Unit Testing** otomatis, bug regresi ini akan langsung terdeteksi dalam hitungan milidetik di komputer lokal developer sebelum kode sempat di-commit.

## 2. Apa Itu dan Buat Apa?

Unit Testing adalah metode pengujian perangkat lunak di mana unit atau komponen terkecil dari kode sumber (seperti fungsi murni, utilitas kalkulasi, parser data, atau custom hook independen) diuji secara terisolasi dari bagian aplikasi lainnya.

Karakteristik utama unit test:
- **Terisolasi**: Tidak bergantung pada database, koneksi jaringan internet, atau rendering DOM browser yang rumit.
- **Deterministik**: Dengan input $X$ yang sama, tes harus selalu menghasilkan output $Y$ yang sama persis kapan pun dan di mana pun dijalankan.
- **Cepat**: Ribuan unit test dapat selesai dieksekusi hanya dalam hitungan detik.

Tujuan utamanya adalah memverifikasi bahwa logika dasar kode berjalan sesuai spesifikasi dan mencegah regresi (*breaking changes*) saat kode di-refactor di kemudian hari.

## 3. Kenapa Harus Pakai Ini?

- **Jaring Pengaman Refactoring**: Anda bisa merombak arsitektur fungsi internal tanpa rasa takut, karena unit test akan langsung berteriak jika ada hasil kalkulasi yang menyimpang.
- **Dokumentasi Hidup (Living Documentation)**: Membaca file tes (`*.test.ts`) seringkali merupakan cara tercepat bagi developer baru untuk memahami input apa yang diterima suatu fungsi dan output apa yang diharapkan.
- **Memaksa Desain Kode yang Bersih**: Kode yang sulit di-unit-test biasanya memiliki *code smell* (terlalu banyak side effects, tight coupling, atau fungsi terlalu panjang). Menulis unit test memaksa developer membuat fungsi murni (*pure functions*).

## 4. Cara Kerjanya

Berikut adalah diagram alur bagaimana runner pengujian (seperti Vitest atau Jest) mengeksekusi siklus **AAA (Arrange, Act, Assert)** pada unit test:

```
[ Runner Membaca File test.ts ]
             |
             v
+-------------------------------------------------------+
|  SIKLUS UJI AAA (Arrange, Act, Assert)                |
|                                                       |
|  1. ARRANGE (Persiapan Data Uji)                      |
|     const items = [{ price: 100000, qty: 2 }];        |
|     const discountCode = "HEMAT10";                   |
|                                                       |
|  2. ACT (Eksekusi Unit yang Diuji)                    |
|     const result = calculateOrderTotal(items, code);  |
|                                                       |
|  3. ASSERT (Validasi Hasil terhadap Ekspektasi)       |
|     expect(result.finalTotal).toBe(180000);           |
+-------------------------------------------------------+
             |
             +-----------------------+-----------------------+
             |                                               |
     [ Hasil Cocok ]                                 [ Hasil Tidak Cocok ]
             |                                               |
             v                                               v
       STATUS: PASS (Hijau)                            STATUS: FAIL (Merah)
    Eksekusi tes berikutnya                    Tampilkan diff: expected vs actual
```

## 5. Cuplikan Logika Inti

### Unit yang Diuji (Pure Utility Function)

```typescript
// src/utils/pricing.ts
export interface CartItem {
  id: string;
  price: number;
  quantity: number;
}

export function calculateOrderTotal(
  items: CartItem[],
  taxRate: number = 0.11,
  discountPercentage: number = 0
): { subtotal: number; discount: number; tax: number; total: number } {
  if (items.length === 0) {
    return { subtotal: 0, discount: 0, tax: 0, total: 0 };
  }

  const subtotal = items.reduce((acc, item) => acc + item.price * item.quantity, 0);
  const discount = Math.round(subtotal * (discountPercentage / 100));
  const taxableAmount = Math.max(0, subtotal - discount);
  const tax = Math.round(taxableAmount * taxRate);
  const total = taxableAmount + tax;

  return { subtotal, discount, tax, total };
}
```

### File Pengujian Unit (Menggunakan Vitest)

```typescript
// src/utils/pricing.test.ts
import { describe, it, expect } from 'vitest';
import { calculateOrderTotal, CartItem } from './pricing';

describe('calculateOrderTotal()', () => {
  it('harus mengembalikan 0 untuk keranjang belanja kosong', () => {
    const result = calculateOrderTotal([]);
    expect(result).toEqual({ subtotal: 0, discount: 0, tax: 0, total: 0 });
  });

  it('harus menghitung subtotal dan pajak standar (PPN 11%) dengan benar', () => {
    const mockItems: CartItem[] = [
      { id: 'item-1', price: 100000, quantity: 2 }, // 200.000
      { id: 'item-2', price: 50000, quantity: 1 },  // 50.000
    ];

    const result = calculateOrderTotal(mockItems, 0.11, 0);

    expect(result.subtotal).toBe(250000);
    expect(result.discount).toBe(0);
    expect(result.tax).toBe(27500); // 11% dari 250.000
    expect(result.total).toBe(277500);
  });

  it('harus menerapkan diskon sebelum menghitung nilai pajak', () => {
    const mockItems: CartItem[] = [
      { id: 'item-1', price: 100000, quantity: 1 },
    ];

    // Diskon 20%
    const result = calculateOrderTotal(mockItems, 0.11, 20);

    expect(result.subtotal).toBe(100000);
    expect(result.discount).toBe(20000);
    expect(result.tax).toBe(8800); // 11% dari 80.000
    expect(result.total).toBe(88800);
  });
});
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

| Kelebihan | Kekurangan / Risiko |
| :--- | :--- |
| Kecepatan eksekusi sangat tinggi (ribuan tes dalam beberapa detik) | Tidak bisa menguji integrasi nyata antar modul atau rendering browser |
| Memberikan sinyal lokasi error yang sangat presisi hingga nomor baris | Terlalu banyak mock (tiruan) bisa membuat tes lulus padahal sistem asli gagal |
| Menurunkan biaya debugging secara drastis jika ditulis sejak awal | Memerlukan waktu pemeliharaan tes saat struktur signature fungsi berubah |

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Wajib Menggunakan Unit Test
- Fungsi logika bisnis murni: utilitas kalkulasi uang, format tanggal, generator slug, validasi regex, transformasi data API ke view model.
- Reducers dan state mutation machines (Redux reducers, Zustand actions, Immer producers).
- Library atau SDK internal yang akan digunakan oleh banyak modul lain.

### Kapan Sebaiknya Menggunakan Level Tes Lain
- Menguji alur pengguna lintas halaman (gunakan **E2E Testing**).
- Menguji interaksi DOM, form submission, dan rendering visual komponen React (gunakan **Component Testing** atau **Integration Testing**).

## 8. Metrik Kunci & Observabilitas

1. **Test Execution Time**: Pantau total waktu run seluruh unit test di CI. Proyek sehat biasanya mengeksekusi ribuan unit test di bawah 10-15 detik.
2. **Branch Coverage on Business Logic**: Targetkan branch coverage > 85% khusus untuk berkas utilitas logika kalkulasi dan reducers.
3. **Flaky Test Count**: Unit test murni harus memiliki angka kegagalan acak (*flakiness*) bernilai 0%. Jika ada unit test yang kadang lulus kadang gagal, periksa penggunaan `Date.now()`, `Math.random()`, atau timer asinkron tanpa fake timers.

## 9. Studi Kasus Industri Nyata

Sebuah perusahaan fintech gateway pembayaran memproses jutaan transaksi multi-mata uang setiap hari. Di awal peluncuran, tim tidak mewajibkan penulisan unit test untuk modul konversi kurs valuta asing demi mengejar deadline rilis fitur.

Suatu hari, bug pembulatan angka desimal (*floating point precision error* pada JavaScript: `0.1 + 0.2 !== 0.3`) menyebabkan nilai transaksi valas terpotong selisih Rp 1 hingga Rp 10 per penarikan dana. Dalam sebulan, akumulasi selisih tersebut menyebabkan selisih neraca audit perbankan hingga puluhan juta rupiah.

Solusi engineering:
1. Menulis rangkaian unit test komprehensif yang menguji ratusan skenario edge case: angka desimal panjang, konversi mata uang tanpa desimal (seperti Yen Jepang), nilai nol, dan angka negatif.
2. Memasang hook git pre-commit yang memblokir commit jika ada unit test yang gagal.
3. Seluruh kalkulasi dialihkan menggunakan library pembulatan eksak (seperti `bignumber.js`) yang terbukti lulus pada 100% unit test.

## 10. Fakta Menarik & Gotchas Production

- **Perangkap Floating Point**: Di JavaScript, `0.1 + 0.2 = 0.30000000000000004`. Dalam unit test perhitungan uang, gunakan `toBeCloseTo()` atau bandingkan representasi integer sen terkecil untuk menghindari kegagalan assertion semu.
- **Jangan Menguji Detail Implementasi**: Uji apa yang dihasilkan fungsi (*public contract / return value*), bukan nama variabel privat di dalam fungsi tersebut. Menguji detail implementasi membuat tes rapuh saat refactoring.
- **Fake Timers**: Jika fungsi Anda menggunakan `setTimeout` atau `setInterval`, gunakan `vi.useFakeTimers()` di Vitest daripada membiarkan unit test Anda menunggu waktu nyata secara lambat.

## 11. Rangkuman Singkat

Unit Testing adalah pengujian terisolasi pada unit terkecil kode (fungsi murni dan kalkulasi) tanpa dependensi eksternal. Dengan siklus Arrange-Act-Assert, unit test memberikan umpan balik instan kepada developer dan menjadi jaring pengaman paling kokoh terhadap bug regresi.
