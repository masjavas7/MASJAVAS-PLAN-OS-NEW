# RENCANA PERBAIKAN APLIKASI MASJAVAS TUBE OS — v1.0

Tanggal: 25 November 2026, 05:02 WIB — diperbarui 07:00 WIB (v1.5)
Dasar: Laporan Status Aplikasi 25 Nov 04:35 WIB (skor 66/100) + verifikasi ulang 05:02 WIB
Status: DISETUJUI + BERJALAN — perintah user 05:20 implementasikan sampai tuntas;
PERUBAHAN PRIORITAS 05:30: LIVESTREAMING dulu sampai tuntas (daftar §0),
Fase 1–3 non-live DITUNDA sampai livestreaming beres.
Dokumen terkait: `ENGINEERING_READINESS_DASHBOARD.md` (Cycle 21), `STATUS_REGISTRY.yaml`,
`RUNBOOK_GANTI_PAT_GITHUB.md`

## 0. PRIORITAS 1: LIVESTREAMING SAMPAI TUNTAS (25 Nov 05:30 WIB)

Perintah user 05:30: utamakan perbaikan fitur livestreaming sampai semua
masalah selesai terverifikasi. Fase 1–3 non-live (Stage 2, CTR audit,
governor kuota, dst.) DITUNDA sampai daftar ini beres. Item yang terblokir
keputusan user diparkir, item lain tetap jalan.

### 0.1 Fakta migrasi (koreksi atas analisa 05:17)
- Migrasi SELESAI 05:30: 6/6 stream DI VPS 8 (live-01) — 706 PID 1132703,
  707 1567578, 114 1568005, 034 1569962, 088 1572102, WULAN 1574157;
  Master 0 ffmpeg, load Master 3,5→1,6. DB 6/6 STREAMING, assigned+lease
  live-01. Sentinel YouTube 05:25: 706/114/707/034 LIVE; 088+WULAN
  UNVERIFIED (broadcast "ready") tapi RTMP ESTABLISHED 6/6.
- KOREKSI: klaim "706/114/707 encoder mati" SALAH — ffmpeg hidup + sentinel
  LIVE terverifikasi langsung di VPS 8 (monitor Master-sentris buta: cek PID
  di Master). "034 MATI 05:08:53" adalah STOP FAILOVER TERKONTROL, bukan
  kematian. Saran "jeda migrasi" sudah tidak berlaku.
- KLASTER KEMATIAN ~05:32 (re-verifikasi koordinator): 707/114/088/WULAN
  mati dalam ~1 menit, kini RECONNECTING; 706 + 034 hidup. VPS 8 load
  8,33/8,13/7,07 (jenuh penuh); live ≈2,6 core, sisanya beban non-live.
- Pre-copy video WAJIB sebelum failover (supervisor start HANYA unduh bila
  path `gdrive://`; file tidak ada → ERROR, live_supervisor.py ~826).
  Skrip failover: `engineering/scripts/live_failover.py`.

### 0.2 Daftar livestreaming (urut prioritas)
P0: (1) Bug failover Fase C #1 + #2 — permanen, bukan workaround
(#1: stop-hook Master menimpa migration_state 'migrating'→'migrating_out'→NULL
di DB Master, run_migration_stop_hook ~2295; derive_orders hanya baca
'migrating' → start tidak pernah terbit; workaround:
`engineering/scripts/live_unwedge_migration.py`. #2: target tidak pernah
menghapus 'migrating_in' lokal → run_health_check mengecualikan task itu,
filter migration_state NULL ~1465); (2) heartbeat — apply_task_status_report
wajib menulis last_heartbeat (~517) + label STREAMING kadaluarsa bila hb
>5 mnt; (3) readonly-DB di reconnect writes (koneksi DB supervisor live);
(4) feeder audio abadi; (5) cookie yt-dlp (ekstraksi manifest gagal saat
reconnect).
P1: (6) dispatch start lintas node ATAU auto-pickup IDLE oleh supervisor
node tujuan; (7) guard double-submit di create/start live; (8) throttle
guard_blocked_start event (168/169 baris node_events, tiap ~5 dtk);
(9) koreksi max_streams 70→~10–12 + perbaiki rumus skor (headroom stream
dibatasi CPU; network jangan penuh saat tx≈0); (10) isolasi prep dari live
(cgroup/batas core atau pindah prep); (11) throttle relay yt-dlp (cache
manifest, batasi konkurensi); (12) instrumentasi B10 (034 exit 0 bersih
masih misteri T7).
P2: (13) 720p→1080p 4 stream; (14) playlist darurat WULAN (2 trek);
(15) error_message basi dibersihkan.

### 0.3 Aturan tambahan fase livestreaming
- Semua 6 live kini DI VPS 8 — perbaikan di sana langsung berdampak ke
  siaran. Live disentuh TERAKHIR, satu stream prioritas terendah dulu,
  dengan jendela observasi. Restart/kill proses live TETAP butuh
  persetujuan eksplisit user.
- Jangan sentuh stream yang sedang LIVE tanpa rencana rollback.
- Verifikasi SETIAP klaim "hidup/mati" LANGSUNG di node bersangkutan
  (PID + ss RTMP + sentinel YouTube), jangan dari Master saja.

### 0.4 Status livestreaming per 25 Nov ~05:57 WIB (terverifikasi koordinator)
- SELESAI KODE + TERVERIFIKASI (commit `e2779ed`, 11 file +1231/−83,
  ter-push; tes 74/74 SAYA jalankan ulang: bundle 24 + fase_c 29 + fase_c2 9
  + relay_throttle 8 + b10_b13 4): (1) failover #1 (SET 'migrating',
  live_supervisor.py ~2487) + #2 (health check sertakan 'migrating_in',
  3 situs filter); (2) heartbeat COALESCE write (node_failover.py:529) +
  label STALE >5 mnt (API + dashboard + sweep supervisor, tanpa mutasi
  status); (4) feeder abadi (restart inner ≤5x); (5) cookie file Netscape
  dulu lalu profil Chrome; (6) start lintas node PENDING_PICKUP +
  pickup IDLE tiap 15 dtk; (7) double-submit 409 (10 mnt) + frontend
  disable tombol; (8) throttle guard 900 dtk + filter node watchdog/
  reconcile/auto-heal; (12) B10 ring buffer + PRE-MORTEM + ringkasan badai
  manifest; (11) hook throttle 3 call site (engine :2448/:2512/:2589).
  AKTIVASI menunggu restart (keranjang persetujuan §0.5); 0 restart
  dilakukan sejauh ini.
- SELESAI + TERVERIFIKASI SEBELUMNYA: (9) max_streams=12 di DB Master +
  rumus skor v2 di kode (`31f3ff0`, endpoint menunggu restart web);
  (10) diagnosis isolasi prep terverifikasi (prep root, CPUQuota 4 core;
  penerapan menunggu persetujuan); (13) rencana 1080p terverifikasi akurat
  (707+088 DB-only, 034+WULAN butuh sumber 1080p + restart encoder;
  eksekusi menunggu headroom load1 <4 + kematian <2/jam);
  (14) playlist darurat WULAN SUDAH 3 trek valid — target tercapai tanpa
  perubahan; (15) error_message 6/6 NULL di Master; encoder yatim: TIDAK ADA
  (6/6 cocok ps = ss:1935 = DB).
- BERJALAN: (3) readonly-DB — DIAGNOSIS S4 TERVERIFIKASI (journal VPS 8
  24 jam = persis 2 kejadian: 24 Nov 19:12:47 ledger audit + 22:56:30
  reconnect live #1795321099635; prep/telemetry User=root, live User=ubuntu,
  watchdog root; divergensi DB TERKONFIRMASI: live-01 max_streams=70 lokal
  vs 12 Master, heartbeat lokal SEGAR — readonly TIDAK terkait divergensi,
  lintas-node hanya HTTP). Fix `adc2b13` CACAT (probe BEGIN IMMEDIATE lolos
  di koneksi readonly — TERBUKTI koordinator) → **DIPERBAIKI S4b, commit
  `b6ceb80` (06:09, ter-push): probe tulis-nyata (CREATE TABLE nama-unik +
  DROP + ROLLBACK, skema utama) + `tests/test_readonly_retry.py` 525 baris.
  TERVERIFIKASI koordinator 06:12: 12/12 tes S4b saya jalankan ulang;
  repro v1: T2 raise diagnostik 3,86 dtk (dulu tidak raise), T4 pemulihan
  transien 0,85 dtk dengan INSERT nyata (dulu gagal mentah); repro v2:
  6/6 attempt probe-gagal lalu raise diagnostik. Regresi 24 passed +
  1 gagal pre-existing (fixture `test_live_metadata_editor.py` tanpa kolom
  assigned_node_id — 0 kemunculan, terverifikasi; S4b buktikan via stash).
  KOREKSI: varian -shm koordinator (holder seproses) BUKAN bukti cacat —
  koneksi seproses memakai ulang node -shm RW holder (VFS unix), jadi
  benar-benar bisa tulis; tes (b) S4b memakai holder subprocess.
  Target 0 readonly 7 hari tetap butuh deploy + restart + observasi.
- KLASTER KEMATIAN LAGI ~06:09 WIB (produksi, kode lama — BUKAN akibat
  lane): 5/6 encoder restart dalam ~90 dtk (PID baru 1618832–1619970);
  2 stream kena RestartAlert "4x dalam 60 menit" (Telegram terkirim);
  6/6 STREAMING lagi ~06:11. TIDAK ADA baris PRE-MORTEM di journal →
  `08e4b8a` belum berjalan di service live VPS 8 (belum restart) — celah
  bukti B10 masih terbuka sampai aktivasi. TEMUAN BARU: sumber relay
  #1795502710790 adalah video YouTube yang DIHAPUS ("removed due to a
  contractual obligation with a music licensor", kizfrmykOkM) — penyebab
  konten, bukan cookie; butuh ganti sumber (keputusan operator, §0.5 item 9).
  Load VPS 8 membaik: 4,76/6,24/7,23. Gate 1080p (kematian <2/jam) BELUM
  terpenuhi (4x/60 mnt di 2 stream).
- BATAS DIKETAHUI: DB Master masih basi soal live (5 STOPPED + 1
  RECONNECTING vs VPS 8 6 STREAMING — sisa stop-hook migrasi; push hanya
  {task_id, status, pid}). Label STALE (e2779ed) akan membuat kebasian
  terlihat pasca-aktivasi; perluasan cakupan sync = tindak lanjut terpisah.
- **STATUS AKHIR FASE KODE (06:15): 15/15 item livestreaming SELESAI KODE +
  TERVERIFIKASI.** Yang tersisa: aktivasi (persetujuan user, §0.5) +
  bukti produksi pasca-restart + eksekusi 1080p + ganti sumber relay
  #1795502710790. Skor produksi tetap ~66/100 (belum ada yang aktif;
  max_streams=12 + error_message bersih sudah efektif via DB).
- KOREKSI atas laporan S4: VPS 8 di `08e4b8a`, BUKAN `ed0ff04` (pull yang
  dibutuhkan: 08e4b8a → adc2b13 + commit S4b). RUNBOOK BARU: jangan pernah
  chmod file DB live jadi restriktif (mode diwariskan ke -shm/-wal → racun
  persisten).
- LIVE ~05:57: 6/6 encoder VPS 8 + 6 RTMP ESTABLISHED + DB 6 STREAMING;
  load VPS 8 6,37/6,94/7,77 (turun dari puncak 13,94). VPS 8 masih di
  `08e4b8a` (belum pull `e2779ed`/`adc2b13` — disengaja, aktivasi menunggu
  persetujuan).
- CELAH TERSISA (dari S3): task SCHEDULED masa depan di node asing belum
  punya penerbit order saat waktunya tiba; task FAILED di node asing butuh
  intervensi manual; `_ensure_public_live_broadcast` belum diverifikasi
  sebagai sumber spam.

### 0.5 Keranjang persetujuan aktivasi (kumulatif — JANGAN eksekusi tanpa user)
STATUS 07:00 WIB: item 1–6 SELESAI + TERVERIFIKASI (lihat §0.6); item 7–9 TETAP PARKIR.
1. Restart web Master 1× (aktifkan: rumus skor v2 + routes/template bundle
   e2779ed — heartbeat STALE, PENDING_PICKUP, 409 double-submit).
2. `git pull` di VPS 8 + restart `masjavas-live.service` VPS 8 (aktifkan:
   hook throttle ~4 core + failover #2 + pickup IDLE + feeder + cookie +
   B10 + guard throttle + readonly fix). SIAP — commit S4b `b6ceb80` sudah
   di `origin/main`; satu restart mengaktifkan semuanya.
3. Restart supervisor Master (aktifkan: failover #1 + stale sweep) +
   restart web/worker Master (memuat connection.py baru).
4. Cap prep VPS 8 ke 2 core TANPA restart (`set-property CPUQuota=200%`,
   efek langsung + persisten; revert mudah).
5. Pre-copy audio depot 109 MB Master→VPS 8 (prasyarat pilot start 645)
   + refresh file cookie `/opt/masjavas-os/data/credentials/yt_cookies.txt`
   + perbaiki izin (saat ini root 0600, tak terbaca supervisor).
6. Pilot start task 645 (IDLE, REGGAE TROPIS, live-01) — HANYA setelah 1–5.
7. 1080p: 707 + 088 update DB (tanpa restart); 034 + WULAN ganti video +
   restart encoder (SETELAH headroom). + keputusan provisioning SSH key
   Master→live-01 untuk pre-copy in-script (alternatif: manual +
   `--skip-precopy`, tanpa persetujuan).
8. [BARU, dari S4] Perbaikan akar permanen readonly (persetujuan terpisah):
   samakan uid pembuka DB jadi ubuntu — `User=root` → `User=ubuntu` di
   `masjavas-prep.service` + `masjavas-node-telemetry.service` (keduanya
   terverifikasi root) + `chown` dulu subdir root-owned di VPS 8
   (`data/loopgen/`, `data/global_audio_pool/`, `data/credentials/`) +
   restart prep. Menghilangkan seluruh kelas race -shm lintas-uid.
   CATATAN: restart supervisor live — apakah encoder ffmpeg yang jalan
   dipertahankan atau ikut reconnect — BELUM terkonfirmasi lane live;
   bukti lama (20 Nov): restart live ~2 mnt downtime + auto-reconnect.
9. [BARU] Ganti sumber relay task #1795502710790 — video sumbernya dihapus
   YouTube (music licensor, kizfrmykOkM); manifest extraction akan SELALU
   gagal untuk sumber ini. Butuh keputusan operator: URL pengganti.

### 0.6 Status aktivasi per 25 Nov ~07:00 WIB (terverifikasi koordinator aktivasi)
- Item 1 SELESAI 06:17:22 — restart web Master (PID 2694043→3246819,
  health 200 pre/post, 0 error journal; jeda 5,7 jam sejak restart
  sebelumnya, aturan 1×/15 mnt terpenuhi; skor v2 + bundle dimuat).
- Item 2 SELESAI 06:25:07 — pull VPS 8 fast-forward 08e4b8a→b6ceb80
  (3 commit, 0 lokal ahead) + restart masjavas-live.service
  (PID 1099269→1631885). TERBUKTI: KillMode control-group mematikan ke-6
  encoder saat restart (6 PRE-MORTEM orphan 00:25 CET); supervisor baru
  reconnect 6/6 RTMP dalam ~3,5 mnt. PRE-MORTEM + korelasi badai
  manifest aktif; 0 error sejak restart.
- Item 3 SELESAI 06:31:31 — restart supervisor Master (2106348→3261459)
  + worker (306566→3261456); 0 error; publish 8 PUBLISHED 0 FAILED;
  task RENDERING 5202 kembali QUEUED via heal (tidak hilang).
- Item 4 SELESAI ~06:33 — cap prep VPS 8 400%→200% TANPA restart
  (MainPID 1575347 tidak berubah; drop-in persisten di
  /etc/systemd/system.control/masjavas-prep.service.d/50-CPUQuota.conf —
  set-property menulis ke system.control, bukan /etc/systemd/system/);
  cgtop prep 86% saat cek, live 6/6 tidak terganggu.
- Item 5 SEBAGIAN SELESAI — audio DONE: ydxp0BAqzjU.m4a 156 MB
  (sha256 fd3ff51d… cocok persis Master vs VPS 8, rsync --partial dua hop
  setelah pipe SSH putus di 137 MB; nama panjang sesuai audio_filename
  task 645; ter-resolve list_available_audio_files()). Cookie SEBAGIAN:
  izin root 0600→ubuntu 0600, BOM UTF-8 dihapus (backup
  yt_cookies.txt.bak-20261125), parse Netscape 10/11 entri valid
  s.d. 2027–2028 — TAPI sesi login kedaluwarsa: uji manifest gagal
  3 jalur × 2 URL relay (bot-check YouTube sistemik). Refresh butuh
  login operator via "Login YouTube 1-Klik" (core/curation/cookies_vnc.py
  capture_cookies_via_cdp). Dampak terbatas: fallback playlist darurat
  terbukti jalan saat reconnect 06:18. Angka "109 MB" koordinator
  sebelumnya tidak cocok dengan apa pun yang ditemukan (file ini 156 MB).
- OBSERVASI pasca-aktivasi (~06:50): heartbeat VPS 8 2–11 dtk 6/6
  (dulu beku sejak 23–24 Nov); Master hb 112–131 dtk; PID cocok lintas DB.
  guard_blocked_start total 178, 0 baris baru/30 mnt (dulu spam ~5 dtk)
  — throttle TERBUKTI. Load VPS 8 5,80/6,60/7,31 (dari puncak 10,45–13,94);
  cgtop live 170%, prep 11%. Publish 25 Nov 06:50: 10 PUBLISHED,
  199 QUEUED, 1 RENDERING, 30 SKIPPED_AUDIO_MISSING, 1 AWAITING_
  MONETIZATION, 0 FAILED — lantai baseline utuh.
- PILOT 645 SELESAI + TERVERIFIKASI 06:54–06:58 — pickup IDLE 00:54:15
  CET → mengudara 00:54:19 (PID 1658850, 720p) → broadcast publik
  ZaJ8UzdEJ8g dibuat 00:55:42 → push Master STREAMING. Encoder mati
  90 dtk kemudian di klaster B10 00:55 CET (exit 146); reconnect 1/5
  sukses 00:56:15 (PID 1660225). Sentinel 00:58:14: verdict LIVE,
  HEALTHY (lifeCycleStatus=live). 7/7 STREAMING, RTMP 7/7. (Task 645
  hanya ada di DB Master — pilot memakai salinan baris via JSON piped
  Master→VPS 8 + INSERT lokal; backup DB pre645 dibuat; file JSON
  berisi stream key sudah dihapus di kedua server.)
- **B10 — AKAR TERIDENTIFIKASI (bukan lagi misteri): RTMP IPv6 tidak
  stabil.** 12 kematian pasca-restart dalam 3 klaster (00:27, 00:36,
  00:46 CET; 4/klaster) + klaster 00:55 (5 mati, termasuk pilot 645).
  Bukti: (a) ss 00:58 — 2 encoder IPv4 (142.251.14.134) selamat sejak
  06:28 melewati 4+ klaster vs 5 encoder IPv6 (2a00:1450:4001:c0f/c17/
  c25::86) mati berulang; (b) pre-mortem: tx MEMBEKU + CPU 0,0%
  30–130 dtk sebelum mati → stall socket RTMP → timeout exit
  146/152/187/224; (c) semua pakai rtmp://a.rtmp.youtube.com/live2
  (hostname sama; DNS kembalikan A + AAAA, ffmpeg pilih per koneksi);
  (d) 088/WULAN MATI 06:18 (dulu IPv6) dan SELAMAT sekarang (IPv4) —
  yang selamat adalah koneksinya, bukan channelnya; (e) tanpa timer/
  cron lokal ~10 mnt; (f) ping: jitter IPv6 lebih tinggi (mdev 46–59ms
  vs 33ms IPv4), 0% loss di luar klaster (stall intermiten).
  FIX: paksa IPv4 untuk RTMP (resolve AF_INET per start/reconnect,
  fallback hostname bila gagal) — worker kode+tes di-spawn 07:00,
  deploy masuk keranjang persetujuan Fase 1 (butuh restart live).
  Prediksi TERKONFIRMASI 07:04–07:05 (konfirmasi ke-5 berturut-turut):
  klaster membunuh PERSIS 5 encoder IPv6 (1660176/1660225/1660552/
  1660634/1660777, exit 152/187/None) dan menyisakan 2 IPv4
  (1633562/1634120, etime 38+ mnt). FIX SELESAI + TERVERIFIKASI
  INDEPENDEN: commit d5e4764 (worker/rtmp_ipv4.py baru 109 baris +
  live_supervisor.py +9/−2: call-site start_stream:992 mencakup
  start+reconnect; matcher DuplicateReaper jadi host-agnostik) —
  tes baru 11/11 + regresi b10_b13 4/4 SAYA jalankan ulang (unittest;
  pytest TIDAK ADA di VPS 8 — Master punya pytest 9.2.0). DEPLOY
  MENUNGGU PERSETUJUAN (keranjang Fase 1): pull VPS 8 + restart
  masjavas-live.service saja; rollback git revert + restart; risiko:
  7 encoder mati saat restart, reconnect ~3,5 mnt. CATATAN: checkout
  Master sudah di d5e4764 (disk mendahului kode jalan b6ceb80 sebanyak
  6 commit incl. live-edit/GUI lane migrasi) — restart web/worker
  Master berikutnya akan memuat commit-commit itu.
- FASE 1 SELESAI (kode+tes+deploy): B10 IPv4 d5e4764 TER-DEPLOY +
  aktif (7/7 RTMP IPv4); audit 1.2/1.3 8fdfcbb terverifikasi 16/16
  (run live 19:00 malam ini). 1.1 Stage 2 = lane migrasi (e1ae9d0,
  ter-deploy 07:22). 1.4 B10 = observasi pasca-fix. LANJUT FASE 2
  (2.1–2.5) — worker di-spawn setelah pembukuan ini.
  Lane chat migrasi (JANGAN duplikasi): watchdog progres encoder,
  ingat penolakan embed, re-manifest cepat, diagnosis+perbaikan Stage 2,
  fitur edit jadwal live.
- Skor solid: BELUM diukur ulang — live belum bisa naik selama klaster
  B10 masih jalan; ukur ulang setelah fix IPv4 ter-deploy + observasi.
- INSIDEN PUBLISH 07:06 (lantai 0 FAILED jebol 1x): task 5142 NO 267
  (Rock Kenangan) FAILED "database is locked" di transisi
  DOWNLOADING_ASSETS→RENDERING (belum ada video, kuota direfund via
  release_quota_units). AKAR: "database is locked" TIDAK ADA di daftar
  transient _is_infra (worker/autonomous_daemon.py:1201-1205) sehingga
  auto-retry 1x tidak terpicu. Cakupan 25 Nov: 27 error lock (web 22,
  worker 3, live 2); lock dipegang 10–30+ dtk (UPDATE retry koordinator
  gagal timeout 30 dtk, sukses timeout 90 dtk). RETRY MANUAL SELESAI:
  backup pre5142retry + UPDATE → QUEUED/AUTO_RETRY, jadwal 07:19:27 WIB
  (replikasi persis jalur auto-retry kode). Fix kelas → Fase 2.5
  (tambah "database is locked" ke _is_infra + audit penulis DB).
- INSIDEN KE-2 07:15: task 5196 NO 42 FAILED "database is locked" —
  TAPI video SUDAH terbit 07:10:44 (ID f45aF9sRBng; bukti: log PUBLISHED
  + ledger VIDEO_INSERT 07:07:24 + thumbnail 07:10:43 + youtu.be 303).
  Gagalnya di tulis status PASCA-upload. DIPULIHKAN 07:27: backup +
  UPDATE → PUBLISHED (tanpa upload ulang, hindari duplikat) + entri
  log MANUAL_RECOVERY (tertulis 2x — percobaan pertama terputus broken
  pipe tapi ternyata sudah jalan; pelajaran: broken pipe ≠ gagal,
  selalu cek state sebelum ulangi; UPDATE berguard, append log belum).
  FAILED hari ini kembali 0; PUBLISHED 12 (07:30). Vault NO 42
  (MV-4375018191F9F7AC): used 3.300 = 5196 (1.650, terpakai sah) +
  5078 (1.650, reserve bocor? — 5078 di-heal ke QUEUED tanpa release;
  verifikasi ke Fase 2.5).
- RESTART 07:21:08 (worker+live Master) + 07:22:42 (live+prep VPS 8)
  = AKTIVASI 4 FIX LANE MIGRASI yang DISETUJUI user di chat migrasi
  (watchdog ed81681, embed d5c8e8c, re-manifest 1188e9b, Stage 2
  e1ae9d0) — BUKAN restart liar. Atribusi terverifikasi: sudo log
  ubuntu `systemctl restart masjavas-os-worker.service
  masjavas-live.service` 07:21:08 + MEMORY.md chat migrasi. Dampak:
  4 tugas + 1 job ingestion di-heal ke QUEUED (5078 RENDERING dll.);
  harmonizer 07:21:55 selaraskan 241 saluran; quotaExceeded sync
  3 channel 07:22:32 (vault habis, non-fatal). "Klaster 07:23"
  yang sempat tercatat = FASE RECOVERY restart ini, BUKAN klaster B10
  (koreksi: jangan hitung sebagai kematian B10).
- FIX IPv4 B10 TER-DEPLOY + AKTIF + **TERVERIFIKASI HILANGKAN KLASTER**:
  d5e4764 ancestor HEAD 23de8d1; ss 07:28: 7/7 RTMP IPv4
  (142.251.x.x / 192.178.183.134), 0 IPv6; 7/7 STREAMING. OBSERVASI
  30 MNT 07:52: **0 PRE-MORTEM non-orphan** sejak restart 07:22:42
  (~3,3 siklus klaster — pola lama akan membunuh ~15 encoder),
  0 RTMP IPv6, load 5,91. Klaster B10 DINYATAKAN HILANG. KOREKSI
  atas §0.6: "klaster 07:23" bukan klaster (recovery restart).
  Sinyal campuran lane migrasi (645 EOF audio/FIFO vs 034 Connection
  reset) TIDAK terwujud pasca-fix — 645 selamat 30 mnt; EOF
  kemungkinan gejala sekunder stall yang sama, bukan mekanisme ke-2
  (tetap pantau; bila 645 mati sendiri di masa depan, selidiki lagi).
- FASE 1.2/1.3 SELESAI + TERVERIFIKASI: commit 8fdfcbb (CTR nyata
  dari metrik reach Analytics + governor 1000 unit/vault/hari-PT,
  per-call check, marker AUDIT_CAP/AUDIT_QUOTA_EXCEEDED; 5 file,
  914+/79-). Tes baru 16/16 SAYA jalankan ulang (pytest Master —
  pytest TIDAK ADA di VPS 8); regresi tetangga 12 lolos + 2 gagal
  PRE-EXISTING (gagal juga di commit induk d5e4764, dibuktikan via
  worktree). Diagnosis: CTR di-hardcode 0 (7027a7f) setelah fabrikasi
  views*22/0.0455 (5191621) dihapus. Ledger: audit hanya ~1–5 unit/
  vault (BUKAN pencuri kuota — hipotesis audit-makan-kuota GUGUR).
  Run audit 19:00 malam ini = verifikasi live (timer oneshot pakai
  kode baru tanpa restart).
- HISTOGRAM JADWAL 07:25 (198 QUEUED): 07–13: 61 tugas; gap 14–15
  (jendela reset PST 15:00); 16–23: 137 tugas; hanya 3 tepat 19:30.
  Proyeksi kasar: pagi feasible (~9,4/jam vs kapasitas ~12,8/jam);
  sore butuh ~17/jam (di atas kapasitas) → hari ini kemungkinan
  ~170–200 PUBLISHED (bisa di bawah lantai 206 — proyeksi, bukan
  pasti; kapasitas kasar 3 slot × ~14 mnt/tugas).
- CATATAN: unit file prep di disk CPUQuota=400% adalah NILAI ASLI
  (bukan perubahan tak teratribusi) — efektif 200% berasal dari
  drop-in system.control item 4. Checkout Master kini 776648a
  (lane live-edit commit cepat); kode jalan worker/live = 23de8d1
  (restart 07:21), web = b6ceb80 (restart 06:17; UI live-edit belum
  aktif — keputusan lane migrasi).

### 0.7 Verifikasi Fase 2 per 25 Nov ~08:00 WIB (terverifikasi koordinator)

- 2.1 + 2.2 DITERIMA — commit `e17a6c0` (10 file, +1095/−52),
  tes baru 13/13 + 18/18 saya jalankan ulang (Master, pytest).
  2.1: hitungan offline kini di-reset saat UNVERIFIED (cacat lama:
  False→UNVERIFIED→False tetap membunuh) + logging terstruktur
  probe/kill; breaker TIDAK dilonggarkan. 2.2: healer ingestion
  ternyata TIDAK PERNAH JALAN sejak 23 Nov 20:39 — worker Master
  `--lane publish`, tick healer mensyaratkan lane all/ingestion
  (0 baris `[Failed Retry Healer]` di journal sejak 24 Nov).
  Tick healer + deficit sweeper kini tanpa gate lane (DB-only).
  #409 (FAILED, attempts=0) auto-retry pasca-aktivasi; #542
  (COMPLETED_PARTIAL, UNPAIRED) bukan kandidat healer — jalur
  sweeper/pairing. Protokol verifikasi 7 hari:
  `docs/engineering/PROTOKOL_VERIFIKASI_SENTINEL_2_1.md`.
- 2.5 DITERIMA — commit `0eb8bab` (13 file, +833/−246), tes baru
  12/12 saya jalankan ulang. "database is locked" masuk `_is_infra`
  + retry decorator lintas proses + release/reconcile ber-retry.
  KOREKSI: heal SUDAH me-release reservasi via
  `reconcile_vault_quota_with_ledger` — hipotesis bocor 5078
  terkoreksi (celahnya: reconcile sendiri tanpa retry).
  Skrip patch idempotent: `engineering/scripts/patch_dblock_fase25.py`.
- 2 GAGAL `TestHealerBlockedResume` (celah_abc) TERBUKTI
  pre-existing — gagal identik di worktree induk e17a6c0
  (b2bbe2d). 2 gagal `test_live_fase_s_sentinel` di checkout
  Master disebabkan PERUBAHAN TAK TER-COMMIT lane live-edit
  (lolos di worktree segar HEAD 8c58229) — BUKAN regresi Fase 2;
  lane live-edit wajib memperbaiki sebelum commit.
- 2.3 + 2.4 DITERIMA 08:27 WIB (worker pengganti, 5 commit:
  `1523da1` F1+F2a+F2b, `9bb9f1e` F3+F6, `f1f3e90` F4+F5,
  `f9bb5bf` 2.4, `edbeb1f` fix F6) — tes baru 22/22 saya jalankan
  ulang. AKAR CELAH AUDIO: janitor hapus audio in-flight —
  terbukti deletion_audit NO 422: 11 file
  JANITOR_UNPAIRED_QUARANTINE_CLEAN 05:08:33 + WORKSPACE_PRUNE_6H
  05:36:43 saat job masih stage AUDIO; hardlink pool warisi mtime
  lama (21–23 Nov) → janitor TTL 1 jam hapus. Fix: touch mtime
  saat materialize, janitor lindungi job RUNNING, self-heal
  re-materialisasi, sweeper smallest-gap-first, status baru
  SKIPPED_VIDEO_MISSING (audio-murni 18/hari, bukan 30).
  Bonus: NameError `settings` di prune 6H (skip-check tak pernah
  jalan) diperbaiki. Koreksi topologi: risiko kini lebih rendah
  (ingestion di prep_worker VPS 8 tanpa janitor sejak 07:21),
  tapi unit repo tanpa `--lane` akan kembalikan lane ALL bila
  reinstall. 2.4: R52 kedaluwarsa — semua service sudah systemd
  kecuali hermes-bridge Restart=on-failure; HANYA 1 detached:
  PID 1338969 `/tmp/stage2-driver-continuous.py` (6d17h, CPU
  1:36, IDLE — kemungkinan superseded; keputusan RETIRE vs
  REVIVE di keranjang). 6 gagal tes lain terbukti pre-existing
  (worktree 36d1288). Temuan #3 (crash `athrow` slot 405/409)
  belum terpin — perlu reproduksi, tidak ditebak.
  **FASE 2 LENGKAP (kode+tes): 2.1, 2.2, 2.3, 2.4, 2.5 semua
  DITERIMA.** Deploy gabung keranjang Fase 2.
- Keranjang persetujuan Fase 2 (belum dieksekusi): Master pull +
  restart web+worker (2.1/2.2/2.5) + kedua node pull + restart
  live (sentinel + live_quota_guard). Verifikasi pasca-aktivasi:
  journal `[Failed Retry Healer]` muncul; #409 FAILED→PENDING
  ≤1 jam; grep "database is locked" target 0 (7 hari).
- pytest 9.2.0 kini TERPASANG JUGA di VPS 8 (diinstal worker
  audit) — catatan AGENTS.md sudah dikoreksi.

### 0.8 Verifikasi Fase 3 per 25 Nov ~08:50 WIB (terverifikasi koordinator)

- 3.1 + 3.7 DITERIMA — tanpa commit repo (skrip + laporan workspace).
  3.1: CONNECTED 243/UNPAIRED 464; AZIZ 240 (51,7% backlog); 460/464
  UNPAIRED tanpa vault (hanya 3–4 P1 siap pairing); 60 vault, 2 over-cap
  (6 ch), 54 slot kosong (~2,8 minggu pada +20/minggu). Widget cocok;
  fallback hardcode 327 usang (dashboard.html:177). Generator +
  daftar prioritas + spek monitoring mingguan di workspace/engineering/.
  3.7: PELAKU TERIDENTIFIKASI — lane Gravity ("Gravity PC executor",
  javaswahid <dev@masjavas.com>, PC 180.254.108.116 via SSH root):
  15 commit rebase 04:56:40–41 committer javaswahid, reflog push
  04:57:01, receive.denyCurrentBranch=updateInstead, sesi SSH tutup
  tepat detik push; diff ecde0ef↔87ab7cea kosong (no-op isi).
- 3.2 DITERIMA — commit `668130c` (5 file, 742+/3−), tes 49/49
  saya jalankan ulang (13 baru + 36 regresi). Pilot judul 5/5 +
  QA 18/20 bersih (DB temp, Rp 20,52; 2 flag false positive angka
  judul lagu). Guard double-submit (409 + idempotent + kunci UI).
  **Fase C apply: TIDAK ADA KODENYA** (grep nol); rollback tak lengkap
  (old_description kosong 102/102, tanpa kolom old_tags, thumbnail
  backup 0/102). Paket keputusan Fase C (98 eligible, 10/batch,
  4.900 unit) di workspace. Produksi tak tersentuh (DRAFT tetap 102).
- 3.3 DITERIMA — commit `1005ced` (9 file, +1286/−21), tes 77/77
  saya jalankan ulang (20 baru + 57 regresi). F4 prelive + HITL-2 +
  F5 handoff (gate default OFF; graph Fase 1 tak diubah). Koreksi
  data: event 20 CANDIDATE + 1 SCHEDULED + 1 REJECTED (bukan 21
  SCHEDULED — seed 24 Nov me-reset state); 4 event malam ini belum
  punya pack/schedule 25 Nov + media 63 pending. Aktivasi butuh:
  keputusan F4, channel event (masih TBD) + stream key + sesi Master.
- 3.4 DITERIMA — commit `4d680b3` (6 file baru, 1.956 baris, 0 file
  existing diubah), tes 52/52 baru + 294/294 regresi -k hermes
  saya jalankan ulang. Konsumen inbox dibangun, loop TIDAK
  diregistrasi di api/main.py (aktivasi menunggu owner). Dogfood 5
  dokumen pilot lolos validator. Uji live-staging cabut key BLOKIR
  (butuh key uji + jendela 30 mnt); verdict pilot menunggu owner.
  Koreksi: hermes_tasks kosong dijelaskan penghapusan Fase 0.3
  (backup 05:21 ada), bukan misteri. DB live: mode manual, key VALID
  (60 model), inbox 1 DRAFT / 0 APPROVED.
- 3.5 DITERIMA — commit `c724c3a` (26 file), tes pemindai 3/3
  saya jalankan ulang. 0 rahasia aktif di working tree.
  **5 PIN operator TERBUKTI SAMA dengan produksi** (hash cocok —
  verifikasi independen, hanya boolean dicetak) → wajib rotasi.
  PAT lama MASIH AKTIF (histori ecae97e) sampai user cabut.
  Proposal deploy key SSH di docs/DEPLOY_KEY_SSH_PROPOSAL.md.
- 3.6 DITERIMA — tanpa commit repo (skrip workspace). Restore drill
  PASS (saya run ulang: 5,5 dtk, integrity ok, 12 tabel cocok,
  FK 312 pra-eksis). 3 alert siap (disk/kuota/ghost-kill, spek cron)
  + panduan UptimeRobot (/healthz 200). Temuan: sentinel_kill_history
  live berisi 29 baris ID uji (kill_at 05:36–07:54 WIB — koreksi klaim
  worker 08:26–08:28), tulisan tes era Fase 2.1 ke DB live; backup
  04:25 = 0 baris (terkonfirmasi). Dampak fungsional NIHIL (query
  per-task_id) tapi perlu dibersihkan (tulis DB → keranjang).
- Crash athrow TERPIN — commit `87e3837` (tes regresi 5/5 saya
  jalankan ulang) + `05b9380` (koreksi hitungan). Mekanisme TERBUKTI:
  retry loop cea60b8 yield 2× pasca-athrow (RuntimeError menutupi
  lock asli). Fix e14e6e2 SUDAH LIVE (0 athrow 25 Nov; lock 88×
  tanpa athrow). Tanpa aksi baru.
- Checkpoint 19:00/19:30 SIAP — skrip verify_audit_1900.py (8 cek)
  + verify_task_5142.py (5 cek) + protokol; saya run ulang --pre
  (exit 0) + --dry-run (5 FAIL tepat: C2/C3/C4/C5/C7). **GAP-1:
  timer audit 18:00 UTC = 01:00 WIB — TIDAK ada run 19:00 WIB**
  tanpa trigger manual (POST /api/audit/run-phase0 OWNER atau
  systemctl start). 5142 QUEUED, jadwal live 20:00 (harmonizer
  geser 19:30→19:35→20:00), verdict 20:30. Cron verifikasi one-time
  dibuat: 19:05/19:20 audit, 20:00/20:30 5142 (goal-owned).
- **KOREKSI AKTIVASI — guard 3.2 SUDAH LIVE di web Master:**
  reflog Master 08:19–08:42 hanya commit (TANPA checkout) — pin
  173adec hanya berlaku di merge VPS 8; restart web 08:40:19 memuat
  working tree c724c3a (668130c commit 08:39:33). Kalender F4/F5
  TIDAK aktif (kalender-web VPS 8 pin 173adec); konsumen Hermes
  TIDAK aktif (belum diregistrasi). Bukti perilaku 409 belum
  teramati (butuh 2 run bersamaan).
- Restart gabungan SELESAI + TERVERIFIKASI parent 08:50 (kedua node;
  VPS 8 live 1743566 restart 08:43:30, prep 08:44:16): 7/7 STREAMING
  + 7 RTMP, GDRIVE_URL aktif (MONA bisa Start #1795570268004),
  publish 26/0 (5254 slot 472 sempat FAILED 08:36 'token tidak
  ditemukan', pulih PUBLISHED 08:45 via retry operator), Stage 2
  DONE 1395. Caveat: load VPS 8 10,46/7,62/7,34 (load1 di atas
  8 core) pasca-restart.
- **FASE 3 LENGKAP (kode+tes): 3.1–3.7 + athrow + checkpoint semua
  DITERIMA.** Total tes baru Fase 3: 20 (3.3) + 13 (3.2) + 52 (3.4) +
  3 (3.5) + 5 (athrow) = 93, semua hijau di verifikasi independen
  (+ 77/49/294 regresi tetangga).
- Keranjang persetujuan Fase 3 (gabung keranjang berjalan; belum
  dieksekusi): (a) cabut PAT lama + rotasi 5 PIN operator (USER-only);
  (b) denyCurrentBranch updateInstead→refuse (L3); (c) penetapan vault
  460 ch + pipeline vault baru + target mingguan + cron monitoring
  pairing; (d) 6 keputusan Fase C + bangun applyer/snapshot/rollback
  (L3, 1–2 hari); (e) keputusan F4 + channel event + stream key/sesi;
  (f) aktivasi konsumen Hermes + restart web + uji live cabut-key
  (key uji + jendela 30 mnt) + verdict pilot; (g) keputusan deploy key
  SSH; (h) 3 cron alert + drill bulanan + akun UptimeRobot;
  (i) bersihkan 29 baris uji sentinel_kill_history + 312 orphan FK;
  (j) trigger manual audit 19:00 ATAU geser checkpoint ke 01:00
  26 Nov; (k) seed Kalender jangan reset state event (perbaikan kecil);
  (l) bersihkan 87 baris CTR fabrikasi video_audit. Tetap parkir:
  item 7–9 (1080p, akar readonly, URL relay 034), RETIRE vs REVIVE
  stage2-driver (unit systemd sudah ada di repo, commit 08:21:06),
  D1–D6 (belum ditulis, butuh restart aktivasi tersendiri).

## 1. Status awal (terverifikasi 04:35–05:02 WIB)

- Skor solid: **66/100** (gate readiness ±65%, NOT READY).
- Komponen: publish 88, live 65, LoopGen/Stage 2 45, ingestion 60, audit 45,
  Agent AI 75, Kalender 70, Hermes 70, infra/keamanan 55.
- Lantai baseline: TERPENUHI — 24 Nov 210 PUBLISHED, 0 FAILED (23 Nov 207).
  25 Nov 05:02: 2 PUBLISHED, 208 QUEUED, 30 SKIPPED_AUDIO_MISSING,
  1 AWAITING_MONETIZATION, 0 FAILED, ghost-kill 0.
- HEAD: `87ab7cea` (lini rebase 16 commit Hermes, isi identik `ecde0ef`,
  sudah di GitHub; ahead 0). PAT baru terpasang di kedua server;
  cabut PAT lama masih menunggu Bapak.
- Tiga merah terbesar: Stage 2 macet (DONE 1.386 mentok, QUEUED 7,
  PARKED 278, antrean tertua >36 jam), CTR audit fabrikasi (2 nilai
  distinct), B10 kambuh 2x semalam.
- UPDATE 05:30–05:32: migrasi live ke VPS 8 SELESAI (6/6 di live-01);
  klaster kematian ~05:32 (4 RECONNECTING). Lihat §0.

## 2. Prinsip pengaman (mengikat — aturan SOLID 22 Nov)

1. Lantai baseline (206 PUBLISHED/hari, 0 FAILED/BLOCKED) adalah FLOOR —
   tiap perubahan harus membuktikan tidak ada regresi.
2. Tiap perubahan: rencana verifikasi pre/post + rollback + jendela
   observasi. Tanpa itu, tidak diimplementasi.
3. Jangan "perbaiki" mekanisme yang belum terbukti dengan tebakan —
   diagnosis dulu (kasus B2 kuota: 6 teori dieliminasi, penyebab tetap
   unknown; langkah diagnosis saja sampai bukti ada).
4. Restart: maks 1x web per 15 menit; restart live/worker hanya dengan
   persetujuan eksplisit + alasan. Live disentuh terakhir.
5. L1/L2 otomatis; L3 (skema/migrasi/auth/infra/refaktor besar) butuh
   persetujuan; L4 dilarang.
6. Disiplin shared-tree git: commit + push dalam eksekusi yang sama;
   dilarang rebase branch bersama tanpa pemberitahuan (pelajaran insiden
   04:57 — push rebase tanpa atribusi).
7. Verifikasi independen: klaim lane lain dicek sendiri sebelum dilaporkan
   selesai (bukti: angka DB / log / endpoint, bukan laporan).

## 3. Ringkasan fase & target skor (proyeksi, bukan janji)

| Fase | Isi | Estimasi | Target skor sesudahnya |
|------|-----|----------|------------------------|
| 0 Quick wins | 4 item kecil | ±1–2 jam | 66 → 68 |
| 1 P0 | Stage 2, CTR + governor audit, B10 | 3–7 hari | → ~74 |
| 2 P1 | Sentinel, healer, audio, systemd, DB-lock | 1–2 minggu | → ~80 |
| 3 P2 | Pairing, Agent AI Fase C, Kalender F4/F5, Hermes, keamanan, monitoring | 2–4 minggu | → ~85+ |

Skor diukur ulang setelah tiap fase dari data live, dengan rubrik yang sama
(9 komponen tertimbang).

## 4. Fase 0 — Quick wins (hari ini) — 3/4 SELESAI (0.2, 0.3, 0.4 terverifikasi 05:25; 0.1 menunggu user)

### 0.1 Cabut PAT lama (Fase 4 runbook) — USER, 5 menit
- Bukti: token baru terverifikasi jalan di kedua server + repo skill
  (ls-remote + fetch hijau 04:58; 16 commit aman di GitHub).
- Langkah: GitHub → Settings → Developer settings → Personal access
  tokens → Delete token lama (BUKAN `masjavas-servers-2026-11`;
  cek tab Fine-grained DAN Classic).
- Verifikasi: sesudah cabut, Muse cek ls-remote + fetch kedua server
  tetap hijau (pakai token baru).
- Risiko: bila ada automasi lain memakai token lama → gagal 401;
  laporkan ke Muse. Rollback: tidak ada (pencabutan final) — mitigasi:
  verifikasi token baru dulu (sudah).

### 0.2 Systemd-kan kalender-web — Muse, ±30 menit
- Bukti: proses hidup PID 771112 sejak 24 Nov 14:40 tapi unit systemd
  inactive; bila proses mati, tidak ada yang me-restart (R52).
- Target: unit aktif + enabled; `Restart=always`; restart manual terbukti
  hidup lagi <60 detik.
- Langkah: tulis unit (ExecStart = perintah uvicorn yang sama:
  `/opt/masjavas-os/venv/bin/python -m uvicorn kalender.web:app --host
  100.108.161.98 --port 8081`), daemon-reload, stop proses detached →
  start via unit → health 200.
- Verifikasi: `systemctl status` active; health endpoint 200; /kalender
  via proxy Master 200; event count tetap 22.
- Risiko: salah ExecStart → kalender down; rollback: stop unit, jalankan
  manual seperti sebelumnya (perintah tercatat). Observasi 24 jam.

### 0.3 Hapus draf REJECTED di inbox Hermes — USER (1 klik) atau Muse, 5 menit
- Bukti: 1 draf uji `a7b6f4a33e82` masih di Inbox.
- Verifikasi: inbox count 1 → 0.

### 0.4 Alert sederhana stall Stage 2 — Muse, ±1 jam
- Bukti: stall ketahuan dari monitor per jam; antrean tertua tembus
  36 jam tanpa alert otomatis.
- Target: cek per 30 menit (cron/hook): bila Stage 2 0 DONE 60 menit +
  QUEUED > 0 → alert ke user (sekali per episode, anti-spam 4 jam).
- Verifikasi: simulasi kondisi stall (saat ini sedang stall, jadi alert
  pertama harus terkirim saat aktivasi) + uji anti-spam.
- Risiko: noise bila ambang terlalu sensitif; mitigasi: ambang 60 menit +
  anti-spam.

## 5. Fase 1 — P0 (3–7 hari) — DITUNDA sampai livestreaming beres (perintah 05:30)

### 1.1 Stage 2 macet (R32) — Muse + lane LoopGen, 1–3 hari
- Bukti (05:02): `loopgen_stage2_runs` DONE 1.386 mentok sejak ~22:47
  24 Nov; QUEUED 7; PARKED 278 (+2 sejak 03:46); FAILED 21; antrean
  tertua >36 jam; driver diam bukan kelaparan input (SUBMITTED 102
  menunggu).
- Target: DONE bergerak lagi (≥10 DONE/24 jam pertama pasca-perbaikan);
  antrean tertua <12 jam dalam 48 jam; 0 stall >2 jam selama 7 hari
  observasi.
- Langkah: (a) diagnosis: cek driver Stage 2 (proses hidup? load gate?
  starvation breaker? log error? slot ffmpeg?); bandingkan dengan fix
  starvation-breaker 20 Nov (apakah kambuh dalam bentuk baru);
  (b) perbaikan sesuai akar yang terbukti (bukan tebakan); (c) tes temp-DB
  + pilot 3 item; (d) deploy + observasi.
- Verifikasi: hitungan DONE/QUEUED/PARKED per jam 24 jam; antrean tertua;
  log driver tanpa error berulang.
- Risiko: perbaikan driver bisa mengganggu ingestion (rebutan slot ffmpeg);
  mitigasi: deploy di luar jam prime + observasi load. Rollback: git
  revert + restart worker prep (dengan persetujuan).
- Keputusan user: restart worker prep bila dibutuhkan (L3, ditanyakan
  saat dibutuhkan).

### 1.2 CTR audit fabrikasi (P0-1) — Lane Antigravity, 2–4 hari
- Bukti: 3.139 baris `video_audit`, ctr hanya 2 nilai distinct
  (0 / 0,0455), kategori A/B = 0; pilot LangGraph membuktikan Analytics
  tidak pernah request impressions.
- Target: run audit berikutnya menghasilkan ctr bervariasi (≥50 nilai
  distinct per 1.000 baris) + impressions > 0 untuk video yang punya
  tayangan; kategori A/B terisi; triase hidup.
- Langkah: (a) audit kode pengambilan metrik (field impressions di request
  Analytics? mapping ctr?); (b) perbaiki + tes dengan 5 channel
  (bandingkan ctr hasil vs YouTube API langsung); (c) run audit terbatas
  (10 channel) → verifikasi distribusi ctr; (d) baru run penuh.
- Verifikasi: distribusi ctr + impressions dari DB; cross-check 5 video
  vs API YouTube langsung; kategori A/B > 0.
- Risiko: perubahan query Analytics bisa menambah konsumsi kuota;
  mitigasi: run terbatas dulu + governor (1.3). Rollback: revert kode
  audit; data lama tidak dihapus (audit append-only).

### 1.3 Governor kuota audit (P2-2) — Lane Antigravity, 1–2 hari (paralel 1.2)
- Bukti: run 01:00 kena quotaExceeded di banyak vault; ledger kuota audit
  gagal tulis ("attempt to write a readonly database") — konsumsi audit
  tidak tercatat.
- Target: audit tidak pernah memakai >10% kuota harian per vault per run;
  ledger audit tercatat (0 write gagal); 0 quotaExceeded akibat audit pada
  run berikutnya.
- Langkah: (a) perbaiki path tulis ledger (izin file/DB — readonly);
  (b) cap + cek kuota sebelum tiap chunk; (c) jadwal audit di luar jam
  publish prime bila memungkinkan; (d) tes run 10 channel.
- Verifikasi: ledger audit terisi; hitungan quotaExceeded saat run audit
  = 0; kuota publish tidak terganggu (PUBLISHED hari itu ≥206).

### 1.4 Instrumentasi B10 (kematian encoder) — Lane Antigravity (live), 2–3 hari
- Bukti: klaster kematian 00:47 + 02:06 (exit 187/152, 3 encoder relay
  mati serentak); auto-reconnect pulih di percobaan 1; penyebab unknown.
- Target: pada kematian berikutnya, tersedia rekaman detik-detik sebelum
  mati (CPU/mem encoder, log relay fetch, watchdog) yang cukup untuk
  diagnosis; ATAU penyebab teridentifikasi.
- Langkah: (a) tambah logging pre-mortem (snapshot resource + 30 detik log
  terakhir per encoder saat exit); (b) korelasikan dengan badai relay
  (R44: 246× manifest error); (c) bila pola ketemu → perbaikan akar;
  bila tidak → tetap jalankan dengan pemulihan yang terbukti.
- Verifikasi: 1 insiden terekam lengkap ATAU 7 hari tanpa klaster (dengan
  instrumentasi aktif).
- Risiko: logging tambahan membebani CPU; mitigasi: ring buffer kecil,
  tulis hanya saat exit.

## 6. Fase 2 — P1 (1–2 minggu) — DITUNDA sampai livestreaming beres (perintah 05:30)

### 2.1 Sentinel probe via API — Lane Antigravity, 1–2 hari
- Bukti: Sentinel salah baca YOUTUBE_OFFLINE (probe /live dialihkan ke
  beranda); breaker menahan kill; 3 ghost-kill WULAN 24 Nov siang berasal
  dari salah baca ini.
- Target: 0 salah baca selama 7 hari (probe vs status API YouTube cocok
  100% pada sampel); breaker tidak pernah terpicu karena salah baca.
- Langkah: ganti probe halaman → `liveBroadcasts.list` (status lifeCycle
  + actualStartTime); simpan hasil probe + keputusan kill ke log
  terstruktur; uji negatif (stream yang memang offline tetap terdeteksi).
- Verifikasi: perbandingan probe vs API 3x/hari selama 7 hari; hitungan
  kill yang benar vs salah.

### 2.2 Healer ingestion retry otomatis — Lane Antigravity / Muse, 1–2 hari
- Bukti: #409 FAILED tanpa retry ~12 jam+; #542 diam ~42 jam
  (eskalasi ke-30).
- Target: job FAILED di-retry otomatis ≤1 jam (maks 3x, backoff); job diam
  >6 jam dieskalasi otomatis; #409 dan #542 beres atau berstatus jelas.
- Langkah: (a) audit logika healer (kenapa #409 tak di-retry); (b) retry +
  backoff + eskalasi; (c) selesaikan #409/#542 manual bila retry tidak
  berlaku (keputusan per kasus).
- Verifikasi: status akhir #409/#542; simulasi 1 job FAILED → retry
  tercatat ≤1 jam.

### 2.3 Tutup celah audio 30/hari — Ingestion, 3–7 hari (paralel)
- Bukti: SKIPPED_AUDIO_MISSING 30/hari (24 + 25 Nov); gap audio 10.066
  item (sensus 20 Nov).
- Target: SKIPPED_AUDIO_MISSING <15/hari dalam 1 minggu; <5/hari dalam
  2 minggu.
- Langkah: prioritas channel dengan gap audio terkecil (smallest-gap-first,
  pola yang sama dengan bahan); diagnosis bottleneck ingestion audio
  (yt-dlp bot-check? slot ffmpeg?) sebelum perbaikan.
- Verifikasi: hitungan SKIPPED harian; PUBLISHED naik seiring audio terisi.

### 2.4 Systemd semua proses detached — Muse, ±1 hari
- Bukti: R52 — web/worker/hermes-bridge Master + kalender-web VPS 8
  detached; hitungan error web/worker tidak terukur.
- Target: semua service produksi di bawah systemd (active + enabled +
  `Restart=always`); metrik error web/worker terukur lagi.
- Langkah: audit proses detached → unit per service → migrasi satu per
  satu (mulai yang paling tidak berisiko) → verifikasi health tiap migrasi.
- Risiko: migrasi service live/worker mengganggu stream/publish; mitigasi:
  satu per satu di luar jam prime + persetujuan per service. Live terakhir.

### 2.5 Tuntaskan kelas cacat DB-lock — Muse + Antigravity, 2–3 hari
- Bukti: presence diperbaiki (`d8af0ba`); `save_bg_job` cacat yang sama;
  ledger audit gagal tulis readonly; batch 100 mati total karena
  `record_usage` di luar try/except (sudah diperbaiki di B2).
  BUKTI BARU 25 Nov 07:06: task 5142 NO 267 FAILED "database is locked"
  (transisi DOWNLOADING_ASSETS→RENDERING); 27 error lock hari itu
  (web 22, worker 3, live 2); lock dipegang 10–30+ dtk. Titik awal
  konkret: tambah "database is locked" ke daftar transient _is_infra
  (worker/autonomous_daemon.py:1201-1205) agar auto-retry 1x berlaku;
  verifikasi update_task_status tahan lock (retry internal) agar
  re-queue tidak gagal diam-diam (except: pass).
- Target: 0 "database is locked" di log web/worker/live selama 7 hari
  (kecuali transient startup); semua tulis DB dari event loop dipindah
  off-loop atau ber-retry.
- Langkah: audit semua pemanggilan DB sinkron di jalur async
  (grep `get_sync_db` + `save_*`); perbaiki per pola presence; tes regresi
  (tulis <2 detik saat DB terkunci).
- Verifikasi: grep log 7 hari; tes regresi hijau.

## 7. Fase 3 — P2 (2–4 minggu) — SELESAI kode+tes 25 Nov ~08:50 (verifikasi §0.8); aktivasi menunggu keputusan user

### 3.1 Pairing 466 channel UNPAIRED — Operator + Muse, berkelanjutan
- Bukti: CONNECTED 241/707 (34%).
- Target: +20 channel paired/minggu (interim: 300 CONNECTED dalam 3 minggu).
- Langkah: daftar prioritas per operator (widget dashboard sudah ada);
  operator pairing via dashboard; Muse pantau hitungan mingguan.
- Keputusan user: target mingguan (+20/minggu atau lain) + dorongan ke operator.

### 3.2 Agent AI: judul + QA + keputusan Fase C — Muse, 2–3 hari + keputusan user
- Bukti: agent judul + QA 0 run; 102 DRAFT belum di-apply (Fase C ditahan).
- Target: pilot judul 5 post + QA 20 DRAFT; keputusan Fase C
  (apply per batch dengan persetujuan).
- Langkah: pilot judul/QA → laporan kualitas → Bapak putuskan Fase C
  (ya/tidak + aturan: per batch, maks N, rollback via `old_*` +
  backup thumbnail).
- Keputusan user: Fase C ya/tidak.

### 3.3 Kalender F4/F5 — Lane Kalender, 2–4 hari + keputusan user
- Bukti: 21 event SCHEDULED, 63 media_tasks pending, renders 0;
  4 event mulai malam ini (PGL Major, Davis Cup, Syed Modi, UCL).
- Target: prelive + HITL-2 jalan untuk event minggu ini; minimal 1 siaran
  Kalender mengudara via F5.
- Keputusan user: lanjut F4 sekarang? (sudah ditanyakan di chat Kalender —
  belum dijawab).

### 3.4 Hermes: verdict + konsumen inbox + uji cabut key — User + Lane Antigravity, 1–2 hari + keputusan
- Bukti: pilot Fase 1 5/5 LOLOS; verdict ≥2/3 'layak pakai' menunggu;
  konsumen inbox belum dibangun; cabut key/reset belum diuji live.
- Target: verdict diputuskan; konsumen inbox dibangun (atau diputuskan
  tidak); uji cabut key + reset di staging (bukan key produksi).

### 3.5 Keamanan — Muse + User, 1–2 hari
- (a) Cabut PAT lama (Fase 0.1). (b) Sanitasi PAT lama yang tertanam di
  `DEPLOYMENT.md:30` + 5 PIN operator di `.env.example` (temuan audit repo
  24 Nov 22:0x — repo tetap privat sampai 3 bloker beres). (c) Pertimbangkan
  deploy key SSH jangka panjang (mengganti PAT sepenuhnya).
- Target: 0 rahasia aktif di repo (scan ulang bersih); sheet EXPERT:
  grant eksplisit SA + operator, cabut anyone-link (bila diputuskan).
- Keputusan user: remediasi publikasi repo — jalan atau tetap ditunda
  (ditunda 22:18)?

### 3.6 Monitoring + restore drill — Muse, 2–3 hari
- Bukti: backup jalan (04:28 OK) tapi restore belum pernah dibuktikan;
  alerting minim (Stage 2 ketahuan dari monitor per jam); monitor buta
  sesekali (SSH proxy flaky).
- Target: 1 restore drill sukses ke DB temp (integrity ok + hitungan
  cocok); alerting untuk: Stage 2 stall (0.4), disk <20 GB,
  quotaExceeded >10/jam, ghost-kill >0.
- Langkah: drill restore (DB temp, bukan live); pasang alert (cron/hook +
  anti-spam); heartbeat eksternal (UptimeRobot free — boleh komersial
  per ToS).

### 3.7 Atribusi push rebase 04:57 — Muse, investigasi ringan
- Bukti: rebase + push 16 commit 04:57:01 tanpa atribusi; isi identik,
  tidak ada yang hilang.
- Target: pelaku teridentifikasi ATAU aturan push diperketat (commit+push
  satu eksekusi, larang rebase branch bersama tanpa pemberitahuan di chat
  koordinasi).
- Langkah: tanya lane Antigravity + cek cron Cycle 22 (04:50) apakah
  menjalankan git push; catat hasilnya; bila tak ketemu, tetapkan aturan +
  umumkan ke semua lane.

## 8. Dependensi & urutan

- 0.1 (cabut PAT) independen — bisa sekarang. 0.4 independen.
- 1.2 (CTR) ↔ 1.3 (governor): governor dulu ATAU bersamaan — run audit
  berikutnya wajib pakai governor.
- 1.1 (Stage 2) independen dari 1.2/1.3 — bisa paralel (lane berbeda).
- 2.1 (Sentinel) sebelum mengendurkan breaker; breaker tetap aktif selama
  2.1 berjalan.
- 2.5 (DB-lock) dikoordinasikan dengan 1.3 — ledger audit butuh tulis DB
  yang benar.
- 3.2 Fase C butuh 1.2 (triase CTR) untuk prioritas — atau jalan tanpa
  prioritas (urutan manual).
- 3.3 F4/F5 independen.
- 2.4 (systemd) satu per satu, live terakhir.

## 9. Keputusan yang menunggu Bapak

1. Setujui rencana ini per fase (atau per item)? — eksekusi dimulai setelah
   persetujuan.
2. Fase C Agent AI: ya/tidak (+ aturan batch)?
3. Kalender F4: lanjut sekarang?
4. Verdict pilot Hermes (≥2/3 'layak pakai')?
5. Remediasi publikasi repo: jalan atau tetap ditunda?
6. Target pairing mingguan (+20/minggu atau lain)?
7. Cabut PAT lama: Bapak eksekusi (5 menit, panduan sudah ada)?
8. Restart worker prep / live bila dibutuhkan Fase 1–2: setuju per kasus
   (akan ditanyakan saat dibutuhkan)?

## 10. Cara kerja eksekusi

- Eksekusi per fase, laporan per fase (KPI: Progress, Hasil, angka terukur).
- Verifikasi independen oleh Muse sebelum item dinyatakan selesai.
- Skor diukur ulang setelah tiap fase (rubrik yang sama: 9 komponen
  tertimbang).
- Satu kartu persetujuan per pekerjaan besar; batch paralel agar 1 kartu
  cukup.
- Bila satu item macet >24 jam: eskalasi ke Bapak dengan pilihan
  (lanjut / tunda / ganti pendekatan).

## Lampiran — data dasar (04:35–05:02 WIB 25 Nov 2026)

- Publish 24 Nov: 210 PUBLISHED / 0 FAILED / 30 SKIPPED_AUDIO_MISSING /
  1 QUEUED. 25 Nov 05:02: 2 PUBLISHED / 208 QUEUED / 30 skipped /
  1 AWAITING_MONETIZATION / 0 FAILED.
- Live (05:02): 6 STREAMING (3 VPS 8 + 3 Master); ghost-kill 0 hari ini,
  total 785. UPDATE 05:30: 6/6 di VPS 8 (live-01), Master 0.
  UPDATE 05:32: klaster kematian — 4 RECONNECTING (707/114/088/WULAN),
  2 STREAMING (706/034).
- Stage 2: DONE 1.386, QUEUED 7, PARKED 278, FAILED 21. Agent tasks:
  IMAGE DONE 5.812, VIDEO_AI DONE 1.197, SUBMITTED 102. Klaim EXPIRED 316 /
  COMPLETED 295 / ACTIVE 6.
- Audit: 3.139 baris, ctr 2 nilai distinct (max 0,0455), kategori A/B 0.
- Agents: 102 DRAFT, Rp 358,52, 8 run, 0 applied.
- Kalender: 22 event (21 SCHEDULED + 1 REJECTED), 63 media_tasks pending,
  renders 0.
- Hermes: P1–P3 live; inbox 1 draf REJECTED; biaya ESTIMASI.
- Infra: HEAD `87ab7cea`, ahead 0; PAT baru aktif; load Master ~3,3;
  disk 66 GB bebas (55%); channels CONNECTED 241 / UNPAIRED 466.
