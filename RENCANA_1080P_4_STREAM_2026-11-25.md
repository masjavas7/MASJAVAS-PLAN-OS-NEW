# RENCANA MIGRASI 720p → 1080p — 4 STREAM LIVE
Tanggal: 25 Nov 2026 ~05:40 WIB | Status: RENCANA — belum dieksekusi, menunggu persetujuan per stream
Dasar: DB Master `live_stream_tasks` + ffprobe video sumber di VPS 8 (live-01)

## Temuan kunci (mengubah rencana)
- Encoder memakai `-c:v copy` (tanpa scale/re-encode) — resolusi OUTPUT =
  resolusi file video sumber, BUKAN kolom DB `resolution`. Kolom DB hanya
  dipakai admission control `can_start` (worker/live_supervisor.py:558-565,
  800) untuk akuntansi TX: 720p = 3,0 Mbps, 1080p = 6,0 Mbps
  (config/settings.py:133-134).
- Dari 4 stream "720p" di DB, **2 sumbernya SUDAH 1080p** — yang salah hanya
  label DB. 2 sisanya sumbernya benar-benar 720p.
- Semua task `priority = 1` (tidak ada pembeda prioritas di DB) — urutan
  di bawah memakai risiko/effort terendah dulu.
- VPS 8 saat rencana dibuat: load 14,3/8 (skor node 19,7; "0.0 core bebas",
  13 kematian encoder/jam terakhir) — JENUH. Lihat § Dependensi.

## Status 6 stream (Master DB + ffprobe VPS 8)

| Task | Channel / Operator | DB res / bitrate | Sumber video (ffprobe) | Output riil |
|---|---|---|---|---|
| 1795191337447 | 706 / MONA | 1080p / 6000k | 1920×1080 | 1080p ✓ |
| 1795226249006 | 114 / MONA | 1080p / 6000k | 1920×1080 | 1080p ✓ |
| 1795321099635 | 707 / MONA | 720p / 3000k | 1920×1080 | **1080p (label salah)** |
| 1795502195546 | WULAN / WULAN | 720p / 3000k | 1280×720 | 720p |
| 1795502341605 | 088 / HAMIDAH | 720p / 3000k | 1920×1080 | **1080p (label salah)** |
| 1795502710790 | 034 / SALSA | 720p / 3000k | 1280×720 | 720p |

## Rencana per stream (urutan eksekusi)

### 1. 707 MONA — DB-ONLY, tanpa restart, risiko ~nol
- Perubahan: `resolution 720p→1080p`, `bitrate_kbps 3000→6000` di
  `live_stream_tasks` (Master DB; backup dulu).
- Restart encoder: TIDAK DIPERLUKAN (output sudah 1080p via copy).
- Dampak: akuntansi TX 3→6 Mbps (koreksi, bukan beban baru); CPU 0.
- Verifikasi: DB + sentinel/YouTube tetap LIVE; TX live-01 naik ≤3 Mbps.
- Persetujuan: 1× (update 2 kolom DB pada task live).

### 2. 088 HAMIDAH — DB-ONLY, tanpa restart, risiko ~nol
- Sama persis dengan 707 (sumber `gdrive_video_166j_MUeXP.mp4` 1920×1080).
- Persetujuan: 1× (update 2 kolom DB pada task live).

### 3. 034 SALSA — BUTUH SUMBER 1080p + restart encoder
- Sumber `gdrive_video_1IF6BuUG8C.mp4` benar-benar 1280×720.
- Langkah: (a) sediakan video loop 1080p pengganti (re-render/upscale dari
  bahan Drive — BUKAN re-encode pakai scale filter di encoder, itu mengubah
  `-c:v copy` menjadi encode penuh ~1+ core/stream); (b) letakkan di
  `/opt/masjavas-os/data/live_videos/` VPS 8 + perbarui `video_path` /
  `video_filename` + `resolution`/`bitrate_kbps` di DB; (c) restart encoder
  034 (stop + start via supervisor, ±1–2 menit downtime siaran).
- Persetujuan: 1× (ganti file video + restart encoder 034).
- Verifikasi: ffprobe output YouTube 1080p + sentinel LIVE + load VPS 8.

### 4. WULAN — BUTUH SUMBER 1080p + restart encoder, TERAKHIR
- Sumber `gdrive_video_1LU2dUOf8b.mp4` benar-benar 1280×720. Langkah sama
  dengan 034. Dikerjakan terakhir: channel namesake operator + riwayat
  3 ghost-kill (paling sensitif).
- Persetujuan: 1× (ganti file video + restart encoder WULAN).

## Dependensi (MENGIKAT — baca sebelum eksekusi)
- **1080p menambah beban per stream dan kotak sedang JENUH (load 14,3/8,
  skor node 19,7, 0,0 core bebas).** Item DB-only (707, 088) aman kapan
  saja (nol CPU, nol downtime, hanya koreksi label). Item 034 + WULAN
  (restart encoder + video baru) HARUS MENUNGGU isolasi prep + throttle
  yt-dlp (dikerjakan agen lain) sampai VPS 8 punya headroom
  (syarat: load1 <4 + kematian encoder <2/jam, konsisten dengan gate
  migrasi). Restart di kotak jenuh berisiko memperpanjang reconnect /
  memicu klaster kematian baru seperti ~05:32 WIB.
- TX BUKAN penghambat: live-01 15,4/294 Mbps; 4 stream × +3 Mbps = +12 Mbps.
- Baseline lantai (206 PUBLISHED/hari, 0 FAILED) tidak tersentuh rencana ini
  (tidak ada perubahan jalur publish; restart hanya encoder live per stream).

## Ringkasan persetujuan yang dibutuhkan (4×, satu per stream)
1. 707: update DB resolution+bitrate (tanpa restart).
2. 088: update DB resolution+bitrate (tanpa restart).
3. 034: ganti video sumber ke 1080p + restart encoder (SETELAH headroom).
4. WULAN: ganti video sumber ke 1080p + restart encoder (SETELAH headroom).
