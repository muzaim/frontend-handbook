# Global State

## 1. Skenario Sistem Nyata

Pada aplikasi e-commerce berskala besar, data **Keranjang Belanja (*Shopping Cart*)** dibutuhkan oleh banyak komponen yang terletak di cabang pohon DOM yang saling berjauhan:
- Komponen **Navbar Header** membutuhkan total jumlah item untuk menampilkan angka lencana (*badge count*): `Cart (3)`.
- Komponen **Halaman Detail Produk** membutuhkan status keranjang untuk mengubah tombol menjadi "Sudah di Keranjang".
- Komponen **Drawer Panel Keranjang** membutuhkan daftar lengkap item, kupon diskon, dan subtotal harga.
- Komponen **Halaman Checkout** membutuhkan data barang untuk proses pembayaran akhir.

Jika developer hanya mengandalkan Local State dan menyalurkan data ini lewat `props`:
- Developer terpaksa melakukan **Prop Drilling Ekstrem** melewati 15 lapis komponen perantara (dari `<App>` -> `<Layout>` -> `<MainContent>` -> `<Page>` -> `<Widget>` -> ...).
- Komponen-komponen perantara yang sebenarnya tidak peduli dengan data keranjang terpaksa ikut menerima dan meneruskan props tersebut. Jika struktur props diubah, 15 berkas komponen harus ikut diubah secara manual.

Solusi: Menggunakan **Global State (State Global)**. Menyimpan data keranjang di satu wadah penyimpanan terpusat (*Single Store of Truth*) di luar pohon komponen. Komponen mana pun yang membutuhkan data keranjang dapat langsung berlangganan (*subscribe*) ke store tersebut secara independen tanpa melewati perantara.

## 2. Apa Itu dan Buat Apa?

Global State (Status Global atau *Application State*) adalah data dinamis yang disimpan di luar pohon komponen hierarki visual, dapat diakses dan diperbarui oleh **komponen mana saja di seluruh aplikasi**, terlepas dari seberapa dalam atau seberapa jauh posisi komponen tersebut di dalam DOM.

Pustaka populer yang mengelola Global State: Redux Toolkit, Zustand, Pinia, MobX, Recoil, Jotai, dan React Context API.

Fungsi utama Global State:
- **Menghilangkan Prop Drilling**: Membebaskan komponen perantara dari tugas meneruskan data yang tidak mereka gunakan.
- **Konsistensi Data Lintas Halaman (*Single Source of Truth*)**: Menjamin bahwa ketika keranjang diperbarui di halaman produk, lencana di navbar langsung mencerminkan angka yang sama persis tanpa ada ketidaksinkronan data.
- **Mempertahankan State Lintas Navigasi**: Data tetap utuh di memori saat pengguna berpindah rute halaman, berbeda dengan local state yang musnah saat unmount.

## 3. Kenapa Harus Pakai Ini?

Tanpa Global State, aplikasi web kompleks yang memiliki ratusan fitur yang saling bergantung akan menjadi rapuh. Mengubah satu alur data mengharuskan developer merombak seluruh rantai hierarki komponen.

Global State menciptakan saluran komunikasi langsung (*direct bus*) antara produsen data (komponen yang mengubah data) dan konsumen data (komponen yang menampilkan data).

## 4. Cara Kerjanya (Diagram Alur Prop Drilling vs Global Store)

```
PROP DRILLING (Rantai Panjang & Melelahkan):
[ Root App: cartState ]
         | (lewatkan props)
         v
[ Main Layout ]
         | (lewatkan props)
         v
[ Dashboard View ]
         | (lewatkan props)
         v
[ Product Section ]
         | (lewatkan props)
         v
[ Target: AddToCartButton ]  <--- Baru sampai di sini!

----------------------------------------------------------------------

GLOBAL STATE PATTERN (Akses Langsung via Subscription):
                     +---------------------------+
                     |    GLOBAL STATE STORE     |
                     |  cart: [ { id: 1, qty: 2 }]
                     +---------------------------+
                                   |
         +-------------------------+-------------------------+
         | Langganan langsung                                | Langganan langsung
         v                                                   v
+------------------+                               +--------------------+
|  Navbar (Badge)  |                               | CheckoutButton     |
|  *Langsung baca! |                               | *Langsung update!  |
+------------------+                               +--------------------+
```

## 5. Cuplikan Logika Inti

### Pola Global Store Modern Berbasis Selector (Contoh Menggunakan Zustand)

Zustand adalah salah satu library global state paling populer karena menggunakan selektor presisi untuk mencegah re-render yang tidak diinginkan:

```typescript
// stores/cartStore.ts
import { create } from "zustand";

interface CartItem {
  id: string;
  title: string;
  price: number;
}

interface CartStore {
  items: CartItem[];
  addItem: (item: CartItem) => void;
  clearCart: () => void;
}

export const useCartStore = create<CartStore>((set) => ({
  items: [],
  addItem: (item) => set((state) => ({ items: [...state.items, item] })),
  clearCart: () => set({ items: [] }),
}));
```

Komponen Konsumen A (Hanya membaca total jumlah item):
```typescript
// components/NavbarCartBadge.tsx
import { useCartStore } from "../stores/cartStore";

export function NavbarCartBadge() {
  // ATURAN EMAS: Gunakan selector! Komponen ini HANYA akan re-render jika jumlah item berubah,
  // BUKAN jika harga atau nama produk di dalam keranjang berubah.
  const itemCount = useCartStore((state) => state.items.length);

  return <div className="badge">Keranjang: {itemCount}</div>;
}
```

Komponen Konsumen B (Menambah item baru):
```typescript
// components/AddToCartButton.tsx
import { useCartStore } from "../stores/cartStore";

export function AddToCartButton({ product }) {
  // Hanya mengambil fungsi action tanpa mendengarkan perubahan data items!
  const addItem = useCartStore((state) => state.addItem);

  return (
    <button onClick={() => addItem(product)}>
      Tambah ke Keranjang
    </button>
  );
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan
- **Arsitektur Bersih dari Prop Drilling**: Komponen UI fokus murni pada rendering tampilan lokal tanpa dibebani mengurus data tetangga.
- **Dukungan Time-Travel Debugging**: Status global dapat dilacak, direkam, dan dibalikkan (*undo/redo*) menggunakan DevTools (seperti Redux DevTools).
- **Sinkronisasi Lintas Fitur Otomatis**: Perubahan satu state langsung terdistribusi ke seluruh subscriber aktif.

### Kekurangan & Risiko
- **Bahaya Re-Render Massal**: Jika selector tidak digunakan dengan hati-hati (misal menulis `const store = useCartStore()`), seluruh komponen yang memanggil hook tersebut akan di-render ulang setiap kali ada nilai apa pun di store yang berubah.
- **Penyimpanan Objek Zombie di RAM**: Data yang disimpan di global state tidak akan pernah dihapus oleh Garbage Collector selama tab browser masih terbuka, meningkatkan risiko memory bloat.

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Menggunakan Global State
- Informasi sesi pengguna yang sedang login (User Token, Hak Akses / Role, Profil Pengguna).
- Tema tampilan global (Dark Mode vs Light Mode).
- Pengaturan preferensi bahasa aplikasi (i18n Locale).
- Keranjang belanja e-commerce atau daftar putar (*playlist*) pemutar media musik.
- Antrean notifikasi global (*Toast Notifications*).

### Kapan JANGAN Menggunakan Global State
- Form input teks sementara (gunakan **Local State**).
- Hasil fetch data dari API server yang memiliki status caching, loading, dan revalidasi (gunakan **Server State** seperti TanStack Query, BUKAN global store manual!).

## 8. Metrik Kunci & Observabilitas

- **Redux / Zustand DevTools Action History**: Amati apakah ada aksi (*actions*) yang ditembakkan berulang kali secara tidak wajar dalam rentang beberapa milidetik (*dispatch spamming*).
- **Subscriber Count**: Pastikan jumlah komponen yang berlangganan ke state besar tidak melebihi ratusan tanpa selektor atomik.

## 9. Studi Kasus Industri Nyata

### Evolusi State Management di Atlassian (Jira Web)

- **Tantangan Rekayasa Riil**:
  Jira board memiliki ribuan tiket pekerjaan. Dulunya, Atlassian menaruh seluruh tiket, komentar, status dropdown, hingga posisi drag-and-drop ke dalam satu Redux Store global raksasa. Menarik satu kartu tiket (*drag ticket*) memicu ratusan selector Redux dievaluasi, menyebabkan papan board terasa berat (*sluggish*).
- **Implementasi Solusi Industri**:
  Atlassian memecah arsitektur state:
  1. Data tiket server dipindahkan keluar dari Redux ke layer cache data otomatis.
  2. Redux store global hanya dipertahankan untuk data tingkat tinggi yang stabil (identitas user, konfigurasi project board).
  3. Status interaksi drag-and-drop dipindahkan sepenuhnya ke Local State komponen board.
- **Dampak & Hasil Sistem**:
  - FPS saat melakukan drag kartu tiket meningkat dari 25 FPS menjadi 60 FPS mulus.
  - Kompleksitas kode boilerplate berkurang drastis.

## 10. Fakta Menarik & Gotchas Production

1. **Jebakan Shallow Comparison pada Object Selector**:
   Jika selector mengembalikan objek baru setiap kali dipanggil:
   ```javascript
   // JEBAKAN FATAL:
   const { items, user } = useMyStore((state) => ({ items: state.items, user: state.user }));
   ```
   Objek `{ items, user }` adalah referensi memori baru di setiap pemanggilan! Hal ini menyebabkan komponen SELALU re-render pada mutasi state apa pun di store.  
   *Solusi*: Gunakan fungsi pembanding `shallow` bawaan library atau panggil hook secara terpisah per properti.
2. **Global State Bukan Pengganti Database**:
   Jangan menggunakan global store untuk menyimpan ribuan entitas data yang jarang dibuka. Simpan hanya apa yang sedang dibutuhkan oleh viewport aplikasi saat ini.

## 11. Rangkuman Singkat

1. **Akses Universal**: Global state memungkinkan data diakses dari mana saja tanpa melewati rantai prop drilling.
2. **Kendalikan Cakupan**: Gunakan hanya untuk data aplikasi sejati (User Session, Cart, Global Preferences, Toast).
3. **Wajib Selektor**: Selalu gunakan fungsi selector atomik untuk membatasi re-render hanya pada properti yang dipantau.
4. **Pisahkan dari Server Data**: Jangan kelola cache data server di global store manual; serahkan urusan API ke Server State library.
