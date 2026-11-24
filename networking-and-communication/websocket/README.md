# WebSocket

## 1. Skenario Sistem Nyata

Pada aplikasi bursa perdagangan aset kripto dan valuta asing (*Crypto & Forex Exchange*), pengguna memantau pergerakan grafik harga Bitcoin (*Candlestick Chart*) dan antrean buku pesanan (*Order Book*) yang mengalami pembaruan hingga **100 transaksi per detik**.

Developer junior mengimplementasikan pembaruan data menggunakan teknik **HTTP Short Polling**:

```javascript
// Kode bermasalah: Mengirim request HTTP GET setiap 500 milidetik
setInterval(async () => {
  const res = await fetch("/api/v1/ticker/btc-usdt");
  const data = await res.json();
  updateChart(data);
}, 500);
```

Dampak pada sistem:
- Setiap kali request HTTP dikirimkan, peramban harus menyertakan ratusan byte header (Cookies, User-Agent, Accept-Encoding), melakukan jabat tangan TCP, dan menunggu server memproses koneksi.
- 10.000 pengguna aktif yang membuka tab bursa secara bersamaan menghasilkan **20.000 request HTTP per detik** ke server backend, meskipun di milidetik tersebut harga Bitcoin belum tentu bergerak.
- Beban CPU server backend meledak hanya untuk memproses handshake HTTP dan overhead header. Latensi penerimaan data melonjak hingga 800 milidetik. Trader mengeluh kalah cepat dalam mengeksekusi order (*slippage*) karena data harga di layar mereka terlambat hampir satu detik.

Solusi: Mengadopsi protokol **WebSocket**. Membuka satu saluran komunikasi dua arah (*full-duplex*) yang persisten melalui koneksi TCP tunggal. Data harga baru didorong (*pushed*) langsung oleh server ke layar trader dalam hitungan **sub-milidetik** dengan overhead header hanya **2 byte per pesan**, tanpa perlu lagi ada request HTTP berulang.

## 2. Apa Itu dan Buat Apa?

WebSocket adalah protokol komunikasi komputer standar IETF (RFC 6455) dan API peramban W3C yang menyediakan saluran komunikasi **dua arah (*full-duplex*) secara simultan dan persisten** melalui satu koneksi TCP tunggal antara browser klien dan server web.

Berbeda dengan siklus request-response HTTP tradisional di mana klien harus selalu berinisiatif meminta data terlebih dahulu, pada WebSocket:
- Server dapat mengirimkan data ke klien kapan saja secara mandiri (*server-push*).
- Klien dapat mengirimkan data ke server kapan saja secara mandiri.
- Keduanya dapat mengirim dan menerima data di waktu yang sama tanpa saling menunggu.

Koneksi dimulai melalui protokol HTTP standar, lalu di-upgrade (*HTTP Upgrade Handshake*) menjadi koneksi biner WebSocket yang persisten (`ws://` atau `wss://` untuk koneksi terenkripsi TLS).

Fungsi utama WebSocket:
- Menghadirkan komunikasi data instan berlatensi ultra-rendah (*real-time applications*).
- Menghilangkan beban overhead header HTTP pada transmisi data frekuensi tinggi.

## 3. Kenapa Harus Pakai Ini?

Untuk fitur-fitur yang menuntut latensi mendekati waktu nyata (*near-zero latency*):
1. **Efisiensi Paket Jaringan Ekstrem**: Satu pesan WebSocket hanya membutuhkan overhead framing sebesar **2 hingga 10 byte**, berbanding terbalik dengan header HTTP yang memakan 500 hingga 2.000 byte per request.
2. **Koneksi Tetap Terbuka (Persistent)**: Tidak ada overhead 3-way handshake berulang setelah koneksi pertama berhasil dibangun.

## 4. Cara Kerjanya (Diagram Alur Handshake & Full-Duplex Stream)

```
FASE 1: UPGRADE HANDSHAKE (Melalui Protokol HTTP/1.1 Standar)
[ Browser ] ------- GET /ws/market-data ----------------------------> [ Server ]
Headers:
  Upgrade: websocket
  Connection: Upgrade
  Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
  Sec-WebSocket-Version: 13
                                                                          |
                                                      Server memverifikasi kunci
                                                                          |
[ Browser ] <--- HTTP/1.1 101 Switching Protocols ------------------------+
Headers:
  Upgrade: websocket
  Connection: Upgrade
  Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=

----------------------------------------------------------------------

FASE 2: FULL-DUPLEX PERSISTENT STREAM (Protokol WebSocket Resmi)
[ Browser ] <================ SALURAN DUA ARAH TCP =================> [ Server ]
     |                                                                   |
     | <--- PUSH: Harga BTC Berubah ($95.120) [Overhead: 2 Byte!] <------+
     |                                                                   |
     +----> EMIT: Pengguna Pasang Beli 0.5 BTC [Overhead: 2 Byte!] ----->|
     |                                                                   |
     | <--- PUSH: Order Sukses Tereksekusi! -----------------------------+
```

## 5. Cuplikan Logika Inti

### Membangun Klien WebSocket Tangguh dengan Auto-Reconnect & Heartbeat Ping/Pong

Implementasi produksi yang menangani koneksi terputus dan pemulihan otomatis:

```typescript
// services/robustWebSocket.ts

export class RobustWebSocket {
  private ws: WebSocket | null = null;
  private reconnectAttempts = 0;
  private maxReconnectAttempts = 5;
  private heartbeatInterval: number | null = null;
  private isDestroyed = false;

  constructor(
    private url: string,
    private onMessageCallback: (data: any) => void
  ) {
    this.connect();
  }

  private connect() {
    if (this.isDestroyed) return;

    console.log("Menghubungkan ke WebSocket...");
    this.ws = new WebSocket(this.url);

    this.ws.onopen = () => {
      console.log("Koneksi WebSocket terhubung!");
      this.reconnectAttempts = 0; // Reset counter percobaan
      this.startHeartbeat();
    };

    this.ws.onmessage = (event) => {
      try {
        const payload = JSON.parse(event.data);
        // Abaikan respon pong internal
        if (payload.type === "PONG") return;
        this.onMessageCallback(payload);
      } catch (err) {
        console.error("Gagal mem-parsing pesan WS:", err);
      }
    };

    this.ws.onclose = (event) => {
      this.stopHeartbeat();
      if (!this.isDestroyed) {
        console.warn(`WebSocket terputus (Code: ${event.code}). Mencoba rekoneksi...`);
        this.handleReconnect();
      }
    };

    this.ws.onerror = (error) => {
      console.error("Terjadi error pada koneksi WebSocket:", error);
      this.ws?.close();
    };
  }

  // Menjaga koneksi tetap hidup agar tidak diputus oleh firewall / NAT router
  private startHeartbeat() {
    this.heartbeatInterval = window.setInterval(() => {
      if (this.ws?.readyState === WebSocket.OPEN) {
        this.ws.send(JSON.stringify({ type: "PING" }));
      }
    }, 30000); // Kirim detak jantung setiap 30 detik
  }

  private stopHeartbeat() {
    if (this.heartbeatInterval) {
      clearInterval(this.heartbeatInterval);
      this.heartbeatInterval = null;
    }
  }

  // Algoritma Exponential Backoff with Jitter untuk rekoneksi
  private handleReconnect() {
    if (this.reconnectAttempts >= this.maxReconnectAttempts) {
      console.error("Gagal menghubungkan kembali WebSocket setelah batas maksimal.");
      return;
    }

    this.reconnectAttempts++;
    const delay = Math.min(1000 * Math.pow(2, this.reconnectAttempts) + Math.random() * 1000, 30000);
    console.log(`Mencoba rekoneksi dalam ${Math.round(delay)} ms...`);
    setTimeout(() => this.connect(), delay);
  }

  public send(data: any) {
    if (this.ws?.readyState === WebSocket.OPEN) {
      this.ws.send(JSON.stringify(data));
    } else {
      console.warn("Gagal mengirim pesan: WebSocket belum dalam status OPEN.");
    }
  }

  public close() {
    this.isDestroyed = true;
    this.stopHeartbeat();
    this.ws?.close();
  }
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan
- **Latensi Terendah**: Mengirimkan data dalam hitungan milidetik karena koneksi TCP sudah terbuka aktif.
- **Overhead Framing Minimal**: Menghemat bandwidth hingga 95% dibandingkan polling HTTP.
- **Komunikasi Dua Arah Sejati**: Klien dan server dapat saling memicu pesan secara instan.

### Kekurangan & Tantangan Nyata
- **Beban Server Skala Masif (*Stateful Connections*)**: Berbeda dengan HTTP yang bersifat *stateless* (selesai request, koneksi putus), WebSocket menjaga socket tetap terbuka. Menjaga 1.000.000 koneksi WebSocket aktif membutuhkan memori RAM besar dan arsitektur backend terdistribusi yang rumit (menggunakan Redis Pub/Sub antar instance server).
- **Tidak Memiliki Mekanisme HTTP Caching**: Pesan WebSocket tidak dapat di-cache secara otomatis oleh browser atau CDN.
- **Masalah Pemutusan Jaringan Diam-diam (*Zombie Connections*)**: Jika kabel internet dicabut, socket TCP mungkin tidak langsung memicu event `onclose` sampai batas timeout sistem operasi habis, mewajibkan implementasi mekanisme Ping/Pong manual.

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Memilih WebSocket
- Aplikasi perdagangan bursa, saham, kripto (*financial trading platforms*).
- Aplikasi perpesanan instan multipengguna (Chatting, WhatsApp Web, Slack).
- Game online multiplayer berbasis browser.
- Kanvas kolaborasi desain real-time (Miro, Figma multiplayer).

### Kapan JANGAN Menggunakan WebSocket
- Jika data hanya mengalir **satu arah dari server ke klien** (misal notifikasi order pesanan atau live score berita) — gunakan **Server-Sent Events (SSE)** yang jauh lebih hemat daya dan berjalan di atas HTTP standar.
- Pengambilan data reguler biasa (katalog produk, artikel, form profil) — gunakan **REST API** atau **GraphQL**.

## 8. Metrik Kunci & Observabilitas

- **WebSocket Frames (Chrome DevTools)**: Buka tab *Network*, filter opsi **WS**, klik nama koneksi, lalu buka tab *Messages*. Anda dapat melihat seluruh frame pesan masuk (panah hijau) dan pesan keluar (panah merah) secara real-time.
- **Reconnection Frequency**: Pantau berapa kali klien mengalami event `onclose` dan melakukan rekoneksi ulang di monitoring Sentry/Datadog.
- **Message Serialization Latency**: Pastikan waktu pemrosesan `JSON.parse` pada data pesan yang masuk tidak memblokir main-thread lebih dari 10 ms.

## 9. Studi Kasus Industri Nyata

### Penanganan Kolaborasi Kursor Real-Time pada Figma Web

- **Tantangan Rekayasa Riil**:
  Saat beberapa desainer membuka kanvas desain Figma yang sama, posisi kursor mouse masing-masing pengguna harus ditampilkan secara langsung di layar pengguna lain dengan koordinat X dan Y yang bergerak mulus. Polling HTTP akan menenggelamkan server dan menghasilkan gerakan kursor yang patah-patah.
- **Implementasi Solusi Industri**:
  Figma mengimplementasikan koneksi **WebSocket biner murni**:
  1. Koordinat mouse dikompresi ke dalam bentuk biner terstruktur (*ArrayBuffer*) berukuran hanya belasan byte alih-alih string JSON yang boros.
  2. Klien mengirim koordinat dengan interval throttle yang disesuaikan dengan refresh rate layar.
  3. Server mendistribusikan frame biner tersebut ke seluruh klien aktif di ruang kanvas yang sama.
- **Dampak & Hasil Sistem**:
  - Kursor mouse rekan kerja bergerak mulus pada 60 FPS tanpa jeda terasa.
  - Penggunaan bandwidth per pengguna tetap berada di bawah beberapa kilobyte per detik.

## 10. Fakta Menarik & Gotchas Production

1. **Jebakan `ws.send()` saat `CONNECTING`**:
   Jika Anda memanggil `ws.send(data)` tepat setelah membuat `new WebSocket(url)`, JavaScript akan melempar error fatal: *`InvalidStateError: Still in CONNECTING state`*. Anda WAJIB menunggu event `ws.onopen` selesai dipicu sebelum diizinkan mengirim data apa pun!
2. **Koneksi `wss://` vs `ws://`**:
   Selalu gunakan protokol terenkripsi **`wss://` (WebSocket Secure)** di lingkungan produksi. Banyak router Wi-Fi kantor dan penyedia internet (ISP) yang memblokir atau merusak paket `ws://` biasa karena menganggapnya sebagai anomali lalu lintas TCP non-standar.

## 11. Rangkuman Singkat

1. **Dua Arah Instan**: WebSocket menghadirkan komunikasi full-duplex dua arah persisten melalui satu koneksi TCP tunggal.
2. **Overhead Minimal**: Mengurangi beban header dari ribuan byte menjadi hanya 2 byte per pesan.
3. **Wajib Heartbeat**: Selalu pasang mekanisme Ping/Pong berkala untuk menjaga koneksi tidak diputus oleh NAT router atau firewall.
4. **Wajib Auto-Reconnect**: Terapkan algoritma Exponential Backoff untuk memulihkan koneksi yang terputus secara otomatis dan anggun.
