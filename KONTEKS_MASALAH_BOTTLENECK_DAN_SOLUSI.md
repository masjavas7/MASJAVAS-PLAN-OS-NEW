# KONTEKS MASALAH, BOTTLENECK, PROBLEM & SOLUSI YANG DIHARAPKAN
## MASJAVAS TUBE OS — 22 Nov 2026

Dokumen kerja program readiness (turunan). Bukan dokumen beku dan bukan
instruksi deployment. Setiap angka di bawah ini terverifikasi dari audit
22 Nov 2026 (V2-001/V2-002/V2-003, plan VPS 8, dashboard) atau dilabeli
sebagai asumsi/estimasi bila belum terverifikasi.

---

## 1. Konteks masalah

**MASJAVAS TUBE OS** adalah autopilot YouTube faceless: **707 channel**
(slot NO 1–707; **237 CONNECTED**), dijalankan operator (5 rotasi + Arief
143 + Agus 100 + AZIZ 251 + MONA 13) di atas **satu box Master**
(Contabo vmi3762689: 4 vCPU, 8 GB RAM, 145 GB disk, port 500 Mbit/s).

Empat beban kerja berbagi satu mesin, satu database SQLite, dan satu port
jaringan:

| Beban | Isi | Sifat |
|---|---|---|
| PUBLISH | 1 video/channel/hari ke YouTube (206 PUBLISHED/hari) | Real-time, terikat kuota API |
| PREPARATION | Ingestion (audio/video/thumbnail) + kurasi audio (reassemble) + LoopGen Stage 2 | Batch, CPU + I/O berat |
| LIVE | Streaming 24/7 (limit 16 channel; 2–3 jalan) | Real-time, butuh egress stabil |
| CONTROL | Web dashboard/API + worker daemon + governor | Harus selalu responsif |

**Baseline "solid" yang harus dilindungi** (terverifikasi 21–22 Nov):
206 PUBLISHED/hari, 0 FAILED/BLOCKED, median delay publish 11,6 mnt
(p90 30,2 mnt), healthz ~10 ms, lock SQLite 31×/24 jam, load Master
3,5–4,2. Aturan perubahan yang mengikat (user, 22 Nov 01:48): aplikasi
sudah solid — setiap perubahan wajib terverifikasi agar tidak merusaknya.

**Pertanyaan pemandu:** bagaimana menambah kapasitas (VPS 8) dan
memperbaiki bottleneck tanpa merusak baseline ini, dan tanpa mengulang
riset arsitektur saat pembelian dilakukan? Jawabannya adalah program
readiness ini: audit dulu (Batch 1), siapkan semua artefak deployment
(runbook, register, acceptance test, rollback), baru eksekusi setelah
gerbang lolos + persetujuan owner.

---

## 2. Bottleneck saat ini (terverifikasi)

### B1. Live terjepit JARINGAN, bukan CPU

- **Bukti:** port Master 500 Mbit/s terukur 91–93%; 50 channel @1080p
  butuh ±310 Mbps egress. Live hanya ±0,07 core untuk 2 stream —
  CPU bukan masalah.
- **Dampak:** tidak ada ruang tumbuh live di Master; setiap stream baru
  berebut port dengan upload prep + sinkronisasi Drive.
- **Status:** dipahami; solusinya VPS 8 (port 600 Mbit/s) — Fase 1.

### B2. Ingestion macet di GATE + bot-check, bukan CPU

- **Bukti:** 123 COMPLETED/7 hari vs 333 SUPERSEDED (**churn 66%**);
  5 cap bertumpuk (shed flag, operational aktif, live aktif, gate RAM,
  gate klaim disk/load/JIT/broadcast window) mengunci konkurensi di **1
  permanen**; **0 stage berat selesai/24 jam**; 29 COMPLETED/24 jam
  semuanya DRIVE_SYNC. Bot-check yt-dlp: hanya WARP (IP datacenter)
  yang hidup; residential/bulk kosong; cookie anonim saja.
  SKIPPED_AUDIO_MISSING melonjak 3 → 31/hari (20–21 Nov).
- **Dampak:** backlog bahan **412 channel / ±28.461 item** (video 9.698,
  audio 10.066, thumbnail 8.697); **195 channel kosong total**; hanya
  284 channel lengkap 30/30/30.
- **Status:** akar di gate (bukan CPU) terbukti; D1 (proxy residential)
  DITUNDA user 20 Nov — jangan tanyakan ulang.

### B3. Governor TIDAK BERTINDAK — atribusi CPU buta

- **Bukti (V2-003):** 1.918 tick V2 dalam 17 jam → **0 aksi efektif**
  (1.886 none, 29 shed_render yang log-only TANPA eksekutor, 0
  shed_ingestion/pause/resume). `classify_lane` hanya mengenali cmdline
  berisi "ingestion": daemon otonom (±18% CPU) tercatat lane `other` →
  `ingestion_cpu_pct` selalu 0,0% → tangga selalu memilih shed_render
  (buntu). Konfigurasi split: guard loop = V2, admission = legacy
  (`LIVE_GUARD_V2_ENABLED=false`, hanya dibaca di `can_start`).
  Knob bermasalah: TX 115 (stale, dari box lama), CPU 90 (inert —
  legacy pakai hard-code 95), 720p (stale vs keputusan 1080p).
- **Dampak:** satu-satunya pengaman beban tidak pernah bekerja; semua
  sinyal shed-load (91×) berasal dari era legacy.
- **Status:** audit selesai (READY_FOR_REVIEW); perbaikan menunggu D9/D10.

### B4. Modal CPU ingestion SALAH HITUNG — reassemble tersembunyi

- **Bukti (koreksi Bapak, 22 Nov 12:10):** perakitan audio (reassemble,
  `core/curation/audio_reassembler.py` — crossfade 6 lagu + remaster
  EBU R128 −14 LUFS) jalan **64×/24 jam, ±0,9 core/job, ±6 mnt/job**
  (±1 core puncak, ±70 MB/job) sebagai anak proses web/daemon, dan
  governor mencatatnya di bucket **render/web** — bukan ingestion.
  Plan VPS 8 sempat menganggarkan ingestion ±0 CPU atas dasar angka
  "0,0% di 1.510 tick" yang ternyata artefak atribusi.
- **Dampak:** total VPS 8 terkoreksi ≈6 → **≈7 core puncak, headroom
  hanya ±1 core**. Anggaran 1 core ingestion di V2.0 ternyata sudah
  benar — reassemble-lah 1 core itu.
- **Status:** plan diperbaiki (3 edit, terverifikasi) + addendum;
  V2.0 tidak perlu diubah angkanya.

### B5. LoopGen Stage 2: I/O dominan + antrean

- **Bukti:** 12,8 dtk CPU vs **100,1 dtk wall/klip** (I/O dominan —
  unduh + unggah + QA); 1.011 DONE/7 hari; antrean 50 QUEUED.
  Driver sempat starvation (2 load gate tanpa breaker, 0 DONE ±4,5 jam
  20 Nov) — diperbaiki, DONE naik 494 → 531; park rate turun 29% → 14%.
- **Dampak:** Stage 2 adalah gerbang throughput, bukan generasi
  (19 Nov: worker menghasilkan 211 video, Stage 2 hanya 20 DONE).
- **Status:** driver diperbaiki; canary 2 slot di VPS 8 direncanakan.

### B6. Live TIDAK STABIL — encoder mati berulang

- **Bukti (V2-002):** encoder slot 114 mati **14×** (12× pada
  21 Nov 14:00–22 Nov 07:00, interval menyusut ke 9–13 mnt; #13 07:17;
  #14 08:09:50). Tanda tangan: `Error writing trailer: Connection
  timed out` (RTMP timeout, bukan crash/OOM). Rantai kematian #14
  terekam lengkap: feeder stall 30 dtk → watchdog fired → fallback
  menyala → encoder mati 6 dtk kemudian (audio submit timeout).
  Orphan slot 706 mengudara **TANPA AUDIO ±17 jam** tanpa terdeteksi
  (bitrate video tetap HEALTHY) — pulih 08:10:28.
- **Dampak:** downtime live berulang; reconnect cap tidak pernah
  eskalasi (`reconnect_count` di-reset tiap sukses) sehingga tidak
  pernah alert.
- **Status:** perbaikan R3–R8 + iterasi 2 deployed (commit `56f7a6d` +
  `021a9f0`, tes 20/20) — **observasi 24 jam GAGAL pada ±6,5 jam:**
  22 Nov 14:47:15–14:47:36 keempat stream mati serentak (lihat B10).
  Watchdog iterasi 2 fired 14:47:11 (feeder stall) tapi encoder 706
  tetap mati 14:47:15 — watchdog mendeteksi, TIDAK menyelamatkan.

### B7. Web adalah compute tersembunyi

- **Bukti:** unit web rata-rata **1,02 core** — bukan HTTP, melainkan
  job ffmpeg berat in-process (reassemble 64×/24 jam, thumb regen,
  fleet run, AI video pipeline); 2 anak ffmpeg @ ±98% teramati 2×.
- **Dampak:** dashboard/API berbagi CPU dengan job batch berat;
  satu job audio menaikkan load server ke 4,63 (11:33).
- **Status:** Fase 0 memindahkan reassemble keluar dari web ke worker.

### B8. SQLite satu penulis + lock

- **Bukti:** "database is locked" **31×/24 jam**; satu event loop
  (G12); G10 belum ditangani. Arsitektur melarang akses SQLite lewat
  jaringan — worker remote wajib REST API.
- **Dampak:** claim job + tulis konkuren sesekali gagal; risiko
  membesar saat beban naik.
- **Status:** audit schema (V2-007) adalah gerbang teknis sebelum
  migrasi apa pun; trigger evaluasi PostgreSQL sudah didefinisikan
  (lock >10/hari pasca-optimasi, p95 tulis >100 ms konsisten).

### B9. Kuota + dependensi eksternal rapuh

- **Bukti:** 1 video = 1.650 unit; doktrin maks 5 channel/vault
  (8.250 < guard 8.400). Insiden 18 Nov: 217 post ekstra membakar
  358.050 dari 444.350 unit, 121 jadwal missed. Mekanisme kuota sync
  Drive (P2) **belum terpecahkan** — 6 teori diuji dan gugur 22 Nov.
  KEIN (OMNI 1.1) **kedaluwarsa 23 Nov 2026**; status live-nya UNKNOWN
  (diterima owner sebagai asumsi — "ASUMSIKAN KEIN AMAN").
  Disk 83/145 GB (58%); janitor mislabel 203 channel (R6).
- **Dampak:** satu kesalahan penjadwalan bisa menghabiskan kuota
  harian; satu dependensi mati bisa menghentikan suplai.
- **Status:** doktrin kuota mengikat; audit janitor (V2-004) tiketnya
  siap, eksekusi antre; fallback KEIN sudah dihargai (DeepInfra FLUX
  Rp 35/gambar, Muse Image Rp 180/gambar) tapi belum diputuskan.

### B10. Live MATI MASSAL + pemulihan TIDAK ANDAL (22 Nov 14:47)

- **Bukti (diagnosis read-only 22 Nov 15:09–15:15, host live):**
  keempat stream mati dalam 21 detik — 706 14:47:15 (exit 146,
  RTMP timeout), 707 14:47:21 (orphan, tanpa handle Popen), 698
  14:47:21 (orphan, tanpa handle Popen), 114 14:47:36 (exit 0,
  keluar bersih — penyebab tidak diketahui). Kematian serentak =
  penyebab bersama (kemungkinan peristiwa jaringan/egress —
  BELUM TERBUKTI, butuh korelasi TX/log jaringan). Supervisor
  menandai 706/114/698 STOPPED pada epoch yang sama (14:47:21).
- **Pemulihan gagal di 5 titik:** (1) reconnect 707 gagal
  "database is locked" 14:47:57 (B8 langsung mematahkan pemulihan);
  (2) 114 + 698 berhasil mengudara lagi (PID baru, terverifikasi
  jalan) tapi status DB tetap STOPPED — `started_at`/`pid`
  ter-update, `updated_at` beku di 14:47:21 (desync status/proses);
  (3) 706 TIDAK PERNAH dicoba reconnect — tanpa encoder sejak
  14:47:15; (4) 707 punya DUA encoder ke satu RTMP key (auto-healer
  web + reconnect supervisor berpacu tanpa koordinasi — dua jalur
  pemulihan); (5) relay 706 tetap hidup untuk task yang sudah mati
  (watchdog membunuh feeder tiap ±88 dtk; "sumber KEMBALI ONLINE"
  5× dalam 90 dtk untuk siaran yang tidak ada).
- **Temuan sampingan:** supervisor (PID 3951691, sejak 08:20)
  masih hidup tapi mencatat "Received > 3 system signals, hard
  exiting" berulang sejak 14:50 — penyebab pengiriman sinyal
  TIDAK DIKETAHUI. 3 zombie ffmpeg di bawah uvicorn (web tidak
  me-reap anaknya); 1 feeder orphan (PPID 1, 4 jam+); health check
  supervisor kena "database is locked" 14:50:25.
- **Dampak:** dari sudut user/dashboard: 706 mati total; 114/698
  tampil STOPPED (encoder zombie mendorong ke broadcast yang sudah
  berakhir); hanya 707 tampil STREAMING (dengan encoder ganda).
  Live 24/7 tidak dapat diandalkan sebagai produk — kematian
  massal + pemulihan setengah jalan lebih buruk dari kematian
  satu encoder.
- **Status:** DICATAT 22 Nov 15:15 atas instruksi owner.
  **Investigasi SELESAI 15:30 (mekanisme TERBUKTI dari kode +
  log + DB):** (1) health check supervisor hanya membaca task
  STREAMING (`live_supervisor.py:1010`) — task STOPPED tidak
  terlihat selamanya; (2) watchdog step saat itu HANYA jalan di
  daemon (supervisor lama 021a9f0 tidak memanggilnya — dibuktikan
  via git) dan cabang terminal-nya menulis STOPPED + "Siaran
  telah berhenti." HANYA 4 kolom (tanpa pid) dengan `now` yang
  dihitung sekali di awal step — commit-nya yang tertunda
  kontensi DB-lock (terbukti 14:47:57) menimpa baris STREAMING
  segar dari start_stream: persis campuran kolom yang teramati
  (pid/started_at baru, status/stopped_at/updated_at 14:47:21);
  (3) guardian (resurrect STOPPED 24/7) saat itu hanya ada di
  daemon dan meninggalkan NOL baris log dalam 27 menit — tidak
  pernah mencoba membangkitkan 706 (perilaku daemon lama tidak
  dapat diverifikasi pasca-fakta); (4) "hard exiting" TIDAK
  DITEMUKAN di kode aplikasi/venv — sumber tidak teridentifikasi,
  mati bersama supervisor lama (0 kemunculan di supervisor baru).
- **Pemulihan (BUKAN oleh agen ini):** lane lain men-deploy
  guardian (commit `0de723a` 15:12, `7ab9409` 15:18,
  `621b24c` 15:23 — perbaikan bug sentinel Row→dict) +
  me-restart layanan web+live 15:14:08 dan live lagi 15:19:18;
  daemon baru (15:15:56) membangkitkan keempat stream 15:16:21–27
  via guardian. **Terverifikasi 15:29 (13 menit stabil): 4/4
  STREAMING, heartbeat segar, TX ±29 Mbps, audio MENGALIR di
  semua task** (feeder wchar +2,6–3,5 MB/19 dtk ≈ 140–186 KB/s).
  Agen ini sengaja TIDAK membunuh proses / mengubah DB apa pun —
  state sudah sehat dan intervensi bedah akan ditimpa oleh loop
  pemulihan yang berpacu + deploy lane lain yang sedang jalan.
- **Ketidakstabilan yang TERSISA (temuan baru 15:30):** sentinel
  ghost-detection lane lain membunuh 114 2× (15:24:43 + 15:27:31,
  "OFFLINE di YouTube") dan restart ganda (supervisor manual +
  daemon) menciptakan encoder ganda 114 lagi; kini TIGA proses
  menjalankan loop pemulihan (supervisor systemd 638988 +
  supervisor manual 650871, test instance sejak 15:23:54 + daemon
  627930); feeder orphan purba masih hidup (114: 2 feeder, 4,5
  jam + 1 hari 5 jam; 698: 1 feeder, 40 menit); perbaikan sentinel
  (`621b24c`) sudah commit tapi BELUM ter-deploy di supervisor
  systemd (berjalan sejak 15:19:22, sebelum commit). Perbaikan
  struktural (SATU pemilik pemulihan) menunggu keputusan owner.
- **UPDATE 19:0x (lihat B11):** supervisor manual 650871 sudah
  tidak ada; sentinel fix ter-deploy via restart 18:47–18:48 —
  dan sentinel kini membunuh SEMUA stream tiap ±4 menit karena
  probe-nya dibutakan bot-check YouTube (B11).

### B11. Relink membeku + sentinel BUTA membunuh siaran sehat (22 Nov 18:58–19:05)

- **Bukti (diagnosis read-only 19:01–19:08):** (1) Endpoint
  relink-publik (`POST /api/live/metadata/{id}/relink-public-
  broadcast`) menjalankan 5–9 panggilan YouTube API SINKRON
  berurutan (`googleapiclient .execute()`, 30× di
  `live_broadcaster.py`, NOL `to_thread`) di dalam satu-satunya
  worker uvicorn → event loop beku 60–120 dtk+ (3 hapusan saja
  16 dtk) di tengah load 9,09. Klik user 18:58:2x BERHASIL di
  server (broadcast publik baru `JmMbPqD3Qlo` dibuat + terikat +
  DB ter-update dari `VYhwLLrMi8E`) tapi responsnya tidak pernah
  sampai ke browser ("Failed to fetch", tanpa baris akses POST
  di log uvicorn). Fitur publik + auto-start sendiri BEKERJA
  (broadcast tayang 19:00:32). (2) Probe sentinel
  (`live_health_sentinel.py`) men-scrape halaman watch publik
  dari IP datacenter VPS dan YouTube membalas
  `playabilityStatus: LOGIN_REQUIRED` ("Login untuk mengonfirmasi
  bahwa Anda bukan bot") — TIDAK ADA data live sama sekali.
  Probe tidak membedakan "tidak dapat memverifikasi" dari
  "offline" → verdict OFFLINE palsu → ghost-kill. Terbukti dari
  jaringan bersih: `JmMbPqD3Qlo` tayang 19:00:32–19:04:40 dan
  DIBUNUH sentinel 19:04:05 (saat sedang tayang sehat).
  (3) Lingkaran pembunuhan: sentinel membunuh tiap ±4 menit
  (90 dtk warmup + 2 probe gagal); broadcast yang diakhiri
  tidak bisa dilanjutkan; guardian me-restart encoder TANPA
  membuat broadcast baru (relink TIDAK PUNYA pemanggil otonom —
  hanya tombol manual yang responsnya hilang) → encoder
  mendorong ke kunci tanpa broadcast live → sentinel membunuh
  lagi. Stream key TERBUKTI cocok (encoder = task = channels).
- **Dampak:** channel tidak punya siaran yang dapat ditonton
  meski encoder jalan; user tidak bisa menerbitkan via UI
  (klik tampak gagal padahal server bekerja — klik ulang =
  broadcast duplikat); keempat stream berputar kill-restart
  tiap ±4 menit.
- **Status:** DIAGNOSIS SELESAI 19:08 atas permintaan owner.
  Perbaikan menunggu keputusan owner: (a) probe sentinel jangan
  membunuh atas hasil tak-kesimpulan — pakai YouTube Data API
  (`liveBroadcasts.list`, otoritatif) atau anggap UNVERIFIED;
  (b) panggilan YouTube ke `to_thread` / relink jadi job async;
  (c) relink otonom saat broadcast terbukti berakhir;
  (d) pertimbangkan matikan ghost-kill sementara sampai probe
  diperbaiki. Lihat PR-17 + R21.

### B12. Kuota YouTube API habis dibakar lingkaran kill-relink — keempat live GELAP di YouTube sejak ±21:30 (22 Nov)

- **Gejala (23 Nov 03:28):** klik relink RIFFORA gagal
  `quotaExceeded` di `liveStreams.list` (screenshot user).
- **Fakta tayang (jaringan bersih, halaman publik YouTube):**
  keempat broadcast berakhir/offline — 707 YFmbo1wSusM tayang
  21:28:11–21:37:11, 706 uPu8qlfw_x0 21:25:37–21:42:48, 114
  IAEHam3JAiM + 698 tKHCQQBwm7U `LIVE_STREAM_OFFLINE`.
  **Keempat channel GELAP sejak ±21:30 (6+ jam)** sementara
  dashboard menampilkan 4/4 STREAMING (hanya memeriksa PID
  encoder hidup — tidak ada verifikasi status YouTube).
- **Mekanisme bakar (terverifikasi):** tiap siklus kill →
  transition complete (50 unit) + unlist replay (50) + ensurer
  detect (3) + relink (±200–350: insert 50 + bind 50 + delete
  50×N + videos.update 50 + list) ≈ **300–450 unit/siklus**.
  618 kill sejak 19:08 menghabiskan 10.000 unit/hari ketiga vault
  live (±21:23–21:25: quotaExceeded pertama 21:23:37 metrics,
  21:25:05 live journal; 5.913 error quotaExceeded sejak 19:00 —
  web 1.897 + live 4.016).
- **Ledger kuota BUTA terhadap panggilan live:** vault 707
  tercatat `daily_quota_used` 28/10.000 — `vault_quota_ledger`
  hanya mencatat VIDEO_INSERT, VIDEO_SET_THUMBNAIL,
  VIDEO_LIST_CCV. Seluruh panggilan liveBroadcasts/liveStreams/
  transition/videos.update TIDAK dicatat dan TIDAK masuk guard
  kuota — guard tidak bisa memperingatkan atau menahan.
- **Reset kuota 15:00 WIB 23 Nov** (tengah malam PT). Tanpa
  menghentikan lingkaran kill, kuota segar habis lagi ±2 jam.
- **Temuan samping:** encoder ganda (707: ≥3–4 bersamaan; 114:
  ≥2 — orphan tak-terpanen tiap restart); ±9 feeder yatim
  (tertua 1,7 hari); zombie ffmpeg tak-terpanen; daemon kedua
  sempat muncul 03:33:32 lalu hilang ±03:37 (aktor tak dikenal).
- **Rem darurat DI-DEPLOY 03:53 (disetujui owner 03:51, commit
  `96e5ee4`):** patch tri-state `live_health_sentinel.py` —
  hasil probe tak-pasti menjadi UNVERIFIED, TIDAK membunuh;
  sinyal offline positif tetap membunuh pada kegagalan ke-2.
  Verifikasi 03:53–04:03: kill BUTA 0 (dari ±12/8 mnt); 114
  dibunuh 2× atas sinyal offline positif asli
  (LIVE_STREAM_OFFLINE) lalu pulih tiap kali; 706/707/698
  stabil (PID 7–12 mnt). Backup file asli di
  `data/backups/sentinel-brake-20261123-0353/`.
- **Temuan lanjutan (04:02):** ensurer HANYA berjalan saat
  stream start/reconnect (supervisor l.578/l.895) — tanpa
  pemicu berkala. 114 pulih sendiri pasca-reset lewat
  siklusnya; 706/707/698 butuh SATU restart encoder setelah
  15:00. Cron satu-kali `live-quota-reset-recovery-20261123`
  dijadwalkan 15:10 WIB untuk cek + restart yang masih gelap.
  Lihat PR-18 + R22.

---

## 3. Problem (daftar terstruktur)

| ID | Problem | Tingkat | Bukti / rujukan |
|---|---|---|---|
| PR-01 | Encoder live mati berulang (114: 14×) | P0 | V2-002; R2; iterasi 2 observasi |
| PR-02 | Governor tidak pernah bertindak efektif | P0 | V2-003; R16/R17/R18; D9/D10 |
| PR-03 | Ingestion terkunci konkurensi 1; backlog 28.461 item | P0 | §2 B2; R-gate; D1 ditunda |
| PR-04 | KEIN kedaluwarsa 23 Nov; status live unknown | P0 | V2-001; R15 (diterima owner) |
| PR-05 | Harga VPS 8 + storage belum terverifikasi | P1 | V2-009; R10; butuh panel user |
| PR-06 | Atribusi CPU buta (ingestion 0,0% artefak) | P1 | V2-003; B4; V2-003-R2 |
| PR-07 | Janitor mislabel 203 channel | P1 | R6; V2-004 tiket siap |
| PR-08 | Konfigurasi split V2/legacy + flag misterius | P1 | V2-003; OQ-010; D10 |
| PR-09 | Mekanisme kuota sync Drive belum terpecahkan | P1 | P2; 6 teori gugur |
| PR-10 | Web menjalankan compute berat in-process | P1 | B7; Fase 0 |
| PR-11 | Lock SQLite 31×/24 jam; schema belum diaudit | P1 | B8; V2-007 (gerbang teknis) |
| PR-12 | Stage 2 adalah gerbang throughput (I/O) | P1 | B5; canary Fase 1 |
| PR-13 | Kode web lawas (R5 belum dimuat di jalur API) | P2 | V2-002; restart web tertunda |
| PR-14 | Orphan tanpa feeder tidak terdeteksi (±17 jam) | P2 | R19 (RESOLVED); R8 observability |
| PR-15 | Master interim ±1 bulan (dukungan Contabo lambat) | P2 | Keputusan ~17 Des; PLAN §10 |
| PR-16 | Live mati massal 14:47 (4 stream/21 dtk) + pemulihan tidak andal (DB lock gagalkan reconnect; status desync; 706 tanpa reconnect; encoder ganda; relay task mati) | P0 | B10; R20; diagnosis 15:09–15:15 |
| PR-17 | Relink publik gagal di UI ("Failed to fetch") + sentinel buta (bot-check) membunuh siaran sehat tiap ±4 mnt; tidak ada relink otonom | P0 | B11; R21; diagnosis 19:01–19:08 |
| PR-18 | Kuota YouTube API 3 vault live habis (±21:25) dibakar lingkaran kill-relink; keempat live gelap di YouTube sejak ±21:30 meski dashboard STREAMING; panggilan live tidak terlacak di ledger/guard kuota | P0 | B12; R22; diagnosis 03:28–03:45 23 Nov |

---

## 4. Solusi yang diharapkan

Prinsip yang mengikat semua solusi: verifikasi independen sebelum
memperbaiki; jangan "memperbaiki" mekanisme yang belum terpecahkan
dengan tebakan (P2, B2-bot-check); setiap perubahan punya rencana
verifikasi pra/pasca + rollback + jendela observasi; peluncuran
menunggu instruksi owner.

### Fase 0 — $0, Master saja (segera)

| Solusi | Menyelesaikan | Status / gerbang |
|---|---|---|
| Kalibrasi admission TX (budget 115 → 500 Mbps) | PR-01 (sebagian: 5 penolakan reconnect 06:15) | DONE (R4, 07:24) |
| Perbaikan encoder R3–R8 + iterasi 2 (watchdog 10 dtk, queue 4096, rw_timeout 30 dtk, log exit code, alert restart, sapu orphan) | PR-01, PR-14 | DEPLOYED; observasi 24 jam GAGAL 14:47 (B10) |
| Investigasi + perbaikan jalur pemulihan live: selidiki penyebab bersama kematian massal; reconnect tahan DB-lock (retry + antrean); status STREAMING atomik dengan start encoder; satu jalur pemulihan (hapus race web vs supervisor); teardown relay saat task berhenti; selidiki sinyal supervisor | PR-16 | Investigasi antre; eksekusi menunggu **keputusan owner** |
| Perbaikan sentinel + relink: probe via YouTube Data API (atau UNVERIFIED saat bot-check — jangan bunuh); panggilan YouTube ke `to_thread` / relink async (job + polling); relink otonom saat broadcast terbukti berakhir; opsi matikan ghost-kill sementara | PR-17 | Diagnosis selesai 19:08; eksekusi menunggu **keputusan owner** |
| Hentikan bakar kuota live: deploy rem darurat sentinel (UNVERIFIED jangan bunuh) + restart supervisor; verifikasi 0 kill + PID stabil; tunggu reset kuota 15:00 WIB (ensurer relink otomatis); catat panggilan live di ledger/guard (tindak lanjut); rapikan encoder/feeders ganda | PR-18 | Patch + tes 10/10 SIAP; deploy menunggu **persetujuan owner** |
| Hentikan layanan box lama (akhiri split-brain) | Risiko data ganda | DONE (R1, 07:05) |
| Pindahkan reassemble keluar dari web ke worker | PR-10, B4 | Direncanakan Fase 0 |
| Perbaikan Guard v2 (V2-003-R1–R9: atribusi lane, executor shed_render, kalibrasi knob, admission V2) | PR-02, PR-06, PR-08 | Menunggu **D9 + D10** (OQ-010 prasyarat R1) |
| Audit janitor (V2-004) | PR-07 | Tiket lengkap; eksekusi antre (SSH) |
| Audit schema (V2-007) | PR-11 | Gerbang teknis sebelum migrasi apa pun |
| Restart web (muat R5 di jalur API) | PR-13 | Jendela maintenance berikut |

### Fase 1 — VPS 8 ($18,65/bln + $2,66 setup; total $34,67/bln)

| Solusi | Menyelesaikan | Status / gerbang |
|---|---|---|
| Topologi 2 box: Master (control + publish) + Live-01 (live + prep worker + DR) | B1, B5, B7 | Menunggu **V2-009** (harga panel) + persetujuan owner |
| Guard lokal per box (pengganti guard saling sandera) | PR-02, G1–G12 | Desain di V2.0; implementasi pasca-Fase 0 |
| Orkestrasi API + SQLite (Option A): register/claim/heartbeat/complete, lease + fencing token, idempotency | PR-03 (skala), B8 | Menunggu V2-007 (audit schema) |
| Canary Stage 2 (2 slot) + prep worker di VPS 8, diuji BERSAMAAN dengan reassemble | PR-12, B4, B5 | Acceptance test plan |
| Backup off-site independen (Restic + Litestream opsional) | Risiko kehilangan data | Tergate biaya storage |
| Monitoring fase 1: heartbeat + watchdog eksternal + alert Telegram + health check bisnis | Deteksi (PR-14) | Rencana di V2.0 |

### Fase 2+ — nanti (persetujuan anggaran terpisah)

- Box ke-3 bila live mendekati batas VPS 8 (70 channel @1080p).
- Evaluasi PostgreSQL bila trigger terpenuhi (lock >10/hari
  pasca-optimasi atau p95 tulis >100 ms konsisten).
- Prometheus/Grafana bila heartbeat + watchdog terbukti kurang.
- Keputusan provider Master interim (±17 Des 2026).

### Yang TIDAK menjadi solusi (ditolak dengan alasan)

- Redis/Celery/Kafka/microservices prematur — V2.0 melarangnya sampai
  metrik membuktikan kebutuhan (Option A dulu).
- Auto-retry pada `quotaExceeded` YouTube — dilarang; permanent error
  → DEAD_LETTER.
- Memindahkan vault antar channel tanpa re-pair — insiden 12 Nov
  membuktikan token patah + refresh ditolak Google.
- Menebak penyebab P2 (kuota Drive) — 6 teori gugur; langkah
  diagnostik saja sampai Cloud Console / audit fallback menjawab.

---

## 5. Peta problem → solusi → status (ringkasan)

| Bottleneck | Solusi utama | Fase | Status 22 Nov 15:15 |
|---|---|---|---|
| B1 jaringan live | VPS 8 (600 Mbps) + admission EMA | 1 | Menunggu V2-009 + owner |
| B2 gate ingestion | Kalibrasi gate + orkestrasi API | 0–1 | Akar terbukti; D1 ditunda |
| B3 governor impoten | Perbaikan Guard v2 (D9) | 0 | Menunggu D9/D10 |
| B4 reassemble tersembunyi | Anggaran ±1 core + pindah ke worker | 0–1 | Plan diperbaiki; eksekusi Fase 0 |
| B5 Stage 2 I/O | Canary 2 slot di VPS 8 | 1 | Driver diperbaiki; canary antre |
| B6 encoder mati | R3–R8 + iterasi 2 | 0 | Deployed; observasi GAGAL 14:47 (B10) |
| B7 web compute | Reassemble keluar dari web | 0 | Direncanakan |
| B8 lock SQLite | Audit schema V2-007 | 0 | Gerbang teknis; lock gagalkan reconnect 14:47 |
| B9 kuota/dependensi | Doktrin 5/vault + fallback dihargai | 0–1 | Doktrin mengikat; P2 terbuka |
| B10 mati massal + pemulihan | Investigasi + perbaikan jalur pemulihan | 0 | Dicatat 15:15; menunggu keputusan owner |
| B11 relink beku + sentinel buta | Probe API + relink async + relink otonom | 0 | Diagnosis selesai 19:08; menunggu keputusan owner |
| B12 kuota habis + live gelap | Rem sentinel + reset kuota 15:00 + lacak panggilan live | 0 | Diagnosis selesai 03:45 23 Nov; deploy rem menunggu persetujuan owner |

**Definisi "siap deploy":** Batch 1 10/10 + semua artefak §9 lengkap +
gerbang V2-007 + V2-009 lolos + D9/D10 diputuskan + observasi encoder
24 jam bersih + persetujuan owner. Observasi 24 jam GAGAL 14:47
(B10) — definisi ini belum terpenuhi. Sampai saat itu, repo ini
adalah paket persiapan — bukan instruksi eksekusi.
