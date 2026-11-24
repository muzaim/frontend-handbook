# Context API

## 1. Skenario Sistem Nyata

Pada aplikasi web edukasi belajar online (LMS), terdapat fitur pergantian tema tampilan: **Light Mode vs Dark Mode**. Pengaturan tema ini dibutuhkan oleh hampir setiap komponen visual di seluruh aplikasi (Navbar, Sidebar, Kartu Materi, Tombol, Footer).

Untuk menyalurkan nilai tema ini ke seluruh komponen, developer junior membungkus seluruh aplikasi menggunakan **React Context API**:

```javascript
// Kode bermasalah: Menggabungkan state yang sering berubah ke dalam satu Context besar
const AppContext = createContext();

export function AppProvider({ children }) {
  const [theme, setTheme] = useState("dark");
  const [currentUser, setCurrentUser] = useState(null);
  const [liveNotificationCount, setLiveNotificationCount] = useState(0); // Berubah tiap 3 detik!

  return (
    <AppContext.Provider value={{ theme, setTheme, currentUser, liveNotificationCount }}>
      {children}
    </AppContext.Provider>
  );
}
```

Dampak pada sistem:
- Setiap kali ada notifikasi baru masuk (`liveNotificationCount` bertambah setiap beberapa detik), **SELURUH komponen di dalam aplikasi yang memanggil `useContext(AppContext)` ikut di-render ulang secara serentak**, termasuk komponen statis yang hanya butuh membaca properti `theme`!
- React Context tidak memiliki mekanisme selektor bawaan: jika nilai objek pada `value={{ ... }}` berganti referensi memorinya, semua komponen konsumen Context dipaksa re-render tanpa ampun.
- Aplikasi mengalami *frame drop* parah dan baterai perangkat pengguna cepat terkuras.

Solusi: Memahami karakteristik dan batasan **Context API**. Memisahkan Context menjadi konteks-konteks kecil yang terisolasi (*Split Contexts*), atau beralih ke state manager berbasis selektor (seperti Zustand) untuk data yang memiliki frekuensi pembaruan tinggi.

## 2. Apa Itu dan Buat Apa?

Context API adalah mekanisme bawaan yang disediakan oleh React (dan konsep setara seperti `provide / inject` di Vue) untuk menyalurkan data ke pohon komponen di bawahnya tanpa harus mengoper props secara manual di setiap tingkat (*Dependency Injection mechanism*).

Context terdiri dari dua komponen utama:
1. **Provider (`<MyContext.Provider value={...}>`)**: Komponen induk yang menyediakan dan menyuntikkan data ke seluruh cabang pohon di bawahnya.
2. **Consumer / Hook (`useContext(MyContext)`)**: Komponen anak di kedalaman mana pun yang membaca data dari Provider terdekat di atasnya.

Fungsi utama Context API:
- Menghilangkan friksi *Prop Drilling* untuk data global yang stabil.
- Menyediakan mekanisme injeksi dependensi lokal (misal: menyediakan konfigurasi formulir unik untuk komponen-komponen input di dalam satu tag `<Form>`).

## 3. Kenapa Harus Pakai Ini?

Context API adalah fitur bawaan standar framework (0 KB dependensi eksternal tambahan).

Sangat cocok untuk data yang memenuhi kriteria:
1. Dibutuhkan oleh banyak komponen di pohon hierarki yang dalam.
2. **Frekuensi perubahannya sangat rendah (*Low-Frequency Updates*)** (seperti tema warna, bahasa aplikasi, atau konfigurasi autentikasi pengguna saat login).

## 4. Cara Kerjanya (Diagram Alur Mekanisme Injeksi Context)

```
+---------------------------------------------------------------+
|                    <ThemeContext.Provider>                    |
|                         value="dark"                          |
+---------------------------------------------------------------+
                               |
       +-----------------------+-----------------------+
       |                                               |
       v                                               v
[ <NavbarLayout /> ]                          [ <SidebarLayout /> ]
(Tidak baca theme, tidak re-render!)          (Tidak baca theme!)
       |                                               |
       v                                               v
[ <UserProfileCard /> ]                       [ <ThemeToggleBtn /> ]
(Tidak baca theme!)                           useContext(ThemeContext)
                                              *Membaca "dark" langsung!*
```

## 5. Cuplikan Logika Inti

### Praktik Terbaik: Pemisahan State dan Dispatch (*Split Context Pattern*)

Pola paling efektif untuk mencegah re-render berlebih pada Context API di React:

```typescript
// context/ThemeContext.tsx
import React, { createContext, useContext, useState, useMemo } from "react";

type Theme = "light" | "dark";

// 1. Pisahkan Context Data (Membaca) dari Context Dispatch (Mengubah)
const ThemeStateContext = createContext<Theme | undefined>(undefined);
const ThemeDispatchContext = createContext<((theme: Theme) => void) | undefined>(undefined);

export const ThemeProvider: React.FC<{ children: React.ReactNode }> = ({ children }) => {
  const [theme, setTheme] = useState<Theme>("dark");

  return (
    <ThemeStateContext.Provider value={theme}>
      <ThemeDispatchContext.Provider value={setTheme}>
        {children}
      </ThemeDispatchContext.Provider>
    </ThemeStateContext.Provider>
  );
};

// 2. Custom hooks untuk keamanan tipe data & kemudahan penggunaan
export function useTheme() {
  const context = useContext(ThemeStateContext);
  if (context === undefined) {
    throw new Error("useTheme harus digunakan di dalam ThemeProvider");
  }
  return context;
}

export function useSetTheme() {
  const context = useContext(ThemeDispatchContext);
  if (context === undefined) {
    throw new Error("useSetTheme harus digunakan di dalam ThemeProvider");
  }
  return context;
}
```

Komponen yang hanya butuh mengubah tema:
```typescript
// components/ThemeToggle.tsx
import { useSetTheme, useTheme } from "../context/ThemeContext";

export function ThemeToggle() {
  const theme = useTheme();
  const setTheme = useSetTheme();

  return (
    <button onClick={() => setTheme(theme === "dark" ? "light" : "dark")}>
      Ganti ke {theme === "dark" ? "Light" : "Dark"} Mode
    </button>
  );
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan
- **Nol Pustaka Pihak Ketiga**: Fitur bawaan resmi React (tidak perlu install paket NPM baru).
- **Scope Berbasis Subtree**: Anda dapat memiliki beberapa Provider dari jenis konteks yang sama pada bagian layar yang berbeda (misal: area formulir A memiliki tema gelap, area formulir B memiliki tema terang).
- **Sangat Mudah Dipelajari**: Kurva pembelajaran sangat singkat bagi developer junior.

### Kekurangan & Risiko
- **Ketiadaan Selektor Bawaan (*Bailout Limitation*)**: Jika Provider mengirimkan objek `{ a, b }` dan nilai `b` berubah, komponen yang HANYA membaca nilai `a` akan tetap dipaksa melakukan render ulang!
- **Context Hell**: Terlalu banyak context bersarang membuat pohon JSX menjadi sangat dalam: `<AuthProvider><ThemeProvider><LocaleProvider><ConfigProvider>...`.

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Pakai
- Pengaturan Tema (Dark / Light mode).
- Preferensi Bahasa dan Lokalisasi (i18n).
- Sesi Autentikasi Pengguna yang stabil (data user info yang jarang berubah).
- Komponen majemuk (*Compound Components Pattern*, seperti komponen UI `<Tabs>` atau `<Accordion>` yang berbagi state lokal antar anak).

### Kapan JANGAN PERNAH Pakai
- Data yang berubah dengan frekuensi tinggi (*High-Frequency State*, seperti posisi kursor mouse, data streaming real-time via WebSocket, timer stopwatch, input form ketik cepat). Untuk kasus ini, gunakan **Zustand** atau **Jotai**.
- Seluruh cache data dari server API backend (gunakan **TanStack Query**).

## 8. Metrik Kunci & Observabilitas

- **Component Re-render Highlighting (React DevTools)**: Perhatikan apakah tombol atau teks berkedip hijau saat context diperbarui. Jika elemen yang tidak berkepentingan ikut berkedip, segera pisahkan context-nya.
- **Context Value Reference Stability**: Pastikan objek `value` di dalam Provider dibungkus dengan `useMemo` jika nilai tersebut berupa objek literal:
  ```javascript
  // PENTING: Bungkus dengan useMemo agar referensi memori tidak selalu baru di tiap render
  const value = useMemo(() => ({ theme, toggleTheme }), [theme]);
  ```

## 9. Studi Kasus Industri Nyata

### Refactoring Dashboard Shopify Polaris

- **Tantangan Rekayasa Riil**:
  Sistem desain Shopify Polaris dulunya mengandalkan satu AppContext raksasa untuk menyalurkan informasi toko, ukuran layar, hak akses staf, dan mata uang transaksi. Ketika ukuran jendela browser diubah (*window resize*), state ukuran layar di context diperbarui, menyebabkan seluruh halaman dashboard toko e-commerce re-render dan terasa macet.
- **Implementasi Solusi Industri**:
  Shopify memecah monolit context tersebut menjadi **Micro-Contexts**:
  1. `ViewportContext` dipisahkan khusus untuk ukuran layar.
  2. `ShopSettingsContext` untuk konfigurasi toko yang statis.
  3. `StaffAuthContext` untuk autentikasi.
- **Dampak & Hasil Sistem**:
  - Re-render yang tidak perlu pada tabel pesanan berhasil dieliminasi 100% saat pengguna mengubah ukuran jendela browser.
  - Performa scrolling dan responsivitas UI meningkat drastis.

## 10. Fakta Menarik & Gotchas Production

1. **Jebakan `value={{ foo, bar }}` Tanpa useMemo**:
   Jika Anda menulis `<MyContext.Provider value={{ count }}>`, objek `{ count }` adalah objek baru di memori di SETIAP KALI komponen Provider re-render. Akibatnya, seluruh komponen konsumen Context akan re-render meskipun nilai `count` di dalamnya sebenarnya tidak berubah!
2. **Context Bukan Pengganti State Management Kompleks**:
   Banyak artikel tutorial mengklaim: *"Context API menggantikan Redux!"*. Ini adalah mitos keliru. Context hanyalah pipa transportasi data (*Dependency Injection*), bukan pustaka state management dengan algoritma selektor atau middleware mutasi.

## 11. Rangkuman Singkat

1. **Injeksi Dependensi Bawaan**: Context API menyalurkan data langsung ke kedalaman komponen tanpa prop drilling.
2. **Hanya untuk Data Stabil**: Sangat ideal untuk data berfrekuensi perubahan rendah (Theme, Locale, Auth Session).
3. **Pecah Konteks**: Jangan satukan seluruh state aplikasi dalam satu Context raksasa; gunakan *Split Contexts*.
4. **Hindari Objek Literal Mentah**: Selalu gunakan `useMemo` pada properti `value` Provider untuk menjaga kestabilan referensi memori.
