# HTTP / HTTPS

## 1. Skenario Sistem Nyata

Pada sebuah aplikasi web perbankan dan portal pembayaran gaji (*payroll*), seorang karyawan sedang mengakses dashboard keuangan perusahaannya melalui koneksi jaringan Wi-Fi publik di kedai kopi bandara.

Aplikasi web tersebut masih menggunakan protokol **HTTP versi 1.1 tanpa enkripsi SSL/TLS (Plain HTTP)**:
- Seluruh paket data yang dikirimkan antara browser karyawan dan server backend mengalir dalam format teks polos (*cleartext*).
- Seorang peretas yang berada di jaringan Wi-Fi yang sama menjalankan alat penyadap paket (*packet sniffer* seperti Wireshark). Peretas tersebut dapat membaca nomor rekening, saldo gaji, dan token autentikasi sesi login karyawan secara transparan (*Man-in-the-Middle / Eavesdropping Attack*).
- Selain masalah keamanan, halaman dashboard memuat 45 file ikon kecil dan stylesheet terpisah. Karena HTTP/1.1 mengalami masalah **Head-of-Line (HoL) Blocking** pada level koneksi TCP (browser dibatasi hanya 6 koneksi simultan per domain), puluhan file tersebut harus mengantre bergantian, membuat waktu pemuatan memakan waktu 4,5 detik.

Solusi: Mengadopsi **HTTPS (HTTP over TLS)** yang dipadukan dengan protokol generasi modern: **HTTP/2 (Multiplexing)** dan **HTTP/3 (berbasis QUIC UDP)**. Seluruh data terenkripsi secara kriptografis anti-sadap, dan seluruh 45 aset dapat diunduh secara paralel melalui satu koneksi tunggal tanpa antrean.

## 2. Apa Itu dan Buat Apa?

HTTP (Hypertext Transfer Protocol) adalah protokol lapisan aplikasi (*application layer protocol*) yang menjadi fondasi pertukaran data di World Wide Web, mengatur bagaimana pesan permintaan (*requests*) dan balasan (*responses*) diformat dan ditransmisikan antara browser klien dan server web.

Evolusi arsitektur protokol web:
1. **HTTP/1.1**: Model permintaan berbasis teks sekuensial. Membatasi peramban hanya dapat mengeksekusi maksimal 6 koneksi TCP paralel per domain (*Head-of-Line Blocking pada layer aplikasi*).
2. **HTTPS (HTTP + TLS)**: Lapisan keamanan kriptografis yang mengenkripsi seluruh muatan data (URL path, header, payload JSON, cookies) menggunakan sertifikat SSL/TLS, menjamin kerahasiaan (*confidentiality*), integritas data (*integrity*), dan autentikasi server.
3. **HTTP/2**: Memperkenalkan format biner dan **Multiplexing** murni. Ratusan request dan response dapat dikirimkan bolak-balik secara bersamaan melalui **satu koneksi TCP tunggal**, disertai kompresi header (HPACK) dan Server Push.
4. **HTTP/3 (QUIC)**: Menggantikan protokol transport TCP dengan protokol **QUIC berbasis UDP**. Menghilangkan Head-of-Line Blocking pada lapisan transport paket jaringan (*packet loss isolation*) dan mendukung pergantian jaringan instan (*Connection Migration*, misal ponsel beralih dari Wi-Fi ke 4G tanpa putus koneksi).

Fungsi utama memahami HTTP/HTTPS:
- Mengamankan komunikasi data sensitif pengguna dari penyadapan dan manipulasi jaringan.
- Memahami batas kemampuan transmisi jaringan agar tidak menulis arsitektur request frontend yang menyumbat pipa browser.

## 3. Kenapa Harus Pakai Ini?

Di era web modern:
1. **Keharusan Industri Mutlak**: Browser modern (Chrome, Safari, Firefox) menandai semua situs HTTP biasa sebagai *"Not Secure"*, memblokir akses ke Web APIs modern (Geolocation, Camera, Service Workers, Clipboard, Web Crypto).
2. **Kecepatan Unduhan Paralel**: HTTP/2 dan HTTP/3 melipatgandakan kecepatan pengunduhan aset frontend hingga 3x lebih cepat tanpa perlu trik lama seperti *domain sharding*.

## 4. Cara Kerjanya (Diagram Alur HTTP/1.1 vs HTTP/2 vs HTTP/3)

```
HTTP/1.1 (6 Koneksi TCP Terpisah, Antrean Menunggu / HoL Blocking):
Koneksi 1: [ Request A ] ===> [ Response A ] ===> [ Request G ] ...
Koneksi 2: [ Request B ] ===> [ Response B ] ...
Koneksi 3: [ Request C ] ===> [ Response C ] ...
(Maksimal 6 TCP socket. File ke-7 sampai ke-45 WAJIB MENGANTRE!)

----------------------------------------------------------------------

HTTP/2 (1 Koneksi TCP Tunggal, Multiplexing Biner Bebas Antrean):
Koneksi TCP Tunggal:
[ Stream 1: Req A ][ Stream 2: Req B ][ Stream 3: Req C ][ Stream 4: Req D ]
                    |
                    v
Server memproses semuanya secara simultan dan mengembalikan frame biner!
*Nol waktu tunggu antrean soket di browser!*

----------------------------------------------------------------------

HTTP/3 (Berbasis QUIC UDP - Kebal Packet Loss):
Koneksi QUIC / UDP:
[ Stream 1 ][ Stream 2 ][ Stream 3 ][ Stream 4 ]
Jika paket Stream 2 hilang di jaringan seluler,
Stream 1, 3, dan 4 TETAP BERJALAN LANCAR tanpa tertahan!
```

## 5. Cuplikan Logika Inti

### Header Keamanan Wajib HTTPS (Security Headers)

Server wajib mengirimkan header ini untuk memastikan browser selalu menggunakan HTTPS:

```http
# 1. HSTS (HTTP Strict Transport Security):
# Memaksa browser hanya berkomunikasi via HTTPS selama 1 tahun ke depan,
# menolak koneksi HTTP biasa bahkan jika pengguna mengetik http:// secara manual!
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload

# 2. Upgrade Insecure Requests:
# Menginstruksikan browser untuk meng-upgrade seluruh link gambar http:// menjadi https://
Content-Security-Policy: upgrade-insecure-requests
```

### Memeriksa Protokol yang Digunakan Melalui JavaScript di Browser

```javascript
// Mendeteksi protokol halaman saat ini
if (window.location.protocol !== "https:" && window.location.hostname !== "localhost") {
  console.warn("Peringatan: Halaman tidak berjalan di atas koneksi HTTPS yang aman!");
}

// Memeriksa protokol transmisi aset via Performance API
const resources = performance.getEntriesByType("resource");
resources.slice(0, 5).forEach((res) => {
  console.log(`Berkas: ${res.name.split("/").pop()} | Protokol: ${res.nextHopProtocol}`);
  // res.nextHopProtocol akan bernilai:
  // - "http/1.1"
  // - "h2" (HTTP/2)
  // - "h3" (HTTP/3)
});
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan
- **Enkripsi Menyeluruh (End-to-End Encryption)**: Mencegah ISP nakal menyuntikkan iklan atau malware (*ad injection*) ke dalam website Anda.
- **Efisiensi TCP Connection Handshake**: HTTP/2 dan HTTP/3 mengurangi beban overhead CPU server karena ratusan request berbagi satu sesi koneksi.
- **Kepatuhan Privasi Global**: Memenuhi standar regulasi kepatuhan data internasional (GDPR, PCI-DSS, UU PDP Indonesia).

### Kekurangan & Biaya
- **Overhead Awal Handshake TLS**: Koneksi pertama membutuhkan 1 hingga 2 round-trip ekstra untuk negosiasi kunci kriptografi sebelum data pertama dikirimkan (dapat dimitigasi dengan TLS 1.3 0-RTT dan session resumption).
- **Manajemen Sertifikat SSL**: Membutuhkan pembaruan dan rotasi sertifikat berkala (meskipun kini telah diotomatisasi secara gratis oleh Let's Encrypt atau Cloudflare).

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Wajib Menggunakan HTTPS & HTTP/2 / HTTP/3
- **100% Seluruh Website dan API Produksi Tanpa Pengecualian**: Di era modern, tidak ada alasan teknis apa pun untuk merilis situs web publik tanpa HTTPS.

### Kapan Diperbolehkan Menggunakan HTTP Biasa
- Pengembangan lokal di komputer sendiri (`http://localhost:3000`), di mana browser memperlakukan `localhost` sebagai *Secure Context* khusus pengembang.

## 8. Metrik Kunci & Observabilitas

- **Connection Time & TLS Handshake Duration**: Pantau di tab Network DevTools (bagian *Timing Waterfall*). Di jaringan modern, durasi TLS handshake dengan TLS 1.3 idealnya **< 50 ms**.
- **Protocol Distribution**: Amati rasio request yang berhasil dilayani menggunakan `h2` atau `h3` dibandingkan `http/1.1`. Jika lebih dari 10% aset masih disajikan via `http/1.1`, periksa konfigurasi web server atau load balancer Anda.

## 9. Studi Kasus Industri Nyata

### Migrasi Google ke Protokol QUIC / HTTP/3

- **Tantangan Rekayasa Riil**:
  Pengguna layanan Google Search dan YouTube di jaringan seluler sering mengalami jeda buffering ketika berada di dalam kendaraan yang berpindah dari pemancar sinyal satu ke pemancar sinyal lain (*Cell Tower Handoff*). Pada protokol TCP konvensional, setiap kali alamat IP ponsel berganti, koneksi TCP putus total dan harus mengulang proses 3-way handshake dari awal.
- **Implementasi Solusi Industri**:
  Google merancang dan mengimplementasikan protokol **QUIC (fondasi HTTP/3)**:
  1. QUIC mengidentifikasi koneksi menggunakan *Connection ID* 64-bit unik, bukan berdasarkan pasangan IP dan Port.
  2. Ketika ponsel berganti jaringan dari Wi-Fi rumah ke sinyal 4G, koneksi tetap berjalan tanpa terputus (*Connection Migration*).
  3. Mengeliminasi masalah Head-of-Line Blocking pada saat terjadi kehilangan paket data (*packet loss*).
- **Dampak & Hasil Sistem**:
  - Waktu buffering video YouTube berkurang sebesar 15% secara global.
  - Latensi pencarian Google pada koneksi seluler lambat membaik sebesar 8%.

## 10. Fakta Menarik & Gotchas Production

1. **Anti-Pattern Domain Sharding di Era HTTP/2**:
   Di era HTTP/1.1 dulu, developer mengakali batasan 6 koneksi dengan membagi aset ke beberapa subdomain (`static1.example.com`, `static2.example.com`). Di era HTTP/2 dan HTTP/3, **teknik domain sharding justru memperlambat website!** Menggunakan banyak subdomain memaksa browser membuka banyak koneksi TCP/TLS baru, mematikan manfaat utama dari multiplexing tunggal!
2. **HTTPS Tidak Mengenkripsi Nama Domain (*SNI*)**:
   Meskipun URL path lengkap (`/akun/rahasia`) dan isi payload terenkripsi rapat di HTTPS, nama domain utama (`bank.com`) tetap terlihat secara publik oleh penyedia internet melalui ekstensi *Server Name Indication (SNI)* saat inisiasi koneksi.

## 11. Rangkuman Singkat

1. **Standar Wajib**: HTTPS adalah keharusan mutlak untuk privasi, integritas data, dan akses Web APIs modern.
2. **Kekuatan Multiplexing**: HTTP/2 menggabungkan ratusan request ke dalam satu koneksi TCP tunggal tanpa antrean.
3. **Masa Depan HTTP/3**: Berjalan di atas protokol QUIC UDP untuk ketahanan terhadap kehilangan paket data dan migrasi jaringan seluler yang mulus.
4. **Tinggalkan Trik Lama**: Hindari domain sharding di era HTTP/2; sajikan seluruh aset melalui satu domain terpadu.
