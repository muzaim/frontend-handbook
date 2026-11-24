# Content Delivery Network (CDN)

## 1. Skenario Sistem Nyata

Sebuah platform SaaS edukasi dan bootcamp online yang berbasis di Jakarta, Indonesia, meluncurkan kurikulum internasional ke pasar Asia Timur, Eropa, dan Amerika Serikat. Server asal (*Origin Server*) aplikasi di-hosting di sebuah pusat data (data center) di Jakarta.

Ketika pengguna dari Tokyo, London, dan San Francisco membuka situs web:
- Sinyal data internet harus melintasi kabel serat optik bawah laut antarbenua sejauh belasan ribu kilometer.
- Setiap kali browser meminta berkas JavaScript, CSS, atau gambar banner, sinyal membutuhkan waktu tempuh fisik (*Round-Trip Time / RTT*) sebesar **250 hingga 350 milidetik** per request.
- Mengunduh 30 aset halaman membutuhkan waktu lebih dari 6 detik hanya untuk latensi jarak geografis (*speed-of-light propagation delay*), meskipun server asal memiliki spesifikasi CPU tercanggih.
- Lebih buruk lagi, ketika ada 50.000 siswa membuka video materi secara bersamaan, bandwidth server asal di Jakarta tersedot habis dan server mengalami kelumpuhan (*bandwidth saturation crash*).

Solusi: Mengadopsi **Content Delivery Network (CDN)** (seperti Cloudflare, Fastly, AWS CloudFront, atau Akamai). Mendistribusikan salinan seluruh aset statis ke ratusan server edge di seluruh kota besar di dunia. Pengguna di Tokyo dilayani oleh server CDN di Tokyo dalam waktu 8 milidetik, pengguna di London dilayani di London, dan server asal di Jakarta terlindungi dari 95% beban trafik.

## 2. Apa Itu dan Buat Apa?

Content Delivery Network (CDN) adalah jaringan server proxy terdistribusi secara geografis di seluruh dunia (*Points of Presence / PoPs*) yang bekerja sama untuk menyajikan konten web (aset statis dan halaman ter-cache) kepada pengguna dari lokasi server yang **paling dekat secara fisik dengan posisi geografis pengguna tersebut**.

Mekanisme inti CDN:
1. **Anycast Routing & GeoDNS**: Mengarahkan request pengguna ke pusat data CDN terdekat secara otomatis berdasarkan topologi jaringan internet.
2. **Edge Caching**: Menyimpan salinan berkas statis (JS, CSS, gambar, video, HTML) di memori RAM dan disk SSD server edge CDN.
3. **Origin Shield / Offloading**: Menyerap lonjakan trafik masif sehingga server backend utama (*origin server*) hanya menerima sedikit beban request.

Fungsi utama CDN:
- **Memangkas Latensi Jaringan (*Latency Reduction*)**: Mengurangi waktu RTT dari ratusan milidetik menjadi sub-15 milidetik.
- **Skalabilitas & Ketahanan DDoS**: Menahan serangan lalu lintas jahat (*Distributed Denial of Service*) di lapisan edge sebelum menyentuh infrastruktur internal perusahaan.
- **Efisiensi Biaya Operasional**: Menghemat biaya bandwidth server asal secara drastis.

## 3. Kenapa Harus Pakai Ini?

Kecepatan cahaya di dalam kabel serat optik adalah hukum alam yang tidak bisa dilanggar oleh software engineer mana pun. Jarak fisik antara pengguna dan server adalah batasan mutlak.

Tanpa CDN:
- Semakin jauh lokasi pengguna dari server asal Anda, semakin lambat halaman web Anda dimuat.
- Satu lonjakan trafik viral dapat langsung menenggelamkan server asal Anda.

CDN memindahkan konten Anda agar berada di "halaman depan rumah" pengguna di seluruh dunia.

## 4. Cara Kerjanya (Diagram Alur Anycast Routing & Edge Cache Hit)

```
[ Pengguna di Tokyo, Jepang ]
             |
             | 1. Meminta: GET /assets/app-v2.js
             v
+---------------------------------------------------------------+
|               EDGE CDN SERVER (Pusat Data Tokyo)              |
+---------------------------------------------------------------+
|  Periksa apakah berkas 'app-v2.js' ada di Cache SSD Tokyo?   |
|                                                               |
|  [CACHE HIT (98% Kasus)]                                      |
|  --> Langsung kembalikan ke pengguna!                         |
|  *LATENSI HANYA 8 MILIDETIK! (Tidak menyentuh Jakarta)*       |
|                                                               |
|  [CACHE MISS (Kasus Pertama Kali)]                            |
|  --> Request diteruskan ke:                                   |
|      ORIGIN SERVER (Jakarta, Indonesia)                       |
|  --> Ambil file, simpan salinan di cache Tokyo,                |
|      lalu kirim ke pengguna Tokyo.                            |
+---------------------------------------------------------------+
```

## 5. Cuplikan Logika Inti

### Konfigurasi Header HTTP untuk Memaksimalkan Edge Caching di CDN

Aset statis yang memiliki hash unik pada nama berkasnya (seperti `app.c9d81a.js` yang dihasilkan oleh bundler Vite/Webpack) dapat di-cache secara permanen:

```http
# Response Header dari Server / CDN untuk Aset Statis dengan File Hashing
HTTP/2 200 OK
Content-Type: application/javascript; charset=UTF-8
Content-Encoding: br

# 1. Cache di browser dan CDN selama 1 tahun (31.536.000 detik)
# 'immutable' memberitahu browser: berkas ini tidak akan pernah berubah kontennya selamanya!
Cache-Control: public, max-age=31536000, immutable

# 2. Tag identifikasi cache CDN (misal Cloudflare)
CF-Cache-Status: HIT
Age: 86400
```

### Konfigurasi untuk File HTML (Dynamic Edge Stale-While-Revalidate):

```http
# File index.html tidak boleh di-cache permanen agar pengguna selalu dapat versi baru!
HTTP/2 200 OK
Content-Type: text/html; charset=UTF-8

# Simpan di browser selama 0 detik, simpan di CDN selama 1 jam,
# izinkan menyajikan versi lama sembari CDN memperbarui di background selama 1 hari
Cache-Control: public, max-age=0, s-maxage=3600, stale-while-revalidate=86400
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan
- **Time to First Byte (TTFB) Secepat Kilat**: Respons HTTP diterima dalam hitungan sub-30 milidetik di seluruh dunia.
- **Perlindungan Terhadap Lonjakan Trafik**: Server asal Anda tidak akan tumbang meskipun ada 1 juta pengunjung masuk bersamaan.
- **Optimasi Otomatis di Lapisan Edge**: Banyak CDN modern menyediakan fitur otomatis seperti konversi format WebP/AVIF, kompresi Brotli, dan HTTP/3 tanpa perlu mengubah kode aplikasi.

### Kekurangan & Tantangan
- **Tantangan Pembersihan Cache (*Cache Invalidation Delay*)**: Ketika Anda melakukan perbaikan darurat (*hotfix*) pada file yang terlanjur di-cache lama di CDN, perubahan tersebut bisa membutuhkan waktu beberapa menit hingga hitungan jam untuk terhapus dari seluruh ribuan server edge di dunia, kecuali Anda menggunakan teknik *Asset Hashing*.
- **Biaya Tambahan untuk Fitur Canggih**: Fitur CDN tingkat lanjut seperti Edge Computing (Cloudflare Workers) atau log streaming real-time memerlukan biaya langganan bulanan.

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Memakai CDN
- **Hampir Seluruh Website Produksi Publik**: Wajib digunakan untuk menyajikan seluruh aset statis (JavaScript, CSS, Gambar, Ikon SVG, Font).
- Aplikasi web yang memiliki audiens pengguna lintas kota, lintas pulau, atau internasional.
- Media video streaming dan dokumen unduhan berukuran besar.

### Kapan Jangan Di-Cache di Edge CDN
- Response API yang memuat data privat spesifik per akun pengguna (misal endpoint `/api/user/profile` atau `/api/wallet/balance`). Data privat harus menyertakan header `Cache-Control: private, no-store` agar tidak tersimpan di server CDN publik dan tertampil ke pengguna lain!

## 8. Metrik Kunci & Observabilitas

- **CDN Cache Hit Ratio (CHR)**: Metrik terpenting performa CDN:
  `Cache Hit Ratio = (Total Cache Hits / Total Requests) * 100%`.
  Target produksi sehat untuk aset statis adalah **> 95%**. Jika angka CHR turun di bawah 80%, periksa kembali konfigurasi header `Cache-Control` Anda.
- **Edge TTFB**: Pantau waktu TTFB aset statis di berbagai belahan dunia (target: **< 50 ms** global).
- **Origin Bandwidth Savings**: Persentase penghematan beban data keluar pada server asal.

## 9. Studi Kasus Industri Nyata

### Penanganan Trafik Piala Dunia pada Layanan Streaming BBC iPlayer

- **Tantangan Rekayasa Riil**:
  Saat pertandingan final Piala Dunia disiarkan secara langsung, jutaan penonton mengakses live streaming video secara bersamaan. Lonjakan trafik mencapai puluhan Terabit per detik (Tbps), jauh melampaui kapasitas pusat data mana pun di dunia.
- **Implementasi Solusi Industri**:
  BBC menerapkan strategi **Multi-CDN Architecture** (menggabungkan Fastly, Akamai, dan AWS CloudFront):
  1. Segmen video streaming dipecah menjadi fragmen-fragmen kecil 2 detik (*HLS chunks*).
  2. Fragmen video di-cache di ribuan edge server CDN yang tersebar di setiap kota.
  3. Menggunakan algoritma penyeimbang beban dinamis (*Global Traffic Management*) untuk mengalihkan rute trafik jika salah satu provider CDN mengalami kejenuhan jaringan.
- **Dampak & Hasil Sistem**:
  - 98,5% lalu lintas video dilayani langsung dari server edge CDN terdekat.
  - Pusat data asal BBC beroperasi dengan normal tanpa pernah mengalami downtime sepanjang turnamen berlangsung.

## 10. Fakta Menarik & Gotchas Production

1. **Jebakan `s-maxage` vs `max-age`**:
   - `max-age`: Mengatur durasi cache untuk peramban klien (*browser cache*).
   - `s-maxage` (*Shared Max Age*): Mengatur durasi cache **khusus untuk server proxy perantara dan CDN**. Ini memungkinkan Anda menyuruh browser me-revalidate setiap saat (`max-age=0`), tetapi menyuruh CDN menyimpan file selama 1 hari (`s-maxage=86400`)!
2. **Kekuatan Asset Hashing**:
   Jangan pernah memberi nama file statis seperti `main.js` atau `style.css`. Selalu gunakan bundler yang menyuntikkan hash konten unik ke nama file: `main.a9f1b2.js`. Dengan teknik ini, Anda dapat menyetel masa cache CDN selama **1 tahun penuh** tanpa takut pengguna menerima versi lama saat ada rilis baru!

## 11. Rangkuman Singkat

1. **Dekatkan Konten**: CDN mendistribusikan berkas web ke ratusan server edge di seluruh dunia untuk memangkas latensi jarak geografis.
2. **Benteng Pelindung Origin**: Menyerap hingga 95% trafik dan melindungi infrastruktur backend dari kelumpuhan lonjakan trafik atau serangan DDoS.
3. **Kombinasi Hash & Immutable**: Beri hash unik pada nama berkas aset dan pasang `Cache-Control: public, max-age=31536000, immutable` untuk performa maksimal.
4. **Pisahkan Data Privat**: Jangan pernah mengizinkan CDN menyimpan cache data personal akun pengguna (`Cache-Control: private`).
