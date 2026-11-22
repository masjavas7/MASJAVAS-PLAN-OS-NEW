# RUNBOOK OPERASIONAL — MASJAVAS TUBE OS

**Status:** DOKUMEN RENCANA + PROSEDUR BERLAKU. Bagian yang bersumber
dari audit/bukti 22 Nov ditandai; prosedur baru (monitoring eksternal,
VPS 8) adalah rancangan sampai Track P7/P8 dieksekusi.
**Sumber utama:** `MASTER_ARCHITECTURE_MASJAVAS_TUBE_OS_V2.md` §10
(governor), §11 (janitor), §13 (monitoring), §19 (failure scenarios);
`IMPLEMENTATION_CHECKLIST_..._V2.md` Track P8.2; `v2-003/` (audit guard);
`v2-002/ENCODER114_FIX_REPORT.md`; `ENGINEERING_READINESS_DASHBOARD.md`;
`PLAN_PENAMBAHAN_VPS8_2026-11-22.md` §2 (scorecard).
**Terakhir diperbarui:** 22 Nov 2026.

**Label bukti:** `[VERIFIED FACT]` / `[DOCUMENTED FACT NOT YET REVALIDATED]` /
`[ASSUMPTION]` / `[PROPOSAL]` / `[OPEN QUESTION]` /
`[OWNER DECISION REQUIRED]`.

**Otoritas:** L1 diagnostik otomatis; L2 perbaikan risiko rendah
otomatis (dengan backup + tes); L3 (skema, migrasi, hapus data, auth,
firewall, SSH, infra produksi, refactor besar) butuh persetujuan user
per kasus; L4 (hapus backup/DB, bocorkan secret, nonaktifkan keamanan,
hapus audit log, perintah destruktif) DILARANG. Sebelum perbaikan apa
pun: aplikasi harus tetap jalan untuk operator lain, pre-flight health
check, verifikasi pasca-perubahan, rollback siap.

---

## 1. Lanskap kini (satu box)

- **Master live:** `masjavas-vps-baru` (Contabo vmi3762689, 4 vCPU /
  8 GB / 145 GB / port 500 Mbps). `[VERIFIED FACT]`
- **Service aktif (4):** `masjavas-os-web`, `masjavas-os-worker`,
  `masjavas-live`, tunnel. `[VERIFIED FACT — 22 Nov 11:33 WIB]`
- **Box lama Tencent:** layanan aplikasi DIHENTIKAN 22 Nov 07:05 (R1).
  Jangan pernah membaca status produksi dari box lama — datanya stale
  sejak migrasi 17 Nov. `[VERIFIED FACT]`
- **Armada:** 707 slot (NO 1–707), CONNECTED 237, limit live 16.
  `[VERIFIED FACT]`
- **VPS 8 (Live-01) BELUM DIBELI.** Semua prosedur VPS 8 di runbook
  ini adalah rancangan. `[VERIFIED FACT — status dashboard]`

## 2. Health check harian

| # | Cek | Cara | Nilai normal | Eskalasi bila |
|---|---|---|---|---|
| H1 | Healthz publik | `https://masjavastube.my.id/healthz` | 200, p95 ±10 ms `[VERIFIED FACT — baseline]` | Non-200, atau p95 >100 ms (ambang scorecard). |
| H2 | Service aktif | `systemctl is-active` ke-4 service | Semua `active` | Satu pun tidak aktif. |
| H3 | Load Master | `uptime` / tick governor | Rata-rata 3,5–4,2 (baseline); target <3,0 pasca-Fase 0 | Load >4,2 menetap, atau naik bersamaan dengan publish tertunda. |
| H4 | Disk Master | `df` | 83/145 GB (58%) pada 22 Nov 11:33 `[VERIFIED FACT — snapshot hari itu]`; puncak wave 72% (baseline) | >70% (ambang scorecard) atau laju burn tidak wajar. |
| H5 | RAM tersedia | `free` | 3,6 GB tersedia pada 22 Nov 11:33 `[VERIFIED FACT — snapshot]` | <2,5 GB (gate ingestion mematok konkurensi ke 1) atau <4,0 GB (cap 2). |
| H6 | Guard tick | Journal `[Guard tick]` | Tick tiap ~30 dtk; `decision=none` saat normal | Tidak ada baris tick >5 menit (guard loop mati — V2-003-R6). |
| H7 | Publish progress | Dashboard/DB: PUBLISHED/hari, FAILED/BLOCKED | 206/hari, 0 FAILED/BLOCKED (baseline) | FAILED/BLOCKED >0, atau publish/hari <200. |
| H8 | SQLite lock | Hitung `database is locked` di log | 31×/24 jam (baseline); target <20 lalu <10 | Lonjakan di atas baseline tanpa sebab. |
| H9 | Stream live | Status DB + proses encoder | STREAMING + encoder hidup + feeder aktif per stream | Status ERROR, encoder mati berulang, atau stream tanpa feeder (lihat §4). |
| H10 | Backup freshness | Cek backup terakhir + hasil restore terakhir | Sesuai jadwal (rencana backup) | Backup tertinggal >1 jadwal, atau drill bulanan terlewat. |

**Business health check (V2.0 §13.3):** jangan hanya memeriksa HTTP
200. Periksa publish progress, job RUNNING terlalu lama, heartbeat
worker, proses encoder, progress FFmpeg, koneksi RTMP, status
broadcast YouTube, stream yang selesai tetapi FFmpeg masih hidup,
upload yang belum direkonsiliasi, disk staging, live TX, frekuensi
lock SQLite, dan backup terakhir. Daftar lengkap ini adalah Track P8.2
(belum diimplementasikan — `[PROPOSAL]`).

**Monitoring eksternal (V2.0 §13.1, Track P8.1):** Healthchecks.io /
UptimeRobot + alert Telegram. Status: `[PROPOSAL — belum
diimplementasikan]`. Prometheus/Grafana adalah tahap berikutnya
(§13.2), bukan prasyarat.

## 3. Governor — perilaku yang diketahui (V2-003)

Temuan audit 22 Nov 07:15–07:40 (host live, read-only). Pahami ini
SEBELUM mempercayai apa pun yang dikatakan governor:

1. **Governor adalah termometer, bukan termostat.** Dalam 17 jam era
   V2: 1.918 tick → 1.886 none (98,5%), 29 `shed_render`, 0
   `shed_ingestion`/pause/resume. **0 aksi efektif.** Satu-satunya
   aksi yang dipilih tidak punya eksekutor (log-only BY DESIGN).
   `[VERIFIED FACT — V2-003]`
2. **Konfigurasi TERBELAH:** guard loop = V2 (governor aktif);
   admission = LEGACY karena `LIVE_GUARD_V2_ENABLED=false` di `.env`
   hanya memengaruhi `can_start`, bukan guard loop. Jangan simpulkan
   guard nonaktif dari flag itu. `[VERIFIED FACT — V2-003]`
3. **Atribusi ingestion BUTA:** `classify_lane` hanya mengenali
   cmdline berisi "ingestion"; daemon otonom (±18% CPU) tercatat
   lane `other` → `ingestion_cpu_pct` selalu 0,0% → governor selalu
   memilih `shed_render` (buntu). **CPU ingestion 0,0% adalah
   artefak atribusi, BUKAN bukti ingestion mati** (ingestion bekerja:
   29 COMPLETED/24 jam, semua DRIVE_SYNC). `[VERIFIED FACT — V2-003]`
4. **Q6 SOLVED:** admission legacy memakai TX instan 5-detik; guard
   tick memakai EMA 300-detik; sampler yang sama. Dengan EMA, 5
   penolakan admission 06:15 tidak terjadi. Anggaran TX sudah
   dikalibrasi ulang ke 500 Mbps (R4, 22 Nov 07:18). `[VERIFIED FACT]`
5. **Gate ingestion mencekik:** 5 cap bertumpuk → konkurensi 1
   permanen; 0 stage berat selesai/24 jam. `[VERIFIED FACT — V2-003]`
6. **Knob bermasalah:** `LIVE_MAX_CPU_PERCENT=90` inert (jalur legacy
   memakai hard-coded 95); `LIVE_DEFAULT_RESOLUTION` 720p stale vs
   keputusan user 1080p-untuk-semua. `[VERIFIED FACT — V2-003]`
7. **Guard loop bertahan dari exception** (13 skip/24 jam, tercatat,
   tanpa alert); snapshot basi → fail-safe NONE; degradasi sampler
   hampir tak terlihat (level DEBUG). `[VERIFIED FACT — V2-003]`

**Perbaikan yang menunggu keputusan owner (D9/D10):** V2-003-R1–R9
(prioritas R1/R2 admission TX). Prasyarat R1: jawab OQ-010 (siapa
men-set flag `false` + CPU 90, dan mengapa — `.env` diubah 21 Nov
17:58). Jangan aktifkan admission governor sebelum OQ-010 terjawab.

**Yang normal vs tidak normal:**

| Sinyal | Normal | Tidak normal → tindakan |
|---|---|---|
| Tick `decision=none` | Ya, 98,5% tick | — |
| `shed_render` sesekali | Diharapkan saat overload render-dominan — TAPI ingat: ini no-op, bukan perbaikan | >10×/24 jam dengan 0 aksi tereksekusi → alert (V2-003-R6, belum diimplementasikan). |
| `ingestion_cpu_pct` 0,0% | SELALU (artefak) — jangan pakai sebagai bukti | — (perbaiki atribusi: V2-003-R3). |
| Tick hilang >5 menit | Tidak | Guard loop mati → selidiki supervisor (V2-003-R6). |
| `Error dalam loop supervisor` | Tidak (13×/24 jam tercatat tanpa eskalasi) | Counter + alert (V2-003-R6, belum diimplementasikan). |

## 4. Respons kematian encoder

### 4.1 Deteksi

- **Log R6 (sejak commit `56f7a6d`):** `Stream #<task> (PID <pid>)
  mati — exit code <n>. Ekor log:` + 20 baris terakhir stderr ffmpeg.
  Untuk orphan/tanpa handle Popen: exit code tidak tersedia, hanya
  ekor log. `[VERIFIED FACT — live sejak 22 Nov 07:19]`
- **Alert R7:** >3 restart dalam 60 menit untuk satu task →
  `send_telegram_alert` (dedup per task, cooldown 1 jam).
  `[VERIFIED FACT — live sejak 22 Nov 07:19]`
- **Watchdog feeder R3b (iterasi 2, commit `021a9f0`):** jendela
  stall 10 dtk, sampling 2 dtk, ambang 100 KB/jendela (sehat
  ±1,7 MB/10 dtk). Log: `[LiveRelay] Watchdog: feeder primer stall
  ... — me-restart feeder.` + `Primary feeder berakhir
  (alasan=watchdog-stall, exit=-9).`
  `[VERIFIED FACT — live sejak 22 Nov ~08:20]`
- **Tanda kematian yang diketahui:** `[aost] Error submitting a
  packet to the muxer: Connection timed out` → `Error muxing` →
  `Error writing trailer: Connection timed out` (tanda RTMP timeout,
  kematian #14, 22 Nov 08:09:50, exit code 146, runtime 44:31).
  `[VERIFIED FACT]`

### 4.2 Triase (urutan baca bukti)

1. Baca ekor R6 lengkap (20 baris) — cari baris error PERTAMA
   (bukan hanya baris statistik). Pada kematian #14, 10 baris
   pertama hanya statistik normal; error ada di baris ke-11+.
2. Baca journal relay di sekitar waktu kematian: baris `Watchdog:`,
   `Primary feeder berakhir (alasan=...)`, `Siaran live sumber
   terputus!`, `KEMBALI ONLINE!`.
3. Baca `relay_<task>.log` (ekor): error HLS keepalive / proxy 503
   menunjukkan sumber/relay bermasalah, bukan encoder.
4. Cek tick governor di saat kematian: bila CPU/TX normal, kematian
   BUKAN masalah resource (kematian #14: CPU 62%, TX 12 Mbps).
5. Cek status DB task: STREAMING / RECONNECTING / ERROR.

### 4.3 Pemulihan

- **Jalur normal:** health check supervisor mendeteksi kematian →
  reconnect otomatis → "berhasil mengudara" (verifikasi: PID baru,
  1080p, relay terhubung, feeder aktif). `[VERIFIED FACT]`
- **Jebakan ERROR-race (insiden 22 Nov 07:19):** bila watchdog web
  sudah menandai task ERROR (setelah 5× pemulihan gagal), supervisor
  MENGABAIKAN task itu — pemulihan otomatis TIDAK terjadi.
  Pemulihan: verifikasi file video ada → flip 1 baris DB
  ERROR → STREAMING (pid 0) → health check mendeteksi → reconnect.
  Alasan 5 kegagalan auto-healer pada insiden itu TIDAK DIKETAHUI
  (tidak ada log Auto-Healing di journal mana pun) — catat bila
  terulang, jangan asumsikan sebabnya.
- **Orphan (R8 sweep):** sweep startup mencatat encoder orphan
  (PPID 1) + status feedernya. Orphan TANPA feeder = mengudara
  tanpa audio (kasus 706: ±17 jam video-only, bitrate tetap
  HEALTHY — tidak terdeteksi metrik). Restart orphan HANYA atas
  persetujuan owner (D11: downtime ±48 dtk, audio terverifikasi
  via wchar feeder ±19,4 MB + relay terhubung).

### 4.4 Larangan

- JANGAN membunuh proses feeder relay tanpa mengidentifikasi
  feeder milik manager yang sedang aktif (feeder lama yang bocor
  menulis ke inode FIFO lama yang sudah tidak terpakai).
- JANGAN membunuh encoder yang sedang mengudara tanpa rencana
  pemulihan (persetujuan owner + jalur recovery supervisor
  terverifikasi).
- JANGAN menyimpulkan "encoder sembuh" dari satu restart sukses —
  observasi 24 jam pasca-deploy adalah metriknya (target: 0
  kematian tak terjelaskan + watchdog MENYELAMATKAN stall
  berikutnya, bukan sekadar mencatat).

## 5. Janitor

- **Mode saat ini: status-only** (Fase 0 langkah 0.6; janitor belum
  dijalankan mode hapus). Target: 0 GB file channel CONNECTED
  terhapus berlabel "unpaired" dalam audit 24 jam.
  `[VERIFIED FACT — dashboard R6]`
- **Risiko R6 (P0, OPEN):** janitor mislabel 203 channel CONNECTED.
  Audit V2-004: tiket lengkap (Cycle 5), eksekusi menunggu sesi
  interaktif + SSH. Baseline reklasifikasi 40,92 GB/24 jam.
  `[VERIFIED FACT — dashboard]`
- **8 cek pra-hapus (V2.0 §11, mengikat):** owner file, status job,
  referensi database, generation, umur file, status lifecycle,
  apakah file sedang digunakan proses, apakah ada hasil valid yang
  masih membutuhkan file. Janitor TIDAK BOLEH menghapus hanya karena
  metadata pasangan tidak ditemukan (Checklist §0.2).
- **Prosedur:** kandidat cleanup → quarantine → catat alasan +
  metadata → grace period → pemeriksaan ulang → hapus hanya bila
  semua syarat terpenuhi → catat hasil.
- **Tangkap `deletion_audit` SEGERA saat eksekusi V2-004** — retensi
  tabel ini belum diketahui (OQ-014). `[OPEN QUESTION]`
- **Kepemilikan janitor** (ikut unit publish vs layanan sendiri)
  masih `[OWNER DECISION REQUIRED — Plan §10 #5]`.

## 6. Doktrin kuota (mengikat)

| Aturan | Nilai | Sumber |
|---|---|---|
| Post per channel | 1/hari | Doktrin 18 Nov |
| Channel per vault (maks) | 5 | 5 × 1.650 = 8.250 unit < guard 8.400 |
| Biaya 1 video | 1.650 unit | Per project: 10.000 unit/hari, 100 videos.insert/hari |
| Vault yang dibutuhkan | ~140 untuk 696 channel; 48 untuk 197 aktif; live 37 | Koreksi kuota 21 Okt |
| `quotaExceeded` YouTube | JANGAN auto-retry | Checklist §0.2; V2.0 §8.5 + §19: pause dan tunggu reset |
| Pindah channel antar vault | Hanya dengan izin operator + prosedur 4 langkah + re-pair | Doktrin 18 Nov + insiden 12 Nov |

**Respons `quotaExceeded`:** pause pemakaian kuota itu, tunggu reset,
jangan retry (retry membakar kuota). Insiden 18 Nov: 217 post ekstra
membakar 358.050 dari 444.350 unit; 121 jadwal terlewat.
`[VERIFIED FACT]`

**Respons kuota Drive (P2):** mekanisme sync quota BELUM DIKETAHUI
(6 teori tereliminasi 22 Nov) — langkah diagnostik saja, jangan
"perbaiki" dengan tebakan. `[VERIFIED FACT]`

## 7. Ingestion & LoopGen — keadaan yang diketahui

- **Ingestion:** konkurensi 1 permanen (5 cap bertumpuk, §3.5);
  churn 66–67% (SUPERSEDED vs COMPLETED); 0 stage berat selesai/
  24 jam; bottleneck = gate + bot-check yt-dlp, bukan CPU.
  `[VERIFIED FACT — V2-003]`
- **D1 (proxy residential Dahono 407):** keputusan user DITUNDA
  20 Nov — JANGAN tanya ulang. Alternatif termurah yang diteliti:
  WaroengProxy Rp 10.000/GB (belum diuji).
  `[VERIFIED FACT — dashboard D3]`
- **KEIN:** plan berakhir 23 Nov 2026. Status pada sistem LIVE
  TIDAK DIKETAHUI (audit V2-001 di host salah); user menutupnya
  dengan asumsi "KEIN aman" (06:55 + 07:03, 22 Nov) dan MENOLAK
  audit ulang. JANGAN tawarkan audit/top-up KEIN lagi kecuali user
  mengangkatnya. `[VERIFIED FACT — dashboard R15/D1]`
- **Reassemble audio:** 64×/24 jam, ±0,9 core/job, ±6 mnt/job,
  ±70 MB/job — satu wadah dengan ingestion (addendum 22 Nov 12:10).
  CPU-nya tercatat di bucket render/web (artefak atribusi yang
  sama dengan §3.3).
- **Stage 2:** 12,8 dtk CPU vs 100,1 dtk wall per klip (I/O
  dominan); 1.011 DONE/7 hari; antrean 50 QUEUED + 4 PENDING.
  `[VERIFIED FACT]`

## 8. Live — operasi

- **Resolusi:** 1080p untuk semua channel (keputusan user; default
  `LIVE_DEFAULT_RESOLUTION` 720p di `.env` masih stale — V2-003-R8).
- **Admission:** budget = `LIVE_MAX_TX_MBPS` (500) × 70% = 350 Mbps.
  Admission legacy memakai TX instan 5-detik (Q6); penolakan palsu
  dimitigasi R4, perbaikan permanen menunggu D9/D10.
- **Prioritas (V2.0 §10.4, Live-01):** live berjalan > supervisor/
  recovery > control/health agent > ingestion > Stage 2 > backup.
  Bila live terancam: hentikan admission job baru → turunkan
  konkurensi prep → throttle upload → hentikan Stage 2 bila perlu →
  JANGAN membunuh live yang sehat karena prep butuh resource.
- **Limit live:** 16 (global, sejak 21 Nov). Limit final (usul
  60/20) masih `[OWNER DECISION REQUIRED — Plan §10 #6]`.

## 9. Eskalasi ke owner

| Situasi | Tindakan |
|---|---|
| Metrik scorecard kritis memburuk | Rollback (bila pasca-perubahan) + laporkan dengan angka. |
| FAILED/BLOCKED >0 | Selidiki + laporkan; bukan angka yang bisa didiamkan. |
| Kematian encoder tak terjelaskan | Kumpulkan bukti R6 (§4.2) sebelum melapor; jangan klaim sebab tanpa bukti. |
| Keputusan tertunda | D9 (Guard v2), D10 (OQ-010), D2 (pilot DeepInfra Rp 800), D4 (pembelian VPS 8), D5 (Plan §10), D6 (launch Stage 2 live). Jangan tanya ulang D1 (KEIN — ditutup) atau D3 (D1 proxy — ditunda). |
| Level C apa pun | Persetujuan eksplisit per kasus sebelum eksekusi. |

## 10. Template log insiden (ringkas)

```
Waktu (WIB): ...
Gejala + bukti (angka/log):
Dampak (stream/channel/publish):
Tindakan yang diambil (perintah + hasil):
Status akhir (pulih/belum + verifikasi):
Tindak lanjut + pemilik:
```

Catatan: laporan subagent/operator = ringkasan + angka + path + nomor
baris — tanpa log mentah (Context Mode, 18 Nov).
