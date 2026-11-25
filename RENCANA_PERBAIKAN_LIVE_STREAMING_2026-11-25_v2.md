# RENCANA PERBAIKAN FITUR LIVE STREAMING — MASJAVAS TUBE OS (v2 TERKOREKSI)

**Status:** DRAFT v2 untuk review pemilik.
**Dibuat:** 25 Nov 2026. **Penyusun v1:** Hermes (Creative Director Agent). **Koreksi v2:** Muse (verifikasi langsung ke kode produksi 25 Nov 09:12–09:20).
**Sumber:** diagnosis handoff `DIAGNOSIS_FITUR_LIVE_STREAMING.md` (25 Nov 08:50) + audit kode Hermes (24 Nov) + verifikasi kode langsung (25 Nov 09:12–09:20).
**Tujuan akhir:** ratusan channel live serempak (target operasional pemilik).

> Prinsip dokumen ini: **ukur dulu, baru skala.** Semua angka wajib diverifikasi di node yang bersangkutan (PID + RTMP + sentinel), bukan dari Master saja.

---

## Changelog v2 (yang berubah dari v1)

1. **A1 ditulis ulang total** — premis v1 salah: audio video loop di-MUTE default (`mute_video_audio` default 1 → cabang silence `anullsrc`). Passthrough trek video hanya berlaku di cabang native-audio (mute=0 eksplisit + tanpa sumber audio lain) — dampak ~nol di armada nyata. Sitasi `content_id_engine.py:288` dihapus (itu assembly audio publish, bukan video loop Stage 2).
2. **A2 diberi peringatan blokir** — cabang relay memakai filter `-af aresample=async=1000` (kompensasi drift); filter apa pun memaksa decode, jadi `-c:a copy` mustahil tanpa menghapus/mengganti kompensasi drift. Ditambah risiko kontinuitas timestamp HLS→RTMP dan kendala codec (audio playlist = MP3).
3. **Kriteria Fase 0 ditambah** — bukan hanya %CPU 10 menit, tapi juga **stabilitas sinkron A/V 2 jam** (drift check). Tanpa ini, A2 bisa "lolos" di CPU tapi rusak di sinkron.
4. **Hitungan biaya diperbaiki** — 100 stream × 0,1 core = 10 core > 1 box 8 core. Butuh **2 box live ≈ $37,30/bln** (+ Master $16,02 = ~$53,32 total, di atas cap $34,67). Klaim v1 "cap cukup" hanya benar untuk milestone 50 stream.
5. **Bagian F baru: prasyarat scale-up (vault + onboarding)** — 459 UNPAIRED tanpa vault adalah penghalang scale-up yang mengikat; v1 tidak menyebutnya sama sekali.
6. **Sitasi nomor baris diganti pointer grep** — nomor baris v1 kedaluwarsa (feeder di `live_stream_engine.py`, bukan supervisor:980; `nice` di ~1263, bukan ~961).
7. **C2 diberi catatan kehati-hatian L3** — lock dipakai publish juga; butuh tes + rollout bertahap setara L3 meski labelnya L2.
8. **Baseline diperbarui** — 8/8 STREAMING (698 live 09:01); ditambah baris vault/pairing dan egress.
9. **Item E5 baru** — operasional segera (login ulang cookie, URL 034).
10. **Keputusan #11 ditambah** — pipeline vault + target pairing mingguan.

---

## 0. Ringkasan eksekutif

MASJAVAS TUBE OS sudah stabil untuk **8 stream** (B10/RTMP IPv6 sudah beres, 0 kematian ~80 menit, 698 MONA live 09:01). Gap menuju ratusan channel ada **tiga**:

1. **Kapasitas CPU (struktural).** Satu stream = ~0,4–0,5 core (bukan 0,05 seperti asumsi lama). VPS 8 (8 core) hanya muat ~10–12 stream. Ini TIDAK bisa diselesaikan dengan bugfix.
2. **Operasional.** Jadwal tidak bisa diedit, editor metadata bercelah, stream key tidak jujur, tidak ada audit trail, telemetri tidak bisa dipercaya.
3. **Vault/pairing (penghalang yang mengikat).** 459 dari 462 UNPAIRED channel tidak punya vault. Tanpa penetapan vault + pipeline vault baru, scale-up ke 100+ channel tidak mungkin — berapa pun box yang dibeli.

**Keputusan strategis terpenting:** jangan beli box dulu. Sebagian besar biaya per stream (transcode audio eksternal + proses ganda feeder) **bisa dihilangkan**. Turunkan dulu biaya/stream, lalu hitung ulang kebutuhan box — bisa menghemat **~2,5–5×** biaya box live. Tapi koreksi penting: bahkan setelah optimasi berhasil, 100 stream tetap butuh box live ke-2 — cap $34,67/bln hanya cukup sampai milestone 50 stream.

---

## 1. Keadaan terverifikasi (baseline 25 Nov 09:01 WIB)

| Aspek | Nilai | Catatan |
|---|---|---|
| Stream aktif | 8 STREAMING | di VPS 8 (live-01); 698 live 09:01 |
| CPU per stream | ~0,4–0,5 core | terukur 05:12–05:20; feeder +4–10% |
| Kapasitas riil VPS 8 | ~10–12 stream | bukan 70 (asumsi lama salah) |
| Load VPS 8 | 7,3–10,5 (jenuh) | live+prep berbagi box |
| Publish | 26 PUBLISHED / 0 FAILED | lantai 206/hari + 0 FAILED |
| DB | SQLite WAL ~137–164 MB | 27 error lock/hari @ 7–8 stream |
| Budget | $34,67/bln (cap) | Master ~$16 + VPS 8 $18,65 |
| Kuota | 1 video = 1.650 unit; 10.000/proyek/hari | maks 5 channel/vault |
| Vault/pairing | 60 vault; 459 UNPAIRED tanpa vault | 2 vault over-cap, 54 slot kosong |
| Egress | ~2 Mbps/stream RTMP | 100 stream ≈ 200 Mbps; 696 ≈ 1,4 Gbps |

**Sudah beres (tidak perlu dikerjakan ulang):** B10 (paksa IPv4), watchdog encoder, embed rejection, re-manifest cepat, Stage 2 breaker, sentinel UNVERIFIED, `nice -n 5`, heartbeat sync, DB-locked retry lintas-proses (Fase 2.5), pre-copy 698 (video+audio, sha256 cocok). D1–D6 (duplikat) selesai + terverifikasi, aktivasi (merge + restart web) sedang berjalan di lane migrasi.

---

## 2. Prinsip strategis

- **Ukur sebelum ubah.** Hipotesis terbesar (audio passthrough) belum diukur. Jalankan eksperimen terkontrol 1 stream, ukur `%CPU` nyata + stabilitas sinkron, baru skala.
- **Satu variabel per eksperimen.** Jangan gabung dua perubahan dalam satu uji — hasilnya tidak bisa diatribusi.
- **Jangan sentuh publish.** Lantai 206 PUBLISHED/hari + 0 FAILED adalah garis mati.
- **Live disentuh terakhir**, satu stream terendah dulu.
- **Setiap perubahan butuh:** verifikasi pre/post + rollback + jendela observasi.
- **Backup DB sebelum tulis langsung ke produksi.** Tes isolasi = DB temp.
- **Sitasi kode wajib grep.** Nomor baris di dokumen bisa kedaluwarsa; verifikasi via `grep -n` sebelum mengutip.

---

## 3. Item perbaikan (urut dampak)

### A. Turunkan biaya CPU per stream — prioritas #1 (ROI terbesar)

**A0. Peta biaya nyata (baca dulu sebelum mengubah apa pun).** Encoder live (`worker/live_supervisor.py`, blok perakitan perintah — cari `base_cmd` + `-c:v copy`):
- Video: `-c:v copy` di SEMUA cabang — video tidak pernah di-encode ulang. Biaya video ≈ 0.
- Audio: `-c:a aac -b:a 192k` di SEMUA cabang — audio selalu di-encode ulang. **Di sinilah CPU dibakar.**
- 6 cabang audio (urut prioritas): (1) relay/FIFO — PCM s16le 44,1 kHz dari feeder + filter `-af aresample=async=1000:min_hard_comp=0.100000:first_pts=0`; (2) playlist concat; (3) file audio; (4) URL audio langsung; (5) MUTED/`mute_video_audio=1` — silence `anullsrc` di-encode AAC 128k; (6) native audio video — HANYA bila mute=0 eksplisit + tanpa sumber audio lain.
- Feeder (`worker/live_stream_engine.py`, cari `_build_*_feeder_cmd` + `LiveAudioRelayManager`): HLS → decode → PCM s16le → FIFO (manager pegang O_RDWR) → encoder re-encode AAC. Decode + encode dua kali.

**A1. Passthrough audio native video — DIBATASI (dampak kecil, prioritas rendah).**
- KOREKSI v1: audio video loop di-MUTE default (cabang 5, `mute_video_audio` default 1). Cabang native (6) hampir tidak pernah terpakai di armada nyata (semua stream memakai audio relay/playlist/GDrive). Passthrough `-c:a copy` di cabang ini menghemat ~nol pada fleet saat ini.
- Hanya kerjakan bila ada channel yang memang memakai audio native video. Uji terpisah dari A2.

**A2. Audio eksternal passthrough — ITEM UTAMA (hipotesis, belum diukur).**
- Target: relay HLS (AAC) → RTMP tanpa decode/encode ulang; gabung feeder + encoder jadi satu proses ffmpeg (`-i <hls> -c:v copy -c:a copy -f flv rtmp://...`).
- **Tiga blokir yang harus dibuktikan satu per satu:**
  1. **Filter aresample:** cabang relay memakai `-af aresample=async=1000` — filter apa pun memaksa decode. Menghapusnya berarti menghapus kompensasi drift A/V. Eksperimen wajib membuktikan sinkron tetap terjaga tanpa filter (atau menggantinya dengan kompensasi lain).
  2. **Kontinuitas timestamp:** desain FIFO+PCM ada untuk menjaga sinkron saat video di-loop (`-stream_loop -1`) + audio jalan terus + manifest HLS kedaluwarsa (~12 dtk). Satu ffmpeg baca HLS langsung harus membuktikan timestamp kontinu melewati refresh manifest — bukan hanya %CPU 10 menit.
  3. **Codec:** audio playlist depot adalah MP3. Passthrough ke FLV/RTMP butuh codec yang konsisten (AAC atau MP3) di seluruh stream; concat campuran AAC+MP3 akan rusak. Verifikasi codec per jalur sebelum klaim passthrough.
- **Kriteria lolos Fase 0 (ketiganya wajib):** %CPU/stream ≤0,15 core (dari 0,4–0,5) DAN sinkron A/V stabil 2 jam (cek drift via log/stats) DAN 0 kematian encoder + sentinel HEALTHY.
- Alternatif bila merge penuh gagal: pertahankan FIFO tapi feeder memompa AAC (bukan PCM) — encoder `-c:a copy` dari pipe ADTS. Silence/fallback pump harus ikut berubah ke AAC.
- Target: 0,4 → ~0,1 core/stream (bila ketiga blokir teratasi).

**A3. Resample — DIGABUNG ke A2 (bukan item mandiri).** `-ar 44100` + `aresample` adalah bagian kompensasi drift cabang relay, bukan beban mati. Jangan dihapus terpisah dari eksperimen A2.

**A4. Bitrate 192k → 128k + `-aac_coder fast`.** Hemat kecil, pelengkap. Catatan: cabang MUTED meng-encode silence 128k — peluang hemat kecil tambahan (turunkan ke 64k).

**Klasifikasi:** L2 (perubahan kode, reversible) — TAPI wajib eksperimen + rollout bertahap (1 stream → 3 → 8).

### B. Pisahkan peran node

**B1. Node live murni live.** VPS 8 kini `live + prep` se-box (QA ffmpeg Stage 2 125% CPU berebut dengan encoder live). Pindahkan `prep_worker` (Stage 2 + ingestion) ke box sendiri → bebaskan core untuk stream.
- **Klasifikasi:** L3 (perubahan infra, butuh persetujuan pemilik + anggaran). **Catatan biaya:** box prep adalah box TAMBAHAN — hitung dalam estimasi (§4).

**B2. Relay audio terpusat.** "Badai yt-dlp" = ~2,5 core + Node.js transient ~3 core tiap reconnect. Ganti: 1 service relay + cache manifest dipakai semua stream, bukan per-stream yt-dlp.
- **Klasifikasi:** L3 (service baru).

### C. Data layer & sinkronisasi (wajib sebelum ratusan channel)

**C1. Bereskan divergensi DB Master ↔ VPS 8.** Status live beda antar-DB (contoh: 698 ERROR di VPS 8 sementara Master tampilkan SCHEDULED). Satu sumber kebenaran (Master), DB lokal per-node untuk hot-write.

**C2. `acquire_db_write_lock` tidak serialize lintas-proses/node.** `asyncio.Lock` per-proses (`core/database/connection.py`, cari `_WRITE_LOCK`). Untuk multi-proses/node yang menulis heartbeat tiap tick, ini pecah. 27 error lock/hari sudah terjadi @ 7–8 stream. Fase 2.5 sudah menambahkan RETRY (toleransi) — C2 menambahkan SERIALISASI (pencegahan); keduanya komplementer, bukan redundan.
- **Klasifikasi:** L2 (kode) — solusi: file-lock lintas-proses (pola `fcntl.flock` sudah ada di `worker/studio_ads_automator.py`) atau writer tunggal. **CATATAN: lock ini dipakai publish juga — perlakukan dengan kehati-hatian setara L3 (tes menyeluruh + rollout bertahap) meski labelnya L2.**

**C3. Pisahkan DB live (hot, high-write) dari publish.** Evaluasi Postgres untuk status live, atau SQLite per-node + sync.
- **Klasifikasi:** L3 (infra/migrasi).

**C4. Jadwal `ANALYZE` + `VACUUM` + `wal_checkpoint(TRUNCATE)`.** DB tumbuh 137–164 MB tanpa maintenance.
- **Klasifikasi:** L2.

### D. Observabilitas & audit (murah, nilai operasional tinggi)

**D1. Metrik per-stream terpusat** (F15): bitrate, frame drop, uptime, reconnect count, %CPU per encoder + alerting Telegram.
- **Klasifikasi:** L2.

**D2. Audit trail** (F16): siapa create/start/stop/edit, kapan.
- **Klasifikasi:** L2/L3 (butuh tabel baru → L3 bila schema berubah).

### E. Operasional — selesaikan yang sudah dirancang

- **E1. Edit jadwal live** (F3): desain + kode Fase 0–5 sudah ada + aktif 08:40 → **verifikasi fungsional**. Bottleneck operasional terbesar.
- **E2. Stream key jujur** + onboarding tuntas (8/13 channel MONA keyless). Auto-key gagal diam-diam = musuh operasional.
- **E3. Perkaya playlist darurat** → ≥30 trek/channel (WULAN 3, 114 6 — sangat tipis).
- **E4. DIRECT_URL** diabaikan supervisor → perbaiki ATAU tolak di form (jangan terima input yang diam-diam jalan bisu).
- **E5. (Baru) Operasional segera:** login ulang cookie yt-dlp oleh operator (sesi kedaluwarsa — relay 403); URL relay pengganti slot 034 (sumber dihapus lisensi).
- **Klasifikasi:** L2.

### F. (Baru) Prasyarat scale-up — vault + onboarding

- **F1. Penetapan vault:** 459 UNPAIRED tanpa vault; 60 vault (2 over-cap 6 ch, 54 slot kosong ≈ 2,8 minggu pada +20/minggu). Tanpa vault, channel tidak bisa live SAMA SEKALI — ini penghalang yang mengikat, bukan sekadar backlog.
- **F2. Pipeline vault baru:** ~140 vault untuk 696 channel (maks 5/vault). Perlu proses pembuatan + pairing + monitoring mingguan (lihat `engineering/LAPORAN_PAIRING_PRIORITAS_2026-11-25.md` + spek monitoring).
- **F3. Onboarding stream key fleet-wide:** audit berapa channel tanpa key tersimpan (MONA 8/13 adalah sampel — ukur fleet-wide).
- **Klasifikasi:** L2/L3 (penetapan vault + pipeline baru butuh keputusan pemilik — keranjang Fase 3 item c).

---

## 4. Estimasi biaya (angka terukur, dikoreksi v2)

| Skenario | 50 stream (milestone) | 100 stream | 696 stream |
|---|---|---|---|
| Scale-out apa adanya (0,4–0,5 core/stream) | 20–25 core ≈ 3 box live ≈ **$56/bln** | 40–50 core ≈ 5–6 box ≈ **$93–112/bln** | ~40–45 box ≈ **$750–840/bln** |
| Setelah A2 berhasil (0,1 core/stream) | ~5 core → **muat di VPS 8 yang ada** (bila prep muat/pindah) | ~10 core → **2 box live ≈ $37,30/bln** | ~70 core ≈ 9 box ≈ **$168/bln** |
| Total dengan Master ($16,02) | ~$34,67 (dalam cap) | ~$53,32 (di atas cap) | ~$184 + box prep (B1) |

**Catatan yang tidak ada di v1:**
- 100 stream pasca-A2 butuh **2 box live**, bukan 1 (10 core > 8 core/box). Klaim v1 "1 box $18,65" salah.
- B1 (pisah prep) menambah 1 box lagi — belum dihitung di tabel di atas.
- Egress: 696 stream ≈ 1,4 Gbps; port per box ~500 Mbps → batas network ~200 stream/box (terpenuhi di 9 box, tapi wajib dimodelkan).
- Kuota/vault: 696 channel butuh ~140 vault (kini 60) — biaya non-box yang belum dihitung.

**Kesimpulan (dikoreksi):** eksperimen passthrough bisa menghemat ~2,5–5× biaya box live. Cap $34,67/bln cukup **sampai milestone 50 stream** (dengan catatan prep muat/pindah). 100 stream butuh anggaran baru (~$53/bln total) — keputusan pemilik.

---

## 5. Roadmap bertahap

| Fase | Isi | Output terukur |
|---|---|---|
| **0. Ukur** | Eksperimen A2 (relay passthrough, 1 stream: %CPU + sync 2 jam); uji A1-native terpisah (prioritas rendah); model egress + kuota/vault; perencanaan F (vault pipeline) | Angka %CPU nyata + bukti sync stabil + model biaya lengkap |
| **1. Operasional cepat** | E1 (edit jadwal), E4 (DIRECT_URL), E5 (cookie + 034), D2 (audit) | Edit jadwal jalan; tidak ada input bisu; cookie segar |
| **2. Optimasi CPU** | A2 rollout bertahap (1→3→8 stream); A4 pelengkap | %CPU/stream turun, load VPS 8 turun |
| **3. Observabilitas** | D1 (metrik + alert) | Dashboard per-stream + alert |
| **4. Data layer** | C2 (lock lintas-proses, kehati-hatian L3), C4 (maintenance), C1 (sync) | 0 error lock; DB sync 1 sumber kebenaran |
| **5. Scale-out** | F (vault pipeline, paralel sejak Fase 0); B1 (pisah prep), lalu B2 (relay terpusat), C3 (DB live) | Kapasitas per box naik; siap tambah box |

**Milestone pembuktian:** capai **50 stream stabil** (bukan langsung ratusan) sebagai validasi arsitektur + kuota + operasional sebelum ekspansi penuh. Milestone ini muat dalam cap $34,67/bln bila A2 berhasil.

---

## 6. Keputusan yang dibutuhkan dari pemilik

1. **Target jumlah pasti:** 50 / 100 / 200 / 500 / 696? Dalam jangka waktu?
2. **Jam tayang:** semua 24/7, atau slot berbagi/jam prime (grid 696 v1.0)?
3. **Resolusi final:** 1080p semua, atau 720p untuk mayoritas (trade-off kapasitas)?
4. **Anggaran:** berapa $/bln untuk box live tambahan? (100 stream ≈ $53/bln total; 696 ≈ $184+/bln.)
5. **Prioritas channel** saat scale-out (per monetisasi / operator / genre)?
6. **PARKED 278 + FAILED 21 Stage 2:** hapus / retry / biarkan?
7. **RETIRE vs REVIVE stage2-driver** (PID 1338969, IDLE 6d17h)?
8. **URL relay pengganti slot 034** (sumber dihapus lisensi)?
9. **Akar readonly (User=root → ubuntu):** setujui perbaikan + restart prep?
10. **Hermes pilot verdict + Agent AI Fase C (102 DRAFT, 0 applied):** lanjut / tahan?
11. **(Baru) Pipeline vault + target pairing mingguan:** setujui penetapan vault 459 ch + pembuatan vault baru + cron monitoring pairing?

---

## 7. Referensi

- `DIAGNOSIS_FITUR_LIVE_STREAMING.md` — handoff diagnosis lengkap (sumber utama).
- `engineering/DESAIN_JADWAL_LIVE_EDITABLE.md` — desain edit jadwal (F3).
- `engineering/ADENDUM_EDIT_METADATA_LIVE.md` — 12 saran editor metadata.
- `engineering/DIAGNOSIS_MONA_LIVE_MANUAL_DAN_AUDIO_GDRIVE.md` — M1–M5/G1–G7/T1–T6/D1–D6.
- `engineering/RENCANA_PERBAIKAN_APLIKASI_2026-11-25.md` — rencana induk (§0.8 = Fase 3).
- `engineering/LAPORAN_PAIRING_PRIORITAS_2026-11-25.md` — pairing + vault (Fase 3.1).
- `engineering/PAKET_KEPUTUSAN_FASE_C_2026-11-25.md` — keputusan Fase C.
- Kode kunci (cari via grep, jangan percaya nomor baris hafalan): `worker/live_supervisor.py` (blok encoder: `base_cmd`, `-c:v copy`, `-af aresample`), `worker/live_stream_engine.py` (feeder: `_build_*_feeder_cmd`, `LiveAudioRelayManager`, FIFO O_RDWR), `worker/live_health_sentinel.py`, `core/database/connection.py` (`_WRITE_LOCK`), `worker/studio_ads_automator.py` (pola `fcntl.flock`).

---

*Hermes (v1) + Muse (koreksi v2). Dokumen DRAFT, menunggu keputusan pemilik sebelum eksekusi.*
