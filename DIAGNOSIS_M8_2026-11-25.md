# DIAGNOSIS M8 — Ke Mana Perginya CPU Feeder Prebuilt + Encoder NO 88

**Worker diagnosis (read-only penuh) — 25 Nov 2026, 14:09–14:45 WIB.**
**Stream:** NO 88 Rock Religi (HAMIDAH, 720p), task 1795502341605, VPS 8.
Feeder prebuilt PID 1957893 · Encoder PID 1957984 · Supervisor PID 1956169 (restart #3, 13:42:43 WIB).
**Tidak ada yang diubah:** tanpa restart, tanpa ubah kode/config/schema, tanpa tulis DB live,
tanpa strace/ganggu PID live, tanpa feeder/encoder baru ke pipe live. Semua ukur baca `/proc`,
`/proc/*/stat|io|status`, `ps`, `ffprobe`, dan micro-benchmark offline di Master (file sintetis).
Semua timestamp WIB (`date`).

## 0. Ringkasan eksekutif

**M8 GAGAL terkonfirmasi, bukan noise sampling.** 30 interval × ~65 dtk selama 33,5 menit
(14:10:03–14:43:34 WIB): **total 0,1746 core/stream (feeder 0,0548 + encoder 0,1199)**,
25/30 interval > 0,15 (min 0,134 · maks 0,209). Angka ini mereplikasi persis observasi A2
(0,174 per 18 mnt) pada jendela yang hampir 2× lebih panjang.

**Temuan utama — biaya BUKAN di kerja data ffmpeg, melainkan di kernel + wakeup di bawah
kondisi host VPS 8:**

| # | Temuan | Bukti |
|---|---|---|
| 1 | **Feeder "copy" murni hampir 0 kerja data.** Tanpa `-re` (kecepatan penuh), demux+mux+copy 4,33 jam audio = **0,00001–0,0002 core/detik-realtime**. Seluruh 0,0548 live adalah overhead pacing/wakeup, bukan pemindahan data (hanya 16 KB/dtk). | F2/Hb2 Master (rusage) |
| 2 | **Feeder live 0,0548 vs 0,006–0,008 di Master** — topologi, file, dan laju syscall IDENTIK (43 tulis/dtk, 43–50 wakeup/dtk). Biaya per wakeup ~7–8× lebih mahal di VPS 8. | F1/F1c/B4/Hb vs live |
| 3 | **Encoder live 0,1199 vs 0,0100 di Master** (demux+FLV-mux+TCP-localhost). `-stats` terukur **0,000**. Sisa ~0,11 belum terisolasi antara RTMP-internet vs mesin/host. | B1/B2/B3/E2c vs live |
| 4 | **Kedua proses didominasi kernel:** feeder 70,5% stime, encoder 74,1% stime. Encoder PCM (transcode) justru kebalikannya (52–66% utime). Pola ini = biaya syscall/penjadwalan/TCP, bukan komputasi ffmpeg. | /proc delta utime/stime |
| 5 | **VPS 8 hanya menghantar ~4,3–5,0 core/detik, bukan 8,0.** Tiap vCPU maju 55–62 ticks/dtk (nominal 100), seragam di 8 vCPU, sejak boot 494/800 = 0,62. `steal=0` selalu — shortfall TIDAK terlihat dari dalam guest (`CONFIG_PARAVIRT_TIME_ACCOUNTING` mati). | /proc/stat per-CPU + uptime |
| 6 | **Feeder + encoder bermigrasi antar-vCPU di hampir tiap wakeup** (feeder 26/30 sampel, encoder 22/30 — seluruh 8 vCPU), sementara feeder identik di Master **0 migrasi/30 sampel** (menetap di CPU 0). Beban userspace murni (4× hog di Master) TIDAK menaikkan biaya feeder (0,0052 vs 0,0057). | Probe migrasi 60 dtk |
| 7 | **Ukuran file prebuilt TIDAK berpengaruh.** Feeder file 4,33 jam / 252 MB = 0,0070 vs file 300 dtk = 0,0057–0,0077 di Master. Hipotesis tabel-sample MP4 GUGUR. | Hb vs F1c/B4 |

**Implikasi keputusan (bukan keputusan — lihat §7):** lantai feeder copy di VPS 8 adalah
~0,05 dan encoder AAC-copy ~0,12, sehingga anggaran realistis per stream adalah
**~0,17–0,18, bukan 0,15**. Dan matematika rollout harus memakai **kapasitas efektif box
~3,0–3,5 core** (70% dari 4,3–5,0 yang terhantar), bukan 5,6 core (70% dari 8 nominal) —
ini ikut mematahkan hitungan "0,07 → 1 box" (§8).

---

## 1. Data live — sampling /proc 33,5 menit (31 sampel, 30 interval)

Sampler: `/tmp/m8_sampler.sh` di VPS 8 (baca `/proc` tiap ~60 dtk + `ps`).
Data mentah: `~/workspace/engineering/m8_samples_final.log`.
Analisis: `~/workspace/engineering/m8_analyse.py`.

### 1.1 Semua proses live (mean delta-ticks / interval)

| PID | Peran | Core mean | Min–maks | utime | stime | vol/s | syscr/s | syscw/s | rchar | wchar |
|---|---|---|---|---|---|---|---|---|---|---|
| 1957893 | **FEED 088 ADTS-copy** | **0,0548** | 0,038–0,084 | 29,5% | 70,5% | 50,5 | 0,5 | 43,1 | 15,7 KB/dtk | 16,0 KB/dtk |
| 1957984 | **ENC 088 AAC-copy** | **0,1199** | 0,093–0,153 | 25,9% | 74,1% | 33,8 | 38,3 | 1,8 | 144,3 KB/dtk | 0,1 KB/dtk |
| 1956650 | FEED concat PCM | 0,0806 | 0,053–0,113 | 47,9% | 52,1% | 110,8 | 1,0 | 93,7 | 31,7 | 172,2 |
| 1957266 | FEED concat PCM | 0,0767 | 0,054–0,104 | 48,0% | 52,0% | 110,4 | 1,0 | 93,7 | 31,7 | 172,2 |
| 1958500 | FEED concat PCM | 0,0416 | 0,026–0,059 | 51,2% | 48,8% | 56,6 | 0,9 | 38,3 | 29,2 | 172,3 |
| 1974444 | FEED prebuilt PCM | 0,0893 | 0,057–0,123 | 40,8% | 59,2% | 83,9 | 0,5 | 43,1 | 15,8 | 172,2 |
| 1978248 | FEED prebuilt PCM | 0,0909 | 0,064–0,115 | 40,6% | 59,4% | 84,9 | 0,5 | 43,0 | 15,9 | 171,9 |
| 1956675 | ENC PCM (706) | 0,2783 | 0,228–0,339 | 65,7% | 34,3% | 42,6 | 5,7 | 1,8 | 244,8 | 0,1 |
| 1957083 | ENC PCM (114) | 0,3687 | 0,299–0,435 | 48,8% | 51,2% | 47,5 | 46,7 | 1,8 | 698,1 | 0,1 |
| 1957292 | ENC PCM (707) | 0,2744 | 0,208–0,322 | 65,0% | 35,0% | 41,8 | 5,2 | 1,8 | 263,7 | 0,1 |
| 1957516 | ENC PCM (034) | 0,3182 | 0,267–0,372 | 54,9% | 45,1% | 47,6 | 35,6 | 1,8 | 458,3 | 0,1 |
| 1958534 | ENC PCM (015) | 0,3137 | 0,268–0,388 | 56,4% | 43,6% | 44,7 | 12,4 | 1,8 | 447,7 | 0,1 |
| 1957353 | ENC single m4a | 0,2646 | 0,216–0,313 | 64,0% | 36,0% | 44,3 | 1,2 | 1,8 | 107,0 | 0,1 |
| 1958617 | ENC single wav (698) | 0,3539 | 0,295–0,439 | 55,1% | 44,9% | 47,8 | 16,0 | 1,8 | 530,3 | 0,1 |
| 1956677 | BUILD prebuilt (706) | 0,4790 | 0,422–0,543 | 75,8% | 24,2% | 384,2 | 4,0 | 2,1 | 129,5 | 64,0 |
| 1957293 | BUILD prebuilt (707) | 0,4757 | 0,436–0,538 | 75,7% | 24,3% | 377,8 | 3,9 | 2,1 | 127,3 | 62,9 |
| 1956169 | SUPERVISOR | 0,1810 | 0,092–0,276 | 70,0% | 30,0% | 24,7 | — | — | — | — |

Keterangan:
- **Encoder 088 (0,1199) adalah encoder termurah di box** — 2,2–3,1× lebih murah dari
  encoder PCM (0,26–0,37). Penghematan A2 (AAC-copy) nyata dan terbesar di antara semua
  komponen; masalahnya hanya ia tidak cukup menutup anggaran bila feeder menyumbang 0,055.
- **syscw encoder = 1,8/dtk di semua encoder** = baris `-stats` ke stderr (wchar 0,1 KB/dtk
  ≈ 71 byte/tulis). RTMP tidak lewat `write()` (pakai `sendmsg`, tak tercatat di `/proc/io`).
- **Builder prebuilt (2× ~0,48) = 0,95 core beban transien** sepanjang jendela — pekerjaan
  satu-kali (concat→AAC 128k), bukan beban live steady-state, tapi ikut menjenuhi box.
- **Total box (ffmpeg live + supervisor): 3,86 core** (encoder 2,29 + feeder 0,43 +
  builder 0,95 + supervisor 0,18). Σ seluruh proses (uji 30 dtk terpisah): **4,38 core/dtk**.
- Feeder concat bervariasi 0,042–0,081 antar-stream (ikut pola tulis: 38 vs 94 tulis/dtk —
  struktur playlist/file memengaruhi chunking tulis ffmpeg, bukan laju data).
- PID 1957059 (feeder concat NO 114) hanya muncul di sampel 0 — **bukan mati**: NO 114
  beralih concat→prebuilt-PCM (feeder 1978248, mulai ~14:05 WIB) oleh supervisor; pergantian
  sumber normal, encoder 1957083 hidup terus. (Lihat §9.)

### 1.2 NO 88 per interval (30 interval, ~65 dtk)

Mean total **0,1746**; 25/30 interval > 0,15; min 0,134; maks 0,209.
Feeder 0,038–0,084; encoder 0,093–0,153. **Tidak ada korelasi dengan load1**
(contoh: load1 7,02 → total 0,167; load1 12,54 → 0,186; load1 10,20 → 0,147 —
variasi interval tidak mengikuti beban box). Variabilitas interval ini nyata tapi
belum terjelaskan (bukan beban, bukan fase loop 60 dtk yang terbukti — lihat §9).

### 1.3 File sumber 088 (baca saja)

- Prebuilt: `prebuilt_1795502341605.m4a`, 254.257.210 byte, durasi 15.546 dtk (4,32 jam),
  AAC 128k stereo 44,1 kHz. Laju baca feeder 15,7 KB/dtk = persis bitrate audio.
- Video: `gdrive_video_166j_MUeXP.mp4`, 7.916.939 byte, 60,0 dtk, h264 **1920×1080** 30 fps
  + AAC, 1.055.591 bps. (Catatan: berkas video 1080p meski stream dilabeli 720p — output
  `-c:v copy` mempertahankan 1080p; bitrate stream terukur 1054–1056 kbps.)
- FIFO: `/tmp/masjavas_live_audio_1795502341605.pipe`, `pipe-max-size` 1 MB (default 64 KB).
- MemAvailable VPS 8: **22,6 GB** — tekanan page-cache BUKAN penjelasan (file 254 MB
  muat penuh di cache).

---

## 2. Profil feeder 0,0548 — pemisahan hipotesis

Feeder live: `-nostdin -loglevel error -re -stream_loop -1 -i prebuilt_….m4a -vn -c:a copy
-f adts pipe:1`. Kerja per detik: baca 15,7 KB + tulis 16,0 KB, 43,1 tulis/dtk
(frame AAC 1024 sampel @44,1 kHz), 50,5 voluntary switch/dtk, 0,5 baca/dtk
(bacaan besar ~28 KB — buffered).

### 2.1 Micro-benchmark Master (file sintetis, offline, BUKAN VPS 8)

Master: AMD EPYC 3195 MHz, load 1,1–1,6/4 vCPU. File: `synth.m4a` 300 dtk AAC 128k
(dibuat dari sine 440 Hz) + `long.m4a` 4,33 jam / 252,3 MB (synth ×52, copy).
Skrip: `~/workspace/engineering/m8bench.py` (v1, cacat topologi — lihat catatan),
`m8bench2.py` (v2), `m8bench3.py` (v3, final). Hasil valid:

| Uji | Varian | Core | utime | stime | Keterangan |
|---|---|---|---|---|---|
| F1 / F1c / B4 | ADTS-copy `-re` → FIFO di-drain (topologi real) | 0,0057 / 0,0057 / 0,0077 | 0,0027–0,0037 | 0,0030–0,0040 | 43,1 tulis/dtk, 43,3–43,5 vol/dtk — **identik dengan live** kecuali biaya |
| F2 | ADTS-copy TANPA `-re` → FIFO (300 dtk audio, 0,07 dtk wall) | **0,0002**/dtk-realtime | — | — | Kerja data murni ≈ nol |
| Hb2 | LONG 4,33 jam TANPA `-re` → FIFO (0,10 dtk wall) | **0,00001**/dtk-realtime | — | — | Kerja data murni ≈ nol, ukuran file tak berpengaruh |
| F4 | ADTS-copy `-re` → null muxer (tanpa tulis output) | 0,0063 | — | — | ≈ F1 — tulis output bukan biaya utama |
| F5 | PCM s16le `-re` → FIFO (decode, varian feeder 1974444) | 0,0098 | 0,0067 | 0,0032 | Decode +~0,004 vs copy |
| F6 | Silence `anullsrc` s16le `-re` → FIFO | 0,0098 | 0,0058 | 0,0040 | ≈ F5 — silence tidak lebih murah dari decode |
| Hb | ADTS-copy `-re` file LONG 4,33 jam → FIFO | 0,0070 | 0,0033 | 0,0037 | ≈ F1 — **ukuran file/tabel-sample MP4 tidak berpengaruh** |
| F1-load | ADTS-copy `-re` → FIFO + 4× `yes` hog (load1 3,83/4 core) | 0,0052 | 0,0030 | 0,0022 | **Beban userspace TIDAK menaikkan biaya**; vol/dtk naik 43→61 |

Hasil INVALID (tidak dipakai, penyebab terdokumentasi): F3/E1/E3/E4 v1+v2 —
ffmpeg 6.1.1 menolak output `/dev/null` tanpa `-y` ("File '/dev/null' already exists").
v3 memperbaikinya (`-y` + file biasa) untuk varian encoder (B1/B3). F1 v1 sempat salah
topologi (stdout ke DEVNULL, bukan ke FIFO) — v2/v3 memperbaikinya dengan redirect bash
dan mereplikasi angka (0,0057) — jadi angka F1 berdiri.

### 2.2 Keputusan hipotesis feeder

| Hipotesis | Putusan | Alasan |
|---|---|---|
| Biaya `-re` pacing (wakeup/tick per frame) | **TERBUKTI DOMINAN** | Tanpa `-re`: 0,0002. Dengan `-re`: 0,006 (Master) / 0,055 (live). Seluruh biaya muncul hanya saat pacing aktif; laju wakeup 43/dtk sama di kedua box. |
| Mux ADTS | Kecil | F4 (null muxer, tanpa tulis) = 0,0063 ≈ F1 0,0057. |
| Tulis pipe FIFO | Kecil di Master, tak terisolasi di live | F1→FIFO 0,0057 vs F4→null 0,0063 — tulis pipe ≈ 0 di Master. Di live, tulis pipe + wakeup tidak terpisah (satu siklus). |
| Baca file / ukuran file / tabel MP4 | **GUGUR** | Hb (252 MB / 4,33 jam) = 0,0070 ≈ F1; Hb2 tanpa `-re` = 0,00001. |
| Beban CPU box (kontensi userspace) | **GUGUR sebagai penjelasan utama** | F1-load (hog 4×, load ~4/4 core) = 0,0052, tidak naik. |
| Migrasi vCPU + throttling host | **KANDIDAT UTAMA (korelasi kuat, belum kausalitas)** | Live: 26 migrasi/30 sampel; Master: 0/30. vCPU terhantar ~58% (§6). Biaya per wakeup live ~1,1 md/dtk-siklus vs ~0,13–0,18 md di Master (7–8×). |
| Page-cache dingin / tekanan memori | GUGUR | MemAvailable 22,6 GB; baca 0,5/dtk. |

**Biaya per siklus wakeup (perkiraan):** live 5,48 ticks/dtk ÷ 50,5 switch/dtk ≈
0,11 tick ≈ **1,1 ms CPU per siklus**; Master 0,57–0,77 ticks/dtk ÷ 43,3 ≈
0,013–0,018 tick ≈ **0,13–0,18 ms per siklus**. Angka live (1,1 ms untuk satu
`write()` 380 byte + `nanosleep`) hanya masuk akal bila tiap wakeup menanggung
biaya kernel/penjadwalan yang besar (pindah vCPU + cache/TLB dingin + antrean
penjadwalan di host yang menjatah ~58% CPU).

---

## 3. Profil encoder 0,1199 — pemisahan hipotesis

Encoder live: `-nostdin -stats -loglevel warning -re -stream_loop -1 -i video.mp4
-f aac -thread_queue_size 4096 -i pipe -map 0:v:0 -map 1:a:0 -c:v copy -c:a copy
-fflags +genpts -rw_timeout 60000000 -rtmp_buffer 8000 -rtmp_live live
-tcp_nodelay 1 -send_buffer_size 2097152 -max_muxing_queue_size 4096
-flvflags no_duration_filesize -f flv rtmp://…`. Kerja per detik: baca 144 KB/dtk
(video ~106 + audio ~16 + overhead), 38,3 baca/dtk, 1,8 tulis/dtk (baris `-stats`),
33,8 voluntary switch/dtk. **utime hanya 25,9% — stime 74,1%.**

### 3.1 Micro-benchmark Master (encoder klon, output lokal — BUKAN RTMP internet)

| Uji | Varian | Core | utime | stime | Keterangan |
|---|---|---|---|---|---|
| E2c | → null muxer (demux saja, tanpa FLV, tanpa `-stats`) | 0,0067 | 0,0048 | 0,0018 | Demux video+audio FIFO, copy |
| B1 | → FLV FILE (mux, tanpa jaringan, tanpa `-stats`) | 0,0073 | 0,0048 | 0,0025 | +FLV mux ≈ +0,0006 |
| B3 | → FLV FILE + `-stats` | 0,0072 | 0,0042 | 0,0030 | **`-stats` = 0,000** (B1→B3 tidak naik) |
| B2 | → FLV via TCP localhost (mux + socket, tanpa RTMP) | 0,0100 | 0,0032 | 0,0068 | +TCP localhost ≈ +0,003 total (stime 0,0025→0,0068) |
| E3c | Video-only → FLV file | INVALID | — | — | Gagal `-y` (v2); tidak diulang — B1 sudah mencakup video+audio |

Dekomposisi Master: demux 0,0067 → +mux FLV 0,0073 → +TCP-localhost 0,0100.
**Live 0,1199 adalah 12× B2.** Sisa ~0,11 belum terisolasi.

### 3.2 Keputusan hipotesis encoder

| Hipotesis | Putusan | Alasan |
|---|---|---|
| Demux mp4 (video 1080p30 + audio FIFO) | Kecil (0,0067 Master) | E2c. Laju baca live 38,3/dtk ≈ bench 45,8/dtk — perilaku sama. |
| Baca FIFO audio | Kecil | Termasuk dalam E2c (0,0067 total). |
| Mux FLV | Kecil (~0,0006) | B1 − E2c. |
| Kirim RTMP + TCP internet | **BELUM TERISOLASI — kandidat terbesar yang tersisa** | B2 (TCP localhost) hanya +0,003. Yang belum diukur: chunking RTMP (128-byte, ~1000 chunk/dtk, userspace — perkiraan kasar 0,001–0,002) + TCP internet nyata (RTT, ACK, congestion, `-tcp_nodelay`) + interaksi keduanya. Perlu uji diskriminan §7 Opsi 5. |
| `-stats` | **GUGUR (terukur 0)** | B1 0,0073 vs B3 0,0072. Pertahankan — ini sumber observasi M2/M3. |
| `thread_queue_size 4096` | **BELUM TERUJI** | E4c gagal karena bug `-y` dan tidak sempat diulang. Perkiraan ~0 (hanya kedalaman antrean), tapi ia mengubah pola wakeup thread demux audio — murah diuji di Master (1×60 dtk). |
| Kompleksitas adegan video loop | **TIDAK TERBUKTI** | `-c:v copy` tidak decode; biaya demux ikut jumlah/ukuran paket, bukan isi adegan. Interval 66 dtk ≈ 1,1 loop (fase loop berbeda tiap interval) tapi variasi interval tidak berpola fase; A2 sudah mencatat tren naik yang sama tanpa penjelasan konten. |
| Migrasi vCPU + throttling host | **KANDIDAT UTAMA (korelasi kuat)** | Encoder live: 22 migrasi/30 sampel, 3,6 ms CPU per wakeup (0,1199÷33,8) vs Master B2 0,33 ms (0,0100÷30,5) — 11×. |

Catatan penting: **encoder PCM (0,26–0,37) justru utime-dominan (52–66%)** — transcode
AAC memang kerja userspace. Encoder AAC-copy membalik profil menjadi stime-dominan
(74%) — setelah transcode dihapus, yang tersisa adalah biaya kernel (baca/pipe/TCP/
penjadwalan). Inilah kenapa "hapus transcode" tidak membawa total ke ~0,02 seperti
yang diasumsikan rencana A3: asumsi itu mengabaikan lantai kernel.

---

## 4. Rekonsiliasi metode — satu kerangka untuk semua angka

Semua angka di bawah memakai **metode yang sama: delta-ticks `/proc/<pid>/stat`
(utime+stime) per interval**. Yang berbeda adalah konfigurasi, bukan metode.

| Pengukuran | Konfigurasi | Feeder | Encoder | Total/stream | Metode |
|---|---|---|---|---|---|
| Fase 0 (25 Nov 09:30–09:48) — mean 8 stream | Encoder PCM (`-c:a aac 192k` + `aresample`), feeder concat/silence | 0,04–0,09 | 0,20–0,40 (mean ~0,30) | **0,348** (p50 0,358 · p90 0,419) | Delta-ticks, 7 sampel/6 interval |
| Fase 0 — NO 88 saja | Encoder PCM 0,198 + silence 0,042 | 0,042 | 0,198 | **0,241** | Delta-ticks |
| Obs 1 — NO 88 | Encoder AAC-copy 0,117 + feeder concat 0,099 | 0,099 | 0,117 | 0,216 | Delta-ticks |
| Obs A2 — NO 88 (18 mnt) | Encoder AAC-copy + feeder prebuilt ADTS | 0,054 | 0,120 | **0,174** | Delta-ticks |
| **Diagnosis ini — NO 88 (33,5 mnt, 30 interval)** | Sama dengan A2 | **0,0548** | **0,1199** | **0,1746** | Delta-ticks |
| `ps %cpu` A2 (lifetime) | Sama dengan A2 | 0,030→0,050 | 0,071→0,111 | 0,101→0,161 | Rata-rata seumur hidup proses |

Penjelasan per baris yang tampak bertentangan:

1. **0,348 vs 0,174 — bukan kontradiksi metode, melainkan konfigurasi + cakupan.**
   0,348 adalah mean 8 stream era PCM (encoder transcode ~0,30). 0,174 adalah SATU
   stream termurah (088) era AAC-copy. Perbandingan apel-ke-apel untuk 088:
   0,241 (Fase 0) → 0,174 (A2) = **−0,067**, terjelaskan penuh: encoder −0,078
   (0,198→0,120) + feeder +0,012 (silence 0,042→prebuilt 0,054). Estimasi mean
   armada era AAC-copy (bila semua 8 stream pindah): encoder ~0,12 + feeder
   ~0,05–0,09 → **~0,18–0,20/stream**, bukan 0,174 (088 adalah kasus termurah).
2. **Feeder concat 0,099 (obs 1) vs 0,042–0,081 (diagnosis ini).** Feeder concat
   bervariasi antar-stream/playlist (38 vs 94 tulis/dtk). Obs 1 mengukur satu
   konfigurasi playlist; rentang diagnosis ini (3 feeder concat: 0,042/0,077/0,081)
   mencakupnya. Bukan kontradiksi — variabilitas antar-sumber audio.
3. **`ps %cpu` lifetime 0,101→0,161 vs delta-ticks 0,174.** `ps` adalah rata-rata
   kumulatif sejak proses lahir, termasuk ~3,6 menit pertama yang berjalan di
   ~0,10 (ramp-up: RTMP connect + isi buffer + baca awal dari cache dingin).
   Rata-rata kumulatif selalu TERTINGGAL dari laju steady-state bila laju awal
   berbeda — di A2 ia naik monoton 0,101→0,161 mendekati 0,174 tapi belum sampai
   setelah 22 menit. Di Fase 0 arahnya terbalik (`ps` sedikit LEBIH TINGGI dari
   delta) karena periode awal pasca-restart di sana justru mahal (probe relay/
   retry). **Kesimpulan: `ps %cpu` bukan estimator steady-state; penilaian M8
   harus memakai delta-ticks jendela stabil — seperti yang sudah diputuskan A2.**
   Angka "benar" tiap komponen (steady-state, NO 88, AAC-copy): **feeder 0,055,
   encoder 0,120, total 0,175**.

---

## 5. Temuan box — kapasitas efektif VPS 8 (di luar dugaan M8, memengaruhi semua hitungan)

1. **Tiap vCPU hanya maju 55–62 ticks/dtk (nominal 100), seragam di 8 vCPU**
   (uji 20 dtk: total agregat 470/dtk; uji 30 dtk: 426/dtk; jendela 31 mnt: 445/dtk;
   sejak boot 47,3 jam: 494/dtk = 0,62 dari nominal 800). **Rasio penghantaran
   ~0,53–0,62.**
2. **`steal = 0` selalu; `CONFIG_PARAVIRT_TIME_ACCOUNTING` tidak diset** (kernel
   6.8.0-151-generic, `CONFIG_HZ=1000`, `CONFIG_NO_HZ_FULL=y`). Shortfall host
   TIDAK TERLIHAT dari dalam guest — tidak ada metrik yang menunjukkannya kecuali
   laju ticks itu sendiri.
3. **Σ semua proses (utime+stime) = 437,7 ticks/dtk ≈ total agregat /proc/stat
   426/dtk** — kedua akuntansi sepakat soal TOTAL yang terhantar; yang menyesatkan
   adalah pembagian `idle 50%` di /proc/stat (itu sebagian besar adalah waktu
   vCPU yang tidak pernah dijadwalkan host, bukan kapasitas menganggur — buktinya
   load 7–14 dan antrean runnable).
4. **Konsekuensi:** box ini secara efektif adalah **box ~4,3–5,0 core, bukan 8 core**.
   Total permintaan terukur saat ini (live 2,73 + supervisor 0,18 + builder 0,95 +
   lain-lain) ≈ 3,9–4,4 core/dtk — box sudah jenuh pada kapasitas YANG TERHANTAR.
   Ini menjelaskan load 7–14 yang tampak "tidak masuk akal" bila dibaca sebagai
   box 8 core setengah-menganggur.
5. **Migrasi:** feeder 26/30, encoder 22/30 perubahan vCPU per 2 dtk (seluruh 8 vCPU);
   feeder identik di Master 0/30 (menetap di CPU 0). Jam CPU Master 3195 MHz vs
   VPS 8 2795 MHz (1,14×) — kecepatan clock BUKAN penjelasan (butuh 10×).

Status temuan ini: **terukur kuat di jendela ini, tapi rasio penghantaran bisa
berubah menurut waktu/host (fair-share)**. Sebelum dipakai sebagai konstanta
perencanaan final, verifikasi independen disarankan (grafik CPU Contabo / uji
beban terkontrol di box kosong — BUKAN di VPS 8 live).

---

## 6. Opsi jalan keluar (bernilai angka, tanpa implementasi — keputusan di parent/user)

### (a) Bentuk feeder yang lebih murah — yang masuk akal diuji

| # | Opsi | Harapan (angka) | Prasyarat uji | Risiko |
|---|---|---|---|---|
| 1 | **Hapus feeder: encoder baca file prebuilt langsung** (single-process, `-c:a copy`, tanpa FIFO — seperti 645/698 yang sudah single-process, tapi dengan copy bukan transcode) | **−0,055/stream (−31%): 0,175 → ~0,12** bila biaya encoder tidak naik. Paling besar dan paling konkret. | Cabang supervisor baru + uji A/B 1 stream (M1–M8, 2 jam). | Rendah–sedang. Menghapus 1 proses + 1 FIFO + ~43 wakeup/dtk per stream. Satu-satunya opsi yang menyerang struktur biaya, bukan meredamnya. |
| 2 | **CPU affinity (`taskset`) per pasangan feeder+encoder ke 1 vCPU** | Tak diketahui; bila migrasi adalah penyebab, bisa mendekati biaya Master (feeder → ~0,01–0,02?). Bila bukan, ~0. | **Persetujuan parent** (intervensi PID live, reversibel instan via `taskset -p`). Uji A/B 30 mnt di 088 vs baseline §1. | Rendah. Reversibel. Perhatian: pinning di vCPU yang "sial" tidak mengurangi total CPU terhantar — hanya pola cache. |
| 3 | **Feeder tanpa `-re`** (encoder yang memacing via `thread_queue_size` + backpressure pipe) | Tak diketahui; wakeup/dtk bisa turun (tulis burst lalu tidur panjang) — atau tidak berubah. | Uji di Master dulu (feeder burst + encoder klon, ukur + cek drift), baru A/B live bila menjanjikan. | **Sedang.** Mengubah perilaku drift/sync (M1 diuji DENGAN `-re`); burst bisa menaikkan latensi antrean. |
| 4 | **Buffer pipe lebih besar (`F_SETPIPE_SZ` → 1 MB)** | Kecil / tak pasti. Feeder `-re` tidak dibatasi blokir pipe (ditentukan timestamp), jadi buffer besar tidak mengubah laju wakeup. | Perubahan supervisor + restart encoder. | Rendah, tapi harapan kecil — jangan jadikan andalan. |
| 5 | **Chunked read / frame lebih besar** | ~0. Frame AAC 1024 sampel adalah tetap; tidak ada parameter yang mengubahnya tanpa re-encode (yang justru menambah biaya). | — | — (ditolak sebagai opsi) |
| 6 | **`-stats` / `thread_queue_size`** | `-stats`: **0,000 terukur — jangan diutak-atik.** `thread_queue_size`: belum teruji; perkiraan ~0; uji murah 1×60 dtk di Master. | Uji Master. | ~Nol. |

### (b) Bila lantai memang feeder ~0,05 + encoder ~0,12 — anggaran realistis & matematika rollout

Asumsi eksplisit: per stream (AAC-copy) **0,175**; supervisor **0,18/box** (relatif tetap);
kapasitas terpakai **70% dari yang terhantar (4,3–5,0) = 3,0–3,5 core/box** (bukan 5,6).

| Skala | Core feeder+encoder | All-in (+supervisor) | Box (÷3,0–3,5) | $/bln (box live + Master $16,02) |
|---|---|---|---|---|
| **16–19 stream (1 box, dalam cap)** | 2,8–3,3 | 3,0–3,5 | **1 box** | **$34,67 (cap)** |
| 50 stream | 8,75 | 9,3 | **3 box** (ketat) / 4 nyaman | **$71,97** / $90,62 |
| 100 stream | 17,5 | 18,6 | **6 box** | $127,92 |
| 696 stream | 121,8 | 129,4 | **35–41 box** | **$668–$781** |

Perbandingan dengan rencana lama (Fase 0 §6, asumsi 5,6 core/box): 50 stream 2 box
($53,32) pada 0,15 — **tidak berlaku lagi**; dengan kapasitas efektif, bahkan pada
**0,15 pun 50 stream butuh 3 box ($71,97)**, dan pada **0,07 pun butuh 2 box ($53,32)**
(3,5 + 0,18 = 3,68 > 3,0–3,5 yang terukur). **Cap $34,67 = 1 box + Master =
maksimal ~16–19 stream pada efisiensi sekarang.** Target 50 stream di dalam cap
membutuhkan BOTH: per-stream ≤ ~0,07 DAN kapasitas efektif ≥ ~3,7 core/box —
keduanya belum terbukti.

### (c) Komponen lain yang bisa dipangkas — hipotesis terukur saja, bukan rekomendasi eksekusi

- **`-stats`: TERBUKTI 0 — pertahankan** (sumber M2/M3). Menghapusnya menghemat 0.
- **`thread_queue_size 4096`: belum teruji** — hipotesis: memengaruhi pola wakeup
  thread demux audio; uji A/B Master 2×60 dtk (dengan vs tanpa) sebelum menyebut angka.
- **Builder prebuilt (0,95 core, 2 proses):** beban transien, bukan per-stream
  steady-state — tapi selama builder berjalan ia memakan ~22% kapasitas terhantar box.
  Penjadwalan builder di luar jam live / throttle builder (`nice`/`cpulimit`) adalah
  pengungkit kapasitas box, BUKAN pengungkit per-stream. Perlu keputusan terpisah
  (builder lambat = prebuilt baru lambat tersedia).
- **Supervisor 0,18 core:** ~0,023/stream pada 8 stream; mengecil per stream bila
  stream/box naik. Bukan target utama.
- **Pindah host / vCPU dedicated:** hanya masuk akal bila uji §7-Opsi-5 membuktikan
  mesin/host sebagai penyebab dominan. Jangan diputuskan dari korelasi migrasi saja.

**Uji diskriminan yang paling dibutuhkan (butuh persetujuan parent — proses baru di
VPS 8 jenuh, 60 dtk, ±0,01–0,12 core, TIDAK menyentuh pipe live):** klon encoder 088
di VPS 8 dengan output FLV ke file lokal `/tmp` (baca video + FIFO uji sendiri, bukan
pipe live, tanpa RTMP). Bila klon itu ≈ 0,01 → penyebabnya RTMP-internet. Bila ≈ 0,12
→ penyebabnya mesin/host VPS 8. Hasilnya menentukan apakah Opsi 1/2 (arsitektur/
affinity) atau Opsi 6 (pindah host) yang dikerjakan.

---

## 7. Apa yang TIDAK boleh disimpulkan dulu

1. **Jangan simpulkan anggaran 0,15 bisa dicapai dengan "optimasi feeder".**
   Kerja data feeder ≈ 0; lantainya (~0,05) adalah overhead wakeup di bawah kondisi
   host — tidak ada parameter ffmpeg yang memangkasnya kecuali menghapus prosesnya
   (Opsi 1) atau mengubah kondisi host/penjadwalan (Opsi 2/6).
2. **Jangan simpulkan migrasi vCPU PASTI penyebabnya.** Korelasi kuat (26/30 vs 0/30;
   hog-test negatif), tapi kausalitas butuh uji affinity (Opsi 2). Bisa juga ada
   faktor host lain yang berkorelasi (mis. penjatahan fair-share itu sendiri).
3. **Jangan simpulkan RTMP-internet adalah/ bukan biaya encoder.** Komponen itu
   belum terisolasi (localhost-TCP hanya +0,003; RTMP-chunking + internet belum
   diukur). Butuh uji diskriminan §6.
4. **Jangan simpulkan VPS 8 muat 50 stream / 16 stream dari angka nominal.**
   Kapasitas efektif terukur 4,3–5,0 core (rasio 0,53–0,62, bisa berubah menurut
   waktu/host). Hitungan §6 memakai jendela ini; verifikasi independen sebelum
   dijadikan konstanta.
5. **Jangan simpulkan dari `ps %cpu` bahwa M8 "hampir lolos" (0,161).**
   Itu rata-rata kumulatif yang tertinggal; laju steady-state 0,175 (25/30
   interval > 0,15).
6. **Jangan simpulkan variabilitas interval (0,134–0,209) sudah dijelaskan.**
   Bukan beban box, bukan fase loop yang terbukti. Penyebab variasi per interval
   masih terbuka.
7. **Jangan simpulkan `-stats`/`thread_queue_size`/buffer-pipe sebagai pengungkit.**
   `-stats` terukur 0; dua lainnya belum teruji / harapan kecil.
8. **Keputusan revisi anggaran 0,15 → ~0,17–0,18 adalah KEPUTUSAN parent/user,**
   bukan diagnosis ini. Laporan ini menyajikan opsi bernomor (§6) + konsekuensi
   biayanya; tidak memutuskan.

---

## 8. Lampiran A — Audit `pkill` (tindak lanjut guardrail parent 14:24 WIB)

Aturan: `pkill` HANYA untuk proses benchmark milik sendiri di Master, pola sempit,
tidak mungkin mengenai proses produksi. **Tidak ada proses produksi yang terbunuh.**
Bukti: ffmpeg produksi Master PID 3996087 (job mp3 stem, terlihat hidup 14:21 WIB
SETELAH semua `pkill` bench1 berjalan) lalu pipeline berlanjut normal ke PID 3997965
(master assembly, 14:25 WIB); service `masjavas-os-web`, `masjavas-os-worker`,
`masjavas-live` (supervisor Master PID 3568736) aktif/running sepanjang waktu.

### A.1 Pola lengkap yang dipakai

| # | Perintah (persis) | Di mana / kapan | Sasaran yang dimaksud |
|---|---|---|---|
| 1 | `pkill -f "python3 m8bench.py"` | Master, via ssh ~14:22 (dalam perintah yang GAGAL konek — lihat A.2) | Skrip bench1 milik sendiri |
| 2 | `pkill -f "cat /tmp/m8bench"` | Master, sama | Drainer `cat` FIFO milik sendiri |
| 3 | `pkill -f "ffmpeg -nostdin -loglevel error -re -stream_loop -1 -i /tmp/m8bench/synth"` | Master, sama + di dalam `m8bench.py` (cleanup E1/E2/E4, ~14:19–14:21) | Feeder bench milik sendiri (awalan ini juga mencakup `synthvid.mp4`) |
| 4 | `pkill -f "ffmpeg -nostdin -stats -loglevel warning -re -stream_loop -1 -i /tmp/m8bench/synthvid"` | Master, sama + di dalam `m8bench.py` (cleanup E1) | Encoder bench E1 milik sendiri |
| 5 | `pkill -f "ffmpeg -nostdin -loglevel error -re -stream_loop -1 -i /tmp/m8bench/synthvid"` | Master, sama + di dalam `m8bench.py` (cleanup E2/E4) | Encoder bench E2/E4 milik sendiri |
| — | (bench2/bench3: TIDAK ADA `pkill` sama sekali) | Master | Cleanup hanya via `killpg` pada process-group bash yang diluncurkan skrip itu sendiri |

Semua pola mengandung literal `/tmp/m8bench/synth…` / `m8bench.py` / `cat /tmp/m8bench`
— proses produksi bekerja di `/opt/masjavas…` / `/opt/masjavas-os…` dan tidak
mungkin cocok. Tidak pernah ada `pkill ffmpeg` pola luas.

### A.2 Apa yang benar-benar dieksekusi vs tidak

- **Dua percobaan ssh manual (~14:22–14:23) yang memuat `pkill` TIDAK PERNAH
  dieksekusi**: SSH gagal di konek (exit 255 dalam 2–5 dtk, output kosong) sebelum
  perintah remote berjalan. File `m8bench2.py` sempat terkirim via `scp` (14:22),
  tapi `pkill`+peluncuran di perintah yang sama tidak jalan; bench2 akhirnya
  diluncurkan dari perintah terpisah (14:26:21 WIB, tanpa `pkill`).
- **`pkill` di dalam `m8bench.py` (bench1) BENAR-BENAR dieksekusi** (~14:19–14:21,
  cleanup E1/E2/E4): menargetkan feeder/encoder bench yang menjadi orphan setelah
  `bash` pembungkusnya di-terminate (maks. 6 proses ffmpeg milik sendiri; sebagian
  sudah mati — encoder E1/E2 "pid not found"). **`pkill` tidak mencatat PID yang
  terbunuh — jadi daftar PID eksak TIDAK DAPAT direkonstruksi** (kejujuran: ini
  cacat pencatatan, bukan cacat sasaran — pola hanya bisa cocok dengan proses bench).
  Yang dapat dipastikan: setelah cleanup, `pgrep` tidak menemukan lagi ffmpeg
  bench (`/tmp/m8bench/synth…`) yang tersisa, dan ffmpeg produksi tetap hidup
  (bukti di atas).
- **Tidak ada `pkill` yang dijalankan di VPS 8.** Semua `pkill` hanya di Master.
- Perintah `pkill` di bench2/bench3: tidak ada (diganti `killpg` group sendiri).

---

## 9. Lampiran B — Bukti, batasan, observasi sampingan

**File bukti (workspace):**
- Laporan ini: `~/workspace/engineering/DIAGNOSIS_M8_2026-11-25.md`
- Data mentah sampler: `~/workspace/engineering/m8_samples_final.log` (31 sampel)
  + `m8_samples_partial.log` (17 sampel, tengah jalan)
- Skrip sampler VPS 8: `~/workspace/engineering/m8_sampler.sh`
- Analisis: `~/workspace/engineering/m8_analyse.py`
- Micro-benchmark Master: `~/workspace/engineering/m8bench.py` (v1),
  `m8bench2.py` (v2), `m8bench3.py` (v3); hasil: `/tmp/m8bench/results.jsonl`,
  `results2.jsonl`, `results3.jsonl` (di Master; ringkasan lengkap di §2–§3)
- Laporan A2 yang didiagnosis: `~/workspace/engineering/obs_a2_no88/LAPORAN_A2_NO88_PREBUILT_2026-11-25.md`
- Baseline pembanding: `~/workspace/engineering/FASE0_BASELINE_2026-11-25.md`

**Batasan:**
- Jendela live 33,5 mnt (bukan 2 jam); M1–M7 tidak diukur ulang (sudah di A2).
- Micro-benchmark Master memakai file sintetis (sine 440 Hz; video testsrc 720p),
  bukan file produksi 088 — perbandingan yang valid adalah pada laju syscall/wakeup
  dan dekomposisi, bukan angka absolut 1:1. Clock Master 1,14× VPS 8 (kecil vs 10×).
- Encoder RTMP-internet belum terisolasi (butuh uji §6-Opsi-5, perlu persetujuan).
- `thread_queue_size` belum teruji (varian E4c gagal karena bug `-y`, tidak sempat
  diulang — 1×60 dtk di Master).
- Rasio penghantaran host 0,53–0,62 terukur di jendela ini; bisa berubah menurut
  waktu/host (fair-share).
- SSH transient "Connection closed by UNKNOWN port 65535" terjadi 4× selama kerja
  (ke VPS 8 dan Master) — selalu pulih dengan tunggu 20–60 dtk + ulangi; tidak ada
  kesimpulan stream mati dari satu SSH gagal (sesuai instruksi tugas).

**Observasi sampingan (dicatat, tidak diintervensi):**
- NO 114 beralih feeder concat (PID 1957059, hidup s.d. ~14:10 WIB) → prebuilt-PCM
  (PID 1978248, mulai ~14:05 WIB) oleh supervisor — pergantian sumber normal,
  encoder 1957083 hidup terus. Bukan kegagalan.
- 2 builder prebuilt (PID 1956677/1957293) berjalan penuh sepanjang jendela
  (~0,95 core) — beban transien box.
- 707 LIVE (broadcast baru ygjGj_yWdT4) dan 698 UNVERIFIED/no_broadcast — sesuai
  konteks tugas; tidak diverifikasi ulang (di luar scope M8).
- Publish 94/1 (konteks tugas) — tidak diverifikasi ulang (tabel di DB Master,
  di luar scope M8).

*— Worker diagnosis M8, 25 Nov 2026 14:45 WIB. Read-only penuh; tidak ada perubahan
apa pun dilakukan pada produksi.*
