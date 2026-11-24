# Frontend Engineering Handbook

Handbook ini dirancang sebagai panduan komprehensif, praktis, dan ramah bagi developer pemula (junior developer) untuk memahami konsep inti rekayasa frontend modern, mekanisme internal browser, arsitektur rendering web, dan manajemen state berskala produksi.

Fokus handbook ini bukan sekadar menghafal sintaksis pustaka atau framework tertentu, melainkan membedah prinsip dasar: bagaimana runtime JavaScript mengeksekusi kode, bagaimana peramban mengonversi HTML dan CSS menjadi piksel di layar, bagaimana strategi rendering dipilih berdasarkan kebutuhan performa, serta bagaimana aliran data dikelola secara efisien tanpa menimbulkan pemborosan komputasi pada perangkat pengguna.

---

## Standar Format Pembahasan

Setiap topik di dalam handbook ini disusun secara konsisten menggunakan 11 bab pembahasan utama:

1. **Skenario Sistem Nyata**: Masalah konkret yang umum ditemui pada aplikasi web skala industri.
2. **Apa Itu dan Buat Apa?**: Penjelasan esensial mengenai konsep dan fungsinya dalam bahasa teknis yang lugas.
3. **Kenapa Harus Pakai Ini?**: Alasan teknis serta permasalahan mendasar browser yang diselesaikan.
4. **Cara Kerjanya (Diagram Alur ASCII)**: Langkah demi langkah proses internal menggunakan diagram teks ASCII murni.
5. **Cuplikan Logika Inti**: Implementasi kode JavaScript/TypeScript murni atau contoh framework idiomatik yang berfokus pada logika esensial.
6. **Kelebihan dan Kekurangan (Trade-offs)**: Kompromi teknis yang perlu dipertimbangkan sebelum mengadopsi suatu teknik.
7. **Kapan Harus Pakai vs Kapan Jangan Pakai**: Matriks panduan pengambilan keputusan untuk kebutuhan proyek sehari-hari.
8. **Metrik Kunci & Observabilitas**: Pemantauan dampak performa menggunakan Core Web Vitals (LCP, INP, CLS), DevTools Profiler, memory heap, dan network trace.
9. **Studi Kasus Industri Nyata**: Solusi arsitektural konkret dari produk teknologi terkemuka.
10. **Fakta Menarik & Gotchas Production**: Jebakan umum dan perilaku tak terduga yang kerap memicu bug di lingkungan produksi.
11. **Rangkuman Singkat**: Poin-poin intisari sebagai referensi cepat.

---

## Peta Kurikulum Arsitektur (3 Kategori - 32 Topik)

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
```

---

## Daftar Isi Materi

### 1. Core Frontend Concepts
Fondasi mesin web, siklus hidup dokumen, event propagation, dan eksekusi paralel pada browser:
- [01. DOM (Document Object Model)](core-frontend-concepts/dom/README.md)
- [02. Virtual DOM](core-frontend-concepts/virtual-dom/README.md)
- [03. Browser Rendering Process](core-frontend-concepts/browser-rendering-process/README.md)
- [04. Event Loop](core-frontend-concepts/event-loop/README.md)
- [05. Event Delegation](core-frontend-concepts/event-delegation/README.md)
- [06. Event Bubbling & Capturing](core-frontend-concepts/event-bubbling-and-capturing/README.md)
- [07. Debouncing](core-frontend-concepts/debouncing/README.md)
- [08. Throttling](core-frontend-concepts/throttling/README.md)
- [09. Web Workers](core-frontend-concepts/web-workers/README.md)
- [10. Web APIs](core-frontend-concepts/web-apis/README.md)

### 2. Rendering & Architecture
Strategi pengiriman HTML dan JavaScript dari server ke client, optimasi beban jaringan, serta modularitas aplikasi:
- [11. CSR (Client-Side Rendering)](rendering-and-architecture/client-side-rendering/README.md)
- [12. SSR (Server-Side Rendering)](rendering-and-architecture/server-side-rendering/README.md)
- [13. SSG (Static Site Generation)](rendering-and-architecture/static-site-generation/README.md)
- [14. ISR (Incremental Static Regeneration)](rendering-and-architecture/incremental-static-regeneration/README.md)
- [15. Hydration](rendering-and-architecture/hydration/README.md)
- [16. Lazy Loading](rendering-and-architecture/lazy-loading/README.md)
- [17. Code Splitting](rendering-and-architecture/code-splitting/README.md)
- [18. Dynamic Imports](rendering-and-architecture/dynamic-imports/README.md)
- [19. Micro Frontends](rendering-and-architecture/micro-frontends/README.md)
- [20. Component-Based Architecture](rendering-and-architecture/component-based-architecture/README.md)

### 3. State Management & Data Fetching
Pemisahan domain data, reaktivitas, pustaka state global, manajemen server cache, dan integrasi backend:
- [21. Local State](state-management-and-data-fetching/local-state/README.md)
- [22. Global State](state-management-and-data-fetching/global-state/README.md)
- [23. Server State](state-management-and-data-fetching/server-state/README.md)
- [24. State Management Patterns](state-management-and-data-fetching/state-management-patterns/README.md)
- [25. Context API](state-management-and-data-fetching/context-api/README.md)
- [26. Redux](state-management-and-data-fetching/redux/README.md)
- [27. Zustand](state-management-and-data-fetching/zustand/README.md)
- [28. Pinia](state-management-and-data-fetching/pinia/README.md)
- [29. TanStack Query](state-management-and-data-fetching/tanstack-query/README.md)
- [30. Optimistic Updates](state-management-and-data-fetching/optimistic-updates/README.md)
- [31. Data Synchronization](state-management-and-data-fetching/data-synchronization/README.md)
- [32. Cache Invalidation](state-management-and-data-fetching/cache-invalidation/README.md)

---

## Panduan Belajar

1. **Pelajari Secara Berurutan**: Disarankan untuk memulai dari Kategori 1 guna memahami keterbatasan single-thread JavaScript dan pipa rendering browser sebelum mempelajari framework atau arsitektur tingkat lanjut.
2. **Evaluasi Trade-offs**: Setiap keputusan arsitektur membawa konsekuensi. Periksa bagian kelebihan, kekurangan, serta kriteria kapan harus pakai vs jangan pakai pada masing-masing modul.
3. **Validasi Melalui DevTools**: Ujilah pemahaman teori dengan membuka Chrome DevTools (panel Performance, Network, Memory, dan Rendering) untuk memverifikasi perilaku alokasi memori dan eksekusi thread secara nyata.
