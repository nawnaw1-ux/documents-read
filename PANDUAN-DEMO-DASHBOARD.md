# PANDUAN DEMO — Dashboard Admin Unit (SDI Al Azhar 13)

> Dibuat: 2026-09-10 · Pendamping demo akun `admin.sd@yapinet.id` (admin unit SD-13).
> Data demo: `DashboardDemoSeeder` (siswa/nilai/presensi/poin/tagihan untuk SD-13 + SMP-12).
> Sumber kebenaran semua angka: `DashboardSummaryController` + `frontend/src/app/admin/page.tsx`.

---

## 1. KAMUS ANGKA — apa maksud tiap kartu (jawaban kalau ditanya "ini angka apa?")

### KPI Cards (baris pertama)

| Kartu | Sumber data | Cara baca / bicara saat demo |
|---|---|---|
| **Siswa Aktif** | `students.status='active'` milik unit | "Anak yang sedang menempuh pendidikan di unit ini." Sub: *baru tahun ini* = masuk TA berjalan; *total* = semua status termasuk lulus/keluar. |
| **Rombel Aktif** | kelas `is_active` TA berjalan | "Jumlah kelas yang berjalan tahun ini." Sub: guru aktif ber-unit + siswa yang punya penempatan kelas (enrollment aktif). |
| **Penerimaan Kas** | Σ `paid_amount` semua tagihan unit (kecuali dibatalkan) | "Uang yang SUDAH masuk ke sekolah dari semua tagihan." ⚠️ **Tidak dibatasi periode** — semua tagihan sejak data ada, bukan kas semester ini. Kalau ditanya "ini kas bulan ini?" jawab: total akumulasi; rincian per periode ada di Laporan Keuangan. |
| **Sisa Piutang** | Σ `remaining_amount` tagihan terbuka (unpaid/partial) | "Tagihan yang masih kami tunggu pembayarannya dari wali." Piutang = receivable, uang yang menjadi hak sekolah tapi belum diterima. |
| **Tagihan Jatuh Tempo** | bill `due_date` < hari ini & masih terbuka | "Tagihan yang sudah LEWAT batas bayar dan perlu ditindaklanjuti (ingatkan wali)." Sub menampilkan nilai rupiah yang tertunggak. |
| **Kehadiran Siswa** | catatan presensi **per sesi mapel**, semester berjalan | "Dari SEMUA sesi pelajaran yang sudah direkap semester ini, berapa persen dihadiri." H/S/I/A = hadir/sakit/izin/alpa per sesi — 1 anak hadir 6 jam pelajaran = 6 catatan hadir. Bukan "% anak masuk hari ini" (itu kartu terpisah). |
| **Prestasi Terverifikasi** | `achievements.status='verified'` | Prestasi yang sudah divalidasi staf; *menunggu* = diajukan wali/guru tapi belum diverifikasi. |
| **Ekstrakurikuler** | ekskul aktif TA berjalan + anggota aktif | Jumlah kegiatan dan total anak yang mengikuti. |

### Ringkasan Akademik (kartu khusus admin unit)

| Elemen | Rumus | Catatan penting |
|---|---|---|
| Siswa / Guru / Kelas | sama dengan KPI, versi unit | — |
| **Siswa Perlu Perhatian** | GABUNGAN (union, tanpa dobel) dari 4 indikasi: alpa ≥ 5, nilai akhir < KKM 70, rata-rata turun ≥ 5 poin vs semester lalu, punya poin pelanggaran | Ini "watchlist" — anak yang masuk SALAH SATU saja sudah dihitung |
| **Absensi Tinggi** | `absent_count ≥ 5` pada enrollment TA berjalan | 5 = asumsi sistem, belum kebijakan resmi sekolah |
| **Penurunan Nilai** | rata-rata nilai akhir turun ≥ 5 poin antar semester | Nilai akhir = Tugas 20% + UTS 30% + UAS 50%, hanya dihitung untuk mapel yang KETIGA nilainya lengkap; anak tanpa semester pembanding tidak dihitung |
| Kehadiran Hari Ini | sesi presensi tanggal hari ini | Persentase hadir dari sesi yang sudah direkap hari ini |
| Nilai Akhir Semester | rata-rata nilai akhir per siswa → dirata-ratakan lagi | KKM 70 = **asumsi dashboard, bukan kebijakan tersimpan** ( komentar kode: "a dashboard threshold, not a stored policy") |
| Capaian Prestasi | verified, terpisah siswa vs guru | — |
| Poin Tata Tertib | catatan merit (+) vs pelanggaran (−), semester berjalan | "N siswa dengan pelanggaran" = anak unik yang pernah dapat poin negatif |

### Panel "Perlu Perhatian" (7 alert)

| Alert | Link saat ini | Status link |
|---|---|---|
| Tagihan lewat jatuh tempo | `/admin/tagihan` | ✅ tepat (ada filter status) |
| Siswa dengan sisa piutang | `/admin/tagihan` | ✅ masuk akal |
| Siswa absensi tinggi | `/admin/laporan` | ❌ **halaman Laporan Keuangan — tidak ada data presensi di sana** |
| Nilai akhir di bawah KKM | `/admin/nilai` | ⚠️ topik benar, tapi tidak ada filter "di bawah KKM" |
| Prestasi menunggu verifikasi | `/admin/prestasi` | ✅ tepat |
| Siswa dengan pelanggaran poin | `/admin/poin` | ✅ tepat |
| Siswa aktif belum ditempatkan di kelas | `/admin/kelas` | ⚠️ masuk akal, tapi tidak ada daftar "belum punya rombel" |

### Grafik Arus Tagihan & Piutang
Bar hijau = kas masuk, bar merah = piutang, proporsional terhadap unit terbesar. Badge % terbayar: ≥80 hijau, ≥50 kuning, <50 merah. ⚠️ Judulnya menyebut "pada Semester X" tapi angkanya **tidak difilter per semester** (semua tagihan).

---

## 2. GAP ROUTING YANG BELUM CLEAR (temuan verifikasi kode)

1. **Watchlist tidak bisa di-drill-down.** Backend hanya mengirim ANGKA (count), tidak pernah nama/ULID siswa yang ter-flag. Tidak ada satu pun halaman yang bisa menjawab "SIAPA sih siswa perlu perhatian itu?" — termasuk halaman tujuan link-nya.
2. **"Absensi Tinggi" me-link ke `/admin/laporan` = Laporan Keuangan & Piutang.** Salah target topik total; tidak ada presensi di halaman itu.
3. **"Penurunan Nilai" → `/admin/nilai`** tanpa filter — admin harus sudah tahu nama anak yang mau dicek.
4. **`/admin/siswa` hanya dukung filter `?unit=`** — tile "Siswa Perlu Perhatian" mendarat di daftar siswa polos.
5. **Angka keuangan all-time, label bermusim.** `bills` di-query tanpa filter tahun ajaran — beda domain dengan presensi/poin (semester) dan siswa/kelas (TA berjalan).
6. **KKM 70 / alpa ≥ 5 / turun ≥ 5 poin = konstanta di kode dashboard.** Belum pernah dikonfirmasi sekolah, belum bisa dibedakan per jenjang.
7. (Risiko teknis demo) **"Kehadiran Hari Ini" pakai tanggal server (timezone app masih UTC)** — sesi pagi sebelum 07:00 WIB berpotensi belum terhitung "hari ini".

---

## 3. DAFTAR PERTANYAAN AKUNDEMIK UNTUK SEKOLah/MENTOR

> Format sama dengan §4 PROGRESS-MAGANG: tiap nomor mandiri, bisa disalin-tempel.
> Kelompok A = paling mendesak (menentukan arah kerja berikutnya).

### Kelompok A — Watchlist & tindak lanjut (inti gap saat ini)

1. **Setelah sistem menandai "siswa perlu perhatian", apa yang diharapkan terjadi?** Sekarang berhenti di angka. Opsi: (a) cukup pantau manual oleh TU/kepala unit, (b) halaman detail berisi daftar nama + alasan flag (+ filter per kelas/wali kelas), (c) otomatis masuk antrian tindak lanjut (panggilan wali, catatan BK) yang statusnya di-track sampai selesai.
2. **Siapa berhak melihat watchlist?** Admin unit saja, atau wali kelas juga (untuk kelasnya), atau guru mapel? (Sekarang: hanya dashboard admin.)
3. **Ambang watchlist boleh diatur per unit/jenjang?** SD mungkin mau alpa ≥ 3, SMA ≥ 7 — sekarang 5 seragam & hardcoded.
4. **Perlu notifikasi ke wali murid saat anak masuk kategori tertentu?** (Sudah ada notifikasi ambang POIN — apakah pola yang sama diperluas ke absensi/nilai?)
5. **Perlu jejak intervensi?** Catatan "sudah dihubungi wali tanggal X, hasil Y" yang menempel pada siswa — supaya monitoring tidak mengulang dari nol tiap semester.

### Kelompok B — Penilaian & KKM

6. **KKM resmi berapa, dan seragam untuk semua mapel/jenjang?** Dashboard memakai 70 untuk semua — betulkah? Perlu per-mapel?
7. **Bobot 20/30/50** — sudah masuk daftar tunggu (§4 no. 6 PROGRESS), tetap ditanyakan.
8. **Kategori nilai cukup Tugas/UTS/UAS?** Kurikulum yang dipakai unit (merdeka?) biasanya punya formatif/sumatif, nilai praktik, atau sikap — apakah perlu?
9. **Ada alur remedial/pengulangan nilai di bawah KKM?** Siapa mencatat, kapan nilai remedial menimpa, apakah tampil beda di rapor?
10. **Predikat (A/B/C/D) perlu dikonversi otomatis dari nilai?** ( Rapor pusat mungkin sudah punya skema — koordinasi dengan pertanyaan rapor T15. )
11. **Deteksi "penurunan nilai" antar semester butuh 2 semester data.** Tahun pertama pemakaian pasti kosong — perlu deteksi intra-semester (mis. tren tugas menurun) atau diterima begitu?

### Kelompok C — Presensi

12. **Presensi gerbang/harian** — paket pertanyaan 4 butir sudah ada (§4 no. 10, T14). Tetap pertanyaan akademik terbesar yang belum dijawab.
13. **Definisi "absensi tinggi"**: hanya ALPA yang dihitung, atau sakit+izin kumulatif juga perlu dipantau? Per semester atau per tahun?
14. **Sesi retroaktif & manual murni** — sudah ditanyakan (§4 no. 11).
15. **Tindak lanjut alpa**: apakah butuh integrasi dengan BK (otomatis muncul di watchlist BK) dan notifikasi wali per-kejadian alpa, bukan hanya di akumulasi 5?

### Kelompok D — BK & poin tata tertib

16. **Apakah saldo poin "reset" tiap semester?** Ledger per semester ada — kebijakan pembacaannya (akumulasi vs reset) belum ditetapkan; memengaruhi arti ambang.
17. **Setelah notifikasi ambang terkirim, apa alur sekolah?** Panggilan orang tua → pembinaan → sanksi: perlu dicatat di sistem (status penyelesaian) atau cukup di luar?
18. **Poin prestasi (merit) punya konsekuensi positif yang perlu ditrack?** (penghargaan periode tertentu?)

### Kelompok E — Rapor & Al Azhar pusat

19. **Paket T15 tetap terbuka** (rekap internal / hapus / ekspor ke pusat) — plus pertanyaan turunannya: kalau ekspor, format apa yang diminta pusat?
20. **Guru input nilai di SIAKAD lalu ditarik pusat, atau double entry?** (§4 no. 9.)

### Kelompok F — Kenaikan kelas, kelulusan, perpindahan

21. **Fitur kenaikan kelas sudah dibangun pihak lain** — kriterianya perlu dicek sekolah: murni keputusan staf per siswa, atau ada syarat otomatis (nilai/presensi) yang harus dipatuhi sistem?
22. **Kelulusan jenjang & lanjut ke unit lain dalam YAPI** (SD-13 → SMP-12/55): apakah siswa "pindah unit" di SIAKAD, atau dianggap lulus + siswa baru via PMB lagi? ( Menentukan apakah perlu fitur transfer antar unit. )
23. **Siswa pindah unit/masuk tengah semester**: alur datanya apa (enrollment baru, tagihan prorata?) — sekarang belum ada alur khusus.
24. **Siswa keluar (dropped_out/transferred)**: apakah perlu dicatat alasan & tanggal efektif untuk laporan Dapodik?

### Kelompok G — Laporan & distribusi

25. **Perlu laporan periodik otomatis?** (ringkasan bulanan per unit ke kepala sekolah/yayasan — sekarang hanya dashboard real-time.)
26. **Dashboard keuangan perlu diperiodekan?** (all-time vs semester vs bulan — pilih default + filter.)
27. **Ekspor yang dibutuhkan unit selain Dapodik?** (rekap nilai per kelas ke Excel, daftar tunggakan per kelas untuk wali kelas, dll.)

---

## 4. ANALISIS MENYELURUH FITUR (posisi sekarang)

**Sudah matang & teruji:** auth OTP (wali+staf, rate-limit), manajemen pengguna (pusat+unit, impor CSV), master data (unit/TA/semester/mapel/kelas/jadwal), siswa + impor anti-ambigu, presensi per mapel (QR/NIS + manual guru + rekap), nilai (input bulk guru, rekap kelas, bobot 20/30/50), poin ledger + aturan + ambang + notifikasi, prestasi (ajuan wali/guru → verifikasi), ekskul (roster + daftar mandiri wali), pengumuman, tagihan lengkap (tarif+komponen+diskon → terbit massal → bayar VA/e-wallet/QRIS/tunai → waive/cancel → reminder), ekspor Dapodik, kenaikan kelas (pihak lain), log aktivitas, dashboard eksekutif.

**Sudah ada tapi menunggu keputusan:** rapor PDF (T15 — bentrok info "rapor resmi dari pusat"), presensi gerbang (T14 — belum didesain), arsip dokumen siswa (skema saja), retry notifikasi gagal.

**Titik lemah terbesar (hasil brainstorm):** semua indikator akademik berhenti sebagai ANGKA — tidak ada nama, tidak ada drill-down, tidak ada alur tindak lanjut. Nilai SIAKAD yang membedakannya dari file Excel justru di lapisan ini: deteksi dini → penugasan → jejak tindak lanjut → resolusi. Itu pertanyaan Kelompok A.

**Ketergantungan lintas keputusan:** rapor pusat (19–20) menentukan nasib modul nilai; watchlist (1–5) menentukan apakah dashboard berubah dari "laporan" jadi "alat kerja"; kenaikan kelas (21) menentukan apakah butuh kriteria otomatis dari nilai/presensi.
