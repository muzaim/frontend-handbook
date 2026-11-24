# Environment Variables

## 1. Skenario Sistem Nyata

Sebuah tim startup frontend sedang mengintegrasikan gateway pembayaran Stripe ke dalam web aplikasi mereka. Developer membaca dokumentasi Stripe yang menyebutkan perlunya dua buah kunci:
1. `Publishable Key` (kunci publik untuk elemen kartu kredit di frontend).
2. `Secret Key` (kunci rahasia untuk memproses charge uang di backend).

Developer membuat berkas `.env` di root proyek frontend:
```bash
VITE_STRIPE_PUB_KEY=pk_live_12345
VITE_STRIPE_SECRET_KEY=sk_live_99999_RAHASIA_SUPER_FATAL
```

Lalu di dalam komponen checkout, developer menulis:
`const stripeSecret = import.meta.env.VITE_STRIPE_SECRET_KEY;`

Aplikasi di-build dan di-deploy ke production. Dalam waktu 4 jam, saldo rekening Stripe perusahaan terkuras habis hingga minus ratusan juta rupiah. Hacker menemukan kunci `sk_live_...` hanya dengan membuka browser DevTools, masuk ke tab *Sources*, dan menekan `Ctrl+F` mencari kata "sk_live" di dalam berkas bundle JavaScript publik.

Developer keliru mengira bahwa variabel di `.env` aman dan tersembunyi. Insiden fatal ini adalah contoh nyata ketidakpahaman atas cara kerja **Environment Variables di Frontend**.

## 2. Apa Itu dan Buat Apa?

Environment Variables (Variabel Lingkungan) adalah nilai konfigurasi dinamis yang disediakan di luar kode sumber aplikasi untuk mengatur perilaku aplikasi di berbagai lingkungan yang berbeda (Local Development, Staging, Production).

Di sisi backend (Node.js/Go/Python), environment variable tersimpan aman di sistem operasi server dan tidak pernah bisa dilihat oleh pengguna luar.

Namun di sisi **Frontend (SPA / Browser)**:
Browser **TIDAK MEMILIKI AKSES** ke sistem operasi server atau berkas `.env`. Oleh karena itu, Build Tools (Vite, Webpack, Next.js) melakukan proses penggantian teks (*string replacement / text baking*) saat proses build berlangsung. Nilai variabel langsung dicetak (*hardcoded*) ke dalam berkas `.js` publik yang dapat diunduh dan dibaca oleh siapa saja di internet.

## 3. Kenapa Harus Pakai Ini?

Meskipun nilainya bersifat publik, environment variable di frontend tetap sangat dibutuhkan untuk:
- **Konfigurasi Alamat Endpoint API Multi-Lingkungan**: Mengarahkan URL backend ke `http://localhost:8080/api` saat di laptop developer, dan ke `https://api.perusahaan.com` saat di production.
- **Konfigurasi Identitas Layanan Pihak Ketiga Publik**: Menyimpan ID pelacakan Google Analytics (`G-XXXXX`), Sentry DSN, Firebase Public API Key, atau Algolia Search App ID.
- **Mengaktifkan Fitur Eksperimental Bersyarat**: Menyalakan atau mematikan fitur tertentu khusus di lingkungan staging (`VITE_ENABLE_DEBUG_MENU=true`).

## 4. Cara Kerjanya

Berikut adalah diagram alur bagaimana Build Tool mencetak (*bake*) nilai environment variable ke dalam bundle JavaScript statis:

```
[ Berkas Konfigurasi di Laptop Developer: .env.production ]
VITE_API_BASE_URL=https://api.perusahaan.com
VITE_ANALYTICS_ID=UA-99999-1
DB_PASSWORD=rahasia123  <-- TIDAK BERAWALAN VITE_ (Aman, diabaikan bundler)

                               |
                               |-- Developer jalankan: npm run build
                               v
[ ENGINE BUILD TOOL (VITE / ESBUILD / ROLLUP) ]
1. Memindai kode sumber mencari referensi import.meta.env.VITE_*
2. Melakukan String Replacement langsung pada Abstract Syntax Tree (AST)

                               |
                               v
[ HASIL BUNDLE PRODUKSI: dist/assets/index-a1b2c3.js ]
(KODE ASLI DEVELOPER):
  const url = import.meta.env.VITE_API_BASE_URL;

(HASIL DI DALAM FILE JS PUBLIK):
  const url = "https://api.perusahaan.com";

                               |
                               v
[ PENGUNJUNG BUKA WEBSITE DI CHROME ]
Pengunjung mengunduh index-a1b2c3.js
Pengunjung BISA MELIHAT string "https://api.perusahaan.com" lewat DevTools!
```

## 5. Cuplikan Logika Inti

### 1. Prefiks Wajib Pengaman (Prefix Safeguard)

Setiap build tool modern mewajibkan awalan khusus (*prefix*) agar developer tidak sengaja mengekspos variabel rahasia mesin ke bundle publik:
- **Vite**: Wajib diawali `VITE_` (misal: `VITE_API_URL`).
- **Next.js**: Wajib diawali `NEXT_PUBLIC_` (misal: `NEXT_PUBLIC_API_URL`).
- **Create React App / Webpack**: Wajib diawali `REACT_APP_` (misal: `REACT_APP_API_URL`).

Variabel tanpa awalan tersebut akan **diabaikan secara otomatis** oleh bundler dan tidak akan pernah dimasukkan ke dalam kode browser.

### 2. Validasi Tipe Environment Variables (TypeScript + Zod)

Pastikan konfigurasi lingkungan divalidasi saat startup agar aplikasi langsung gagal build (*fail-fast*) jika ada variabel wajib yang lupa diisi:

```typescript
// src/config/env.ts
import { z } from 'zod';

const envSchema = z.object({
  VITE_API_BASE_URL: z.string().url('URL API Backend tidak valid'),
  VITE_APP_ENV: z.enum(['development', 'staging', 'production']),
  VITE_ENABLE_ANALYTICS: z
    .string()
    .transform((val) => val === 'true')
    .default('false'),
});

// Parse import.meta.env secara aman
const parseResult = envSchema.safeParse(import.meta.env);

if (!parseResult.success) {
  console.error('Konfigurasi Environment Tidak Valid:', parseResult.error.format());
  throw new Error('Aplikasi gagal dimuat karena variabel lingkungan tidak lengkap.');
}

export const env = parseResult.data;
```

### 3. Konsumsi di Komponen

```typescript
// src/services/apiClient.ts
import { env } from '../config/env';

export async function fetchUserOrders() {
  // env.VITE_API_BASE_URL dijamin ada dan berupa string URL valid
  const res = await fetch(`${env.VITE_API_BASE_URL}/v1/orders`);
  return res.json();
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

| Kelebihan | Kekurangan / Risiko |
| :--- | :--- |
| Memisahkan kode logika dari konfigurasi lingkungan (sesuai *12-Factor App*) | Nilai variabel dicetak permanen ke dalam file JS publik; sama sekali bukan rahasia |
| Otomatisasi penggantian URL antar staging dan production tanpa ubah baris kode | Perubahan nilai `.env` memerlukan proses re-build dan re-deploy ulang seluruh bundle |
| Mendukung alur kerja lokal yang fleksibel via `.env.local` | Risiko kebocoran kunci rahasia backend jika developer tidak paham konsep client-side bundling |

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai (Batas Keamanan Frontend)

### Yang BOLEH Disimpan di Frontend `.env`
- URL Base API Backend (`https://api.staging.domain.com`).
- Public API Keys yang memang dirancang untuk konsumsi publik (Firebase Public API Key, Stripe Publishable Key, Sentry DSN, Google Maps Public Key).
- Konfigurasi tampilan (Nama aplikasi, Versi rilis, Tema default).

### Yang DILARANG KERAS Disimpan di Frontend `.env`
- **Database Password / Connection String** (PostgreSQL, MongoDB, Redis).
- **Private API Keys** (Stripe Secret Key, Midtrans Server Key, AWS Secret Access Key).
- **JWT Secret Key / Enkripsi Kriptografis**.
- *Aturan Emas*: Jika sebuah kunci bisa digunakan untuk mencuri data dari database atau mendebit uang tanpa izin, **KUNCI TERSEBUT HARUS BERADA DI BACKEND!**

## 8. Metrik Kunci & Observabilitas

1. **Secret Scanning in CI**: Pasang alat pemindai rahasia otomatis (seperti GitGuardian, Gitleaks, atau Trufflehog) di repositori untuk mendeteksi jika ada file `.env` atau kunci privat yang tidak sengaja ter-commit ke Git.
2. **Missing Env Build Failures**: Pastikan script `npm run build` di CI langsung berhenti dengan error non-zero jika variabel wajib di staging/production belum disetel di dashboard CI (GitHub Secrets / Vercel Env).

## 9. Studi Kasus Industri Nyata

Sebuah aplikasi fintech di Asia Tenggara mengintegrasikan layanan SMS OTP pihak ketiga (Twilio) untuk proses verifikasi pendaftaran akun.

Developer frontend menyimpan kredensial autentikasi API Twilio langsung di berkas `.env` aplikasi React:
```bash
REACT_APP_TWILIO_ACCOUNT_SID=AC12345...
REACT_APP_TWILIO_AUTH_TOKEN=abcdef12345...
```
Lalu komponen frontend langsung memanggil endpoint API Twilio untuk mengirimkan SMS ke nomor handphone pengguna.

Akibatnya:
Seorang penyerang mengekstrak `AUTH_TOKEN` tersebut dari bundle JavaScript publik. Penyerang kemudian menggunakan token tersebut untuk menyewa ribuan nomor SMS premium internasional milik sindikat penipu, membebankan tagihan senilai \$45.000 (sekitar Rp 700 juta) ke akun Twilio perusahaan dalam kurun waktu 48 jam.

Solusi arsitektur yang benar:
1. Menghapus seluruh kredensial Twilio dari frontend.
2. Frontend hanya memanggil endpoint internal backend: `POST /api/v1/auth/request-otp`.
3. Server backend (yang memegang kredensial aman di OS environment server) yang bertugas menghubungi Twilio dan menerapkan rate limit ketat per IP address.

## 10. Fakta Menarik & Gotchas Production

- **Hierarki Prioritas File `.env` di Vite**:
  1. `.env.production.local` (prioritas tertinggi, di-ignore git).
  2. `.env.production`.
  3. `.env.local` (selalu di-ignore git).
  4. `.env` (prioritas terendah).
- **Jangan Commit File `.env.local`**: Selalu pastikan `.env.local`, `.env.*.local`, dan `.env` yang memuat kredensial staging terdaftar di dalam berkas `.gitignore`. Repositori hanya boleh memuat berkas `.env.example` yang berisi daftar nama variabel dengan nilai kosong sebagai panduan developer baru.
- **Runtime Injection di Docker**: Jika Anda menyajikan aplikasi React di dalam Docker container via Nginx, variabel lingkungan di-bake saat image di-build (`npm run build`). Jika ingin image yang sama bisa berpindah dari staging ke prod tanpa re-build, gunakan teknik *Runtime Config Injection* (misal: menyuntikkan objek `window.__RUNTIME_CONFIG__` dari script HTML saat container start).

## 11. Rangkuman Singkat

Environment Variables di frontend digunakan untuk konfigurasi multi-lingkungan (seperti URL API dan public keys) yang dicetak (*baked*) langsung ke dalam bundle JavaScript publik saat proses build. Seluruh variabel yang diawali prefiks (`VITE_` / `NEXT_PUBLIC_`) dapat dibaca oleh siapa saja di browser; rahasia privat dan kunci database wajib disimpan secara terisolasi di sisi server backend.
