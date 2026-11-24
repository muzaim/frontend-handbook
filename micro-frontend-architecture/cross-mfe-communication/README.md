# Cross-MFE Communication

## 1. Skenario Sistem Nyata

Sebuah platform marketplace multi-vendor memisahkan halaman belanja menjadi dua Micro-App independen:
1. `mfe-navbar` (Tim Navigasi): Memuat header, menu profil, dan ikon keranjang belanja dengan badge jumlah barang (`CartBadge`).
2. `mfe-product-detail` (Tim Transaksi): Memuat deskripsi barang, varian warna, dan tombol "Tambah ke Keranjang".

Untuk menghubungkan keduanya, seorang arsitek awal memutuskan membuat satu store Redux global di aplikasi Host dan membagikan (*share*) instance Redux tersebut ke seluruh Micro-App.

Enam bulan kemudian, arsitektur mengalami kebuntuan fatal (*architectural dead-end*):
- Setiap kali Tim Transaksi ingin menambah properti baru pada payload cart, mereka harus meminta izin dan berkoordinasi dengan Tim Navigasi.
- Terjadi *tight coupling*: pembaruan versi minor Redux di satu tim merusak tipe data di tim lain.
- Prinsip otonomi Micro Frontends hancur karena seluruh tim kembali terikat pada satu *shared state store* monolitik di memori.

Pendekatan *Shared Global Store* antar Micro-App adalah antipattern. Solusi industri yang teruji adalah **Komunikasi Berbasis Event Terdekupel (Event-Driven Decoupled Communication)**.

## 2. Apa Itu dan Buat Apa?

Cross-MFE Communication adalah strategi dan mekanisme pertukaran data atau pemberitahuan aksi antar Micro-App yang berjalan berdampingan di halaman yang sama, dengan tetap mempertahankan batas otonomi (*loose coupling*) masing-masing aplikasi.

Prinsip emas komunikasi Micro Frontends:
1. **Minimalkan Komunikasi**: Jika dua komponen harus terus-menerus berbagi data secara intensif (misal: setiap milidetik saat mengetik teks), kedua komponen tersebut seharusnya berada di dalam **satu Micro-App yang sama**, bukan dipisah secara artifisial!
2. **Kirim Event, Bukan State Utuh**: Jangan membagikan seluruh objek database atau store. Cukup kirimkan sinyal kejadian (*domain events*), misalnya: `"CartUpdated"` atau `"UserLoggedOut"`.
3. **Agnostik Framework**: Mekanisme komunikasi tidak boleh memaksa remote app menggunakan pustaka state tertentu (React, Vue, atau Vanilla JS harus bisa saling berkomunikasi).

## 3. Kenapa Harus Pakai Ini? (Empat Pola Komunikasi Utama)

1. **Native DOM CustomEvents (`window.dispatchEvent`)**:
   - Memanfaatkan standar native browser `CustomEvent`. Sangat ringan (0 KB bundle), bekerja lintas framework, dan langsung didukung oleh semua browser.
2. **Type-Safe Event Bus (Shared Pub/Sub)**:
   - Pustaka mini independen (seperti `mitt` atau `nanoevents`, < 1KB) yang dibagikan antar tim dengan kontrak tipe data TypeScript (*Contract Types*) yang ketat.
3. **URL Query Parameters & Path State**:
   - Menggunakan URL browser sebagai sumber kebenaran (*Single Source of Truth*), misal: `?product_id=123&currency=USD`. Semua Micro-App membaca URL yang sama tanpa perlu saling mengirim pesan langsung.
4. **BroadcastChannel API**:
   - Saluran komunikasi pesan satu origin yang bekerja mulus antar iframe atau bahkan antar tab browser yang berbeda secara native.

## 4. Cara Kerjanya

Berikut adalah diagram alur bagaimana interaksi di Micro-App Produk mengabari Micro-App Navbar menggunakan pola **DOM CustomEvent Bus**:

```
[ MFE-Product (Tim Transaksi) ]
              |
  Pengguna klik: [ Tambah ke Keranjang ]
              |
              |-- 1. Buat event dengan payload minimal
              |      const event = new CustomEvent('app:cart-item-added', {
              |        detail: { itemId: 'prod-42', quantity: 1 }
              |      });
              v
[ BROWSER WINDOW EVENT TARGET (Event Hub) ]
  window.dispatchEvent(event)
              |
              +-------------------------------+
              |                               |
              v                               v
[ MFE-Navbar (Tim Navigasi) ]       [ MFE-Analytics (Tim Data) ]
  - Menangkap event                   - Menangkap event
  - Fetch status keranjang terbaru    - Catat metrik event ke Amplitude
  - Update angka badge (+1)           - Tidak saling kenal dengan Tim Navbar!
              |                               |
              v                               v
[ UI Navbar Terupdate ]             [ Telemetri Terkirim ]
(Ketiga aplikasi tetap terisolasi 100%, tanpa dependensi kode langsung!)
```

## 5. Cuplikan Logika Inti

### 1. Definisi Kontrak Event Terpusat (Shared Event Types)

Kedua tim menyepakati kontrak event melalui file tipe data TypeScript ringan:

```typescript
// packages/event-contracts/src/cartEvents.ts
export interface CartItemAddedPayload {
  productId: string;
  name: string;
  quantity: number;
}

export interface UserLoggedOutPayload {
  reason: 'timeout' | 'manual';
}

// Deklarasi global custom event agar type-safe di seluruh repositori
export type AppEventMap = {
  'cart:item-added': CustomEvent<CartItemAddedPayload>;
  'auth:user-logged-out': CustomEvent<UserLoggedOutPayload>;
};
```

### 2. Sisi Pengirim Pesan (MFE Product Detail)

```typescript
// mfe-product/src/services/eventDispatcher.ts
import { CartItemAddedPayload } from '@repo/event-contracts';

export function dispatchCartUpdated(payload: CartItemAddedPayload) {
  const event = new CustomEvent('cart:item-added', {
    bubbles: true,
    composed: true, // Wajib bernilai true jika MFE menggunakan Shadow DOM!
    detail: payload,
  });

  window.dispatchEvent(event);
  console.log('Event cart:item-added berhasil dipancarkan:', payload);
}
```

### 3. Sisi Penerima Pesan (MFE Navbar Badge)

```tsx
// mfe-navbar/src/components/CartBadge.tsx
import React, { useEffect, useState } from 'react';
import { CartItemAddedPayload } from '@repo/event-contracts';

export function CartBadge() {
  const [cartCount, setCartCount] = useState(0);

  useEffect(() => {
    const handleCartAdded = (event: Event) => {
      const customEvent = event as CustomEvent<CartItemAddedPayload>;
      console.log('Navbar menerima event dari produk:', customEvent.detail.productId);
      
      // Update badge secara lokal
      setCartCount((prev) => prev + customEvent.detail.quantity);
    };

    // Dengarkan event di level window
    window.addEventListener('cart:item-added', handleCartAdded);

    // Bersihkan listener saat komponen di-unmount
    return () => {
      window.removeEventListener('cart:item-added', handleCartAdded);
    };
  }, []);

  return <div className="badge">Keranjang: {cartCount}</div>;
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

| Pendekatan | Kelebihan | Kekurangan / Risiko |
| :--- | :--- | :--- |
| **DOM CustomEvents** | Zero-dependency, native browser standar, bekerja lintas semua framework | Sulit melacak riwayat state masa lalu (*tidak ada time-travel debugging*) |
| **URL Parameters** | Mampu di-bookmark, mendukung navigasi tombol Back/Forward browser | Terbatas untuk tipe data string primitif sederhana (tidak bisa objek kompleks) |
| **Shared Global Store (Anti-pattern)** | Mudah bagi developer yang terbiasa dengan Redux/Zustand | Menciptakan tight coupling, mempersulit rilis independen, dan rawan versi konflik |

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Menggunakan CustomEvents / Event Bus
- Komunikasi lintas domain bisnis yang berfrekuensi rendah hingga sedang: status login pengguna, notifikasi transaksi selesai, pembukaan drawer keranjang.
- Integrasi antar Micro-App yang dibangun menggunakan framework berbeda (misal Host React berbicara dengan Remote Vue atau Svelte).

### Kapan Harus Menyatukan Modul (Jangan Pakai MFE)
- Form multi-step di mana setiap input field saling memengaruhi validasi field lainnya secara instan. Menempatkan field form di Micro-App yang berbeda adalah tanda pembagian batas (*boundary design*) yang salah.

## 8. Metrik Kunci & Observabilitas

1. **Event Delivery Latency**: Waktu dari saat `dispatchEvent` dipanggil hingga handler penerima selesai dieksekusi. Pada browser modern, transmisi event di main thread memakan waktu kurang dari 0.1 milidetik.
2. **Zombie Listener Count**: Pastikan setiap Micro-App selalu memanggil `removeEventListener()` saat di-unmount agar tidak terjadi penumpukan listener usang di objek `window`.
3. **Payload Contract Version Drift**: Pantau apakah ada tim yang mengirim payload dengan properti yang tidak sesuai dengan schema versi paket kontrak bersama.

## 9. Studi Kasus Industri Nyata

Sebuah portal asuransi enterprise memiliki 3 Micro-App: Portal Pengajuan Polis, Portal Pembayaran Premi, dan Widget Notifikasi Header.

Di awal peluncuran, ketiga tim menggunakan modul state management bersama yang di-load via Module Federation. Ketika Tim Pembayaran merilis pembaruan schema transaksi, mereka mengubah nama properti `invoiceId` menjadi `paymentReferenceNumber`.

Perubahan ini tidak diketahui oleh Tim Notifikasi. Akibatnya, saat nasabah membayar polis senilai jutaan rupiah, transaksi sukses di database namun widget notifikasi mengalami error crash karena membaca properti `undefined`. Nasabah panik mengira uangnya hilang.

Transformasi arsitektural:
1. Menghapus seluruh shared state store.
2. Mengalihkan komunikasi menggunakan standar **Typed CustomEvent Bus** yang dikawal oleh skema Zod pada paket bersama `@company/event-contracts`.
3. Sebelum memproses event, Micro-App memvalidasi payload: jika bentuk event tidak cocok, sistem mencatat telemetri dan menampilkan fallback tanpa memicu crash antarmuka.

## 10. Fakta Menarik & Gotchas Production

- **Arti Atribut `composed: true`**: Jika salah satu Micro-App Anda dibungkus di dalam *Shadow DOM* (Web Components) untuk isolasi CSS, event native secara default **tidak bisa keluar menembus batas Shadow DOM**. Anda wajib menyetel `{ bubbles: true, composed: true }` agar event dapat melompat keluar ke objek `window`.
- **Memory Leak pada Objek Global**: Objek `window` adalah GC Root permanen. Menambahkan listener `window.addEventListener('event', fn)` tanpa fungsi cleanup di `useEffect` akan menahan seluruh komponen dan state anak di RAM selamanya.
- **Pola Request-Response Lintas MFE**: Jika MFE A membutuhkan jawaban balik dari MFE B, Anda bisa menyertakan callback identifier unik di dalam detail payload:
  `window.dispatchEvent(new CustomEvent('get:user-profile', { detail: { replyTo: 'reply:user-profile-123' } }))`.

## 11. Rangkuman Singkat

Komunikasi antar Micro-App wajib berlandaskan prinsip *loose coupling* menggunakan pola berbasis event (*Event-Driven Architecture*). Memanfaatkan native DOM `CustomEvent` atau URL query state menjaga otonomi masing-masing tim, mencegah kekacauan dependency, dan memastikan aplikasi tetap kokoh saat salah satu sub-aplikasi diperbarui secara independen.
