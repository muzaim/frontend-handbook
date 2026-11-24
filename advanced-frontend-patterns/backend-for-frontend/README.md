# Backend-for-Frontend (BFF)

## 1. Skenario Sistem Nyata

Bayangkan Anda memimpin tim frontend untuk aplikasi dashboard e-commerce berskala enterprise. Halaman detail pesanan (*Order Details*) membutuhkan data dari 5 microservice backend internal yang berbeda:
1. `Order Service`: Rincian ID pesanan, total harga, dan tanggal transaksi.
2. `User Service`: Nama pembeli, nomor telepon, dan preferensi akun.
3. `Shipping Service`: Status kurir, nomor resi pengiriman, dan estimasi tiba.
4. `Payment Service`: Status transfer bank, kartu kredit, dan mutasi saldo.
5. `Product Service`: Nama produk, gambar thumbnail, dan spesifikasi barang.

Tanpa layer Backend-for-Frontend (BFF), browser pengguna di jaringan seluler harus menembakkan 5 HTTP request terpisah (*request waterfalls*). Setiap service backend mengembalikan JSON raksasa berisi ratusan field database internal, padahal komponen UI hanya membutuhkan 10-15 field saja. Lebih buruk lagi, jika service kurir mengalami timeout atau down, browser crash dan menampilkan error layar putih karena tidak ada agregasi yang menangani *partial failure*.

Dengan pola **Backend-for-Frontend (BFF)**, tim frontend membuat dan mengelola server perantara tipis (misalnya menggunakan Fastify atau Node.js). Browser hanya menembakkan **1 request tunggal** ke BFF. BFF di jaringan internal cloud yang sangat cepat (sub-millisecond latency) memanggil kelima microservice secara paralel, memangkas 85% data yang tidak dibutuhkan UI, menormalkan format error, dan mengembalikan satu payload rapi yang langsung cocok dengan struktur props komponen React/Vue.

## 2. Apa Itu dan Buat Apa?

Backend-for-Frontend (BFF) adalah pola arsitektur perangkat lunak di mana sebuah server layer perantara dibuat khusus untuk melayani kebutuhan antarmuka pengguna (*client UI*) tertentu. Dalam ekosistem rekayasa modern, layer BFF **dimiliki, dirawat, dan di-deploy oleh tim frontend**, bukan tim backend inti.

Tujuan utama BFF dari perspektif frontend:
- **API Aggregation**: Menggabungkan respons dari banyak microservices internal menjadi satu payload terpadu untuk menghemat round-trip jaringan seluler.
- **Data Shaping & Payload Pruning**: Membuang field-field berlebih dari backend dan mentransformasikan struktur data agar langsung siap dikonsumsi komponen antarmuka (*view-centric*).
- **Security Shield & Token Translation**: Mengonversi session cookie `HttpOnly` dari browser menjadi token otentikasi internal (seperti JWT Bearer atau mTLS) sehingga token rahasia tidak pernah terekspos ke JavaScript client.
- **Error Normalization**: Menyembunyikan kompleksitas error code upstream (500, 502, timeout gRPC) dan mengubahnya menjadi status kesalahan yang terstandar dan ramah pengguna.

## 3. Kenapa Harus Pakai Ini? (Sudut Pandang Tim Frontend)

1. **Otonomi Penuh Tim Frontend**: Tim frontend tidak perlu mengajukan tiket perubahan skema API ke tim backend setiap kali ada komponen UI baru atau revisi layout. Tim frontend bebas membentuk endpoint dan respons di layer BFF sendiri.
2. **Eliminasi Network Waterfalls**: Mengubah 5-10 request serial/paralel dengan latensi jaringan seluler tinggi (misal 150ms per panggilan) menjadi 1 panggilan tunggal ke BFF. Komunikasi antar-service di dalam data center cloud hanya membutuhkan latensi 1-5ms.
3. **Pemberantasan Bahaya XSS pada Token Rahasia**: Browser tidak lagi menyimpan Access Token atau API Key pihak ketiga di `localStorage` peramban. Token rahasia disimpan di memori/environment BFF.
4. **Resiliensi UI Terhadap Partial Failures**: Jika 1 dari 4 service downstream mengalami kendala, BFF dapat mengembalikan data parsial dengan status degradasi anggun (*graceful degradation*) sehingga antarmuka utama tetap berfungsi.

## 4. Cara Kerjanya

BFF bertindak sebagai fasilitator antara dunia browser yang berorientasi pada komponen tampilan (*View-Centric*) dan dunia backend yang berorientasi pada integritas domain database (*Data-Centric*).

```
+-------------------------------------------------------------------------+
|                  ARSITEKTUR LAYER BFF UNTUK FRONTEND                    |
+-------------------------------------------------------------------------+
|                                                                         |
|  [ BROWSER / CLIENT UI ]                                                |
|         |                                                               |
|         | (1) GET /api/bff/order-details/123                            |
|         |     Cookie: session_id=abc (HttpOnly, Secure, SameSite)       |
|         v                                                               |
|  +-------------------------------------------------------------------+  |
|  | BACKEND-FOR-FRONTEND (BFF) LAYER (Node.js / Nitro / Fastify)      |  |
|  | Dikelola & Dimiliki oleh Tim Frontend                             |  |
|  |                                                                   |  |
|  |  +---------------------+  +-----------------+  +----------------+ |  |
|  |  | Auth Bridge:        |  | Orchestration:  |  | Data Trimming: | |  |
|  |  | Validasi Cookie ->  |  | Promise.all()   |  | Filter 80%     | |  |
|  |  | Injeksi Bearer JWT  |  | Paralel Fetch   |  | Field Mubazir  | |  |
|  |  +---------------------+  +-----------------+  +----------------+ |  |
|  +-------------------------------------------------------------------+  |
|         |                   |                    |                      |
|         | Internal RPC/HTTP | Internal RPC/HTTP  | Internal RPC/HTTP    |
|         v (Sub-5ms Network) v (Sub-5ms Network)  v (Sub-5ms Network)    |
|   +---------------+   +------------------+   +--------------------+     |
|   | Order Service |   | Shipping Service |   | Payment Service    |     |
|   | (Data-Centric)|   | (Data-Centric)   |   | (Data-Centric)     |     |
|   +---------------+   +------------------+   +--------------------+     |
|                                                                         |
+-------------------------------------------------------------------------+
```

### 1. Perbedaan Data-Centric API vs View-Centric API
- **Data-Centric API (Milik Backend Inti)**: Mengikuti model entitas database (misal table `orders`, table `shipments`, table `invoices`). Dirancang untuk netral dan melayani semua konsumen (web, mobile, partner pihak ketiga).
- **View-Centric API (Milik BFF)**: Mengikuti hierarki visual komponen antarmuka (misal `OrderCardProps`, `ShippingTrackerWidgetProps`). Hanya berisi field yang benar-benar akan dirender di layar.

### 2. Pola Penanganan Partial Failure
Ketika BFF memanggil beberapa service, jangan gunakan `Promise.all()` naif yang langsung melempar error jika ada satu service gagal (*fail-fast*). Gunakan `Promise.allSettled()` sehingga jika service non-kritis (misal widget rekomendasi barang atau review) down, BFF tetap mengembalikan data inti transaksi disertai flag status.

## 5. Cuplikan Logika Inti

### 1. Implementasi Server BFF: Agregasi, Pruning, dan Error Normalization

```javascript
// bff-server.js (Node.js / Fastify / Express)
import Fastify from 'fastify';

const fastify = Fastify({ logger: true });

// Endpoint khusus yang melayani halaman Order Details UI
fastify.get('/api/bff/order-details/:orderId', async (request, reply) => {
  const { orderId } = request.params;
  const sessionId = request.cookies.session_id;

  // 1. Validasi Auth: Tukar session cookie dengan internal service token
  if (!sessionId) {
    return reply.status(401).send({
      error: { code: 'UNAUTHORIZED', message: 'Sesi login telah berakhir.' }
    });
  }

  const internalHeaders = {
    'Authorization': `Bearer ${process.env.INTERNAL_SERVICE_TOKEN}`,
    'X-Correlation-ID': request.headers['x-correlation-id'] || crypto.randomUUID()
  };

  try {
    // 2. Parallel Orchestration: Panggil microservice secara non-blocking
    const [orderResult, shippingResult, promoResult] = await Promise.allSettled([
      fetch(`http://order-svc/orders/${orderId}`, { headers: internalHeaders }).then(r => r.json()),
      fetch(`http://shipping-svc/tracking/${orderId}`, { headers: internalHeaders }).then(r => r.json()),
      fetch(`http://promo-svc/vouchers/${orderId}`, { headers: internalHeaders }).then(r => r.json())
    ]);

    // Jika service inti (Order Service) gagal, kita tidak bisa melanjutkan
    if (orderResult.status === 'rejected') {
      fastify.log.error(orderResult.reason);
      return reply.status(502).send({
        error: {
          code: 'ORDER_SERVICE_UNAVAILABLE',
          message: 'Gagal memuat rincian pesanan. Silakan coba lagi.'
        }
      });
    }

    const orderData = orderResult.value;

    // 3. Graceful Fallback: Jika service sekunder (Shipping / Promo) gagal
    const shippingData = shippingResult.status === 'fulfilled' ? shippingResult.value : null;
    const promoData = promoResult.status === 'fulfilled' ? promoResult.value : null;

    // 4. Data Shaping & Payload Pruning: Bentuk payload persis seperti kebutuhan UI
    // Memangkas 70% field audit database backend yang tidak diperlukan frontend
    const viewProps = {
      order: {
        id: orderData.id,
        invoiceNumber: orderData.invoice_no,
        dateFormatted: new Intl.DateTimeFormat('id-ID').format(new Date(orderData.created_at)),
        totalPrice: orderData.grand_total,
        items: orderData.line_items.map((item) => ({
          name: item.title,
          quantity: item.qty,
          price: item.unit_price,
          thumbnailUrl: item.media_url
        }))
      },
      shipping: shippingData ? {
        status: shippingData.current_status,
        courierName: shippingData.vendor_name,
        trackingCode: shippingData.airway_bill
      } : {
        status: 'UNAVAILABLE',
        message: 'Informasi pelacakan pengiriman sedang dalam pemeliharaan.'
      },
      hasActiveDiscount: Boolean(promoData && promoData.discount_applied)
    };

    return reply.status(200).send(viewProps);
  } catch (err) {
    fastify.log.error(err);
    return reply.status(500).send({
      error: { code: 'INTERNAL_BFF_ERROR', message: 'Terjadi gangguan internal pada server antarmuka.' }
    });
  }
});
```

### 2. Konsumsi Bersih di Komponen Frontend Client

```javascript
// OrderDetailsView.jsx (React Client)
import React, { useEffect, useState } from 'react';

export function OrderDetailsView({ orderId }) {
  const [data, setData] = useState(null);
  const [error, setError] = useState(null);
  const [isLoading, setIsLoading] = useState(true);

  useEffect(() => {
    // Frontend hanya memanggil 1 endpoint bersih tanpa tahu ada 3 microservice di belakangnya
    fetch(`/api/bff/order-details/${orderId}`, {
      credentials: 'include' // Otomatis mengirim HttpOnly cookie
    })
      .then((res) => {
        if (!res.ok) throw res;
        return res.json();
      })
      .then((payload) => {
        setData(payload);
        setIsLoading(false);
      })
      .catch(async (errRes) => {
        const errJson = await errRes.json().catch(() => ({}));
        setError(errJson.error?.message || 'Gagal terhubung ke server.');
        setIsLoading(false);
      });
  }, [orderId]);

  if (isLoading) return <div>Memuat ringkasan pesanan...</div>;
  if (error) return <div className="alert-error">{error}</div>;

  return (
    <div className="order-container">
      <h2>Pesanan: {data.order.invoiceNumber}</h2>
      <p>Tanggal: {data.order.dateFormatted}</p>

      {/* Render komponen shipping dengan graceful handling */}
      <div className="shipping-box">
        {data.shipping.status === 'UNAVAILABLE' ? (
          <p className="notice">{data.shipping.message}</p>
        ) : (
          <p>Kurir: {data.shipping.courierName} ({data.shipping.trackingCode})</p>
        )}
      </div>
    </div>
  );
}
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

| Kelebihan | Kekurangan / Batasan |
| :--- | :--- |
| **Kecepatan Jaringan Maksimal**: Menggantikan puluhan HTTP request terpisah dengan satu roundtrip yang teroptimasi | **Tambahan Node Infrastruktur**: Memerlukan runtime server (container Node.js) yang harus di-maintain, di-monitor, dan di-scale |
| **Keamanan Terjamin**: Kunci API rahasia dan token downstream terlindungi di server-side, tidak pernah sampai ke browser | **Bahaya Code Duplication**: Jika ada banyak aplikasi client (Web Desktop, Mobile App, Tablet), logika agregasi berpotensi ditulis berulang |
| **Penyederhanaan State UI**: Komponen frontend tidak perlu lagi mengelola koordinasi 5 loading spinner terpisah | **Risiko Anti-Pola "Fat BFF"**: Bahaya engineer mulai menaruh logika bisnis berat (seperti kalkulasi pajak atau database direct writes) di BFF |
| **Ketahanan Parsial**: Komponen UI tidak langsung meledak jika salah satu microservice pelengkap mengalami gangguan | **Titik Kegagalan Tunggal (SPOF)**: Jika server instance BFF crash, seluruh tampilan halaman frontend yang dilayani akan mati |

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Harus Pakai
- **Arsitektur Microservices Rumit**: Aplikasi frontend mengonsumsi puluhan microservices independen yang dikelola oleh tim backend berbeda.
- **Klien Web Seluler (Low Bandwidth)**: Pengguna dominan mengakses web melalui smartphone dengan jaringan 4G/3G yang lambat dan kuota terbatas.
- **Sistem dengan Tingkat Keamanan Tinggi (Fintech / Healthcare)**: Aplikasi dilarang keras menyimpan access token atau data sensitif mentah di client-side storage karena regulasi audit keamanan.

### Kapan Jangan Pakai
- **Backend Monolitik (Monolith REST / GraphQL)**: Jika backend aplikasi Anda sudah berbentuk monolitik atau sudah memiliki GraphQL gateway terpusat yang bisa melakukan data picking, membuat BFF hanya akan menambah lapisan latensi (*hop*) yang sia-sia.
- **Aplikasi CRUD Sederhana**: Aplikasi internal admin tool yang penggunaannya rendah dan struktur datanya 1:1 persis dengan tabel database.

## 8. Metrik Kunci & Observabilitas

1. **Downstream Call Latency vs Total BFF Latency**: Memonitor selisih antara waktu respon BFF ke browser dengan total waktu pemanggilan service downstream. Membantu mendeteksi apakah bottleneck terjadi di internal BFF atau di microservice backend.
2. **Partial Failure Rate**: Rasio persentase request di mana BFF mengembalikan payload dengan salah satu service berstatus fallback/null. Menjadi indikator kesehatan ekosistem microservices downstream.
3. **Bandwidth Savings Ratio**: Perbandingan ukuran total payload mentah dari semua microservices vs ukuran payload yang dikirimkan BFF ke browser. Target ideal: pemangkasan ukuran data mencapai 50% hingga 80%.
4. **Node.js Event Loop Lag**: Karena BFF adalah I/O-heavy gateway, pemantauan Event Loop Lag di runtime Node.js sangat krusial untuk mencegah antrean request macet saat traffic tinggi.

## 9. Studi Kasus Industri Nyata

- **SoundCloud**: SoundCloud adalah salah satu pelopor pertama arsitektur BFF. Awalnya mereka menghadapi kendala di mana mobile web dan desktop app mereka harus memanggil puluhan microservices internal yang membuat mobile browser kehabisan koneksi soket. Mereka membagi layer API menjadi tim-spesifik BFF: satu BFF khusus Web, dan satu BFF khusus Mobile. Hasilnya, produktivitas deploy tim frontend meningkat drastis tanpa perlu berkoordinasi dengan tim backend inti.
- **Netflix**: Netflix mengadopsi pola BFF untuk ratusan jenis perangkat pemutar film (Smart TV, Apple TV, Browser Web, Android TV). Setiap perangkat memiliki kapabilitas rendering dan ukuran memori yang sangat berbeda. BFF khusus Web dirancang mengirimkan payload ringkas yang sesuai dengan DOM browser, sementara BFF untuk Smart TV mengirimkan format data yang dioptimalkan untuk memori decoding grafis TV yang terbatas.

## 10. Fakta Menarik & Gotchas Production

1. **Anti-Pola "Fat BFF" (Logika Bisnis yang Salah Kamar)**: Kesalahan paling umum bagi tim frontend yang memiliki BFF adalah mulai menulis aturan bisnis domain di BFF (misalnya validasi diskon promo atau penulisan query langsung ke database). BFF **hanya boleh** bertindak sebagai transformator dan orkestrator data tampilan (*view-orchestration*). Aturan bisnis inti tetap harus hidup di backend domain service.
2. **Kaskade Timeout Jaringan**: Jika Service A memakan waktu 3 detik, Service B memakan waktu 4 detik, dan timeout default BFF diset 5 detik, BFF akan sering mengalami timeout misterius. Selalu terapkan timeout agresif (misal 800ms) untuk setiap panggilan downstream independen dengan bantuan `AbortController`.
3. **Tracing Distributed Wajib Dipasang**: Saat terjadi error di frontend, sulit mengetahui service downstream mana yang menjadi biang kerok jika tidak ada correlation ID. Selalu teruskan header `X-Request-ID` atau `traceparent` (OpenTelemetry) dari browser -> BFF -> microservice downstream.

## 11. Rangkuman Singkat

Backend-for-Frontend (BFF) adalah jembatan arsitektur penting yang membebaskan tim frontend dari jeratan kompleksitas microservices backend. Dengan menempatkan server gateway tipis yang dimiliki dan dikontrol oleh tim frontend, aplikasi web dapat mengubah puluhan panggilan jaringan yang lambat menjadi satu request yang teroptimasi, memangkas ukuran payload JSON secara signifikan, melindungi rahasia otentikasi di level server, dan menghadirkan pengalaman antarmuka pengguna yang tangguh terhadap kegagalan parsial.
