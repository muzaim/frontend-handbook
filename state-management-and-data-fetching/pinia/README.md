# Pinia

## 1. Skenario Sistem Nyata

Pada aplikasi dashboard manajemen gudang logistik berbasis **Vue.js**, developer menggunakan pustaka state management resmi generasi lama Vue, yaitu **Vuex 3 / 4**.

Ketika aplikasi berkembang memiliki puluhan fitur (inventaris barang, pelacakan armada truk, scan barcode, cetak surat jalan):
- Developer mengeluh penulisan kode di Vuex sangat melelahkan karena harus memisahkan logika menjadi **Mutations** (mutasi sinkron) dan **Actions** (operasi asinkron).
- Mengubah status sederhana mengharuskan developer menulis kode mutasi ganda: `commit('SET_ITEM', item)` di dalam action dan `SET_ITEM(state, item)` di dalam mutation.
- Tipe data TypeScript tidak terdukung secara alami: pemanggilan aksi berbasis string mentah (`dispatch('inventory/fetchTrucks')`) tidak memiliki validasi auto-complete (*autocomplete*), memicu bug salah ketik huruf (*typo*) saat rilis ke production.

Solusi: Beralih ke **Pinia**—standar resmi dan de-facto state management untuk ekosistem **Vue 3**. Pinia menghapus konsep *Mutations* yang bertele-tele, mendukung Composition API secara penuh, dan memberikan dukungan TypeScript otomatis tanpa konfigurasi rumit.

## 2. Apa Itu dan Buat Apa?

Pinia (dinamai dari buah nanas dalam bahasa Spanyol, melambangkan kumpulan buah kecil yang bersatu) adalah pustaka manajemen state global resmi untuk ekosistem **Vue.js** (khususnya Vue 3 dan Vue 2.7+).

Pinia menggantikan Vuex sepenuhnya dan dirancang agar selaras dengan **Composition API** dan sistem reaktivitas modern Vue.

Tiga pilar utama di dalam sebuah Pinia Store:
1. **`state`**: Fungsi yang mengembalikan objek data reaktif (setara dengan `data` di komponen Vue).
2. **`getters`**: Nilai komputasi turunan yang di-cache secara otomatis (setara dengan `computed` di komponen Vue).
3. **`actions`**: Metode untuk memutasi state atau menjalankan logika asinkron API (dapat berupa fungsi biasa atau fungsi `async`). Tidak ada lagi pemisahan artifisial antara mutation dan action!

Fungsi utama Pinia:
- Mengelola state global aplikasi Vue dengan dukungan TypeScript 100% presisi.
- Memberikan modularitas alami: setiap domain store dibuat dalam berkas terpisah tanpa konsep namespacing yang rumit.

## 3. Kenapa Harus Pakai Ini?

Bagi tim yang membangun aplikasi menggunakan Vue 3:
1. **Rekomendasi Resmi Tim Inti Vue**: Evan You dan tim resmi Vue secara eksplisit menetapkan Pinia sebagai penerus resmi Vuex.
2. **TypeScript Out-of-the-Box**: Anda mendapatkan auto-complete untuk seluruh properti state, getters, dan actions tanpa perlu menulis interface manual yang berbelit-belit.
3. **Ukuran Sangat Ringan**: Hanya berukuran sekitar **1,5 Kilobyte**.
4. **Dukungan SSR & Hydration Mulus**: Dirancang khusus agar kompatibel dengan framework SSR Nuxt.js (Nuxt 3).

## 4. Cara Kerjanya (Diagram Alur Reaktivitas Pinia Vue 3)

Pinia memanfaatkan sistem reaktivitas `reactive()` dan `computed()` bawaan dari Vue 3 engine:

```
Komponen Vue (<script setup>)
             |
             | Memanggil: store.incrementItem(id)
             v
+---------------------------------------------------------------+
|                          PINIA STORE                          |
|                                                               |
|  state: () => ({ cart: [ { id: 1, qty: 1 } ] })               |
|                                                               |
|  actions: {                                                   |
|    incrementItem(id) {                                        |
|      const item = this.cart.find(i => i.id === id);           |
|      item.qty++; // Mutasi langsung & reaktif secara alami!   |
|    }                                                          |
|  }                                                            |
|                                                               |
|  getters: {                                                   |
|    totalQty: (state) => state.cart.reduce(...)                |
|  }                                                            |
+---------------------------------------------------------------+
                               |
               Otomatis terdeteksi oleh Vue Reactivity
                               v
+---------------------------------------------------------------+
|                    DOM COMPONENT TEMPLATE                     |
|           <span>Total Barang: {{ store.totalQty }}</span>     |
+---------------------------------------------------------------+
```

## 5. Cuplikan Logika Inti

### Membuat Store Menggunakan Setup Syntax (Composition API Style)

Pinia mendukung dua gaya penulisan: *Options Stores* dan *Setup Stores*. Gaya *Setup Store* adalah yang paling direkomendasikan di Vue 3 modern:

```typescript
// stores/warehouse.ts
import { defineStore } from "pinia";
import { ref, computed } from "vue";

export const useWarehouseStore = defineStore("warehouse", () => {
  // 1. STATE (Menggunakan ref)
  const packages = ref<string[]>([]);
  const isScanning = ref<boolean>(false);

  // 2. GETTERS (Menggunakan computed)
  const totalPackages = computed(() => packages.value.length);

  // 3. ACTIONS (Fungsi biasa untuk mutasi atau fetch API)
  function addPackage(barcodeId: string) {
    packages.value.push(barcodeId);
  }

  async function syncToCloud() {
    isScanning.value = true;
    try {
      await fetch("/api/warehouse/sync", {
        method: "POST",
        body: JSON.stringify(packages.value),
      });
      packages.value = []; // Bersihkan setelah sinkronisasi
    } finally {
      isScanning.value = false;
    }
  }

  return { packages, isScanning, totalPackages, addPackage, syncToCloud };
});
```

Penggunaan di Komponen Vue (`BarcodeScanner.vue`):
```vue
<script setup lang="ts">
import { useWarehouseStore } from "@/stores/warehouse";
import { storeToRefs } from "pinia";

const store = useWarehouseStore();

// PENTING: Gunakan storeToRefs jika ingin melakukan destructuring state/getters
// agar reaktivitas tidak terputus!
const { isScanning, totalPackages } = storeToRefs(store);
const { addPackage, syncToCloud } = store;
</script>

<template>
  <div class="scanner-card">
    <h2>Total Paket Terdata: {{ totalPackages }}</h2>
    
    <button @click="addPackage('PKG-' + Date.now())">
      Simulasi Scan Paket
    </button>

    <button :disabled="isScanning" @click="syncToCloud">
      {{ isScanning ? "Menyinkronkan..." : "Kirim Data ke Pusat" }}
    </button>
  </div>
</template>
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan
- **Tanpa Mutasi Bertele-tele**: Menghapus konsep `mutations` dari Vuex lama; logika menjadi jauh lebih bersih dan ringkas.
- **Dukungan TypeScript Sempurna**: Auto-complete dan validasi tipe bekerja otomatis di IDE (VSCode / WebStorm).
- **Integrasi Vue DevTools yang Luar Biasa**: Mendukung timeline tracking, visualisasi state, dan time-travel debugging.
- **Modular Secara Alami**: Tidak ada nested modules berbelit-belit; setiap store adalah file TypeScript mandiri.

### Kekurangan & Risiko
- **Khusus Ekosistem Vue**: Tidak dapat digunakan di framework lain (React, Svelte).
- **Jebakan Destructuring**: Jika developer melakukan destructuring state secara langsung (`const { packages } = store`), reaktivitas Vue akan terputus kecuali dibungkus dengan helper `storeToRefs()`.

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Memilih Pinia
- Semua aplikasi web modern yang dibangun di atas framework **Vue 3** atau **Nuxt 3**.
- Mengelola data UI global (autentikasi pengguna, pengaturan tema, notifikasi, keranjang belanja di Vue).
- Menggantikan kode lama Vuex dalam proses migrasi proyek.

### Kapan Tidak Perlu Pinia
- Jika data hanya digunakan oleh 2 komponen yang bertetangga (gunakan `provide / inject` bawaan Vue).
- Jika Anda bekerja di ekosistem React (gunakan **Zustand** atau **Redux Toolkit**).

## 8. Metrik Kunci & Observabilitas

- **Vue DevTools Pinia Tab**: Buka tab Pinia di browser DevTools. Anda dapat memantau seluruh state aktif, memodifikasi nilai state secara langsung untuk pengujian (*live editing*), dan melihat riwayat aksi.
- **Store Subscription Tracking**: Gunakan `$subscribe()` dan `$onAction()` untuk mencatat log aktivitas pengguna ke monitoring backend (seperti Datadog atau Sentry).

## 9. Studi Kasus Industri Nyata

### Migrasi E-Commerce Carrefour Web dari Vuex ke Pinia

- **Tantangan Rekayasa Riil**:
  Situs e-commerce grosir Carrefour dibangun menggunakan Nuxt dan Vuex dengan lebih dari 80 modul bersarang (*nested modules*). Developer sering mengalami kesulitan pelacakan tipe data pada modul keranjang dan promosi diskon, serta waktu kompilasi build yang lambat.
- **Implementasi Solusi Industri**:
  Tim teknik melakukan migrasi bertahap dari Vuex ke **Pinia**:
  1. Setiap modul diubah menjadi file store flat mandiri: `useCartStore`, `useUserStore`, `usePromoStore`.
  2. Memanfaatkan `storeToRefs` untuk mempertahankan reaktivitas saat melakukan integrasi di komponen.
  3. Memanfaatkan integrasi otomatis Pinia dengan Nuxt 3 SSR hydration.
- **Dampak & Hasil Sistem**:
  - Ukuran bundle JavaScript berkurang sebesar 15%.
  - Kasus bug tipe data di runtime turun hingga 70% berkat auto-complete TypeScript Pinia.

## 10. Fakta Menarik & Gotchas Production

1. **Jebakan Kehilangan Reaktivitas Saat Destructuring**:
   ```javascript
   // BENCANA: packages kehilangan koneksi reaktif ke store!
   const { packages } = useWarehouseStore(); 

   // SOLUSI BENAR: Gunakan storeToRefs
   import { storeToRefs } from 'pinia';
   const { packages } = storeToRefs(useWarehouseStore());
   ```
2. **Fitur `$reset()` Bawaan**:
   Pinia menyediakan metode `$reset()` bawaan untuk mengembalikan seluruh nilai state di dalam store ke kondisi awal saat aplikasi pertama kali dimuat (sangat berguna untuk membersihkan data saat pengguna menekan tombol *Logout*).

## 11. Rangkuman Singkat

1. **Standar Emas Vue**: Pinia adalah pustaka manajemen state global resmi dan de-facto untuk Vue 3 dan Nuxt 3.
2. **Kompak & Simpel**: Menghilangkan mutasi bertele-tele; hanya terdiri dari `state`, `getters`, dan `actions`.
3. **Kekuatan TypeScript**: Inferensi tipe data otomatis tanpa konfigurasi rumit.
4. **Ingat storeToRefs**: Selalu gunakan `storeToRefs()` saat melakukan destructuring properti state agar reaktivitas tidak hilang.
