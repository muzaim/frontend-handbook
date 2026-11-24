# Clickjacking

## 1. Skenario Sistem Nyata

Seorang pengguna sedang membuka situs kuis hiburan `https://kuis-berhadiah.xyz`. Di layar muncul tombol besar bertuliskan:
"Klik di Sini untuk Mengklaim iPhone Gratis!"

Pengguna yang bersemangat mengarahkan kursor dan mengklik tombol tersebut berkali-kali. Namun, tanpa disadari, di atas tombol kuis tersebut terdapat elemen `<iframe>` transparan (`opacity: 0; z-index: 9999`) yang memuat halaman pengaturan akun media sosial milik korban (`https://social.com/settings/delete-account`).

Tombol "Klaim iPhone" yang dilihat oleh pengguna sejatinya berada tepat di bawah tombol "Konfirmasi Hapus Akun" dari situs media sosial di dalam iframe. Dengan mengklik tombol palsu tersebut, pengguna secara tidak sadar menghapus akun media sosialnya sendiri.

Serangan ini dikenal luas dalam dunia keamanan siber sebagai **Clickjacking (UI Redressing)**.

## 2. Apa Itu dan Buat Apa?

Clickjacking (atau *User Interface Redressing*) adalah teknik serangan berbahaya di mana penyerang membungkus situs web target yang sah ke dalam bingkai transparan (`<iframe>`), lalu melapisinya di atas antarmuka tipuan. 

Tujuannya adalah memanipulasi korban agar tanpa sengaja mengklik tombol, link, atau form sensitif di situs target (seperti transfer uang, menghapus data, atau menyetujui izin aplikasi).

Pencegahan Clickjacking bertujuan melarang atau mengontrol secara ketat situs web mana saja yang diizinkan untuk menyematkan (*embed*) halaman aplikasi Anda ke dalam tag `<iframe>`, `<frame>`, `<embed>`, atau `<object>`.

## 3. Kenapa Harus Pakai Ini?

Browser secara default mengizinkan website mana pun di internet untuk menyematkan URL website lain ke dalam tag `<iframe>`.

Tanpa perlindungan anti-clickjacking:
- Penyerang bisa menyematkan halaman transfer bank korban, lalu mengarahkan klik user ke tombol "Transfer Saldo".
- Penyerang bisa menyematkan halaman persetujuan izin OAuth ("Izinkan aplikasi ini mengakses Google Drive Anda").
- Penyerang bisa menyematkan tombol "Like", "Follow", atau share palsu untuk memanipulasi algoritma media sosial.

Dengan perlindungan anti-clickjacking:
- Browser menolak menampilkan halaman aplikasi Anda di dalam iframe yang berasal dari domain yang tidak sah, menggantinya dengan halaman blank atau pesan error penolakan frame.

## 4. Cara Kerjanya

Berikut adalah diagram perbandingan antara anatomi serangan Clickjacking dan mekanisme pemblokiran oleh browser:

```
--- ANATOMI SERANGAN CLICKJACKING ---
Layer Tampilan yang Dilihat Korban:
+---------------------------------------------------+
|  Situs Penyerang: kuis-berhadiah.xyz              |
|                                                   |
|       [ KLIK UNTUK HADIAH Rp 1.000.000 ]          |
|                                                   |
+---------------------------------------------------+

Layer Nyata di Bawah Kursor (Iframe Transparan opacity: 0):
+---------------------------------------------------+
|  Iframe: bank.com/transfer/confirm                |
|                                                   |
|       [ KONFIRMASI TRANSFER Rp 50.000.000 ]       |  <-- Klik mendarat di sini!
|                                                   |
+---------------------------------------------------+


--- PERTAHANAN BROWSER DENGAN CSP & X-FRAME-OPTIONS ---
[ Browser Memuat Halaman Penyerang ] -> <iframe src="https://bank.com/transfer">
                  |
                  v
[ Request HTTP GET ke bank.com ]
                  |
                  v
[ Respon Server bank.com ]
HTTP 200 OK
Content-Security-Policy: frame-ancestors 'none';
X-Frame-Options: DENY
                  |
                  v
[ Browser Engine Memeriksa Header ]
- Apakah parent window diizinkan menyematkan halaman ini?
- Jawaban: TIDAK! ('none' / DENY)
                  |
                  v
[ BROWSER MEMBLOKIR RENDER IFRAME ]
Menampilkan error: "bank.com refused to connect inside frame"
Serangan Clickjacking Gagal Total!
```

## 5. Cuplikan Logika Inti

### Konfigurasi HTTP Header di Server (Nginx & Express)

Metode pertahanan anti-clickjacking standar dan terkuat adalah melalui konfigurasi HTTP header di level web server atau reverse proxy:

#### 1. Konfigurasi Nginx
```nginx
# Melarang SEMUA domain (termasuk domain sendiri) menyematkan halaman ini ke iframe
add_header X-Frame-Options "DENY" always;

# Standar Modern (CSP Level 2 & 3): Menggantikan X-Frame-Options dengan kontrol lebih fleksibel
add_header Content-Security-Policy "frame-ancestors 'none';" always;
```

#### 2. Konfigurasi Node.js Express (dengan Helmet)
```typescript
// backend/src/server.ts
import express from 'express';
import helmet from 'helmet';

const app = express();

app.use(
  helmet({
    // Header lawas untuk kompatibilitas browser lama (Internet Explorer / Safari lama)
    xFrameOptions: { action: 'deny' },
    
    // Header standar modern W3C
    contentSecurityPolicy: {
      directives: {
        defaultSrc: ["'self'"],
        // 'none' = tidak boleh di-iframe oleh siapa pun
        // 'self' = hanya boleh di-iframe oleh domain yang sama persis
        // 'self' https://partner.com = izinkan domain mitra tertentu
        frameAncestors: ["'none'"],
      },
    },
  })
);
```

### Mengapa Skrip JavaScript "Frame Busting" Tidak Cukup?

Dahulu developer menggunakan skrip pengecekan manual:
```javascript
// JANGAN HANYA MENGANDALKAN KODE INI!
if (window.top !== window.self) {
  window.top.location = window.self.location;
}
```
Metode ini mudah dilumpuhkan oleh penyerang dengan menambahkan atribut HTML5: `<iframe src="..." sandbox="allow-forms allow-scripts">` tanpa menyertakan `allow-top-navigation`, yang memblokir skrip pengalihan frame busting. Gunakan selalu HTTP Header!

## 6. Kelebihan dan Kekurangan (Trade-offs)

| Solusi | Kelebihan | Kekurangan / Risiko |
| :--- | :--- | :--- |
| `Content-Security-Policy: frame-ancestors` | Standar modern, sangat granular (bisa whitelist beberapa domain spesifik) | Tidak didukung oleh browser purba (IE11 ke bawah) |
| `X-Frame-Options: DENY` | Didukung oleh 100% browser di seluruh dunia | Hanya mendukung nilai biner `DENY` atau `SAMEORIGIN`, tidak bisa multi-domain whitelist |
| Mengizinkan Iframe Penuh | Fleksibel untuk widget pihak ketiga | Membuka celah serangan pembajakan klik jika halaman berisi tombol mutasi data |

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Menggunakan `frame-ancestors 'none'` / `DENY`
- Halaman sensitif: dashboard perbankan, form transfer, reset password, hapus data, checkout pembayaran.
- Seluruh aplikasi web SaaS internal perusahaan.

### Kapan Boleh Mengizinkan Iframe (`frame-ancestors 'self' https://trusted-partner.com`)
- Halaman widget yang memang dirancang untuk disematkan di web lain (misal: widget pemutar YouTube video, widget peta Google Maps, atau tombol pembayaran terintegrasi Midtrans/Stripe).
- Halaman dokumentasi publik atau blog statis yang tidak memiliki aksi login atau perubahan data.

## 8. Metrik Kunci & Observabilitas

1. **Missing Anti-Clickjacking Headers Count**: Jalankan vulnerability scanner (seperti OWASP ZAP) secara berkala di pipeline CI/CD untuk memastikan setiap route memiliki header `X-Frame-Options` atau `frame-ancestors`.
2. **CSP Frame Ancestor Violations**: Pantau laporan CSP jika ada upaya domain asing yang mencoba menyematkan situs Anda ke dalam frame mereka tanpa izin.

## 9. Studi Kasus Industri Nyata

Sebuah platform media sosial terkemuka pernah menjadi korban serangan clickjacking berskala masif yang dijuluki *"Likejacking"*.

Penyerang membuat ribuan situs web berita palsu dan video viral. Di setiap halaman tersebut, tombol "Like" resmi dari media sosial disematkan di dalam iframe transparan yang selalu mengikuti pergerakan kursor mouse pengguna. Setiap kali pengunjung mengklik di sembarang tempat pada halaman untuk memutar video, mereka secara otomatis mengklik "Like" pada postingan spam penyerang.

Hasilnya, halaman penyerang mendapatkan jutaan like organik palsu dalam hitungan hari, mendistorsi trending topic secara drastis.

Solusi engineering:
1. Menerapkan header `X-Frame-Options: SAMEORIGIN` pada seluruh halaman platform.
2. Untuk widget tombol like publik yang memang harus bisa di-embed, tim sekuriti menambahkan konfirmasi dua langkah (*two-factor click confirmation*) dan validasi interaksi mouse berbasis entropy untuk mendeteksi kursor buatan.

## 10. Fakta Menarik & Gotchas Production

- **Meta Tag Tidak Bekerja**: Anda **tidak bisa** menyetel `X-Frame-Options` atau `frame-ancestors` menggunakan tag `<meta http-equiv="...">`. Browser akan mengabaikannya secara sengaja! Kedua proteksi ini hanya dipatuhi jika dikirim melalui HTTP Response Header server.
- **Prioritas Browser**: Jika sebuah response menyertakan `X-Frame-Options: SAMEORIGIN` DAN `Content-Security-Policy: frame-ancestors 'none'`, browser modern akan memprioritaskan instruksi CSP (`frame-ancestors`) dan mengabaikan `X-Frame-Options`.
- **Drag-and-Drop Clickjacking**: Penyerang juga bisa memanipulasi korban untuk men-drag objek dari website tipuan dan melepaskannya (*drop*) ke dalam iframe tersembunyi untuk mengisi formulir secara otomatis.

## 11. Rangkuman Singkat

Clickjacking adalah eksploitasi antarmuka di mana korban ditipu untuk mengklik aksi sensitif di dalam iframe tersembunyi milik situs target. Lindungi seluruh halaman aplikasi Anda menggunakan kombinasi response header modern `Content-Security-Policy: frame-ancestors 'none'` dan header kompatibilitas `X-Frame-Options: DENY`.
