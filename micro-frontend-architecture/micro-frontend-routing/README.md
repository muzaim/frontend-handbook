# Micro-Frontend Routing

## 1. Skenario Sistem Nyata

Sebuah portal SuperApp korporat menggabungkan tiga Micro-App independen ke dalam satu aplikasi Shell (Host):
1. `Shell (Host)`: Menangani rute `/` dan rute tingkat atas seperti `/dashboard/*`.
2. `MFE Billing`: Menangani sub-rute tagihan: `/dashboard/billing/invoices` dan `/dashboard/billing/payment-methods`.

Kedua tim menggunakan pustaka `react-router-dom` dengan konfigurasi standar `BrowserRouter` (`createBrowserRouter`).

Ketika pengguna membuka `/dashboard/billing/invoices`, lalu mengklik tab "Metode Pembayaran", URL di address bar browser berubah menjadi `/dashboard/billing/payment-methods`. Namun saat pengguna menekan tombol **Back (Kembali)** di browser, keanehan terjadi:
- URL di address bar mundur kembali ke `/dashboard/billing/invoices`, tetapi tampilan layar tetap diam di halaman metode pembayaran.
- Saat tombol Back ditekan sekali lagi, browser mendadak melompat jauh keluar dari aplikasi ke halaman mesin pencari Google!

Desinkronisasi riwayat browser (*History Stack Desynchronization*) ini terjadi karena ada dua router berbasis `BrowserRouter` yang berebut mengendalikan objek global `window.history` pada saat yang sama.

Masalah kritis ini diselesaikan melalui arsitektur **Sinkronisasi Routing Terpadu (Shell & Sub-App Routing Coordination)**.

## 2. Apa Itu dan Buat Apa?

Micro-Frontend Routing adalah strategi perancangan dan penyelarasan navigasi URL antara aplikasi penampung utama (Host / Shell Router) dan aplikasi mikro turunan (Sub-App Routers) agar pengguna merasakan pengalaman navigasi yang mulus layaknya satu aplikasi SPA tunggal.

Tujuan utama Micro-Frontend Routing:
1. **Pemisahan Domain Rute (Path Scoping)**: Setiap Micro-App hanya bertanggung jawab atas segmen URL tertentu (misal: `/shop/*`, `/account/*`, `/support/*`).
2. **Dukungan Tautan Langsung (Deep Linking)**: Pengguna dapat membagikan URL spesifik (misal `https://app.com/shop/item/42`) ke orang lain, dan aplikasi Shell langsung memuat Micro-App yang tepat di halaman yang benar.
3. **Penyelarasan Riwayat Navigasi (History Synchronization)**: Tombol *Back* dan *Forward* browser bekerja secara konsisten tanpa ada lompatan riwayat ganda atau halaman macet.

## 3. Kenapa Harus Pakai Ini? (Dua Pendekatan Utama Routing MFE)

1. **Top-Level Route-Based Splitting (Coarse-Grained)**:
   - Host Router murni memetakan segmen URL awal untuk memutuskan Micro-App mana yang harus di-mount ke layar:
     - `/marketing/*` -> Mount MFE Landing
     - `/catalog/*` -> Mount MFE Produk
     - `/checkout/*` -> Mount MFE Pembayaran
   - Hanya ada 1 Micro-App aktif di viewport pada satu waktu. Ini adalah pola yang paling sederhana, stabil, dan paling banyak diadopsi di industri.
2. **Composite Page Routing (Fine-Grained)**:
   - Beberapa Micro-App berada di satu halaman yang sama secara simultan (misal MFE Navbar, MFE Sidebar, dan MFE Konten).
   - Host Router mengendalikan URL utama, dan sub-aplikasi mendengarkan perubahan parameter query atau event navigasi.

## 4. Cara Kerjanya: Pola MemoryRouter + Browser History Sync

Pola industri standar untuk mencegah perebutan objek `window.history`:
- **Host App**: Menggunakan **BrowserRouter** karena Host adalah satu-satunya entitas yang berhak memodifikasi URL riwayat browser asli.
- **Sub-App (Remote)**: Menggunakan **MemoryRouter** (router yang hidup murni di memori JavaScript tanpa menyentuh address bar), lalu menyinkronkan lokasinya dengan Host melalui callback navigasi.

```
[ PENGGUNA MENEKAN TOMBOL BACK DI BROWSER ]
                     |
                     v
      [ Browser Window: popstate Event ]
                     |
                     v
+-------------------------------------------------------+
|  HOST APPLICATION (BrowserRouter)                     |
|                                                       |
|  1. Mendeteksi perubahan URL: /shop/cart              |
|  2. Beri tahu Sub-App via Callback:                   |
|     onParentNavigate('/shop/cart')                    |
+-------------------------------------------------------+
                     |
                     v
+-------------------------------------------------------+
|  SUB-APP REMOTE (MemoryRouter)                        |
|                                                       |
|  3. memoryHistory.push('/cart')                       |
|  4. Sub-App merender komponen halaman keranjang       |
|                                                       |
|  (Jika user mengklik link di dalam Sub-App):          |
|  5. onChildNavigate('/shop/checkout')                 |
|     -> Beri tahu Host untuk browserHistory.push()     |
+-------------------------------------------------------+
```

## 5. Cuplikan Logika Inti

### 1. Sisi Sub-App Remote (Mengekspos Fungsi Mount Mandiri)

Sub-App tidak langsung merender dirinya ke elemen DOM root window, melainkan mengekspos fungsi `mount` yang menerima router callback dari Host:

```tsx
// remote-shop/src/bootstrap.tsx
import React from 'react';
import ReactDOM from 'react-dom/client';
import { RouterProvider, createMemoryRouter } from 'react-router-dom';
import { shopRoutes } from './routes';

interface MountOptions {
  initialPath?: string;
  onNavigate?: (path: string) => void;
}

export function mount(el: HTMLElement, { initialPath = '/', onNavigate }: MountOptions) {
  // Gunakan MemoryRouter terisolasi
  const router = createMemoryRouter(shopRoutes, {
    initialEntries: [initialPath],
  });

  // Jika user bernavigasi di dalam Sub-App, kabari Host agar URL browser terupdate
  if (onNavigate) {
    router.subscribe((state) => {
      onNavigate(state.location.pathname);
    });
  }

  const root = ReactDOM.createRoot(el);
  root.render(<RouterProvider router={router} />);

  // Kembalikan listener untuk menerima navigasi dari Host (tombol Back browser)
  return {
    onParentNavigate(nextPathname: string) {
      if (router.state.location.pathname !== nextPathname) {
        router.navigate(nextPathname);
      }
    },
    unmount() {
      root.unmount();
    },
  };
}
```

### 2. Sisi Host App (Menghubungkan Browser History ke Sub-App)

```tsx
// host-app/src/components/ShopMicroAppWrapper.tsx
import React, { useEffect, useRef } from 'react';
import { useNavigate, useLocation } from 'react-router-dom';
// Impor fungsi mount dari container Module Federation
// @ts-ignore
import { mount } from 'mfe_shop/ShopApp';

export function ShopMicroAppWrapper() {
  const containerRef = useRef<HTMLDivElement>(null);
  const navigate = useNavigate();
  const location = useLocation();

  useEffect(() => {
    if (!containerRef.current) return;

    // Inisialisasi Sub-App dengan menyambungkan kedua router
    const { onParentNavigate, unmount } = mount(containerRef.current, {
      initialPath: location.pathname.replace('/shop', '') || '/',
      onNavigate: (subPath: string) => {
        const fullPath = `/shop${subPath === '/' ? '' : subPath}`;
        if (location.pathname !== fullPath) {
          navigate(fullPath); // Sinkronkan ke URL browser asli
        }
      },
    });

    return () => {
      unmount();
    };
  }, []);

  return <div ref={containerRef} id="mfe-shop-container" />;
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

| Kelebihan | Kekurangan / Risiko |
| :--- | :--- |
| Tombol Back/Forward browser berfungsi 100% normal dan konsisten bagi pengguna | Memerlukan adapter boilerplate sinkronisasi router di sisi Host dan Remote |
| Mendukung deep linking penuh (pengguna bisa membagikan URL spesifik halaman dalam) | Kedua tim harus menyepakati aturan konvensi path prefix (misal awalan `/shop/`) |
| Setiap Micro-App tetap bisa dijalankan dan dites secara mandiri dalam mode standalone | Sedikit overhead memori untuk memelihara dua router state (Host dan Remote) |

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Menggunakan Pola Sinkronisasi MemoryRouter
- Aplikasi Micro Frontends di mana sub-aplikasi memiliki alur navigasi multi-halaman internal yang kompleks (misal alur pendaftaran 5 langkah, pengaturan profil multi-tab).
- Ketika sub-aplikasi harus bisa berjalan ganda: bisa jalan mandiri (*standalone dev*) via `BrowserRouter` dan bisa berjalan di dalam Host via `MemoryRouter`.

### Kapan Cukup Menggunakan Flat Routing Biasa
- Jika Micro-App murni berupa widget tunggal tanpa rute internal (seperti widget cuaca, banner promo, atau tombol checkout tunggal).

## 8. Metrik Kunci & Observabilitas

1. **Route Transition Latency**: Waktu yang dibutuhkan dari saat user mengklik link navigasi hingga komponen remote Micro-App selesai di-mount. Pantau agar tidak melebihi 200 milidetik.
2. **404 Route Mismatch Rate**: Persentase navigasi pengguna yang tersesat di halaman 404 akibat perbedaan penulisan prefix path antar tim.
3. **Deep-Link Success Ratio**: Verifikasi apakah URL deep link yang dibagikan pengguna berhasil me-render state halaman yang tepat saat dimuat langsung via cold-start.

## 9. Studi Kasus Industri Nyata

Sebuah platform SaaS manajemen HR enterprise menggabungkan modul Penggajian (Payroll) dan modul Absensi (Attendance) ke dalam Shell portal.

Kedua modul diprogram menggunakan `createBrowserRouter` langsung ke objek `window.history`. Masalah muncul saat pengguna membuka slip gaji di rute `/payroll/slips/2026-11` lalu mengklik tombol navigasi ke modul Absensi `/attendance/calendar`.

Browser memuat modul baru, namun saat pengguna menekan tombol Back, terjadi loop navigasi tak berujung: kedua router saling berebut memanggil `history.pushState()`, menghasilkan 15 entri URL palsu di riwayat browser dalam hitungan detik. Pengguna tidak bisa kembali ke halaman dashboard.

Solusi engineering:
1. Merombak seluruh Micro-App sub-modul agar menggunakan **MemoryRouter** yang terisolasi dari window.
2. Host App mengambil kendali mutlak sebagai *Single Source of Truth* untuk riwayat browser asli.
3. Menetapkan konvensi isolasi prefix path yang ketat di API gateway dan router.
4. Masalah loop navigasi riwayat browser teratasi sepenuhnya di seluruh 12 modul HR.

## 10. Fakta Menarik & Gotchas Production

- **Trailing Slash Trap**: Ketidakkonsistenan penulisan garis miring akhir (misal `/shop` vs `/shop/`) sering memicu re-render ganda pada router. Selalu buat fungsi normalisasi path di level Host.
- **Scroll Restoration**: Saat berpindah antar Micro-App, browser sering kali mempertahankan posisi scroll halaman sebelumnya. Pastikan Host App memanggil `window.scrollTo(0, 0)` saat transisi rute lintas Micro-App terjadi.
- **Framework Agnostic Shell**: Jika Host dibuat dengan Vue dan Remote dibuat dengan React, Anda tetap bisa menyinkronkan rute menggunakan native browser `popstate` event dan CustomEvents tanpa perlu memaksakan kesamaan library router.

## 11. Rangkuman Singkat

Micro-Frontend Routing menuntut koordinasi ketat antara Host Router (penguasa `BrowserRouter` browser) dan Sub-App Router (menggunakan `MemoryRouter` terisolasi). Melalui mekanisme sinkronisasi dua arah via callback, aplikasi gabungan menjamin navigasi tombol Back/Forward yang stabil dan kemampuan deep linking yang andal.
