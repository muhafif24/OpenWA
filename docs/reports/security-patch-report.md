# Laporan Penambalan Keamanan (Security Patch Report)

**Tanggal:** 8 Juni 2026
**Repository:** `muhafif24/OpenWA`

Laporan ini mendokumentasikan penambalan untuk 5 kerentanan keamanan kritis yang ditemukan selama proses audit keamanan awal, sebagai persiapan sebelum proyek di-hosting secara publik (misal: di environment Proxmox VE).

## Ringkasan 5 Kerentanan & Solusinya

Meskipun dalam _Implementation Plan_ sempat dikelompokkan menjadi 4 poin utama, pada praktiknya seluruh **5 kerentanan spesifik** telah diperbaiki secara tuntas:

### 1. Path Traversal (Storage / File System)
*   **Masalah:** Penyerang dapat mengirimkan input seperti `../../../etc/passwd` atau `../.env` pada endpoint *storage* untuk membaca file sistem di luar direktori yang diizinkan.
*   **Lokasi:** `src/common/storage/storage.service.ts` dan `src/core/plugins/plugin-storage.service.ts`.
*   **Perbaikan:** Menambahkan utilitas `resolveSafePath` internal yang menggunakan fungsi `path.resolve` dari Node.js untuk memastikan *path* akhir selalu diawali (*startsWith*) dengan direktori *base* yang diizinkan (misalnya `./data`). Jika ada upaya melenceng, sistem langsung melempar _Error_.

### 2. Path Traversal (Infrastruktur / Import Data)
*   **Masalah:** Mirip dengan kerentanan pertama, parameter `filePath` pada metode `importStorage` bisa dimanipulasi untuk mengekstrak file *tar.gz* dari luar zona aman.
*   **Lokasi:** `src/modules/infra/infra.controller.ts`.
*   **Perbaikan:** Melakukan enkapsulasi `path.resolve(process.cwd(), 'data', filePath)` dan memvalidasi letaknya sebelum memberikan _stream_ baca.

### 3. Hilangnya Role-Based Access Control (RBAC) pada Endpoint Berbahaya
*   **Masalah:** Rute seperti mengubah konfigurasi server (`PUT /infra/config`) atau *restart* server bisa diakses oleh pemegang API Key level *Viewer* atau *Operator*, yang berpotensi memicu eskalasi hak istimewa (privilege escalation).
*   **Lokasi:** `src/modules/infra/infra.controller.ts`.
*   **Perbaikan:** Mengimpor dan membubuhkan decorator `@RequireRole(ApiKeyRole.ADMIN)` di atas metode krusial tersebut agar hanya Admin yang dapat mengeksekusinya.

### 4. Data Transfer Object (DTO) Config Tidak Divalidasi
*   **Masalah:** Parameter konfigurasi sistem (Database, Engine, Redis) menggunakan `interface` biasa, sehingga luput dari _ValidationPipe_ NestJS. Hal ini memungkinkan injeksi _payload_ aneh ke dalam sistem (misalnya _type mismatch_).
*   **Lokasi:** `src/modules/infra/infra.controller.ts` (pada `SaveConfigDto`).
*   **Perbaikan:** Mengubah `interface` menjadi `class` dan membekalinya dengan validasi ketat menggunakan _decorators_ bawaan `@nestjs` / `class-validator` (seperti `@IsString`, `@IsBoolean`, `@IsIn`).

### 5. Kombinasi Kerentanan API Key (Predictable Key & Plaintext Storage)
*   **Masalah:** 
    *   (a) Jika aplikasi berjalan tanpa `NODE_ENV=production`, ia akan menghasilkan kunci yang bisa ditebak: `dev-admin-key`.
    *   (b) Kunci tersebut disimpan di disk (`data/.api-key`) dengan file permissions _default_, sehingga _user_ Linux lain berpotensi bisa membacanya.
*   **Lokasi:** `src/modules/auth/auth.service.ts`.
*   **Perbaikan:** 
    *   (a) Memaksa sistem untuk **selalu** membuat *random string* hex (sepanjang 32 byte) sebagai _Fallback Key_, apa pun _environment_-nya.
    *   (b) Memanfaatkan parameter Node.js `fs.writeFileSync` dengan `mode: 0o600` untuk memastikan file hanya dapat dibaca dan ditulis oleh *owner/user* yang menjalankan proses tersebut.

---

> Laporan ini dibuat secara otomatis pasca-tindakan _git commit_ dan ditujukan sebagai bukti historis bahwa *codebase* sudah mencapai standar minimal keamanan untuk eksposur ke internet.
