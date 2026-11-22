# PLAN PENAMBAHAN 1 VPS 8 — MASJAVAS TUBE OS
### (Rancangan ulang dari seluruh temuan 22 Nov 2026)

**Tanggal:** Minggu, 22 Nov 2026, 04:09–04:35 WIB
**Status:** PLAN — menunggu keputusan user. Tidak ada pembelian atau
perubahan yang dilakukan.
**Dokumen ini MENGGANTIKAN** `RANCANGAN_INFRASTRUKTUR_2_VPS_2026-11-22.md`
sebagai plan of record penambahan VPS 8 (dokumen lama tetap berlaku
sebagai analisis opsi + Lampiran A/B). Dokumen ini mengintegrasikan
seluruh temuan malam ini: health check, diagnosis bottleneck, verifikasi
B1–B7, evaluasi P1–P16, verifikasi CPU, audit modal, dan rencana
orchestrator.
**Perintah user:** rancang ulang plan penambahan 1 VPS 8 dengan konteks
problem + tujuan yang diperjelas, untuk memenuhi kebutuhan modal semua
worker agar tidak saling sandera dan bisa di-orchestrator dengan baik.
**Goal: aplikasi MAKIN SOLID dari sekarang.**

---

## 0. Ringkasan eksekutif

**Keputusan yang diusulkan:** tambah **1× Contabo Core VPS 8**
(8 vCPU / 24 GB / 300 GB / 600 Mbit/s; $18,65/bln incl. PPN 11%) —
**TETAPI beli saat pemicu demand (live mendekati 12–16 stream), bukan
sekarang** (baru 2 stream, 6 stream key). Total berjalan $34,67/bln,
sesuai anggaran mengikat 21 Nov.

**Inti plan:** 3 bagian logika (publish / isi bahan / live) di 2 box
fisik. Master = control plane + publish + prep (2 proses terpisah).
VPS 8 = live + worker prep (ingestion + Stage 2) + DR replica +
watchdog. Setiap worker mendapat modal yang terverifikasi (§4), setiap
guard bersifat lokal sehingga **tidak ada guard saling sandera**,
dan orchestration berjalan pull-based lewat API (§5).

**Urutan:** Fase 0 ($0, Master) dulu — pisah proses, keluarkan job berat
dari web, perbaiki churn, lokalkan guard. Fase 1 (VPS 8) setelah Fase 0
lolos gerbang + pemicu demand tercapai. Fase 2 (box ke-3) hanya bila
dana + kebutuhan membenarkan (persetujuan terpisah).

**Goal "makin solid" diukur dengan scorecard §2** — setiap fase harus
mempertahankan atau memperbaiki SEMUA metrik baseline; satu metrik yang
memburuk = gerbang gagal = rollback.

---

## 1. Konteks problem (semua terverifikasi malam ini)

### 1.1 Satu box = satu domain kegagalan + guard saling sandera

Seluruh aplikasi (web, worker publish, ingestion, LoopGen, live,
tunnel) berbagi 1 box Contabo Plus 4 (4 vCPU / 8 GB / 145 GB / 500
Mbit/s) + 1 file SQLite. Beban satu lane menjadi hukuman bagi lane
lain lewat 12 guard yang terverifikasi (G1–G12, detail di
`RENCANA_ORCHESTRATOR_2026-11-22.md` §3):

- **G1–G5 (ingestion dicekik):** load gate (konkurensi ↓ saat load1
  ≥2,0; claim ditolak saat >2,8), shed flag governor, RAM gate (≥4 GB;
  tersedia 3,2 GB), disk gate (≥40 GB bebas; sempat 39–41 GB saat
  puncak wave), broadcast window 17:00–22:30 (5,5 jam/hari berhenti).
  Load Master 3,5–4,2 → semua gate menutup. Bukti: hanya 2 RUNNING +
  1 PENDING padahal backlog 412 channel / ±28.461 item.
- **G2 asimetris:** governor meminta render mengalah 112×/24 jam
  (`shed_render`) — semuanya log-only, tanpa eksekutor. Render tidak
  pernah mengalah; ingestion selalu yang kalah.
- **G6 (Stage 2 mengalah):** yield saat load1 ≥2,0 / load15 ≥3,0 +
  flag `STUDIO_RENDER_BUSY` (in-process). 20 Nov: 0 DONE ±4,5 jam
  sebelum starvation breaker.
- **G9 (churn):** 333 job ingestion SUPERSEDED vs 123 COMPLETED dalam
  7 hari (66%) — sweeper + Fleet Runner + retry membuat job baru untuk
  channel yang job-nya masih berjalan; kerja terbuang.
- **G10 (DB):** "database is locked" 31×/24 jam (live 26×) — health
  check live terlewat karena kontensi tulis.
- **G12 (satu loop):** publish + ingestion + LoopGen + janitor dalam
  1 proses Python + 1 event loop (`masjavas-os-worker`).

### 1.2 Live terjepit jaringan, bukan CPU

- Port Master 500 Mbit/s terukur 91–93%. Live 50 channel @1080p butuh
  ±310 Mbps egress terus-menerus (±100 TB/bln).
- TX box melonjak ke 60 Mbps (52% acuan governor) dengan hanya 2 stream
  — lalu lintas non-live (upload/sync) ikut memakai port yang sama.
- Live sendiri hampir nol CPU: 0,07 core untuk 2 stream (±2–6% CPU per
  stream). CPU tidak pernah menjadi kendala live.
- Contabo TIDAK menjual add-on bandwidth/port untuk Cloud VPS (hanya
  VDS); naik paket Plus 4 → Plus 8 (+€21,50/bln) lebih mahal dari VPS 8
  kedua (+€14,00/bln) dan tanpa isolasi (Lampiran A rancangan lama).

### 1.3 Ingestion kurang ngebut (bukan karena CPU)

- CPU ingestion 0,0% di semua 1.510 tick governor 24 jam. Bottleneck:
  gate (§1.1) + bot-check yt-dlp (WARP-only, D1 belum selesai) + feed
  sweeper batch 2/120 dtk (±1 channel/menit).
- **KOREKSI 22 Nov 12:10 (Bapak):** angka 0,0% itu artefak atribusi
  governor — CPU perakitan audio (reassemble, 64×/24 jam, ±1 core/job)
  tercatat di bucket render/web, bukan ingestion, padahal reassemble
  satu wadah dengan ingestion (ingestion.py memanggil
  `process_and_reassemble_ingestion_audio`). Modal CPU ingestion yang
  benar mencakup reassemble — lihat §4.
- Throughput: 123 COMPLETED/7 hari (rata-rata 17/hari; terbaik 57/hari
  pada 21 Nov). SKIPPED_AUDIO_MISSING melonjak 3 → 31/hari (20–21 Nov)
  seiring krisis bot-check.

### 1.4 LoopGen Stage 2 berebut core + I/O

- Tahap 1/2 API-bound (KEIN/Gemini; lokal ±0). Stage 2: edit 12,8 dtk
  CPU/klip, total wall 100,1 dtk/klip (I/O dominan — unduh + unggah +
  QA), max 900,8 dtk. 1.011 DONE/7 hari; antrean kini 50 QUEUED.
- Butuh SLOT KONKURENSI + jaringan Drive, bukan CPU mentah. Driver
  standalone idle (KANDIDAT 0; 10 channel-nya 300/300 selesai).

### 1.5 Web adalah compute tersembunyi

- Unit web rata-rata **1,02 core** — bukan HTTP, melainkan job ffmpeg
  berat in-process: reassemble audio (64×/24 jam), thumb regen, fleet
  run, AI video pipeline. 2 anak ffmpeg @ ±98% teramati 2× snapshot.
- Peta CPU Master: render ±1,9 + web ±1,0 + daemon ±0,2 + live ±0,07 +
  Stage 2 ±0,06 ≈ 3,2 dari 4 core → load 3,5–4,2.

### 1.6 Live tidak stabil (di luar soal box)

- Encoder slot 706 mati mendadak **5×/24 jam** (pola berulang; akar
  penyebab belum didiagnosis; auto-restart ±15 dtk tiap kali).
- 1 encoder ORPHAN (18 jam, parent PID 1) berjalan tanpa pengawas.
- Traceback live = DB-locked di health check (`live_supervisor.py:878,
  920`). Pindah box TIDAK memperbaiki ketiganya — perlu perbaikan
  terpisah (P16).

### 1.7 Baseline "solid" yang harus dilindungi

21 Nov: **206 PUBLISHED, 0 FAILED/BLOCKED** (237 tugas: +31 skip audio).
Median keterlambatan terbit 11,6 menit (p90 30,2 menit). Healthz 200
dalam ±10 ms. Janitor aktif (121 GB terhapus/24 jam; disk membaik
104 → 81 GB terpakai semalam). **Perbaikan yang menurunkan angka ini
adalah perbaikan yang GAGAL.**

---

## 2. Tujuan — "makin solid dari sekarang" (scorecard terukur)

"Solid" bukan slogan — ini scorecard-nya. Baseline = kondisi
terverifikasi 21–22 Nov. **Setiap fase harus mempertahankan atau
memperbaiki SEMUA baris; satu baris memburuk = gerbang gagal.**

| # | Metrik | Baseline (21–22 Nov) | Target setelah Fase 0 | Target setelah Fase 1 |
|---|---|---|---|---|
| 1 | Publish/hari | 206 (21 Nov) | ≥200, 0 FAILED | ≥200, 0 FAILED |
| 2 | Keterlambatan terbit p90 | 30,2 menit | ≤30 menit | ≤25 menit |
| 3 | Kematian encoder tak direncanakan | 5×/24 jam (slot 706) | ≤1×/24 jam (setelah P16) | 0×/72 jam |
| 4 | Encoder orphan | 1 | 0 | 0 |
| 5 | "database is locked"/24 jam | 31× | <20× | <10× |
| 6 | Load Master (rata-rata) | 3,5–4,2 | <3,0 | <2,5 |
| 7 | CPU unit web (rata-rata) | 1,02 core | <0,2 core | <0,2 core |
| 8 | Ingestion RUNNING saat backlog ada | 2 job | ≥3 job (konkurensi penuh) | ≥3 di Master + penuh di VPS 8 |
| 9 | Churn SUPERSEDED ingestion | 66% (7 hari) | <20% | <20% |
| 10 | Disk Master (puncak wave) | 72% | ≤70% | ≤70% |
| 11 | Healthz p95 | ±10 ms | <100 ms | <100 ms |
| 12 | Stream live aktif | 2 | 2 (stabil) | Sesuai demand, s.d. 16 → 50 |
| 13 | Error class baru pasca-fase | — | 0 dalam 72 jam observasi | 0 dalam 72 jam observasi |

---

## 3. Prinsip desain (doktrin yang mengikat)

1. **Tidak ada guard saling sandera** (user 22 Nov 04:01). Guard hanya
   menjaga sumber daya box/prosesnya sendiri.
2. **Pembagian 3 bagian** (user 22 Nov 02:50): (1) PUBLISH: download +
   rakit + upload; (2) ISI BAHAN: ingestion + LoopGen + isi folder
   GDrive branding; (3) KHUSUS LIVE STREAMING.
3. **Drive = media serah-terima.** Prep menulis bahan (01–04); publish
   membaca. Tidak ada transfer bahan antar-box selain lewat Drive/API.
4. **SQLite hanya di 1 box (Master).** Tidak pernah dibuka lewat
   jaringan. Worker remote memakai API claim/submit.
5. **Live prioritas tertinggi di box-nya.** Co-tenant tidak pernah
   boleh mengganggu live; bila mengganggu, co-tenant dimatikan dulu.
6. **Pull, bukan push.** Worker mengambil job saat punya kapasitas
   (klaim + lease). Satu owner per (channel, kind).
7. **Setiap perubahan terverifikasi** (aturan 22 Nov 01:48): rencana
   verifikasi pra/pasca + rollback + observasi 72 jam. Tanpa itu,
   tidak ada implementasi.
8. **Bertahap karena dana.** 2 VPS dahulu; box ke-3 hanya bila dana +
   kebutuhan membenarkan.

---

## 4. Modal semua worker (kebutuhan terverifikasi)

| Worker | CPU | RAM | Disk | Jaringan | Akses data | Box |
|---|---|---|---|---|---|---|
| **Publish** (scheduler + render + upload) | 2–3 core (render 2+1 slot) | 4 GB | 60 GB kerja + bahan | Unduh Drive (RX) + upload YouTube (TX) | DB langsung | Master |
| **Ingestion** | ±0 (0,0% terukur) | ≥4 GB tersedia (gate) | 100 GB (staging + salinan pool 20,5 GB) | Unduh yt-dlp (RX) + sync Drive (TX, throttle) | API claim/submit | VPS 8 (Fase 1); Master sementara |
| **Reassemble audio** (kurasi; satu wadah ingestion) | ±1 core puncak (0,25 rata-rata; 64×/24 jam, ±6 mnt/job) | 0,1 GB/job | — (kerja di staging ingestion) | — | API claim/submit | VPS 8 (Fase 1, ikut prep worker) |
| **LoopGen Stage 2** | 1 core/render; cgroup max 4 core | 4 GB | 50 GB (kerja + cache; data LoopGen 30 GB) | Unduh/unggah Drive (TX, throttle) | API claim/submit | VPS 8 (Fase 1) |
| **LoopGen Tahap 1/2** | ±0 (API-bound) | ±0 | ±0 | API KEIN/Gemini | API provider | Di mana saja (orchestrator Master) |
| **Live** (s.d. 50 stream) | 1–3 core | 1 GB | 5 GB (file loop) | RTMP TX 310 Mbps (prioritas tertinggi) | API status + state lokal | VPS 8 |
| **Web/API** (setelah P15) | <0,2 core (HTTP murni) | 2 GB | 5 GB | API + dashboard | DB langsung | Master |
| **Control plane** (dispatcher + governor) | <0,5 core | 1 GB | — | Tailscale ke VPS 8 | DB langsung | Master |
| **DR replica + watchdog** | ±0 | <1 GB | 100 GB (salinan app + DB + aset) | Sync malam hari | rsync/API | VPS 8 |

**Kecukupan box:**

- **Master (4 vCPU / 8 GB / 145 GB / 500 Mbit/s):** publish ±2,5 +
  web ±0,2 + control ±0,5 ≈ 3,2 core — muat SETELAH job web keluar
  (Fase 0) dan prep pindah (Fase 1). Sebelum Fase 0, Master jenuh
  (3,2 core terpakai + prep menumpang).
- **VPS 8 (8 vCPU / 24 GB / 300 GB / 600 Mbit/s):** live ±2 + Stage 2
  (cgroup 4) + ingestion ±0 + reassemble ±1 (puncak) + DR ±0 ≈ 7 core
  puncak; RAM ±6 GB (reassemble ±0,1 GB/job); disk ±155 GB;
  TX live 310 Mbps + prep throttle ≤100 Mbps < 420 Mbps (70% port).
  Muat, headroom menyempit ke ±1 core pada puncak (KOREKSI 22 Nov 12:10:
  reassemble sebelumnya tidak masuk anggaran — lihat §1.3).

---

## 5. Arsitektur target

### 5.1 Master — control plane + publish + prep (sementara)

```
Master — Contabo Plus 4 (4 vCPU / 8 GB / 145 GB / 500 Mbit/s)
├── masjavas-os-web        : API + dashboard + API agent (HTTP murni)
├── masjavas-os-worker-publish : PROSES 1 — scheduler, unduh bahan,
│                              render (ffmpeg 2+1), upload YouTube
├── masjavas-os-worker-prep    : PROSES 2 — ingestion + LoopGen
│                              (PINDAH ke VPS 8 di Fase 1)
├── SQLite (WAL)           : SATU-SATUNYA pemegang DB
├── janitor                : pembersihan workspace (milik publish)
├── masjavas-os-tunnel     : Cloudflare Quick Tunnel
└── Tailscale              : jaringan privat ke VPS 8 ($0)
```

### 5.2 VPS 8 (Live-01) — live + prep + DR

```
Live-01 — Contabo Core VPS 8 (8 vCPU / 24 GB / 300 GB / 600 Mbit/s)
├── masjavas-live.service  : supervisor + ffmpeg (stream-copy → YouTube)
├── Network governor lokal : admission tolak stream baru bila
│                              TX total + bitrate > 70% port (420 Mbps)
├── masjavas-prep-worker   : ingestion + Stage 2 (claim via API Master)
│   ├── cgroup: Stage 2 max 4 core; ingestion tanpa batas CPU (±0)
│   └── TX throttle: unggahan Drive dijeda saat TX live >50%
├── DR replica             : sync malam (app + DB backup + aset kritis)
├── watchdog               : monitor Master dari luar + alarm
└── RTMP langsung ke YouTube (tidak pernah lewat Master/tunnel)
```

### 5.3 Antarmuka antar-box (Tailscale, $0, sudah jalan)

| Alur | Arah | Mekanisme |
|---|---|---|
| Klaim + lapor job (ingestion/Stage 2) | VPS 8 → Master API | REST + lease + heartbeat (pola agent LoopGen) |
| Penugasan shard live | Master → VPS 8 | Single-owner + lease + fencing token |
| Heartbeat live | VPS 8 → Master | 30 dtk; stale bila 90 dtk tanpa heartbeat |
| Bahan | Prep → Drive → Publish | Drive sebagai media (tidak ada transfer langsung) |
| DR sync | Master → VPS 8 | rsync malam hari |
| Governor | Lokal per box | TIDAK ADA shed antar-box |

### 5.4 Guard lokal per box (pengganti guard saling sandera)

| Guard | Box | Ambang | Bila trip |
|---|---|---|---|
| Slot render | Master | 2 + 1 longform | Antre (bukan bunuh) |
| Kuota vault | Master | 8.400 unit (guard; 5 ch/vault) | Tunda publish vault itu |
| Load gate ingestion | Per box (load box sendiri) | load1 ≥2,0/≥3,0 | Kurangi konkurensi box itu saja |
| Disk gate ingestion | Per box | <40 GB bebas | Tolak klaim baru box itu saja |
| Broadcast window | Master SAJA | 17:00–22:30 WIB | Ingestion Master jeda; VPS 8 jalan terus |
| TX governor live | VPS 8 | TX total >70% port | Tolak stream baru; jeda TX prep |
| Cgroup Stage 2 | VPS 8 | 4 core | Render antre |
| Kuota disk co-tenant | VPS 8 | Sisakan ≥50 GB bebas | Co-tenant berhenti isi |

---

## 6. Phasing + gerbang verifikasi

Setiap langkah: verifikasi pra → implementasi → verifikasi pasca →
observasi 72 jam → lanjut/rollback. **Satu metrik scorecard (§2)
memburuk = gerbang gagal = rollback.**

### Fase 0 — $0 (Master saja; tanpa pembelian)

| # | Langkah | Verifikasi pasca | Rollback |
|---|---|---|---|
| 0.1 | Pisah proses publish vs prep (2 unit systemd) | 2 proses jalan; restart prep tidak ganggu publish (uji eksplisit) | Kembali ke 1 unit |
| 0.2 | Job berat keluar dari web (P15: reassemble, regen, fleet, AI-video → worker) | CPU web <0,2 core; 0 anak ffmpeg di web | Kembalikan route lama (git revert) |
| 0.3 | Klaim pull + 1-owner per (channel, kind) (G9) | Churn SUPERSEDED <20% dalam 7 hari | Kembalikan logika antrean lama |
| 0.4 | Sweeper batch 2 → 5; guard dilokalkan + log alasannya | Feed ±2,5 channel/menit; setiap penahanan tercatat | Kembalikan batch 2 |
| 0.5 | Diagnosis + perbaikan P16 (encoder 706 + orphan) | 0 kematian/72 jam; 0 orphan | — (diagnosis dulu) |
| 0.6 | Janitor status-only (P4/R1) | 0 GB file channel CONNECTED terhapus berlabel "unpaired" dalam audit 24 jam | Kembalikan klausa lama |

**Gerbang Fase 0:** scorecard baris 1, 2, 5, 6, 7, 8, 9, 11 membaik
atau tetap selama 72 jam; 0 error class baru.

### Fase 1 — VPS 8 ($18,65/bln + $2,66 setup)

**Pemicu pembelian (harus terpenuhi):** (a) Fase 0 lolos gerbang; (b)
live mendekati 12–16 stream ATAU 6 stream key sudah terisi semua +
antrean channel siap-live ≥10; (c) harga Master terverifikasi di panel
Contabo dan total ≤ $34,67/bln.

| # | Langkah | Verifikasi pasca | Rollback |
|---|---|---|---|
| 1.1 | Beli VPS 8 (bulanan); Ubuntu + deps; join Tailscale | SSH + Tailscale OK; port 600 Mbit/s terukur | Batalkan dalam 14 hari (hak withdrawal) |
| 1.2 | Deploy live service; sharding + lease + heartbeat | Live-01 hanya jalankan shard yang ditugaskan | Matikan Live-01; live kembali ke Master |
| 1.3 | Migrasi live canary (1–2 stream → observasi → semua) | Stream stabil 72 jam; TX governor menolak di atas batas; Master responsif | Pindahkan stream kembali ke Master |
| 1.4 | DR replica + watchdog | Restore uji berhasil (DB + app jalan dari salinan) | — (non-kritis) |
| 1.5 | Worker prep canary (ingestion 1 job + Stage 2 1 slot) | Klaim API jalan; 0 gangguan live; guard lokal bekerja | Matikan worker VPS 8; prep kembali ke Master |
| 1.6 | Worker prep penuh (ingestion konkurensi penuh + Stage 2 4 slot) | Ingestion RUNNING penuh saat backlog ada; Stage 2 throughput naik terukur; TX live tidak terganggu | Turunkan ke canary / matikan |

**Gerbang Fase 1:** scorecard SEMUA baris membaik atau tetap selama
72 jam pasca-1.6; Master tidak lagi menjalankan live reguler.

### Fase 2 — Box ke-3 (nanti; persetujuan anggaran terpisah)

**Pemicu:** prep di VPS 8 terbukti menekan live ATAU Master publish
butuh seluruh 4 core ATAU armada live mendekati 60 stream (butuh box
live ke-2 — keputusan terpisah: box ke-3 untuk prep vs untuk live).
**Isi:** prep pindah penuh ke compute box → pembagian 3 bagian menjadi
fisik penuh. **Biaya:** +±$18,65/bln (persetujuan terpisah).

---

## 7. Biaya (mengikat — dari dokumen anggaran 21 Nov)

| Item | Per bulan + pajak | Rp |
|---|---|---|
| Master (Plus 4) | ±$16,02 | ±Rp 293 rb |
| Live-01 (Core VPS 8) | $18,65 | Rp 341 rb |
| **TOTAL** | **$34,67** | **Rp 634 rb** |
| Setup sekali (VPS 8) | $2,66 | Rp 49 rb |

Tambahan bersih vs hari ini: **+$4,25/bln** (Tencent $14,40/bln
berakhir 21 Nov). **Peringatan:** harga Master ±$14,43 pra-pajak adalah
ESTIMASI — verifikasi panel Contabo sebelum pembelian; bila total riil
> $34,67, eskalasi, jangan lanjut diam-diam. Opsi hemat: kontrak
12 bulan VPS 8 → total $31,87/bln (bayar di muka $190,21) — putuskan
setelah model terbukti (bulanan dulu).

**Mengapa bukan sekadar tambah bandwidth:** Contabo tidak menjual add-on
port untuk VPS; naik paket Plus 4 → Plus 8 = +€21,50/bln (lebih mahal,
tanpa isolasi). Detail: Lampiran A rancangan lama.

---

## 8. Risiko + mitigasi

| Risiko | Mitigasi |
|---|---|
| Fase 0 mengganggu publish | Deploy bertahap per langkah + observasi 72 jam + rollback per langkah (§6) |
| Satu Live Box = SPOF 50 stream | Master menampung darurat ±20 stream; box live ke-2 bila rapuh (persetujuan terpisah) |
| Sharding salah → 2 supervisor berebut 1 channel | Single-owner + lease + fencing; canary dulu |
| Co-tenant (prep) mengganggu live di VPS 8 | Cgroup + TX throttle + aturan "matikan co-tenant dulu"; canary 1 slot dulu |
| Harga Master riil ≠ estimasi | Verifikasi panel sebelum beli; eskalasi bila > cap |
| Broadcast auto-complete tak terdeteksi | Health check broadcast (bedakan FFmpeg hidup vs broadcast live) |
| Tailscale putus → VPS 8 kehilangan kendali | Stream jalan tetap jalan; penugasan baru tertunda; alarm heartbeat >90 dtk |
| Migrasi Master interim (±17 Des) | Rancangan ini mengikuti Master ke mana pun pindah; keputusan pindah terpisah |
| D1 belum selesai → audio tetap lambat | Plan ini TIDAK mengklaim memperbaiki audio; D1 (eskalasi Dahono) jalur terpisah |

## 9. Yang TIDAK diselesaikan plan ini

- **Bot-check audio (D1):** butuh eskalasi Dahono (Order ID
  `cmwcy3vvx0bkes4zqc6tydnxr`) + proxy residential. Plan ini memberi
  ingestion jalur yang tidak dicekik — tapi unduhan audio tetap gagal
  sampai D1 selesai.
- **Kuota sync P2:** mekanisme belum terpecahkan (6 teori dieliminasi).
  Plan ini mengurangi beban tulis DB, bukan mengobati kuota.
- **Kuota KEIN:** Tahap 1/2 tetap API-bound; plan KEIN berakhir 23 Nov.
- **Encoder 706 (P16):** diagnosis akar penyebab terpisah (langkah 0.5).

## 10. Keputusan yang dibutuhkan dari user

1. **Setuju plan ini sebagai plan of record?** (Menggantikan rancangan
   2 VPS sebagai dokumen acuan.)
2. **Mulai Fase 0 sekarang?** (Semua $0; 6 langkah §6.)
3. **Pemicu pembelian VPS 8:** setuju ambang 12–16 stream (atau nilai
   lain)?
4. **Nilai operasional:** sweeper batch (usul 5), ambang TX throttle
   prep (usul jeda saat TX live >50%), failover live otomatis vs
   manual.
5. **Kepemilikan janitor:** ikut unit publish (usul) atau layanan
   sendiri?
6. **Limit live final:** global + per operator (usul 60/20; kini 16).
7. **Master interim:** tetap Contabo melewati 17 Des atau pindah
   provider (keputusan terpisah)?

## Lampiran — Jejak bukti (dokumen sumber)

| Dokumen | Isi |
|---|---|
| `LAPORAN_BOTTLENECK_UNPAIRED_RENDER_2026-11-22.md` | Diagnosis unpaired + janitor bug + doktrin ingestion + R4 (superseded) |
| `ANALISA_ARSITEKTUR_BOTTLENECK_2026-11-22.md` | Peta proses + B1–B7 awal |
| `VERIFIKASI_BOTTLENECK_2026-11-22.md` | Verifikasi B1–B7 (B2: 6 teori dieliminasi) |
| `EVALUASI_SEBELUM_PERBAIKAN_2026-11-22.md` | Register P1–P16 + matriks + keputusan |
| `RANCANGAN_INFRASTRUKTUR_2_VPS_2026-11-22.md` | Analisis opsi 1/2/3 + Lampiran A (bandwidth) + B (idle + koreksi ingestion) |
| `VERIFIKASI_CPU_INGESTION_LOOPGEN_LIVE_2026-11-22.md` | Peta CPU + web-compute + insiden live |
| `RENCANA_ORCHESTRATOR_2026-11-22.md` | Doktrin tanpa sandera + audit modal + G1–G12 + 8 aturan main |
| `live-streaming/KONTEKS_MASALAH_TUJUAN_SOLUSI_BUDGET_2026-11-21.md` | Anggaran mengikat $34,67/bln + spesifikasi VPS 8 |
