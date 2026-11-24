# Server-Sent Events (SSE)

## 1. Skenario Sistem Nyata

Pada aplikasi pelacak status pengiriman makanan online (seperti GrabFood atau GoFood Web), pengguna sedang menunggu kurir mengantarkan makanan. Status pengiriman diperbarui dalam beberapa tahap: *"Pesanan Diterima Restoran"*, *"Sedang Dimasak"*, *"Makanan Diambil Pengemudi"*, dan *"Kurir Menuju Lokasi Anda"*.

Aliran data ini murni bersifat **satu arah**: server yang memberi tahu klien saat ada kemajuan status; pengguna tidak pernah mengirimkan perintah apa pun dari halaman tersebut.

Developer junior memutuskan menggunakan protokol WebSocket yang rumit:
- Membangun server WebSocket terpisah, mengelola jabat tangan biner, mengurus protokol ping/pong kustom, dan menangani load balancing socket yang rumit di backend.
- Ketika pengguna berada di jaringan kantor perusahaan yang memasang proxy ketat, koneksi WebSocket diblokir oleh firewall perusahaan karena dianggap lalu lintas non-standar.
- Ketika koneksi seluler terputus di lift, developer harus menulis puluhan baris kode manual untuk rekoneksi dan melacak event apa saja yang terlewat.

Solusi: Menggunakan **Server-Sent Events (SSE)** via Web API native **`EventSource`**. Berjalan murni di atas protokol HTTP standar (`Content-Type: text/event-stream`), melewati proxy firewall perusahaan dengan mulus tanpa hambatan, memiliki fitur **Automatic Reconnection bawaan**, dan secara otomatis menyertakan header `Last-Event-ID` untuk mengambil data yang terlewat tanpa satu baris kode tambahan pun di frontend.

## 2. Apa Itu dan Buat Apa?

Server-Sent Events (SSE) adalah teknologi standar web W3C dan HTML5 yang memungkinkan server web mengirimkan aliran data teks secara berkelanjutan (**One-Way Push Streaming**) ke peramban klien melalui satu koneksi HTTP standar yang tetap terbuka (*persistent HTTP connection*).

Pada SSE:
- Komunikasi bersifat **Satu Arah (Server-ke-Klien)**. Klien membuka koneksi sekali, lalu server dapat menembakkan (*push*) data kapan saja secara asinkron.
- Berjalan di atas protokol HTTP/HTTPS biasa dengan format media tipe khusus: **`text/event-stream`**.
- Data dikirimkan dalam format teks UTF-8 polos dengan struktur sederhana: `id`, `event`, dan `data`.
- Peramban web menyediakan antarmuka native yang sangat elegan bernama **`EventSource`**.

Fungsi utama Server-Sent Events:
- Menghadirkan pembaruan data real-time satu arah tanpa kompleksitas infrastruktur WebSocket.
- Menyediakan aliran respon teks kecerdasan buatan (*AI Token Streaming*, seperti yang digunakan oleh ChatGPT / Claude).
- Menyajikan notifikasi aplikasi, papan skor langsung (*live scores*), dan pembaruan status sistem.

## 3. Kenapa Harus Pakai Ini?

Banyak developer mengira bahwa setiap fitur yang membutuhkan kata "real-time" harus menggunakan WebSocket. **Ini adalah kesalahan arsitektur yang umum**.

Jika aplikasi Anda hanya membutuhkan pembaruan satu arah dari server ke browser:
1. **Jauh Lebih Sederhana**: Anda tidak perlu memikirkan framing biner atau server socket khusus; endpoint backend-nya hanyalah endpoint HTTP standar yang tidak menutup respons (`Transfer-Encoding: chunked`).
2. **Rekoneksi Otomatis Native**: Jika kabel LAN dicabut atau sinyal ponsel mati, peramban secara otomatis mencoba menghubungkan kembali (*auto-reconnect*) setiap 3 detik tanpa butuh library pihak ketiga.
3. **Mendukung Fitur Resume Otomatis**: Browser secara otomatis mengirimkan header `Last-Event-ID` saat rekoneksi, memberitahu server ID pesan terakhir yang diterimanya sehingga server dapat mengirimkan pesan yang sempat tertinggal.

## 4. Cara Kerjanya (Diagram Alur Format Stream Teks SSE)

Struktur aliran data teks di dalam koneksi HTTP:

```
[ Browser Klien ] ---------------- GET /api/live-stream ----------------> [ Server ]
Headers:
  Accept: text/event-stream
                                                                              |
                                                          Server membalas &   |
                                                          koneksi TETAP DIBUKA|
                                                                              |
[ Browser Klien ] <--- HTTP/1.1 200 OK ---------------------------------------+
Headers:
  Content-Type: text/event-stream
  Cache-Control: no-cache
  Connection: keep-alive

----------------------------------------------------------------------

ALIRAN DATA TEKS YANG DITERIMA BROWSER DARI SERVER SECARA BERKALA:

id: 101\n
event: order_status\n
data: {"status": "Sedang Dimasak"}\n\n

(Jeda 10 detik hening... Koneksi HTTP tetap menyala tenang)

id: 102\n
event: order_status\n
data: {"status": "Diambil Pengemudi", "driver": "Budi"}\n\n

*SETIAP DUA BARIS ENTER (\n\n) MEMICU EVENT LISTENER DI BROWSER!*
```

## 5. Cuplikan Logika Inti

### Mengonsumsi Server-Sent Events Menggunakan API Native `EventSource`

Implementasi pemantauan status pesanan di sisi frontend:

```typescript
// services/orderTrackingSSE.ts

export function trackOrderStatus(
  orderId: string,
  onStatusUpdate: (data: any) => void,
  onError: (err: any) => void
) {
  // 1. Inisialisasi koneksi SSE ke server
  const eventSource = new EventSource(`/api/orders/${orderId}/stream`);

  // 2. Mendengarkan event kustom bernama 'order_status'
  eventSource.addEventListener("order_status", (event: MessageEvent) => {
    try {
      const data = JSON.parse(event.data);
      console.log(`Menerima ID Pesan #${event.lastEventId}:`, data);
      onStatusUpdate(data);

      // Jika pesanan sudah selesai, tutup koneksi secara manual
      if (data.status === "Selesai") {
        console.log("Pesanan selesai, menutup koneksi SSE.");
        eventSource.close();
      }
    } catch (err) {
      console.error("Gagal mem-parsing payload data SSE:", err);
    }
  });

  // 3. Mendengarkan event default biasa (tanpa nama event kustom)
  eventSource.onmessage = (event: MessageEvent) => {
    console.log("Pesan umum:", event.data);
  };

  // 4. Penanganan error koneksi (Browser otomatis melakukan rekoneksi di latar belakang)
  eventSource.onerror = (error) => {
    console.warn("Koneksi SSE mengalami gangguan, browser mencoba menyambung ulang...", error);
    onError(error);
  };

  // 5. Mengembalikan fungsi cleanup untuk dipanggil saat komponen unmount
  return () => {
    console.log("Membersihkan koneksi SSE...");
    eventSource.close();
  };
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan
- **Ramah Firewall & Proxy**: Berjalan di atas HTTP biasa (port 80/443), tidak pernah diblokir oleh proxy kantor atau filter keamanan sekolah.
- **Kecerdasan Bawaan Browser**: Fitur rekoneksi otomatis dan pelacakan `Last-Event-ID` sudah ditanam di tingkat C++ browser.
- **Konsumsi Baterai Rendah**: Jauh lebih hemat daya pada smartphone dibandingkan menjaga socket WebSocket terbuka.

### Kekurangan & Keterbatasan
- **Hanya Satu Arah (Mono-directional)**: Klien tidak bisa mengirimkan data melalui pipa SSE ini. Jika klien ingin membalas, klien harus mengirimkan request HTTP POST terpisah.
- **Batasan 6 Koneksi pada HTTP/1.1**: Pada HTTP/1.1 lama, browser membatasi maksimal 6 koneksi per domain. Jika pengguna membuka 6 tab yang sama-sama menjalankan SSE, tab ke-7 akan macet total! (Solusi: Wajib aktifkan **HTTP/2** atau **HTTP/3** di mana koneksi di-multiplexing).
- **Ketiadaan Header Kustom pada `EventSource` Native**: API `EventSource` bawaan browser tidak mengizinkan kita menyuntikkan header Authorization kustom (seperti `Authorization: Bearer token`). Sebagai gantinya, token harus dilewatkan via query parameter (`/stream?token=...`) atau menggunakan cookie `HttpOnly`.

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Memilih Server-Sent Events (SSE)
- Tampilan teks generatif AI (*ChatGPT-style text token streaming*).
- Sistem notifikasi antarmuka pengguna (notifikasi lonceng di navbar).
- Dashboard analitik pemantauan log server atau status pipeline deployment CI/CD.
- Pelacakan pesanan e-commerce atau live ticker skor pertandingan olahraga.

### Kapan Harus Memilih WebSocket
- Aplikasi percakapan instan dua arah (Chatting di mana pengguna mengetik dan menerima pesan terus-menerus).
- Game online multiplayer berbasis browser yang membutuhkan transmisi data puluhan frame per detik.

## 8. Metrik Kunci & Observabilitas

- **SSE Stream Inspection (Chrome DevTools)**: Buka tab *Network*, klik request endpoint SSE Anda, lalu pilih tab **EventStream**. Anda dapat melihat baris event `Id`, `Type`, `Data`, dan `Time` yang masuk secara real-time.
- **Open Connection Count**: Memantau jumlah koneksi persisten SSE yang aktif di server reverse proxy (Nginx).

## 9. Studi Kasus Industri Nyata

### Arsitektur Streaming Respon Teks pada OpenAI ChatGPT Web

- **Tantangan Rekayasa Riil**:
  Ketika pengguna mengirimkan pertanyaan ke ChatGPT, model bahasa kecerdasan buatan (LLM) membutuhkan waktu 10 hingga 30 detik untuk menghasilkan jawaban lengkap ratusan kata. Jika antarmuka web menunggu seluruh jawaban selesai sebelum mengembalikan respon, pengguna akan melihat layar loading berputar selama 30 detik tanpa kepastian.
- **Implementasi Solusi Industri**:
  OpenAI menggunakan **Server-Sent Events (SSE)**:
  1. Klien mengirim pertanyaan via HTTP POST standar.
  2. Server AI mengembalikan respon sebagai aliran `text/event-stream`.
  3. Setiap kata (*token*) yang selesai diproses langsung didorong ke browser: `data: {"delta": "Halo"}`, `data: {"delta": " dunia"}`.
  4. Antarmuka web ChatGPT menangkap aliran data teks tersebut dan menampilkannya dengan efek mengetik instan di layar.
- **Dampak & Hasil Sistem**:
  - Time to First Token (TTFT) terasa sangat cepat dalam hitungan ratusan milidetik.
  - Pengguna merasa sistem sangat cerdas dan responsif karena teks muncul secara mengalir.

## 10. Fakta Menarik & Gotchas Production

1. **Jebakan Buffer di Balik Nginx Proxy**:
   Banyak developer kebingungan: *"Saya sudah pakai SSE, kenapa kata-katanya tidak mengalir lancar melainkan keluar sekaligus di akhir?"*. Penyebabnya adalah server reverse proxy (seperti Nginx) secara default **menahan dan mengumpulkan buffer data (*proxy buffering*)** sebelum dikirim ke klien!  
   *Solusi*: Wajib tambahkan konfigurasi Nginx: `proxy_buffering off;` atau kirim header dari backend: `X-Accel-Buffering: no`.
2. **Karakter Pemisah Wajib (`\n\n`)**:
   Spesifikasi SSE mewajibkan setiap paket pesan diakhiri dengan **dua karakter newline berturut-turut (`\n\n`)**. Jika backend Anda hanya mengirim satu `\n`, browser akan mengira pesan belum selesai dan tidak akan memicu event handler sama sekali!

## 11. Rangkuman Singkat

1. **Aliran Satu Arah**: SSE adalah protokol transmisi data real-time satu arah dari server ke browser melalui HTTP standar.
2. **Pilihan Utama AI Streaming**: Standar de-facto untuk menampilkan teks streaming LLM (seperti ChatGPT).
3. **Rekoneksi Otomatis**: Browser secara native menghubungkan kembali koneksi yang putus dan melanjutkan data via `Last-Event-ID`.
4. **Matikan Proxy Buffering**: Selalu pastikan buffer proxy Nginx dimatikan agar data teks terdorong instan tanpa tertahan.
