# LAPORAN AUDIT MENYELURUH — SIAKAD YAPI

> Tanggal audit: 7 September 2026
> Jenis audit: Menyeluruh (fitur, integrasi, arsitektur, keamanan, stabilitas)
> Catatan: Tahap ini **HANYA audit & pelaporan**. Tidak ada kode yang diubah.

---

## A. RINGKASAN EKSEKUTIF

Project SIAKAD YAPI sudah berjalan di produksi (`siakad.yapinet.id`) dan secara keseluruhan
**matang & terencana rapi**. Arsitekturnya kuat: Laravel 13 REST API headless + Next.js 16
SPA, auth tanpa password (OTP), enkripsi PII + blind-index, ledger tunggal untuk uang &
poin, idempotensi webhook via `integration_events`, dan scope unit via `visibleTo()`.
Fase 1-3 (auth/handoff, keuangan, kesiswaan) nyaris lengkap; Fase 4 (akademik) sedang berjalan.

Temuan paling kritis (urut keparahan):

1. **BUG FUNGSIONAL (Kritis): Halaman riwayat pembayaran wali 404.**
   Frontend memanggil `/api/wali/bills/payments` (`frontend/src/app/pembayaran/page.tsx:69`)
   tetapi backend hanya mendefinisikan `/api/wali/payments`
   (`routes/api.php:98`). Halaman riwayat transaksi wali tidak akan pernah memuat data.

2. **Config BSI/Bank ID tidak terverifikasi (Rawan).** `services.billing_api.banks.bsi.bank_id`
   default `'1'` — dikomentari di kode sendiri bahwa itu "almost certainly NOT BSI's real
   bank_id", jadi VA BSI "unlikely genuinely payable" sampai e-SPP memberi nilai asli.
   Ini berarti kanal pembayaran BSI yang ditawarkan ke wali berpotensi gagal di produksi.

3. **`UserFactory.php` rusak (Sedang).** Factory masih menulis kolom `password` yang sudah
   di-drop oleh migration. Test apa pun yang memanggil `User::factory()->create()` akan error
   SQL di PostgreSQL.

4. **`achievements.source` kehilangan nilai `'guru'` (Rendah/Sedang).** Enum hanya
   `pmb, sekolah`. Guru sudah bisa input prestasi (langsung verified) via `source='sekolah'`
   + `achiever_type`, jadi bukan blocker — tapi menyalahi spesifikasi dan mencampur semantik
   "sekolah" untuk prestasi guru.

5. **Seeder duplikat unit phantom (Rendah).** `DatabaseSeeder` membuat 4 placeholder
   (`TK-SAKINAH`, `SD-SAKINAH`, `SMP-SAKINAH`…) yang "nyangkut" berdampingan dengan 8 unit
   asli dari `TestPaymentSeeder` — database fresh jadi punya 11 unit, bukan 8.

Keamanan & otorisasi **sebagian besar sehat**: semua endpoint role-gated, row-security
lewat `visibleTo()`, no IDOR yang jelas, webhook HMAC/TOKEN, dev-only `simulate-settle`
di-guard ganda. Kekurangan utamanya di area non-fungsional & integrasi produksi.

---

## B. PETA FITUR & ROLE

### B.1 Inventarisasi Role

| Role | Home path | Scope | Fitur utama yang tersedia |
|---|---|---|---|
| `admin` (pusat) | `/admin` | semua unit | user CRUD, siswa CRUD/import, tarif CRUD, diskon, unit, tahun ajaran, kelas, jadwal, titik, prestasi, informasi, kenaikan kelas, nilai, laporan, dashboard |
| `admin_unit` | `/admin` | satu unit | tidak ubah tarif/user/siswa/unit; kelola kelas/jadwal/poin/prestasi/informasi unitnya, laporan unitnya |
| `guru` | `/guru` | unit tempat mengajar | kelas, poin (single/bulk/revoke), presensi (buka sesi/QR/roster), nilai (hanya mapel terjadwal), prestasi (verified), ekskul yang dibina |
| `orangtua` (wali) | `/dashboard` | semua anaknya (lintas unit) | anak, tagihan + keranjang + checkout, pembayaran, poin, prestasi (ajukan), presensi, nilai, rapor, ekstrakurikuler, pengumuman |

### B.2 Fitur yang Kurang / Perlu Ditambah

| Role | Fitur yang kurang | Alasan/urgensi |
|---|---|---|
| **Semua** | Rapor **belum di-output/diperiksa** di sisi hasil — `RaporPdfService` ada tapi belum ada pengujian e2e/verifikasi isi | Rapor = output utama siklus akademik; belum tentu siap produksi |
| **Semua** | Halaman **`anak/[ulid]`** (wali) mengakses rapor, tapi admin/guru tidak punya UI preview rapor (hanya arsip PDF) | Wali melihat rapor; guru/admin tidak bisa cek dulu sebelum dirilis |
| **guru** | Tidak ada UI **edit nilai setelah input** (re-save menimpa, tapi berdasar kategori — tidak ada halaman rekap nilai per kelas utuh) | Guru perlu melihat ringkasan nilai kelas, bukan cuma input per kategori |
| **guru/admin_unit** | Tidak ada **rekap presensi per kelas** bagi guru (hanya laporan admin & rekap per siswa wali) | Guru perlu memonitor kehadiran kelasnya |
| **guru** | **Ekstrakurikuler**: tidak ada alur mendaftarkan siswa *selain* lewat dashboard admin/guru roster | Siswa/wali tidak punya form daftar ekskul (sudah ada roster ekskul di wali, tapi bukan pendaftaran) |
| **orangtua** | Tidak ada **alur reset/pulihkan akses** selain OTP (reset via undangan `purpose=reset` ada di skema tapi tidak terlihat di UI/frontend) | Wali yang lupa email/HP kehilangan satu-satunya jalan masuk |
| **orangtua** | Tidak ada **notifikasi in-app / push** — hanya email/WA via log | Komunikasi real-time belum ada; semua bergantung eksternal |
| **orangtua** | **Pendaftaran ekskul & dokumen siswa** tidak bisa diinput wali (hanya admin) | Sebagian besar perlu hubungan wali, tapi sengaja dibatasi? (lihat pertanyaan) |
| **admin** | Tidak ada **dash ini** untuk monitoring webhook/`integration_events` yang gagal | Kegagalan webhook tercatat di DB tapi tidak tampil di UI |
| **admin** | Tidak ada **audit log viewer** (`activity_logs` ditulis tapi tidak ada halaman lihat) | Log ada, tidak terekspos; auditabilitas kurang |
| **admin** | Tidak ada **manajemen `fee_components`/item seragam** per sesi — hanya via tarif | Katalog seragam itemizable tapi UI-nya terbatas (perlu konfirmasi) |
| **admin** | Tidak ada **fitur WAIVED/report** khusus untuk laporan ragam fee_type, hanya receivables/collections umum | Laporan masih umum |

> Catatan: beberapa "kekurangan" di atas bisa jadi sengaja didesain (mis. pendaftaran ekskul
> dikontrol admin). Semua dituangkan juga di bagian **G. Pertanyaan untuk Konfirmasi**.

---

## C. MASALAH / BUG YANG DITEMUKAN

| Area/Modul | Masalah | Lokasi file | Keparahan |
|---|---|---|---|
| Wali — Pembayaran | Halaman riwayat pembayaran memakai path salah → **404**, riwayat tidak pernah tampil | `frontend/src/app/pembayaran/page.tsx:69` vs `routes/api.php:98` | **Kritis** |
| Payment config | VA/`bank_id` BSI tidak terverifikasi (default `'1'` = salah bank) — risiko VA wali tidak bisa dibayar | `config/services.php:122`, `BillingApiGateway.php:51-53` | **Kritis (integrasi produksi)** |
| Testing | `UserFactory::create()` menulis kolom `password` yang sudah di-drop → gagal di Postgres | `database/factories/UserFactory.php:31` | Sedang |
| Database | `achievements.source` enum kurang nilai `'guru'` (spesifikasi vs implementasi; dicover oleh `achiever_type`) | `2026_08_18_000026`, `Achievement.php:66` | Rendah/Sedang |
| Database | `integration_events.source` enum kurang `billing_api` (divakumkan jadi string di SQLite; Postgres belum) | `2026_08_14_000010`, `2026_08_20_000030` | Sedang (produksi) |
| Database | `staff_profiles` kehilangan kolom `phone` (encrypted) yang disyaratkan spesifikasi | `2026_08_14_000005` | Rendah |
| Database | `payment_allocations` tidak punya `ulid` (melanggar konvensi "semua tabel ber-ulid") | migration payments | Rendah |
| Database | `extracurricular_members` tidak ada unique constraint (bisa dobel anggota 1 siswa-1 ekskul-1 tahun) | migration ekskul | Rendah |
| Database | `class_schedules` tidak ada constraint anti-overlap waktu guru/kelas | migration jadwal | Rendah |
| Database | `student_guardians` tidak ada index langsung pada `guardian_id` (query "semua anak untuk guardian" agak lambat) | `2026_08_14_*` | Rendah |
| Database | Seeder unit phantom: `TK-SAKINAH`, `SD-SAKINAH`, `SMP-SAKINAH` nyangkut vs 8 unit asli | `database/seeders/DatabaseSeeder.php` | Rendah |
| Auth/Keamanan | `score` nilai tidak divalidasi rentang (bisa disembuhkan ≤0/≥100) — catatan: `score decimal(5,2)` | `GradeService` | Rendah |
| Notification | `SendagoMailGateway` & `SendagoWhatsAppGateway` saat send gagal hanya `Log::warning`, **tidak retry & tidak menandai `notification_logs` berstatus `failed` secara konsisten** | `SendagoMailGateway.php:59-75`, `SendagoWhatsAppGateway.php:40-56` | Sedang |
| Scheduler/Queue | Tidak ada job retry khusus untuk polling VA selain cron; `jobs`/`failed_jobs` tidak dimonitor via UI | `routes/console.php` | Rendah |
| Frontend (poin) | `admin/poin/aturan` hanya implement toggle+delete; PUT edit belum di-wire (backend punya `PointRuleController::update`) | `frontend/src/app/admin/poin/aturan` | Rendah |
| Frontend (ambang) | `admin/poin/ambang` hanya create; edit/delete loop belum utuh | `frontend/src/app/admin/poin/ambang` | Rendah |
| App config | `config('app.timezone')` di-hardcode `'UTC'` walau `.env` `APP_TIMEZONE=Asia/Jakarta`; dikompensasi per-call `Carbon::today('Asia/Jakarta')` → risiko inkonsistensi tanggal | `config/app.php` | Rendah/Sedang |
| Tentang | `sendago.whatsapp` & `sendagomail` credential tidak wajib — produksi bisa "senyap" hanya-log jika env kosong | config/services.php | Sedang (audit env produksi) |
| Dev-only | Default kredensial e-SPP `admin/admin123` sebagai fallback di config (hanya fallback; produksi harus override) | `config/services.php:84` | Rendah |

---

## D. STATUS INTEGRASI

### D.1 Integrasi Eksternal

| Nama integrasi | Status | Catatan |
|---|---|---|
| **PMB Handoff** (webhook push) | **Sehat** | HMAC-SHA256, `integration_events` idempoten, job `ProcessPmbHandoffEvent` retry 5× backoff. Kuat. |
| **e-SPP Virtual Account — Muamalat** | **Sehat/Rawan** | `bank_id` `'1'` belum diverifikasi (dikomentari). VA deterministik + dedup + `diagnose:va-collisions`. |
| **e-SPP Virtual Account — BSI** | **Rawan** | `bank_id` default `'1'` "almost certainly NOT BSI's real id" → VA BSI kemungkinan tidak bisa dibayar. **Perlu nilai asli dari e-SPP.** |
| **SendagoPay** (QRIS/e-wallet) | **Sehat (dev) / Rawan (prod)** | Unset key → hanya log. Tidak ada retry. |
| **Xendit** (fallback) | **Sehat (fallback, tak dipakai rilis)** | Unset secret → checkout tanpa invoice (sengaja). |
| **SendagoMail** (email) | **Rawan** | Unset credential → hanya log. Gagal kirim → `Log::warning` tanpa retry. |
| **SendagoWhatsApp** (WA/OTP/undangan/reminder) | **Rawan** | Unset credential → hanya log. Gagal kirim → `Log::warning` tanpa retry. **Send OTP WA = jalan satu-satunya bagi yang tak punya email.** |
| **File/Cloud storage** | **Sehat** | File lokal `storage/` (filesystems default), private-served via auth. Belum cloud (S3). |

Poin penting integrasi:
- **Error handling & retry**: webhook PMB punya retry solid. Gateway pembayaran e-SPP ada
  `BillingApiException` + logging. Namun **gateway notifikasi (email/WA) TIDAK punya retry** dan
  tidak konsisten menulis `notification_logs.status='failed'`.
- **Log kegagalan**: semua tercatat ke `Log::*` dan sebagian ke `notification_logs`.
- **Deprecated/versi lama**: `barryvdh/laravel-dompdf ^3.1` OK; tidak ada library deprecated
  mencolok. Laravel 13 + Sanctum 4 + Next 16 sudah rilis terbaru.

### D.2 Integrasi Antar-Modul Internal

| Hubungan modul | Status | Catatan |
|---|---|---|
| Siswa (akademik) → Tagihan/Billing | **Sehat** | `bills.student_id` + scope `visibleTo` konsisten; `PaymentAllocator` satu-satunya penulis saldo. |
| Siswa → Poin | **Sehat** | `point_records.student_id`; ledger append-only. |
| Siswa → Presensi & Nilai/Rapor | **Sehat** | student_id/classroom_id/term_id dinormalisasi konsisten; `GradeService` validasi guru terjadwal. |
| Siswa → Ekskul | **Sehat/Rendah** | Tidak ada unique → risiko dobel baris anggota; `requires_roster_membership` (M35) sudah menautkan ke tagihan ekskul. |
| Handoff (PMB) → akun/undangan wali | **Sehat** | `AccountInvitationSender` + aktivasi via token; idempoten. |
| Term/Yahun → semua ledger (poin/presensi/nilai) | **Rendah** | `term_id` restrictOnDelete konsisten utk poin/presensi/nilai, tapi `point_threshold_notifications.term_id` cascadeOnDelete → **tidak konsisten** (lihat C). |
| Fee selection → tagihan | **Sehat** | `locked_at` membekukan pilihan begitu tagihan terbit; diskon dibekukan. |

Temuan integrasi internal:
- **Konsistensi data antar-modul baik** — tidak ditemukan orphan yang nyata berkat FK
  `cascade/restrict/nullOnDelete` yang terencana (85 FK).
- **Ketidakkonsistenan kecil**: aturan hapus `term` berbeda antara `point_records`
  (restrict) dan `point_threshold_notifications` (cascade).

---

## E. KUALITAS KODE & KONSISTENSI ARSITEKTUR

**Kesimpulan: Arsitektur konsisten & rapi — salah satu yang terbaik untuk project ukuran ini.**

- **Pola penamaan**: konsisten. Controllers per-role (Admin/Guru/Wali), Services per-domain
  (Billing/Payment/Points/Attendance/Academic), Models pakai trait `HasUlidKey` +
  `HasEncryptedAttributes`.
- **Scope unit**: semua lewat `scopeVisibleTo()` / `scopeManageableBy()` di model — bukan `if`
  di controller. Konsisten di ~15 model. Ini pola yang sangat baik.
- **Validation**: terjadi inline di controller (tidak ada `app/Http/Requests`). Tidak salah,
  tapi kurang seragam/tertangkap vs `FormRequest` — **rekomendasi untuk konsistensi baru**.
- **TODO/FIXME/HACK**: **nol** ditemukan di `app/` dan `frontend/src/`. Bersih.
- **Dead code / setengah jadi**:
  - Backend punya `PointRuleController@update` & `PointThresholdController@update` tetapi
    frontend tidak men-wire full edit UI (hanya toggle/delete). => *endpoint ada, UI kurang*.
  - `simulate-settle` dev-only (sengaja, ter-guard).
  - `achievements.source='guru'` tidak dipakai (pakai `achiever_type`).
- **Trait/concern**: rapi; `HasEncryptedAttributes` dipakai konsisten.
- **Komentar**: informatif (menjelaskan *kenapa*), sesuai aturan project.

---

## F. KEAMANAN & OTORISASI

| Aspek | Status | Catatan |
|---|---|---|
| Proteksi role per endpoint | **Sehat** | Middleware `role:...` di semua grup; `is_active` juga dicek. |
| Row-level security (IDOR) | **Sehat** | `visibleTo()` / `manageableBy()` di model; unassigned unit → `1=0` (fail-to-nothing). Rekomendasi `404` untuk baris di luar scope sudah diterapkan. |
| Endpoint tanpa auth | **Sehat** | Hanya publik yang sengaja (presensi token-gated, files di-auth, webhook signature). |
| Validasi input | **Sebagian** | Validasi inline bagus; tapi rentang `score` nilai tidak di-pas, dan ada beberapa field yang hanya length-check. |
| Exposure data sensitif di response | **Sehat** | NIK/NISN/phone dienkripsi + blind-index; API memakai ULID, bukan numerik. |
| Webhook signature | **Sehat** | HMAC (PMB), token (Xendit/SendagoPay), fail-closed saat secret kosong; `hash_equals` timing-safe. |
| Anti enumerasi | **Sehat** | Respons OTP sama utk akun terdaftar/tidak; 404-utk-out-of-scope. |
| Dev backdoor | **Sehat** | `simulate-settle` di-guard env ganda. |
| Rate limiting | **Sehat** | OTP throttle ganda (identifier + IP), presensi, dll. |

Tidak ditemukan celah IDOR yang jelas. Arsitektur keamanan menonjol.

---

## G. PERTANYAAN UNTUK KONFIRMASI (bukan asumsi bug)

1. **Pendaftaran ekstrakurikuler oleh wali** — sengaja dibatasi ke admin/guru saja? Atau wali
   harus bisa mendaftar? (saat ini wali hanya bisa *melihat* roster, tidak mendaftar).
2. **Dokumen siswa** (`student_documents`) — sengaja tidak ada UI/dikoordinasi oleh admin
   (upload manual), atau memang belum dikerjakan? (fitur ada di skema, tidak ada endpoint).
3. **`source='guru'` di achievements** — menerima solusi saat ini (`achiever_type`) sebagai
   pengganti, atau mau tetap memakai enum `guru` agar selaras spesifikasi?
4. **`integration_events.source='billing_api'`** — apakah e-SPP callback memang masuk satu
   `source` spesifik ini, atau justru sengaja memakai path `/api/payment-webhook/{uuid}` yang
   terpisah? (kalau terpisah, enum gap tidak berdampak).
5. **Retry notifikasi (email/WA)** — apakah kebijakan saat ini "gagal = log saja, tidak retry"
   sudah disepakati, atau perlu ditambah retry/alerting?
6. **Standarisasi FormRequest** — apakah mau pindah ke `FormRequest` untuk konsistensi
   validasi, atau biarkan inline?
7. **`bills.issuing` via scheduler** tanpa monitor UI untuk `failed_jobs`/`billing_runs.error` —
   apakah sudah dipantau di luar aplikasi (ops)?

---

## H. REKOMENDASI PRIORITAS PERBAIKAN

Dikerjakan dalam sesi terpisah setelah laporan direview. Urut dari paling mendesak:

### Prioritas 1 — Kritis (produksi terpengaruh)
1. **Perbaiki path riwayat pembayaran wali** → ubah `/api/wali/bills/payments` →
   `/api/wali/payments` di `pembayaran/page.tsx:69`. *(1 baris)*
2. **Verifikasi & set `bank_id` BSI (dan Muamalat) yang asli** ke e-SPP, lalu masukkan ke env
   produksi. Tanpa ini kanal bayar BSI berisiko gagal di lapangan.

### Prioritas 2 — Sedang (kestabilan & integritas)
3. **Bersihkan `UserFactory.php`** — buang kolom `password` agar test tidak pecah.
4. **Seragamkan kebijakan retry + penandaan `failed`** pada gateway email/WA; aktifkan
   alerting saat `notification_logs.status='failed'` (terutama OTP & reminder SPP).
5. **Konfirmasi & (jika perlu) tambah `billing_api`** ke enum `integration_events.source` agar
   aman di PostgreSQL produksi.
6. **Konsistenkan `term_id` deletion** antara `point_records` (restrict) dan
   `point_threshold_notifications` (cascade).

### Prioritas 3 — Rendah (kualitas & konsistensi)
7. **Bersihkan seeder** unit phantom (hapus placeholder agar hanya 8 unit asli).
8. Tambahkan `ulid` di `payment_allocations` untuk konsistensi konvensi.
9. Tambahkan unique constraint `extracurricular_members` dan index `student_guardians.guardian_id`.
10. Validasi rentang `score` nilai (0–100).
11. Pertimbangkan `FormRequest` untuk konsistensi validasi.
12. Selesaikan CRUD edit `point-rules` / `point-thresholds` di frontend (backend sudah siap).

### Prioritas 4 — Roadmap fitur (bahas dulu, jangan dikerjakan tanpa konfirmasi)
13. Verifikasi konten rapor e2e sebelum dianggap produksi.
14. UI reset akses non-OTP & revisi strategi notifikasi (in-app vs eksternal).
15. Monitoring/UI untuk webhook & failed queue.
