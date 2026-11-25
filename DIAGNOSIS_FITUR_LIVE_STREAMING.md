# Diagnosis Fitur Live Streaming — MASJAVAS TUBE OS

**Dokumen handoff untuk agent AI lain: re-analisis + re-arsitektur.**
Dibuat: 25 Nov 2026 ~08:50 WIB. Status fakta: per 08:47 WIB (terverifikasi langsung Master + VPS 8).
Repo: `masjavas7/MASJAVASTUBE-OS`, path `/opt/masjavas-os` di kedua server. Dashboard: https://masjavastube.my.id.
Bahasa dokumen: Indonesia. Semua angka di dokumen ini adalah hasil verifikasi langsung (SSH/DB/log), bukan estimasi — kecuali yang ditandai "estimasi" atau "hipotesis".

---

## 0. Ringkasan eksekutif

MASJAVAS TUBE OS menjalankan live streaming YouTube 24/7 untuk armada channel musik (707 slot). Saat ini **7 channel live serempak** di satu node (VPS 8 / live-01), **1 terjadwal** (698 MONA). Target pemilik: **ratusan channel live serempak**.

**Keadaan 25 Nov 08:47 — stabil tapi rapuh:**
- Kematian massal encoder (B10, 19 kematian/30 mnt) **TERBUKTI HILANG**: akar = RTMP IPv6 tidak stabil; fix paksa-IPv4 aktif sejak restart 07:22; 0 kematian ~80 menit sebelum restart gabungan.
- Restart gabungan selesai + TERVERIFIKASI 08:50 di kedua node (Master web+worker+live+hermes-bridge 08:40:19; VPS 8 live 08:43:30 + prep 08:44:16; keduanya di-pin di commit `173adec` secara disengaja). 7/7 STREAMING + 7 RTMP kembali, endpoint baru 401 (dulu 405), dashboard 200, cabang audio GDrive aktif — MONA kini bisa tekan Start pada task 698. D1–D6 (duplikat) AKTIF 09:17 (merge 61fc49f + restart web; sempat crash-loop web ~6 mnt karena migrasi skema — diperbaiki 6f1b3c7).
- **Update 08:51–08:53:** Fase 3 LENGKAP (kode+tes, 93 tes baru, 6 commit — pairing, Agent AI, Kalender F4/F5, Hermes, keamanan, monitoring; atribusi push 04:57 teridentifikasi lane Gravity; crash athrow terpin + fix sudah live). **Koreksi aktivasi:** web Master ternyata memuat tree `c724c3a` (bukan pin 173adec — pin hanya berlaku di merge VPS 8; reflog 08:19–08:42 hanya commit, tanpa checkout) → guard double-submit agent (3.2) SUDAH LIVE di web Master. **BARU 08:53: task 698 MONA ERROR di DB lokal VPS 8** — berkas video `gdrive_video_1844C6SeW-.mp4` tidak ditemukan di storage server (pre-copy video belum dilakukan); Master masih tampilkan SCHEDULED (divergensi DB). MONA belum bisa Start sampai video di-pre-copy ke VPS 8. **SELESAI 09:00: video + audio di-pre-copy (sha256 cocok), status lokal kembali SCHEDULED — MONA bisa Start.**
- Publish aman: 26 PUBLISHED / 0 FAILED (25 Nov per 08:47). Ghost-kill total 785, 0 baru sejak 22 Nov.
- **Gap terbesar menuju ratusan channel: KAPASITAS CPU.** Satu stream live menelan ~0,4–0,5 core (bukan ~0,05 seperti asumsi lama). Satu VPS 8 (8 core) hanya muat ~10–12 stream. 100 stream butuh ~40–50 core encoder saja (~5–6 box VPS 8 khusus live). Arsitektur saat ini (1 node live + prep berbagi box yang sama) tidak bisa diskalakan ke ratusan tanpa redesign.
- **Gap terbesar kedua: OPERASIONAL.** Jadwal live tidak bisa diedit (ganti sumber = hapus + buat ulang); editor metadata punya 3 celah; stream key 8/13 channel MONA kosong; auto-key gagal diam-diam; tidak ada audit trail create/start; telemetri tidak bisa dipercaya (live_share_pct 51,5%→123%).

**Yang dokumen ini berikan ke agent penerima:** (1) arsitektur saat ini persis seperti adanya, (2) tiap masalah dengan bukti + status perbaikan + commit, (3) analisis gap kuantitatif menuju ratusan channel, (4) daftar fitur live yang harus ada, (5) pertanyaan terbuka yang hanya pemilik yang bisa jawab.

---

## 1. Arsitektur saat ini

### 1.1 Topologi 2 node

| | VPS Master ("Master 4") | VPS 8 ("live-01") |
|---|---|---|
| Host | Contabo vmi3762689 | Contabo vmi3778263 |
| Tailnet | 100.65.206.22 | 100.108.161.98 |
| CPU / RAM / Disk | 4 vCPU / 7 GB / 145 GB (58% terpakai) | 8 vCPU / 23 GB / 290 GB (19% terpakai) |
| Peran | Web dashboard + worker publish + supervisor (siaga) + DB utama | Live supervisor (7 encoder) + prep_worker (Stage 2 + ingestion) + kalender-web + audit + watchdog |
| Jam | WIB (UTC+7) | CET (WIB−6 jam) — hati-hati membaca log |
| HEAD repo (08:47) | `4d680b3` (Fase 3 Hermes) | `173adec` (T6 form live) |
| HEAD ter-deploy | `173adec` (pin disengaja; repo sudah 4d680b3) | `173adec` (pin disengaja; origin/main sudah `c724c3a` Kalender F4/F5 — TIDAK diaktifkan) |
| Restart terakhir | web+worker+live+hermes-bridge 08:40:19 WIB | live 08:43:30, prep 08:44:16 WIB |

Semua live (7/7) dimigrasikan dari Master ke VPS 8 pada 25 Nov 05:00–05:30 via failover manual. Master kini 0 encoder live.

### 1.2 Komponen kunci (path kode)

- **Web dashboard** — FastAPI + Tailwind CDN; unit `masjavas-os-web.service` (Master). Endpoint live di `api/routes/live_stream.py` (create/start/stop/pause/resume/priority/delete; TIDAK ADA edit/update — lihat §3.12).
- **Live supervisor** — `worker/live_supervisor.py` + `worker/live_stream_engine.py`; unit `masjavas-live.service` (kedua node). Memantau task STREAMING, (re)start encoder ffmpeg, enforce public/embed, heartbeat.
- **Encoder** — 1 proses ffmpeg per stream (video loop + audio AAC 192k + aresample) + 1 feeder audio terpisah per stream (concat/relay). RTMP ke ingest YouTube (`a.rtmp.youtube.com`).
- **Relay audio** — `LiveAudioRelayManager`: re-resolve manifest tiap 12 detik; FIFO dipegang manager mode O_RDWR (feeder mati TIDAK BISA meng-EOF-kan encoder — terbukti).
- **Sentinel kesehatan** — `worker/live_health_sentinel.py`: probe YouTube API (lifeCycleStatus) + putuskan kill/reconnect. Sejak `e17a6c0`: verdict UNVERIFIED memutus rantai kill.
- **Prep worker** — `worker/prep_worker.py` (VPS 8): Stage 2 (rakit final loop: intro + loop + audio playlist + outro + spektrum) + ingestion (unduh audio/video/thumbnail ke Drive).
- **Database** — SQLite WAL `/opt/masjavas-os/data/masjavas_tube.sqlite` (Master, ~137–164 MB). **VPS 8 punya DB lokal sendiri yang DIVERGEN** (max_streams live-01 = 70 di sana vs 12 di Master; heartbeat lokal segar). Sinkronisasi Master↔VPS 8 adalah follow-up terbuka (§3.3).
- **Kuota YouTube** — per PROJECT: 100 videos.insert + 100 search.list/hari + 10.000 unit/hari; **1 video = 1.650 unit**. Aturan mengikat: maks 5 channel/vault (5 × 1.650 = 8.250 unit, di bawah guard 8.400). Target ~140 vault untuk 696 channel; live: 37 vault.

### 1.3 Aliran normal (create → live)

1. Operator isi form `/live/create` (channel, judul, video, audio, node, resolusi) → POST `/api/live/schedule` → task SCHEDULED di DB Master.
2. Supervisor node tujuan pickup task (sejak `e2779ed`: PENDING_PICKUP + pickup IDLE tiap 15 detik; sebelumnya TIDAK PERNAH pickup IDLE).
3. Supervisor start ffmpeg (unduh video HANYA bila path `gdrive://`, bila file tidak ada → ERROR; pre-copy manual wajib untuk failover).
4. Encoder RTMP ke YouTube; supervisor enforce public + embeddable; Sentinel verifikasi LIVE via API.
5. Reconnect otomatis maks 5 percobaan; feeder restart maks 5×; guard anti-duplikat 409 (channel+judul+operator, 10 menit).

### 1.4 Baseline produksi (25 Nov 2026, terverifikasi)

- Publish: 24 Nov FINAL 210 PUBLISHED / 0 FAILED; 25 Nov 26 PUBLISHED / 0 FAILED (per 08:47). Lantai SOLID: 206/hari + 0 FAILED.
- Live: 7 STREAMING + 1 SCHEDULED; 0 kematian ~80 mnt pasca-fix IPv4; load VPS 8 7,3–10,5 (jenuh); Master 1,7–2,5 (sehat).
- Stage 2: DONE 1395, QUEUED 4, RUNNING 2, PARKED 278, FAILED 21.
- Channel: CONNECTED 241 / UNPAIRED 466 / total 707 slot. Operator: AZIZ 251, ARIEF 143, AGUS 100, MONA 13, rotasi 5 operator × 40.
- Ghost-kill: total 785, 0 baru sejak 22 Nov (29 jendela verifikasi).

---

## 2. Status perbaikan per 08:47 WIB

| # | Perbaikan | Commit | Status |
|---|---|---|---|
| B10 IPv6 | Paksa RTMP IPv4 | `d5e4764` | **AKTIF** sejak 07:22; 0 kematian ~80 mnt |
| Watchdog encoder | Kill encoder beku 30 dtk + reconnect | `ed81681` | AKTIF sejak 07:22; 0 aksi sejauh ini (belum ada beku) |
| Embed rejection | Ingat penolakan per channel (tabel baru) | `d5c8e8c` | AKTIF sejak 07:22; 0 baris (terisi saat penolakan berikut) |
| Re-manifest cepat | Feeder 403 → re-manifest | `1188e9b` | AKTIF sejak 07:22 |
| Stage 2 breaker | Port breaker ke prep_worker + safety RAM/disk | `e1ae9d0` | AKTIF sejak 07:22; DONE bergerak (1395) |
| Bundle livestreaming | Failover, heartbeat, pickup IDLE, double-submit, throttle guard, B10 pre-mortem, cookie Chrome | `e2779ed` + `b6ceb80` | AKTIF sejak 06:25 |
| Sentinel + healer | UNVERIFIED putus rantai kill; healer tanpa gate lane | `e17a6c0` | AKTIF sejak restart 08:40/08:43 |
| DB-lock | "database is locked" → infra + retry lintas proses | `0eb8bab` | AKTIF sejak restart 08:40/08:43; verifikasi 7 hari berjalan (target 0) |
| Celah audio (janitor) | Touch mtime, lindungi RUNNING, self-heal, sweeper, SKIPPED_VIDEO_MISSING | `1523da1`–`edbeb1f` (5 commit) | AKTIF sejak restart 08:40/08:43 |
| CTR + governor audit | CTR nyata dari Analytics + cap 1000 unit/vault/hari | `8fdfcbb` | AKTIF (web 08:40); verifikasi live: run audit 19:00 |
| Live-edit jadwal | Edit endpoint, hot-swap relay, template, revisi/rollback, bulk | Fase 0–5 (15+ commit) | Web aktif 08:40; supervisor aktif 08:43 — verifikasi fungsional pending |
| Metadata S1–S5 | Status jujur, antrean retry, simpan deskripsi+tag, validasi thumbnail | Fase 2b | Web aktif 08:40 — verifikasi fungsional pending |
| MONA M1–M3/G1–G7/T1–T6 | Checklist, auto-key jujur, audio GDrive, judul 100 karakter | ~10 commit | Web aktif 08:40; cabang G1 supervisor TERVERIFIKASI aktif 08:50 (line 1191) — MONA bisa Start 698 |
| Duplikat D1–D6 | Kunci tombol, idempotency key, kartu "tugas sudah ada" | `61fc49f` + fix `6f1b3c7` | **AKTIF 09:17** (merge + restart web; insiden: migrasi skema crash-loop web ~6 mnt — indeks dibuat di SCHEMA_SQL sebelum kolom ALTER; pelajaran: uji migrasi di salinan DB live, bukan hanya DB fresh) |
| Fase 3.1 Pairing | Prioritas pairing (459/462 UNPAIRED tanpa vault), spek monitoring mingguan | — (skrip + laporan) | Selesai (kode+tes); penetapan vault butuh keputusan (keranjang c) |
| Fase 3.2 Agent AI | Harness pilot judul+QA + guard double-submit | `668130c` | Guard SUDAH LIVE di web Master (c724c3a); Fase C apply TIDAK ADA KODENYA — butuh keputusan + bangun applyer (keranjang d) |
| Fase 3.3 Kalender F4/F5 | Prelive + HITL-2 + handoff broadcast (gate OFF) | `1005ced` | Selesai (kode+tes); aktivasi butuh keputusan F4 + channel + key (keranjang e) |
| Fase 3.4 Hermes | Konsumen inbox (1.956 baris, 0 file existing diubah) | `4d680b3` | Selesai (kode+tes); loop BELUM diregistrasi — aktivasi + verdict menunggu owner (keranjang f) |
| Fase 3.5 Keamanan | Sanitasi rahasia + pemindai + proposal deploy key | `c724c3a` | Selesai; **5 PIN operator SAMA dengan produksi → wajib rotasi**; PAT lama MASIH AKTIF (keranjang a, g) |
| Fase 3.6 Monitoring | Restore drill PASS + 3 alert + panduan UptimeRobot | — (skrip workspace) | Selesai; 29 baris uji SKH + 312 orphan FK perlu dibersihkan (keranjang h, i) |
| Fase 3.7 Atribusi | Pelaku push rebase 04:57 TERIDENTIFIKASI (lane Gravity, javaswahid) | — (laporan) | Selesai; `denyCurrentBranch` → refuse butuh keputusan (keranjang b) |
| Crash athrow | Mekanisme terpin (retry loop yield 2× pasca-athrow) | `87e3837` + `05b9380` | Fix SUDAH LIVE (0 athrow 25 Nov); tanpa aksi baru |
| Duplikat D1–D6 | Kunci tombol, idempotency key, kartu "tugas sudah ada" | scope ke-4 | Kode+tes (belum aktif — butuh restart berikutnya) |
| Akar readonly (User=root) | prep + telemetri jadi ubuntu + chown subdir | — | **BELUM DIKERJAKAN** (item 8, butuh persetujuan terpisah) |
| URL relay 034 | Video sumber dihapus (lisensi) | — | **BUTUH INPUT OPERATOR** (item 9) |
| 1080p 4 stream | 707+088 DB-only; 034+WULAN butuh sumber 1080p | — | **PARKIR** (item 7, gate kematian <2/jam + headroom) |
| RETIRE vs REVIVE stage2-driver | PID 1338969, /tmp, jalan 6d17h, IDLE | — | **KEPUTUSAN USER** (menunggu) |


## 3. Diagnosis masalah (dengan bukti)

### 3.1 B10 — kematian massal encoder [SELESAI, akar terbukti]

**Gejala:** 5–6 encoder mati serentak dalam hitungan detik, berulang tiap ~9–10 menit. Klaster tercatat: 22 Nov (4 stream MONA mati 14 jam), 24 Nov 00:47, 02:06, 05:03 (WULAN exit 187 + 088 exit 152), 05:32, 06:09, 06:18 (5 encoder exit 146), 07:05, 07:14 (5 encoder yang sama, 19 kematian/30 menit). Recovery otomatis bekerja (reconnect attempt 1/5), tapi downtime berulang + risiko kill-loop.

**Diagnosis (berlapis, terbukti):**
1. Pre-mortem (B10 instrumentation, aktif 06:25) menangkap ekor log: `Error submitting a packet to the muxer: Connection reset by peer` + `Broken pipe` — koneksi RTMP di-reset; encoder BEKU (CPU 0%, tx/size/time identik) ≥6 detik sebelum mati.
2. Bukti penyingkir: relay HLS 403 hanya gejala paralel (feeder mati tidak bisa meng-EOF-kan encoder — FIFO dipegang manager O_RDWR); load 5,28–6,06 + mem 8% + swap 0 (bukan kehabisan resource); IPv6 statis valid + 0 flap link; TX bertahan 5,4–5,5 Mbps selama klaster (bukan blackout total).
3. **Akar akhir: RTMP IPv6 tidak stabil.** Bukti: 2 encoder IPv4 selamat melewati 4+ klaster vs 5 encoder IPv6 mati berulang; 088/WULAN mati 06:18 (saat itu IPv6) dan selamat setelahnya (IPv4). **Prediksi klaster 07:04 TEPAT: 5 IPv6 mati (exit 152/187), 2 IPv4 (etime 38+ mnt) selamat** — konfirmasi ke-5.
4. Tanda tangan campuran di klaster yang sama: 645 mati `End of file` (EOF audio/FIFO) vs 034 `Connection reset by peer` (RTMP) — dua mekanisme mati aktif bersamaan, tapi pemicu bersama adalah socket stall IPv6 (pre-mortem: tx beku + CPU 0% selama 30–130 detik → timeout).

**Perbaikan:** `d5e4764` paksa RTMP IPv4 (diff +9/−2 supervisor + modul baru `rtmp_ipv4.py`, tes 11/11 + regresi 4/4). Aktif sejak restart 07:22 (7/7 RTMP kini IPv4, 0 IPv6). Verifikasi: 0 kematian 30 menit (pola lama = ~15), 0 kematian ~80 menit (7/7 encoder jalan 01:16–01:18). **Watchdog progres encoder (`ed81681`, kill beku 30 dtk + reconnect) adalah jaring pengaman, bukan obat** — 0 aksi sejauh ini.
**Sisa:** hipotesis pemicu tingkat lanjut (stall sesaat jalur egress ke ingest YouTube / reset sisi YouTube) belum terbukti 100% — butuh capture live (ping kontinu ke a.rtmp + monitor). Bila IPv6 kembali dipakai di masa depan, masalah ini akan kembali.

### 3.2 Kapasitas CPU per stream — angka riil vs asumsi [MASALAH STRUKTURAL, belum selesai]

**Temuan terukur (25 Nov 05:12–05:20):** satu encoder makan **~0,4–0,5 core** (35–53% CPU/encoder — transcode audio AAC 192k + aresample + concat audio terpisah), BUKAN ~0,05 core seperti klaim UI/skor node. Feeder audio ~4–10% CPU/stream. **Kapasitas riil VPS 8 (8 core) ≈ 10–12 stream, bukan 70.**
- Bukti beban: load VPS 8 7,5–13,94 saat 6–7 stream; load1 10,47 saat restart storm (08:47); load15 ~7,3 (jenuh penuh).
- VPS 8 BUKAN kotak live khusus: role `live+prep` — prep_worker (Stage 2 + ingestion) + kalender-web + audit + watchdog se-box; QA Stage 2 ffmpeg 125% CPU bersamaan dengan encoder live.
- Badai relay yt-dlp: 3× yt-dlp + Node.js ≈ 2,5 core + warp-svc 20%; Node.js transien ~3 core tiap badai reconnect (siklus setan beban).
- `max_streams` dikoreksi 70→12 di DB Master (terverifikasi). DB lokal VPS 8 masih 70 (divergen — §3.3).
- Mitigasi yang sudah jalan: cap prep 200% via `set-property` + drop-in persisten (item 4, tanpa restart); throttle yt-dlp (commit `31f3ff0`); skor node cap CPU.

**Implikasi untuk target ratusan channel: lihat §4.** Ini satu-satunya masalah yang TIDAK BISA diselesaikan dengan bugfix — butuh re-arsitektur (scale-out / pisah prep / kurangi biaya encoder).

### 3.3 Sinkronisasi Master ↔ VPS 8 [SEBAGIAN SELESAI]

- **Heartbeat beku:** `apply_task_status_report` tidak menulis `last_heartbeat` → heartbeat 706/114/707 beku sejak 23–24 Nov; Master menampilkan label basi (5 STOPPED + 1 RECONNECTING padahal 6 STREAMING di VPS 8). Fix heartbeat COALESCE (`e2779ed`) + label STALE >5 menit. **Sync pulih 07:10: heartbeat Master segar + PID baris Master cocok 7/7 dengan encoder VPS 8.**
- **DB lokal divergen:** VPS 8 punya DB sendiri (137 MB) — `max_streams` 70 vs 12 Master; status live lokal (4 STREAMING + 2 RECONNECTING pada 06:38) vs Master (6 STREAMING). Perluasan cakupan sync adalah follow-up terbuka (belum dikerjakan).
- **Monitor Master-sentris buta:** cek PID di Master tidak membuktikan apa pun untuk stream migrasi (pelajaran 05:17 — klaim "706/114/707 encoder mati" SALAH, encoder hidup di VPS 8). Aturan: verifikasi klaim hidup/mati HARUS di node yang bersangkutan (PID + RTMP + sentinel YouTube).

### 3.4 Bug failover Fase C [SELESAI, kode]

Dua bug deterministik (macet 3/3 saat migrasi 05:00–05:30):
1. **#1:** stop-hook Master menimpa `migration_state` 'migrating'→'migrating_out'→NULL di DB Master — `derive_orders` hanya baca 'migrating' → perintah start tidak pernah terbit. Workaround: skrip `live_unwedge_migration.py`.
2. **#2 (blind spot health check):** target tidak pernah menghapus 'migrating_in' lokal → `run_health_check` mengecualikan task itu → encoder 034 mati ~23:14 tanpa terdeteksi 3 menit. Workaround: hapus flag lokal pasca-migrasi.
Fix di bundle `e2779ed`. Catatan: pre-copy video WAJIB sebelum failover — supervisor start HANYA unduh bila path `gdrive://`; file tidak ada → ERROR (`live_supervisor.py` ~826).

### 3.5 Dispatch + pickup task IDLE [SELESAI, terverifikasi]

**Sebelum:** tidak ada dispatch lintas node (grep dispatch kosong) + supervisor TIDAK PERNAH pickup task IDLE (hanya pantau STREAMING) → alur start ke VPS 8 BUNTU TOTAL. Guard Fase C menolak dengan benar — yang rusak alurnya, bukan guard-nya.
**Perbaikan (`e2779ed`):** status PENDING_PICKUP + pickup IDLE tiap 15 detik. **Pilot 645 TERVERIFIKASI 06:54:** pickup 00:54:15 CET, mengudara 00:54:19 (PID 1658850, 720p), broadcast publik `ZaJ8UzdEJ8g` dibuat 00:55:42, Sentinel verdict LIVE + HEALTHY (lifeCycleStatus=live). Encoder sempat mati 90 detik kemudian (klaster B10 00:55, exit 146) lalu reconnect 1/5 sukses.

### 3.6 Double-submit + guard duplikat [SEBAGIAN SELESAI]

- **Duplikat 645:** 2 task IDLE STAGING-SLOT-645 (double-submit, tanpa guard) sejak 05:09/05:10. Satu dihapus Fase 0 (backup server-side). Guard double-submit 409 (`e2779ed`).
- **Bug MONA 08:31 (BARU, D1–D6):** MONA klik MULAI SIARAN → task #1795570268004 TERBUAT + SCHEDULED (698, live-01, GDRIVE_URL, 08:31:08) tapi UI tampilkan 409 "Tugas duplikat... 0 detik lalu". Log: POST `/api/live/schedule` 08:31:08 + 08:31:26 → 409; create pertama berhasil di detik yang sama. Guard `live_stream.py:1531` (channel+judul+operator, 10 menit). Tombol SUDAH dikunci saat submit — double-fire bukan klik ganda polos (kemungkinan kunci terlambat setelah `await`). **Perbaikan D1–D6** (scope ke-4, kode+tes, belum aktif): kunci tombol di baris pertama handler; idempotency key; 409 → kartu "tugas sudah ada" + tombol aksi (bukan toast merah); guard abaikan task FAILED; 409 sertakan task_id + deep link.

### 3.7 Cookie yt-dlp + relay audio [SEBAGIAN SELESAI]

- Cookie: format Netscape→Chrome profile (`e2779ed`); izin root 0600 (tidak bisa dibaca supervisor) → ubuntu 0600; BOM UTF-8 dihapus; kini parse Netscape (10/11 entri valid s.d. 2027–2028). **TAPI sesi login kedaluwarsa — refresh butuh login operator via dashboard** (item 5, sebagian selesai).
- Uji ekstraksi manifest gagal di 3 jalur × 2 URL relay (bot-check YouTube sistemik); fallback playlist darurat terbukti jalan saat reconnect.
- Relay HLS 403 (URL manifest kedaluwarsa) → re-manifest cepat `1188e9b` (58/58 tes, aktif 07:22).
- **Sumber relay slot 034 (SALSA) DIHAPUS YouTube** (video `kizfrmykOkM`, "removed due to a contractual obligation with a music licensor") — tidak ada perbaikan teknis; operator harus sediakan URL pengganti (item 9). Catatan: task #1795502710790 sempat broadcast sukses 00:26:35 CET via jalur audio darurat/fallback.
- **DIRECT_URL diterima API tapi DIABAIKAN supervisor** → task jalan BISU (celah sistemik).
- **Mode audio GDRIVE_URL baru (G1–G7):** pakai ulang mesin video `gdrive://` + `download_gdrive_video`, cabang supervisor + cache per file ID, tombol "Cek link" probe ffprobe, batas 500 MB, 0 unit kuota YouTube. Aktif setelah restart VPS 8 08:43 — **TERVERIFIKASI aktif 08:50** (cabang supervisor line 1191); MONA kini bisa tekan Start pada task #1795570268004 (sebelum restart, Start = audio bisu).
- **BARU 08:53 — task 698 ERROR (video belum di-pre-copy):** DB lokal VPS 8 mencatat 698 ERROR — "Berkas video 'gdrive_video_1844C6SeW-.mp4' tidak ditemukan di storage server" (updated_at ~08:53 WIB, setelah restart). Audio G1 aktif, tapi file VIDEO belum ada di storage VPS 8 (supervisor hanya unduh path `gdrive://`; pre-copy wajib — §3.4). Master masih tampilkan SCHEDULED (divergensi DB §3.3). MONA belum bisa Start sampai video di-pre-copy ke VPS 8 (via endpoint download-gdrive atau salin manual). **SELESAI 09:00:** video + audio di-pre-copy via scp Master→VPS 8 (sha256 cocok), status lokal kembali SCHEDULED.

### 3.8 Penolakan embed YouTube [SELESAI]

WULAN + 034 kena `invalidEmbedSetting` 400 di TIAP reconnect (enforce_public_embeddable) — membuang kuota + memenuhi log. Fix `d5c8e8c`: ingat penolakan per channel (tabel `channel_embed_rejections`, 0 unit kuota; 12/12 tes; aktif 07:22).

### 3.9 Readonly-DB [PERBAIKAN SEMENTARA AKTIF, AKAR BELUM]

- Gejala: "attempt to write a readonly database" — ledger kuota audit gagal dicatat (01:12:47) + reconnect gagal tulis. Kelas koneksi DB read-only dipakai untuk tulis.
- **Probe `adc2b13` TERBUKTI CACAT:** `BEGIN IMMEDIATE` + ROLLBACK lolos di koneksi readonly → retry tidak pernah berjalan. Diganti probe tulis-nyata `b6ceb80` (12/12 tes; aktif 06:25).
- **Akar BELUM diperbaiki (item 8):** unit prep + telemetri jalan sebagai `User=root` + subdir root-owned → butuh ubah ke ubuntu + chown + restart prep. Butuh persetujuan terpisah (belum diajukan).

### 3.10 Database-locked [FIX AKTIF, VERIFIKASI 7 HARI]

- Gejala: "database is locked" — task #5196 (STAGING-SLOT-042) FAILED 07:15:31; task #5142 (NO 267 Rock Kenangan) gagal 07:06 di transisi DOWNLOADING_ASSETS→RENDERING; **27 error lock dalam sehari** (web 22, worker 3, live 2); lock dipegang 10–30+ detik.
- **#5196 ternyata videonya SUDAH TERBIT di YouTube** (ID `f45aF9sRBng`, 07:10:44) — FAILED hanya gagal tulis DB pasca-upload. Status dikoreksi ke PUBLISHED (backup dulu, tanpa upload ulang). FAILED hari ini kembali 0.
- Akar: "database is locked" TIDAK ADA di daftar transient `_is_infra` (`autonomous_daemon.py:1201-1205`) → auto-retry 1× tidak terpicu. #5142 di-retry manual ke 07:19:27 (kuota 1.650 unit direfund).
- Fix kelas `0eb8bab` (Fase 2.5): lock → `_is_infra` + retry decorator lintas proses + release/reconcile ber-retry (12/12 tes; aktif 08:40/08:43). Koreksi: heal SUDAH me-release reservasi via `reconcile_vault_quota_with_ledger` — hipotesis bocor kuota 5078 terkoreksi (celahnya: reconcile sendiri tanpa retry). Verifikasi: grep "database is locked" target 0 selama 7 hari.

### 3.11 Editor metadata — sudah ada, 3 celah [FIX AKTIF DI WEB, VERIFIKASI PENDING]

Editor judul/deskripsi/thumbnail SUDAH ADA (modal `modalLiveMetadataEditor` + POST `/api/live/metadata/{task_id}` + `update_live_broadcast_metadata` + kolom title/description/thumbnail_path di skema) — tapi 3 celah terverifikasi (adendum 07:31–07:35):
1. **Pesan sukses PALSU** — endpoint selalu kembalikan "berhasil di YouTube" meski broadcaster mengembalikan `quota_blocked` (flag diabaikan).
2. **Sync tertunda TIDAK PERNAH dicoba lagi** — tidak ada retry worker/antrean/indikator.
3. **UPDATE lokal hanya simpan title/broadcast_id/url/thumbnail_path** — kolom `description` tidak pernah ditulis, tags tidak punya kolom → edit deskripsi yang ter-defer HILANG TOTAL.
Plus: thumbnail tanpa validasi dimensi/ukuran; tombol berlabel jargon "In-Flight SEO"; modal edit Fase 1 juga mengedit judul (dua permukaan bertabrakan).
**Fix Fase 2b S1–S5** (status jujur + antrean retry `live_metadata_sync_queue` + simpan semua field lokal + validasi thumbnail + satukan ke wizard edit langkah "Judul & SEO"): aktif di web sejak restart 08:40 — verifikasi fungsional pending. Dokumen: `engineering/ADENDUM_EDIT_METADATA_LIVE.md`.

### 3.12 Jadwal live TIDAK BISA DIEDIT [BOTTLENECK UTAMA OPERASIONAL]

**Terverifikasi dari kode (06:38): TIDAK ADA endpoint edit/update untuk live task** — hanya create/start/stop/pause/resume/priority/delete di `api/routes/live_stream.py`. Editor dashboard hanya ubah judul/deskripsi/tag/thumbnail (In-Flight SEO). Ganti sumber video/audio/URL relay = **hapus + buat ulang dari nol** (kehilangan binding broadcast + riwayat). Saat problem muncul (video dihapus, relay mati, audio salah), operator tidak bisa menindaklanjuti dengan cepat.
**Kabar baik:** skema `live_stream_tasks` SUDAH punya semua kolom yang dibutuhkan (video_path/url, audio_source_type/path/url, youtube_live_url, start/stop epoch, repeat_mode, resolution, bitrate).
**Desain lengkap:** `engineering/DESAIN_JADWAL_LIVE_EDITABLE.md` (14 bagian + 2 lampiran; disetujui user 06:43). Isi: edit endpoint POST + hot-swap URL relay tanpa matikan encoder (via `LiveAudioRelayManager.request_source_swap`, FIFO O_RDWR) + validasi sumber saat simpan (probe_media_availability + manifest — video dihapus tidak bisa tersimpan) + panel quick-fix + template jadwal + riwayat revisi & rollback 1 klik + bulk-edit + perubahan terjadwal; antrean order lintas-node baru `live_task_orders`; optimistic locking `config_version`. Keputusan: hot-swap bertahap, bulk-edit OWNER saja, retensi 50 revisi/task, P2 dibangun sebagai kode tapi aktivasi ditunda.
**Implementasi Fase 0–5** (kode+tes, 15+ commit): web-side aktif 08:40, supervisor handlers aktif 08:43 — verifikasi fungsional pending.
**Panduan GUI** (berlaku untuk semua UI baru): `engineering/PANDUAN_GUI_MODERN_MASJAVAS.md` — 1 aksen emerald + 4 warna status; tipografi 3 ukuran (14/12/12-mono, teks <12px dilarang); maks 2 tombol per baris + menu ⋯; form create → wizard 4 langkah. (Audit: dashboard memakai 10 warna aksen, teks 9–11px dominan, tombol gradasi rose→red, kartu blur/transparan, 5–8 tombol per baris.)

### 3.13 MONA gagal live manual [DIAGNOSIS 2× KOREKSI]

- **Diagnosis pertama (07:59, SEBAGIAN SALAH):** MONA login 07:51:08, tekan auto-key 2× (07:51:49 + 07:52:14, HTTP 200), form TIDAK PERNAH ter-submit (0 POST `/api/live/schedule`). 8/13 channel MONA tanpa stream key tersimpan (153, 697, 699–702, 704, 705); key field WAJIB. Auto-key Tier 2 (YouTube API) kemungkinan gagal karena kuota vault error pagi itu (burst quotaExceeded 07:22) / scope OAuth; Tier 3 butuh upgrade otorisasi. Channel yang dipilih + pesan di layar tidak tercatat (body respons tidak di-log) — belum 100% pasti.
- **Koreksi (08:09, dari screenshot):** untuk percobaan ini, auto-key BERHASIL (key `tm4d-8r3e-xd8y-uvew-c6qk` terisi) — yang menggagalkan adalah **judul >100 karakter** ("PMGC 2026 LIVE | PUBG Mobile Global Championship Istanbul — Live Scores,..." → 422 Pydantic `body.title: String should have at most 100 characters`). Mekanisme: form punya 2 field judul — autoBroadcastTitle (PUNYA maxlength=100) vs Judul Tugas Internal (TANPA maxlength, satu-satunya maxlength di form 2136 baris); payload kirim judul internal sebagai body.title; server validasi max_length=100. Akar: judul internal + judul broadcast YouTube disatukan dalam satu field (broadcaster potong diam-diam [:100]). Temuan keyless-channel (8/13) TETAP VALID untuk channel lain.
- **Perbaikan:** M1–M3 (checklist pra-syarat, auto-key jujur, jalur key manual) + G1–G7 (audio GDrive) + T1–T6 (maxlength+counter judul internal; pisah judul internal 200 vs broadcast_title 100; terjemahkan 422 ke Bahasa + highlight field; tombol "pendekkan otomatis"; peringatan 0 lagu; tampilkan node tujuan + kapasitas) + D1–D6 (§3.6). Dokumen: `engineering/DIAGNOSIS_MONA_LIVE_MANUAL_DAN_AUDIO_GDRIVE.md`.
- **Celah sistemik:** form tanpa checklist pra-syarat; auto-key gagal diam-diam (200 kosong); create/start tidak diaudit; DIRECT_URL diabaikan supervisor (§3.7).

### 3.14 Telemetri + skor node tidak bisa dipercaya [SEBAGIAN SELESAI]

- TX live-01 terbaca 0,3 Mbps (seharusnya ~12 untuk 6 stream); kini 12,9 Mbps setelah fix heartbeat — realistis.
- Modal node 3+5=8 stream vs DB 6 STREAMING; hanya 3/4 koneksi RTMP established (saat itu).
- **Rumus skor BENAR** (46 dan 59 direproduksi persis dari komponen) — yang salah INPUT-nya (CPU per stream, stabilitas, freshness).
- `live_share_pct` masih ngaco (51,5%→123%).
- `guard_blocked_start`: 178+ baris node_events, spam tiap ~5 detik → throttle 900 detik (`e2779ed`).

### 3.15 Ghost-kill + Sentinel [SELESAI, verifikasi berjalan]

Total ghost-kill 785 (beku sejak 22 Nov — brake deployed 03:53 menghentikan blind kill). Perbaikan inti B11: Sentinel UNVERIFIED memutus rantai kill (`e17a6c0`, +132 baris, tes 326+335; aktif 08:43). Protokol verifikasi 7 hari: `docs/engineering/PROTOKOL_VERIFIKASI_SENTINEL_2_1.md`.

### 3.16 Healer ingestion tidak pernah jalan [SELESAI, kode]

Healer ingestion TIDAK PERNAH JALAN sejak 23 Nov 20:39 — worker Master `--lane publish`, tick healer mensyaratkan lane all/ingestion (0 baris `[Failed Retry Healer]` di journal sejak 24 Nov). Tick healer + deficit sweeper kini tanpa gate lane (DB-only) — `e17a6c0`. #409 (FAILED, attempts=0) auto-retry pasca-aktivasi; #542 (COMPLETED_PARTIAL, UNPAIRED) bukan kandidat healer — jalur sweeper/pairing.

### 3.17 Frontend/backend tidak sinkron [SELESAI via restart 08:40]

Template baru (counter 31/200+100/100, checklist, tab GDrive) sudah tayang di browser MONA, tapi backend web masih kode 06:17 (PID 3246819) → `/api/live/audit-event` + `/api/live/validate-source` → 405 (checklist "Pemeriksaan gagal"). Teratasi oleh restart web 08:40. **Pelajaran:** deploy frontend (template) dan backend (Python) tidak atomik — template baru bisa tayang sebelum backend-nya aktif.

### 3.18 Playlist darurat tipis

WULAN 3 trek, 114 hanya 6 trek — fallback audio saat relay mati sangat tipis. Perlu diperkaya per channel (target: ≥30 trek atau playlist penuh).

### 3.19 Stage 2 — pasokan bahan loop [SEBAGIAN SELESAI]

Bukan live langsung, tapi live memutar video loop yang dirakit Stage 2. Diagnosis: DONE macet (+1/3 jam; 1387, QUEUED 7–9, PARKED 277–279, FAILED 21). **Akar PASTI:** `prep_worker` VPS 8 memanggil `process_stage2_item` tanpa `allow_high_load`; gerbang load (load15 6,5 / load1 7,0) selalu tertutup di bawah load 9,6–11,3; tiap percobaan bakar timeout 300 detik lalu defer (27× `resource_deferred` beruntun). Breaker 20 Nov tertinggal di daemon Master yang dorman (0 baris LoopGen 24 jam). Fix `e1ae9d0`: port breaker ke prep_worker + safety RAM/disk (14/14 tes; aktif 07:22; DONE 1395 + bergerak). Sisa: PARKED 278 (mayoritas QA spektrum marginal) + FAILED 21 butuh keputusan operator.

### 3.20 Janitor menghapus audio in-flight [SELESAI, kode]

Terkait bahan (bukan live langsung): janitor menghapus audio saat job ingestion masih stage AUDIO — terverifikasi dari `deletion_audit`: 12 file Coffee Corner dihapus 05:08:33–34 (`JANITOR_UNPAIRED_QUARANTINE_CLEAN`) + 8 file 05:36:43 (`WORKSPACE_PRUNE_6H`). Mekanisme: audio Global Pool via hardlink mewarisi mtime lama file pool (21–23 Nov) → langsung "berumur >1 jam" → janitor TTL 1 jam hapus (tanpa cek job RUNNING). Skala janitor: ~3.000–3.400 file / ~100–126 GB per hari (20–24 Nov). Fix 2.3 (5 commit): touch mtime saat materialize, janitor lindungi job RUNNING, self-heal re-materialisasi, sweeper smallest-gap-first, status baru SKIPPED_VIDEO_MISSING (audio-murni 18/hari, bukan 30 — 12/30 SKIPPED ternyata video-only). Bonus: bug `NameError settings` di prune 6H (skip-check tidak pernah jalan karena import hilang ditelan `except: pass`) diperbaiki.

### 3.21 Proses deploy/restart rapuh [TEMUAN 08:36–08:50]

- `systemctl restart` TANPA sudo GAGAL "Interactive authentication required" di kedua server — SELALU pakai `sudo systemctl restart` (sudo passwordless).
- **Tree VPS 8 hampir seluruhnya root-owned (51.363 file di luar .git + 51.382 di .git)** — pull sebagai ubuntu mustahil sebelum `sudo chown -R ubuntu:ubuntu /opt/masjavas-os` (0 sisa). Layanan jalan normal setelah chown. (Terkait §3.9: file root-owned tersebar di tree adalah masalah sistemik, bukan insiden tunggal.)
- **Pull gagal meninggalkan worktree setengah-terupdate (43 file):** pull VPS 8 mati di tengah karena file root-owned (`core/storage/lease_registry.py`). Sebelum membuang: verifikasi file kotor adalah SUBSET file pull — baru `git checkout -- .`. Jangan `git clean` (data/ berisi DB live).
- **File untracked memblokir merge:** 12 file untracked (tes Fase 2 + docs + service stage2-driver) — pindahkan ke backup /tmp dulu, merge, lalu diff backup-vs-hasil (12/12 identik = tidak ada yang hilang).
- **Pin HEAD saat lane lain aktif push:** origin/main bergerak 1005ced→c724c3a DALAM restart (Kalender F4/F5 Fase 3). Merge ke commit PIN (`git merge --ff-only 173adec`), bukan ke origin/main — kalau tidak, kode lane lain yang belum direstui ikut aktif. Kedua node kini berjalan di 173adec (pin disengaja).
- Prep VPS 8 perlu re-restart pasca daemon-reload karena definisi unit berubah.
- **Implikasi re-arsitektur:** butuh pipeline deploy yang atomik + terverifikasi (pin versi, pre-flight check kepemilikan file, health-check pasca-restart otomatis, rollback 1 perintah) — saat ini deploy = SSH manual + git pull + restart, rapuh terhadap kondisi tree.

---

## 4. Target: ratusan channel live serempak — analisis gap

### 4.1 Matematika kapasitas (angka terukur, bukan asumsi)

| Skala | Core encoder (0,4–0,5/stream) | Box VPS 8 (8 core, khusus live) | Catatan |
|---|---|---|---|
| 7 stream (kini) | ~2,6–3,5 core | <1 box (berbagi dengan prep) | Load sudah 7,3–10,5 |
| 12 stream (maks 1 box) | ~5–6 core + feeder + relay | 1 box (tanpa prep) | Batas aman box saat ini |
| 50 stream | ~20–25 core | ~3–4 box | + feeder/relay ~5 core |
| 100 stream | ~40–50 core | ~6–7 box | Egress RTMP ~200 Mbps |
| 200 stream | ~80–100 core | ~12–13 box | Egress RTMP ~400 Mbps |
| 696 stream (seluruh armada) | ~280–350 core | ~40–45 box | Egress RTMP ~1,4 Gbps |

**Asumsi yang HARUS diuji ulang oleh agent penerima:** 0,4–0,5 core/stream diukur pada 720p (645) + 1080p campuran dengan transcode AAC 192k penuh. Bila audio bisa passthrough (tanpa transcode) atau feeder digabung ke encoder, biaya per stream bisa turun signifikan — ini hipotesis, belum diukur.

### 4.2 Bottleneck di luar CPU

1. **Kuota YouTube API:** 1 video = 1.650 unit; 10.000 unit/hari/proyek; maks 5 channel/vault → 100 channel butuh 20 vault, 696 butuh ~140 vault. Live (broadcast insert + enforce + sentinel probe) juga makan unit — hitung terpisah dari publish.
2. **Bandwidth egress:** ~2 Mbps/stream RTMP. 100 stream = 200 Mbps; 696 = ~1,4 Gbps. Port VPS 8 terukur ~91–93% dari 500 Mbps (paralel). Satu box tidak bisa menampung >~200 stream dari sisi network saja.
3. **Operasional manusia:** 1 operator menangani 40–143 channel; onboarding per channel butuh OTP/nomor + Gmail + stream key + pairing. 8/13 channel MONA tanpa key menunjukkan onboarding belum tuntas bahkan di skala kecil.
4. **Single point of failure:** DB SQLite tunggal di Master (WAL, ~164 MB); tidak ada replika; backup harian. Pada ratusan stream, tulis heartbeat/status dari banyak node akan menekan lock SQLite (27 error lock/hari sudah terjadi pada skala 7 stream).
5. **Observabilitas:** tidak ada metrik per-stream terpusat (bitrate, frame drop, uptime, reconnect count) + alerting; operator tahu stream mati dari laporan manual / screenshot.

### 4.3 Opsi arsitektur (untuk dianalisis agent penerima — BUKAN keputusan)

1. **Scale-out node live:** tambah VPS 8 (atau setara) per ~10 stream; pisah peran: node live HANYA live (prep pindah ke box sendiri). Paling sederhana, paling mahal secara linear.
2. **Turunkan biaya per stream:** audio passthrough (tanpa transcode AAC bila sumber sudah AAC); gabung feeder ke proses encoder; preset ffmpeg lebih ringan; turunkan ke 720p untuk sebagian channel (trade-off terbuka — keputusan 1080p sudah diambil user 18 Nov, 720p ditawarkan sebagai penghemat: VPS 8 muat 134 stream 720p vs 70 stream 1080p per estimasi LAMA yang asumsi CPU-nya salah — **angka ini harus diukur ulang**).
3. **Penjadwalan slot berbagi:** tidak semua channel live 24/7 bersamaan — grid "slot berbagi per klaster, jam prime" (sudah dipilih user 13 Nov sebagai grid 696 v1.0). 696 channel dalam 3–4 shift jam prime = ~175–230 stream serempak pada satu waktu.
4. **Relay terpusat + cache:** satu layanan relay audio (bukan per-stream yt-dlp) + cache manifest; kurangi badai yt-dlp (~2,5 core) dan bot-check.
5. **Antrean + orkestrasi:** dispatcher lintas node yang sesungguhnya (saat ini tidak ada — §3.5), penjadwalan start/stop otomatis, auto-failover antar node, health-check terpusat.
6. **Data layer:** evaluasi SQLite → Postgres (atau SQLite per node + sync) bila tulis konkuren menjadi bottleneck; pisah DB live (hot, high-write) dari DB publish.
7. **Observabilitas:** metrik per-stream (Prometheus/Grafana atau sepadan, ringan), alerting (Telegram/Discord), dashboard kesehatan armada.

### 4.4 Anggaran saat ini (konteks biaya)

Cap yang ditetapkan pemilik: **$34,67/bulan termasuk pajak 11% + one-time $2,66 setup** (Master ~$16,02 + VPS 8 $18,65). Box live tambahan + worker Render/API berada DI LUAR anggaran ini dan butuh persetujuan terpisah. **Agent penerima WAJIB menyertakan estimasi biaya per opsi arsitektur** (per box, per 100 stream, per bulan).

---

## 5. Fitur live yang harus ada (checklist target)

Dari permintaan pemilik + temuan diagnosis. Status: ✅ ada & aktif, 🟡 ada tapi bercelah / verifikasi pending, ❌ belum ada.

| # | Fitur | Status | Catatan |
|---|---|---|---|
| F1 | Buat jadwal live (form + validasi) | 🟡 | Form ada; validasi judul 100 karakter + checklist baru aktif 08:40 |
| F2 | Edit judul / deskripsi / thumbnail / tag + simpan | 🟡 | Editor ada; 3 celah diperbaiki S1–S5 (aktif 08:40, verifikasi pending) |
| F3 | Edit jadwal (ganti video, audio, relay, resolusi, waktu) | 🟡 | Desain lengkap; implementasi Fase 0–5 aktif 08:40/08:43, verifikasi pending |
| F4 | Hot-swap sumber tanpa matikan encoder | 🟡 | Via `request_source_swap` (FIFO O_RDWR); belum diuji live |
| F5 | Ganti video (lokal / GDrive / URL) | 🟡 | GDrive video ada; GDrive audio G1 baru aktif 08:43 |
| F6 | Ganti audio (file / GDrive / relay URL / playlist darurat) | 🟡 | Relay + playlist darurat ada; GDrive audio baru; DIRECT_URL masih diabaikan |
| F7 | Template jadwal + pakai ulang | 🟡 | CRUD template ter-commit (Fase 4); aktivasi 08:40 |
| F8 | Riwayat revisi + rollback 1 klik | 🟡 | Desain ada; implementasi Fase 0–5 |
| F9 | Bulk-edit + perubahan terjadwal (P2) | ❌ | Dibangun sebagai kode, aktivasi ditunda (keputusan pemilik) |
| F10 | Panel quick-fix saat stream bermasalah | 🟡 | Desain ada (P0); implementasi Fase 0–5 |
| F11 | Checklist pra-syarat + validasi sumber saat simpan | 🟡 | M1 + validasi probe; aktif 08:40 |
| F12 | Stream key: auto-key jujur + manual + status per channel | 🟡 | M2–M3 aktif 08:40; 8/13 channel MONA masih tanpa key |
| F13 | Penjadwalan start/stop otomatis + repeat + prioritas | ✅ | Ada (start/stop epoch, repeat_mode, priority) |
| F14 | Resolusi/bitrate per stream (720p/1080p) | 🟡 | Kolom ada; 1080p 4 stream parkir (item 7) |
| F15 | Monitoring per-stream + alert | ❌ | Hanya log + Sentinel verdict; tidak ada dashboard metrik/alerting |
| F16 | Audit trail (siapa buat/start/stop/edit, kapan) | ❌ | create/start tidak diaudit (temuan MONA) |
| F17 | Operasi massal (start/stop/edit banyak channel) | ❌ | P2, ditunda |
| F18 | Scoping operator (operator hanya lihat channel-nya) | ✅ | Live server-side; OWNER lihat semua |
| F19 | Failover antar node | 🟡 | Manual (skrip); otomatis belum ada |
| F20 | Playlist darurat per channel (≥30 trek) | ❌ | WULAN 3 trek, 114 6 trek |

---

## 6. Pertanyaan terbuka (hanya pemilik yang bisa jawab)

1. **Target jumlah pasti:** "ratusan" = 100? 200? 500? 696 (seluruh armada)? Dan dalam jangka waktu apa?
2. **Jam tayang:** semua channel 24/7, atau slot berbagi per klaster/jam prime (grid 696 v1.0)?
3. **Resolusi final:** 1080p semua (keputusan 18 Nov) atau 720p untuk sebagian demi kapasitas?
4. **Anggaran:** berapa $/bulan untuk box live tambahan? (Cap saat ini $34,67/bln untuk 2 box.)
5. **Prioritas channel:** channel mana yang live duluan saat scale-out? (Per monetisasi? Per operator? Per genre?)
6. **PARKED 278 + FAILED 21 Stage 2:** hapus, retry, atau biarkan? (Butuh keputusan operator.)
7. **RETIRE vs REVIVE stage2-driver** (PID 1338969, /tmp, IDLE 6d17h)?
8. **URL relay pengganti slot 034** (sumber dihapus lisensi)?
9. **Akar readonly (User=root → ubuntu):** setujui perbaikan + restart prep?
10. **Hermes pilot verdict + Agent AI Fase C (102 DRAFT repackage, 0 applied):** lanjut atau tahan?

---

## 7. Referensi

**Dokumen rencana + desain:**
- `engineering/RENCANA_PERBAIKAN_APLIKASI_2026-11-25.md` (v1.4+, §0.7 verifikasi Fase 2, §3 Fase 3) — rencana induk perbaikan
- `engineering/DESAIN_JADWAL_LIVE_EDITABLE.md` (14 bagian + 2 lampiran) — desain edit jadwal
- `engineering/PANDUAN_GUI_MODERN_MASJAVAS.md` — panduan UI (berlaku semua UI baru)
- `engineering/ADENDUM_EDIT_METADATA_LIVE.md` — 12 saran perbaikan editor metadata
- `engineering/DIAGNOSIS_MONA_LIVE_MANUAL_DAN_AUDIO_GDRIVE.md` (M1–M5, G1–G7, T1–T6, D1–D6) — diagnosis MONA
- `engineering/RENCANA_1080P_4_STREAM_2026-11-25.md` — rencana 1080p (item 7)
- `engineering/fase23_24_progress.md` — progres Fase 2.3+2.4
- `docs/engineering/PROTOKOL_VERIFIKASI_SENTINEL_2_1.md` — protokol verifikasi 7 hari Sentinel

**File kode kunci:**
- `api/routes/live_stream.py` — endpoint live (create/start/stop/pause/resume/priority/delete; guard duplikat :1531)
- `worker/live_supervisor.py` + `worker/live_stream_engine.py` — supervisor + engine (FIFO O_RDWR, `_run_loop_inner`)
- `worker/live_health_sentinel.py` — Sentinel (+132 di `e17a6c0`)
- `core/curation/global_audio_pool.py` — pool audio (F1 utime :153)
- `worker/gdrive_sync.py` — sync + prune 6H (import settings :17)
- `worker/ingestion.py` — ingestion + self-heal (`AUDIO_REMATERIALIZED` :4973, sweeper :6611)
- `tools/vps_disk_janitor.py` — janitor (step 1b TTL 1 jam)
- `worker/prep_worker.py` — Stage 2 breaker (`e1ae9d0`)
- `deploy/systemd/` — unit systemd (repo; DRIFT vs terpasang: web `--workers 2` vs 4; worker tanpa `--lane` vs `--lane publish` — JANGAN reinstall membabi-buta)

**Aturan kerja yang mengikat (dari pemilik):**
- Aplikasi SOLID: lantai 206 PUBLISHED/hari + 0 FAILED; setiap perubahan butuh verifikasi pre/post + rollback + jendela observasi; jangan "perbaiki" mekanisme yang belum terbukti.
- L1/L2 otomatis; L3 (schema/migrasi/auth/infra) butuh persetujuan; L4 dilarang. Restart/kill proses produksi butuh persetujuan eksplisit.
- Web restart maks 1×/15 menit. Live disentuh terakhir, satu stream terendah dulu.
- Rahasia (token, stream key, PIN, cookie) tidak pernah ditulis ke file/laporan/memory.
- Backup DB sebelum tulis langsung ke produksi. Tes isolasi = DB temp. Python selalu `/opt/masjavas-os/venv/bin/python`.
- SSH selalu `ssh -F /home/hatch/.ssh/config <alias>`; alias lama `masjavas-vps` MATI. SSH VPS 8 kadang "Connection closed by UNKNOWN port 65535" — tunggu 20–60 dtk lalu retry.
- Git shared-tree: commit+push dalam satu eksekusi; JANGAN PERNAH rebase branch bersama; patch file live langsung di VPS (backup dulu, diff sesudahnya) — jangan fetch-edit-scp.
- Verifikasi klaim hidup/mati di node yang bersangkutan (PID + RTMP + sentinel), bukan dari Master saja.
- Setelah broken pipe, SELALU cek state dulu sebelum mengulang (UPDATE berguard idempotent).
- Laporan memakai KPI + angka terukur; format channel: NO/baris + nama channel (NO n = baris n+1 di sheet EXPERT).

---

## Lampiran: timeline insiden 22–25 Nov 2026 (ringkas)

- **22 Nov:** 4 stream MONA mati 14 jam; dashboard tampil STREAMING (berdasarkan PID saja); sentinel ghost-kill loop bakar kuota; brake deployed 03:53 (ghost-kill berhenti, total 785).
- **23 Nov:** user putuskan VPS 8 "boros" karena kapasitas bukan bottleneck (saat itu benar — bottleneck-nya B10 + alur); Fase 1 perbaiki desync stream-key; 706 hidup lagi.
- **24 Nov 20:01–25 Nov 00:47:** klaster B10 00:47; audit run 01:00 (CTR fabrikasi + ledger readonly + kuota habis tanpa governor).
- **25 Nov 02:06:** klaster B10 ke-2. **04:35:** skor solid 66/100. **04:51–05:00:** rotasi PAT GitHub; 16 commit ter-push sebagai lini rebase oleh aktor TAK TERIDENTIFIKASI (atribusi: Fase 3 item 3.7).
- **25 Nov 05:00–05:30:** migrasi 6/6 live ke VPS 8 (2 bug failover Fase C ditemukan). **05:12–05:20:** analisa 6 akar VPS 8 (CPU/stream, prep berbagi box, badai yt-dlp, loop kematian, alur start buntu, telemetri salah).
- **25 Nov 05:30:** perintah "livestreaming dulu sampai tuntas". **06:11:** 15/15 kode livestreaming selesai (110 tes). **06:14:** aktivasi 1–5 disetujui.
- **25 Nov 06:17–06:34:** aktivasi 1–4 selesai (web restart, pull+restart live VPS 8, supervisor+worker Master, cap prep 200%). **06:25:** B10 pre-mortem aktif. **06:39:** mandat berkelanjutan ("perbaiki terus sampai tidak ada masalah").
- **25 Nov 06:54:** pilot 645 terverifikasi (pickup IDLE + broadcast + Sentinel LIVE). **07:04–07:14:** klaster ke-4 + ke-5 (5 encoder, 19 kematian/30 mnt) — prediksi IPv6 terkonfirmasi 07:04.
- **25 Nov 07:13:** 4/4 perbaikan (watchdog, embed, re-manifest, Stage 2) selesai + terverifikasi. **07:20–07:27:** aktivasi disetujui + terverifikasi (7/7, 0 kematian). **07:52:** B10 dinyatakan HILANG (0 kematian 30 mnt pasca-fix IPv4).
- **25 Nov 07:31–08:10:** adendum metadata (3 celah) + diagnosis MONA (keyless 8/13 → koreksi judul >100 karakter) + audio GDrive (G1–G7).
- **25 Nov 08:27:** Fase 2 lengkap (65 tes) + akar celah audio (janitor) terbukti. **08:31:** bug duplikat MONA (task dibuat, UI error 409). **08:36:** user perintahkan restart gabungan. **08:40:19:** Master web+worker+live+hermes-bridge restart. **08:43:30/08:44:16:** VPS 8 live+prep restart. **08:50:** restart gabungan TERVERIFIKASI (kedua node di 173adec, pin disengaja; rintangan: tree VPS 8 51.363 file root-owned, pull setengah-terupdate 43 file, merge terblokir 12 untracked; G1 terverifikasi aktif; MONA bisa Start 698; D1–D6 belum ditulis).
- **25 Nov 08:51–08:53:** Fase 3 LENGKAP (93 tes, 6 commit; atribusi push teridentifikasi; crash athrow terpin). Koreksi: web Master memuat c724c3a (guard 3.2 live). **08:53: task 698 ERROR di VPS 8** — video GDrive belum di-pre-copy (MONA belum bisa Start).
- **25 Nov 09:00:** pre-copy 698 selesai (video + audio, sha256 cocok di 3 tempat); status lokal kembali SCHEDULED; MONA bisa Start.
- **25 Nov 09:11–09:17:** D1–D6 aktif (merge 61fc49f; insiden migrasi skema crash-loop web ~6 mnt; fix 6f1b3c7; 8/8 streaming tak terganggu; publish 30/0).

---

*Akhir dokumen. Untuk pertanyaan tentang fakta di dokumen ini, rujuk ke file referensi §7 atau verifikasi langsung di server (baca-saja). Jangan berasumsi — ukur.*
