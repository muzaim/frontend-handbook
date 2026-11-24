# State Management Patterns

## 1. Skenario Sistem Nyata

Pada aplikasi kanvas diagram alur interaktif (seperti Miro atau Draw.io berbasis web), pengguna dapat menambahkan ratusan kotak, menarik garis panah penghubung (*connectors*), mengubah warna, dan melakukan aksi pembatalan (*Undo / Redo / History Time-Travel*).

Developer junior mengelola status data secara ad-hoc dengan memutasi objek secara langsung (*Direct Object Mutation*):

```javascript
// Kode bermasalah: Mutasi langsung objek state
function moveBox(boxId, newX, newY) {
  const box = appState.boxes.find((b) => b.id === boxId);
  box.x = newX; // Mutasi langsung properti objek!
  box.y = newY;
  renderCanvas();
}
```

Dampak pada sistem:
- Ketika tim ingin menambahkan fitur **Undo / Redo**, mereka tidak bisa melakukannya karena state lama sudah ditimpa secara permanen di memori tanpa jejak riwayat mutasi.
- Terjadi *Race Condition* dan bug misterius: beberapa komponen tidak mengetahui bahwa nilai koordinat kotak sudah berubah karena referensi memori objek `box` tetap sama persis (*reference equality check* menghasilkan `true`).
- Kode aplikasi menjadi spaghetti, di mana komponen mana saja bisa mengubah data apa saja secara liar dari berbagai sudut kode tanpa ada audit jejak (*audit trail*).

Solusi: Memahami dan menerapkan **State Management Patterns (Pola Manajemen State)**, seperti **Immutability**, **Flux Architecture (Unidirectional Data Flow)**, **Observer / Reactive Pattern**, atau **State Machine**.

## 2. Apa Itu dan Buat Apa?

State Management Patterns adalah kumpulan prinsip arsitektur dan pola desain perangkat lunak yang mengatur **bagaimana data aplikasi disimpan, bagaimana data dialirkan, dan bagaimana aturan perubahan data diberlakukan** secara terprediksi di lingkungan frontend.

Alih-alih membiarkan kode mengubah variabel global secara acak, pola ini membatasi mutasi melalui aturan yang terstruktur.

Pola-pola arsitektur state paling berpengaruh di ekosistem frontend:
1. **Unidirectional Data Flow (Aliran Data Satu Arah / Flux Pattern)**: Data hanya mengalir dalam satu arah melingkar: `Action -> Dispatcher -> Store -> View`. Komponen View tidak boleh memutasi state secara langsung.
2. **Immutability Pattern**: Data tidak pernah diubah di tempat (*mutated in-place*); setiap pembaruan menghasilkan salinan objek baru dengan perubahan yang diterapkan.
3. **Reactive / Proxy Pattern**: Membungkus objek data dengan `Proxy` (seperti di Vue 3 atau MobX) untuk mendeteksi pembacaan dan penulisan secara otomatis.
4. **Finite State Machines (FSM)**: Membatasi state aplikasi hanya pada daftar status yang terdefinisi secara eksplisit (misal: `idle -> loading -> success | error`), mencegah status yang mustahil terjadi (*impossible states*).

## 3. Kenapa Harus Pakai Ini?

Seiring bertambahnya kompleksitas logika bisnis di frontend, sumber bug terbesar hampir selalu berakar pada satu hal: **Ketidakpastian mutasi data (*Unpredictable State Mutations*)**.

Menerapkan pola manajemen state yang disiplin memberikan:
1. **Prediktabilitas Mutasi**: Anda selalu tahu fungsi mana, aksi apa, dan kapan sebuah data berubah.
2. **Kemudahan Debugging**: Memungkinkan pencatatan log histori mutasi (*Action Logging*) dan fitur Time-Travel Debugging.
3. **Eliminasi Status Mustahil**: Mencegah kondisi UI aneh (seperti menampilkan status `isLoading: true` DAN `isSuccess: true` secara bersamaan).

## 4. Cara Kerjanya (Diagram Alur Flux Architecture)

Pola Flux yang dipopulerkan oleh Facebook memastikan data selalu mengalir dalam satu arah tanpa loop silang:

```
+---------------------------------------------------------------+
|                       VIEW (Komponen UI)                      |
|            Pengguna menekan tombol "Hapus Akun"               |
+---------------------------------------------------------------+
                               |
                   1. Memicu Aksi (Dispatch)
                               v
+---------------------------------------------------------------+
|                            ACTION                             |
|          { type: 'DELETE_ACCOUNT', payload: { id: 42 } }       |
+---------------------------------------------------------------+
                               |
                   2. Diteruskan ke Reducer / Store
                               v
+---------------------------------------------------------------+
|                      STORE / REDUCER LOGIC                    |
|       (Fungsi Murni: Menerima State Lama + Action,           |
|        Menghasilkan Salinan State Baru yang Bersih)          |
+---------------------------------------------------------------+
                               |
                   3. Notifikasi Perubahan ke Subscriber
                               v
+---------------------------------------------------------------+
|                       VIEW (Komponen UI)                      |
|            Komponen membaca state baru dan re-render!         |
+---------------------------------------------------------------+
```

## 5. Cuplikan Logika Inti

### Perbandingan: Pola Flux (Immutability) vs Pola Reactive Proxy

**1. Pola Flux / Reducer Murni (Immutable Data):**
```javascript
// State lama tidak disentuh; selalu kembalikan objek baru!
function todoReducer(state, action) {
  switch (action.type) {
    case "ADD_TODO":
      return {
        ...state,
        todos: [...state.todos, action.payload], // Salinan array baru
      };
    case "TOGGLE_TODO":
      return {
        ...state,
        todos: state.todos.map((todo) =>
          todo.id === action.payload.id ? { ...todo, completed: !todo.completed } : todo
        ),
      };
    default:
      return state;
  }
}
```

**2. Pola Reactive Proxy (Vue 3 / MobX Style):**
```javascript
// Menggunakan JavaScript Proxy bawaan untuk mendeteksi mutasi secara otomatis
function reactive(target, onMutation) {
  return new Proxy(target, {
    set(obj, prop, value) {
      obj[prop] = value;
      onMutation(prop, value); // Otomatis picu re-render!
      return true;
    },
  });
}

const state = reactive({ count: 0 }, (prop, val) => {
  console.log(`Properti ${prop} berubah menjadi: ${val}`);
  document.getElementById("counter-display").textContent = val;
});

// Penulisan terasa seperti mutasi biasa, tetapi di bawah kap ada pengamat reaktif
state.count += 1;
```

## 6. Kelebihan dan Kekurangan (Trade-offs)

### Kelebihan
- **Pelacakan Bug Deterministik**: Setiap perubahan memiliki nama aksi (*action type*) yang tercatat di DevTools.
- **Kemudahan Pembuatan Fitur Kompleks**: Fitur seperti Undo, Redo, Snapshot Session, dan Sinkronisasi Multi-Tab menjadi sangat mudah dibangun di atas data immutable.
- **Kerapian Kolaborasi Tim**: Standar alur data yang seragam mencegah developer menulis logika mutasi data yang berantakan.

### Kekurangan
- **Overhead Boilerplate pada Pola Flux**: Menulis action, reducer, dan type definition untuk perubahan data kecil terasa bertele-tele.
- **Biaya Memori Kloning Objek**: Membuat salinan objek baru secara terus-menerus membebani Garbage Collector jika struktur data sangat dalam (*deeply nested*). (Solusi: Gunakan library *Structural Sharing* seperti **Immer**).

## 7. Kapan Harus Pakai vs Kapan Jangan Pakai

### Kapan Menggunakan Pola Flux / Reducer
- Logika perubahan data yang memiliki banyak cabang kondisi bisnis yang kompleks.
- Aplikasi yang mewajibkan riwayat audit log transaksi atau fitur Undo/Redo.

### Kapan Menggunakan Pola Finite State Machine (XState)
- Formulir multi-langkah (*multi-step wizard*), alur checkout pembayaran, pemutar video dengan banyak status transisi (`idle`, `buffering`, `playing`, `paused`, `error`).

### Kapan Cukup Menggunakan Variabel Lokal Sederhana
- Fitur CRUD dasar atau prototipe cepat di mana siklus data hanya dibaca dan ditampilkan tanpa transformasi rumit.

## 8. Metrik Kunci & Observabilitas

- **State Mutation Frequency**: Memantau berapa banyak action yang dipicu per detik di Redux DevTools. Jika ada action yang terpicu 60 kali per detik di event scroll, itu indikasi masalah arsitektur.
- **Immer / Structural Sharing Efficiency**: Memastikan properti objek yang tidak berubah mempertahankan referensi memori yang sama (`oldState.user === newState.user`).

## 9. Studi Kasus Industri Nyata

### Reduksi Bug Status pada Alur Checkout Airbnb

- **Tantangan Rekayasa Riil**:
  Alur pemesanan kamar Airbnb memiliki banyak status validasi: verifikasi identitas, ketersediaan tanggal kalender, kalkulasi kupon diskon, dan otorisasi kartu kredit. Saat menggunakan boolean flag biasa (`isLoading`, `isVerified`, `isDateAvailable`), sering terjadi kondisi mustahil (*impossible states*) di mana tombol pembayaran aktif padahal tanggal kamar baru saja diambil pengguna lain.
- **Implementasi Solusi Industri**:
  Airbnb menerapkan pola **Finite State Machine (FSM)**. Alur checkout dimodelkan sebagai grafik status yang kaku:
  `IDLE -> SELECTING_DATES -> VERIFYING_AVAILABILITY -> PAYMENT_PENDING -> CONFIRMED | FAILED`.
  Aplikasi tidak lagi mengizinkan transisi langsung dari `SELECTING_DATES` ke `PAYMENT_PENDING` tanpa melewati verifikasi server secara deterministik.
- **Dampak & Hasil Sistem**:
  - Kasus reservasi ganda (*double booking*) berkurang drastis hingga nol.
  - Skenario pengujian otomatis (*test cases*) menjadi 100% deterministik dan mudah dipetakan.

## 10. Fakta Menarik & Gotchas Production

1. **Prinsip Immutability Bukan Deep Copy**:
   Dalam JavaScript, membuat salinan immutable menggunakan spread operator (`{ ...state, user: { ...state.user } }`) melakukan teknik yang disebut **Structural Sharing**. Properti yang tidak diubah (misal array produk) tidak disalin ulang memorinya, melainkan tetap berbagi referensi pointer yang sama dengan state lama. Ini alasan mengapa immutability di JS tetap hemat memori jika dilakukan dengan benar!
2. **Library Immer**:
   Sebagian besar toolkit modern (seperti Redux Toolkit) menyertakan library **Immer** di bawah kap. Anda diizinkan menulis kode mutasi langsung (`state.todos.push(newTodo)`), dan Immer menggunakan JavaScript `Proxy` untuk secara ajaib menghasilkan salinan immutable di belakang layar secara otomatis.

## 11. Rangkuman Singkat

1. **Aliran Data Teratur**: Pola manajemen state menetapkan aturan disiplin tentang bagaimana data boleh diubah di frontend.
2. **Flux & Unidirectional**: Aliran satu arah (Action -> Reducer -> Store -> View) menjamin prediktabilitas mutasi.
3. **Cegah Status Mustahil**: Gunakan Finite State Machine untuk alur transaksi kritis agar UI tidak menampilkan status ganda yang keliru.
4. **Manfaatkan Structural Sharing**: Terapkan Immutability untuk mendapatkan keuntungan time-travel debugging tanpa membuang memori RAM secara boros.
