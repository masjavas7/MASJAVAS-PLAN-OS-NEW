# ANALISA + SARAN PERBAIKAN INGESTION — Kurangi Antrean, Paralelkan di VPS 8

**Tanggal:** 24 November 2026, 04:45 WIB
**Diminta oleh:** Owner — kurangi antrean di `/ingestion`, percepat dengan
menambah paralelisme + optimalkan VPS 8
**Status:** Analisa + saran. Belum ada implementasi.

---

## 1. KEADAAN ANTREAN SAAT INI (terverifikasi 04:41 WIB)

| Metrik | Nilai |
|--------|-------|
| Job PENDING | **43** (dibuat 19:00 23 Nov – 01:15 24 Nov) |
| Job BLOCKED / PAUSED | 2 / 3 (PAUSED dari 19 Nov — 5 hari) |
| Job selesai 24 jam terakhir | **21** (~1 job/jam) |
| Estimasi habis antrean (tanpa job baru) | ~2 hari |
| Item ingestion | 17.132 total, SEMUA sudah READY/SYNCED — tidak ada item PENDING. Backlog ada di level JOB, bukan item. |
| Durasi job (selesai 24 jam terakhir) | 3 menit – 11 jam (median ~4 jam; sebagian besar adalah WAKTU TUNGGU antrean, bukan waktu proses) |

**Aktivitas ingestion VPS 8:** setelah fleet sync aktif (~04:30), ingestion
meledak 30 langkah dalam 12 menit (5 siklus × 6 langkah), lalu DIAM ~7 menit
(heartbeat 04:40 masih 30 langkah, 0 proses yt-dlp berjalan). Pola
burst-lalu-diam ini konsisten dengan throttle/governor yang aktif
(lihat §3).

---

## 2. PEMBATAS PARALELISME SAAT INI (terverifikasi di kode + konfigurasi)

| Pembatas | Nilai | Lokasi |
|----------|-------|--------|
| `--ingestion-jobs` (prep worker) | **2** job/siklus | `masjavas-prep.service` ExecStart |
| Default `run_ingestion_worker_cycle` | 4 job/siklus | `ingestion.py:6238` (tidak dipakai — dipanggil dengan 2) |
| **Throttle autopilot** | **→ 1 job** saat publish aktif | `ingestion.py:6166` |
| `max_steps_per_job` | 6 langkah/job/siklus | `ingestion.py:6241` |
| `MASJAVAS_FFMPEG_MAX_SLOTS` | **4** slot ffmpeg | `masjavas-prep.service` |
| Governor background ffmpeg | blokir bila load1 > 7,0 / load15 > 6,5 | `masjavas-prep.service` |
| Governor Stage 2 | blokir bila load1 > 6,0 | `.env` VPS 8 |
| Broadcast window 17:00–22:30 | heavy stage dijeda (TAPI turbo ON) | `INGESTION_TURBO_MODE=1` di `.env` — tidak berpengaruh saat ini |

**Temuan kunci 1 — throttle autopilot sudah KEDALUWARSA.**
Throttle memotong ingestion ke 1 job saat publish Master aktif. Ini masuk
akal saat ingestion + publish berbagi 1 CPU (arsitektur lama). Sekarang
publish di Master, ingestion di VPS 8 — CPU-nya TIDAK berbagi. Sumber daya
yang benar-benar berbagi hanya DB + kuota Drive. Throttle ini membuat
ingestion berjalan 1 job hampir sepanjang hari (publish hampir selalu aktif:
209 QUEUED), padahal slotnya 2. **Menghapus throttle = throughput job naik
2× seketika, tanpa menambah beban CPU apa pun di VPS 8.**

**Temuan kunci 2 — governor load sedang MENGEREM ingestion.**
Load15 VPS 8 saat ini 8,5–8,6 — DI ATAS ambang governor 6,5. Governor
memblokir ffmpeg background (termasuk langkah ingestion yang memakai ffmpeg).
Inilah kemungkinan penyebab ingestion diam setelah burst 30 langkah.

**Temuan kunci 3 — driver load terbesar adalah QA Stage 2, bukan ingestion.**
Snapshot CPU VPS 8 (04:41):

| Proses | CPU |
|--------|-----|
| 3× ffmpeg QA Stage 2 (blackdetect/freezedetect/silencedetect) | **~314% (~3,1 core)** |
| 3× encoder live | ~100% (~1 core) |
| prep_worker.py | 25% |
| warp-svc + live_supervisor | ~30% |
| **Total ffmpeg berjalan** | **10 proses** |

QA Stage 2 (verifikasi video hasil render) memakan ~3 core — lebih besar
dari encoder live. QA inilah yang mendorong load melewati ambang governor
dan mengerem ingestion. Ironis: pekerjaan verifikasi menghambat pekerjaan
produksi.

---

## 3. ANGGARAN CPU VPS 8 (8 vCPU)

| Pengguna | Core | Prioritas |
|----------|------|-----------|
| Encoder live (3 stream) | ~1,0 | **TERTINGGI — tidak boleh kelaparan** |
| Stage 2 render (4 slot) | ~2–3 (berfluktuasi) | Tinggi |
| Stage 2 QA (3 konkuren) | ~3,1 | Menengah (verifikasi, bisa tertunda) |
| Ingestion (download + proses) | ~0,5–1 | Menengah |
| Sistem (warp, supervisor, dll) | ~0,5 | — |
| **Total permintaan puncak** | **~7–8,6** | vs 8 core tersedia |

Kesimpulan: VPS 8 JENUH pada puncak (load 12–15). Menambah paralelisme
ingestion TANPA mengatur prioritas akan memperburuk kejenuhan dan berisiko
mengganggu live. Paralelisme harus ditambah secara CERDAS (lihat §4):
download (network-bound) boleh agresif, proses CPU harus dijaga.

---

## 4. SARAN PERBAIKAN

### P0-1 — Hapus / longgarkan throttle autopilot (dampak terbesar, risiko nol)

- **Apa:** hapus pemotongan ke 1 job di `ingestion.py:6166`, ATAU ubah
  pemicunya dari "publish aktif" menjadi "sumber berbagi tertekan"
  (kontensi DB / kuota Drive).
- **Dampak:** konkurensi efektif 1 → 2 job (2× throughput) tanpa
  menambah CPU VPS 8 sepersen pun — karena publish tidak memakai CPU VPS 8.
- **Risiko:** nol untuk CPU. Satu-satunya risiko adalah beban DB Master
  naik (2 job ingestion menulis DB bersamaan dengan publish) — kecil,
  DB sudah menangani beban jauh lebih besar.
- **Verifikasi:** throughput job/hari naik dari ~21; heartbeat
  "Ingestion Steps" naik stabil (bukan burst-lalu-diam).

### P0-2 — Naikkan `--ingestion-jobs` 2 → 4 (di prep worker)

- **Apa:** ubah ExecStart `masjavas-prep.service` + restart prep worker
  (1 restart, ~10 detik; antrean Stage 2 aman karena run tersimpan di DB).
- **Dampak:** konkurensi job 2 → 4. Digabung P0-1: efektif 1 → 4 job
  (4× throughput potensial).
- **Syarat:** HARUS bersamaan dengan P1-1 (prioritas live) + P1-2 (cap QA).
  Tanpa itu, 4 job ingestion + 4 slot Stage 2 + 3 QA akan mendorong load
  ke ~18–20 dan mengancam live.
- **Verifikasi:** antrean PENDING turun dari 43; load1 VPS 8 dijaga
  < 10; live tetap 3/3 STREAMING; kematian encoder tidak naik.

### P1-1 — Lindungi encoder live dengan prioritas CPU (prasyarat keamanan)

- **Apa:** beri live encoder prioritas tertinggi via `nice -10` /
  cgroup CPU shares (atau `cpuset`: reservasi 2 core untuk live).
  Stage 2 + ingestion jalan di prioritas normal/rendah.
- **Dampak:** paralelisme boleh naik tanpa risiko live kelaparan.
  Live adalah satu-satunya pekerjaan yang TIDAK BOLEH tertunda
  (stream putus = penonton hilang); render/ingestion boleh antre.
- **Verifikasi:** di bawah load buatan (4 Stage 2 + 4 ingestion),
  encoder tetap 3/3 STREAMING tanpa frame drop / RTMP timeout baru.

### P1-2 — Batasi QA Stage 2 (bebaskan ~2 core untuk ingestion)

- **Apa (pilih satu atau gabung):**
  - (a) Cap QA konkuren 3 → 1 (QA antre, render tidak tertunda —
    QA adalah pasca-proses, bukan jalur kritis).
  - (b) Jalankan QA dengan `-threads 1` + `nice 10` (rendah).
  - (c) Jadwalkan QA di jendela load rendah (tengah malam – subuh).
- **Dampak:** membebaskan ~2 core → ruang untuk 2 job ingestion
  tambahan TANPA menaikkan load total.
- **Verifikasi:** load15 turun di bawah 6,5 (governor lepas);
  QA tetap selesai (hanya tertunda, tidak hilang).

### P1-3 — Pisah lane download (network) vs proses (CPU)

- **Apa:** dalam satu job ingestion, tahap download yt-dlp (network-bound,
  CPU ~0) diparalelkan agresif (6–8 download bersamaan), sementara
  tahap proses ffmpeg tetap dibatasi semaphore (4 slot).
- **Dampak:** memanfaatkan bandwidth VPS 8 (terukur 315 Mbps down)
  yang saat ini menganggur (0 yt-dlp berjalan saat dicek). Download
  30 trek/channel bisa selesai dalam hitungan menit, bukan antre
  berurutan.
- **Catatan:** perlu perubahan kode di `ingestion.py` (download
  paralel per item dalam satu job, bukan per job). Upaya sedang.

### P2-1 — Naikkan slot ffmpeg 4 → 6 (dengan reservasi per lane)

- **Apa:** `MASJAVAS_FFMPEG_MAX_SLOTS=6`, dengan reservasi: Stage 2
  maks 3, ingestion maks 3 (tidak ada lane yang memonopoli).
- **Syarat:** sesudah P1-1 (prioritas live) — karena live encoder
  tidak memakai semaphore, perlindungannya harus dari cgroup/nice,
  bukan dari slot.
- **Dampak:** lebih banyak pekerjaan ffmpeg bersamaan tanpa
  saling blokir antar lane.

### P2-2 — Visibilitas antrean di halaman /ingestion

- **Apa:** tampilkan di halaman: (a) estimasi waktu habis antrean
  (43 job ÷ laju/jam), (b) perincian per tahap (berapa job di
  download / proses / upload / QA), (c) STATUS throttle/governor
  saat ini ("ingestion dijeda: governor load" / "throttle autopilot
  aktif" / "berjalan normal"). Saat ini operator melihat antrean
  tapi tidak tahu KENAPA lambat.
- **Dampak:** operator bisa membedakan "antrean normal" vs "ingestion
  macet" tanpa harus SSH.

### P2-3 — Bereskan 2 BLOCKED + 3 PAUSED (5 hari)

- **Apa:** tinjau 2 job BLOCKED + 3 job PAUSED (sejak 19 Nov).
  Batalkan bila channel-nya sudah tidak relevan, atau perbaiki
  penyebab blokirnya.
- **Dampak:** membersihkan antrean + membebaskan channel-nya
  (channel yang job-nya BLOCKED/PAUSED tidak bisa dibuatkan job baru).

### P2-4 — Profil durasi per tahap (diagnostik, bukan tebakan)

- **Apa:** ukur 1 job dari awal sampai selesai: berapa lama download
  vs proses ffmpeg vs upload Drive vs tunggu semaphore. Saat ini
  kita tahu job lambat, tapi tidak tahu tahap mana yang lambat.
- **Dampak:** perbaikan berikutnya bisa menyasar tahap yang benar
  (mis. bila upload Drive yang lambat, solusinya bukan paralelisme
  CPU melainkan tuning upload).

---

## 5. YANG TIDAK DISARANKAN

| Ide | Alasan ditolak |
|-----|----------------|
| Naikkan paralelisme tanpa prioritas live | Load sudah 12–15; tanpa P1-1, live akan kelaparan. |
| Pindahkan ingestion kembali ke Master | Master CPU menganggur (load 0,3) tapi arsitektur sudah diputuskan: ingestion di VPS 8. Membagi ingestion ke 2 node menghidupkan kembali kerumitan sync (B4). Opsi darurat saja bila VPS 8 jenuh total. |
| Proxy residential untuk yt-dlp (D1) | Ditunda owner 20 Nov — tidak diangkat lagi. |
| Hapus governor load | Governor adalah satu-satunya yang mencegah overload total. Yang perlu diperbaiki adalah DRIVER load-nya (QA), bukan governornya. |

---

## 6. URUTAN EKSEKUSI YANG DISARANKAN

| Urutan | Tindakan | Dampak | Risiko |
|--------|----------|--------|--------|
| 1 | P0-1 hapus throttle autopilot | 2× throughput, risiko nol | Rendah |
| 2 | P1-1 prioritas live (nice/cgroup) | Pengaman untuk semua langkah berikut | Rendah |
| 3 | P1-2 cap QA 3 → 1 | Bebaskan ~2 core | Rendah (QA hanya tertunda) |
| 4 | P0-2 `--ingestion-jobs` 2 → 4 + restart prep | 4× throughput potensial | Sedang (perlu observasi load 30 mnt) |
| 5 | P2-3 bereskan BLOCKED/PAUSED | Bersihkan antrean | Rendah |
| 6 | P1-3 lane download paralel | Download 6–8× lebih cepat | Sedang (perubahan kode) |
| 7 | P2-1 slot ffmpeg 4 → 6 | Headroom ffmpeg | Sedang |
| 8 | P2-2 visibilitas halaman + P2-4 profil tahap | Observabilitas | Rendah |

**Target yang bisa dijanjikan (bila langkah 1–4 jalan):** antrean 43 job
habis dalam ~12–18 jam (dari ~2 hari), dan ingestion mengimbangi
kedatangan job baru sehingga antrean tidak menumpuk lagi — dengan live
tetap 3/3 STREAMING.
