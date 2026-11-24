# Frontend Engineering Handbook

Handbook ini dirancang sebagai panduan komprehensif, praktis, dan ramah bagi developer pemula (junior developer) untuk memahami konsep inti rekayasa frontend modern, mekanisme internal browser, arsitektur rendering web, manajemen state, optimasi performa, integrasi jaringan, keamanan klien, pengujian kualitas kode, infrastruktur tooling modern, ketahanan arsitektur, serta perancangan sistem Micro Frontends berskala enterprise.

Fokus handbook ini bukan sekadar menghafal sintaksis pustaka atau framework tertentu, melainkan membedah prinsip dasar: bagaimana runtime JavaScript mengeksekusi kode, bagaimana peramban mengonversi HTML dan CSS menjadi piksel di layar, bagaimana strategi rendering dipilih berdasarkan kebutuhan performa, bagaimana aliran data dikelola secara efisien, bagaimana mengamankan aplikasi dari kerentanan web modern, bagaimana menguji dan merilis aplikasi secara andal, serta bagaimana memecah aplikasi monolitik raksasa menjadi aplikasi mikro yang otonom.

***

## Peta Kurikulum Arsitektur (11 Kategori - 84 Topik)

```
+-------------------------------------------------------------------------+
| core-frontend-concepts/ (Fondasi Mesin Browser & Eksekusi JavaScript)   |
| 1. DOM                   2. Virtual DOM          3. Browser Rendering   |
| 4. Event Loop            5. Event Delegation     6. Bubbling/Capturing  |
| 7. Debouncing            8. Throttling           9. Web Workers         |
| 10. Web APIs                                                            |
+-------------------------------------------------------------------------+
                                    |
                                    v
+-------------------------------------------------------------------------+
| rendering-and-architecture/ (Strategi Pengiriman & Arsitektur UI)       |
| 11. CSR                  12. SSR                 13. SSG                |
| 14. ISR                  15. Hydration           16. Lazy Loading       |
| 17. Code Splitting       18. Dynamic Imports     19. Micro Frontends    |
| 20. Component Arch                                                      |
+-------------------------------------------------------------------------+
                                    |
                                    v
+-------------------------------------------------------------------------+
| state-management-and-data-fetching/ (Sinkronisasi & Manajemen Data)     |
| 21. Local State          22. Global State        23. Server State       |
| 24. State Patterns       25. Context API         26. Redux              |
| 27. Zustand              28. Pinia               29. TanStack Query     |
| 30. Optimistic Updates   31. Data Synchronization 32. Cache Invalidation|
+-------------------------------------------------------------------------+
                                    |
                                    v
+-------------------------------------------------------------------------+
| performance-optimization/ (Akselerasi & Efisiensi Resource Browser)     |
| 33. Core Web Vitals      34. Lazy Loading Images 35. Image Optimization |
| 36. Tree Shaking         37. Bundle Optimization 38. Memoization        |
| 39. Virtualization       40. Prefetch/Preload    41. CDN                |
| 42. Browser Caching                                                     |
+-------------------------------------------------------------------------+
                                    |
                                    v
+-------------------------------------------------------------------------+
| networking-and-communication/ (Protokol Jaringan & Keamanan Transmisi)  |
| 43. HTTP / HTTPS         44. REST API Integration 45. GraphQL           |
| 46. WebSocket            47. Server-Sent Events  48. CORS               |
| 49. Cookies & Sessions   50. JWT Authentication  51. CSRF               |
| 52. XSS                                                                 |
+-------------------------------------------------------------------------+
                                    |
                                    v
+-------------------------------------------------------------------------+
| frontend-security/ (Proteksi Klien & Pertahanan Browser)                |
| 53. Content Sec Policy   54. Input Sanitization  55. Output Encoding    |
| 56. Secure Token Storage 57. Clickjacking        58. Subresource Integ  |
+-------------------------------------------------------------------------+
                                    |
                                    v
+-------------------------------------------------------------------------+
| testing-and-code-quality/ (Penjaminan Kualitas & Otomasi Pengujian)     |
| 59. Unit Testing         60. Integration Testing 61. E2E Testing        |
| 62. Component Testing    63. Mocking API         64. Test Coverage      |
| 65. Accessibility (a11y) 66. Type Safety         67. Linting/Formatting |
+-------------------------------------------------------------------------+
                                    |
                                    v
+-------------------------------------------------------------------------+
| tooling-and-infrastructure/ (Tooling Modern & Ekosistem Rilis)          |
| 68. Build Tools          69. Environment Vars    70. CI/CD for Frontend |
| 71. Monorepo Arch                                                       |
+-------------------------------------------------------------------------+
                                    |
                                    v
+-------------------------------------------------------------------------+
| advanced-resilience-and-design/ (Ketahanan Sistem & Skalabilitas UI)    |
| 72. Memory Leaks         73. Error Boundaries    74. Frontend Monitoring|
| 75. Design Systems       76. Feature Flags                              |
+-------------------------------------------------------------------------+
                                    |
                                    v
+-------------------------------------------------------------------------+
| micro-frontend-architecture/ (Arsitektur Terdistribusi & Modul Federasi)|
| 77. Module Fed Internals 78. Cross-MFE Comm      79. MFE Routing        |
| 80. CSS/Asset Isolation  81. MFE Resilience      82. Dynamic Remotes    |
+-------------------------------------------------------------------------+
                                    |
                                    v
+-------------------------------------------------------------------------+
| advanced-frontend-patterns/ (Pola Arsitektur Frontier & Offline-First)  |
| 83. Progressive Web Apps (PWA) 84. Backend-for-Frontend (BFF)           |
+-------------------------------------------------------------------------+
```

***

## Daftar Isi Materi

### 1. Core Frontend Concepts

Fondasi mesin web, siklus hidup dokumen, event propagation, dan eksekusi paralel pada browser:

* [01. DOM (Document Object Model)](core-frontend-concepts/dom/README.md) - Representasi pohon objek dokumen HTML di memori browser yang dapat dimanipulasi oleh JavaScript.
* [02. Virtual DOM](core-frontend-concepts/virtual-dom/README.md) - Abstraksi pohon elemen ringan di memori RAM untuk meminimalkan biaya mutasi langsung ke Real DOM via diffing.
* [03. Browser Rendering Process](core-frontend-concepts/browser-rendering-process/README.md) - Tahapan kritis peramban mengubah kode mentah menjadi visual: DOM + CSSOM -> Render Tree -> Layout -> Paint -> Composite.
* [04. Event Loop](core-frontend-concepts/event-loop/README.md) - Mekanisme pengatur giliran eksekusi asynchronous antara Call Stack, Microtask Queue, dan Macrotask Queue.
* [05. Event Delegation](core-frontend-concepts/event-delegation/README.md) - Pola efisiensi memori dengan memasang satu handler di elemen induk untuk mengelola event anak via bubbling.
* [06. Event Bubbling & Capturing](core-frontend-concepts/event-bubbling-and-capturing/README.md) - Tiga fase perambatan event W3C di hierarki DOM: turun pada fase capture, tiba di target, dan merambat naik pada fase bubble.
* [07. Debouncing](core-frontend-concepts/debouncing/README.md) - Menunda eksekusi fungsi berat sampai pengguna berhenti melakukan aksi selama jeda waktu tertentu.
* [08. Throttling](core-frontend-concepts/throttling/README.md) - Membatasi eksekusi fungsi maksimal sekali dalam rentang interval waktu teratur untuk gerakan berulang.
* [09. Web Workers](core-frontend-concepts/web-workers/README.md) - Eksekusi JavaScript multithreading sejati di background thread tanpa memblokir antarmuka main thread.
* [10. Web APIs](core-frontend-concepts/web-apis/README.md) - Sekumpulan antarmuka bawaan lingkungan browser di luar JavaScript murni untuk akses hardware, storage, dan jaringan.

### 2. Rendering & Architecture

Strategi pengiriman HTML dan JavaScript dari server ke client, optimasi beban jaringan, serta modularitas aplikasi:

* [11. CSR (Client-Side Rendering)](rendering-and-architecture/client-side-rendering/README.md) - Pola rendering di mana pembuatan elemen UI dan manipulasi data dieksekusi sepenuhnya di browser pengguna.
* [12. SSR (Server-Side Rendering)](rendering-and-architecture/server-side-rendering/README.md) - Server menghasilkan dokumen HTML lengkap berisi data terkini di setiap request untuk FCP cepat dan SEO optimal.
* [13. SSG (Static Site Generation)](rendering-and-architecture/static-site-generation/README.md) - Menghasilkan seluruh file HTML statis pada saat build time untuk kecepatan TTFB maksimal via CDN.
* [14. ISR (Incremental Static Regeneration)](rendering-and-architecture/incremental-static-regeneration/README.md) - Kombinasi SSG dan SSR yang meregenerasi halaman statis secara bertahap di latar belakang tanpa full rebuild.
* [15. Hydration](rendering-and-architecture/hydration/README.md) - Proses browser menyambungkan Virtual DOM dan event listener interaktif ke HTML statis hasil kiriman server.
* [16. Lazy Loading](rendering-and-architecture/lazy-loading/README.md) - Menunda pemuatan aset atau komponen berat sampai elemen tersebut benar-benar dibutuhkan oleh pengguna.
* [17. Code Splitting](rendering-and-architecture/code-splitting/README.md) - Memecah bundle JavaScript tunggal menjadi potongan-potongan chunk kecil yang dimuat secara modular.
* [18. Dynamic Imports](rendering-and-architecture/dynamic-imports/README.md) - Sintaksis asinkron berbasis Promise untuk memuat modul JavaScript tertentu tepat saat interaksi terjadi.
* [19. Micro Frontends](rendering-and-architecture/micro-frontends/README.md) - Arsitektur memecah monolit antarmuka menjadi beberapa aplikasi mikro independen yang dikelola tim berbeda.
* [20. Component-Based Architecture](rendering-and-architecture/component-based-architecture/README.md) - Paradigma menyusun UI dari balok-balok Lego kecil yang mandiri, terisolasi, dan dapat digunakan kembali.

### 3. State Management & Data Fetching

Pemisahan domain data, reaktivitas, pustaka state global, manajemen server cache, dan integrasi backend:

* [21. Local State](state-management-and-data-fetching/local-state/README.md) - Data dinamis yang terisolasi dan hanya hidup di dalam satu komponen tunggal untuk mencegah re-render domino.
* [22. Global State](state-management-and-data-fetching/global-state/README.md) - Wadah data bersama di luar pohon komponen untuk menghilangkan prop drilling pada data aplikasi yang luas.
* [23. Server State](state-management-and-data-fetching/server-state/README.md) - Paradigma mengelola data milik server backend yang bersifat asinkron, rentan basi, dan membutuhkan caching.
* [24. State Management Patterns](state-management-and-data-fetching/state-management-patterns/README.md) - Pola desain arsitektur aliran data (Flux unidirectional, immutability, proxy reaktif, dan finite state machines).
* [25. Context API](state-management-and-data-fetching/context-api/README.md) - Mekanisme injeksi dependensi bawaan React untuk menyalurkan data berfrekuensi perubahan rendah tanpa pustaka pihak ketiga.
* [26. Redux](state-management-and-data-fetching/redux/README.md) - Manajemen state global berbasis aksi dan fungsi reducer murni dengan kemampuan time-travel debugging teruji.
* [27. Zustand](state-management-and-data-fetching/zustand/README.md) - Pustaka state global minimalis 1 KB tanpa pembungkus Provider yang mendukung selektor atomik berperforma tinggi.
* [28. Pinia](state-management-and-data-fetching/pinia/README.md) - Pustaka manajemen state global resmi untuk Vue 3 yang modular, type-safe, dan mendukung Composition API.
* [29. TanStack Query](state-management-and-data-fetching/tanstack-query/README.md) - Mesin pengelola data asinkron dengan fitur caching otomatis, deduplikasi request, dan background revalidation.
* [30. Optimistic Updates](state-management-and-data-fetching/optimistic-updates/README.md) - Memperbarui tampilan UI seketika sebelum konfirmasi server tiba untuk menciptakan pengalaman zero-latency.
* [31. Data Synchronization](state-management-and-data-fetching/data-synchronization/README.md) - Menjaga konsistensi data antara browser dan database menggunakan polling, SSE, WebSockets, atau CRDT.
* [32. Cache Invalidation](state-management-and-data-fetching/cache-invalidation/README.md) - Disiplin membatalkan atau menandai cache kedaluwarsa setelah mutasi agar UI tidak menyajikan data usang.

### 4. Performance Optimization

Akselerasi performa halaman, reduksi ukuran unduhan, efisiensi memori, dan penyaluran aset:

* [33. Core Web Vitals](performance-optimization/core-web-vitals/README.md) - Metrik standar industri Google (LCP, INP, CLS) untuk mengukur kecepatan pemuatan, interaktivitas, dan stabilitas visual.
* [34. Lazy Loading Images](performance-optimization/lazy-loading-images/README.md) - Penundaan pemuatan file gambar di luar viewport menggunakan atribut native atau teknik Low-Quality Image Placeholder.
* [35. Image Optimization](performance-optimization/image-optimization/README.md) - Strategi format modern (WebP/AVIF), kompresi visual, dan penyajian resolusi responsif via tag picture dan srcset.
* [36. Tree Shaking](performance-optimization/tree-shaking/README.md) - Eliminasi kode mati (dead-code elimination) oleh bundler dengan memanfaatkan struktur statis ES Modules.
* [37. Bundle Optimization](performance-optimization/bundle-optimization/README.md) - Teknik kompresi (Brotli/Gzip), minifikasi, dan analisis treemap untuk memangkas ukuran unduhan JavaScript.
* [38. Memoization](performance-optimization/memoization/README.md) - Teknik caching hasil kalkulasi komputasi mahal dan stabilisasi referensi fungsi/objek antar siklus render.
* [39. Virtualization](performance-optimization/virtualization/README.md) - Teknik windowing list yang hanya merender elemen DOM yang sedang tampak di viewport untuk menangani ribuan baris data.
* [40. Prefetching & Preloading](performance-optimization/prefetching-and-preloading/README.md) - Memberikan petunjuk dini ke browser untuk mengunduh aset kritis lebih awal via tag link rel preload dan prefetch.
* [41. CDN (Content Delivery Network)](performance-optimization/cdn/README.md) - Jaringan server edge terdistribusi geografis yang menyajikan aset statis dengan latensi jaringan minimal.
* [42. Browser Caching](performance-optimization/browser-caching/README.md) - Konfigurasi header HTTP Cache-Control, ETag, dan Service Worker Cache API untuk mencegah pengunduhan ulang.

### 5. Networking & Communication

Protokol transmisi data, integrasi API, mekanisme real-time, dan pertahanan lintas domain:

* [43. HTTP / HTTPS](networking-and-communication/http-https/README.md) - Fondasi protokol komunikasi web, perbedaan HTTP/1.1 pipeline, HTTP/2 multiplexing, dan HTTP/3 berbasis QUIC UDP.
* [44. REST API Integration](networking-and-communication/rest-api-integration/README.md) - Pola integrasi HTTP client modern menggunakan Fetch/Axios, penanganan timeout, dan request/response interceptors.
* [45. GraphQL](networking-and-communication/graphql/README.md) - Bahasa query data deklaratif yang memungkinkan klien meminta data presisi tanpa masalah over-fetching atau under-fetching.
* [46. WebSocket](networking-and-communication/websocket/README.md) - Saluran komunikasi dua arah (full-duplex) berlatensi sangat rendah melalui satu koneksi TCP persisten.
* [47. Server-Sent Events (SSE)](networking-and-communication/server-sent-events/README.md) - Aliran transmisi data satu arah dari server ke klien melalui koneksi HTTP persisten standar.
* [48. CORS (Cross-Origin Resource Sharing)](networking-and-communication/cors/README.md) - Mekanisme keamanan browser yang mengatur izin akses permintaan sumber daya antar-origin berbeda via preflight request.
* [49. Cookies & Sessions](networking-and-communication/cookies-and-sessions/README.md) - Mekanisme penyimpanan status sesi dengan atribut keamanan HttpOnly, Secure, SameSite, dan Domain scope.
* [50. JWT Authentication](networking-and-communication/jwt-authentication/README.md) - Struktur token digital mandiri (Header.Payload.Signature) dan pola rotasi token menggunakan short-lived access token.
* [51. CSRF (Cross-Site Request Forgery)](networking-and-communication/csrf/README.md) - Serangan eksploitasi kredensial otomatis peramban dan mitigasinya menggunakan SameSite cookies dan anti-CSRF token.
* [52. XSS (Cross-Site Scripting)](networking-and-communication/xss/README.md) - Varian serangan injeksi skrip berbahaya (Stored, Reflected, DOM-based) dan implikasinya pada transport kredensial.

### 6. Frontend Security

Instrumen perlindungan klien, pertahanan injeksi kode, penyimpanan token aman, dan integritas aset:

* [53. Content Security Policy (CSP)](frontend-security/content-security-policy/README.md) - Header proteksi peramban yang membatasi asal domain eksekusi skrip, gaya tampilan, dan koneksi jaringan.
* [54. Input Sanitization](frontend-security/input-sanitization/README.md) - Proses pembersihan muatan payload pengguna dari tag berbahaya sebelum diproses menggunakan pustaka teruji seperti DOMPurify.
* [55. Output Encoding](frontend-security/output-encoding/README.md) - Konversi karakter spesial HTML/JS/CSS menjadi entitas aman sebelum dicetak ke dalam pohon DOM untuk menetralkan injeksi.
* [56. Secure Token Storage](frontend-security/secure-token-storage/README.md) - Analisis komparasi penyimpanan token autentikasi antara LocalStorage, SessionStorage, In-Memory, dan HttpOnly Cookie.
* [57. Clickjacking](frontend-security/clickjacking/README.md) - Serangan penipuan antarmuka menggunakan iframe transparan dan mitigasinya dengan header X-Frame-Options serta CSP frame-ancestors.
* [58. Subresource Integrity (SRI)](frontend-security/subresource-integrity/README.md) - Verifikasi kriptografi hash SHA pada tag script CDN untuk memastikan file pustaka eksternal tidak dimanipulasi peretas.

### 7. Testing & Code Quality

Penjaminan kualitas kode, otomasi pengujian berjenjang, aksesibilitas, dan standarisasi sintaks:

* [59. Unit Testing](testing-and-code-quality/unit-testing/README.md) - Pengujian terisolasi fungsi logika murni, utilitas kalkulasi, dan reducers menggunakan siklus Arrange-Act-Assert.
* [60. Integration Testing](testing-and-code-quality/integration-testing/README.md) - Verifikasi kolaborasi harmonis antar komponen antarmuka, state management, dan jaringan data mock.
* [61. E2E Testing](testing-and-code-quality/e2e-testing/README.md) - Otomasi browser nyata (Playwright) untuk memvalidasi alur bisnis paling kritis dari kacamata pengguna.
* [62. Component Testing](testing-and-code-quality/component-testing/README.md) - Pengujian komponen UI berbasis role aksesibilitas dan simulasi interaksi pengguna yang refactor-resilient.
* [63. Mocking API](testing-and-code-quality/mocking-api/README.md) - Simulasi respons server di tingkat protokol jaringan menggunakan Mock Service Worker (MSW) tanpa mengotori kode produksi.
* [64. Test Coverage](testing-and-code-quality/test-coverage/README.md) - Metrik radar pelacak titik buta kode dengan fokus utama pada branch coverage dibanding sekadar mengejar 100% line coverage.
* [65. Accessibility (a11y)](testing-and-code-quality/accessibility-a11y/README.md) - Penerapan prinsip POUR WCAG, navigasi keyboard penuh, HTML semantik, dan atribut ARIA untuk web inklusif.
* [66. Type Safety](testing-and-code-quality/type-safety/README.md) - Integrasi pemeriksaan tipe statis compile-time TypeScript dan validasi gerbang data runtime via skema Zod.
* [67. Linting & Formatting](testing-and-code-quality/linting-and-formatting/README.md) - Standarisasi kualitas logika (ESLint) dan estetika tata letak visual (Prettier) yang dikawal oleh Git pre-commit hooks.

### 8. Tooling & Infrastructure

Infrastruktur bundler modern, pengelolaan variabel lingkungan, otomasi rilis CI/CD, dan arsitektur repositori:

* [68. Build Tools (Vite, Webpack, esbuild)](tooling-and-infrastructure/build-tools/README.md) - Evolusi mesin kompilasi web dari bundler tradisional berbasis berkas hingga dev server Native ESM berkecepatan tinggi.
* [69. Environment Variables](tooling-and-infrastructure/environment-variables/README.md) - Konfigurasi multi-lingkungan yang dicetak saat build time serta pemahaman batas keamanan variabel klien publik.
* [70. CI/CD for Frontend](tooling-and-infrastructure/ci-cd/README.md) - Otomasi alur validasi kualitas (lint, test, build) pada Pull Request dan penyaluran rilis otomatis ke edge network.
* [71. Monorepo Architecture](tooling-and-infrastructure/monorepo-architecture/README.md) - Pengelolaan multi-aplikasi dan paket bersama dalam satu repositori Git via pnpm workspaces dan Turborepo.

### 9. Advanced Resilience & Design

Strategi ketahanan sistem aplikasi web, isolasi kesalahan, pemantauan produksi, dan skalabilitas desain:

* [72. Memory Leaks](advanced-resilience-and-design/memory-leaks/README.md) - Deteksi dan pencegahan akumulasi memori RAM tak terpakai akibat uncleared listener, detached DOM, dan closure usang.
* [73. Error Boundaries](advanced-resilience-and-design/error-boundaries/README.md) - Komponen penangkap kesalahan deklaratif untuk mengisolasi kegagalan rendering dan mencegah White Screen of Death.
* [74. Frontend Monitoring & Error Tracking](advanced-resilience-and-design/monitoring-and-error-tracking/README.md) - Real User Monitoring (RUM) via Sentry untuk menangkap crash, breadcrumbs, dan session replay dengan source maps aman.
* [75. Design Systems](advanced-resilience-and-design/design-systems/README.md) - Fondasi skalabilitas antarmuka melalui Design Tokens, pustaka komponen headless aksesibel, dan dokumentasi Storybook.
* [76. Feature Flags](advanced-resilience-and-design/feature-flags/README.md) - Pemisahan proses deployment dari release bisnis, mendukung Trunk-Based Development, rollout bertahap, dan kill switch instan.

### 10. Micro Frontend Architecture

Arsitektur sistem web terdistribusi, federasi modul runtime, isolasi styling, dan koordinasi container:

* [77. Module Federation Internals](micro-frontend-architecture/module-federation-internals/README.md) - Mekanisme internal shared scope runtime, negosiasi versi semantik, dan resolusi singleton dependency di browser.
* [78. Cross-MFE Communication](micro-frontend-architecture/cross-mfe-communication/README.md) - Pola pertukaran data terdekupel antar sub-aplikasi menggunakan native DOM CustomEvents dan URL state tanpa shared global store.
* [79. Micro-Frontend Routing](micro-frontend-architecture/micro-frontend-routing/README.md) - Penyelarasan navigasi riwayat browser antara BrowserRouter Host dan MemoryRouter Sub-App dengan dukungan deep linking.
* [80. CSS and Asset Isolation](micro-frontend-architecture/css-and-asset-isolation/README.md) - Proteksi benturan styling CSS global (Tailwind reset vs Bootstrap) via PostCSS prefixing, Shadow DOM, dan resolusi publicPath aset.
* [81. MFE Resilience and Fallbacks](micro-frontend-architecture/mfe-resilience-and-fallbacks/README.md) - Isolasi dampak kerusakan (blast radius containment) dan circuit breaker saat container remote mengalami timeout atau HTTP 504.
* [82. Dynamic Remotes Management](micro-frontend-architecture/dynamic-remotes-management/README.md) - Service discovery frontend via Manifest API runtime untuk arsitektur multi-environment, canary rollout, dan rollback instan.

### 11. Advanced Frontend Patterns

Pola arsitektur tingkat lanjut untuk ketahanan aplikasi offline-first dan integrasi orkestrasi backend yang berorientasi pada antarmuka pengguna:

* [83. Progressive Web Apps (PWA)](advanced-frontend-patterns/progressive-web-apps/README.md) - Arsitektur web offline-first memanfaatkan Service Worker, strategi CacheStorage, App Shell, dan instalabilitas beranda native.
* [84. Backend-for-Frontend (BFF)](advanced-frontend-patterns/backend-for-frontend/README.md) - Layer orkestrasi API yang dimiliki tim frontend untuk agregasi microservices, pemangkasan payload, proteksi token otentikasi, dan normalisasi error.

***

## Panduan Belajar

1. **Pelajari Secara Berurutan**: Disarankan untuk memulai dari Kategori 1 guna memahami keterbatasan single-thread JavaScript dan pipa rendering browser sebelum mempelajari framework atau arsitektur tingkat lanjut.
2. **Evaluasi Trade-offs**: Setiap keputusan arsitektur membawa konsekuensi. Periksa bagian kelebihan, kekurangan, serta kriteria kapan harus pakai vs jangan pakai pada masing-masing modul.
3. **Validasi Melalui DevTools**: Ujilah pemahaman teori dengan membuka Chrome DevTools (panel Performance, Network, Memory, dan Rendering) untuk memverifikasi perilaku alokasi memori dan eksekusi thread secara nyata.
