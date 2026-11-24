# Zustand

## 1. Skenario Sistem Nyata

Sebuah tim startup sedang membangun aplikasi pemutar audio podcast dan musik streaming berbasis web (seperti web player Spotify mini). Aplikasi memiliki fitur:
- Pemutar audio global di bagian bawah layar (*Mini Player Bar*).
- Halaman daftar lagu (*Playlist View*).
- Pengatur volume, status Shuffle, dan Repeat.

Ketika menggunakan Redux Toolkit, tim mengeluh:
- Untuk menambahkan satu status sederhana seperti `isPlaying: boolean` dan action `togglePlay()`, developer harus membuat file slice baru, mendaftarkan reducer ke store root, menulis typed hooks, dan membungkus aplikasi dengan `<Provider>`.
- Menulis fitur kecil terasa lambat karena banyaknya *ceremony* boilerplate.

Ketika menggunakan React Context API:
- Setiap kali durasi pemutaran lagu detik demi detik diperbarui (`currentTime` berubah setiap 1 detik), seluruh halaman re-render karena Context tidak memiliki selektor bawaan.

Solusi: Mengadopsi **Zustand**. Pustaka state management minimalis, modern, dan sangat cepat (~1 KB). Tidak membutuhkan `<Provider>` pembungkus, nol boilerplate, dan mendukung selektor atomik bawaan untuk mencegah re-render yang tidak diinginkan.

## 2. Apa Itu dan Buat Apa?

Zustand (kata dalam bahasa Jerman yang berarti "status" atau "kondisi") adalah pustaka manajemen state global yang ringan, cepat, dan berbasis pada prinsip Hook sederhana untuk ekosistem React.

Diciptakan oleh tim di balik *Poimandres* (pencipta React Three Fiber), Zustand dirancang dengan filosofi pragmatis: **"State management tanpa rasa sakit"**.

Karakteristik utama Zustand:
- **Nol Boilerplate**: Membuat store semudah membuat satu fungsi kustom hook.
- **Provider-less Architecture**: Tidak perlu membungkus pohon komponen dengan `<Provider store={...}>`. Hook store dapat dipanggil di komponen mana pun secara langsung.
- **Dukungan Akses di Luar Komponen**: State dapat dibaca dan dimutasi di dalam file JavaScript murni (seperti di dalam middleware fetch, helper utility, atau router navigation guard) tanpa memerlukan context React.
- **Selektor Presisi Tinggi**: Komponen hanya me-render ulang jika nilai spesifik yang dipilih berubah.

## 3. Kenapa Harus Pakai Ini?

Di era pengembangan frontend modern (2024-2026), Zustand telah menjadi pustaka state management paling dicintai oleh komunitas React:
1. **Ukuran Sangat Kecil**: Hanya sekitar **1,1 Kilobyte** (dibandingkan Redux Toolkit yang berukuran ~13 KB).
2. **Keseimbangan Terbaik**: Menghilangkan seluruh kerumitan Redux lama tanpa mengorbankan performa selektor granular.
3. **TypeScript First-Class**: Definisi tipe data otomatis terinferensi dengan sangat bersih tanpa perlu trik typing yang rumit.

## 4. Cara Kerjanya (Diagram Alur Mekanisme Pub/Sub Zustand)

Zustand menggunakan pola Publisher/Subscriber sederhana di luar React Runtime, lalu menjembatani sinkronisasi ke React menggunakan hook native `useSyncExternalStore`:

```
+---------------------------------------------------------------+
|                    ZUSTAND STORE (Di Luar React)              |
|  state = { isPlaying: false, track: 'Lagu A' }                |
|  listeners = Set([ listenerNavbar, listenerPlayerBar ])       |
+---------------------------------------------------------------+
                               |
                   Komponen memanggil: set({ isPlaying: true })
                               v
+---------------------------------------------------------------+
|                   EVALUASI SELEKTOR ATOMIK                    |
|                                                               |
|  1. Evaluasi Navbar:   (state.track)      --> Tidak berubah   |
|     HASIL: Navbar TIDAK re-render!                            |
|                                                               |
|  2. Evaluasi PlayerBar:(state.isPlaying)  --> Berubah!        |
|     HASIL: HANYA PlayerBar yang di-render ulang!              |
+---------------------------------------------------------------+
```

## 5. Cuplikan Logika Inti

### Membuat Store dan Menggunakannya di Komponen

Hanya membutuhkan satu berkas ringkas:

```typescript
// stores/useAudioStore.ts
import { create } from "zustand";
import { persist } from "zustand/middleware";

interface AudioState {
  currentTrack: string | null;
  isPlaying: boolean;
  volume: number;
  playTrack: (trackName: string) => void;
  togglePlay: () => void;
  setVolume: (vol: number) => void;
}

// Membuat store dengan middleware otomatis simpan ke localStorage!
export const useAudioStore = create<AudioState>()(
  persist(
    (set) => ({
      currentTrack: null,
      isPlaying: false,
      volume: 0.8,

      // Mutasi state langsung & deklaratif
      playTrack: (trackName) => set({ currentTrack: trackName, isPlaying: true }),
      togglePlay: () => set((state) => ({ isPlaying: !state.isPlaying })),
      setVolume: (volume) => set({ volume }),
    }),
    {
      name: "audio-storage", // Nama key di localStorage
      partialize: (state) => ({ volume: state.volume }), // Hanya simpan volume
    }
  )
);
```

Penggunaan di Komponen Antarmuka:
```typescript
// components/PlayerBar.tsx
import React from "react";
import { useAudioStore } from "../stores/useAudioStore";

export function PlayerBar() {
  // ATURAN PERFORMA: Gunakan selektor agar komponen ini HANYA re-render saat isPlaying berubah!
  const isPlaying = useAudioStore((state) => state.isPlaying);
  const togglePlay = useAudioStore((state) => state.togglePlay);
  const currentTrack = useAudioStore((state) => state.currentTrack);

  return (
    <footer className="player-bar">
      <span>Sedang Memutar: {currentTrack || "Tidak ada lagu"}</span>
      <button onClick={togglePlay}>
        {isPlaying ? "⏸ Jeda" : "▶ Putar"}
      </button>
    </footer>
  );
}
```

### Akses State di Luar Komponen React (Vanilla JavaScript):

```javascript
// helpers/networkInterceptor.js
import { useAudioStore } from "../stores/useAudioStore";

// Membaca state secara sinkron di mana saja tanpa hook:
const currentVolume = useAudioStore.getState().volume;

// Mengubah state di mana saja tanpa hook:
useAudioStore.getState().setVolume(0.5);

// Berlangganan perubahan state di file JS biasa:
const unsubscribe = useAudioStore.subscribe((state) => {
  console.log("Status audio berubah menjadi:", state.isPlaying);
});
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan
- **Super Ringan & Cepat**: Ukuran bundle ~1 KB, footprint memori minimal.
- **Nol Context Provider**: Tidak ada lagi piramida Provider bersarang di berkas `App.tsx`.
- **Middleware Kaya Bawaan**: Menyertakan middleware `persist` (localStorage/IndexedDB), `devtools` (koneksi ke Redux DevTools), dan `immer` secara langsung.

### Kekurangan & Risiko
- **Ketiadaan Konvensi Kaku (*Less Opinionated*)**: Karena Zustand sangat fleksibel, developer junior dalam tim yang tidak disiplin dapat menulis pola arsitektur store yang tidak konsisten.
- **Bahaya Lupa Selektor**: Jika developer memanggil `const store = useAudioStore()`, komponen akan re-render pada setiap perubahan properti apa pun di store tersebut.

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Memilih Zustand
- Hampir seluruh kebutuhan Global State pada aplikasi React skala kecil, menengah, hingga besar modern.
- Mengelola data UI yang kompleks (status modal, keranjang belanja, pengaturan layout, multi-step wizard).
- Membutuhkan sinkronisasi state ke penyimpanan lokal (*localStorage*) secara otomatis via middleware.

### Kapan Memilih Alternatif Lain
- Jika aplikasi Anda berskala enterprise multinasional yang mewajibkan kepatuhan audit log ketat dan konvensi arsitektur kaku yang seragam di puluhan tim (Gunakan **Redux Toolkit**).
- Jika data berasal dari REST / GraphQL API server backend (Gunakan **TanStack Query**).

## 8. Metrik Kunci & Observabilitas

- **Redux DevTools Connection**: Aktifkan middleware `devtools(myStore)` pada Zustand untuk memeriksa histori mutasi state di panel browser DevTools.
- **Selector Precision Check**: Periksa apakah komponen hanya me-render ulang saat properti targetnya dimutasi. Gunakan ekstensi React Profiler.

## 9. Studi Kasus Industri Nyata

### Migrasi Komponen Kanvas Excalidraw ke Zustand

- **Tantangan Rekayasa Riil**:
  Excalidraw adalah aplikasi papan tulis virtual (*virtual whiteboard*) open-source yang sangat populer. Saat pengguna menggambar goresan pena (*sketch strokes*) atau memindahkan elemen kanvas, koordinat kursor mouse memicu mutasi puluhan kali per detik. State management lama yang berbasis React Context menyebabkan kanvas lag dan baterai laptop cepat panas.
- **Implementasi Solusi Industri**:
  Tim Excalidraw mengadopsi **Zustand** untuk mengelola state kanvas:
  1. Status interaksi kursor dan koordinat dipindahkan ke Zustand store di luar React tree.
  2. Komponen kanvas membaca state menggunakan selektor atomik murni.
  3. Akses state di dalam requestAnimationFrame dilakukan secara langsung melalui `useStore.getState()` tanpa memicu siklus re-render React.
- **Dampak & Hasil Sistem**:
  - Goresan kanvas berjalan mulus 60-120 FPS tanpa frame drop.
  - Latensi input pen stylus berkurang drastis pada perangkat tablet.

## 10. Fakta Menarik & Gotchas Production

1. **Membalik Nilai State Tanpa Membaca Ulang**:
   Anda tidak perlu membaca state terlebih dahulu hanya untuk membalikkan boolean. Gunakan fungsi callback di dalam `set`:
   ```javascript
   toggleSidebar: () => set((state) => ({ isSidebarOpen: !state.isSidebarOpen }))
   ```
2. **Penggunaan Selector `useShallow`**:
   Jika Anda ingin mengambil beberapa properti sekaligus tanpa memicu re-render yang tidak perlu:
   ```javascript
   import { useShallow } from 'zustand/react/shallow';
   // HANYA re-render jika volume ATAU isPlaying bernilai beda secara shallow
   const { volume, isPlaying } = useAudioStore(useShallow((s) => ({ volume: s.volume, isPlaying: s.isPlaying })));
   ```

## 11. Rangkuman Singkat

1. **Minimalis & Bertenaga**: Zustand menghadirkan manajemen state global 1 KB tanpa kerumitan boilerplate.
2. **Nol Provider**: Dapat dipanggil langsung di komponen mana pun tanpa pembungkus `<Provider>`.
3. **Dukungan Vanilla JS**: Dapat dibaca dan dimutasi di luar siklus komponen React via `.getState()` dan `.setState()`.
4. **Wajib Pasang Selektor**: Selalu gunakan fungsi selector untuk menjamin performa render yang optimal.
