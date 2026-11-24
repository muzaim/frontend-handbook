# JWT Authentication (JSON Web Token)

## 1. Skenario Sistem Nyata

Sebuah platform streaming video memiliki ratusan microservices di belakang API Gateway. Setiap detik, terdapat 200.000 HTTP requests yang masuk ke berbagai layanan: pemutaran video, rekomendasi, komentar, dan analitik.

Jika sistem menggunakan arsitektur session terpusat di database atau Redis, setiap kali ada 1 request masuk ke salah satu microservice, service tersebut harus melakukan kueri ke Redis untuk mengecek validitas token user. Redis menjadi bottleneck raksasa dan single point of failure.

Untuk mengatasi ini, arsitektur beralih ke **JWT Authentication**, di mana identitas user dan izin hak akses diverifikasi secara mandiri (stateless) oleh masing-masing microservice tanpa perlu menyentuh database sesi.

## 2. Apa Itu dan Buat Apa?

JWT (JSON Web Token) adalah standar terbuka (RFC 7519) yang mendefinisikan format token kompak dan mandiri (*self-contained*) untuk mentransmisikan informasi terpercaya antar pihak sebagai objek JSON yang ditandatangani secara kriptografis (*digitally signed*).

Struktur JWT terdiri dari tiga bagian yang dipisahkan oleh titik (`.`):
1. **Header**: Algoritma enkripsi dan tipe token (misal: `{"alg": "HS256", "typ": "JWT"}`).
2. **Payload**: Klaim data pengguna (misal: `{"sub": "123", "name": "Budi", "role": "admin", "exp": 1700000000}`).
3. **Signature**: Tanda tangan digital hasil kalkulasi `HMACSHA256(base64(Header) + "." + base64(Payload), secretKey)`.

JWT digunakan untuk otentikasi stateless dan otorisasi terdistribusi antar layanan mikro atau SPA modern.

## 3. Kenapa Harus Pakai Ini?

- **Stateless & Scalable**: Server penerima tidak perlu menyimpan data sesi pengguna di RAM atau database. Cukup dengan memvalidasi tanda tangan kriptografi menggunakan *secret key* atau *public key* (RSA/ECDSA), server sudah tahu siapa pengguna tersebut.
- **Microservices-Friendly**: Satu token JWT yang diterbitkan oleh Auth Server dapat divalidasi langsung oleh Microservice A, B, dan C tanpa saling bergantung pada Redis terpusat.
- **Self-contained Payload**: Informasi dasar seperti `userId`, `organizationId`, dan daftar `permissions` langsung tersedia di dalam token tanpa perlu query tambahan ke tabel user.

## 4. Cara Kerjanya

Pola produksi paling aman adalah kombinasi **Short-lived Access Token** (di memori JavaScript) dan **Long-lived Refresh Token** (di HttpOnly Cookie):

```
[ Frontend Client ]             [ API Gateway / BFF ]             [ Auth Service ]
        |                                 |                               |
        |--- 1. POST /login ------------->|------------------------------>|
        |    {email, password}            |                               | (Verifikasi creds)
        |                                 |                               | (Generate Access &
        |                                 |                               |  Refresh Token)
        |<-- 2. HTTP 200 OK --------------|<------------------------------|
        |    Body: { accessToken (15m) }  |                               |
        |    Cookie: refreshToken (7d)    |                               |
        |    (HttpOnly, Secure, SameSite) |                               |
        |                                 |                               |
  (Simpan accessToken di memori JS)       |                               |
        |                                 |                               |
        |--- 3. GET /api/data ----------->|                               |
        |    Header: Bearer <accessToken> |                               |
        |                                 |-- Verifikasi Signature OK --->| (Kirim data tanpa
        |<-- 4. HTTP 200 OK --------------|<-- Return resource data ------|  query database)
        |                                 |                               |
  (15 Menit kemudian: Access Token Expired)                               |
        |                                 |                               |
        |--- 5. POST /api/auth/refresh -->|------------------------------>|
        |    (Cookie refreshToken kirim)  |                               | (Cek revoke list DB)
        |<-- 6. HTTP 200 OK --------------|<------------------------------| (Kirim new accessToken)
        |    Body: { newAccessToken }     |                               |
        v                                 v                               v
```

## 5. Cuplikan Logika Inti

### Client-side: Token Refresh Interceptor (Axios / Fetch)

Logika frontend untuk menyisipkan Access Token di memori dan melakukan *silent refresh* ketika token kedaluwarsa (HTTP 401):

```typescript
// frontend/src/api/authClient.ts
let inMemoryAccessToken: string | null = null;

export function setAccessToken(token: string | null) {
  inMemoryAccessToken = token;
}

export function getAccessToken(): string | null {
  return inMemoryAccessToken;
}

// Interceptor fetch otomatis untuk silent refresh
export async function secureFetch(url: string, options: RequestInit = {}): Promise<Response> {
  const headers = new Headers(options.headers || {});
  
  if (inMemoryAccessToken) {
    headers.set('Authorization', `Bearer ${inMemoryAccessToken}`);
  }

  let response = await fetch(url, { ...options, headers, credentials: 'include' });

  // Jika token kedaluwarsa (401), minta token baru via refresh endpoint
  if (response.status === 401) {
    const refreshResponse = await fetch('/api/auth/refresh', {
      method: 'POST',
      credentials: 'include', // Kirim HttpOnly refresh cookie
    });

    if (refreshResponse.ok) {
      const data = await refreshResponse.json();
      setAccessToken(data.accessToken);

      // Ulangi request asli dengan token baru
      headers.set('Authorization', `Bearer ${data.accessToken}`);
      response = await fetch(url, { ...options, headers, credentials: 'include' });
    } else {
      // Refresh token juga sudah kedaluwarsa, wajib login ulang
      setAccessToken(null);
      window.location.href = '/login';
    }
  }

  return response;
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

| Kelebihan | Kekurangan / Risiko |
| :--- | :--- |
| Benar-benar stateless di sisi server, sangat ideal untuk sistem high-throughput | Sulit membatalkan (*revoke*) token yang belum kedaluwarsa secara instan |
| Mengurangi beban read database/Redis untuk pengecekan session | Ukuran header HTTP membesar karena payload JWT di-encode Base64 |
| Fleksibilitas otorisasi lintas berbagai domain dan service | Jika private key server bocor, penyerang bisa memalsukan token siapapun |

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Pakai
- Arsitektur Microservices di mana autentikasi diverifikasi di banyak backend independen.
- Mobile Application (iOS/Android) yang berkomunikasi dengan REST/GraphQL backend.
- Single Sign-On (SSO) lintas platform pihak ketiga (menggunakan protokol OAuth 2.0 / OIDC).

### Kapan Jangan Pakai
- Website monolitik sederhana: Session berbasis cookie di Redis jauh lebih sederhana dan mudah dibatalkan kapan saja.
- Kebutuhan fitur keamanan instan: Sistem perbankan di mana admin harus bisa mencabut akses akun karyawan secara detik itu juga (*immediate revocation*).

## 8. Metrik Kunci & Observabilitas

1. **Token Refresh Rate**: Frekuensi hit pada endpoint refresh token. Lonjakan abnormal dapat mengindikasikan bug loop pada interceptor frontend.
2. **JWT Signature Verification Errors**: Jumlah request dengan status 401 akibat tanda tangan token tidak valid (indikasi manipulasi payload atau pergantian secret key yang tidak sinkron).
3. **Access Token Lifespan**: Pantau metrik durasi kedaluwarsa. Rekomendasi durasi di industri adalah 5 hingga 15 menit untuk membatasi celah penyalahgunaan.

## 9. Studi Kasus Industri Nyata

Sebuah platform SaaS edukasi online awalnya menyimpan JWT di browser `localStorage` dengan durasi kedaluwarsa 30 hari agar pengguna tidak perlu repot sering login ulang.

Insiden keamanan terjadi ketika salah satu instruktur membuka link phising yang mengeksploitasi celah XSS pada halaman diskusi murid. Script XSS berhasil mencuri JWT berdurasi 30 hari milik instruktur tersebut. Penyerang menggunakannya untuk mengubah nilai ujian dan menghapus materi video kursus. Tim backend tidak bisa berbuat banyak karena token bersifat stateless dan server tidak memiliki mekanisme pencabutan token.

Solusi mitigasi:
1. Menurunkan durasi Access Token menjadi 10 menit dan menyimpannya hanya di memori variabel JavaScript (in-memory).
2. Menyimpan Refresh Token di Cookie dengan flag `HttpOnly`, `Secure`, dan `SameSite=Strict` dengan mekanisme rotasi (*Refresh Token Rotation*).
3. Menyediakan Redis blacklist kecil di level API Gateway hanya untuk menampung refresh token yang telah dibatalkan secara manual.

## 10. Fakta Menarik & Gotchas Production

- **JWT Bukan Enkripsi**: Payload JWT hanya di-encode Base64Url, **bukan dienkripsi**. Siapapun yang memegang token bisa melihat isi datanya di situs seperti `jwt.io`. Jangan pernah menyimpan password, PIN, atau nomor kartu kredit di dalam JWT payload!
- **Kelemahan Algoritma "none"**: Bug historis terkenal pada library JWT parser lama adalah mengeksekusi token dengan `{"alg": "none"}` yang mengizinkan bypass tanda tangan secara total. Selalu pastikan parser Anda mengunci algoritma yang diizinkan (misal hanya `HS256` atau `RS256`).
- **Penyimpanan di LocalStorage**: Menyimpan JWT di `localStorage` adalah antipattern keamanan yang sangat berbahaya karena sangat rentan terhadap pencurian via XSS.

## 11. Rangkuman Singkat

JWT adalah token terformat JSON yang ditandatangani secara kriptografis untuk otentikasi stateless di arsitektur microservices dan web modern. Praktik terbaik produksi adalah menggunakan short-lived access token di memori JavaScript dipadukan dengan silent refresh token di HttpOnly cookie.
