# GraphQL

## 1. Skenario Sistem Nyata

Pada aplikasi web marketplace e-commerce multi-kategori, tim frontend sedang membangun antarmuka kartu produk mini (*Product Card*) untuk bilah rekomendasi di sidebar. Kartu mini tersebut hanya membutuhkan 3 data sederhana: **ID Produk, Judul Produk, dan Gambar Thumbnail**.

Ketika menggunakan arsitektur REST API konvensional:
- Endpoint yang tersedia di backend adalah `GET /api/v1/products/101`.
- Backend mengembalikan objek JSON raksasa berisi lebih dari 80 properti (deskripsi lengkap HTML 5.000 kata, riwayat stok gudang, dimensi berat kemasan, daftar 50 ulasan pembeli, informasi pajak manufaktur, hingga profil lengkap penjual).
- Halaman sidebar memuat 20 kartu produk mini, menghasilkan transfer data sebesar **3,8 Megabyte JSON mentah** yang harus diunduh dan diparsing oleh browser, padahal 95% dari data tersebut tidak pernah ditampilkan di layar (*Over-Fetching* parah).
- Sebaliknya, ketika pengguna mengklik kartu tersebut dan masuk ke halaman detail, frontend harus menembak 4 endpoint terpisah (`GET /products/101`, `GET /users/50`, `GET /reviews?productId=101`, `GET /shipping-rates`) hanya untuk merender satu halaman (*Under-Fetching / N+1 Problem*).

Solusi: Mengadopsi **GraphQL**. Frontend memiliki kendali penuh untuk mendeklarasikan secara presisi field apa saja yang dibutuhkannya dalam satu dokumen query deklaratif tunggal, memangkas payload jaringan hingga 90% dan menggabungkan banyak endpoint ke dalam satu request jaringan efisien.

## 2. Apa Itu dan Buat Apa?

GraphQL adalah bahasa kueri (*query language*) untuk API dan lingkungan eksekusi runtime di sisi server yang dikembangkan oleh Facebook (Meta) pada tahun 2012 dan dirilis sebagai standar terbuka pada tahun 2015.

Berbeda dengan REST API di mana bentuk data balasan ditentukan secara kaku oleh server backend (*server-defined responses*), pada GraphQL **klien frontend yang menentukan secara persis struktur dan properti data yang ingin diterimanya (*client-driven queries*)**.

Tiga operasi utama dalam GraphQL:
1. **Query**: Operasi pembacaan data (setara dengan `GET` di REST).
2. **Mutation**: Operasi penulisan, pembaruan, atau penghapusan data (setara dengan `POST`, `PUT`, `DELETE`).
3. **Subscription**: Aliran data real-time berbasis WebSocket di mana server secara otomatis mengirimkan pembaruan data saat terjadi mutasi tertentu di database.

Fungsi utama GraphQL:
- **Mengeliminasi Over-Fetching**: Klien tidak pernah menerima data yang tidak dimintanya.
- **Mengeliminasi Under-Fetching**: Mengambil relasi data bertingkat (*nested graphs*, misal: produk beserta ulasan dan nama penulis ulasan) dalam satu kali panggilan jaringan.
- **Type-Safety Kuat**: Didukung oleh skema tipe data ketat (*Schema Definition Language / SDL*) yang dapat diintegrasikan dengan generator TypeScript otomatis.

## 3. Kenapa Harus Pakai Ini?

Untuk aplikasi frontend kompleks yang memiliki banyak tampilan antarmuka berbeda (aplikasi web desktop, web seluler, jam pintar, smart TV):
1. **Satu Endpoint untuk Semua Kebutuhan**: Seluruh request dikirimkan ke satu endpoint tunggal (biasanya `POST /graphql`).
2. **Kemandirian Tim Frontend**: Frontend developer dapat menambah atau mengurangi field di UI tanpa perlu meminta tim backend membuatkan endpoint baru setiap minggu.
3. **Normalized Cache Canggih**: Pustaka klien GraphQL (seperti Apollo Client atau Urql) secara otomatis menormalisasi entitas data di memori berdasarkan `__typename` dan `id`.

## 4. Cara Kerjanya (Diagram Alur Over-Fetching REST vs Presisi GraphQL)

```
PENDEKATAN REST API (Server Mengatur Segalanya):
[ Browser ] ------- GET /api/products/42 -------> [ Backend ]
[ Browser ] <--- 80 Kolom JSON (Ukuran: 180 KB) - [ Backend ]
*95% data terbuang sia-sia karena UI hanya butuh nama dan harga!*

----------------------------------------------------------------------

PENDEKATAN GRAPHQL (Klien Meminta Sesuai Kebutuhan):
[ Browser ] ------- POST /graphql ----------------> [ Backend ]
Payload Query:
query {
  product(id: "42") {
    title
    price
  }
}
                                                          |
                                           Server mengeksekusi resolver
                                                          |
[ Browser ] <--- Response: -------------------------------+
{
  "data": {
    "product": {
      "title": "Sepatu Lari Pro",
      "price": 750000
    }
  }
}
*UKURAN HANYA 120 BYTE! (NOL DATA SAMPAH)*
```

## 5. Cuplikan Logika Inti

### Mengirim Query GraphQL Menggunakan Fetch API Murni Tanpa Library

GraphQL pada dasarnya hanyalah request HTTP POST standar dengan payload JSON:

```typescript
// services/graphqlClient.ts

interface GraphQLResponse<T> {
  data?: T;
  errors?: Array<{ message: string; locations?: any; path?: any }>;
}

export async function fetchGraphQL<T>(
  query: string,
  variables: Record<string, any> = {}
): Promise<T> {
  const response = await fetch("/graphql", {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      Authorization: `Bearer ${localStorage.getItem("token") || ""}`,
    },
    body: JSON.stringify({
      query,
      variables,
    }),
  });

  const result: GraphQLResponse<T> = await response.json();

  // GraphQL mengembalikan error di dalam properti 'errors' meskipun HTTP status 200 OK!
  if (result.errors && result.errors.length > 0) {
    throw new Error(result.errors.map((e) => e.message).join(", "));
  }

  if (!result.data) {
    throw new Error("Tidak ada data yang dikembalikan oleh server GraphQL.");
  }

  return result.data;
}

// Contoh Penggunaan Query:
const GET_PRODUCT_CARD = `
  query GetProductMini($id: ID!) {
    product(id: $id) {
      id
      title
      price
      thumbnailUrl
    }
  }
`;

// Panggil di komponen:
// const data = await fetchGraphQL(GET_PRODUCT_CARD, { id: "101" });
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan
- **Efisiensi Jaringan Maksimal**: Menghemat konsumsi bandwidth secara drastis pada perangkat seluler.
- **Dokumentasi Skema Otomatis**: Klien dapat melakukan introspeksi skema (*Schema Introspection*) untuk melihat seluruh dokumentasi API di alat seperti GraphQL Playground atau Apollo Studio.
- **Ekosistem Code Generation**: Pustaka seperti *GraphQL Code Generator* dapat membaca berkas `.graphql` Anda dan secara otomatis menghasilkan tipe data TypeScript murni 100% akurat.

### Kekurangan & Tantangan Nyata
- **HTTP Caching Standar Rusak**: Karena hampir seluruh request GraphQL dikirim menggunakan metode **HTTP POST ke satu endpoint yang sama**, mekanisme caching bawaan peramban dan CDN (seperti `Cache-Control` pada GET request) **tidak berfungsi secara otomatis**! Anda memerlukan pustaka khusus dengan *Normalized In-Memory Cache* (Apollo Client) atau CDN cerdas yang mendukung GraphQL caching (Stellate).
- **Ukuran Bundle Klien Lebih Besar**: Pustaka klien GraphQL lengkap (seperti Apollo Client) memiliki ukuran bundle yang cukup besar (~30 KB gzipped).
- **Beban Kompleksitas di Sisi Backend**: Query yang terlalu dalam bersarang (*Deeply Nested Queries*) dapat melumpuhkan database backend jika tidak dilindungi dengan batasan kedalaman kueri (*Query Depth Limiting*).

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Memilih GraphQL
- Aplikasi web berskala menengah-besar dengan struktur data grafik relasional yang kompleks (seperti jejaring sosial, platform kolaboratif, dashboard analitik multientitas).
- Proyek yang melayani banyak antarmuka klien yang berbeda (Web Desktop, Mobile App iOS/Android, Smartwatch) yang masing-masing membutuhkan bentuk data yang berbeda dari sumber yang sama.

### Kapan Sebaiknya Tetap Menggunakan REST API
- Operasi pengunggahan berkas biner berukuran besar (*File Uploads / Media Streaming*) — REST jauh lebih alami menangani multipart form data.
- Aplikasi sederhana (CRUD standar) dengan tim developer kecil.
- Layanan publik pihak ketiga (*Public Developer APIs*) di mana kemudahan integrasi dengan `curl` dan HTTP GET caching menjadi prioritas utama.

## 8. Metrik Kunci & Observabilitas

- **GraphQL Execution Errors Rate**: Pantau kemunculan properti `errors` pada respon GraphQL di alat pelacak Sentry.
- **Payload Size Savings**: Bandingkan ukuran muatan data JSON yang diterima antara query GraphQL teroptimasi vs endpoint REST tradisional.
- **Cache Hit Ratio di Apollo Client / Urql**: Memastikan query identik dijawab langsung dari normalized cache memori lokal klien.

## 9. Studi Kasus Industri Nyata

### Transformasi Arsitektur Web & Mobile di GitHub

- **Tantangan Rekayasa Riil**:
  Sebelumnya, GitHub menyediakan REST API v3 yang sangat lengkap. Namun, untuk merender halaman Issue yang memuat status tiket, komentar, label, timeline commit, dan nama reviewer, aplikasi web dan seluler GitHub terpaksa mengirimkan belasan request REST paralel. Hal ini membuat latensi pemuatan halaman lambat dan menghabiskan kuota *rate limit* pengguna.
- **Implementasi Solusi Industri**:
  GitHub merilis **GitHub GraphQL API v4**:
  1. Menggabungkan seluruh data issue, komentar, dan author ke dalam satu kueri GraphQL tunggal.
  2. Memanfaatkan GraphQL Subscription untuk memperbarui komentar baru secara real-time tanpa polling.
- **Dampak & Hasil Sistem**:
  - Jumlah request HTTP yang dibutuhkan untuk merender satu halaman Issue berkurang dari 12 request menjadi **tepat 1 request tunggal**.
  - Waktu render halaman di aplikasi seluler meningkat lebih dari 2x lebih cepat.

## 10. Fakta Menarik & Gotchas Production

1. **Jebakan Status HTTP 200 dengan Error**:
   Pada GraphQL, jika terjadi kesalahan logika di backend (misal data produk tidak ditemukan atau database timeout), server GraphQL secara default **TETAP MENGEMBALIKAN STATUS HTTP 200 OK**, dan menyematkan pesan kesalahan di dalam array `errors: [{ message: "..." }]`. Jika kode frontend Anda hanya memeriksa `if (res.ok)`, Anda akan mengira request berhasil padahal datanya gagal dimuat!
2. **Normalized Cache Merekam Berdasarkan `__typename` + `id`**:
   Pustaka seperti Apollo Client menormalisasi memori cache dengan menggabungkan nama tipe dan ID (misal: `Product:42`). Jika Anda sengaja tidak meminta kolom `id` di dalam kueri Anda, Apollo tidak dapat menormalisasi data tersebut, menyebabkan komponen lain tidak ikut ter-update saat mutasi terjadi.

## 11. Rangkuman Singkat

1. **Klien yang Menentukan**: GraphQL memungkinkan klien frontend meminta data dengan presisi tinggi sesuai kebutuhan tampilan UI.
2. **Solusi Over & Under-Fetching**: Mengeliminasi data berlebih dan memangkas banyak request jaringan menjadi satu panggilan tunggal.
3. **Waspadai HTTP Caching**: Karena menggunakan HTTP POST ke satu endpoint, caching HTTP standar browser tidak berfungsi; gunakan normalized client cache.
4. **Periksa Properti Errors**: Jangan tertipu oleh status HTTP 200 OK; selalu periksa apakah ada objek `errors` di dalam payload balasan GraphQL.
