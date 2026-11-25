# FASE 0 — BASELINE PENGUKURAN LIVE STREAMING (TANPA PERUBAHAN KODE)

**Worker:** M (pengukuran baseline) · **Tanggal:** 25 Nov 2026
**Jendela ukur:** 09:30–09:48 WIB (03:30–03:48 CET) · **Node:** VPS 8 / live-01 (HEAD `173adec`)
**Metode:** read-only — `ps`/`/proc` (delta CPU ticks), `journalctl`, SQLite `mode=ro`, baca kode.
Tidak ada kode ditulis, tidak ada commit, tidak ada restart, tidak ada file ditulis di server.
Semua RTMP URL/stream key disensor dalam laporan ini.

---

## 0. Ringkasan eksekutif — 5 angka kunci

| # | Temuan | Angka |
|---|---|---|
| 1 | **Biaya per stream (encoder+feeder), terukur** | **mean 0,348 core/stream** (p50 0,358 · p90 0,419) — sedikit di bawah asumsi rencana 0,4–0,5 |
| 2 | **Total ffmpeg live 8 stream** | **mean 2,78 core** (261–290%) + supervisor 0,19 core |
| 3 | **Load VPS 8 selama jendela ukur** | load1 6,16–10,26 · load15 7,65–8,64 (8 core — jenuh, tapi turun dari 10,63 awal jendela) |
| 4 | **Relay HLS: TIDAK ADA yang mengalir** | **0/6 stream pipe memakai HLS** — semua di fallback playlist (3) atau silence (3); 0× "KEMBALI ONLINE" sejak restart 08:43 WIB |
| 5 | **Stream uji A2: TERBLOKIR (prasyarat)** | Tidak ada kandidat yang memenuhi "relay HLS bekerja" — E5 (cookie yt-dlp) harus diperbaiki dulu. Kandidat bersyarat: **NO 015** (utama) / **NO 088** (alternatif) |

**Implikasi terbesar:** baseline ini mengukur biaya dalam kondisi **fallback/silence**, bukan kondisi HLS-relay aktif. Bila HLS kembali hidup, biaya/stream kemungkinan naik sedikit (feeder HLS + probe yt-dlp). Angka 0,4–0,5 core/stream di rencana tetap aman sebagai batas atas.

---

## 1. Baseline %CPU (7 sampel, 6 interval, rentang 15,7 menit)

**Metode:** baca `/proc/<pid>/stat` (utime+stime) tiap ~3 menit, hitung `%CPU = Δticks / (Δdetik × 100) × 100`.
Sampel: 09:33:02, 09:36:01, 09:38:56, 09:41:53, 09:44:50, 09:47:46, 09:48:44 WIB.
Interval 1–5 ≈ 175–179 dtk; interval 6 hanya 58 dtk (sampel penutup) — sedikit lebih berisik, tetap dipakai.
Semua 8 encoder + 6 feeder PID-nya stabil sepanjang jendela (0 kematian, 0 restart).
`ps %CPU` (rata-rata seumur hidup proses) dipakai hanya sebagai cross-check.

### 1.1 Per stream — encoder, feeder, total

| NO / Channel | Cabang audio saat ukur | Encoder %CPU mean / p50 / p90 | Feeder %CPU mean (jenis) | **Total/stream mean / p50 / p90** |
|---|---|---|---|---|
| NO 015 Lelah Jingga (WULAN, 720p) | relay→fallback concat | 33,7 / 33,6 / 35,6 | 5,2 (concat) | **39,0 / 38,8 / 41,6** |
| NO 034 Ballad Jingga (SALSA, 720p) | relay→silence | 24,9 / 24,7 / 29,6 | 4,1 (silence) | **29,0 / 28,6 / 34,2** |
| NO 088 Rock Religi (HAMIDAH, 720p) | relay→silence | 19,8 / 19,8 / 21,1 | 4,2 (silence) | **24,1 / 23,8 / 25,6** |
| NO 114 Ruang Galau (MONA, 1080p) | relay→silence | 30,2 / 30,0 / 31,8 | 3,6 (silence) | **33,8 / 33,7 / 35,2** |
| NO 645 Reggae Tropis (AZIZ, 720p) | depot (file m4a) | 31,0 / 30,6 / 33,7 | — (file langsung) | **31,0 / 30,6 / 33,7** |
| NO 698 RuangIrama (MONA, 720p) | GDrive (file wav) | 39,5 / 39,2 / 41,3 | — (file langsung) | **39,5 / 39,2 / 41,3** |
| NO 706 melodi cover (MONA, 1080p) | URL→fallback concat | 31,4 / 31,5 / 33,0 | 9,3 (concat) | **40,8 / 40,5 / 42,5** |
| NO 707 RIFFORA (MONA, 720p) | relay→fallback concat | 31,9 / 32,1 / 34,3 | 9,2 (concat) | **41,1 / 41,3 / 44,3** |
| **Pooled 8 stream (n=48)** | — | — | — | **mean 0,348 core · p50 0,358 · p90 0,419** |

### 1.2 Total & overhead

| Komponen | %CPU mean (rentang antar-interval) |
|---|---|
| Semua encoder (8) | 242,5% (226,9–254,7) |
| Semua feeder (6) | 35,7% (33,7–39,0) |
| **Total ffmpeg live** | **278,2% (261,1–289,9)** |
| Supervisor `live_supervisor` (PID 1743566) | 18,7% (16,3–20,6) — overhead bersama, ≈ 0,023 core/stream bila diamortisasi |
| yt-dlp probe relay | Transien, tidak tertangkap stabil: burst ~40% selama hitungan detik, muncul di 4 dari 7 sampel (URL 707, 034, 088, 706 bergantian) + 1 proses `node` pendamping tiap probe |
| Stage 2 ffmpeg (prep) | PID berganti tiap sampel (job pendek); 1 sampel sempat 105–125% — BUKAN beban live, dicatat terpisah |

### 1.3 Load VPS 8 tiap sampel

| Sampel (WIB) | load1 | load5 | load15 |
|---|---|---|---|
| 09:33:02 | 9,93 | 8,83 | 8,26 |
| 09:36:01 | 10,26 | 9,56 | 8,64 |
| 09:38:56 | 7,13 | 8,44 | 8,38 |
| 09:41:53 | 7,62 | 7,95 | 8,19 |
| 09:44:50 | 6,16 | 7,10 | 7,81 |
| 09:47:46 | 6,68 | 7,28 | 7,77 |
| 09:48:44 | 6,39 | 7,01 | 7,65 |

Load1 turun 10,26 → 6,39 selama jendela (job Stage 2 pendek selesai); load15 tetap 7,65–8,64 — box tetap jenuh (8 core).
Catatan: snapshot `ps` awal 09:30 menunjukkan load 10,63/8,90/8,22.

### 1.4 Catatan baca angka

- **Encoder mendominasi: ~87% dari total ffmpeg live** (242,5 dari 278,2). Feeder hanya ~13%.
  Konsekuensi untuk A2: penghematan utama ada di encoder (`-c:a copy` menghilangkan transcode AAC),
  bukan di feeder. Feeder concat 5–9% dan silence ~4% adalah biaya kecil.
- **NO 698 paling mahal (39,5%)** meski tanpa pipe/`aresample` — input wav + `-ar 44100` di encoder.
  **NO 088 paling murah (24,1%)** — 720p + silence.
- **1080p vs 720p tidak tampak dominan:** 114 (1080p, 33,8%) lebih murah dari 706/707 (40,8/41,1%).
  Video `-c:v copy` di semua cabang; selisih lebih dipengaruhi feeder & sumber audio.
- Cross-check `ps %CPU` 09:30 (rata-rata seumur hidup): encoder 25–43% — konsisten dengan delta,
  sedikit lebih tinggi karena mencakup periode awal pasca-restart.

---

## 2. Peta proses per task (snapshot 09:30 + verifikasi 09:48)

Supervisor: PID **1743566** (`python -m worker.live_supervisor`, etime 47 mnt @09:30, %CPU ~18,7%).
Semua encoder PPID 1743566. Semua encoder: `-nostdin -stats -loglevel warning -re -stream_loop -1`,
video `-c:v copy`, audio `-c:a aac -b:a 192k -ar 44100`, output `-f flv rtmp://…` (disensor).

| NO | Encoder PID (etime @09:48) | Input encoder (ringkas) | Feeder PID + jenis (etime) | yt-dlp terkait |
|---|---|---|---|---|
| 015 | 1746378 (62 mnt) | video `gdrive_video_1LU2dUOf8b.mp4` + pipe `…015.pipe` + `-af aresample=async=1000…` | 1746349 — **concat** `playlist_emerg_1795502195546.txt` (3 trek) | probe periodik (terlihat 03:32 CET: `…/live/HWj1FwivQW8`) |
| 034 | 1745438 (63 mnt) | video `gdrive_video_1IF6BuUG8C.mp4` + pipe `…034.pipe` + aresample | 1761271 — **silence** `anullsrc` stereo 44,1 kHz (42 mnt) | probe periodik (`…/live/kizfrmykOkM` — URL sumber sudah dihapus) |
| 088 | 1746076 (63 mnt) | video `gdrive_video_166j_MUeXP.mp4` + pipe `…088.pipe` + aresample | 1753802 — **silence** `anullsrc` (54 mnt) | probe periodik (`…/live/weEWecCjTxM`, terlihat 09:41) |
| 114 | 1744795 (64 mnt) | video `gdrive_video_1K6tqWou5J.mp4` + pipe `…114.pipe` + aresample | 1764163 — **silence** `anullsrc` (37 mnt) | probe periodik (tidak tertangkap di sampel; manajer aktif) |
| 645 | 1745242 (64 mnt) | video `gdrive_video_1jLItvRcBx.mp4` + file m4a (Global Audio Pool) — **tanpa pipe, tanpa aresample** | — (file langsung, `-stream_loop -1`) | — |
| 698 | 1757771 (48 mnt) | video `…_broadcast_ready.mp4` + file wav GDrive — **tanpa pipe, tanpa aresample** | — (file langsung, `-stream_loop -1`) | — |
| 706 | 1747186 (62 mnt) | video `gdrive_video_1u5LFoUTS1.mp4` + pipe `…706.pipe` + aresample | 1747148 — **concat** `playlist_emerg_1795191337447.txt` (405 trek) | probe periodik (`…/live/SvMufMOj01I`, terlihat 09:44) |
| 707 | 1745183 (64 mnt) | video `gdrive_video_1jLItvRcBx.mp4` + pipe `…707.pipe` + aresample | 1745151 — **concat** `playlist_emerg_1795321099635.txt` (405 trek) | probe periodik (`…/live/GPC6SyIjBC0`, terlihat 09:32–09:33) |

**Cara pemetaan feeder silence:** cmdline `anullsrc` tidak membawa ID task — dipetakan via
`readlink /proc/<pid>/fd/1` → `/tmp/masjavas_live_audio_<task_id>.pipe`, dikonfirmasi `fuser` pada tiap pipe
(3 pihak per pipe: supervisor 1743566 + encoder + feeder).
**Tidak ada feeder HLS ffmpeg** (`-i https://…m3u8`) di snapshot mana pun (09:30, 7 sampel, 09:48).
**Tidak ada proses relay feeder HLS yang tersembunyi:** grep `ffmpeg` penuh + `lsof`-style fd-scan
hanya menemukan 8 encoder + 3 concat + 3 silence (+ Stage 2 pendek).

Status DB lokal VPS 8 (09:30, `mode=ro`): 8/8 `STREAMING`, `reconnect_count=0`, `priority=1` **untuk semua**,
`last_heartbeat` segar. `audio_source_type`: 114/707/015/088/034 = `YOUTUBE_LIVE_RELAY`,
706 = `YOUTUBE_URL`, 645 = `DEPOT`, 698 = `GDRIVE_URL`. Resolusi: 114 & 706 = 1080p, sisanya 720p.
Sentinel (journal 09:29–09:32): 7/8 `LIVE`, 698 `UNVERIFIED` (`no_broadcast`), 034 sempat `UNVERIFIED`
(`api_error`) lalu `LIVE`. 0 kematian encoder sejak restart 08:43 WIB (etime 62–64 mnt untuk 7 stream lama;
698: 48 mnt, live sejak 09:01).

---

## 3. Status feeder relay — HLS TIDAK mengalir di stream mana pun

### 3.1 Bukti berlapis

1. **Proses:** 0 feeder HLS ffmpeg di semua snapshot (lihat §2).
2. **Journal supervisor sejak restart (02:43 CET):** tepat **6× "Gagal ekstrak manifest sumber live"**
   — satu per stream relay/URL (706, 114, 707, 034, 088, 015), semua dalam 3 menit pertama pasca-restart —
   lalu **0× "✅ Siaran live sumber KEMBALI ONLINE!"** sampai 09:48 WIB (~65 menit).
3. **Alasan gagal (journal):**
   - 706/114/707/088/015: pesan cookie/bot-check yt-dlp (`-do-i-pass-cookies-to-yt-dlp …`) — ekstraksi
     manifest diblokir YouTube; sesi cookie kedaluwarsa (E5).
   - 034: `This video has been removed due to a contractual obligation with a music licensor` (video
     `kizfrmykOkM` dihapus) — tidak ada perbaikan teknis.
4. **Relay log (`data/live_logs/relay_<task>.log`):** ekor log 706/114/707 menunjukkan percobaan feeder HLS
   di 02:43:38 gagal buka segmen (`Error when loading first segment … /ip/2a09:…` — IPv6) +
   `Error opening input file https://manifest.googlevideo.com/…`; setelah itu hanya error feeder concat.
5. **Probe tetap jalan tapi gagal diam-diam:** manajer mem-probe manifest tiap 12 dtk
   (`live_stream_engine.py`, loop fallback: `ticks_since_probe` 6×2 dtk → `resolve_manifest_throttled`;
   kegagalan hanya dicatat ke `deque(maxlen=24)` via `_record_manifest_failure` — tanpa baris log per
   kegagalan, itulah sebabnya journal hanya menunjukkan 6 kegagalan). Bukti probe hidup: proses yt-dlp
   singkat muncul bergantian untuk URL 707/034/088/706 di 4 dari 7 sampel + 1 proses `node` pendamping.
6. **Fallback lalu mati sebagian → silence:** journal mencatat feeder fallback mati lalu beralih ke silence:
   - 088 (1795502341605): 02:54:15 CET — `Fallback feeder mati (exit=0) — playlist/file cadangan rusak`
   - 034 (1795502710790): 03:06:48 CET — sama
   - 114 (1795226249006): 03:11:32 CET — sama
   Relay log ekor 088/034/114: `[in#0/concat …] Error during demuxing: Operation not permitted` berulang —
   feeder concat gagal membaca entri playlist (stale/hilang). Feeder silence lalu mengambil alih dan
   stabil (etime 37–54 mnt @09:48).

### 3.2 Status per stream relay (09:48 WIB)

| NO | Cabang DB | Audio yang benar-benar mengalir | Bukti |
|---|---|---|---|
| 114 | `YOUTUBE_LIVE_RELAY` | **Silence** (`anullsrc`) sejak 03:11 CET | feeder silence PID 1764163 → pipe 114; fallback concat mati 03:11:32; playlist emerg hanya 6 trek |
| 707 | `YOUTUBE_LIVE_RELAY` | **Playlist darurat** (concat 405 trek) sejak 02:44 CET | feeder concat PID 1745151 hidup; HLS gagal 02:44:50 |
| 088 | `YOUTUBE_LIVE_RELAY` | **Silence** sejak 02:54 CET | feeder silence PID 1753802; fallback concat mati 02:54:15; playlist emerg hanya 1 trek |
| 034 | `YOUTUBE_LIVE_RELAY` | **Silence** sejak 03:06 CET | feeder silence PID 1761271; sumber dihapus lisensi; playlist emerg 1 trek |
| 015 | `YOUTUBE_LIVE_RELAY` | **Playlist darurat** (concat 3 trek) sejak 02:46 CET | feeder concat PID 1746349 hidup; HLS gagal 02:46:06 |
| 706 | `YOUTUBE_URL` | **Playlist darurat** (concat 405 trek) sejak 02:44 CET | feeder concat PID 1747148 hidup; manajer relay restart 1× 02:46:47, tetap gagal manifest |

**Kesimpulan §3:** label DB `YOUTUBE_LIVE_RELAY` tidak sama dengan "HLS mengalir". Saat ini **0/6** stream
pipe yang audio-nya berasal dari relay HLS live. Tiga stream (114/088/034) bahkan **bisu** (silence) —
encoder tetap jalan dan Sentinel tetap `LIVE` karena yang diverifikasi hanya lifecycle broadcast,
bukan isi audio.

---

## 4. Rekomendasi stream uji A2

### 4.1 Putusan: UJI A2 BELUM BISA DIMULAI — prasyarat tidak terpenuhi

Kriteria rencana (§3 A2): *relay + HLS mengalir + prioritas terendah*. **Tidak satu pun dari 6 stream
pipe memenuhi "HLS mengalir" saat ini.** Menguji AAC-over-FIFO pada stream yang HLS-nya mati berarti
menguji pada feeder fallback/silence — bukan pada jalur yang hendak dioptimasi — dan hasilnya tidak
membuktikan apa-apa tentang passthrough HLS.

**Prasyarat sebelum uji A2 (urutan):**
1. **Perbaiki E5 — login ulang cookie yt-dlp** (operator, via dashboard) sampai `yt-dlp -j -g` pada satu
   URL relay berhasil manual.
2. **Verifikasi HLS mengalir:** feeder ffmpeg `-i https://…m3u8` muncul di `ps` + journal mencatat
   "KEMBALI ONLINE" + feeder bertahan ≥30 menit tanpa jatuh ke fallback.
3. Baru pilih stream uji dari daftar di bawah.

### 4.2 Penilaian per kandidat (kondisi bila HLS sudah pulih)

| Kandidat | Operator / Resolusi | Alasan pro | Alasan kontra | Verdict |
|---|---|---|---|---|
| **NO 015 Lelah Jingga** | WULAN / 720p | Bukan MONA; 720p; **fallback concat HIDUP** (jaring pengaman bila HLS putus saat uji); playlist 3 trek | Fallback tipis (3 trek) — perbaiki dulu ke ≥30 trek (E3) sebelum uji | **PILIHAN UTAMA** |
| **NO 088 Rock Religi** | HAMIDAH / 720p | Bukan MONA; 720p; encoder termurah (24,1%) — baseline rendah, efek A2 paling mudah dibaca | **Fallback concat SUDAH MATI** (playlist 1 trek rusak) — saat ini silence; tanpa fallback hidup, uji berisiko bisu bila HLS putus | Alternatif — hanya bila playlist darurat diperbaiki dulu |
| NO 114 Ruang Galau | MONA / 1080p | — | MONA (prioritas lebih tinggi); 1080p; fallback mati → silence; playlist 6 trek | Hindari |
| NO 707 RIFFORA | MONA / 720p | Fallback 405 trek hidup | MONA | Hindari |
| NO 706 melodi cover | MONA / 1080p | Fallback 405 trek hidup | MONA; 1080p; tipe `YOUTUBE_URL` (bukan relay murni) | Hindari |
| NO 034 Ballad Jingga | SALSA / 720p | — | **DIKECUALIKAN** — sumber relay dihapus YouTube (lisensi); butuh URL pengganti dari operator (E5) | Tidak eligible |

**Catatan prioritas:** kolom `priority` di DB = **1 untuk semua 8 stream**, dan supervisor mengurutkan
`ORDER BY priority ASC` (1 = tertinggi) — jadi prioritas DB tidak membedakan siapa "terendah".
Pemilihan di atas memakai proksi yang tersedia: operator non-MONA + 720p + fallback hidup.

### 4.3 Pilihan final (bersyarat)

**NO 015 Lelah Jingga (WULAN, 720p, task 1795502195546)** — segera setelah HLS-nya terbukti mengalir
≥30 menit dan playlist daruratnya diperkaya (E3). NO 088 sebagai cadangan bila 015 tidak bisa dipakai,
dengan syarat playlist daruratnya diperbaiki dulu (saat ini 1 trek, rusak).

---

## 5. Protokol ukur sinkron 2 jam (uji A2)

### 5.1 Yang benar-benar diukur hari ini vs yang diasumsikan rencana

Rencana menyebut "parse `-progress pipe:1` + monitor stderr Non-monotonous DTS/pts warning".
Verifikasi kode (`grep`, `live_supervisor.py` / `live_stream_engine.py`):

- **`-progress` TIDAK dipakai supervisor saat ini** — tidak ada kemunculan `-progress` di
  `live_supervisor.py`. Encoder di-spawn dengan `-stats -loglevel warning` (`live_supervisor.py:1267-1271`)
  dan stdout+stderr digabung ke **file log** `data/live_logs/live_<task_id>.log` (mode **append**,
  `live_supervisor.py:1256`; `Popen(..., stdout=log_file, stderr=STDOUT)`, `:1366`).
  Memakai `-progress` butuh perubahan kode — **protokol di bawah memakai mekanisme yang sudah ada**.
- **Fungsi parse yang sudah ada:** `parse_ffmpeg_stats_tail()` (`:509`) membaca 64 KB ekor log dan mengambil
  baris `-stats` terakhir → `(size_bytes, time_sec)`; `get_stream_progress(task_id)` (`:546`) membungkusnya.
  Watchdog memakai fungsi yang sama.
- **"Non-monotonous DTS"/`pts` warning: 0 kemunculan di SELURUH 8 log encoder** (file 0,2–17 MB, sejak
  restart). Jadi metrik ini **tidak punya baseline** — tidak bisa jadi metrik lolos/gagal utama;
  dipakai hanya sebagai pencatat kejadian (kemunculan apa pun = sinyal untuk ditinjau).
- **Watchdog yang sudah ada** (`:114-115, :2176`): `ENCODER_WATCHDOG_FREEZE_SEC=30`,
  `ENCODER_WATCHDOG_WARMUP_SEC=90`; verdict progres tiap ≥30 dtk (`:1830-1846`): size/time bergerak =
  sehat; size+time mundur bersamaan = encoder baru (re-seed, karena log append); diam = beku →
  kill + reconnect bila beku ≥30 dtk setelah warmup 90 dtk. Tick supervisor 5 dtk
  (`LIVE_GUARD_SAMPLER_INTERVAL_SEC=5`, `:358`).
- **"Sinkron" di sini bersifat operasional:** video adalah loop yang diputar ulang dan audio adalah
  sumber independen — tidak ada ground truth lip-sync. Yang bisa diukur: (a) jam stream maju 1:1
  dengan jam dinding (tidak drift), (b) tidak ada jeda/beku audio, (c) tidak ada kematian encoder,
  (d) tidak ada peringatan timestamp.

### 5.2 Definisi operasional "sinkron stabil"

Uji 2 jam pada SATU stream (NO 015, bila prasyarat §4.1 terpenuhi), A2 langkah 1 (AAC-over-FIFO,
`-c:a copy`, tanpa `aresample`) aktif sejak t=0.

| # | Metrik | Cara ukur (tanpa ubah kode) | Ambang LOLOS (2 jam penuh) |
|---|---|---|---|
| M1 | **Drift jam stream** | Tiap 30 dtk baca ekor `live_<task>.log` via `parse_ffmpeg_stats_tail` (atau `get_stream_progress`): catat `time_sec` + jam dinding. Drift per jendela 10 mnt = Δtime − Δwall. | \|drift\| ≤ **1,0 dtk per 10 mnt** DAN kumulatif ≤ **6 dtk per 2 jam** |
| M2 | **Kecepatan `speed=`** | Dari baris `-stats` yang sama: rata-rata `speed=` per jendela 10 mnt. Baseline hari ini (log 114): dominan 0,995–1,02x. | Mean per jendela **0,98–1,02x**; tidak ada jendela 10 mnt dengan mean di luar 0,97–1,03x |
| M3 | **Freeze (beku)** | `size=` DAN `time=` diam bersamaan (kriteria watchdog). | **0 kejadian** beku ≥30 dtk (watchdog akan kill bila terjadi — kill apa pun = gagal) |
| M4 | **Kematian encoder / reconnect** | PID encoder + `reconnect_count` DB + journal `[WatchdogProgres]` / kematian. | **0 kematian**, 0 reconnect, PID sama 2 jam |
| M5 | **Kontinuitas feeder** | PID feeder HLS + state manajer (`_ACTIVE_RELAY_MANAGERS` / journal): tidak ada "Siaran live sumber terputus" / fallback / silence selama uji. | **0 fallback** — feeder HLS yang sama hidup 2 jam |
| M6 | **Peringatan timestamp** | `grep -c "Non-monotonous DTS\|DTS.*invalid\|pts"` pada `live_<task>.log` tiap 10 mnt (baseline: 0). | **0** = lolos; 1–10 = lolos bersyarat (catat + tinjau manual); **>10 = gagal** |
| M7 | **Sentinel** | Journal `sentinel_probe` task tsb tiap ~2 mnt. | `verdict=LIVE` **sepanjang uji** (tanpa `UNVERIFIED` beruntun) |
| M8 | **%CPU/stream** | Metode §1 (delta ticks, sampel tiap 3 mnt) selama 2 jam. | **≤0,15 core** (kriteria rencana A2) |

**Kriteria lolos keseluruhan:** M1–M5 + M7 + M8 semua lolos DAN M6 ≤10. **Satu saja gagal → uji gagal**,
rollback ke konfigurasi sebelum A2, dan diagnosis sebelum uji ulang.
**Kriteria gagal cepat (stop uji sebelum 2 jam):** M3/M4 terjadi, atau M5 jatuh ke fallback, atau drift
kumulatif >6 dtk sebelum 2 jam — hentikan, catat waktu + log, rollback.

### 5.3 Prosedur pelaksanaan

1. **Pre-check (t=0):** HLS mengalir ≥30 mnt (feeder HLS di `ps` + 0 fallback); catat baseline drift 10 mnt
   SEBELUM perubahan (dengan `aresample` masih aktif) — drift yang selama ini diserap `aresample` belum
   pernah diukur, jadi baseline ini wajib diambil.
2. **Aktivasi A2 langkah 1** pada NO 015 saja (perubahan kode + restart encoder stream tsb — butuh
   persetujuan aktivasi per fase; live lain tidak disentuh).
3. **Sampling:** tiap 30 dtk — `time_sec`/`size=`/`speed=` (ekor log 64 KB) + PID feeder/encoder;
   tiap 5 mnt — load VPS 8 + `reconnect_count` + hitung M6; tiap 10 mnt — hitung drift jendela + mean speed.
   Total 240 sampel/2 jam. Semua pembacaan read-only terhadap log/DB (jangan `tail -f` menahan file).
4. **Pasca-uji:** bandingkan %CPU pra/pasca (metode §1, jendela 15 mnt yang sama) + laporkan M1–M8
   dalam tabel + putuskan lolos/gagal.
5. **Catatan:** log encoder mode append dan tumbuh ~15 MB/45 mnt pada stream lama — pastikan ruang disk
   cukup untuk 2 jam (estimasi +~20 MB) dan ukur dari `time_sec` (bukan ukuran file — pelajaran Fase S:
   file tumbuh meski stream beku).

---

## 6. Model biaya terukur (hitung ulang §4 rencana)

**Input terukur (§1):** mean 0,348 core/stream (encoder+feeder); p50 0,358; p90 0,419.
Overhead supervisor 0,187 core (÷8 = 0,023/stream) → **all-in 0,371 core/stream**.
Kapasitas efektif/box (headroom 30%): **5,6 core**. Harga box VPS 8: **$18,65/bln**; Master $16,02/bln;
cap pemilik **$34,67/bln**. 1 box live murni muat **5,6/0,348 ≈ 16 stream** (all-in ≈ 15; konservatif p90: 13).

### 6.1 As-is (0,348 core/stream terukur)

| Skala | Core encoder+feeder | Core all-in (+supervisor) | Box (÷5,6, dibulatkan ke atas) | $/bln (box live + Master) |
|---|---|---|---|---|
| 50 stream | 17,4 | 18,6 | **4 box** | 4×18,65 + 16,02 = **$90,62** |
| 100 stream | 34,8 | 37,1 | **7 box** | 7×18,65 + 16,02 = **$146,57** |
| 696 stream | 242,2 | 258,4 | **44 box** (47 all-in) | 44×18,65 + 16,02 = **$836,62** (47 box: $892,57) |

Perbandingan dengan §4 rencana (asumsi 0,4–0,5): 50 stream 4–5 box / 100 stream 8–9 box /
696 stream 50–63 box. **Terukur sedikit lebih murah** (44 vs 50–63 box untuk 696) — karena jendela ukur
dalam kondisi fallback/silence (tanpa feeder HLS). Bila HLS aktif, biaya mendekati asumsi rencana
(0,4–0,5). **Rekomendasi: tetap pakai 0,4–0,5 sebagai batas atas perencanaan; 0,348 adalah lantai
kondisi-fallback, bukan angka untuk scale-up.**

### 6.2 Pasca-A (target rencana) — untuk konteks, bukan hasil ukur

| Target | Core/stream | 50 stream | 100 stream | 696 stream |
|---|---|---|---|---|
| A2 lolos (≤0,15) | 0,15 | 7,5 core → 2 box ($53,32) | 15 core → 3 box ($71,97) | 104 core → 19 box ($370,37) |
| Jalur (b) cap (≤0,07) | 0,07 | 3,5 core → **1 box ($34,67, dalam cap)** | 7 core → 2 box ($53,32) | 49 core → 9 box ($183,87) |

Catatan: A2 harus memangkas **~57%** dari 0,348 untuk mencapai 0,15, dan **~80%** untuk 0,07.
Karena encoder = 87% biaya, pemangkasan itu praktis hanya mungkin bila transcode AAC encoder
benar-benar hilang (`-c:a copy`) — feeder saja (13%) tidak cukup.

### 6.3 Batas lain yang tetap berlaku

- Egress: ~0,95–0,97 Mbps/stream terukur dari baris `-stats` (720p; bitrate log 951–968 kbps) →
  50 stream ≈ 48 Mbps; 696 ≈ 660 Mbps (di bawah asumsi rencana ~2 Mbps/stream — angka rencana
  tetap dipakai sebagai batas atas; 1080p lebih tinggi).
- Prep berbagi box: load15 7,65–8,64 saat ini SUDAH termasuk prep — 16 stream/box di atas adalah
  kapasitas live-murni; bila prep tetap di VPS 8, kapasitas live efektif lebih rendah (jalan (c):
  prep pindah ke Master).

---

## Lampiran — bukti & reproduksibilitas

- **Data mentah sampel:** `~/workspace/engineering/fase0_samples_raw.txt` (7 sampel + 1 sampel penutup);
  skrip sampler: `~/workspace/engineering/fase0_sampler.sh` (SSH singkat + retry, baca `/proc` saja).
- **Perintah kunci:** `ps -eo pid,ppid,%cpu,etime,args` (RTMP disensor);
  `sqlite3 "file:/opt/masjavas-os/data/masjavas_tube.sqlite?mode=ro"`;
  `journalctl -u masjavas-live.service --since "2026-11-25 02:43"`;
  `ls -l /proc/<pid>/fd/1` + `fuser /tmp/masjavas_live_audio_*.pipe`;
  `grep -n` pada `worker/live_supervisor.py` (`:114-115, :358, :509, :1256, :1267-1271, :1366, :2176`)
  dan `worker/live_stream_engine.py` (`:90, :2777-2787, :2809, :2941, :3200-3345`).
- **Keterbatasan ukur:** (1) jendela 15,7 mnt, 6 interval — cukup untuk mean/p90, bukan untuk variabilitas
  harian; (2) kondisi fallback/silence, bukan HLS aktif; (3) kontribusi rata-rata burst yt-dlp tidak
  terkuantifikasi (hanya tertangkap sesaat); (4) liveness sumber relay (apakah siaran sumber masih
  live di YouTube) tidak diverifikasi independen — pola gagalnya identik di semua URL (cookie/bot-check),
  tetapi konfirmasi per-URL butuh cek browser; (5) `live_share_pct` governor tercatat 565,0% —
  telemetri itu tetap tidak bisa dipercaya (sesuai diagnosis §3.14).
- **Tidak ada perubahan apa pun** dilakukan pada produksi selama pengukuran ini.

*— Worker M, Fase 0 baseline, 25 Nov 2026 09:30–09:48 WIB.*
