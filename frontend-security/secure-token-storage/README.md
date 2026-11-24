# Secure Token Storage

## 1. Skenario Sistem Nyata

Sebuah startup B2B SaaS menyimpan Access Token dan Refresh Token JWT di dalam `localStorage` browser agar status login pengguna tetap tersimpan meski tab ditutup.

Suatu ketika, salah satu dependency npm open-source yang digunakan aplikasi disusupi skrip berbahaya (Supply Chain Malware). Skrip tersebut menjalankan:
`window.postMessage({ stolen: localStorage }, '*');`

Akibatnya, seluruh token dari ribuan enterprise customer dicuri secara massal. Manajemen perusahaan panik dan meminta audit menyeluruh untuk menentukan di mana sebenarnya tempat penyimpanan token yang paling aman di browser modern.

## 2. Apa Itu dan Buat Apa?

Secure Token Storage adalah penentuan arsitektur dan media penyimpanan kredensial otentikasi (Access Token, Refresh Token, Session ID, API Key) di sisi browser client dengan meminimalkan permukaan serangan (*attack surface*).

Tujuan utamanya adalah menyeimbangkan antara dua risiko keamanan terbesar di web:
1. **Risiko Pencurian Skrip (XSS - Cross-Site Scripting)**
2. **Risiko Pembajakan Request Antar Situs (CSRF - Cross-Site Request Forgery)**

Memilih tempat penyimpanan yang salah dapat membuat aplikasi rentan terhadap kebocoran kredensial tanpa disadari oleh pengguna.

## 3. Kenapa Harus Pakai Ini? (Perbandingan Opsi Storage Browser)

| Lokasi Penyimpanan | Akses JavaScript? | Kebal XSS? | Kebal CSRF? | Persistensi Setelah Tab Ditutup? |
| :--- | :--- | :--- | :--- | :--- |
| **LocalStorage** | Ya (`window.localStorage`) | Tidak (Sangat Rentan) | Ya (Secara Alami) | Ya (Permanen sampai dihapus) |
| **SessionStorage** | Ya (`window.sessionStorage`) | Tidak (Sangat Rentan) | Ya (Secara Alami) | Tidak (Hilang saat tab ditutup) |
| **In-Memory (JS Variable)** | Ya (Hanya di scope kode) | Sebagian (Tergantung XSS) | Ya (Secara Alami) | Tidak (Hilang saat reload halaman) |
| **HttpOnly Cookie** | **TIDAK BISA** (Diblokir browser) | **YA (KEBAL TOTAL)** | Rentan jika tanpa `SameSite` | Ya (Sesuai `Max-Age` / `Expires`) |
| **Web Worker Storage** | Ya (Hanya di thread worker) | Tinggi (Isolasi context) | Ya (Secara Alami) | Tidak (Hilang saat reload halaman) |

## 4. Cara Kerjanya

Pola standar industri yang direkomendasikan oleh OWASP dan tim sekuriti modern adalah **Hybrid Storage (In-Memory Access Token + HttpOnly Refresh Cookie)**:

```
                  +-------------------------------------------------------------+
                  |                       BROWSER CLIENT                        |
                  |                                                             |
                  |   +-----------------------+     +-----------------------+   |
                  |   |    JavaScript RAM     |     |   Browser Encrypted   |   |
                  |   |   (In-Memory State)   |     |     Cookie Jar        |   |
                  |   +-----------------------+     +-----------------------+   |
                  |   | Access Token (10m)    |     | Refresh Token (7d)    |   |
                  |   | Digunakan untuk API   |     | HttpOnly, Secure,     |   |
                  |   | calls via header      |     | SameSite=Strict       |   |
                  |   |                       |     | JS TIDAK BISA AKSES   |   |
                  |   +-----------------------+     +-----------------------+   |
                  +---------------|-----------------------------|---------------+
                                  |                             |
                                  |                             |
                      1. Request Data API             2. Refresh Token Saat Expired
                      Header: Bearer <Token>          POST /api/auth/refresh
                                  |                   Cookie: refreshToken (Otomatis)
                                  |                             |
                                  v                             v
                  +-------------------------------------------------------------+
                  |                     BACKEND API GATEWAY                     |
                  +-------------------------------------------------------------+
```

## 5. Cuplikan Logika Inti

### Client-side: In-Memory Token Manager (TypeScript)

Menyimpan Access Token di dalam closure JavaScript murni (bukan di `window` global):

```typescript
// frontend/src/auth/tokenManager.ts

// Variabel ini berada di dalam module scope terisolasi (tidak menempel di window)
let accessToken: string | null = null;
let refreshPromise: Promise<string> | null = null;

export const tokenManager = {
  getToken(): string | null {
    return accessToken;
  },

  setToken(token: string | null): void {
    accessToken = token;
  },

  clearToken(): void {
    accessToken = null;
  },

  // Logika silent refresh untuk meminta access token baru saat halaman di-reload
  async refreshAccessToken(): Promise<string> {
    // Hindari duplikasi request refresh jika banyak komponen memanggil bersamaan
    if (refreshPromise) {
      return refreshPromise;
    }

    refreshPromise = (async () => {
      try {
        const res = await fetch('/api/auth/refresh', {
          method: 'POST',
          credentials: 'include', // Mengirimkan HttpOnly refresh cookie otomatis
        });

        if (!res.ok) {
          throw new Error('Sesi kedaluwarsa');
        }

        const data = await res.json();
        this.setToken(data.accessToken);
        return data.accessToken;
      } finally {
        refreshPromise = null;
      }
    })();

    return refreshPromise;
  },
};
```

### Konfigurasi Backend: Menerbitkan Cookie yang Aman (Express)

```typescript
// backend/src/routes/auth.ts
import { Response } from 'express';

export function sendAuthCookies(res: Response, refreshToken: string) {
  res.cookie('refreshToken', refreshToken, {
    httpOnly: true,              // Kunci utama: JS di browser tidak bisa mencuri token ini
    secure: true,                // Hanya dikirim lewat koneksi HTTPS
    sameSite: 'strict',          // Pertahanan mutlak terhadap serangan CSRF
    path: '/api/auth/refresh',   // Batasi pengiriman cookie HANYA ke endpoint refresh
    maxAge: 7 * 24 * 60 * 60 * 1000, // 7 hari
  });
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

| Pendekatan | Kelebihan | Kekurangan / Risiko |
| :--- | :--- | :--- |
| **LocalStorage** | Sangat mudah diimplementasikan, token tetap ada setelah reload | Celah keamanan kritis: Satu bug XSS bisa menguras seluruh token user |
| **HttpOnly Cookie Penuh** | Kebal 100% dari pencurian token script XSS | Membutuhkan perlindungan CSRF (`SameSite` + Anti-CSRF header) dan konfigurasi CORS teliti |
| **In-Memory + Refresh Cookie** | Kombinasi terbaik: Access token tidak tersimpan di storage permanen, refresh aman dari XSS | Pengguna mengalami sedikit delay (silent refresh) saat pertama kali membuka halaman baru |

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Menggunakan In-Memory + HttpOnly Cookie
- Aplikasi perbankan, finansial, e-commerce, atau dashboard korporat yang memproses data sensitif.
- Aplikasi web production yang ingin mematuhi standar sertifikasi keamanan (ISO 27001, SOC2, PCI-DSS).

### Kapan Boleh Menggunakan LocalStorage
- Aplikasi prototipe, hackathon, atau aplikasi edukasi internal yang tidak memproses data pribadi/finansial sama sekali.
- Data preferensi non-sensitif (misal: pilihan dark mode, ukuran font, filter tabel terakhir).

## 8. Metrik Kunci & Observabilitas

1. **Storage Access Audits**: Gunakan audit DevTools Application panel untuk memastikan tidak ada token sensitif yang bocor di `localStorage` atau `sessionStorage`.
2. **Refresh Failure Ratio**: Persentase kegagalan silent refresh token. Jika tinggi pada user aktif, periksa masalah izin third-party cookies atau konfigurasi `SameSite`.
3. **Session Replay Masking**: Pastikan tool session recording (seperti LogRocket atau FullStory) secara ketat mem-blacklist elemen form dan memori auth agar token tidak terekam ke server pihak ketiga.

## 9. Studi Kasus Industri Nyata

Sebuah platform dompet digital kripto berbasis web awalnya menyimpan kunci privat terenkripsi dan token login di `localStorage`.

Seorang penyerang berhasil menemukan celah XSS DOM-based pada fitur widget chat bantuan pengguna. Penyerang menyuntikkan skrip yang mengeksekusi `JSON.stringify(localStorage)` dan mengirimkannya ke webhook publik. Ratusan akun pengguna terkuras saldonya dalam hitungan menit.

Pasca insiden, tim security melakukan perombakan menyeluruh:
1. Menghapus seluruh token dan kunci dari `localStorage`.
2. Memindahkan otentikasi ke arsitektur `BFF (Backend for Frontend)` di mana session disimpan dalam HttpOnly, Secure, SameSite=Strict cookie yang ditangani langsung oleh server edge Next.js.
3. Kunci privat kripto tidak pernah lagi disimpan di browser, melainkan menggunakan mekanisme Hardware Signer via WebAuthn/FIDO2.

## 10. Fakta Menarik & Gotchas Production

- **Mitos Enkripsi di LocalStorage**: Mengenkrpsi token di `localStorage` menggunakan secret key di JavaScript adalah hal yang sia-sia, karena penyerang yang memiliki akses XSS juga bisa membaca secret key tersebut dari memori JavaScript yang sama.
- **Path Scoping pada Cookie**: Anda bisa memperketat keamanan refresh token cookie dengan menambahkan direktif `Path=/api/auth/refresh`. Dengan cara ini, browser tidak akan mengirimkan cookie refresh pada setiap kali memanggil endpoint gambar atau REST API biasa lainnya.
- **Web Workers sebagai Vault**: Beberapa arsitektur tingkat lanjut menyimpan token dan melakukan panggilan fetch di dalam Web Worker terpisah. Karena Web Worker tidak memiliki akses ke DOM, skrip XSS di halaman utama tidak bisa langsung mengakses token tersebut.

## 11. Rangkuman Singkat

Menyimpan token otentikasi di `localStorage` adalah praktik berisiko tinggi terhadap serangan XSS. Standar industri yang paling aman adalah menyimpan Access Token berumur pendek di memori JavaScript (*in-memory*), dipadukan dengan Refresh Token berumur panjang di dalam Cookie berbendera `HttpOnly`, `Secure`, dan `SameSite=Strict`.
