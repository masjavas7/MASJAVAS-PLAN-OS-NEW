# ANALISA WORKFLOW + BOTTLENECK MULTI-NODE — Penempatan File Master vs VPS 8

**Tanggal:** 24 November 2026, 04:15 WIB
**Diminta oleh:** Owner — "worker di VPS 8 seharusnya mengolah file di hardisk VPS 8;
periksa lagi workflow dan bottleneck yang terjadi"
**Premis owner: SEBAGIAN BENAR, SEBAGIAN BELUM TERJADI.**

---

## 1. PETA ALIRAN FILE (yang terverifikasi)

```
AGEN LOOPGEN (eksternal)
  │ submit file ke API Master
  ▼
MASTER DISK — agent-results/ 13 GB + photo-assets/ 14 GB  ← MASALAH B1
  │ (file mentah menumpuk di Master; Stage 2 TIDAK membaca dari sini)
  │
  │  Master membuat baris loopgen_stage2_runs (QUEUED) di DB Master
  │  ✗ TIDAK ADA mekanisme yang mengirim antrean ini ke VPS 8  ← MASALAH B4
  ▼
VPS 8 PREP WORKER (Stage 2)
  │ 1. Download RAW dari GOOGLE DRIVE (stage2.py:1861 — BUKAN dari Master)
  │ 2. Render lokal di VPS 8 (/opt/masjavas-os/data/loopgen/stage2, 437 MB)
  │ 3. Upload final LANGSUNG ke Drive folder channel (stage2.py:2177)
  ▼
GOOGLE DRIVE (folder channel 01_VIDEOS)

PUBLISH (worker Master)
  │ render dari bahan Drive → workspace/channels 40 GB di MASTER ✓ benar
  ▼
  DR SYNC (cron 1×/hari, 19:30 CET) menyalin SELURUH workspace 40 GB
  ke VPS 8 → /opt/masjavas-dr/workspace 39 GB  ← MASALAH B2 (duplikat)

INGESTION (prep worker VPS 8, 2 slot)
  │ menulis ke /opt/masjavas/workspace = POHON DR-SYNC (dengan --delete)
  │ ✗ 0 langkah dalam ~1 jam terakhir, padahal 43 job PENDING di Master
  ▼  ← MASALAH B3 (idle + konflik direktori)

LIVE (encoder VPS 8) → live_videos 68 MB di VPS 8 ✓ benar
```

---

## 2. YANG SUDAH BENAR

| Aliran | Penempatan | Keterangan |
|--------|-----------|------------|
| Stage 2 render | VPS 8 lokal (1,1 GB: stage2 437 MB + proof 638 MB) | Input dari Drive, output ke Drive — Master tidak ada di jalur data. Desain ini BENAR. |
| Live encoder | VPS 8 (live_videos 68 MB) | Benar. |
| Prep worker | Berjalan di VPS 8 (PID 286601, 4 slot Stage 2) | Benar. |

---

## 3. BOTTLENECK

### B1 — Input LoopGen (27 GB) menumpuk di Master untuk kerja yang dilakukan VPS 8 [KRITIS-DISK]

- Agen submit ke API Master → `agent-results/` 13 GB + `photo-assets/`
  14 GB tumbuh di Master. Terverifikasi MASIH TUMBUH: 97 file
  agent-results + 194 file photo-assets berubah dalam 2 jam terakhir
  (channel 127, 129–133).
- Stage 2 di VPS 8 TIDAK PERNAH membaca file-file ini — ia download
  raw dari Drive (`stage2.py:1861`, `download_drive_file`).
- Artinya salinan Master adalah REDUNDAN begitu salinan Drive ada.
  Master menanggung 27 GB untuk pekerjaan yang dikerjakan VPS 8.
- **Perbaikan struktural (pilih satu):**
  - (a) Agen submit langsung ke Drive + kirim file-ID ke Master
    (Master simpan metadata saja, bukan byte file). — Terbaik, tapi
    butuh ubah protokol submit agen.
  - (b) Master auto-hapus salinan lokal setelah upload Drive
    terkonfirmasi + Stage 2 DONE.
  - (c) Janitor hapus agent-results/photo-assets untuk run yang
    Stage 2-nya DONE (ini yang sedang dikerjakan subagent pembersihan
    sekarang — mengobati gejala, bukan akar).

### B2 — DR sync menduplikasi 40 GB cache yang bisa dibuat ulang [BOROS]

- `dr_sync_from_master.sh` langkah 5: `rsync -avz --delete` SELURUH
  `/opt/masjavas/workspace/` → VPS 8 (39 GB).
- Workspace adalah render cache (mp4 19,9 GB) + audio unduhan
  (mp3 16 GB + m4a 2,5 GB) — SEMUANYA bisa dibuat ulang dari Drive +
  YouTube. Bukan data yang harus diselamatkan saat bencana.
- Dampak: bandwidth + I/O disk tiap malam (churn ~38 GB/hari, jadi
  sebagian besar tersinkron ulang), 39 GB disk VPS 8 terpakai untuk
  salinan cache, rsync bersaing dengan encoder live + prep.
- **Perbaikan:** kecualikan render cache dari DR sync (mis. exclude
  `*.mp4` di workspace, atau sinkron hanya DB + live_videos +
  audio pool). Yang benar-benar perlu DR: snapshot DB (128 MB) +
  live_videos (68 MB).

### B3 — Ingestion di VPS 8 IDLE + menulis ke pohon DR [FUNGSI MATI + RISIKO]

- Heartbeat prep worker: "Ingestion Steps: 0" selama ~1 jam terakhir
  (uptime 54–57 menit, 0 langkah), padahal Master punya 43 job
  ingestion PENDING.
- Log berulang: "Autopilot actively running 5 operational task(s).
  Throttling ingestion concurrency to 1" — throttle aktif, tapi 0
  langkah berarti tidak ada job yang berjalan sama sekali.
- **Konflik direktori (terbukti di kode):** `ingestion.py:102,750`
  menulis ke `CHANNELS_WORKSPACE` = `/opt/masjavas/workspace` =
  pohon DR-sync. DR sync memakai `--delete`: file apa pun yang
  dibuat VPS 8 di pohon itu (tidak ada di Master) akan DIHAPUS pada
  sync berikutnya (setiap hari 19:30 CET = 01:30 WIB). Bila ingestion
  mulai bekerja, outputnya berisiko terhapus — dan bila sync jalan
  saat job sedang berjalan, file yang sedang ditulis bisa hilang
  di tengah jalan (race condition).
- **Perbaikan:** (1) selidiki kenapa 0 langkah (apakah antrean
  ingestion tidak terlihat dari DB VPS 8 — terkait B4?); (2) pindah
  direktori kerja ingestion VPS 8 ke luar pohon DR
  (mis. `/opt/masjavas-prep/workspace`).

### B4 — DB Master dan VPS 8 DIVERGEN — antrean Stage 2 akan habis ~06:00 WIB [KRITIS]

Bukti angka (04:10 WIB):

| Tabel `loopgen_stage2_runs` | Master DB | VPS 8 DB |
|---|---|---|
| DONE | 1.219 | **1.273** (+54, tak terlihat Master) |
| QUEUED | **179** | 33 |
| PARKED | 190 | 210 |
| RUNNING | — (tidak ada) | 6 |
| FAILED | — | 5 |
| Max run ID | **1.657** | 1.596 (tertinggal 61 run) |

- TIDAK ADA mekanisme sinkronisasi antrean: tidak ada kode sync di
  `master_watchdog.py`, tidak ada timer, tidak ada cron (selain DR
  harian — dan DR sync menulis ke file TERPISAH
  `dr_master_latest.sqlite`, BUKAN ke DB live VPS 8). Tidak ada
  skrip `dr_populate` (hanya file `.bak` sisa setup).
- DB VPS 8 adalah snapshot beku dari setup (~23 Nov). Prep worker
  bekerja dari snapshot itu: run-1563 sedang berjalan, 33 QUEUED
  tersisa. Laju ~19 run/jam → **antrean VPS 8 habis ~06:00 WIB**,
  lalu Stage 2 di VPS 8 MENGANGGUR sementara Master menumpuk 179
  QUEUED (dan terus bertambah).
- Arah sebaliknya juga putus: 54 DONE di VPS 8 tidak pernah sampai
  ke Master — Master tidak tahu pekerjaan itu selesai.
- **Perbaikan (pilih satu):**
  - (a) Prep worker klaim run via API Master (bukan baca DB lokal)
    + laporkan DONE via API. — Arsitektur yang benar (P4 arsitektur
    multi-node: mutasi hanya via API).
  - (b) Sinkronisasi tabel berkala (Master→VPS 8 untuk QUEUED,
    VPS 8→Master untuk DONE) — rapuh, dua arah, rawan konflik.
  - Rekomendasi: (a).

### B5 — `prep_worker.py` TIDAK ADA di git [RISIKO KEHILANGAN KODE]

- File hanya ada di VPS 8 (`/opt/masjavas-os/worker/prep_worker.py`),
  status git `??` (untracked). TIDAK ADA di Master, TIDAK ADA di repo.
- Bila VPS 8 rusak/hilang, kode prep worker hilang (yang berjalan
  hanya ada di memori + disk VPS 8).
- **Perbaikan:** commit + push segera (1 perintah). Masalah yang sama:
  `scripts/tunnel_guardian.py` juga untracked di Master.

---

## 4. RINGKASAN DAMPAK DISK

| Lokasi | Ukuran | Seharusnya | Status |
|--------|--------|-----------|--------|
| Master: agent-results + photo-assets | 27 GB | 0 (agen → Drive langsung) atau dihapus pasca-DONE | ✗ B1 |
| Master: workspace/channels | 40 GB | Di Master (publish jalan di Master) | ✓ benar |
| VPS 8: salinan workspace (DR) | 39 GB | 0 (cache tidak perlu DR) | ✗ B2 |
| VPS 8: loopgen stage2 + proof | 1,1 GB | Di VPS 8 (Stage 2 jalan di VPS 8) | ✓ benar |
| VPS 8: live_videos | 68 MB | Di VPS 8 | ✓ benar |
| Master: global_audio_pool | 16 GB | Perlu keputusan cap (5 vs 15 GB) | ⚠️ dari analisa L5 |

**Total yang bisa dibebaskan dengan perbaikan struktural:**
~27 GB di Master (B1) + ~39 GB di VPS 8 (B2) — tanpa kehilangan data
produktif apa pun.

---

## 5. URUTAN PERBAIKAN YANG DISARANKAN

1. **SEGERA (hari ini):** commit + push `prep_worker.py` (B5) — 5 menit,
   risiko nol, mencegah kehilangan kode.
2. **SEGERA (sebelum ~06:00 WIB):** putuskan mekanisme umpan antrean
   Stage 2 (B4) — atau Stage 2 di VPS 8 menganggur. Solusi darurat:
   salin snapshot DB Master ke VPS 8 secara manual 1× (memberi makan
   ~179 run, ~9 jam kerja) sambil membangun solusi API (a).
3. **P1 (minggu ini):** pindah submit agen ke Drive-langsung (B1a)
   atau auto-hapus pasca-DONE (B1b); kecualikan cache dari DR sync (B2);
   selidiki ingestion 0 langkah + pindah direktorinya (B3).
4. **P2:** arsitektur multi-node formal (§7.3 dokumen arsitektur:
   antrean terdistribusi + klaim via API) — B4 adalah bukti nyata
   kenapa dokumen itu dibutuhkan.
