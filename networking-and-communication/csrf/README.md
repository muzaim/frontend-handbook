# CSRF (Cross-Site Request Forgery)

## 1. Skenario Sistem Nyata

Seorang pengguna perbankan sedang login ke akunnya di `https://mybank.com`. Tanpa sengaja, ia membuka tab baru dan mengklik tautan promo menarik yang mengarah ke `https://situs-undian-palsu.com`.

Di dalam halaman situs undian tersebut, terdapat elemen HTML tersembunyi:
```html
<form action="https://mybank.com/api/transfer" method="POST" id="evilForm">
  <input type="hidden" name="recipient" value="hacker123" />
  <input type="hidden" name="amount" value="50000000" />
</form>
<script>document.getElementById('evilForm').submit();</script>
```

Saat halaman dimuat, skrip tersebut otomatis melakukan submit form ke `mybank.com`. Karena browser secara default menyertakan cookie otentikasi milik pengguna ke endpoint target, server bank menganggap request tersebut sah berasal dari pemilik rekening. Saldo pengguna terpotong Rp 50.000.000 tanpa izin sama sekali.

Serangan ini dikenal sebagai **CSRF (Cross-Site Request Forgery)**.

## 2. Apa Itu dan Buat Apa?

CSRF (Cross-Site Request Forgery) adalah jenis serangan di mana penyerang memperdaya browser korban yang telah terotentikasi untuk mengirimkan request HTTP berbahaya yang tidak diinginkan ke server aplikasi web target.

Tujuan perlindungan CSRF (Anti-CSRF) adalah memverifikasi bahwa setiap aksi mutasi data (POST, PUT, DELETE, PATCH) yang diterima oleh backend benar-benar dipicu secara sadar oleh user dari antarmuka aplikasi resmi, bukan dipicu secara silang oleh website pihak ketiga.

## 3. Kenapa Harus Pakai Ini?

Browser memiliki perilaku bawaan (*ambient authority*): browser akan selalu melampirkan cookie yang cocok dengan domain tujuan ketika mengirim request HTTP, terlepas dari website mana yang memicu request tersebut.

Tanpa proteksi Anti-CSRF:
- Penyerang bisa mengubah alamat email atau password akun korban.
- Penyerang bisa melakukan transaksi transfer finansial, pembelian barang, atau mengubah konfigurasi router dan firewall korban.

Dengan proteksi Anti-CSRF:
- Server menolak semua request mutasi data yang tidak menyertakan bukti rahasia yang hanya diketahui oleh halaman aplikasi resmi (Anti-CSRF token) atau yang melanggar batasan `SameSite` cookie.

## 4. Cara Kerjanya

Berikut adalah diagram alur serangan CSRF tanpa proteksi vs mekanisme pertahanan menggunakan **Synchronizer Token Pattern**:

```
--- SKENARIO SERANGAN CSRF (TANPA PROTEKSI) ---
User Login di Bank.com ---> Membuka Evil.com di Tab Baru
                              |
                              |-- Form Submit POST /transfer -------------> Server Bank
                              |   Cookie: session_id=valid (Otomatis)      (Request diproses!)
                              v

--- SKENARIO PERTAHANAN: SYNCHRONIZER TOKEN & SAMESITE ---
[ User di frontend resmi: mybank.com ]                     [ Backend Server ]
                 |                                                 |
                 |--- 1. GET /api/csrf-token --------------------->|
                 |<-- 2. HTTP 200 { csrfToken: "rnd_token_abc" } --| (Simpan di session)
                 |                                                 |
                 |--- 3. POST /api/transfer ---------------------->|
                 |    Headers:                                     | (Server mencocokkan
                 |      Cookie: session_id=valid                   |  X-CSRF-Token di header
                 |      X-CSRF-Token: "rnd_token_abc"              |  dengan token di session)
                 |<-- 4. HTTP 200 Sukses --------------------------|
                 |                                                 |
[ User membuka evil.com ]                                          |
                 |                                                 |
                 |--- 5. POST /api/transfer (Palsu) -------------->|
                 |    Cookie: session_id (Ditolak oleh SameSite)   | (Jika lolos pun, evil.com
                 |    X-CSRF-Token: KOSONG / SALAH                |  TIDAK bisa membaca token
                 |<-- 6. HTTP 403 Forbidden -----------------------|  resmi karena aturan SOP)
                 v                                                 v
```

## 5. Cuplikan Logika Inti

### Client-side: Melampirkan Anti-CSRF Token pada Fetch Wrapper

Frontend mengambil token saat inisialisasi aplikasi atau membaca dari meta tag, lalu menyematkannya ke setiap request mutasi data:

```typescript
// frontend/src/services/apiClient.ts
let csrfToken: string | null = null;

export async function initCsrfProtection(): Promise<void> {
  // Ambil token dari backend atau elemen meta tag SSR
  const metaTag = document.querySelector<HTMLMetaElement>('meta[name="csrf-token"]');
  if (metaTag) {
    csrfToken = metaTag.content;
  } else {
    const res = await fetch('/api/auth/csrf', { credentials: 'include' });
    const data = await res.json();
    csrfToken = data.csrfToken;
  }
}

export async function secureMutation(endpoint: string, method: 'POST' | 'PUT' | 'DELETE', payload: any) {
  if (!csrfToken) {
    await initCsrfProtection();
  }

  const response = await fetch(endpoint, {
    method,
    headers: {
      'Content-Type': 'application/json',
      // Header kustom Anti-CSRF wajib disematkan
      'X-CSRF-Token': csrfToken || '',
      'X-Requested-With': 'XMLHttpRequest',
    },
    credentials: 'include',
    body: JSON.stringify(payload),
  });

  if (response.status === 403) {
    throw new Error('Validasi Anti-CSRF gagal. Silakan muat ulang halaman.');
  }

  return response.json();
}
```

### Konfigurasi Backend: SameSite Cookie & Token Verifier (Express)

```typescript
// backend/src/middleware/csrf.ts
import { Request, Response, NextFunction } from 'express';

export function enforceSameSiteCookie(res: Response, sessionId: string) {
  res.cookie('session_id', sessionId, {
    httpOnly: true,
    secure: true,
    sameSite: 'strict', // Pertahanan garis depan paling efektif terhadap CSRF
    path: '/',
  });
}

export function verifyCsrfToken(req: Request, res: Response, next: NextFunction) {
  // Lewatkan metode aman yang idempotent (GET, HEAD, OPTIONS)
  if (['GET', 'HEAD', 'OPTIONS'].includes(req.method)) {
    return next();
  }

  const clientToken = req.headers['x-csrf-token'];
  const sessionToken = req.session?.csrfSecret;

  if (!clientToken || clientToken !== sessionToken) {
    return res.status(403).json({ error: 'Akses ditolak: CSRF token tidak cocok atau kedaluwarsa' });
  }

  next();
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

| Kelebihan | Kekurangan / Risiko |
| :--- | :--- |
| Mencegah aksi mutasi akun berbahaya yang dipicu oleh website pihak ketiga | Menambah kompleksitas state frontend untuk mengelola siklus hidup token |
| Sangat efektif melindungi sistem transaksi keuangan dan administrasi | Dapat memicu kegagalan transaksi sah jika tab user terlalu lama idle (token expired) |
| Cookie `SameSite=Strict` melindungi request tanpa perlu kalkulasi token manual | Memerlukan sinkronisasi token ekstra saat pengguna membuka multi-tab secara bersamaan |

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Wajib Dipakai
- Aplikasi berbasis otentikasi Cookie Sesi di mana browser otomatis mengirimkan kredensial.
- Endpoint sensitif yang melakukan perubahan status data: transfer uang, reset password, hapus akun, update email.

### Kapan Tidak Perlu
- Sistem yang otentikasinya murni menggunakan header `Authorization: Bearer <JWT>` yang disimpan di memori JavaScript (bukan cookie). Website pihak ketiga tidak bisa memaksa browser menyertakan header custom tersebut tanpa izin CORS.
- Endpoint idempotent (hanya melakukan pembacaan data via HTTP `GET`).

## 8. Metrik Kunci & Observabilitas

1. **CSRF Validation Failure Count**: Jumlah log error HTTP 403 dengan alasan invalid CSRF token. Lonjakan mendadak bisa berarti serangan aktif atau bug frontend pada token refresh.
2. **SameSite Cookie Adoption Rate**: Persentase cookie autentikasi yang sudah memiliki atribut `SameSite=Lax` atau `SameSite=Strict`. Harus 100% di production.
3. **Multi-tab Form Desync Rate**: Jumlah komplain pengguna yang mengalami "Session Expired" saat mengisi form panjang akibat token CSRF tertimpa oleh tab baru.

## 9. Studi Kasus Industri Nyata

Sebuah platform SaaS manajemen proyek populer mengalami insiden di mana penyerang menyebarkan link di forum komunitas publik. Ketika link dibuka oleh user yang sedang login di platform tersebut, skrip di situs penyerang mengirimkan request `POST https://saas.com/api/workspace/delete-all`.

Karena endpoint backend saat itu hanya mengandalkan cookie sesi tanpa validasi Anti-CSRF token, ratusan workspace pelanggan terhapus secara permanen.

Langkah pemulihan keamanan yang diterapkan:
1. Memperbarui semua cookie sesi menjadi `SameSite=Lax`.
2. Menerapkan *Double Submit Cookie* dan header `X-CSRF-Token` pada seluruh endpoint berjenis POST, PUT, DELETE.
3. Menambahkan verifikasi ulang password (*Re-authentication / Step-up Auth*) untuk aksi destruktif tingkat tinggi.

## 10. Fakta Menarik & Gotchas Production

- **GET Request Tidak Boleh Mutasi**: Jika endpoint backend Anda mengizinkan aksi mutasi data via HTTP GET (misal: `<img src="https://bank.com/transfer?amount=1000">`), maka proteksi CSRF token di POST menjadi tidak berguna karena browser otomatis mengeksekusi tag `img` tanpa preflight.
- **SameSite=Lax Bukan Obat Mutlak**: `SameSite=Lax` tetap mengirimkan cookie pada navigasi top-level (misal user mengklik link tag `<a>`). Jika backend Anda mengubah data penting via GET, Anda tetap rentan terhadap CSRF.
- **Hubungan dengan XSS**: Jika aplikasi Anda memiliki celah XSS, proteksi CSRF bisa dibobol dengan mudah karena skrip penyerang bisa membaca DOM dan mencuri Anti-CSRF token sebelum mengirim request.

## 11. Rangkuman Singkat

CSRF adalah eksploitasi di mana website pihak ketiga memanfaatkan cookie otomatis browser untuk mengeksekusi aksi ilegal atas nama korban. Pertahanan utamanya adalah kombinasi cookie `SameSite=Strict/Lax`, penggunaan header kustom Anti-CSRF token, dan memastikan metode HTTP GET tidak pernah memodifikasi data.
