# Dokumentasi Aplikasi Pencarian Diagnosis

## Daftar Isi

1. [Istilah & Definisi](#1-istilah--definisi)
2. [Tujuan Aplikasi](#2-tujuan-aplikasi)
3. [Prinsip Utama](#3-prinsip-utama)
4. [Target Pengguna](#4-target-pengguna)
5. [Arsitektur Sistem](#5-arsitektur-sistem)
6. [Alur Pencarian Dokter](#6-alur-pencarian-dokter)
7. [Alur Manajemen Data Master (Admin)](#7-alur-manajemen-data-master-admin)
8. [Alur Monitoring & Logs (Admin)](#8-alur-monitoring--logs-admin)
9. [Struktur Data](#9-struktur-data)
10. [Batasan & Yang Bukan Cakupan Aplikasi](#10-batasan--yang-bukan-cakupan-aplikasi)
11. [Konfigurasi Environment](#11-konfigurasi-environment)

---

## 1. Istilah & Definisi

| Istilah | Penjelasan |
|---|---|
| **ICD-10** | *International Classification of Diseases, 10th Revision* — standar kode diagnosis penyakit terbitan WHO. Contoh: `B36.0` untuk Pityriasis versicolor (panu). |
| **ICD-9** | *International Classification of Diseases, 9th Revision* — dipakai khusus untuk kode prosedur/tindakan medis (bukan diagnosis). Contoh: `21.0` untuk Control of epistaxis. |
| **LOINC** | *Logical Observation Identifiers Names and Codes* — standar kode internasional untuk pemeriksaan laboratorium dan observasi klinis. Contoh: kode untuk "Body temperature". |
| **KFA** | *Kamus Farmasi dan Alat Kesehatan* — kamus kode resmi Kementerian Kesehatan RI untuk produk obat dan alat kesehatan yang beredar di Indonesia. |
| **SNOMED CT** | *Systematized Nomenclature of Medicine Clinical Terms* — standar terminologi klinis internasional yang berfungsi sebagai "hub" penghubung berbagai sistem kode (satu konsep klinis bisa dipetakan ke banyak kode ICD/LOINC/KFA). Di Indonesia, akses resminya lewat lisensi Kemenkes/SATUSEHAT. |
| **Satu Sehat** | Platform interoperabilitas data kesehatan milik Kementerian Kesehatan RI. Aplikasi ini terintegrasi dengan **KFA v2 API** milik Satu Sehat untuk data obat/alat kesehatan secara real-time. |
| **Normalisasi istilah** | Proses mengubah istilah awam/bebas yang diketik dokter (misal "panu") menjadi istilah medis baku dan sinonimnya (misal "tinea versicolor", "pityriasis versicolor") — dilakukan oleh AI, bukan manusia. |
| **Pencocokan langsung (*direct match*)** | Mode pencarian yang langsung mencari ke tabel ICD-10/ICD-9/LOINC/KFA tanpa melalui SNOMED sebagai perantara. Ini mode default aplikasi saat ini. |
| **Hub konsep klinis** | Mode pencarian via SNOMED CT, di mana satu konsep klinis SNOMED punya banyak pemetaan ke ICD-10/ICD-9/LOINC/KFA sekaligus — memberi hasil yang lebih terstruktur secara semantik. Saat ini belum aktif (menunggu lisensi). |
| **Fulltext search** | Teknik pencarian teks di database (MySQL `MATCH ... AGAINST`) yang mencari relevansi kata, bukan sekadar pencocokan string persis. |

---

## 2. Tujuan Aplikasi

Aplikasi ini dibangun untuk menjawab masalah nyata di praktik klinis sehari-hari:

> **Dokter sering tidak hafal kode ICD-10/ICD-9/LOINC/KFA secara persis, dan proses mencari kode yang benar secara manual (buka buku ICD atau cari di sistem lama) memakan waktu dan berisiko salah kode.**

Tujuan utama:

1. **Mempercepat pencarian kode** — dokter tinggal mengetik istilah yang ia pahami (awam maupun medis), sistem langsung menampilkan kode yang relevan dari database resmi.
2. **Menjembatani istilah awam ke istilah medis baku** — misalnya dokter mengetik "mimisan", sistem otomatis memahami ini terkait "epistaxis" dan menampilkan kode ICD-10 `R04.0` beserta kode ICD-9, LOINC, dan KFA yang relevan.
3. **Menjaga validitas kode** — kode yang ditampilkan **selalu** berasal dari baris nyata di database master resmi, tidak pernah dari hasil AI yang bisa saja salah/mengarang (*hallucination*). Ini prinsip paling fundamental dari aplikasi ini.
4. **Siap untuk standar interoperabilitas nasional** — arsitektur dirancang agar bisa "naik kelas" ke SNOMED CT (standar semantik yang lebih kaya) begitu lisensi resmi dari Kemenkes/SATUSEHAT didapat, tanpa perlu menulis ulang aplikasi.

---

## 3. Prinsip Utama

Beberapa prinsip ini dipegang ketat sejak awal pembangunan aplikasi dan mempengaruhi semua keputusan desain:

### 3.1 AI hanya boleh menormalisasi istilah, tidak pernah membuat kode

AI (lewat model bahasa yang dikonfigurasi via `config('ai.*')`) **hanya** digunakan untuk:
- Memperluas istilah pencarian dokter menjadi istilah medis baku dan sinonimnya.
- Menerjemahkan label hasil pencarian ke Bahasa Indonesia (fitur opsional, atas permintaan dokter).

AI **tidak pernah**:
- Menyebutkan atau mengarang kode ICD-10/ICD-9/LOINC/KFA/SNOMED apa pun.
- Menjawab pertanyaan medis atau memberi saran pengobatan.
- Menjadi sumber data yang dikirim ke frontend sebagai "hasil pencarian" — hasil selalu berasal dari query database.

### 3.2 Pemisahan akses: dokter publik, admin wajib login

- Halaman pencarian dokter (`/diagnosis`, `/api/diagnosis/search`, `/api/diagnosis/translate`) **bisa diakses siapa saja tanpa login**. Ini murni alat bantu, tidak menyimpan data pribadi pasien atau identitas dokter.
- Halaman manajemen data master dan monitoring (`/admin/*`) **wajib login**. Dipisahkan secara arsitektural lewat middleware `auth` di level route, dan lewat pemisahan file frontend (`Pages/DiagnosisSearch.jsx` vs `Pages/Admin/*.jsx`) — bukan lewat kondisi login yang ditempel di komponen yang sama.

### 3.3 SNOMED CT bersifat opsional (*pluggable*), bukan ketergantungan keras

- Fitur pencarian **harus tetap berfungsi penuh** meski SNOMED CT belum tersedia (lisensi belum didapat, data belum di-import).
- Saat SNOMED nonaktif (kondisi saat ini): pencarian langsung ke tabel ICD-10/ICD-9/LOINC dan real-time ke KFA API.
- Saat SNOMED aktif (nanti, setelah lisensi didapat): pencarian beralih otomatis memakai SNOMED sebagai hub konsep klinis — cukup ubah nilai konfigurasi `SNOMED_ENABLED=true` di `.env`, tanpa mengubah kode.
- Logika peralihan ini terpusat di satu tempat (`DiagnosisSearchService`), tidak bercabang di controller atau frontend.

### 3.4 KFA diambil real-time dari Satu Sehat, bukan tabel lokal

Berbeda dari ICD-10/ICD-9/LOINC yang datanya di-import ke database lokal, data KFA **diambil langsung secara real-time** dari KFA v2 API milik Satu Sehat setiap kali ada pencarian. Ini memastikan data obat/alat kesehatan selalu up-to-date tanpa perlu proses import manual berulang.

### 3.5 Kegagalan komponen eksternal tidak boleh menggagalkan pencarian dokter

- Kalau AI (normalisasi istilah) gagal/timeout → sistem tetap mencari pakai istilah asli yang diketik dokter.
- Kalau Satu Sehat API (KFA) gagal/timeout → hasil ICD-10/ICD-9/LOINC tetap tampil normal, hanya bagian KFA yang kosong.
- Kalau pencatatan log gagal → tidak pernah mempengaruhi response yang diterima dokter.

---

## 4. Target Pengguna

| Peran | Kebutuhan | Akses |
|---|---|---|
| **Dokter / tenaga medis** | Cari kode diagnosis dengan cepat pakai istilah awam sehari-hari saat praktik | Publik, tanpa login |
| **Admin / tim data** | Kelola data master (import ICD-10/ICD-9/LOINC), pantau performa & biaya AI | Wajib login |

Target institusi: klinik, rumah sakit, atau penyedia sistem informasi kesehatan (SIMRS/EMR) yang butuh alat bantu cepat cari kode diagnosis untuk tenaga medis, sambil tetap punya kontrol data terpusat di sisi admin/IT. Arsitekturnya juga disiapkan untuk institusi yang sedang bertransisi ke standar interoperabilitas SATUSEHAT.

---

## 5. Arsitektur Sistem

### 5.1 Stack Teknologi

- **Backend**: Laravel 12 (PHP 8.4)
- **Frontend**: Inertia.js v3 + React 19 (dengan React Compiler aktif)
- **Styling**: Tailwind CSS v4 (CSS-first, lewat `@theme`)
- **Animasi**: `motion` (pengganti `framer-motion`)
- **Komponen accessible**: `@headlessui/react`
- **Ikon**: `lucide-react`
- **Import Excel**: `maatwebsite/excel`
- **Autentikasi**: Laravel Breeze (varian Inertia + React)
- **Database**: MySQL (fulltext index untuk pencarian teks)

### 5.2 Komponen Utama

```
┌─────────────────────────────────────────────────────────────┐
│                     HALAMAN DOKTER (PUBLIK)                  │
│  Pages/DiagnosisSearch.jsx                                   │
│  - Input pencarian, debounce, quick chips                    │
│  - Kartu hasil dengan badge per sistem kode                  │
│  - Tombol terjemahan (opsional)                               │
└───────────────────┬───────────────────────────────────────────┘
                     │ fetch (JSON API)
                     ▼
┌─────────────────────────────────────────────────────────────┐
│              DiagnosisSearchController                       │
│  - Validasi input, ukur durasi, catat log                    │
└───────────────────┬───────────────────────────────────────────┘
                     │
        ┌────────────┴─────────────┐
        ▼                          ▼
┌──────────────────────┐  ┌─────────────────────────┐
│ QueryNormalizerService│  │ DiagnosisSearchService  │
│ - Panggil AI          │  │ - Cek snomed_enabled    │
│ - Parse istilah medis │  │ - searchDirect() /      │
│ - Fallback jika gagal │  │   searchViaSnomed()     │
└──────────────────────┘  └───────────┬─────────────┘
                                       │
                    ┌──────────────────┼──────────────────┐
                    ▼                  ▼                  ▼
          ┌──────────────────┐ ┌───────────────┐ ┌──────────────────┐
          │ Tabel Lokal MySQL │ │ SnomedConcept │ │ KfaApiSearchService│
          │ icd10/icd9/loinc  │ │ (kalau aktif) │ │ → Satu Sehat API   │
          └──────────────────┘ └───────────────┘ └──────────────────┘
```

```
┌─────────────────────────────────────────────────────────────┐
│                    HALAMAN ADMIN (LOGIN)                     │
│  Pages/Admin/DataMaster.jsx     Pages/Admin/SearchLogs.jsx   │
└───────────────────┬─────────────────────┬─────────────────────┘
                     ▼                     ▼
        ┌──────────────────────┐  ┌──────────────────────┐
        │ DataMasterController │  │  SearchLogController  │
        │ - Import ICD-10/9    │  │  - Tampilkan riwayat  │
        │ - Import LOINC(queue)│  │  - Filter & agregasi  │
        └──────────────────────┘  └──────────────────────┘
```

### 5.3 Peta File Kode Sumber

| Layer | File | Fungsi |
|---|---|---|
| Config | `config/diagnosis.php` | Flag `snomed_enabled` |
| Config | `config/ai.php` | Kredensial & endpoint AI |
| Config | `config/satusehat.php` | Kredensial & endpoint Satu Sehat |
| Model | `app/Models/Icd10Code.php`, `Icd9Code.php`, `LoincCode.php` | Representasi tabel master lokal |
| Model | `app/Models/SnomedConcept.php` | Konsep SNOMED + resolusi mapping ke kode nyata |
| Model | `app/Models/SnomedMapping.php`, `SnomedSynonym.php` | Relasi SNOMED |
| Model | `app/Models/SearchLog.php` | Baris riwayat pencarian |
| Service | `app/Services/QueryNormalizerService.php` | Panggil AI, normalisasi istilah |
| Service | `app/Services/DiagnosisSearchService.php` | Orkestrasi pencarian (direct/SNOMED) |
| Service | `app/Services/KfaApiSearchService.php` | Pencarian KFA real-time ke Satu Sehat |
| Service | `app/Services/SatuSehatAuthService.php` | OAuth2 client_credentials ke Satu Sehat |
| Service | `app/Services/ResultTranslatorService.php` | Terjemahan label hasil (opsional) |
| Service | `app/Services/SearchLogService.php` | Simpan log lewat queue |
| Service | `app/Services/LoincArchiveExtractor.php` | Ekstrak file zip LOINC |
| Controller | `app/Http/Controllers/DiagnosisSearchController.php` | Endpoint pencarian publik |
| Controller | `app/Http/Controllers/ResultTranslationController.php` | Endpoint terjemahan publik |
| Controller | `app/Http/Controllers/Admin/DataMasterController.php` | Endpoint import data (admin) |
| Controller | `app/Http/Controllers/Admin/SearchLogController.php` | Endpoint riwayat log (admin) |
| Job | `app/Jobs/RecordSearchLogJob.php` | Simpan log secara asinkron |
| Import | `app/Imports/Icd10CodesImport.php`, `Icd9CodesImport.php`, `LoincCodesImport.php` | Parsing file Excel/CSV master |
| Command | `app/Console/Commands/Diagnosis/*.php` | Import via CLI, prune log lama |
| Frontend | `resources/js/Pages/DiagnosisSearch.jsx` | Halaman pencarian dokter |
| Frontend | `resources/js/Components/Diagnosis/*.jsx` | Kartu hasil, chip kode, empty/error state |
| Frontend | `resources/js/Pages/Admin/DataMaster.jsx`, `SearchLogs.jsx` | Halaman admin |

---

## 6. Alur Pencarian Dokter

### 6.1 Diagram Alur

```
Dokter buka /diagnosis
        │
        ▼
Ketik istilah (misal "mimisan")
        │
        ▼
Debounce 400ms (mencegah request berlebihan saat mengetik)
        │
        ▼
GET /api/diagnosis/search?q=mimisan
        │
        ▼
┌───────────────────────────────────────────┐
│ 1. QueryNormalizerService::normalize()     │
│    Kirim ke AI: "mimisan"                  │
│    AI kembalikan: ["mimisan", "epistaksis",│
│    "epistaxis", "perdarahan hidung", ...]  │
│    (AI TIDAK menyebutkan kode apa pun)     │
└───────────────────┬───────────────────────┘
                     │ Jika AI gagal/timeout →
                     │ fallback: ["mimisan"] saja
                     ▼
┌───────────────────────────────────────────┐
│ 2. DiagnosisSearchService::search()        │
│    Cek config('diagnosis.snomed_enabled')  │
│                                             │
│    Karena SNOMED belum aktif:              │
│    → searchDirect($terms)                  │
└───────────────────┬───────────────────────┘
                     ▼
┌───────────────────────────────────────────┐
│ 3. Untuk setiap istilah, cari paralel ke:  │
│    - icd10_codes (fulltext + LIKE)         │
│    - icd9_codes  (fulltext + LIKE)         │
│    - loinc_codes (fulltext + LIKE)         │
│    - KFA v2 API Satu Sehat (real-time)     │
│      (coba istilah berurutan, berhenti     │
│      begitu ada hasil relevan, maks 3x)    │
└───────────────────┬───────────────────────┘
                     ▼
┌───────────────────────────────────────────┐
│ 4. Gabungkan hasil round-robin antar sistem│
│    (ICD-10 → ICD-9 → LOINC → KFA → ulang)  │
│    supaya satu sistem tidak mendominasi    │
│    seluruh slot hasil                      │
└───────────────────┬───────────────────────┘
                     ▼
┌───────────────────────────────────────────┐
│ 5. Catat log pencarian (asinkron, lewat    │
│    queue) — tidak memperlambat response    │
└───────────────────┬───────────────────────┘
                     ▼
Response JSON ke frontend:
{
  "query": "mimisan",
  "candidate_terms": [...],
  "snomed_enabled": false,
  "results": [
    { "source": "direct", "preferred_term": "Epistaxis",
      "mappings": { "icd10": [{"code":"R04.0", ...}], ... } },
    ...
  ]
}
        │
        ▼
Frontend render kartu hasil dengan badge per sistem
(ICD-10 biru, ICD-9 ungu, LOINC hijau, KFA kuning)
        │
        ▼
(Opsional) Dokter klik "Terjemahkan ke Bahasa Indonesia"
        │
        ▼
POST /api/diagnosis/translate
→ AI menerjemahkan label yang SUDAH ADA (bukan generate baru)
→ Hasil terjemahan tampil sebagai teks tambahan di kartu
```

### 6.2 Detail Tiap Tahap

**Tahap 1 — Normalisasi istilah (AI)**

`QueryNormalizerService::normalize()` mengirim istilah dokter ke AI dengan system prompt yang eksplisit melarang penyebutan kode apa pun. AI mengembalikan array istilah medis baku dan sinonimnya. Kalau pemanggilan AI gagal (timeout, API down), sistem tetap lanjut dengan istilah asli yang diketik dokter — pencarian tidak pernah berhenti total karena AI bermasalah.

**Tahap 2 — Keputusan sumber pencarian**

`DiagnosisSearchService::search()` adalah satu-satunya titik keputusan apakah memakai SNOMED atau pencarian langsung. Controller dan frontend tidak tahu dan tidak perlu tahu keputusan ini — mereka selalu menerima bentuk hasil yang konsisten.

**Tahap 3 — Pencarian ke masing-masing sistem**

- ICD-10, ICD-9, LOINC: query ke tabel MySQL lokal dengan fulltext search, fallback ke `LIKE` kalau fulltext tidak menemukan hasil.
- KFA: panggil `KfaApiSearchService::search()` yang menghubungi Satu Sehat KFA v2 API secara real-time (dengan cache 10 menit per istilah). Karena parameter `product_type` di API Satu Sehat wajib diisi, sistem memanggil dua kali (`farmasi` dan `alkes`) untuk mencakup obat dan alat kesehatan. Ada filter relevansi tambahan (pencocokan substring) untuk mencegah hasil fuzzy Satu Sehat yang tidak relevan (misal "panu" salah cocok ke "PANADOL").

**Tahap 4 — Penggabungan hasil (round-robin)**

Hasil dari keempat sistem digabung secara bergilir (round-robin), bukan berurutan. Ini mencegah satu sistem yang punya banyak kecocokan (misal ICD-9 dengan banyak variasi kode prosedur) menghabiskan seluruh slot hasil sebelum sistem lain (LOINC/KFA) sempat ditampilkan.

**Tahap 5 — Logging asinkron**

Setiap pencarian dicatat ke `search_logs` lewat `RecordSearchLogJob` yang dieksekusi via queue (`php artisan queue:work`), sehingga proses simpan log tidak menambah waktu tunggu dokter. Data yang dicatat: istilah asli, istilah hasil normalisasi, sumber hasil, jumlah hasil, metrik AI (durasi, token), durasi total request. **Tidak ada identitas dokter/pasien yang disimpan** — log ini murni anonim untuk keperluan monitoring performa dan biaya AI.

---

## 7. Alur Manajemen Data Master (Admin)

### 7.1 Diagram Alur

```
Admin login → /admin/data-master
        │
        ▼
Upload file (ICD-10/ICD-9: .xlsx/.csv, LOINC: .zip/.csv)
        │
        ▼
POST /admin/data-master/import/{system}
        │
        ▼
┌───────────────────────────────────────────┐
│ Jika LOINC & file .zip:                    │
│ → LoincArchiveExtractor ekstrak dulu,      │
│   cari LoincTableCore.csv di dalamnya      │
└───────────────────┬───────────────────────┘
                     ▼
┌───────────────────────────────────────────┐
│ Jalankan Import class yang sesuai:         │
│ - Icd10CodesImport / Icd9CodesImport       │
│   (sinkron, chunk 500 baris)               │
│ - LoincCodesImport                         │
│   (via queue, chunk 1000 baris — karena    │
│   bisa ratusan ribu baris)                 │
│                                             │
│ Baris gagal validasi (kode kosong, dst)    │
│ di-skip dan dicatat, tidak menggagalkan     │
│ seluruh proses import                       │
└───────────────────┬───────────────────────┘
                     ▼
Data upsert ke tabel master (icd10_codes/
icd9_codes/loinc_codes) — baris dengan kode
yang sama akan di-update, bukan duplikat
        │
        ▼
Cleanup file sementara (khusus hasil ekstrak zip)
        │
        ▼
Redirect dengan pesan status (jumlah baris
berhasil/gagal)
```

**Catatan:** KFA tidak punya form import di halaman ini karena datanya diambil real-time dari Satu Sehat API (lihat bagian 6.2).

### 7.2 Cara Alternatif: Import via CLI

Untuk devops/admin teknis, tersedia command Artisan tanpa perlu buka halaman web:

```bash
php artisan diagnosis:import-icd10 {path/ke/file.xlsx}
php artisan diagnosis:import-icd9 {path/ke/file.xlsx}
php artisan diagnosis:import-loinc {path/ke/file.zip-atau-csv}
```

---

## 8. Alur Monitoring & Logs (Admin)

```
Admin login → /admin/logs
        │
        ▼
SearchLogController@index
        │
        ▼
Query search_logs, urut terbaru dulu, paginasi 25/halaman
        │
        ▼
Hitung agregasi: rata-rata latensi AI, rata-rata token,
total pencarian hari ini/minggu ini
        │
        ▼
Frontend tampilkan tabel dengan filter:
- Rentang tanggal (from/to)
- Sumber hasil (direct/snomed/mixed)
- Pencarian teks di istilah asli
```

Tujuan halaman ini: memantau pola pemakaian AI (biaya token, latensi) dan pola pencarian dokter secara agregat — **bukan** untuk melacak dokter individu (karena tidak ada identitas yang disimpan).

Untuk menjaga ukuran tabel log tidak tumbuh tanpa batas, tersedia command pembersihan:

```bash
php artisan diagnosis:prune-logs --days=90
```

---

## 9. Struktur Data

### 9.1 Tabel Master

| Tabel | Kolom Utama | Fulltext Index |
|---|---|---|
| `icd10_codes` | `code`, `description`, `chapter`, `version` | `description` |
| `icd9_codes` | `code`, `description`, `version` | `description` |
| `loinc_codes` | `code`, `long_common_name`, `component`, `system`, `scale_type` | `long_common_name`, `component` |

KFA **tidak punya tabel lokal** — diambil real-time dari Satu Sehat.

### 9.2 Tabel SNOMED (disiapkan, belum aktif)

| Tabel | Fungsi |
|---|---|
| `snomed_ct` | Konsep klinis SNOMED (`concept_id`, `preferred_term`, `semantic_tag`) |
| `snomed_synonyms` | Sinonim/istilah lokal untuk tiap konsep |
| `snomed_mappings` | Pemetaan satu konsep SNOMED ke banyak kode ICD-10/ICD-9/LOINC/KFA, dengan `map_group`/`map_priority` untuk menangani mapping alternatif |

### 9.3 Tabel Log

| Tabel | Fungsi |
|---|---|
| `search_logs` | Riwayat pencarian: istilah asli, istilah ternormalisasi, sumber hasil, metrik AI (latensi/token), durasi total. Anonim (tanpa `user_id`). |

### 9.4 Bentuk Response Hasil Pencarian

Setiap item hasil pencarian punya bentuk konsisten, terlepas dari sumbernya (SNOMED atau pencocokan langsung):

```json
{
  "source": "direct",
  "concept_id": null,
  "preferred_term": "Epistaxis",
  "semantic_tag": null,
  "mappings": {
    "icd10": [{ "code": "R04.0", "label": "Epistaxis", "equivalence": null, "map_advice": null }],
    "icd9": [],
    "loinc": [],
    "kfa": []
  }
}
```

Field `equivalence` dan `map_advice` hanya terisi untuk hasil berbasis SNOMED (menunjukkan apakah pemetaan bersifat setara/lebih luas/lebih sempit/terkait).

---

## 10. Batasan & Yang Bukan Cakupan Aplikasi

Penting untuk dipahami agar aplikasi ini dipakai sesuai konteksnya:

- **Bukan sistem koding rekam medis formal** — tidak ada keterkaitan ke data pasien/kasus, tidak ada implementasi aturan koding resmi (*Excludes1/2*, *Code Also*, urutan diagnosis, grouper INA-CBG), dan tidak ada workflow verifikasi/audit per klaim.
- **Bukan alat diagnosis** — aplikasi ini tidak pernah menyarankan diagnosis atau pengobatan. AI hanya menormalisasi istilah pencarian.
- **Hasil pencarian tetap butuh judgment dokter** — terutama untuk hasil KFA yang bersumber dari pencarian fuzzy pihak ketiga (Satu Sehat), dokter/petugas tetap perlu memverifikasi relevansi hasil sebelum dipakai.
- **Cocok sebagai**: alat bantu referensi cepat untuk validasi kode saat praktik, bukan pengganti sistem informasi rekam medis atau sistem klaim.

---

## 11. Konfigurasi Environment

Variabel environment kunci (`.env`):

```env
# Database
DB_CONNECTION=mysql
DB_DATABASE=laravel_health

# Fitur SNOMED (opsional)
SNOMED_ENABLED=false

# AI untuk normalisasi istilah & terjemahan
AI_PROVIDER=openrouter
OPENROUTER_API_KEY=
OPENROUTER_BASE_URL=
OPENROUTER_MODEL=

# Integrasi KFA via Satu Sehat
SATUSEHAT_CLIENT_KEY=
SATUSEHAT_SECRET_KEY=
SATUSEHAT_ORGANIZATION_ID=
SATUSEHAT_ENV=sandbox

# Queue (wajib aktif untuk logging asinkron & import LOINC)
QUEUE_CONNECTION=database
```

**Catatan penting**: proses `php artisan queue:work` harus selalu berjalan di background (idealnya dikelola lewat Supervisor/Windows Service) agar log pencarian (`RecordSearchLogJob`) dan import LOINC (`LoincCodesImport`, `ShouldQueue`) benar-benar tereksekusi, bukan hanya menumpuk di tabel `jobs`.
