# REST API Integration

## 1. Skenario Sistem Nyata

Pada aplikasi kasir dan pencatatan inventaris toko retail (*Point of Sale / POS*), kasir menekan tombol "Bayar Tagihan" untuk memproses pembayaran pelanggan sebesar Rp 2.500.000.

Developer menulis pemanggilan API menggunakan `fetch()` standar tanpa penanganan batas waktu (*timeout*), tanpa pembatalan request (*AbortController*), dan tanpa standardisasi penanganan error (*error normalization*):

```javascript
// Kode bermasalah: Pemanggilan API naif tanpa batas waktu dan proteksi error
async function handleCheckout(cartData) {
  try {
    const res = await fetch("/api/v1/checkout", {
      method: "POST",
      body: JSON.stringify(cartData),
    });
    const data = await res.json();
    showReceipt(data);
  } catch (err) {
    alert("Terjadi kesalahan!");
  }
}
```

Dampak pada sistem:
- Ketika server backend mengalami kemacetan antrean database (*connection pool deadlock*), request HTTP tersebut menggantung (*hanging*) tanpa batas waktu. Tombol bayar terus berputar selama 3 menit tanpa ada kejelasan status.
- Kasir yang panik mengira sistem tidak merespons, lalu menekan tombol "Bayar" sebanyak 5 kali berturut-turut. Akibat ketiadaan token idempoten dan pembatalan request, kartu kredit pelanggan terdebit 5 kali lipat!
- Lebih buruk lagi, ketika server mengembalikan error 500 dengan payload HTML (misal error page dari Nginx), perintah `res.json()` melempar syntax error *`Unexpected token < in JSON at position 0`*, membuat pesan error yang tampil di layar sangat membingungkan pengguna.

Solusi: Membangun arsitektur **REST API Client Terpusat**: mengonfigurasi batas waktu timeout otomatis menggunakan `AbortSignal.timeout()`, menyuntikkan interceptor untuk logging dan token refresh otomatis, menstandarisasi objek respon error, serta melindungi aksi transaksi dengan header idempotensi.

## 2. Apa Itu dan Buat Apa?

REST API Integration di frontend adalah arsitektur yang mengoordinasikan seluruh interaksi pertukaran data antara aplikasi peramban klien dan server backend melalui arsitektur **Representational State Transfer (REST)** menggunakan metode HTTP standar (`GET`, `POST`, `PUT`, `PATCH`, `DELETE`).

Pilar-pilar utama integrasi API kelas industri:
1. **HTTP Client Terpusat (Centralized Client)**: Menggunakan instance terpusat (Axios atau wrapper Fetch kustom) yang secara otomatis menyertakan `baseURL`, header default, dan timeout.
2. **Interceptors (Pengintersepsi)**:
   - *Request Interceptor*: Menyuntikkan Authorization Bearer Token ke setiap request keluar secara otomatis.
   - *Response Interceptor*: Menangkap status HTTP 401 Unauthorized secara global untuk memicu refresh token otomatis sebelum melempar error ke komponen.
3. **Penyelarasan Batas Waktu (*Timeout & Cancellation*)**: Menggunakan `AbortController` untuk membatalkan request jika server tidak membalas dalam batas waktu wajar (misal 10 detik).
4. **Normalisasi Error (*Error Normalization*)**: Mengonversi variasi error jaringan, error HTTP 4xx, dan error 5xx menjadi satu format objek error seragam yang mudah dibaca oleh antarmuka UI.

## 3. Kenapa Harus Pakai Ini?

Menulis fungsi `fetch()` mentah yang tercecer di puluhan komponen UI yang berbeda adalah resep malapetaka arsitektur:
1. Jika endpoint base URL server berubah, Anda harus mengubah puluhan file.
2. Jika format otentikasi token berganti, Anda harus memperbarui seluruh komponen.
3. Sangat sulit menangani skenario koneksi internet terputus secara seragam.

Membangun modul integrasi API terpusat menciptakan lapisan abstraksi bersih antara logika presentasi UI dan lapisan jaringan (*Network Layer*).

## 4. Cara Kerjanya (Diagram Alur Pipa Request & Response Interceptors)

```
[ Komponen UI ] memanggil: apiClient.get('/orders')
       |
       v
+---------------------------------------------------------------+
|                      REQUEST INTERCEPTOR                      |
|  1. Ambil access token dari memori / store                    |
|  2. Sisipkan: Headers['Authorization'] = 'Bearer eyJhbG...'  |
|  3. Sisipkan: Headers['X-Request-Id'] = UUID unik             |
|  4. Pasang batas waktu: Timeout 10 detik                      |
+---------------------------------------------------------------+
                               |
                   Kirim melalui jaringan internet
                               v
                       [ SERVER BACKEND ]
                               |
         +---------------------+---------------------+
         | (Status 200 OK)                           | (Status 401 Unauthorized)
         v                                           v
+---------------------------------+   +---------------------------------+
|      RESPONSE INTERCEPTOR       |   |      RESPONSE INTERCEPTOR       |
|  Ekstrak payload: res.data      |   |  1. Tahan request dalam antrean |
|  Kembalikan data bersih ke UI   |   |  2. Eksekusi refresh token!     |
|                                 |   |  3. Tembak ulang request awal!  |
+---------------------------------+   +---------------------------------+
         |                                           |
         v                                           v
[ Komponen UI Menerima Data ]       [ Transaksi Berhasil Mulus ]
```

## 5. Cuplikan Logika Inti

### Membangun Klien API Tangguh Menggunakan Native Fetch & AbortController

Modul klien terpusat bebas dependensi pustaka luar:

```typescript
// api/apiClient.ts

interface RequestOptions extends RequestInit {
  timeoutMs?: number;
}

export class ApiError extends Error {
  constructor(
    public status: number,
    public message: string,
    public data?: any
  ) {
    super(message);
    this.name = "ApiError";
  }
}

export async function apiClient<T>(
  endpoint: string,
  options: RequestOptions = {}
): Promise<T> {
  const { timeoutMs = 10000, headers, ...customConfig } = options;

  // 1. Inisialisasi pembatalan batas waktu otomatis
  const controller = new AbortController();
  const timeoutId = setTimeout(() => controller.abort(), timeoutMs);

  const token = localStorage.getItem("access_token");

  const defaultHeaders: HeadersInit = {
    "Content-Type": "application/json",
    ...(token ? { Authorization: `Bearer ${token}` } : {}),
  };

  try {
    const response = await fetch(`/api/v1${endpoint}`, {
      ...customConfig,
      headers: { ...defaultHeaders, ...headers },
      signal: options.signal || controller.signal,
    });

    clearTimeout(timeoutId);

    // 2. Tangani kasus response kosong (seperti status 204 No Content)
    if (response.status === 204) {
      return {} as T;
    }

    // 3. Normalisasi parsing JSON atau fallback teks
    const isJson = response.headers.get("content-type")?.includes("application/json");
    const data = isJson ? await response.json() : await response.text();

    // 4. Jika status bukan 2xx, lempar error terstandarisasi
    if (!response.ok) {
      throw new ApiError(
        response.status,
        (data && data.message) || `HTTP Error ${response.status}`,
        data
      );
    }

    return data as T;
  } catch (error: any) {
    clearTimeout(timeoutId);

    if (error.name === "AbortError") {
      throw new ApiError(408, "Batas waktu request terlampaui (Timeout). Silakan coba lagi.");
    }

    if (error instanceof ApiError) {
      throw error;
    }

    // Tangani error jaringan terputus (Offline)
    throw new ApiError(0, "Koneksi jaringan terputus. Periksa sinyal internet Anda.");
  }
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan
- **Arsitektur DRY (Don't Repeat Yourself)**: Pengaturan header, token autentikasi, dan baseUrl berada di satu titik kontrol.
- **Ketahanan Antarmuka**: Mencegah aplikasi membeku selamanya melalui batas waktu timeout yang ketat.
- **Pesan Kesalahan Ramah Pengguna**: Mengeliminasi pesan error teknis mentah di layar pengguna.

### Kekurangan & Tantangan
- **Tantangan Refresh Token Berbarengan (*Concurrent Token Refresh*)**: Jika ada 4 request API yang gagal dengan status 401 secara bersamaan, klien API harus cerdas menggunakan antrean (*promise queuing*) agar fungsi refresh token hanya ditembakkan **satu kali**, bukan 4 kali berturut-turut.
- **Penanganan Serialization Kompleks**: Pengiriman form dengan lampiran file biner membutuhkan perlakuan khusus (`FormData`) tanpa menyertakan header `Content-Type: application/json`.

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Memakai Pola Klien Terpusat
- Seluruh aplikasi web yang berkomunikasi dengan backend berbasis REST API.
- Aplikasi yang mewajibkan otorisasi token Bearer JWT dan pelacakan audit header (*tracing headers*).

### Kapan Mempertimbangkan Protokol Alternatif
- Jika kebutuhan data Anda membutuhkan pengambilan grafik relasi data yang sangat fleksibel tanpa over-fetching (Gunakan **GraphQL**).
- Jika komunikasi membutuhkan pertukaran data dua arah berlatensi sub-milidetik (Gunakan **WebSocket**).

## 8. Metrik Kunci & Observabilitas

- **HTTP Error Rate (Rasio Error 4xx / 5xx)**: Pantau di monitoring frontend (Sentry/Datadog):
  `Error Rate = (Total Request Gagal / Total Request) * 100%`. Waspadai jika terjadi lonjakan error status 401 atau 500.
- **API Response Latency**: Amati waktu respons P95 endpoint backend di tab Network DevTools. Jika latensi P95 > 2.000 ms, backend memerlukan optimasi query atau caching database.

## 9. Studi Kasus Industri Nyata

### Penanganan Timeout dan Retries pada Airbnb Mobile Web

- **Tantangan Rekayasa Riil**:
  Pengguna Airbnb sering memesan kamar saat sedang bepergian di area wisata dengan sinyal internet seluler tidak stabil. Request pemesanan kamar sering kali gagal di tengah jalan akibat timeout jaringan mikro (*micro-disconnections*), menyebabkan pengguna mengulang proses pemesanan dari awal.
- **Implementasi Solusi Industri**:
  Airbnb membangun klien jaringan khusus:
  1. Menyertakan batas waktu adaptif (*Adaptive Timeout*) berdasarkan kualitas jaringan pengguna saat itu.
  2. Mengimplementasikan **Exponential Backoff with Jitter** otomatis untuk request yang gagal akibat error jaringan sementara (*idempotent GET queries*).
  3. Menyertakan header `X-Idempotency-Key` pada mutasi kritis agar server tidak membuat pemesanan ganda saat terjadi retry otomatis.
- **Dampak & Hasil Sistem**:
  - Tingkat keberhasilan transaksi pada jaringan seluler buruk meningkat sebesar 14%.
  - Kasus reservasi ganda akibat ketidaksabaran pengguna turun ke nol.

## 10. Fakta Menarik & Gotchas Production

1. **`fetch()` Tidak Melempar Error pada Status 404 atau 500**:
   Jebakan klasik nomor satu bagi junior developer: **Fungsi `fetch()` bawaan browser TIDAK AKAN melempar blok `catch` jika server mengembalikan HTTP 404 Not Found atau 500 Internal Server Error!** `fetch()` hanya akan melempar blok `catch` jika koneksi jaringan putus total atau domain tidak ditemukan. Anda WAJIB memeriksa properti `if (!response.ok)` secara manual!
2. **Bahaya Mengabaikan `AbortController` pada Komponen yang Unmount**:
   Jika pengguna membuka halaman profil dan langsung mengklik tombol kembali dalam 200ms sebelum request selesai, respons yang tiba di komponen yang sudah mati (*unmounted component*) dapat memicu memory leak jika tidak dibatalkan via `signal: controller.signal`.

## 11. Rangkuman Singkat

1. **Pusatkan Klien Jaringan**: Jangan pernah menulis `fetch()` mentah yang tercecer di dalam komponen antarmuka.
2. **Wajib Batas Waktu**: Selalu gunakan `AbortController` dengan batas waktu timeout maksimal untuk mencegah antarmuka membeku.
3. **Ingat response.ok**: Validasi selalu flag `response.ok` karena `fetch()` tidak melempar error pada kode status HTTP 4xx dan 5xx.
4. **Normalisasi Error**: Bungkus respon kesalahan ke dalam satu format objek seragam yang mudah ditangani oleh UI.
