# PM – PDF Uploader & Compressor (≤ 32 MB, Temp 10 Menit)

## 1. Gambaran Umum Proyek

**Tujuan ringkas:**
- Layanan web sederhana untuk unggah PDF ➜ kompres (hasil **≤ 32 MB**) ➜ rename (maks **32 karakter**) ➜ simpan sementara **≤ 10 menit** lalu hapus otomatis.

**Lingkungan:**
- Server: Ubuntu (on-prem / VPS)
- Stack: Bebas (backend bisa Node.js / Python / PHP / Go, dsb.)
- UI/UX: Sederhana, minimalis, fokus pada kemudahan penggunaan
- Penyimpanan: direktori temporer lokal (dihapus berkala)

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
  - HTML/CSS/JS sederhana (framework opsional: Alpine.js / Vue / React).
  - Fitur drag-and-drop, progress bar, daftar file, status per file.
  - Menampilkan ringkasan hasil kompresi + tombol unduh.

- **Backend Service**
  - REST API untuk upload, status, dan download.
  - Modul kompresi PDF:
    - Tool CLI (Ghostscript / qpdf) atau library (`pikepdf`, `pdf-lib`).
    - Parameter kompresi dapat di-tune berdasarkan target 32 MB.
  - Modul penamaan file (normalisasi + pemotongan + hash pendek).
  - Modul cleanup file temporer (job scheduler internal atau cron).

- **Storage**
  - Direktori lokal (misal `/var/tmp/pdf-uploader/` dengan subfolder tanggal untuk memudahkan bersih-bersih).

- **Scheduler / Cleanup**
  - Cron job tiap menit atau worker internal:
    - Menghapus file yang lebih tua dari 10 menit.
    - Membersihkan metadata cache yang kadaluarsa.

---

## 5. Alur Utama

1. Pengguna buka halaman ➜ drop/pilih file PDF.
2. Frontend kirim `POST /upload` (multipart) ➜ terima respons job ID/token.
3. Backend simpan sementara ➜ validasi ➜ kompresi ➜ simpan hasil ke direktori temp.
4. Frontend polling (atau SSE/websocket opsional) `GET /status/{token}` untuk status upload/kompres.
5. Setelah selesai, pengguna klik link unduh dari UI (atau salin link token) ➜ `GET /download/{token}`.
6. Cron/scheduler hapus file + metadata yang berumur > 10 menit.

---

## 6. Rencana Implementasi & Manajemen Proyek

### 6.1 Scope MVP
- Satu halaman UI sederhana.
- Upload tunggal/bermultiple dengan progress.
- Kompresi Ghostscript preset tunggal, target 32 MB, error jika gagal.
- Cleanup internal interval (fallback cron di produksi).
- Download via token unik.

### 6.2 RACI Ringkas
- Product/Stakeholder: definisi kebutuhan & prioritas.
- Dev Backend: endpoint upload/kompres/download, cleanup, logging.
- Dev Frontend: UI drag-and-drop, progress, status, unduh.
- DevOps: provisioning Ubuntu, reverse proxy, TLS, job cleanup, logging & monitoring.
- QA: test fungsional, keamanan dasar, UAT.

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

**Opsional cepat (MVP minimal):**
- [ ] Satu service monolit sederhana (misal Express/FastAPI) dengan middleware upload.
- [ ] Ghostscript wrapper dengan preset kompresi tunggal.
- [ ] Scheduler berbasis interval di dalam proses (fallback cron untuk produksi).

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

**UX quick wins:**
- [ ] Progress bar per file + status teks (uploading/compressing/done/failed).
- [ ] Tombol “Salin link unduh”.
- [ ] Notifikasi snackbar untuk error dan sukses.

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

**Keamanan & kepatuhan baseline:**
- [ ] Non-root runtime user untuk service.
- [ ] Limitasi folder temp dengan `noexec,nodev,nosuid` bila memungkinkan.
- [ ] Rate limit & header keamanan dasar (CSP, X-Content-Type-Options, dsb.).
- [ ] Health check endpoint yang tidak bocorkan detail sensitif.

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

**UAT (User Acceptance Test) singkat:**
- [ ] Pengguna awam bisa selesai unggah ➜ kompres ➜ unduh tanpa panduan.
- [ ] UI tetap responsif pada jaringan lambat (simulasi throttling).

**Observability/monitoring check:**
- [ ] Log error terpusat + alert dasar (misal email/Slack) untuk kompresi gagal > X kali/5 menit.
- [ ] Metric: total upload, rata-rata waktu kompres, p99 waktu unduh, jumlah file gagal, disk usage temp.

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

**Backlog detail (bisa dibuat tiket terpisah):**
- [ ] FE-01: Halaman unggah dengan drag-and-drop + tombol pilih file.
- [ ] FE-02: Progress bar & status per file + notifikasi error.
- [ ] FE-03: Tabel hasil kompres + link unduh + copy link.
- [ ] BE-01: Endpoint upload (validasi PDF, size hard limit, simpan temp awal).
- [ ] BE-02: Modul kompresi + konfigurasi preset (target ≤ 32 MB) + fallback gagal.
- [ ] BE-03: Modul pemendekan nama file (normalize + truncate + hash pendek).
- [ ] BE-04: Endpoint download dengan token + validasi usia file 10 menit.
- [ ] BE-05: Job cleanup (cron/internal) + script shell aman.
- [ ] OPS-01: Provisioning server Ubuntu + runtime + Ghostscript/qpdf.
- [ ] OPS-02: Reverse proxy (Nginx/Caddy) + SSL.
- [ ] OPS-03: Logging & rotasi + alert dasar.
- [ ] QA-01: Test case fungsional & keamanan dasar.
- [ ] DOC-01: README deployment + SOP operasional.

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
