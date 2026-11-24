# CORS (Cross-Origin Resource Sharing)

## 1. Skenario Sistem Nyata

Aplikasi dashboard internal frontend di-host di `https://admin.perusahaan.com`. Frontend ini perlu memanggil REST API backend pembayaran yang berada di domain berbeda, yaitu `https://api-payment.perusahaan.com`.

Ketika developer frontend menjalankan perintah `fetch('https://api-payment.perusahaan.com/v1/payouts', { method: 'POST', ... })`, request tiba-tiba diblokir oleh browser dengan pesan error di console:
`Access to fetch at 'https://api-payment.perusahaan.com/v1/payouts' from origin 'https://admin.perusahaan.com' has been blocked by CORS policy: No 'Access-Control-Allow-Origin' header is present on the requested resource.`

Anehnya, jika endpoint yang sama dipanggil via Postman atau terminal `curl`, request berhasil sukses 200 OK. Developer junior bingung dan sering mengira ada bug di browser atau mencoba "mematikan CORS" di frontend.

## 2. Apa Itu dan Buat Apa?

CORS (Cross-Origin Resource Sharing) adalah mekanisme keamanan berbasis HTTP header yang diterapkan oleh browser untuk mengatur apakah aplikasi web yang berjalan di satu *origin* diizinkan meminta dan membaca resource dari *origin* yang berbeda.

Origin ditentukan oleh kombinasi tiga elemen: **Protokol** (http/https), **Host/Domain** (admin.perusahaan.com), dan **Port** (80, 443, 3000). Jika salah satu berbeda, request dikategorikan sebagai *cross-origin*.

CORS bukan mekanisme keamanan di sisi server untuk melindungi database dari hacker, melainkan kebijakan perlindungan browser (Same-Origin Policy enforcement) untuk melindungi pengguna browser agar website jahat tidak bisa secara diam-diam mencuri data sensitif dari API target menggunakan session user.

## 3. Kenapa Harus Pakai Ini?

Tanpa aturan pembatasan origin (Same-Origin Policy dan CORS):
- Jika Anda sedang login di `https://bank.com`, lalu membuka tab baru `https://situs-jahat.com`.
- Skrip di `situs-jahat.com` bisa mengeksekusi `fetch('https://bank.com/api/saldo')`.
- Browser otomatis mengirim cookie session Anda ke `bank.com`, lalu skrip situs jahat dapat membaca seluruh isi saldo dan riwayat transaksi rekening Anda.

Dengan SOP dan CORS:
- Browser mengizinkan request terkirim jika simple request, namun **melarang keras** skrip JavaScript di `situs-jahat.com` membaca respon data tersebut kecuali backend `bank.com` secara eksplisit menyertakan header izin `Access-Control-Allow-Origin: https://situs-jahat.com`.
- Untuk request non-simple (seperti custom headers, JSON body), browser bahkan mengirimkan request pengecekan awal (*Preflight Request*) sebelum request asli diizinkan dikirim.

## 4. Cara Kerjanya

Berikut adalah diagram alur Preflight Request (`OPTIONS`) dan respon CORS pada HTTP cross-origin:

```
[ Frontend: admin.perusahaan.com ]              [ Backend: api-payment.perusahaan.com ]
               |                                                   |
               |--- 1. Preflight OPTIONS /v1/payouts ------------->|
               |    Origin: https://admin.perusahaan.com           |
               |    Access-Control-Request-Method: POST            |
               |    Access-Control-Request-Headers: Content-Type   |
               |                                                   |
               |<-- 2. HTTP 204 No Content ------------------------|
               |    Access-Control-Allow-Origin:                   |
               |      https://admin.perusahaan.com                 |
               |    Access-Control-Allow-Methods: POST, GET, OPTIONS|
               |    Access-Control-Allow-Headers: Content-Type     |
               |    Access-Control-Max-Age: 86400                  |
               |                                                   |
 (Browser memvalidasi header izin)                                 |
               |                                                   |
               |--- 3. Actual Request: POST /v1/payouts ---------->|
               |    Origin: https://admin.perusahaan.com           |
               |    Content-Type: application/json                 |
               |    Body: {"amount": 500000}                       |
               |                                                   |
               |<-- 4. HTTP 200 OK --------------------------------|
               |    Access-Control-Allow-Origin:                   |
               |      https://admin.perusahaan.com                 |
               |    Data: {"status": "success"}                    |
               v                                                   v
```

Jika server tidak mengembalikan `Access-Control-Allow-Origin` yang cocok, browser memutus koneksi dan JavaScript menerima `TypeError: Failed to fetch`.

## 5. Cuplikan Logika Inti

### Client-side (Fetch API dengan Credentials)

Ketika request cross-origin membutuhkan cookie atau credential auth, frontend harus menyetel opsi `credentials: 'include'`:

```typescript
// frontend/src/services/paymentApi.ts
interface PayoutPayload {
  recipientId: string;
  amount: number;
}

export async function submitPayout(payload: PayoutPayload): Promise<void> {
  const response = await fetch('https://api-payment.perusahaan.com/v1/payouts', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'X-Client-Version': '2.4.0',
    },
    // Wajib disetel agar cookie session cross-origin dikirimkan browser
    credentials: 'include',
    body: JSON.stringify(payload),
  });

  if (!response.ok) {
    throw new Error(`HTTP Error: ${response.status}`);
  }

  const result = await response.json();
  return result;
}
```

### Konfigurasi Backend (Node.js Express CORS Middleware)

Solusi CORS yang benar selalu berada di konfigurasi server backend, bukan mematikan browser security:

```typescript
// backend/src/server.ts
import express from 'express';
import cors from 'cors';

const app = express();

const allowedOrigins = [
  'https://admin.perusahaan.com',
  'https://staging-admin.perusahaan.com',
];

app.use(
  cors({
    origin: (origin, callback) => {
      // Izinkan tools server-to-server atau periksa whitelist
      if (!origin || allowedOrigins.includes(origin)) {
        callback(null, true);
      } else {
        callback(new Error('Akses diblokir oleh kebijakan CORS'));
      }
    },
    methods: ['GET', 'POST', 'PUT', 'DELETE', 'OPTIONS'],
    allowedHeaders: ['Content-Type', 'Authorization', 'X-Client-Version'],
    credentials: true, // Wajib bernilai true jika frontend mengirim cookie/auth credentials
    maxAge: 86400,     // Cache respon preflight OPTIONS selama 24 jam di browser
  })
);
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

| Kelebihan | Kekurangan / Risiko |
| :--- | :--- |
| Mencegah pencurian data browser antar origin secara otomatis oleh browser | Menambah latency awal karena preflight request `OPTIONS` sebelum request utama |
| Kontrol granular origin mana saja yang boleh mengakses endpoint tertentu | Konfigurasi sering salah kaprah menggunakan wildcard `*` bersamaan dengan `credentials: true` |
| Mengizinkan arsitektur microservices terdistribusi antar subdomain terpisah | Pesan error browser sengaja dibuat generik demi privasi, sehingga sering membingungkan developer junior |

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Dikonfigurasi
- Domain frontend dan backend berbeda (misal: frontend di `app.domain.com` dan backend di `api.domain.com`).
- Public API yang sengaja dibuka untuk diakses oleh website publik manapun (`Access-Control-Allow-Origin: *`).
- Frontend lokal saat masa development (`localhost:3000`) memanggil API backend di staging (`https://api-staging.domain.com`).

### Kapan Tidak Perlu (Atau Menggunakan Reverse Proxy)
- Arsitektur monolit atau Same-Origin: Frontend dan backend disajikan di origin yang sama persis (misal via Nginx reverse proxy di mana `/api/*` diteruskan ke backend dan `/` diteruskan ke static frontend).
- Server-to-server communication: Komunikasi antar backend Node.js, Go, atau Python tidak melalui browser, sehingga CORS sama sekali tidak berlaku.

## 8. Metrik Kunci & Observabilitas

1. **Preflight Latency (OPTIONS duration)**: Pantau waktu respon HTTP `OPTIONS` di tab DevTools Network. Preflight yang lambat (misal >300ms) akan memperlambat waktu respon aksi user.
2. **Preflight Cache Hit Ratio (`Access-Control-Max-Age`)**: Pastikan server mengirimkan header `Access-Control-Max-Age: 86400` agar browser tidak mengulang preflight `OPTIONS` pada setiap kali klik tombol.
3. **Console Security Errors**: Pantau error log di Sentry atau browser console log agregator untuk mendeteksi domain sah yang terblokir akibat whitelist CORS tertinggal.

## 9. Studi Kasus Industri Nyata

Sebuah platform SaaS analytics memisahkan frontend dashboard ke `app.saas.io` dan REST API ke `api.saas.io`. 

Setiap kali pengguna mengklik filter data dashboard, frontend melakukan HTTP POST dengan JSON payload. Karena payload bertipe `application/json`, browser menganggapnya request non-simple dan mengirimkan request `OPTIONS` terlebih dahulu.

Pada arsitektur awal, backend tidak menyetel header `Access-Control-Max-Age`. Akibatnya, setiap 1 aksi user memicu 2 request HTTP: 1 OPTIONS (250ms) + 1 POST (200ms) = total 450ms. Latency dashboard terasa lambat.

Solusi engineering:
1. Menambahkan `Access-Control-Max-Age: 86400` (24 jam) pada respon preflight server.
2. Memasang Reverse Proxy (Cloudflare / Nginx) di edge layer sehingga `app.saas.io/api/*` di-proxy ke origin backend. Perubahan ini mengubah interaksi menjadi satu origin (Same-Origin), mengeliminasi request OPTIONS 100% dan memangkas latency interaksi user hingga 55%.

## 10. Fakta Menarik & Gotchas Production

- **Wildcard + Credentials Haram**: Browser akan menolak request jika backend merespon dengan `Access-Control-Allow-Origin: *` sementara frontend menyetel `credentials: 'include'`. Origin harus dikembalikan secara spesifik (refleksi origin).
- **CORS Bukan Firewall**: Jika Anda melakukan POST tanpa preflight (simple request seperti form urlencoded), data **tetap masuk dan diproses di database server**, namun responnya yang diblokir oleh browser agar JavaScript Anda tidak bisa membacanya. Jangan jadikan CORS sebagai satu-satunya sistem autentikasi data!
- **Error Status HTTP Backend**: Jika endpoint backend mengembalikan HTTP 500 error tanpa menyertakan CORS header di middleware error handler, browser akan menampilkan error `CORS Policy blocked`, menutupi pesan error 500 yang sebenarnya. Selalu tempatkan middleware CORS di baris paling pertama pipeline backend.

## 11. Rangkuman Singkat

CORS adalah mekanisme keamanan browser yang mengatur pertukaran resource antar origin yang berbeda via HTTP headers. Kesalahan CORS diselesaikan dengan konfigurasi header yang tepat di backend (`Access-Control-Allow-Origin`, `Access-Control-Allow-Methods`, `Access-Control-Max-Age`), bukan dengan mematikan keamanan browser di frontend.
