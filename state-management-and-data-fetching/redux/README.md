# Redux (Redux Toolkit)

## 1. Skenario Sistem Nyata

Pada aplikasi perbankan investasi dan bursa perdagangan aset digital (Crypto / Stock Trading), terdapat alur eksekusi order transaksi yang sangat kritis. Pengguna dapat melakukan order beli, memasang batas kerugian (*Stop Loss*), dan membagi order menjadi beberapa sub-order otomatis.

Regulator kepatuhan keuangan mewajibkan sistem memiliki:
- **Audit Trail 100% Deterministik**: Setiap perubahan angka pada formulir order harus tercatat: aksi apa yang terjadi, jam berapa, nilai sebelum dan sesudah mutasi.
- **Fitur Simulasi & Replay**: Tim QA dan analis risiko harus dapat mengulang kembali (*replay*) seluruh rentetan klik dan kalkulasi pengguna detik demi detik saat terjadi insiden salah eksekusi dana.

Jika aplikasi menggunakan state management biasa yang berbasis mutasi langsung tanpa struktur kaku:
- Riwayat transaksi tidak dapat direkonstruksi secara runut.
- Sulit membedakan apakah sebuah perubahan angka disebabkan oleh kalkulasi diskon lokal, event websocket harga pasar, atau klik pengguna.

Solusi: Menggunakan **Redux (khususnya Redux Toolkit / RTK)**. Arsitektur Redux yang berbasis *Actions*, *Reducers (Pure Functions)*, dan *Immutable State* menjamin bahwa seluruh mutasi data tercatat secara kronologis dan dapat diputar ulang (*Time-Travel Debugging*) dengan akurasi 100%.

## 2. Apa Itu dan Buat Apa?

Redux adalah pustaka manajemen state global yang berbasis pada pola arsitektur **Flux** dan prinsip **Immutability**. 

Dalam Redux:
1. Seluruh status aplikasi disimpan di dalam **satu objek pohon tunggal (*Single Store of Truth*)**.
2. State bersifat *Read-Only* (hanya baca). Satu-satunya cara untuk mengubah state adalah dengan mengirimkan (*dispatch*) sebuah objek **Action** yang mendeskripsikan peristiwa yang terjadi.
3. Perubahan data dihitung oleh fungsi murni (*pure functions*) yang disebut **Reducers**, yang menerima state lama dan action, lalu mengembalikan salinan state baru.

Di era modern, komunitas Redux secara resmi mewajibkan penggunaan **Redux Toolkit (RTK)**, yang menyederhanakan konfigurasi boilerplate lama dan mengintegrasikan library **Immer** secara otomatis.

Fungsi utama Redux:
- Mengelola state global yang sangat kompleks dan berskala enterprise dengan prediktabilitas mutlak.
- Menyediakan kemampuan debugging kelas industri melalui ekosistem **Redux DevTools**.

## 3. Kenapa Harus Pakai Ini?

Redux adalah standar de-facto teruji (*battle-tested*) pada aplikasi web skala enterprise global:
1. **Prediktabilitas Skala Masif**: Tidak ada mutasi tersembunyi; setiap mutasi memiliki nama aksi dan muatan data (*payload*).
2. **Dukungan Middleware Tangguh**: Memungkinkan penyisipan logika intersepsi di antara aksi dan reducer (misal: logging otomatis ke Datadog, validasi keamanan, sinkronisasi persistensi memori).
3. **Ekosistem RTK Query**: Menyertakan modul data-fetching dan caching bawaan yang terintegrasi langsung dengan Redux Store.

## 4. Cara Kerjanya (Diagram Alur Redux Unidirectional Cycle)

```
Pengguna Mengklik Tombol "Beli Saham AAPL"
                         |
                         v
+---------------------------------------------------------------+
|                       KOMPONEN UI (View)                      |
|       dispatch(orderPlaced({ symbol: 'AAPL', qty: 10 }))      |
+---------------------------------------------------------------+
                               |
                               v
+---------------------------------------------------------------+
|                        MIDDLEWARE LAYER                       |
|  - Logger (Mencatat log ke konsol & monitoring backend)       |
|  - Analytics Tracker (Kirim event ke Google Analytics)       |
+---------------------------------------------------------------+
                               |
                               v
+---------------------------------------------------------------+
|                        ROOT REDUCER                           |
|  (Fungsi Murni: Menghitung saldo baru & status pesanan)       |
|  state.balance -= totalCost;                                  |
|  state.orders.push(newOrder);                                 |
+---------------------------------------------------------------+
                               |
                               v
+---------------------------------------------------------------+
|                        REDUX STORE                            |
|             (Pohon State Baru Disimpan di RAM)                |
+---------------------------------------------------------------+
                               |
           Notifikasi ke Komponen via useSelector()
                               v
+---------------------------------------------------------------+
|                       KOMPONEN UI (View)                      |
|          Saldo baru dan daftar portofolio ter-update!         |
+---------------------------------------------------------------+
```

## 5. Cuplikan Logika Inti

### Implementasi Modern Menggunakan Redux Toolkit (RTK)

Pola modern yang ringkas menggunakan `createSlice`:

```typescript
// features/trading/tradingSlice.ts
import { createSlice, PayloadAction } from "@reduxjs/toolkit";

interface Order {
  id: string;
  symbol: string;
  quantity: number;
}

interface TradingState {
  balance: number;
  orders: Order[];
}

const initialState: TradingState = {
  balance: 50000,
  orders: [],
};

// createSlice secara otomatis membuat Action Creators dan Reducer
export const tradingSlice = createSlice({
  name: "trading",
  initialState,
  reducers: {
    // Berkat integrasi library Immer di RTK, kita BISA menulis mutasi "langsung"
    // Immer akan secara ajaib mengonversinya menjadi salinan immutable di belakang layar!
    placeOrder: (state, action: PayloadAction<Order>) => {
      state.orders.push(action.payload);
      state.balance -= action.payload.quantity * 150; // Asumsi harga per unit
    },
    resetPortfolio: (state) => {
      state.orders = [];
    },
  },
});

export const { placeOrder, resetPortfolio } = tradingSlice.actions;
export default tradingSlice.reducer;
```

Penggunaan di Komponen React:
```typescript
// components/OrderButton.tsx
import React from "react";
import { useDispatch, useSelector } from "react-redux";
import { RootState, AppDispatch } from "../app/store";
import { placeOrder } from "../features/trading/tradingSlice";

export function OrderButton() {
  const dispatch = useDispatch<AppDispatch>();
  
  // Gunakan selector untuk membaca state secara presisi
  const balance = useSelector((state: RootState) => state.trading.balance);

  const handleBuy = () => {
    dispatch(placeOrder({ id: "ord-101", symbol: "AAPL", quantity: 5 }));
  };

  return (
    <div>
      <p>Saldo Tersedia: ${balance}</p>
      <button onClick={handleBuy}>Beli 5 Lembar Saham AAPL</button>
    </div>
  );
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan
- **Audit Jejak Terbaik di Industri**: Ekstensi Redux DevTools memungkinkan Anda melihat setiap mutasi, membatalkan aksi (*Undo/Jump*), dan mengekspor seluruh state ke file JSON.
- **Konsistensi Arsitektur Tim Besar**: Standar slicing dan konvensi yang sangat terstruktur mencegah developer menulis pola yang berbeda-beda.
- **Performa Selektor Teroptimasi**: Integrasi dengan pustaka `reselect` memungkinkan memoized selectors untuk kalkulasi data turunan (*derived state*).

### Kekurangan
- **Boilerplate Paling Tinggi**: Masih membutuhkan lebih banyak baris kode konfigurasi (store configuration, slice definitions, typed hooks) dibandingkan library minimalis seperti Zustand.
- **Kurva Pembelajaran Curam**: Konsep reducer murni, middleware, immutability, dan thunk membutuhkan pemahaman mendalam dari developer junior.
- **Ukuran Bundle Tambahan**: Ukuran bundel `@reduxjs/toolkit` (~13 KB gzipped) lebih besar daripada alternatif modern lainnya.

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Pakai
- Aplikasi perbankan, fintech, e-commerce enterprise dengan puluhan alur transaksi yang kompleks.
- Aplikasi yang mewajibkan fitur undo/redo, kolaborasi multipengguna yang kompleks, atau kepatuhan audit log ketat.
- Tim developer berskala puluhan orang yang membutuhkan standar struktur kode yang kaku agar tidak ada developer yang menulis mutasi liar.

### Kapan Jangan Pakai
- Proyek berskala kecil atau menengah, MVP startup, atau prototipe cepat. (Gunakan **Zustand** atau **Context API**).
- Jika kebutuhan Anda 90% murni hanya mengambil dan menampilkan data dari server API (Gunakan **TanStack Query**).

## 8. Metrik Kunci & Observabilitas

- **Redux DevTools Action Latency**: Pastikan durasi eksekusi reducer untuk setiap action tetap di bawah **2 milidetik**. Reducer tidak boleh melakukan operasi I/O atau loop berat!
- **State Tree Size**: Amati ukuran total objek Redux Store. Jika ukuran objek JSON store melebihi beberapa megabyte di RAM, pertimbangkan melakukan normalisasi data (*normalization*) dengan `createEntityAdapter`.

## 9. Studi Kasus Industri Nyata

### Normalisasi Data Kompleks pada Instagram Web

- **Tantangan Rekayasa Riil**:
  Feed Instagram memuat postingan gambar, akun pembuat, komentar, dan like. Satu pengguna yang sama bisa muncul sebagai pembuat postingan di feed, sekaligus muncul di kotak komentar postingan lain, dan muncul di daftar story. Pada arsitektur lama tanpa normalisasi, jika pengguna tersebut mengganti foto profilnya, foto profil di 5 komponen berbeda tidak sinkron karena data terduplikasi di beberapa cabang state.
- **Implementasi Solusi Industri**:
  Instagram mengadopsi Redux dengan pola **Normalized State** menggunakan schema relational (ID-based lookup):
  `state.users.byId = { 'usr-1': { name: 'Syafri', avatar: '...' } }`.
  Postingan hanya menyimpan ID user (`authorId: 'usr-1'`). Ketika foto profil diperbarui, Redux hanya memutasi 1 entitas di `users.byId['usr-1']`, dan seluruh komponen di aplikasi otomatis menampilkan foto terbaru.
- **Dampak & Hasil Sistem**:
  - Inkonsistensi data pengguna antar komponen tereliminasi 100%.
  - Pengurangan konsumsi memori cache hingga 50% karena ketiadaan data ganda.

## 10. Fakta Menarik & Gotchas Production

1. **Reducer WAJIB Murni (Pure Function)**:
   Jangan pernah memanggil `Math.random()`, `new Date()`, atau menembak request `fetch()` di dalam fungsi Reducer! Reducer harus deterministik: input state A + action B harus SELALU menghasilkan output state C yang sama persis kapan pun dan di mana pun dieksekusi.
2. **Redux Toolkit Menyertakan Immer Secara Bawaan**:
   Di masa lalu (Redux lama), menulis reducer membutuhkan sintaksis spread operator yang mengerikan: `{ ...state, user: { ...state.user, address: { ...state.user.address, city: 'Jakarta' } } }`. Dengan Redux Toolkit modern, Anda cukup menulis: `state.user.address.city = 'Jakarta'`.

## 11. Rangkuman Singkat

1. **Satu Sumber Kebenaran**: Redux menyimpan seluruh state aplikasi dalam satu pohon objek terpusat.
2. **Aksi & Reducer Murni**: Perubahan hanya bisa dipicu melalui Action dan dihitung oleh Reducer fungsi murni.
3. **Penyelamat Enterprise**: Pilihan terbaik untuk aplikasi skala besar yang membutuhkan auditabilitas mutlak dan debugging time-travel.
4. **Wajib Gunakan RTK**: Selalu gunakan Redux Toolkit (`@reduxjs/toolkit`) modern; hindari menulis kode Redux gaya lama yang bertele-tele.
