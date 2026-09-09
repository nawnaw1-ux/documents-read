# PANDUAN UJI ALUR BISNIS — SIAKAD YAPI (4 Role)

> Dibuat: 2026-09-09 · Dokumen lokal (tidak di-push), pendamping `TUTORIAL-RUN-LOCALHOST.md`.
> Tujuan: bisa menguji **end-to-end** tiap proses bisnis dengan 4 akun role yang ada:
> **admin pusat**, **admin unit**, **guru**, **wali murid**.
> Semua jalur di bawahini mengasumsikan aplikasi jalan lokal (backend :8000, frontend :3000).

---

## 0. PRASYARAT MENJALANKAN (ringkas)

Rincian lengkap ada di `TUTORIAL-RUN-LOCALHOST.md`. Ringkasannya — 3 terminal:

| Terminal | Perintah | Fungsi |
|---|---|---|
| 1 (root) | `php artisan serve` | Backend API di :8000 |
| 2 (`frontend/`) | `npm run dev` | Frontend di :3000 (buka ini di browser) |
| 3 (root, opsional) | `php artisan queue:work` | Eksekusi job: **email undangan, handoff PMB, notifikasi** — tanpa ini alur PMB/undangan "berhenti diam-diam" |

**Login TIDAK pakai password** (prinsip D2). Cara termudah:

```
php artisan otp:issue admin@yapinet.id
```

→ kode 6 digit tercetak di terminal → masukkan di `localhost:3000/login`. Alternatif: minta
kode dari halaman login; karena gateway email/WA kosong di lokal, kodenya tertulis di
`storage/logs/laravel.log`.

**Reset total kapan pun:** `php artisan migrate:fresh --seed` lalu ulangi persiapan data (§1.3).

---

## 1. DATA UJI BAWAAN SEEDER

### 1.1 Akun yang langsung bisa dipakai

| Email | Role | Catatan |
|---|---|---|
| `admin@yapinet.id` | **admin pusat** | Bisa semua hal, semua unit |
| `adisumardi888@gmail.com` | **admin pusat** | Akun kedua (dipakai pemilik repo) |
| `admin.sd@yapinet.id` | **admin_unit** | ⚠️ terpasang di unit placeholder **SD-SAKINAH** yang TIDAK punya siswa — lihat §1.4 |
| `adiesumardy@gmail.com` (login bisa juga via WA `081292702075`) | **orangtua** | Punya 2 anak lintas unit (kasus D3: satu akun banyak anak) |
| — | **guru** | ⚠️ **tidak ada di seeder** — harus dibuat dulu (§1.3 langkah 1) |

### 1.2 Data siswa/tagihan yang sudah tersedia

| Data | Isi |
|---|---|
| Unit | 4 unit placeholder (`PG/TK/SD/SMP-SAKINAH`) + unit asli dari `TestPaymentSeeder` (`SD-13`, `SMP-12`, dll. — total jadi 11 unit; ini temuan audit "unit phantom", belum dibersihkan) |
| Tahun ajaran | 2026/2027 **aktif**, semester **ganjil aktif** (per 2026-09-09) |
| Siswa | Muhammad Rayhan Pratama (SD-13, kelas 1-A, NIS `202613001`) & Aisyah Putri Azzahra (SMP-12, kelas 7-A, NIS `202612001`) — keduanya anak wali `adiesumardy@gmail.com` |
| Kelas | 1-A (SD-13) dan 7-A (SMP-12) sudah ada |
| Tagihan | SPP bulan berjalan + bulan berikutnya untuk kedua anak (dibuat langsung oleh seeder, bukan lewat billing run) |
| Tarif | SPP dev per unit placeholder (PG 450rb … SMP 750rb); unit asli belum tentu ber-tarif |

### 1.3 Yang TIDAK di-seed → wajib disiapkan sebelum menguji alur akademik

Alur **presensi, nilai/rapor, poin-guru** butuh data yang seeder tidak buat. Siapkan sekali
(lewat admin pusat, ±5 menit):

1. **Akun guru** — `/admin/users` → Tambah User (role `guru`, pilih unit, aktif). Untuk
   menguji guru di kelas ber-siswa: unit **SD-13**. Login guru pakai `otp:issue <email>`.
2. **Mata pelajaran** — `/admin` (menu mapel/nilai) → buat mis. Matematika, B. Indonesia (unit SD-13).
3. **Jadwal pelajaran** — `/admin/jadwal` → pilih kelas 1-A → tambah jadwal: mapel × guru × hari/jam.
   Jadwal inilah "penugasan mengajar" — tanpa jadwal, guru tidak bisa input nilai dan tidak
   bisa buka sesi presensi.
4. (Opsional, untuk poin) **Aturan poin & ambang** — `/admin/poin/aturan` + `/admin/poin/ambang`.

### 1.4 Catatan penting untuk menguji admin_unit

Anak-anak seeder ada di unit **SD-13 / SMP-12**, sedangkan `admin.sd@yapinet.id` terpasang
di unit placeholder SD-SAKINAH (kosong). Dua pilihan:

- **A (disarankan):** buat admin_unit baru di unit SD-13 via `/admin/users` → gunakan itu
  untuk menguji semua alur admin unit berdampingan data seeder; atau
- **B:** login `admin@yapinet.id` → kelola unit SD-SAKINAH untuk menguji alur dari nol
  (buat siswa baru lewat import, dst.).

Yang membedakan admin_unit vs admin pusat (uji juga sisi "tidak bisa"):

| Aksi | admin pusat | admin_unit |
|---|---|---|
| Set tarif/diskon, CRUD user/unit/siswa, import, tahun ajaran, kenaikan kelas | ✅ | ❌ (menu disembunyikan/endpoint menolak) |
| Billing run, tagihan, catat tunai, laporan, kelas, jadwal, poin aturan/ambang unitnya, prestasi verifikasi, pengumuman | ✅ semua unit | ✅ **hanya unitnya** |
| Aturan/ambang "Seluruh sekolah" | ✅ | read-only (UI menandai "Dikelola admin pusat") |
| Buka data unit lain | ✅ | ❌ **404** (bukan 403 — prinsip R3) |

---

## 2. PETA 4 ROLE

| Role | Setelah login mendarat di | Scope | Ringkasan fitur |
|---|---|---|---|
| admin pusat | `/admin` | Semua unit | Master data (unit, user, tarif, diskon, tahun ajaran), siswa + import + ekspor Dapodik, billing, laporan, kelas/jadwal/mapel, poin, prestasi, ekskul, kenaikan kelas, nilai oversight, log aktivitas |
| admin_unit | `/admin` | Satu unit | Kelas/jadwal/mapel unitnya, billing run unitnya, tagihan + catat tunai, laporan unit, poin aturan/ambang unitnya, prestasi, pengumuman |
| guru | `/guru` | Unit tempat mengajar (semua kelas di unitnya) | Daftar kelas + catat poin (satuan/massal/revoke), presensi (buka sesi/QR/roster live), nilai (mapel terjadwal saja) + rekap kelas, prestasi (langsung verified), ekskul yang dibina |
| orangtua | `/dashboard` | Semua anaknya (lintas unit) | Anak, tagihan + keranjang + checkout + riwayat bayar, pilihan seragam, poin, presensi, nilai + rapor PDF, prestasi (ajukan), ekskul (lihat), pengumuman |

---

## 3. ALUR UJI PER PROSES BISNIS

Urutan alur di bawah dianggap seperti siklus nyata: siswa masuk (PMB) → ditagih (SPP) →
kegiatan harian (presensi/poin/prestasi/nilai) → akhir semester (rapor) → akhir tahun (kenaikan kelas).

### A. PMB → siswa & akun wali baru (handoff webhook)

**Konsep (D1):** handoff terjadi saat **uang pangkal lunas** di PMB — bukan saat pengumuman
kelulusan. PMB mendorong webhook `student.enrolled`; aplikasi ini tidak pernah menarik data.

Yang diuji: siswa baru lahir `active`, akun wali dibuat, undangan terkirim, aktivasi tanpa password.

Langkah:

1. **Prasyarat:** set `PMB_HANDOFF_SECRET` di `.env` (kosong = webhook ditolak, fail-closed),
   jalankan `queue:work`. PMB asli tidak ada di lokal → simulasikan kirimnya dari `php artisan tinker`
   (kontrak body lengkap: `docs/02-INTEGRASI-PMB.md`):
   ```php
   $body = [
     'event' => 'student.enrolled',
     'event_id' => (string) Illuminate\Support\Str::ulid(),
     'occurred_at' => now()->toIso8601String(),
     'student' => [ /* identitas lengkap: nama, TTL, NIK, alamat, unit_code, academic_year, tingkat_masuk */ ],
     'guardians' => [ ['hubungan' => 'ayah', 'nama' => '...', 'no_hp' => '...', 'email' => 'tes.wali@example.com', 'is_primary' => true] ],
     'account' => ['email' => 'tes.wali@example.com', 'name' => '...'],
   ];
   $payload = json_encode($body);
   Illuminate\Support\Facades\Http::withHeaders([
       'X-PMB-Signature' => 'sha256='.hash_hmac('sha256', $payload, config('services.pmb.handoff_secret')),
   ])->withBody($payload, 'application/json')
     ->post('http://localhost:8000/api/webhooks/pmb/students');
   ```
2. Respons harus `202 {"status":"queued"}` → job masuk queue → **`queue:work` memprosesnya**.
3. Baca email undangan di `storage/logs/laravel.log` (cari `school_account_invite`) →
   salin tautan `/aktivasi?token=...`.
4. Buka tautan di browser → akun langsung aktif **tanpa memasukkan apa pun** (token = kredensial).
5. Login berikutnya: `otp:issue tes.wali@example.com` (atau minta kode dari halaman login).

**Uji idempotensi (D5/R5):** kirim ulang payload yang sama (event_id sama) → masih `202`,
tidak ada siswa ganda. Cek juga tabel `integration_events`.
**Uji `student.cancelled`:** kirim event cancel → siswa jadi `dropped_out`, akun nonaktif,
tagihan belum-bayar batal.
**Kepemilikan data (R12):** `student.updated` setelah tahun ajaran dimulai → ditolak 409.

### B. SPP: tarif → terbit → bayar → lunas

**Konsep:** tagihan digenerate massal & idempoten (`dedup_key`); semua pembayaran online
lewat VA (produksi: e-SPP Muamalat; lokal: tidak ada bank asli); tunai dicatat admin dan
langsung lunas — **tidak ada** alur unggah/verifikasi bukti transfer.

1. **(admin pusat) Siapkan tarif** — `/admin/tarif`: pastikan fee type `spp` punya rate untuk
   unit & tahun ajaran berjalan (seeder hanya memberi rate ke 4 unit placeholder).
2. **(admin pusat/admin unit) Terbitkan tagihan** — `/admin/generate`:
   - **Preview (dry-run) dulu** → periksa jumlah tagihan & total; cocokkan dengan tarif.
   - **Jalankan** → tagihan terbit per siswa. Jalankan ulang → tidak ada duplikat (dedup_key).
   - Di produksi langkah ini otomatis tiap tanggal 1 (`bills:generate`), plus
     `bills:mark-overdue` (lewat jatuh tempo) & `bills:send-reminders` (H-7/H-1/H+3).
     Lokal bisa dipicu manual: `php artisan bills:generate --type=spp`, `php artisan bills:mark-overdue`, `php artisan bills:send-reminders`.
3. **(wali) Lihat & bayar** — login wali → `/tagihan`: semua tagihan **lintas anak** satu layar.
   - Pilih beberapa tagihan (boleh campur anak & bulan — kasus D3) → **checkout** metode
     Virtual Account (bank Muamalat/BSI) atau QRIS/e-wallet.
   - Halaman `/pembayaran` menampilkan payment + nomor VA. Di produksi wali transfer ke VA,
     polling `payments:poll-billing-va` (tiap 2 menit) yang melunaskan; **di lokal** gunakan
     tombol **simulasi lunas** di halaman pembayaran (endpoint dev-only `simulate-settle`,
     hanya terdaftar saat `APP_ENV=local`).
4. **(admin) Tunai di loket** — `/admin/tagihan` → tagihan siswa → **catat pembayaran tunai**
   → langsung `completed`, `PaymentAllocator` mengalokasikan ke tagihan (boleh sekaligus
   beberapa, dan cicilan untuk fee yang mengizinkan).
5. **Verifikasi:** tagihan jadi LUNAS di portal wali & admin; unduh **invoice/kuitansi PDF**
   (wali: `/tagihan/[ulid]`; admin: dari daftar tagihan); riwayat pembayaran wali di `/pembayaran`.
6. **(admin) Laporan** — `/admin/laporan`: tunggakan (receivables) & penerimaan (collections)
   per unit; harus konsisten dengan langkah 3–4.

**Uji diskon (admin pusat):** `/admin/diskon` → buat skema (mis. sibling/potongan %) →
pasang ke siswa → generate ulang → tagihan terpotong sesuai skema yang dibekukan saat terbit.

### C. Seragam: pilih item/ukuran → ditagih sesuai pilihan

1. **(admin pusat)** `/admin/tarif` → fee type `seragam` (`requires_selection`) → isi katalog
   komponen (kemeja, celana, …) + daftar ukuran + harga per item.
2. **(wali)** `/tagihan/pilihan` → pilih item & ukuran per anak → subtotal dihitung dari
   pilihan (bukan tarif flat). Submit → pilihan menunggu tagihan.
3. **(admin)** billing run jenis seragam → tagihan terbit **sesuai pilihan**, dan pilihan
   **terkunci** (`locked_at`) setelah tagihan terbit — ubah pilihan setelahnya harus ditolak.
4. Bayar seperti alur B.

### D. Presensi per mata pelajaran (QR / NIS)

**Konsep:** presensi melekat pada **sesi jadwal pelajaran**, bukan hari. Siswa check-in
mandiri tanpa punya akun — token sesi adalah kredensialnya.

Prasyarat: §1.3 (guru + mapel + jadwal; uji dengan kelas 1-A SD-13).

1. **(guru)** `/guru` → pilih kelas → panel **Jadwal Hari Ini** muncul hanya jika ada jadwal
   hari itu (day_of_week harus cocoki hari Anda menguji — atau buat jadwal untuk hari ini).
2. Klik **Buka Presensi** → sesi dibuat + **QR/token**. Halaman `/guru/presensi/[ulid]`
   menampilkan QR + roster live.
3. **(siswa)** scan QR / buka `presensi/[token]` (publik) → masukkan **NIS** (mis. `202613001`)
   → nama muncul untuk konfirmasi → check-in. Uji juga: NIS salah, check-in dua kali, token
   kedaluwarsa (sesi berakhir).
4. **(guru)** roster live ter-update; curiga "titip absen"? → **batalkan entri** (revoke,
   bukan delete — alasan tercatat). Tutup sesi dengan **complete**.
5. **Verifikasi rekap:** panel **Rekap Presensi Kelas** (collapsed) di halaman kelas guru —
   H/S/I/A per siswa rentang semester; entri revoked tidak terhitung; siswa tanpa catatan
   tetap tampil (baris nol = temuan).
6. **(wali)** `/anak/[ulid]` → presensi anak.
7. **(admin)** `/admin/laporan` → rekap presensi per kelas/unit.

### E. Poin: catat → ambang → wali diberi tahu

**Konsep (D6):** poin = **ledger**. Revoke = batalkan baris dengan alasan, tidak pernah
menghapus. Saldo = penjumlahan baris.

Prasyarat: §1.3 langkah 4 (aturan + ambang, mis. ambang −10 pelanggaran).

1. **(guru)** halaman kelas `/guru/kelas/[ulid]`:
   - **Catat satuan** per siswa (aturan wajib-bukti → lampirkan file) atau centang beberapa
     siswa → **catat massal**.
   - **Riwayat per siswa** bisa dibuka dari baris siswa; **batalkan** catatan = wajib alasan.
2. **(wali)** `/anak/[ulid]` → meteran poin + riwayat; status "dibatalkan" tampil redup
   dan tidak terhitung.
3. **Ambang:** penuhi saldo melewati ambang → jalankan `php artisan points:evaluate-thresholds`
   (produksi: otomatis 06:30) → email/WA ke wali (lokal: `laravel.log`). **Uji sekali per
   ambang per semester:** jalankan ulang → tidak ada notifikasi kedua.
4. **(admin pusat)** `/admin/log-aktivitas` → semua aksi poin tercatat (siapa-kapan-apa).

### F. Prestasi

Dua pintu masuk dengan tingkat kepercayaan berbeda (R: guru langsung verified; wali menunggu):

1. **(guru)** `/guru/prestasi` → input prestasi (opsional beri poin sekaligus) → langsung
   berstatus verified.
2. **(wali)** `/anak/[ulid]` → ajukan prestasi → status pending, tidak membawa poin sendiri.
3. **(admin)** `/admin/prestasi` → verifikasi / tolak pengajuan wali.
4. Sertifikat/foto diunggah → tampil & bisa diunduh hanya oleh yang berhak (file privat,
   lewat pemeriksaan `visibleTo`).

### G. Nilai & rapor (alur T8 baru saja melengkapi)

**Konsep (R8):** hanya guru yang **ditugaskan lewat jadwal** bisa input nilai; bobot
Tugas 20% / UTS 30% / UAS 50% (asumsi — belum dikonfirmasi sekolah); rapor diagregasi
on-demand, tidak disimpan (R9).

Prasyarat: §1.3 (guru + mapel + jadwal untuk kelas ber-siswa).

1. **(guru)** `/guru/nilai` → kartu per (kelas, mapel) yang diampu → masuk → pilih kategori
   **Tugas/UTS/UAS** → isi nilai 0–100 per siswa → simpan. Re-save kategori sama = koreksi
   (timpa). Uji: nilai di luar 0–100 ditolak.
2. **(guru)** kembali ke halaman kelas → panel **Rekap Nilai Kelas** (baru, T8): pill per
   mapel (semua mapel kelas, bukan cuma miliknya) → T/UTS/UAS/**Akhir** per siswa;
   Akhir "—" berarti masih ada kategori kurang; baris bawah rata-rata kelas.
3. **(wali)** `/anak/[ulid]` → nilai per mapel + **unduh rapor PDF** (render on-demand
   `RaporPdfService`): nilai + ringkasan presensi & poin.
4. **(admin)** `/admin/nilai` → oversight seluruh kelas + unduh rapor arsip (untuk mengecek
   sebelum rilis — celah audit B.2 yang dulu tidak ada).
5. **Uji batas:** guru bukan pengampu mapel itu → 403; kelas unit lain → 404; tanpa semester
   aktif → 422.

### H. Ekstrakurikuler

1. **(admin)** `/admin/ekstrakurikuler` → buat ekskul (pilih **pembina** = guru) → kelola
   anggota (roster). Tagihan ekskul hanya untuk anggota roster.
2. **(guru pembina)** `/guru/ekskul` → hanya ekskul yang dibinanya; tambah/keluarkan anggota.
3. **(wali)** `/anak/[ulid]` → roster ekskul anak (saat ini **lihat saja** — pendaftaran
   mandiri wali masih menunggu keputusan mentor, pertanyaan §4 no. 1 di PROGRESS-MAGANG.md).
4. Uji antar-role: guru non-pembina tidak melihat ekskul itu.

### I. Kenaikan kelas massal (akhir tahun ajaran)

**Konsep (R10):** kenaikan = **menambah baris enrollment** tahun ajaran baru, bukan
menimpa `classroom_id` — riwayat kelas per tahun tetap utuh.

1. **(admin pusat)** buat/aktifkan tahun ajaran baru (`/admin/unit` → tahun ajaran) →
   buat kelas tujuan di tahun baru.
2. `/admin/kenaikan-kelas` → pilih kelas sumber → lihat roster & kandidat tujuan →
   eksekusi promosi massal (pilih naik/tinggal — per siswa).
3. Verifikasi: siswa punya enrollment baru di tahun baru, enrollment lama tidak berubah;
   tagihan baru (mis. SPP bulan pertama tahun baru) mengikuti kelas baru.

### J. Pengumuman & aktivitas

1. **(admin/admin_unit)** `/admin/informasi` → buat pengumuman scope sekolah/unit/kelas,
   lampiran file.
2. **(wali)** `/informasi` → hanya pengumuman yang sesuai scope-nya (anaknya di kelas itu).
3. **(admin pusat)** `/admin/log-aktivitas` (T6): telusuri aksi uang & poin yang Anda
   lakukan di alur B–E — filter aksi/pelaku/tanggal.

---

## 4. CHEAT SHEET ARTISAN UNTUK TESTING

```powershell
# login siapa pun
php artisan otp:issue admin@yapinet.id
php artisan otp:issue <email-guru@yapinet.id>

# picu scheduler secara manual (produksi otomatis)
php artisan bills:generate --type=spp      # terbitkan SPP bulan berjalan (idempoten)
php artisan bills:mark-overdue             # tandai lewat jatuh tempo
php artisan bills:send-reminders           # pengingat H-7/H-1/H+3
php artisan points:evaluate-thresholds     # notifikasi ambang poin (sekali per ambang)
php artisan payments:poll-billing-va       # cek pelunasan VA (produksi tiap 2 menit)

# diagnostik & master data
php artisan diagnose:va-collisions         # cek tabrakan nomor VA (produksi)
php artisan units:sync                     # sinkron unit dari PMB (butuh jaringan)

# kerja job (email undangan, handoff, notifikasi) — biarkan terbuka saat menguji
php artisan queue:work

# reset total
php artisan migrate:fresh --seed
```

Email/WA "terkirim" di lokal semuanya berakhir di `storage/logs/laravel.log`.

## 5. BEDA LOKAL vs PRODUKSI (supaya tidak salah simpul saat menguji)

| Hal | Lokal | Produksi (siakad.yapinet.id) |
|---|---|---|
| Email/WA | Log saja (`laravel.log`) | SendagoMail/SendagoWhatsApp (tanpa retry — kebijakan menunggu mentor) |
| Pembayaran online | Tombol **simulasi lunas** (dev-only) | VA Muamalat/BSI via e-SPP + polling 2 menit; **VA BSI belum terverifikasi** (domen mentor) |
| Scheduler | Manual via artisan | Otomatis (`schedule:work`) |
| Database | SQLite | PostgreSQL |
| Siswa baru | Import admin / webhook simulasi | Handoff PMB otomatis saat uang pangkal lunas |
