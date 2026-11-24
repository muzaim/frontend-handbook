# Component Testing

## 1. Skenario Sistem Nyata

Tim frontend sedang membangun komponen tombol pemroses pembayaran: `<PaymentButton amount={50000} onConfirm={handlePay} />`. Komponen ini memiliki beberapa state internal yang rumit:
1. Menampilkan teks nominal uang yang diformat.
2. Ketika diklik, tombol berubah status menjadi `loading` (spinner berputar) dan properti `disabled` aktif untuk mencegah double-click oleh pengguna yang tidak sabaran.
3. Jika proses pembayaran gagal, tombol menampilkan status error berwarna merah dengan pesan gagal dan tombol coba lagi.

Seorang developer junior melakukan refactor pada struktur CSS dan mengganti elemen `<button>` dengan elemen `<div>` ber-event `onClick` agar lebih mudah di-styling dengan flexbox.

Akibatnya fatal: pengguna yang menavigasi website menggunakan tombol keyboard (Tab dan Enter) tidak bisa lagi menekan tombol tersebut, dan status `disabled` menjadi tidak berfungsi sehingga user bisa mengklik berkali-kali saat loading.

Masalah aksesibilitas dan perilaku state komponen ini dapat dicegah sejak awal menggunakan **Component Testing** berbasis React Testing Library.

## 2. Apa Itu dan Buat Apa?

Component Testing adalah metodologi pengujian yang berfokus pada pengujian komponen antarmuka pengguna (UI component) secara mandiri untuk memverifikasi fungsionalitas rendering, penanganan input props, interaksi event pengguna, serta aksesibilitas elemen.

Filosofi utama component testing modern (yang dipopulerkan oleh React Testing Library):
*"Semakin mirip pengujian Anda dengan cara perangkat lunak digunakan oleh pengguna nyata, semakin besar tingkat kepercayaan yang dapat diberikannya."*

Component testing tidak menguji detail implementasi internal (seperti nama variabel state `const [isLoading, setIsLoading] = useState()`), melainkan menguji perilaku yang terlihat oleh pengguna: apakah tombol muncul, apakah teks berubah saat diklik, dan apakah event callback dipanggil dengan argumen yang benar.

## 3. Kenapa Harus Pakai Ini?

- **Memverifikasi State dan Props Secara Cepat**: Anda dapat dengan mudah menguji variasi tampilan komponen (kondisi default, loading, disabled, error, varian warna) tanpa perlu memuat seluruh halaman website atau backend nyata.
- **Menegakkan Aksesibilitas (a11y-First Queries)**: Pengujian komponen mendorong penggunaan query berbasis role aksesibilitas (seperti `getByRole('button')` atau `getByLabelText('Email')`), memastikan bahwa komponen yang Anda buat ramah bagi pengguna pembaca layar (*screen readers*).
- **Mencegah Kerusakan Akibat Refactor**: Anda bebas merombak kode internal komponen (misal dari Class Component ke Functional Hooks, atau mengganti pustaka styling) selama perilaku luar yang dirasakan pengguna tidak berubah, tes akan tetap lulus.

## 4. Cara Kerjanya

Berikut adalah diagram alur bagaimana runner pengujian komponen merender dan memvalidasi interaksi komponen di lingkungan virtual DOM (jsdom) atau browser sandbox:

```
[ Test Runner: Vitest + React Testing Library ]
                       |
                       |-- 1. render(<PaymentButton amount={50000} onConfirm={spyFn} />)
                       v
[ Virtual DOM Sandbox (jsdom) ]
  <button type="button" aria-busy="false">Bayar Rp 50.000</button>
                       |
                       |-- 2. Query elemen via Accessible Role
                       |      const btn = screen.getByRole('button', { name: /Bayar Rp 50.000/i });
                       |
                       |-- 3. Simulasi Interaksi Pengguna Asli
                       |      await userEvent.click(btn);
                       v
[ Komponen Memperbarui State Internal ]
  <button type="button" aria-busy="true" disabled>Memproses...</button>
                       |
                       |-- 4. Assertions (Ekspektasi Perilaku):
                       |      - expect(spyFn).toHaveBeenCalledTimes(1);
                       |      - expect(btn).toBeDisabled();
                       |      - expect(btn).toHaveAttribute('aria-busy', 'true');
                       v
                 [ STATUS: PASS ]
```

## 5. Cuplikan Logika Inti

### Komponen yang Diuji (Accessible Payment Button)

```tsx
// src/components/PaymentButton.tsx
import React, { useState } from 'react';

interface PaymentButtonProps {
  amount: number;
  onConfirm: () => Promise<void>;
}

export function PaymentButton({ amount, onConfirm }: PaymentButtonProps) {
  const [status, setStatus] = useState<'idle' | 'loading' | 'error'>('idle');

  const handleClick = async () => {
    if (status === 'loading') return;
    setStatus('loading');
    try {
      await onConfirm();
      setStatus('idle');
    } catch {
      setStatus('error');
    }
  };

  return (
    <div>
      <button
        type="button"
        onClick={handleClick}
        disabled={status === 'loading'}
        aria-busy={status === 'loading'}
        className="px-4 py-2 bg-blue-600 text-white rounded disabled:opacity-50"
      >
        {status === 'loading' ? 'Memproses Transaksi...' : `Bayar Rp ${amount.toLocaleString('id-ID')}`}
      </button>
      {status === 'error' && (
        <p role="alert" className="text-red-500 mt-2">
          Pembayaran gagal diproses. Silakan coba lagi.
        </p>
      )}
    </div>
  );
}
```

### File Pengujian Komponen (Vitest + React Testing Library)

```tsx
// src/components/PaymentButton.test.tsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { describe, it, expect, vi } from 'vitest';
import { PaymentButton } from './PaymentButton';

describe('<PaymentButton />', () => {
  it('harus merender teks nominal uang yang diformat dengan benar', () => {
    render(<PaymentButton amount={75000} onConfirm={vi.fn()} />);

    // Menggunakan getByRole untuk memastikan elemen tersebut adalah tombol interaktif sah
    const button = screen.getByRole('button', { name: /Bayar Rp 75\.000/i });
    expect(button).toBeInTheDocument();
    expect(button).toBeEnabled();
  });

  it('harus memanggil onConfirm, menonaktifkan tombol, dan menampilkan status loading saat diklik', async () => {
    const user = userEvent.setup();
    // Buat promise yang belum di-resolve untuk memeriksa status loading
    let resolvePromise: () => void;
    const mockConfirm = vi.fn().mockImplementation(() => {
      return new Promise<void>((resolve) => {
        resolvePromise = resolve;
      });
    });

    render(<PaymentButton amount={50000} onConfirm={mockConfirm} />);
    const button = screen.getByRole('button', { name: /Bayar Rp 50\.000/i });

    // Klik tombol
    await user.click(button);

    // Verifikasi callback dipanggil tepat 1 kali
    expect(mockConfirm).toHaveBeenCalledTimes(1);

    // Tombol harus disabled dan teks berubah menjadi loading
    expect(button).toBeDisabled();
    expect(button).toHaveTextContent('Memproses Transaksi...');
    expect(button).toHaveAttribute('aria-busy', 'true');

    // Selesaikan proses asinkron
    resolvePromise!();
  });

  it('harus menampilkan pesan error saat callback pembayaran melempar error', async () => {
    const user = userEvent.setup();
    const mockFailedConfirm = vi.fn().mockRejectedValue(new Error('Network error'));

    render(<PaymentButton amount={50000} onConfirm={mockFailedConfirm} />);
    const button = screen.getByRole('button');

    await user.click(button);

    // Verifikasi elemen error terender dengan atribut role="alert"
    const alertMessage = await screen.findByRole('alert');
    expect(alertMessage).toHaveTextContent(/Pembayaran gagal diproses/i);
  });
});
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

| Kelebihan | Kekurangan / Risiko |
| :--- | :--- |
| Memvalidasi interaksi UI dan aksesibilitas komponen secara cepat di lingkungan virtual | Tidak menguji kompatibilitas visual layout CSS tingkat piksel browser asli |
| Refactor-resilient: tes tidak rusak hanya karena perubahan struktur internal kode | Snapshot testing yang berlebihan dapat menghasilkan *false alarms* yang diabaikan developer |
| Menumbuhkan kebiasaan menulis kode HTML semantik yang ramah screen reader | Perlu menyediakan mock props atau context wrapper (tema, router) secara mandiri |

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Menggunakan Component Testing
- Komponen Design System / UI Library (Button, Modal Dialog, Accordion, Dropdown Menu, Data Table, Date Picker).
- Komponen formulir interaktif dengan validasi lokal instan (Input password dengan meteran kekuatan, checkbox persetujuan).
- Komponen yang memiliki banyak variasi state bersyarat (*conditional states*).

### Kapan Menggunakan Level Lain
- Jika komponen murni hanya memetakan layout visual statis tanpa logika interaksi (lebih cocok menggunakan alat visual regression seperti Storybook + Chromatic).
- Jika ingin menguji bagaimana 5 komponen berkolaborasi dengan backend API nyata (gunakan **Integration Testing** atau **E2E Testing**).

## 8. Metrik Kunci & Observabilitas

1. **Accessible Query Ratio**: Persentase query pengujian yang menggunakan query tingkat aksesibilitas (`getByRole`, `getByLabelText`) dibandingkan query selector CSS rapuh (`querySelector('.btn-blue')`). Targetkan > 90%.
2. **Component Test Execution Time**: Seluruh suite tes komponen UI harus selesai dieksekusi dalam hitungan beberapa detik di lingkungan CI.
3. **Flakiness Zero Guarantee**: Tes komponen di lingkungan jsdom harus sepenuhnya deterministik tanpa ada ketergantungan pada delay waktu jaringan riil.

## 9. Studi Kasus Industri Nyata

Sebuah platform e-commerce travel memiliki komponen `<FlightDateRangePicker />`. Komponen ini memiliki logika kalender rumit: memilih tanggal berangkat, tanggal pulang, memblokir tanggal masa lalu, dan menandai tanggal hari libur nasional.

Awalnya, tim hanya mengandalkan E2E test Playwright untuk menguji kalender ini di halaman pencarian tiket. Namun setiap kali pengembang mengubah styling kalender, E2E test di CI memakan waktu 8 menit dan sering gagal karena masalah timeout loading font.

Solusi engineering:
1. Memindahkan pengujian logika kalender ke **Component Testing** terisolasi menggunakan React Testing Library.
2. Menguji 15 variasi skenario tanggal secara terisolasi (misal: memilih tanggal pulang sebelum tanggal pergi, memilih rentang lebih dari 30 hari).
3. Suite 15 tes komponen kalender ini selesai dieksekusi hanya dalam 1,2 detik di Vitest.
4. E2E test dikurangi cukup hanya menguji 1 alur pencarian standar, memangkas durasi pipeline CI sebesar 40%.

## 10. Fakta Menarik & Gotchas Production

- **Hirarki Query Terbaik**: Urutan prioritas query yang disarankan W3C dan React Testing Library:
  1. `getByRole` (paling direkomendasikan karena mencerminkan cara kerja screen reader).
  2. `getByLabelText` (sangat bagus untuk input formulir).
  3. `getByPlaceholderText` / `getByText`.
  4. `getByTestId` (gunakan sebagai jalan terakhir jika elemen tidak memiliki peran semantik yang jelas).
- **Bahaya Snapshot Testing**: Snapshot testing (`expect(tree).toMatchSnapshot()`) sering kali menjadi jebakan. Developer sering menekan `u` (*update snapshot*) tanpa membaca perubahannya secara teliti saat ada perubahan markup kecil, membuat tes kehilangan fungsinya. Uji perilaku yang spesifik alih-alih seluruh pohon HTML.
- **`userEvent` vs `fireEvent`**: Selalu pilih library `@testing-library/user-event` daripada `fireEvent`. `userEvent` mensimulasikan seluruh rangkaian interaksi browser asli (hover -> mousedown -> focus -> mouseup -> click), sedangkan `fireEvent` hanya memicu event sintetis tanpa transisi browser.

## 11. Rangkuman Singkat

Component Testing memverifikasi bahwa komponen antarmuka pengguna merender props dengan benar, menangani aksi interaksi pengguna secara presisi, dan mematuhi standar aksesibilitas web. Dengan fokus pada perilaku eksternal dan query berbasis role, tes komponen tetap kokoh (*resilient*) saat kode internal dirombak.
