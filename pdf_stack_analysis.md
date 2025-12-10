# Analisis Stack untuk PDF Uploader & Compressor

## 🎯 Rekomendasi Utama: **Go + Ghostscript**

### Alasan Utama
- **Memory footprint paling kecil** (~10-20 MB idle)
- **Single binary** - deploy mudah, no dependencies
- **Concurrency native** (goroutines) - handle multiple uploads efisien
- **Performance tinggi** untuk I/O operations
- **Resource usage stabil** saat kompresi berjalan

---

## 📊 Perbandingan Stack

### 1. **Go + Ghostscript** ⭐ RECOMMENDED

**Pros:**
- Memory idle: ~10-20 MB
- CPU efficient untuk concurrent operations
- Built-in HTTP server sangat cepat
- Single binary deployment (no runtime installation)
- Goroutines sempurna untuk background cleanup jobs
- Excellent untuk file operations

**Cons:**
- Learning curve jika tim belum familiar
- Ecosystem library lebih kecil vs Node/Python

**Resource Usage:**
```
Base: 10-20 MB RAM
Per request: +5-10 MB
Ghostscript process: +50-100 MB (temporary)
Total typical: 100-150 MB untuk 5-10 concurrent users
```

**Tools:**
- Framework: Fiber / Echo (atau std lib)
- PDF: exec Ghostscript via os/exec
- Cleanup: time.Ticker goroutine

---

### 2. **Python + FastAPI + Ghostscript**

**Pros:**
- Sintaks mudah dipahami
- Rich ecosystem untuk PDF (PyPDF2, pikepdf)
- FastAPI modern & cepat
- Script cleanup sederhana

**Cons:**
- Memory usage lebih besar (~40-80 MB idle)
- GIL limitation untuk CPU-bound tasks
- Perlu virtual environment management

**Resource Usage:**
```
Base: 40-80 MB RAM
Per request: +20-30 MB
Ghostscript: +50-100 MB
Total typical: 200-300 MB untuk 5-10 concurrent users
```

**Rating: ⭐⭐⭐⭐ (Good alternative)**

---

### 3. **Node.js + Express + Ghostscript**

**Pros:**
- JavaScript full-stack (familiar untuk web devs)
- NPM ecosystem besar
- Event-driven cocok untuk I/O
- PM2 untuk process management

**Cons:**
- Memory usage menengah (~50-100 MB idle)
- Single-threaded (perlu cluster/worker)
- V8 memory overhead

**Resource Usage:**
```
Base: 50-100 MB RAM
Per request: +15-25 MB
Ghostscript: +50-100 MB
Total typical: 250-350 MB untuk 5-10 concurrent users
```

**Rating: ⭐⭐⭐ (Acceptable)**

---

### 4. **PHP + Slim/Lumen + Ghostscript**

**Pros:**
- Lightweight untuk simple tasks
- Shared hosting compatible
- Mature ecosystem
- Process isolation per request

**Cons:**
- Process overhead jika pakai FPM
- Kurang ideal untuk long-running tasks
- Cleanup job perlu cron external

**Resource Usage:**
```
Base (FPM): 30-50 MB RAM
Per request: +10-20 MB
Ghostscript: +50-100 MB
Total typical: 200-250 MB
```

**Rating: ⭐⭐⭐ (Workable, tapi tidak optimal)**

---

## 🔧 Rekomendasi Tools Kompresi

### **Ghostscript** ⭐ RECOMMENDED
```bash
# Kompresi aggressive, target <32MB
gs -sDEVICE=pdfwrite \
   -dCompatibilityLevel=1.4 \
   -dPDFSETTINGS=/ebook \
   -dNOPAUSE -dQUIET -dBATCH \
   -sOutputFile=output.pdf \
   input.pdf
```

**Resource:** ~50-100 MB per proses
**Speed:** Cepat untuk file <50 MB

### Alternatif: **qpdf**
- Lebih ringan (~20-40 MB)
- Tapi kompresi kurang aggressive
- Cocok untuk optimize struktur PDF

---

## 💡 Optimasi Resource Tambahan

### 1. **Limit Concurrent Jobs**
```go
// Go example
semaphore := make(chan struct{}, 3) // Max 3 concurrent
```

### 2. **Streaming Upload**
- Jangan load seluruh file ke memory
- Stream langsung ke disk

### 3. **Cleanup Job Efisien**
```bash
# Cron setiap menit
* * * * * find /tmp/pdf-uploader -type f -mmin +10 -delete
```

### 4. **Nginx Buffer Settings**
```nginx
client_max_body_size 200M;
client_body_buffer_size 10M;
proxy_buffering off; # untuk large files
```

---

## 📈 Kesimpulan Resource Usage

| Stack | Idle RAM | Peak RAM (10 users) | CPU Baseline | Deploy Size |
|-------|----------|---------------------|--------------|-------------|
| **Go** | 10-20 MB | 150-200 MB | <5% | 10-15 MB |
| Python | 40-80 MB | 300-400 MB | 5-10% | 50-100 MB |
| Node.js | 50-100 MB | 350-450 MB | 5-10% | 30-50 MB |
| PHP | 30-50 MB | 250-350 MB | 5-15% | 20-40 MB |

---

## 🎯 Final Recommendation

**Untuk Ubuntu VPS dengan resource terbatas:**

### Pilihan 1: **Go + Ghostscript** (BEST)
- Server specs minimal: 512 MB RAM, 1 vCPU
- Cocok untuk: 20-50 concurrent users
- Deployment: Single binary + systemd

### Pilihan 2: **Python FastAPI** (GOOD)
- Server specs minimal: 1 GB RAM, 1 vCPU  
- Cocok untuk: 10-30 concurrent users
- Deployment: venv + systemd/supervisor

### Jangan pilih jika resource sangat terbatas:
- ❌ Node.js cluster mode (memory hungry)
- ❌ Heavy frameworks (Laravel, Django)

---

## 🚀 Quick Start dengan Go

```go
// Struktur minimal
main.go (HTTP server)
├── handlers/ (upload, download, status)
├── compressor/ (Ghostscript wrapper)
├── cleaner/ (background job)
└── storage/ (file management)

// Dependencies
- Fiber/Echo: HTTP framework
- cron library: untuk cleanup job
- uuid: generate unique filenames
```

**Total size aplikasi:** <15 MB binary + Ghostscript (~30 MB)