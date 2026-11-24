# Optimistic Updates

## 1. Skenario Sistem Nyata

Pada aplikasi media sosial (seperti Twitter/X atau Instagram Web), seorang pengguna sedang berada di dalam kereta komuter bawah tanah dengan sinyal internet seluler yang tidak stabil (*spotty 3G network*). Pengguna melihat sebuah postingan lucu dan mengklik tombol ikon hati (**"Like"**).

Jika aplikasi menggunakan pendekatan konvensional (Pessimistic UI):
- Setelah tombol diklik, ikon hati menampilkan ikon loading spinner kecil berputar-putar.
- Aplikasi menunggu request HTTP POST berjalan ke server melalui sinyal kereta yang lambat (memakan waktu 2,5 detik).
- Baru setelah server membalas dengan status 200 OK, ikon hati berubah warna menjadi merah dan angka like bertambah.
- Pengalaman pengguna terasa kaku, lambat, dan membosankan. Jika pengguna mengklik like pada 10 postingan berurutan, halaman dipenuhi oleh belasan spinner berputar.

Solusi: Menerapkan **Optimistic Updates (Pembaruan Optimistis)**. Aplikasi **berasumsi secara optimistis bahwa request ke server 99% akan berhasil**. Begitu pengguna mengklik tombol like, ikon hati **LANGSUNG berubah menjadi merah dan angka like bertambah dalam waktu 0 milidetik**. Request ke server dikirimkan di latar belakang secara hening. Jika ternyata koneksi gagal (request error), aplikasi secara anggun membatalkan perubahan tersebut (*rollback*) ke kondisi semula dan menampilkan notifikasi peringatan singkat.

## 2. Apa Itu dan Buat Apa?

Optimistic Updates adalah pola arsitektur antarmuka pengguna di mana aplikasi memperbarui tampilan UI secara instan **sebelum** server backend mengonfirmasi keberhasilan operasi mutasi data tersebut.

Alih-alih menahan pengguna dalam status loading menunggu respon jaringan, aplikasi langsung menampilkan hasil akhir yang diharapkan, menciptakan ilusi bahwa aplikasi berjalan dengan kecepatan kilat (*zero-latency perceived performance*).

Tiga pilar dalam siklus Optimistic Update yang tangguh:
1. **Mutasi Optimistis Segera**: Memperbarui status UI lokal / cache instan saat aksi dipicu.
2. **Snapshot Cadangan (*Rollback State*)**: Menyimpan salinan data lama sebelum dimutasi.
3. **Penyelarasan Akhir (*Reconciliation / Rollback*)**: Jika request server sukses, konfirmasi data final; jika request gagal, kembalikan data menggunakan snapshot cadangan (*rollback*).

Fungsi utama Optimistic Updates:
- **Perceived Performance Maksimal**: Menghilangkan waktu tunggu interaksi pada aksi-aksi mikro yang sering dilakukan pengguna.
- **Pengalaman Mirip Aplikasi Desktop Native**: Memberikan umpan balik instan pada tombol interaktif (Like, Bookmark, Checkbox Todo, Upvote).

## 3. Kenapa Harus Pakai Ini?

Hukum fisika jaringan menyatakan bahwa latensi jaringan seluler (*Round-Trip Time / RTT*) selalu membutuhkan waktu antara 50 ms hingga 1.500 ms tergantung jarak geografis dan kualitas pemancar seluler.

Manusia mempersepsikan interaksi antarmuka sebagai "instan" hanya jika umpan balik visual terjadi dalam waktu **< 100 milidetik**. Menunggu konfirmasi server untuk setiap klik like atau centang todo akan selalu terasa lambat bagi persepsi manusia. Optimistic Updates menjembatani kesenjangan fisik ini.

## 4. Cara Kerjanya (Diagram Alur Happy Path vs Rollback Path)

```
Pengguna Mengklik Tombol "Like Post"
                 |
                 v
+---------------------------------------------------------------+
|                      LANGKAH 1: OPTIMISTIC                     |
|  1. Simpan snapshot data lama: { likes: 10, isLiked: false }  |
|  2. LANGSUNG UBAH UI:          { likes: 11, isLiked: true }   |
|  *PENGGUNA MELIHAT HATI MERAH DALAM 0 MILIDETIK!*             |
+---------------------------------------------------------------+
                                 |
                                 v
+---------------------------------------------------------------+
|                      LANGKAH 2: REQUEST JARINGAN              |
|              Kirim: POST /api/posts/42/like ke Server         |
+---------------------------------------------------------------+
                                 |
         +-----------------------+-----------------------+
         |                                               |
         | (KASUS A: SUKSES 200 OK)                      | (KASUS B: GAGAL / ERROR 500)
         v                                               v
+---------------------------------+             +---------------------------------+
|          HAPPY PATH             |             |          ROLLBACK PATH          |
|  - Konfirmasi data di cache     |             |  1. Batalkan perubahan:         |
|  - Selesai! Tidak ada kedipan   |             |     Kembalikan snapshot lama    |
|    UI sama sekali.              |             |     { likes: 10, isLiked: false}|
|                                 |             |  2. Munculkan pesan Toast:      |
|                                 |             |     "Gagal menyukai postingan." |
+---------------------------------+             +---------------------------------+
```

## 5. Cuplikan Logika Inti

### Implementasi Optimistic Update yang Aman Menggunakan TanStack Query

Contoh penanganan checklist Todo list dengan snapshot rollback otomatis:

```typescript
import { useMutation, useQueryClient } from "@tanstack/react-query";

interface Todo {
  id: number;
  title: string;
  completed: boolean;
}

export function useToggleTodo() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: async ({ id, completed }: { id: number; completed: boolean }) => {
      const res = await fetch(`/api/todos/${id}`, {
        method: "PATCH",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ completed }),
      });
      if (!res.ok) throw new Error("Gagal menyimpan ke server");
      return res.json();
    },

    // 1. DIJALANKAN SEBELUM REQUEST JARINGAN DIKIRIM (ON MUTATE)
    onMutate: async (newTodoData) => {
      // Batalkan query yang sedang berjalan agar tidak menimpa state optimistik kita
      await queryClient.cancelQueries({ queryKey: ["todos"] });

      // Ambil snapshot data lama sebagai cadangan untuk rollback
      const previousTodos = queryClient.getQueryData<Todo[]>(["todos"]);

      // Perbarui cache lokal secara instan (Optimistic Update)!
      queryClient.setQueryData<Todo[]>(["todos"], (old) =>
        old?.map((todo) =>
          todo.id === newTodoData.id ? { ...todo, completed: newTodoData.completed } : todo
        )
      );

      // Kembalikan konteks yang berisi snapshot lama
      return { previousTodos };
    },

    // 2. JIKA REQUEST SERVER GAGAL (ON ERROR)
    onError: (err, newTodoData, context) => {
      // ROLLBACK: Kembalikan cache ke data sebelum mutasi optimistik dilakukan!
      if (context?.previousTodos) {
        queryClient.setQueryData(["todos"], context.previousTodos);
      }
      alert("Gagal memperbarui status. Perubahan dibatalkan.");
    },

    // 3. DIJALANKAN DI AKHIR (BAIK SUKSES MAUPUN GAGAL)
    onSettled: () => {
      // Sinkronkan kembali dengan database riil server untuk menjamin konsistensi
      queryClient.invalidateQueries({ queryKey: ["todos"] });
    },
  });
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan
- **Kecepatan Reaksi Sempurna**: Menghilangkan rasa lag pada interaksi pengguna berfrekuensi tinggi.
- **Kenyamanan Visual**: Tidak ada animasi spinner kecil yang mengganggu konsentrasi visual.
- **Mendukung Offline First**: Aplikasi terasa tetap berfungsi meskipun sinyal internet sedang terputus sesaat.

### Kekurangan & Risiko
- **Kompleksitas Logika Rollback**: Menulis logika pembatalan state secara benar membutuhkan kode tambahan yang teliti.
- **Kebingungan Pengguna Saat Rollback**: Jika pengguna melihat tombol sudah tercentang, tetapi 3 detik kemudian tiba-tiba kembali tidak tercentang karena error jaringan, pengguna bisa merasa bingung jika tidak ada notifikasi yang jelas.

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Pakai
- Interaksi bervolume tinggi dengan tingkat kegagalan server rendah (< 0.1%): Tombol Like, Bookmark, Follow/Unfollow, Star rating.
- Checklist to-do item, arsip email, penandaan pesan sudah dibaca (*mark as read*).
- Menambah atau menghapus komentar sederhana.

### Kapan JANGAN PERNAH Menggunakan Optimistic Updates
- **Transaksi Finansial & Pembayaran**: Jangan pernah mengubah saldo dompet digital pengguna atau menampilkan status "Pembayaran Berhasil" sebelum server bank mengonfirmasinya secara sah!
- Operasi destruktif berisiko tinggi (misal: "Hapus Akun Permanen" atau "Format Database").
- Formulir dengan validasi server yang rumit (misal: transfer dana dengan validasi PIN atau OTP).

## 8. Metrik Kunci & Observabilitas

- **Rollback Rate (Persentase Rollback)**: Pantau rasio mutasi optimistik yang mengalami rollback di analytics frontend:
  `Rollback Rate = (Total Rollbacks / Total Optimistic Mutations) * 100%`. Standar sehat di production adalah **< 0.5%**. Jika angka melonjak ke > 5%, artinya ada gangguan pada endpoint backend.
- **Interaction to Next Paint (INP)**: Optimistic update menjamin pembaruan visual tombol terjadi seketika di bawah **16 ms**, menghasilkan skor INP yang sempurna (*Good* hijau).

## 9. Studi Kasus Industri Nyata

### Pengalaman Mengirim Pesan Instan di WhatsApp Web / Slack

- **Tantangan Rekayasa Riil**:
  Ketika pengguna menekan tombol "Kirim" pada aplikasi chat, jika pesan baru muncul di layar setelah server menerima dan menyimpan pesan tersebut, percakapan akan terasa tersendat dan lambat.
- **Implementasi Solusi Industri**:
  Aplikasi chat menerapkan **Optimistic Message Insertion**:
  1. Begitu tombol Enter ditekan, balon pesan langsung disuntikkan ke tampilan chat lokal dengan ikon jam kecil (*Clock Icon: Sending*).
  2. Begitu server membalas dengan ack, ikon jam berganti menjadi centang satu (*Sent*).
  3. Jika koneksi timeout, ikon berganti menjadi tanda seru merah (*Failed*) dengan tombol "Coba Lagi" (*Retry*).
- **Dampak & Hasil Sistem**:
  - Pengetikan dan alur obrolan terasa instan dan alami bagi pengguna.
  - Pengguna tidak pernah kehilangan teks pesan yang sudah diketik jika sinyal internet tiba-tiba hilang.

## 10. Fakta Menarik & Gotchas Production

1. **Jebakan ID Sementara (*Temporary Client IDs*)**:
   Saat Anda membuat item baru secara optimistik (misal menambah baris komentar baru), item tersebut belum memiliki `id` dari database server. Jika pengguna langsung mencoba mengklik "Hapus" pada komentar baru tersebut sebelum server membalas, aplikasi akan bingung karena `id` belum ada!  
   *Solusi*: Buat UUID acak sementara di client (`crypto.randomUUID()`), lalu gantikan dengan ID database asli saat server membalas.
2. **Kekacauan Urutan (*Out-of-Order Updates*)**:
   Jika pengguna mengklik toggle aktif-nonaktif secara sangat cepat 5 kali berturut-turut, request ke server bisa tiba dalam urutan yang terbalik karena latensi jaringan. Menangani hal ini membutuhkan pembatalan request lama (*AbortController*) di setiap mutasi baru.

## 11. Rangkuman Singkat

1. **Kecepatan Instan**: Optimistic updates mengubah UI seketika dengan asumsi operasi jaringan akan berhasil.
2. **Wajib Miliki Rollback**: Selalu simpan snapshot data lama untuk mengembalikan tampilan jika server menolak request.
3. **Pilih Fitur yang Tepat**: Sempurna untuk like, bookmark, dan checklist; haram untuk pembayaran dan transaksi finansial.
4. **Beri Notifikasi Saat Gagal**: Jika terjadi rollback, selalu tampilkan pesan error yang jelas agar pengguna tidak bingung.
