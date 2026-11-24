# ANALISA FITUR AUDIT & REPACKAGING + INSTRUKSI PERBAIKAN UNTUK ANTIGRAVITY

**Tanggal:** Selasa, 24 November 2026
**Penulis:** Muse (analisa independen — dibaca dari kode + data live, bukan dari laporan)
**Status:** INSTRUKSI PERBAIKAN — menunggu eksekusi Antigravity
**Dokumen acuan:** DESAIN_FITUR_AUDIT_REPACKAGING_VIDEO_2026-11-24.md v1.1 (13 bab)

> Ringkasan satu kalimat: Fase 0 sudah hidup dan arsitekturnya benar,
> tetapi triase tidak berfungsi karena CTR difabrikasi menjadi konstanta
> 4,55%, hasil worker VPS 8 tidak akan pernah tampil di dashboard karena
> tidak ada sinkronisasi tabel audit, dan API audit terbuka untuk publik
> tanpa autentikasi.

---

## §1 — KONTEKS

### 1.1 Yang dibangun (5 commit, 07:07–07:32 WIB, 24 Nov 2026)

| Commit | Jam | Isi |
|---|---|---|
| 5191621 | 07:07:08 | Fase 0 Video Audit API, classifier, collector & Cockpit GUI (858 insersi) |
| 7baee29 | 07:07:48 | Batch audit worker dengan load-governor (`worker/audit_worker.py`, 84 baris) |
| 6b24b41 | 07:10:18 | Unify audit run epoch + 1-hour batch matching |
| 75190d9 | 07:14:02 | Ensure batch audited_at consistency |
| 3ecf76f | 07:32:11 | Route audit cockpit page via main.py dengan session auth |

File ditulis 07:26:37; web restart 07:27:02 (PID 971437) memuat seluruh
kode audit. Isi commit 3ecf76f SUDAH aktif di web yang berjalan (file
ditulis sebelum restart, meski commit dicap 07:32:11).

### 1.2 Status deployment (terverifikasi 07:32–07:40 WIB)

- Halaman `/audit`: LIVE, OWNER-only (GET tanpa login → 307 redirect ke `/login`).
- API `/api/audit/*`: LIVE, TANPA autentikasi (lihat §3 P1-1).
- Worker VPS 8: `masjavas-audit.timer` AKTIF (OnCalendar 18:00 UTC =
  01:00 WIB harian), service oneshot Nice=15 — BELUM PERNAH JALAN
  (journal kosong, DB VPS 8 0 baris).
- Tabel: `video_audit`, `audit_channel_summary`, `video_repackage_jobs`,
  `video_repackage_outcomes` — keempatnya ADA di DB Master, skema sesuai
  desain §4 + kolom v1.1.

### 1.3 Data live saat analisa

- Total video terbit: 2.768 (`operational_tasks`, status PUBLISHED +
  `youtube_video_id` terisi).
- Sudah diaudit: 251 video / 20 channel, satu batch ~07:12 WIB
  (semua `audited_at` = 1795479140).
- Kategori: A=0, B=0, C=0, D=0, "-"=251. Rekomendasi: PANTAU 20.
  Top opportunities: kosong.
- Biaya audit: 15 unit (15 entri AUDIT_READ di `vault_quota_ledger`).
- CTR: rata-rata 0,0158, maksimum 0,0455 (= persis 1/22 — lihat §3 P0-1).

---

## §2 — YANG SUDAH BENAR (terverifikasi, jangan diutak-atik)

1. Skema 4 tabel sesuai desain §4, termasuk kolom v1.1
   (`monetization_status`, `is_gone`), histori
   `UNIQUE(video_id, audited_at)`, dan indeks.
2. Collector memakai `youtube_video_id` dari DB — JOIN
   `operational_tasks` × `channels` tidak kehilangan satu video pun
   (2.768 = 2.768).
3. Pencatatan kuota ke `vault_quota_ledger` dengan operation_type
   AUDIT_READ, 1 unit per panggilan API — biaya benar.
4. Load governor worker (skip bila load1 > 90% vCPU) + Nice=15 +
   timer 01:00 WIB — sesuai desain.
5. Formula skor potensi persis v1.1: `impressions × (0,05 − CTR)`,
   di classifier maupun query summary.
6. Fase 0 benar-benar read-only: tidak ada satu baris pun kode edit
   video (`videos.update` / `thumbnails.set` tidak ada).
7. Penanganan video hilang (`is_gone = 1`) ada.
8. Halaman `/audit` terproteksi OWNER (307 → `/login` tanpa sesi).
9. Pencocokan batch 1 jam (6b24b41/75190d9) adalah perbaikan
   konsistensi yang masuk akal.
10. Murah: 20 channel hanya 15 unit.

---

## §3 — TEMUAN (urut prioritas)

### P0-1. CTR difabrikasi menjadi konstanta — triase mati total

**Bukti (kode):** `core/audit/collector.py` — query Analytics hanya
meminta metrics `views,estimatedMinutesWatched,averageViewDuration`
(TANPA impressions). Lalu:
- `impressions_est = int(views × 22)`
- `ctr_est = views / impressions_est = 1/22 ≈ 0,0455` — SELALU, untuk
  setiap video dengan views > 0.

**Bukti (data live):** `max(ctr)` = 0,0455 persis; distribusi CTR
bimodal (0 atau 0,0455); 251 video semuanya kategori "-"; A/B/C/D = 0.

**Dampak:** kategori A (syarat CTR < 2%) tidak pernah bisa muncul;
skor peluang = views × 0,1 (hanya urutan views); dashboard menampilkan
"semua sehat" (PANTAU 20) padahal classifier buta. Ini presentasi
paling berbahaya: hijau palsu.

**Perbaikan yang diharapkan:** minta metrik `impressions` langsung dari
Analytics API (`reports.query`, `dimensions=video` — metrik ini
tersedia). Hitung CTR = klik/impresi dari data asli. Bila metrik
impressions benar-benar tidak dapat diperoleh untuk channel tertentu,
klasifikasikan TANPA CTR dan tandai "CTR tidak tersedia" — JANGAN
PERNAH fabrikasi angka. Selaraskan jendela pengukuran: saat ini views
dari Data API adalah lifetime sementara durasi tonton dari Analytics
hanya 30 hari terakhir (`t_start` = now−33 hari, `t_end` = now−3 hari)
— dokumentasikan campuran ini atau selaraskan ke jendela yang sama.

**Kriteria penerimaan:** (a) distribusi CTR di `video_audit` menunjukkan
variasi nyata (bukan 2 nilai); (b) setidaknya ada video kategori A pada
audit penuh pertama setelah perbaikan (atau penjelasan berbasis data
bila benar-benar 0); (c) tidak ada lagi konstanta 22 atau rumus
views×22 di collector.

### P0-2. Hasil worker VPS 8 tidak akan tampil di dashboard (split-brain)

**Bukti:** `scripts/sync_fleet_queues.py` TIDAK menyebut tabel audit
sama sekali (grep 0 hit). Worker VPS 8 memakai `get_db_connection()` →
DB lokal VPS 8. Dashboard Master membaca DB Master. Timer pertama jalan
01:00 WIB malam ini → hasilnya masuk DB VPS 8, dashboard tetap kosong.

**Dampak:** seluruh audit terjadwal menjadi tidak terlihat; operator
mengira audit tidak jalan; dua sumber kebenaran untuk data audit.

**Perbaikan yang diharapkan (pilih satu):** (a) tambahkan `video_audit` +
`audit_channel_summary` ke `sync_fleet_queues.py` (dua arah, seperti
Stage 2 + ingestion); ATAU (b) worker menulis hasil lewat API Master
(rekomendasi D-1 desain); ATAU (c) jalankan worker terjadwal di Master
(audit itu API-bound, bukan CPU-bound — Master mampu). Opsi (a) paling
konsisten dengan arsitektur multi-node yang sudah ada.

**Kriteria penerimaan:** setelah satu siklus worker VPS 8, baris yang
ditulis di DB VPS 8 terlihat di `/api/audit/summary` Master dalam
≤ 5 menit (atau langsung, bila memakai opsi b/c).

### P1-1. API audit tanpa autentikasi, terekspos ke internet publik

**Bukti:** GET `/api/audit/summary` dan `/api/audit/channels`
mengembalikan JSON lengkap dari internet publik TANPA login (diuji via
curl publik 07:35 WIB). POST `/api/audit/run-phase0` juga tanpa auth
di kode — siapa pun bisa memicu audit berulang dan menghabiskan kuota
YouTube.

**Dampak:** kebocoran data performa armada + potensi pemborosan kuota
oleh pihak luar.

**Perbaikan yang diharapkan:** tambahkan dependency OWNER (pola yang
sama dengan route halaman di `main.py`: `get_user_from_request` + role
OWNER) ke SEMUA route `/api/audit/*`.

**Kriteria penerimaan:** GET `/api/audit/summary` tanpa sesi → 307/401;
dengan sesi OWNER → 200; POST `/api/audit/run-phase0` tanpa sesi →
ditolak.

### P1-2. Status audit in-memory, tidak andal di 4 worker uvicorn

**Bukti:** `api/routes/audit.py` — `_AUDIT_STATUS` adalah dict
module-level. Web berjalan dengan 4 worker uvicorn, tiap worker punya
salinannya sendiri. Guard `is_running` dan tampilan progres bergantung
pada worker mana yang melayani request.

**Dampak:** dua run bersamaan bisa terjadi di worker berbeda; progres
di dashboard bisa menunjukkan "tidak berjalan" saat audit sedang berjalan
di worker lain.

**Perbaikan yang diharapkan:** pindahkan status ke DB (tabel `audit_runs`
kecil: id, started_at, completed_at, current_channel, processed_videos,
total_videos, error) atau pakai file lock yang dibaca semua worker.

**Kriteria penerimaan:** dua POST `run-phase0` berurutan → yang kedua
ditolak dengan pesan "sedang berjalan", terlepas dari worker yang
melayani; `/api/audit/status` menampilkan progres yang sama dari worker
mana pun.

### P1-3. Rekomendasi HENTIKAN tidak pernah tercapai (bug urutan cabang)

**Bukti:** `core/audit/classifier.py` `summarize_channel` — cabang
HENTIKAN (semua video B, umur ≥ 90 hari) diletakkan SETELAH cabang
STRATEGI_LAIN (count_b ≥ 5). Channel yang semua videonya B dengan total
≥ 5 selalu tertangkap STRATEGI_LAIN dulu.

**Dampak:** kartu "KANDIDAT HENTIKAN" di dashboard akan selalu 0;
keputusan alokasi ulang kuota (salah satu tujuan fitur) tidak pernah
muncul.

**Perbaikan yang diharapkan:** periksa HENTIKAN SEBELUM STRATEGI_LAIN.

**Kriteria penerimaan:** unit test: channel dengan 5 video B berumur
100 hari → HENTIKAN; channel dengan 5 video B berumur 40 hari →
STRATEGI_LAIN.

### P2-1. Klasifikasi C/D memakai detik absolut, bukan retensi relatif

**Bukti:** classifier memakai `avg_view_duration` < 120 detik (D) dan
< 90 detik (C). Video armada adalah kompilasi 30–180 menit — durasi
tonton < 120 detik hampir selalu benar. Begitu CTR asli masuk, kategori
D akan mendominasi secara artifisial. Collector tidak mengambil
`contentDetails.duration` dari `videos.list`.

**Perbaikan:** tambahkan part `contentDetails` ke `videos.list`, simpan
durasi video, hitung retensi = `avg_view_duration` / durasi. Ambang
D/C memakai retensi % (desain §6.1: retensi < 30%).

### P2-2. Tidak ada governor kuota

**Bukti:** collector mencatat pemakaian ke ledger (AUDIT_READ) tetapi
tidak ada yang membaca ledger sebelum belanja. Tidak ada cap harian,
cap per-vault, atau cadangan publish (desain §5.5 mensyaratkan 4 aturan
governor).

**Perbaikan:** pre-flight check sebelum run: total hari ini (ledger) +
estimasi run ≤ anggaran; per-vault ≤ 500 unit/hari; cadangan publish
1.650 unit/vault tidak boleh disentuh; bila kuota tidak cukup → run
ditunda dan dicatat, bukan gagal diam-diam.

### P2-3. Tidak ada audit bertingkat

**Bukti:** setiap run mengaudit SEMUA video terbit (~500 unit untuk
audit penuh). Flag `--max-channels` hanya mengambil N channel pertama
(bukan bergilir).

**Perbaikan (desain v1.1 §3.1):** harian inkremental (video baru +
jendela ukur T7/T14, ~50–100 unit), mingguan bergilir (¼ katalog),
bulanan penuh. Timer 01:00 WIB menjalankan mode inkremental; mode penuh
dijadwalkan terpisah.

### P2-4. Kegagalan autentikasi vault menutupi data baik sebelumnya

**Bukti:** `collector.py` — bila `get_valid_google_credentials` gagal,
collector menyimpan baris nol (views 0, kategori "-") untuk semua video
channel itu. Karena ringkasan memakai batch terkini
(`MAX(audited_at)` − 3600), baris nol ini menjadi tampilan "saat ini"
meski data baik masih ada di histori.

**Perbaikan:** jangan simpan baris untuk channel yang gagal auth (atau
simpan dengan penanda gagal dan kecualikan dari ringkasan). Catat
kegagalan auth di log + tampilkan di dashboard sebagai "channel gagal
diaudit".

### P2-5. Nama channel tidak tampil; skor potensi tidak tersimpan

**Bukti:** record audit tidak membawa `channel_name` (query collect
mengembalikannya, tetapi tidak diteruskan ke record) →
`audit_channel_summary.channel_name` berisi ID channel.
`total_potential_score` dihitung di `summarize_channel` tetapi tidak
ada kolomnya di tabel dan tidak ikut INSERT → daftar channel diurutkan
`count_a`, bukan skor potensi (desain v1.1 §6.3).

**Perbaikan:** teruskan `channel_name` ke record; tambahkan kolom
`total_potential_score` ke `audit_channel_summary`; urutkan
`/api/audit/channels` berdasarkan skor potensi.

### P2-6. Dashboard belum memakai endpoint yang sudah ada + label menyesatkan

**Bukti (template):** `audit.html` tidak menampilkan tabel video
(endpoint `/api/audit/videos` ada) maupun tabel channel (endpoint
`/api/audit/channels` ada); tidak ada tampilan kuota audit terpakai,
timestamp audit terakhir (API mengembalikan `last_audited_at` tetapi
tidak dirender), kesenjangan 497 channel belum publish, atau breakdown
monetisasi. Teks "210 channel" hardcode. Kolom CTR menampilkan angka
fabrikasi seolah asli (kolom impresi jujur berlabel "Est.", kolom CTR
tidak). Klaim header "Klasifikasi berbasis respon penonton (Impresi,
CTR, Retensi)" saat ini tidak benar.

**Perbaikan:** tambahkan tabel video (filter per kategori) + tabel
channel; tampilkan kuota terpakai (jumlah AUDIT_READ dari ledger) +
timestamp audit terakhir + kesenjangan 497 channel + breakdown
monetisasi; ganti hardcode dengan data; beri label "Est." pada CTR
sampai data asli masuk (atau sembunyikan kolomnya); perbaiki klaim
header sampai P0-1 selesai.

### P2-7. Kolom traffic source selalu 0

**Bukti:** `traffic_search`/`suggested`/`browse` selalu 0 — query
Analytics tidak pernah meminta dimensi traffic source.

**Perbaikan:** tambahkan query kedua dengan
`dimensions=insightTrafficSourceType`, atau hapus kolom dari tampilan
sampai datanya ada.

---

## §4 — SOLUSI YANG DIHARAPKAN & KRITERIA PENERIMAAN (ringkas)

| # | Perbaikan | Kriteria penerimaan terukur |
|---|---|---|
| P0-1 | CTR asli dari Analytics | Distribusi CTR bervariasi; tidak ada konstanta 22; kategori A muncul pada audit penuh pertama (atau penjelasan berbasis data) |
| P0-2 | Sinkronisasi hasil VPS 8 | Hasil worker VPS 8 terlihat di API Master ≤ 5 menit |
| P1-1 | Auth OWNER di semua `/api/audit/*` | Tanpa sesi → 307/401; dengan sesi OWNER → 200 |
| P1-2 | Status audit di DB | Guard run ganda andal lintas worker; progres konsisten |
| P1-3 | Urutan cabang HENTIKAN | Unit test HENTIKAN vs STRATEGI_LAIN lolos |
| P2-1 | Retensi % | Ambang C/D memakai retensi relatif; durasi tersimpan |
| P2-2 | Governor kuota | Pre-flight check aktif; run ditunda bila kuota tidak cukup |
| P2-3 | Audit bertingkat | Mode inkremental/bergilir/penuh berjalan sesuai jadwal |
| P2-4 | Baris gagal-auth | Tidak menutupi data baik; kegagalan tampil di dashboard |
| P2-5 | Nama + skor tersimpan | Dashboard tampil nama channel; urutan berdasarkan skor potensi |
| P2-6 | Dashboard lengkap | Tabel video + channel + kuota + timestamp + kesenjangan 497 |
| P2-7 | Traffic source | Data terisi atau kolom dihapus dari tampilan |

---

## §5 — SARAN & IDE TAMBAHAN

1. **Audit delta:** tampilkan perubahan antar-audit (video yang pindah
   kategori) — lebih berguna dari snapshot statis.
2. **Kalibrasi ambang dari data:** setelah CTR asli masuk, hitung median
   CTR armada dan kalibrasi ambang 2% dari distribusi nyata (catatan
   desain §6.1).
3. **Antrean Fase 1:** tombol "masukkan ke antrean Fase 1" per video A —
   persiapan tanpa eksekusi (Fase 1 tetap menunggu keputusan owner).
4. **Alert:** bila channel yang tadinya sehat tiba-tiba punya ≥3 video A
   baru → notifikasi.
5. **Ekspor CSV** hasil audit untuk analisis di sheet.
6. **Validasi silang views:** bandingkan views lifetime (Data API) vs
   30 hari (Analytics) untuk mendeteksi video yang distribusinya baru
   saja berhenti.
7. **Tampilkan biaya kuota audit kumulatif + per-vault** di dashboard
   (transparansi).

---

## §6 — INSTRUKSI PELAKSANAAN

**Urutan:** P0-1 → P0-2 (SEBELUM timer pertama 01:00 WIB, 25 Nov) →
P1-1 → P1-3 → P1-2 → P2-1 s.d. P2-7.

**Aturan perubahan (mengikat, dari owner 22 Nov):** aplikasi sudah SOLID
— setiap perubahan wajib rencana verifikasi pra/pasca + rollback +
jendela observasi. Jangan pernah mengorbankan bagian yang solid demi
perbaikan. Baseline (206 PUBLISHED/hari, 0 FAILED/BLOCKED) adalah lantai.

**Aturan verifikasi (mengikat):**
- Buktikan perbaikan AKTIF dengan membandingkan waktu commit vs waktu
  start proses (web / prep / audit worker).
- Buktikan hasil PUBLIK (endpoint mengembalikan data yang benar), bukan
  hanya status DB.
- Jangan pernah mengklaim "100% SELESAI" tanpa bukti terukur per
  kriteria §4.
- Setiap perbaikan disertai unit test minimal untuk logika yang diubah
  (classifier, governor, guard).

**Catatan teknis:** file-file audit saat ini root-owned —
`sudo chown ubuntu:ubuntu` sebelum mengedit. Jangan commit file yang
tidak terkait. Jangan sentuh kode live/publish/ingestion/Stage 2 dalam
perbaikan ini.

**Format laporan per fase:** apa yang diubah (commit hash), bukti
aktivasi (PID + waktu start), bukti hasil (angka sebelum/sesudah +
query verifikasi), apa yang masih terbuka.

---

## §7 — LARANGAN

1. JANGAN membangun Fase 1 (Generator/Applier/pengeditan video) —
   keputusan owner K-1 s.d. K-7 belum dibuat. Fase 0 read-only saja.
2. JANGAN mengubah kode live streaming, publish, ingestion, atau
   Stage 2 dalam perbaikan ini.
3. JANGAN memicu audit penuh berulang untuk pengujian — setiap run
   penuh ≈ 500 unit. Uji dengan `--max-channels` kecil.
4. JANGAN mengklaim perbaikan selesai tanpa bukti aktivasi + hasil
   publik.
5. JANGAN mencetak stream key, token OAuth, atau URL bertanda tangan
   di log/laporan.
6. JANGAN menghapus data audit yang sudah ada (251 baris) — histori
   berguna untuk perbandingan.
7. JANGAN restart live/publish worker — hanya web (bila perlu memuat
   kode baru) dan itu pun maksimal 1 restart/15 menit lintas agent.

---

## §8 — LAMPIRAN: FAKTA KUNCI

- Repo: `masjavas7/MASJAVASTUBE-OS`, HEAD 3ecf76f (07:32:11 WIB,
  24 Nov 2026).
- Web: PID 971437, start 07:27:02 WIB, 24 Nov 2026 (memuat kode audit).
- File: `api/routes/audit.py` (194 baris), `core/audit/collector.py`
  (305), `core/audit/classifier.py` (96), `worker/audit_worker.py` (84),
  `web/templates/audit.html` (253), `api/main.py` (route `/audit` +
  auth OWNER, l.429–440), `config/systemd/masjavas-audit.service` +
  `.timer` (VPS 8).
- Timer VPS 8: OnCalendar 18:00 UTC = 01:00 WIB harian; service
  oneshot, Nice=15, User=ubuntu.
- Tabel: `video_audit`, `audit_channel_summary`,
  `video_repackage_jobs`, `video_repackage_outcomes` (Master).
- Data: 2.768 video terbit; 251 diaudit (20 channel, `audited_at`
  1795479140, ~07:12 WIB); A/B/C/D = 0; PANTAU 20; biaya 15 unit.
- Query verifikasi CTR: `SELECT avg(ctr), max(ctr), count(*) FROM
  video_audit;` — max harus BUKAN 0,0455 setelah P0-1.
- Query verifikasi kategori: `SELECT category, count(*) FROM
  video_audit GROUP BY category;`
- Query verifikasi kuota: `SELECT sum(units_consumed), count(*) FROM
  vault_quota_ledger WHERE operation_type='AUDIT_READ';`
- Query verifikasi sync P0-2: bandingkan `SELECT count(*) FROM
  video_audit;` di Master vs VPS 8 setelah satu siklus worker.
- Desain acuan:
  `~/workspace/engineering/DESAIN_FITUR_AUDIT_REPACKAGING_VIDEO_2026-11-24.md`
  v1.1 (763 baris, 13 bab).

---

*Dokumen ini disusun dari pembacaan kode langsung + data live
terverifikasi (24 Nov 2026, 07:32–07:40 WIB). Semua klaim dapat
diverifikasi ulang dari file dan query yang tercantum.*
