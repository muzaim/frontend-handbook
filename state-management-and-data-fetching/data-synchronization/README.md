# Data Synchronization

## 1. Skenario Sistem Nyata

Pada aplikasi kolaboratif manajemen proyek (seperti Jira atau Trello board), seorang Product Manager sedang membuka tiket pekerjaan "Fitur Login v2" di laptopnya di kantor. Di saat yang sama, seorang Software Engineer di tempat lain sedang mengedit deskripsi dan memindahkan status tiket yang sama dari **"In Progress"** menjadi **"Code Review"**.

Jika aplikasi frontend hanya mengandalkan pengambilan data statis satu kali saat halaman dimuat:
- Layar laptop Product Manager tetap menampilkan status lama "In Progress".
- Setengah jam kemudian, Product Manager tersebut mengetik catatan di tiket dan menekan tombol "Simpan", secara tidak sengaja menimpa perubahan deskripsi dan memundurkan status tiket yang sudah dikerjakan oleh engineer tadi (*Blind Overwrite / Lost Update Problem*).
- Terjadi konflik data serius, waktu kerja tim terbuang, dan komunikasi proyek menjadi kacau.

Solusi: Menerapkan **Data Synchronization (Sinkronisasi Data Real-Time)**. Menghubungkan aplikasi frontend dengan server menggunakan strategi sinkronisasi aktif: **Polling**, **Long-Polling**, **Server-Sent Events (SSE)**, **WebSockets**, atau algoritma kolaboratif (seperti CRDT / OT). Perubahan data di server langsung terdorong (*pushed*) ke layar seluruh pengguna yang sedang aktif melihat dokumen tersebut.

## 2. Apa Itu dan Buat Apa?

Data Synchronization adalah serangkaian mekanisme dan protokol arsitektur yang bertugas **menjaga konsistensi data antara penyimpanan lokal frontend (browser memory/cache) dan sumber kebenaran data di server backend (database)** secara berkelanjutan seiring berjalannya waktu.

Empat spektrum utama strategi sinkronisasi data di frontend:
1. **Short Polling**: Client mengirim request HTTP secara berkala di latar belakang (misal setiap 5 detik). Paling sederhana, namun boros bandwidth jika tidak ada data baru.
2. **Server-Sent Events (SSE)**: Koneksi HTTP persisten satu arah (*one-way stream*) di mana server dapat mendorong data baru ke browser kapan saja secara efisien.
3. **WebSockets**: Saluran komunikasi dua arah (*full-duplex*) berlatensi sub-milidetik untuk aplikasi real-time interaktif (Chat, Game, Trading).
4. **CRDTs (Conflict-free Replicated Data Types)**: Struktur data matematika canggih yang memungkinkan banyak pengguna mengedit dokumen secara bersamaan (*multiplayer editing* seperti Google Docs atau Figma) dan secara otomatis menyelesaikan konflik tanpa lock server.

Fungsi utama Data Synchronization:
- Mencegah data basi (*stale data*) pada lingkungan multipengguna.
- Mengeliminasi masalah konflik penimpaan data (*Lost Updates*).
- Mendukung aplikasi berkemampuan luring (*Offline-First Architecture*).

## 3. Kenapa Harus Pakai Ini?

Aplikasi web modern tidak lagi beroperasi sebagai dokumen statis yang dibaca sendirian. Aplikasi web saat ini adalah ruang kerja bersama (*collaborative workspaces*).

Tanpa sinkronisasi data yang dirancang dengan matang:
1. Pengguna akan sering melihat data yang sudah kedaluwarsa.
2. Terjadi tabrakan data saat dua orang menyimpan perubahan pada waktu yang berdekatan.
3. Aplikasi tidak dapat beroperasi secara andal di perangkat mobile yang sering mengalami gangguan sinyal.

## 4. Cara Kerjanya (Diagram Alur Spektrum Sinkronisasi)

Perbandingan model tarikan (*Pull / Polling*) vs model dorongan (*Push / SSE & WebSocket*):

```
MODEL 1: SHORT POLLING (Pull - Boros Request)
[Browser] --- Ada data baru? ---> [Server] --> "Tidak ada"
(Jeda 5s)
[Browser] --- Ada data baru? ---> [Server] --> "Tidak ada"
(Jeda 5s)
[Browser] --- Ada data baru? ---> [Server] --> "Ya! Ini ada 1 update"

----------------------------------------------------------------------

MODEL 2: SERVER-SENT EVENTS / WEBSOCKET (Push - Real-Time Efisien)
[Browser] === 1. Buka Koneksi Persisten (Handshake) ===> [Server]
     |                                                       |
     | [Koneksi Tetap Terbuka Secara Hemat Daya]             |
     |                                                       |
     | <====== 2. PUSH: Data Diubah oleh User Lain! <========+
     v
[Browser langsung perbarui UI dalam hitungan milidetik!]
```

## 5. Cuplikan Logika Inti

### Sinkronisasi Real-Time Menggunakan Server-Sent Events (SSE)

Mendengarkan aliran data dari server menggunakan Web API native `EventSource`:

```javascript
// hooks/useTicketSync.js
import { useEffect } from "react";
import { useQueryClient } from "@tanstack/react-query";

export function useTicketSync(ticketId) {
  const queryClient = useQueryClient();

  useEffect(() => {
    // 1. Buka saluran koneksi Server-Sent Events ke backend
    const eventSource = new EventSource(`/api/tickets/${ticketId}/stream`);

    // 2. Tangkap event saat ada mutasi data dari pengguna lain di server
    eventSource.addEventListener("ticket_updated", (event) => {
      const updatedData = JSON.parse(event.data);
      console.log("Menerima pembaruan data real-time:", updatedData);

      // 3. Perbarui cache TanStack Query secara langsung tanpa perlu request ulang!
      queryClient.setQueryData(["ticket", ticketId], updatedData);
    });

    eventSource.onerror = (err) => {
      console.error("Koneksi SSE terputus, browser akan mencoba reconnect otomatis...", err);
    };

    // 4. Bersihkan koneksi saat komponen unmount
    return () => {
      eventSource.close();
    };
  }, [ticketId, queryClient]);
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan
- **Integritas Data Sempurna**: Seluruh pengguna melihat versi data yang sama secara serentak.
- **Kolaborasi Tanpa Hambatan**: Memungkinkan fitur kolaborasi canggih (melihat kursor mouse orang lain, indikator siapa yang sedang mengetik).
- **Pengurangan Traffic Redundan**: Model push (SSE/WebSocket) menghilangkan jutaan query polling kosong ke server backend.

### Kekurangan & Kompleksitas
- **Beban Koneksi Persisten Server**: Menjaga 100.000 koneksi WebSocket / SSE terbuka bersamaan membutuhkan arsitektur server backend berskala besar (memori RAM dan socket descriptor).
- **Penanganan Reconnect & Paket Tertinggal**: Jika koneksi seluler ponsel terputus selama 30 detik saat melewati terowongan, aplikasi harus memiliki logika untuk meminta data yang terlewat selama durasi putus tersebut (*catch-up synchronization*).

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Menggunakan Short Polling
- Data yang jarang berubah (misal memeriksa status proses export PDF yang memakan waktu 30 detik).
- Sistem backend sederhana yang tidak mendukung infrastruktur WebSocket atau SSE.

### Kapan Menggunakan Server-Sent Events (SSE)
- Notifikasi pembaruan feed, papan status pesanan makanan, live score pertandingan olahraga, dashboard harga pasar (komunikasi satu arah dari server ke client).

### Kapan Menggunakan WebSockets
- Aplikasi obrolan instan (Chatting), game online multipengguna, kanvas desain kolaboratif dua arah.

## 8. Metrik Kunci & Observabilitas

- **Connection Health & Reconnect Count**: Pantau seberapa sering koneksi WebSocket/SSE terputus dan melakukan rekoneksi ulang pada perangkat pengguna.
- **Sync Lag (Jeda Waktu Sinkronisasi)**: Selisih waktu (*timestamp delta*) antara saat data diubah di database server dengan saat data selesai ditampilkan di layar browser client (target: **< 100 ms**).

## 9. Studi Kasus Industri Nyata

### Mesin Sinkronisasi Kolaboratif Figma (Multiplayer Engine)

- **Tantangan Rekayasa Riil**:
  Figma memungkinkan 20 desainer mengedit satu kanvas dokumen yang sama secara bersamaan. Jika desainer A memindahkan lingkaran merah dan desainer B mengubah warnanya di detik yang sama, dokumen tidak boleh mengalami konflik atau menimpa pekerjaan salah satu pihak.
- **Implementasi Solusi Industri**:
  Figma membangun mesin sinkronisasi berbasis WebSockets yang dipadukan dengan pohon scene grafis terdistribusi:
  1. Perubahan dimodelkan sebagai operasi mutasi kecil (*atomic operations*).
  2. Server bertindak sebagai sequencer terpusat yang menetapkan urutan operasi secara deterministik.
  3. Client menerapkan mutasi secara instan lokal, lalu menyelaraskan posisi objek saat ack dari server diterima.
- **Dampak & Hasil Sistem**:
  - Kolaborasi kanvas terasa secepat aplikasi offline lokal tanpa pernah ada konflik file simpan ganda.

## 10. Fakta Menarik & Gotchas Production

1. **Batas Maksimal Koneksi SSE di HTTP/1.1**:
   Jika server Anda masih berjalan di protokol **HTTP/1.1**, browser membatasi maksimal hanya **6 koneksi persisten SSE per domain**! Jika pengguna membuka 6 tab browser berbeda, tab ke-7 akan macet total tidak bisa memuat data apa pun.  
   *Solusi*: Wajib gunakan **HTTP/2** atau **HTTP/3**, di mana batas koneksi di-multiplexing menjadi ratusan stream dalam 1 koneksi TCP tunggal.
2. **Kelebihan SSE Dibanding WebSockets**:
   Banyak developer langsung meloncat menggunakan WebSockets padahal hanya butuh aliran data satu arah. SSE berjalan di atas protokol HTTP standar, melewati firewall/proxy perusahaan dengan mulus tanpa konfigurasi khusus, dan memiliki fitur **Automatic Reconnection bawaan** di browser tanpa satu baris kode pun!

## 11. Rangkuman Singkat

1. **Kekompakan Data**: Sinkronisasi data menjamin status di browser selalu konsisten dengan data riil di server.
2. **Pilih Spektrum Sesuai Kebutuhan**: Gunakan Polling untuk kasus sederhana, SSE untuk aliran data satu arah, dan WebSockets untuk interaksi dua arah real-time.
3. **Waspadai Kasus Offline**: Siapkan mekanisme penangkap data tertinggal saat koneksi internet client sempat terputus.
4. **Cegah Lost Updates**: Jangan biarkan client menimpa seluruh objek data secara buta tanpa mencocokkan nomor versi data (*version check / optimistic locking*).
