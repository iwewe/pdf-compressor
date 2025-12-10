# PM – PDF Uploader & Compressor (≤ 32 MB, Temp 10 Menit)

## 1. Gambaran Umum Proyek

**Tujuan:**  
Membangun layanan web sederhana untuk mengunggah file PDF, mengompresnya agar ukuran akhir **tidak lebih dari 32 MB**, mempersingkat nama file (maksimal **32 karakter**), dan menyimpan file hanya secara **sementara selama maksimum 10 menit** sebelum otomatis dihapus.

**Lingkungan:**  
- Server: Ubuntu (on-prem / VPS)  
- Stack: Bebas (backend bisa Node.js / Python / PHP / Go, dsb.)  
- UI/UX: Sederhana, minimalis, fokus pada kemudahan penggunaan

---

## 2. Kebutuhan Fungsional

1. **Upload PDF**
   - [ ] Pengguna dapat mengunggah satu atau beberapa file PDF.
   - [ ] Validasi tipe file (hanya `application/pdf`).
   - [ ] Validasi ukuran awal file (misal, hard limit maksimal 200 MB atau sesuai kebijakan).

2. **Kompresi PDF**
   - [ ] Sistem mengompresi file PDF yang diunggah.
   - [ ] Target ukuran akhir **≤ 32 MB**.
   - [ ] Jika setelah kompres tetap > 32 MB:
     - [ ] Tampilkan pesan bahwa file tidak bisa dikompres di bawah batas 32 MB.
     - [ ] Opsional: tawarkan opsi unduh versi terkompresi apa adanya + warning.
   - [ ] Pilihan level kompresi (misal: normal / high-quality / maximum compression) – opsional.

3. **Pemendekan Nama File**
   - [ ] Nama file hasil kompres mengikuti aturan:
     - [ ] Maksimum 32 karakter (tanpa ekstensi).
     - [ ] Karakter tidak aman (spasi, simbol aneh) dinormalisasi (misal: spasi → `_`, huruf kecil semua).
     - [ ] Jika nama asli > 32 karakter, dipotong dan mungkin ditambah hash pendek untuk mencegah duplikasi.
   - [ ] Ekstensi `.pdf` tetap dipertahankan.

4. **Penyimpanan Sementara**
   - [ ] File yang terkompresi disimpan di folder temporer (misal `/tmp/pdf-uploader/` atau direktori khusus).
   - [ ] Setiap file memiliki **timestamp** pembuatan/kompresi.
   - [ ] File otomatis dihapus setelah **maksimal 10 menit**.
   - [ ] Mekanisme cleanup:
     - [ ] Background job (cron/systemd timer) atau scheduler di dalam aplikasi.
     - [ ] Fungsi yang mengecek semua file dan menghapus yang `now - created_at > 10 menit`.

5. **Link Unduh & Informasi Hasil**
   - [ ] Setelah kompresi berhasil, UI menampilkan:
     - [ ] Nama file pendek.
     - [ ] Ukuran file sebelum & sesudah kompres.
     - [ ] Persentase pengurangan ukuran.
     - [ ] Tautan unduh (URL unik, misal `/download/{token}`).
   - [ ] Link unduh hanya valid selama file masih ada (maks 10 menit).
   - [ ] Jika link diakses setelah file dihapus:
     - [ ] Tampilkan pesan “file sudah kedaluwarsa”.

6. **UI/UX**
   - [ ] Halaman tunggal (single-page) sederhana:
     - [ ] Area drag-and-drop + tombol “Pilih File”.
     - [ ] Daftar file yang sedang / telah diproses.
     - [ ] Indikator progress upload (progress bar per file).
     - [ ] Indikator status (uploading, compressing, done, failed).
   - [ ] Notifikasi error yang jelas (file bukan PDF, terlalu besar, gagal kompres, dsb.).
   - [ ] Opsional: tampilan riwayat file yang diunggah dalam sesi tersebut (selama halaman belum di-refresh).

7. **API / Integrasi**
   - [ ] Endpoint `POST /upload` untuk menerima file PDF.
   - [ ] Endpoint `GET /download/{id|token}` untuk mengunduh file terkompres.
   - [ ] Opsional: API `GET /status/{id|token}` untuk memeriksa status kompresi.
   - [ ] Opsional: API key sederhana jika layanan akan dikonsumsi sistem lain.

---

## 3. Kebutuhan Non-Fungsional

1. **Keamanan**
   - [ ] Validasi tipe file dan magic bytes (hindari upload file berbahaya yang disamarkan sebagai PDF).
   - [ ] Batasi ukuran upload maksimum per request.
   - [ ] Sanitasi nama file sebelum disimpan.
   - [ ] Gunakan direktori temporer yang tidak dapat dieksekusi (`noexec`) jika memungkinkan.
   - [ ] Rate limiting untuk mencegah penyalahgunaan (DoS/bruteforce).
   - [ ] HTTPS untuk semua akses UI dan API.

2. **Kinerja & Skalabilitas**
   - [ ] Proses kompresi tidak memblokir server utama (bisa pakai worker / job queue untuk beban besar).
   - [ ] Batasan jumlah file per batch upload.
   - [ ] Monitoring CPU/RAM, karena kompresi PDF cukup berat.

3. **Reliabilitas**
   - [ ] Graceful error handling saat kompresi gagal.
   - [ ] Logging untuk semua error dan operasi penting.
   - [ ] Mekanisme restart otomatis (systemd / PM2 / supervisor, dsb.).

4. **Observability**
   - [ ] Log akses dan error tersimpan di file log terpisah.
   - [ ] Metric dasar: jumlah upload per jam, rasio keberhasilan kompresi, rata-rata waktu proses.
   - [ ] Opsional: integrasi dengan tools monitoring (Prometheus, Grafana, dsb.).

---

## 4. Desain Arsitektur Tingkat Tinggi

### 4.1 Komponen Utama

- **Frontend (Web UI)**
  - HTML/CSS/JS sederhana (bisa pakai framework ringan: Alpine.js / Vue / React jika perlu).
  - Fitur drag-and-drop, progress bar, daftar file dan status.

- **Backend Service**
  - REST API untuk upload, status, dan download.
  - Modul kompresi PDF:
    - Menggunakan tool CLI (misal: Ghostscript, qpdf) atau library (misal: `pdf-lib`, `pikepdf`, dsb.).
    - Menyesuaikan parameter kompresi (downsampling, quality, dsb.).
  - Modul penamaan file (normalisasi + pemotongan + hash pendek).
  - Modul cleanup file temporer.

- **Storage**
  - Direktori lokal (misal `/var/tmp/pdf-uploader/`).
  - Opsional: gunakan subfolder per hari untuk memudahkan pembersihan manual.

- **Scheduler / Cleanup**
  - Cron job tiap menit:
    - Menjalankan script untuk menghapus file yang lebih tua dari 10 menit.
  - Atau scheduler internal di aplikasi yang berjalan periodik.

### 4.2 Opsi Stack Teknis (Contoh)

Hanya contoh, bisa diganti sesuai preferensi:

- **Opsi A – Node.js**
  - Backend: Node.js + Express
  - Kompresi: panggil Ghostscript via child process.
  - Frontend: HTML + Tailwind + sedikit vanilla JS.

- **Opsi B – Python**
  - Backend: FastAPI / Flask
  - Kompresi: Ghostscript / qpdf dipanggil via subprocess.
  - Frontend: template Jinja2 + JS minimal.

- **Opsi C – PHP**
  - Backend: Laravel / Lumen / Slim.
  - Kompresi: eksekusi Ghostscript via `exec()` dengan sanitasi parameter ketat.

---

## 5. Brainstorming Fitur

### 5.1 Fitur Inti (MVP)

- [ ] Upload file PDF (single & multi-file).
- [ ] Kompresi otomatis hingga target ≤ 32 MB.
- [ ] Pemendekan nama file hingga 32 karakter.
- [ ] Tampilan ukuran sebelum/sesudah + persentase kompresi.
- [ ] Download link dengan masa aktif 10 menit.
- [ ] Auto-delete file setelah 10 menit.
- [ ] UI sederhana: drag-and-drop, progress bar, status.

### 5.2 Fitur Tambahan (Nice-to-Have)

- [ ] Pilihan level kompresi (Low, Medium, High).
- [ ] Pratinjau halaman pertama PDF (thumbnail).
- [ ] Dark mode toggle.
- [ ] Multi-bahasa (misal: ID/EN).
- [ ] Riwayat upload selama sesi browser (tanpa login).
- [ ] Notifikasi browser ketika kompresi selesai (jika proses lama).
- [ ] Dukungan integrasi API dengan token sederhana.
- [ ] Batas kuota per IP (misal maksimum 50 file/hari).

### 5.3 Fitur Admin / Ops

- [ ] Dashboard sederhana untuk:
  - [ ] Melihat statistik penggunaan (jumlah file, ukuran rata-rata, dsb.).
  - [ ] Melihat log error terbaru.
- [ ] Health check endpoint (`/health`):
  - [ ] Mengecek disk space, akses ke tool kompresi, dsb.

---

## 6. Rencana Kerja (Project Management / DevOps Style)

### 6.1 Tahap Inisiasi

- [ ] Konfirmasi kebutuhan bisnis:
  - [ ] Batas maksimal ukuran upload sebelum kompres.
  - [ ] Batas jumlah file per upload.
  - [ ] Apakah perlu autentikasi pengguna atau publik bebas akses.
- [ ] Definisikan **Definition of Done (DoD)** dan acceptance criteria.
- [ ] Pilih stack utama (Node/Python/PHP/Go) dan tool kompresi (Ghostscript, qpdf, dll).

### 6.2 Tahap Desain

- [ ] Desain arsitektur logis:
  - [ ] Diagram alur: Upload → Validasi → Kompresi → Simpan Temp → Download → Cleanup.
- [ ] Definisikan spesifikasi API:
  - [ ] `POST /upload`
  - [ ] `GET /download/{token}`
  - [ ] `GET /status/{token}` (opsional)
  - [ ] `GET /health`
- [ ] Desain skema direktori penyimpanan:
  - [ ] Struktur folder, format nama file, format metadata (jika ada).
- [ ] Desain UI/UX:
  - [ ] Wireframe halaman utama.
  - [ ] Pesan error dan success states.
- [ ] Desain mekanisme cleanup:
  - [ ] Cron job atau scheduler internal, plus script yang jelas.

### 6.3 Tahap Implementasi Backend

- [ ] Setup proyek backend (framework, struktur folder).
- [ ] Implementasi endpoint upload:
  - [ ] Parsing file upload (multipart).
  - [ ] Validasi tipe & ukuran.
  - [ ] Simpan file sementara sebelum kompresi.
- [ ] Implementasi modul kompresi:
  - [ ] Wrapper untuk tool kompresi (Ghostscript/qpdf).
  - [ ] Pengaturan parameter kompresi.
  - [ ] Loop atau logika untuk mencoba beberapa level kompresi jika diperlukan.
- [ ] Implementasi modul penamaan file:
  - [ ] Normalisasi karakter.
  - [ ] Potong ke 32 karakter.
  - [ ] Tambahkan suffix hash pendek bila perlu.
- [ ] Implementasi endpoint download:
  - [ ] Validasi token/id.
  - [ ] Pastikan file masih ada dan belum kedaluwarsa.
- [ ] Implementasi endpoint health check.
- [ ] Logging & error handling terstruktur.

### 6.4 Tahap Implementasi Frontend

- [ ] Buat halaman utama dengan:
  - [ ] Area drag-and-drop + tombol “Pilih File”.
  - [ ] Tabel/list file yang diunggah + status.
  - [ ] Progress bar per file.
- [ ] Integrasi dengan API upload (AJAX/fetch).
- [ ] Menampilkan hasil kompres:
  - [ ] Nama file pendek.
  - [ ] Ukuran sebelum/sesudah.
  - [ ] Link unduh.
- [ ] Tampilkan pesan error yang ramah pengguna.

### 6.5 Tahap DevOps & Deployment

- [ ] Siapkan server Ubuntu:
  - [ ] Update paket (`apt update && apt upgrade`).
  - [ ] Install runtime (Node/Python/PHP, dsb.).
  - [ ] Install tool kompresi (Ghostscript / qpdf).
- [ ] Setup reverse proxy:
  - [ ] Nginx / Caddy untuk HTTPS, gzip, dsb.
  - [ ] Konfigurasi domain & SSL (Let’s Encrypt).
- [ ] Setup sistem service:
  - [ ] systemd service / PM2 / supervisor untuk menjalankan backend.
- [ ] Setup job cleanup:
  - [ ] Cron job setiap menit/5 menit untuk menghapus file > 10 menit.
  - [ ] Script shell yang aman untuk `find` & `delete`.
- [ ] Setup CI/CD (opsional):
  - [ ] Pipeline build & deploy otomatis dari Git repo.
- [ ] Konfigurasi logging:
  - [ ] Direktori log, rotasi log (logrotate).

### 6.6 Tahap Testing & QA

- [ ] Uji unit:
  - [ ] Fungsi validasi file.
  - [ ] Fungsi kompresi (mocked).
  - [ ] Fungsi pemendekan nama file.
- [ ] Uji integrasi:
  - [ ] Alur penuh upload → kompres → download.
- [ ] Uji beban rendah:
  - [ ] Upload banyak file kecil sekaligus.
- [ ] Uji edge-case:
  - [ ] File PDF besar yang sulit dikompres.
  - [ ] File rusak / bukan PDF.
  - [ ] File yang tepat 32 MB setelah kompres.
  - [ ] Akses link setelah 10 menit (harus tidak bisa).
- [ ] Uji keamanan dasar:
  - [ ] Coba upload file non-PDF.
  - [ ] Coba path traversal dalam nama file.
  - [ ] Coba flood upload (lihat efek rate limit).

### 6.7 Tahap Go-Live & Operasi

- [ ] Deploy ke server produksi.
- [ ] Pantau log untuk 1–2 hari pertama:
  - [ ] Error kompresi.
  - [ ] Lonjakan beban.
- [ ] Siapkan SOP operasional:
  - [ ] Cara restart service.
  - [ ] Cara memeriksa disk usage.
  - [ ] Cara memeriksa apakah cron cleanup berjalan.
- [ ] Dokumentasi singkat untuk pengguna (help page).

---

## 7. Checklist To-Do Singkat

**High-level To-Do (ringkas):**

- [ ] Finalisasi kebutuhan bisnis & batasan.
- [ ] Pilih stack & tool kompresi.
- [ ] Desain API & UI.
- [ ] Implementasi backend (upload, kompres, download, cleanup).
- [ ] Implementasi frontend (UI sederhana + integrasi API).
- [ ] Setup server Ubuntu + tool pendukung.
- [ ] Setup cleanup job 10 menit.
- [ ] Testing fungsional & keamanan dasar.
- [ ] Deployment & monitoring awal.
- [ ] Dokumentasi + SOP operasional.

---

## 8. Definition of Done (DoD) – Versi Awal

Fitur dianggap **selesai** jika:

1. [ ] Pengguna dapat mengunggah minimal 1 file PDF via UI dan menerima file terkompres.
2. [ ] Ukuran file setelah kompres **tidak melebihi 32 MB**, atau pengguna mendapat error yang jelas jika tidak memungkinkan.
3. [ ] Nama file hasil kompres maksimum 32 karakter (tanpa ekstensi) dan aman untuk sistem file.
4. [ ] File terkompres dihapus otomatis dalam waktu **≤ 10 menit** (terbukti melalui pengujian).
5. [ ] Log dasar tercatat: upload, hasil kompres, error.
6. [ ] Layanan berjalan stabil di Ubuntu dengan reverse proxy dan HTTPS.
7. [ ] Terdapat dokumentasi singkat:
   - [ ] Cara memakai (untuk user).
   - [ ] Cara menjalankan & merawat (untuk admin/ops).

