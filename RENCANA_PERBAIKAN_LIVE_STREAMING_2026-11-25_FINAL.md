# RENCANA PERBAIKAN FITUR LIVE STREAMING — MASJAVAS TUBE OS (FINAL)

**Status:** FINAL — siap dipakai sebagai spesifikasi implementasi.
**Dibuat:** 25 Nov 2026. **Penyusun:** Hermes v1 → Muse v2 (koreksi) → Hermes v3 (validasi + penyempurnaan) → Muse FINAL (koreksi akhir, verifikasi DB 09:24).
**Sumber:** diagnosis handoff `DIAGNOSIS_FITUR_LIVE_STREAMING.md` (25 Nov 08:50) + audit kode Hermes (24 Nov) + verifikasi kode + DB langsung (25 Nov 09:12–09:24).

> Prinsip: **ukur dulu, baru skala.** Semua angka wajib diverifikasi di node yang bersangkutan (PID + RTMP + sentinel). Sitasi kode wajib `grep -n` — nomor baris bisa kedaluwarsa antar-commit.

---

## Changelog FINAL (yang berubah dari v3)

1. **Urutan A diperbaiki sesuai distribusi TERUKUR (bukan asumsi).** Distribusi cabang audio live 8/8 (DB, 09:24): relay 5, depot 1, GDrive 1, URL 1, **silence 0**, native 0. A2 (relay) kembali jadi item pertama; A1 (silence) dipindah ke terakhir — berlaku 0/8 saat ini. Klarifikasi: `mute_video_audio=1` (di semua 8 stream) artinya trek video di-mute, BUKAN cabang silence; cabang silence hanya terpakai bila stream tidak punya sumber audio sama sekali.
2. **Tabel biaya menerapkan headroom 30% secara konsisten** (kapasitas efektif 5,6 core/box). 696 stream pasca-A = **12–13 box, bukan 9**. Baris as-is juga dikoreksi (50 stream = 4–5 box, 100 = 8–9 box).
3. **Kontradiksi baris 50 stream diselesaikan.** 5 core muat di VPS 8 tapi hanya menyisakan 0,6 core untuk prep — milestone 50 stream dalam cap bergantung pada hasil ukur Fase 0 (A2 ≤0,07 core) atau prep pindah (+$18,65, total $53,32, di atas cap). Lihat catatan §4.
4. **A2: ketiga jenis feeder harus pindah ke AAC** (utama HLS, fallback playlist, silence pump) — format input encoder tetap sejak start; satu pompa masih PCM merusak `-c:a copy`.
5. **A3: klaim "bonus atasi E3" DIHAPUS (salah)** — pre-transcode tidak menambah jumlah trek; E3 adalah operasi konten. Ditambah: storage (~115 MB/channel, ~80 GB armada) + trigger rebuild saat playlist berubah.
6. **A1: file silence = stereo 44,1 kHz** (menyamai pipeline, bukan mono/32k) agar tidak ada perubahan layout audio bila sumber audio berubah.
7. **D1–D6: status AKTIF 09:17** (v3 menulis "aktivasi berjalan" — kedaluwarsa).
8. **Prinsip baru: migrasi skema wajib diuji di salinan DB live** (pelajaran insiden 25 Nov 09:11–09:17).

---

## 0. Ringkasan eksekutif

Stabil untuk **8 stream** (B10 beres, 698 live 09:01, D1–D6 aktif 09:17). Gap menuju ratusan channel ada **tiga**:

1. **Kapasitas CPU (struktural).** ~0,4–0,5 core/stream (bukan 0,05). VPS 8 (8 core) muat ~10–12 stream. Tidak bisa diselesaikan dengan bugfix.
2. **Operasional.** Jadwal tak bisa diedit, metadata bercelah, stream key tak jujur, tanpa audit trail, telemetri tak dipercaya.
3. **Vault/pairing (penghalang mengikat).** 459 UNPAIRED tanpa vault. Tanpa vault, channel TIDAK BISA live sama sekali — berapa pun box yang dibeli.

**Keputusan strategis:** jangan beli box dulu. Sebagian besar biaya/stream adalah **transcode audio live** — dan sebagian besarnya bisa **dipindah ke prep (sekali) atau dihilangkan (copy)**. Prinsip: ukur distribusi sumber audio dulu, lalu optimasi cabang yang dominan. **Distribusi terukur (09:24): relay 5/8 — jadi A2 (relay passthrough) adalah item pertama.** Penghematan potensial ~2,5–5× biaya box.

---

## 1. Keadaan terverifikasi (baseline 25 Nov 09:24 WIB)

| Aspek | Nilai | Catatan |
|---|---|---|
| Stream aktif | 8 STREAMING | VPS 8 (live-01); 698 live 09:01 |
| Cabang audio live | relay 5, depot 1, GDrive 1, URL 1, silence 0, native 0 | terukur DB 09:24; mute=1 di semua 8 |
| CPU per stream | ~0,4–0,5 core | feeder +4–10% tambahan |
| Kapasitas riil VPS 8 | ~10–12 stream | bukan 70 |
| Load VPS 8 | 7,3–10,5 (jenuh) | live + prep berbagi box |
| Publish | 30 PUBLISHED / 0 FAILED | lantai 206/hari + 0 FAILED |
| DB | SQLite WAL ~137–164 MB | 27 error lock/hari @ 8 stream |
| Kuota | 1.650 unit/video; 10.000/proyek/hari | maks 5 channel/vault |
| Vault/pairing | 60 vault; 459 UNPAIRED | 2 vault over-cap, 54 slot kosong |
| Egress | ~2 Mbps/stream | 100 ≈ 200 Mbps; 696 ≈ 1,4 Gbps |

**Sudah beres:** B10 (IPv4), watchdog, embed rejection, re-manifest, Stage 2 breaker, sentinel UNVERIFIED, `nice -n 5`, heartbeat sync, DB-locked retry (Fase 2.5), pre-copy 698, **D1–D6 AKTIF 09:17** (merge 61fc49f + fix migrasi 6f1b3c7; aktivasi sempat crash-loop web ~6 mnt — lihat prinsip migrasi skema).

---

## 2. Prinsip strategis

- **Ukur distribusi cabang audio DULU** — ROI tiap optimasi tergantung cabang mana yang dominan di armada. Distribusi saat ini sudah terukur (relay 5/8); ukur ulang bila armada bertambah.
- **Pindahkan transcode dari jalur live ke jalur prep.** Encode 24/7 itu mahal; encode sekali saat prep itu murah.
- **Satu variabel per eksperimen.** Jangan gabung dua perubahan.
- **Jangan sentuh publish** (lantai 206/hari + 0 FAILED).
- **Verifikasi pre/post + rollback + jendela observasi** untuk tiap perubahan.
- **Backup DB sebelum tulis produksi.** Tes isolasi = DB temp.
- **Migrasi skema wajib diuji di salinan DB live** (bukan hanya DB fresh): jalankan `init_schema()` 2× pada salinan backup DB live; indeks dibuat di blok migrasi SETELAH loop ALTER (pola expenses). Pelajaran insiden 25 Nov 09:11–09:17: `CREATE UNIQUE INDEX` di SCHEMA_SQL yang merujuk kolom hasil ALTER = web crash-loop ~6 menit.
- **Sitasi kode wajib grep.**

---

## 3. Item perbaikan (urut dampak)

### A. Turunkan biaya CPU per stream — prioritas #1

**A0. Peta biaya + distribusi cabang (baca & ukur sebelum mengubah apa pun).**
- Encoder (`worker/live_supervisor.py`, blok perintah — `grep -n 'base_cmd\|-c:v copy\|aresample'`): video `-c:v copy` di SEMUA cabang (biaya ≈ 0); audio `-c:a aac -b:a 192k` di semua cabang (di sinilah CPU dibakar).
- **6 cabang audio:** (1) relay/FIFO — PCM s16le 44,1 kHz + `-af aresample=async=1000`; (2) playlist concat; (3) file audio; (4) URL langsung; (5) MUTED/silence — `anullsrc` AAC 128k, HANYA bila tidak ada sumber audio sama sekali; (6) native audio video (hanya bila mute=0 + tanpa sumber lain).
- **Distribusi TERUKUR (DB, 25 Nov 09:24):** relay 5, depot/playlist 1, file GDrive 1, URL 1, silence 0, native 0. `mute_video_audio=1` di semua 8 stream — itu artinya trek video di-mute (sehingga audio eksternal dipakai), BUKAN cabang silence.
- **Aturan urutan:** kerjakan cabang dominan dulu. Saat ini: A2 (relay) → A3 (playlist) → A1 (silence, hanya bila ada channel tanpa audio).

**A1. Silence pre-encoded (cabang 5) — kondisional, kerjakan TERAKHIR.**
- Kini: `anullsrc` di-encode ke AAC live 24/7. Ganti: generate 1 file AAC sunyi (**stereo 44,1 kHz**, mis. 60 dtk — menyamai pipeline) sekali → encoder `-c:a copy` (atau `-stream_loop` file sunyi). Hampir nol CPU audio.
- **Dampak saat ini: 0/8 stream** — tidak ada channel live yang memakai cabang silence. Kerjakan hanya bila ada channel tanpa sumber audio, atau sebagai persiapan scale-up (channel baru tanpa audio akan jatuh ke cabang ini).
- Klasifikasi: L2.

**A2. Relay passthrough (cabang 1) — ITEM UTAMA (5/8 stream).**
- Target akhir: relay HLS (AAC) → RTMP tanpa decode/encode ulang.
- **Tiga blokir (harus dibuktikan satu per satu):** (1) filter `aresample` memaksa decode → harus dihapus/diganti tanpa merusak kompensasi drift; (2) kontinuitas timestamp saat video di-loop + audio kontinu + manifest HLS kedaluwarsa ~12 dtk; (3) codec konsisten (playlist depot = MP3; campur AAC+MP3 akan rusak).
- **Urutan aman (2 langkah):**
  1. **AAC-over-FIFO dulu:** feeder pompa AAC (ADTS, `-c:a aac` atau `-f adts`) bukan PCM; encoder `-c:a copy` dari pipe. **PERTAHANKAN FIFO O_RDWR** (ketahanan feeder-mati ≠ encoder-mati tetap ada). Hapus `aresample` → buktikan sinkron tetap terjaga.
  2. **Merge penuh kedua:** satu ffmpeg `-i <hls> -c:v copy -c:a copy -f flv rtmp://...` dengan `-reconnect 1 -reconnect_streamed 1 -reconnect_delay_max`. Hanya setelah langkah 1 terbukti 2 jam stabil.
- **WAJIB: ketiga jenis feeder pindah ke AAC** — feeder utama HLS, feeder fallback playlist (transcode MP3→AAC saat fallback aktif), dan silence pump (`anullsrc`→AAC atau loop file silence AAC). Format input encoder tetap sejak start; satu pompa yang masih PCM akan merusak `-c:a copy` saat pompa itu mengambil alih.
- **Kriteria lolos (wajib semua):** %CPU/stream ≤0,15 core DAN sinkron A/V stabil **2 jam** DAN 0 kematian + sentinel HEALTHY.
- **Ukur sinkron:** parse `-progress pipe:1` (out_time video vs audio) + monitor stderr "Non-monotonous DTS"/"pts" warning.
- Klasifikasi: L2 (rollout 1→3→8 stream).

**A3. Playlist/static pre-transcode offline (cabang 2/3/4) — menang untuk channel non-relay.**
- Playlist darurat (MP3) di-concat + transcode ke SATU file AAC saat prep (offline, sekali), bukan live. Encoder `-c:a copy` file AAC tersebut.
- **KOREKSI v3: A3 TIDAK mengatasi E3.** Pre-transcode tidak menambah jumlah trek — playlist 3 trek tetap 3 trek, hanya lebih murah diputar. E3 (playlist tipis) adalah operasi konten: tambah trek ke depot.
- **Storage:** ±115 MB/channel (30 trek × ~4 mnt × 128k); armada penuh 696 ≈ 80 GB. Muat di VPS 8 (236 GB kosong) tapi wajib dimonitor.
- **Trigger rebuild:** file AAC harus di-rebuild otomatis saat playlist berubah (ingestion menambah trek) — tanpa trigger, file jadi basi dan trek baru tidak pernah tayang.
- Klasifikasi: L2.

**A4. Bitrate (pelengkap).** 192k → 128k + `-aac_coder fast`; cabang MUTED → 64k (hanya bila A1 belum dikerjakan).
- Klasifikasi: L2.

### B. Pisahkan peran node

- **B1. Node live murni live — TANPA box baru (instruksi pemilik 09:28).** Optimalkan 2 box yang ada: evaluasi pindah `prep_worker` (Stage 2 + ingestion) ke Master (load ~1,9, longgar) agar VPS 8 penuh untuk live, atau load-gating + penjadwalan prep di VPS 8. JANGAN beli box baru. Klasifikasi: **L3** (infra — pindah service antar-node).
- **B2. Relay audio terpusat.** 1 service relay + cache manifest, bukan per-stream yt-dlp (badai ~2,5 core + Node.js ~3 core). Klasifikasi: **L3**.

### C. Data layer & sinkronisasi

- **C1. Bereskan divergensi DB Master ↔ VPS 8** (contoh: 698 ERROR lokal vs SCHEDULED Master). Satu sumber kebenaran (Master).
- **C2. `acquire_db_write_lock` lintas-proses.** `asyncio.Lock` per-proses (`grep -n '_WRITE_LOCK' core/database/connection.py`). Pola `fcntl.flock` sudah ada (`worker/studio_ads_automator.py`). **CATATAN: lock dipakai publish juga — perlakukan setara L3 (tes menyeluruh + rollout bertahap).**
- **C3. Pisah DB live dari publish** (Postgres / SQLite per-node + sync). **L3.**
- **C4. Jadwal `ANALYZE` + `VACUUM` + `wal_checkpoint(TRUNCATE)`.** L2.

### D. Observabilitas & audit

- **D1. Metrik per-stream** (bitrate, frame drop, uptime, reconnect, %CPU) + dashboard + alert Telegram. L2.
- **D2. Audit trail** create/start/stop/edit. L2/L3 (tabel baru → L3; bila menambah tabel, patuhi prinsip migrasi skema §2).

### E. Operasional

- **E1. Edit jadwal live** (F3) — kode Fase 0–5 sudah ada + aktif → verifikasi fungsional.
- **E2. Stream key jujur** + onboarding tuntas (8/13 MONA keyless; audit fleet-wide).
- **E3. Playlist darurat ≥30 trek/channel** (WULAN 3, 114 6) — operasi konten, bukan A3.
- **E4. DIRECT_URL** diabaikan supervisor → perbaiki ATAU tolak di form.
- **E5. Operasional segera:** login ulang cookie yt-dlp; URL relay pengganti 034 (butuh input operator).
- Klasifikasi: L2.

### F. Prasyarat scale-up — vault + onboarding

- **F1. Penetapan vault** 459 UNPAIRED (2 vault over-cap, 54 slot kosong ≈ 2,8 minggu @ +20/minggu). Penghalang mengikat.
- **F2. Pipeline vault baru** ~140 vault untuk 696 channel + cron monitoring pairing mingguan.
- **F3. Audit stream key fleet-wide** (berapa channel tanpa key — MONA 8/13 hanya sampel).
- Klasifikasi: L2/L3 (penetapan + pipeline butuh keputusan pemilik).

---

## 4. Estimasi biaya (dengan headroom 30% — diterapkan konsisten)

Basis headroom: **kapasitas efektif 5,6 core/box** (70% dari 8 core — jangan isi box >70%).

| Skenario | 50 stream | 100 stream | 696 stream |
|---|---|---|---|
| Apa adanya (0,4–0,5 core/stream) | 20–25 core ≈ 4–5 box ≈ **$75–93/bln** | 40–50 core ≈ 8–9 box ≈ **$149–168/bln** | ~278–348 core ≈ 50–63 box ≈ **$933–1.175/bln** |
| Pasca A (0,1 core/stream) | 5 core → **1 box** (lihat catatan) | 10 core → **2 box ≈ $37,30/bln** | 70 core → **12–13 box ≈ $224–243/bln** |
| Total + Master ($16,02) | $34,67 atau $53,32 (lihat catatan) | ~$53,32 (di atas cap) | ~$240–259 + box prep B1 |

**Catatan 50 stream (menyelesaikan kontradiksi v3):** 5 core muat di VPS 8 (5,6 efektif) tapi hanya menyisakan 0,6 core untuk prep + sistem — praktis tidak cukup. Tiga jalan: (a) prep pindah ke box sendiri (+$18,65 → total $53,32, **di atas cap**) — DITOLAK pemilik 09:28 (tanpa box baru); (b) A2 mencapai ≤0,07 core/stream (3,5 core, sisa 2,1 untuk prep → total $34,67, **dalam cap**); atau (c) prep pindah ke Master (box yang ada, load ~1,9 longgar — tanpa biaya baru) → VPS 8 penuh untuk live, total tetap $34,67 **dalam cap**. Jalan (c) adalah yang diperintahkan pemilik 09:28. Milestone 50 stream dalam cap bergantung pada hasil ukur Fase 0 + evaluasi B1, bukan asumsi.

**Catatan lain:** egress 696 ≈ 1,4 Gbps (batas ~200 stream/box di port 500 Mbps). B1 (pisah prep) TIDAK menambah box — prep di-rebalance ke Master (box yang ada) sesuai instruksi pemilik 09:28. Kuota/vault: 696 channel butuh ~140 vault (kini 60).

**Kesimpulan:** cap $34,67/bln cukup sampai milestone 50 stream **hanya bila** A2 mencapai ≤0,07 core/stream (prep tetap) — bila tidak, milestone butuh anggaran baru ($53,32). 100 stream butuh anggaran baru (~$53/bln). Keputusan pemilik.

---

## 5. Roadmap bertahap

| Fase | Isi | Output terukur |
|---|---|---|
| **0. Ukur** | (0) distribusi cabang — TERUKUR 09:24: relay 5/8 dominan → A2 dulu; (1) A2 AAC-over-FIFO; (2) A3 playlist offline; (3) A1 silence (hanya bila ada channel tanpa audio); model egress + vault (F) | %CPU per cabang + bukti sinkron 2 jam + model biaya |
| **1. Operasional cepat** | E1, E4, E5, D2 | edit jadwal jalan; tak ada input bisu |
| **2. Optimasi CPU** | A2→A3→A1 rollout 1→3→8 stream; A4 | %CPU/stream turun, load VPS 8 turun |
| **3. Observabilitas** | D1 | dashboard per-stream + alert |
| **4. Data layer** | C2 (hati-hati L3), C4, C1 | 0 error lock; DB sync 1 sumber |
| **5. Scale-out** | F (vault, paralel sejak Fase 0); B1 (rebalance ke box yang ada — tanpa box baru), B2, C3 | kapasitas/box naik; box live tambahan hanya bila scale menuntut + pemilik setuju |

**Milestone:** 50 stream stabil (bukan langsung ratusan) — dalam cap bila A2 ≤0,07 core/stream (lihat §4).

---

## 6. Keputusan pemilik

1. Target jumlah (50/100/200/500/696) + jangka waktu?
2. Jam tayang (24/7 vs slot berbagi grid 696 v1.0)?
3. Resolusi (1080p semua vs 720p mayoritas)?
4. Anggaran box live tambahan (/bln)?
5. Prioritas channel saat scale-out?
6. PARKED 278 + FAILED 21 Stage 2 (hapus/retry/biarkan)?
7. RETIRE vs REVIVE stage2-driver?
8. URL relay pengganti 034?
9. Akar readonly (root→ubuntu) setujui + restart prep?
10. Hermes pilot + Agent AI Fase C (lanjut/tahan)?
11. Pipeline vault + target pairing mingguan (setujui)?

---

## 7. Referensi

- `DIAGNOSIS_FITUR_LIVE_STREAMING.md` — handoff diagnosis.
- `engineering/DESAIN_JADWAL_LIVE_EDITABLE.md`, `ADENDUM_EDIT_METADATA_LIVE.md`, `DIAGNOSIS_MONA_LIVE_MANUAL_DAN_AUDIO_GDRIVE.md`, `RENCANA_PERBAIKAN_APLIKASI_2026-11-25.md`, `LAPORAN_PAIRING_PRIORITAS_2026-11-25.md`, `PAKET_KEPUTUSAN_FASE_C_2026-11-25.md`.
- Insiden migrasi skema 25 Nov 09:11–09:17 (AGENTS.md): uji migrasi di salinan DB live.
- Kode (via grep): `worker/live_supervisor.py` (encoder: `base_cmd`, `-c:v copy`, `aresample`, `anullsrc`), `worker/live_stream_engine.py` (feeder: `_build_*_feeder_cmd`, `LiveAudioRelayManager`, FIFO `O_RDWR`), `core/database/connection.py` (`_WRITE_LOCK`), `worker/studio_ads_automator.py` (`fcntl.flock`).

---

*Hermes v1 → Muse v2 → Hermes v3 → Muse FINAL. Dokumen FINAL, menunggu keputusan pemilik sebelum eksekusi.*
