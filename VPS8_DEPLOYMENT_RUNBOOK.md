# RUNBOOK DEPLOYMENT VPS 8 — MASJAVAS TUBE OS V2

**Status:** DRAFT — disiapkan, BELUM dieksekusi. Dokumen ini adalah
rencana eksekusi; menjalankannya butuh persetujuan owner per fase
(Level C, Master Autonomous Engineering Directive).
**Versi:** 1.0 — 22 Nov 2026, 12:35 WIB.
**Ruang lingkup:** Fase 0 ($0, Master saja) → Fase 1 (VPS 8) → Fase 2
(box ke-3, pemicu saja). Dokumen pendamping: `VPS8_ACCEPTANCE_TEST_PLAN.md`
(kriteria lolos tiap langkah) dan `VPS8_ROLLBACK_RUNBOOK.md` (prosedur
mundur tiap langkah).

## Legenda label klaim

- **[VF]** VERIFIED FACT — terverifikasi langsung (log/DB/pengukuran).
- **[DF]** DOCUMENTED FACT NOT YET REVALIDATED — tercatat di dokumen
  sumber, belum divalidasi ulang sejak ditulis.
- **[AS]** ASSUMPTION — asumsi yang dipakai runbook ini; harus
  dikonfirmasi saat eksekusi.
- **[PR]** PROPOSAL — usulan dalam runbook ini, belum diputuskan.
- **[OQ]** OPEN QUESTION — pertanyaan terbuka, belum ada jawaban.
- **[OD]** OWNER DECISION REQUIRED — butuh keputusan owner.

## Sumber

- `PLAN_PENAMBAHAN_VPS8_2026-11-22.md` §2–§10 (plan of record; §6 =
  phasing + gerbang) — termasuk koreksi reassemble 22 Nov 12:10.
- `ADDENDUM_REASSEMBLE_CPU_2026-11-22.md` (modal CPU reassemble).
- `migrasi/RENCANA_MIGRASI_SERVER_BARU_CONTABO.md` +
  `migrasi/MIGRASI_PROGRESS.md` — playbook migrasi 17 Nov yang
  TERBUKTI (Tencent → Contabo, cutover sukses).
- `MASTER_ARCHITECTURE_MASJAVAS_TUBE_OS_V2.md` §12 (deployment policy)
  — DOKUMEN BEKU, hanya dirujuk, tidak diubah.
- `IMPLEMENTATION_CHECKLIST_MASJAVAS_TUBE_OS_V2.md` Track P7 (VPS 8
  provisioning + canary) dan P9 (deployment compatibility).
- `ENGINEERING_READINESS_DASHBOARD.md` — status kesiapan (saat ini
  NOT READY; Batch 1: 3/10).

## Aturan yang mengikat setiap langkah

1. **Satu metrik scorecard memburuk = gerbang gagal = rollback.**
   [VF — PLAN §2/§6; aturan perubahan 22 Nov 01:48]
2. **Urutan tiap langkah:** verifikasi pra → implementasi → verifikasi
   pasca → observasi 72 jam → lanjut/rollback. [VF — PLAN §6]
3. **Deployment policy V2.0** (berlaku untuk setiap rilis):
   build + tes → migration backward-compatible → deploy Master dulu
   bila API baru kompatibel dengan worker lama → satu worker canary →
   smoke test → perluas rollout → hentikan pemberian job ke worker
   lama → **drain job aktif sebelum restart** → rollback bila
   acceptance gate gagal. [DF — V2.0 §12]
4. **Expand-and-contract migration:** jangan hapus kolom lama sebelum
   seluruh worker dan release memakai skema baru. [DF — V2.0 §12]
5. **Release manifest bersama:** Master dan VPS 8 memakai manifest
   rilis yang sama; worker melaporkan versinya saat register/heartbeat.
   [DF — V2.0 §12/Track P7.1]
6. **Tidak ada migration skema sebelum V2-007 disetujui.**
   [VF — gerbang teknis Batch 1; dashboard NOT READY]
7. **Restart:** maks 1 restart/15 menit lintas agent; validasi hasil
   perhitungan jeda (angka sebesar epoch = pembacaan timestamp gagal,
   jangan lewati sleep). [VF — AGENTS.md, insiden 19 Nov]
8. **Sesi SSH panjang putus ~40–60 dtk:** sleep di exec lokal, lalu
   SSH pendek untuk polling; restart lewat `nohup … &` + polling.
   [VF — AGENTS.md, 20 Nov]
9. **`.env` mengandung secret:** jangan pernah dibaca utuh; salin via
   transfer terenkripsi, `chmod 600`, verifikasi sha256.
   [VF — playbook migrasi 17 Nov]
10. **DB tidak pernah dibuka lewat jaringan; SQLite hanya di Master.**
    [VF — PLAN §3 prinsip 4; V2.0]

---

## 0. Gerbang pembelian VPS 8 (harus lolos SEMUA sebelum 1.1)

| # | Pemicu | Status saat runbook ditulis |
|---|---|---|
| (a) | Fase 0 lolos gerbang (scorecard membaik/tetap 72 jam) | BELUM — Fase 0 belum dieksekusi [VF] |
| (b) | Live mendekati 12–16 stream, ATAU 6 stream key terisi semua + antrean siap-live ≥10 | BELUM — 2 stream reguler aktif (114, 706) [VF]; limit live 16 [VF] |
| (c) | Harga Master terverifikasi di panel Contabo dan total ≤ $34,67/bln | BELUM — harga Master ±$16,02 adalah ESTIMASI [DF]; panel Contabo terhalang Cloudflare Turnstile untuk browser cloud — verifikasi butuh user [VF] |

**Biaya mengikat:** Master ±$16,02 + VPS 8 $18,65/bln + setup $2,66
sekali; total $34,67/bln; tambahan bersih +$4,25/bln vs hari ini
(Tencent $14,40/bln berakhir 21 Nov). [DF — PLAN §7; harga belum
terverifikasi panel = V2-009] Bila total riil > $34,67: eskalasi,
jangan lanjut diam-diam. [DF — PLAN §7]
**Opsi hemat (keputusan terpisah):** kontrak 12 bulan VPS 8 →
total $31,87/bln (bayar di muka $190,21) — putuskan setelah model
terbukti; bulanan dulu. [DF — PLAN §7]
**[OD]** Persetujuan pembelian + konfirmasi pemicu (b) oleh owner.

---

## 1. Fase 0 — $0 (Master saja; tanpa pembelian)

**Tujuan:** keluarkan compute tersembunyi dari web, pisah proses,
perbaiki antrean — tanpa membeli apa pun. [DF — PLAN §6]

### 0.1 Pisah proses publish vs prep (2 unit systemd)

- **Keadaan kini:** satu unit worker (`masjavas-os-worker`) menjalankan
  publish + prep bersama. [VF — migrasi 17 Nov]
- **Target:** `masjavas-os-worker-publish` (PROSES 1: scheduler, unduh
  bahan, render ffmpeg 2+1, upload YouTube) dan
  `masjavas-os-worker-prep` (PROSES 2: ingestion + LoopGen).
  [PR — nama unit dari PLAN §5.1; belum ada di produksi]
- **Verifikasi pra:** catat baseline scorecard (publish/hari,
  keterlambatan p90, load Master, healthz p95); backup DB via
  `VACUUM INTO`; git HEAD tercatat.
- **Verifikasi pasca:** 2 proses jalan; **uji eksplisit:** restart prep
  tidak mengganggu publish (1 job publish selesai normal selama
  restart prep). [DF — PLAN §6]
- **Rollback:** kembali ke 1 unit. → `VPS8_ROLLBACK_RUNBOOK.md` §2.1.

### 0.2 Job berat keluar dari web (P15)

- **Keadaan kini:** unit web rata-rata 1,02 core — bukan HTTP,
  melainkan job ffmpeg in-process: reassemble audio (64×/24 jam),
  thumb regen, fleet run, AI video pipeline. [VF — PLAN §1.5]
- **Target:** reassemble, regen, fleet, AI-video pindah ke worker;
  web menjadi HTTP murni <0,2 core. [DF — PLAN §6]
- **Catatan reassemble:** job reassemble (±0,9 core/job, ±6 mnt/job,
  64×/24 jam) adalah satu wadah dengan ingestion; pemindahannya ke
  worker adalah prasyarat Fase 1.5. [VF — ADDENDUM_REASSEMBLE]
- **Verifikasi pasca:** CPU web <0,2 core (rata-rata 24 jam); 0 anak
  ffmpeg di bawah proses web. [DF — PLAN §6]
- **Rollback:** kembalikan route lama (git revert). → Rollback §2.2.

### 0.3 Klaim pull + 1-owner per (channel, kind) (G9)

- **Keadaan kini:** churn SUPERSEDED 66% (333 vs 123 COMPLETED/7 hari).
  [VF — PLAN §1.3; V2-003 mengonfirmasi churn 67% masih terjadi]
- **Target:** churn <20% dalam 7 hari. [DF — PLAN §6]
- **Catatan:** desain dedup menunggu V2-010; tidak ada migration
  sebelum V2-007 disetujui. Bila 0.3 butuh perubahan skema, langkah
  ini DITUNDA sampai gerbang V2-007 lolos. [VF — aturan Batch 1]
- **Rollback:** kembalikan logika antrean lama. → Rollback §2.3.

### 0.4 Sweeper batch 2 → 5; guard dilokalkan + log alasannya

- **Keadaan kini:** feed sweeper batch 2/120 dtk (±1 channel/menit).
  [VF — PLAN §1.3]
- **Target:** feed ±2,5 channel/menit; setiap penahanan guard
  tercatat alasannya di log. [DF — PLAN §6]
- **Catatan:** nilai batch 5 adalah USUL — [OD] PLAN §10 butuh
  keputusan owner (sweeper batch + ambang TX throttle + failover
  live otomatis vs manual).
- **Verifikasi pasca:** ukur laju feed 24 jam; audit log penahanan
  (100% beralasan).
- **Rollback:** kembalikan batch 2. → Rollback §2.4.

### 0.5 Diagnosis + perbaikan P16 (encoder + orphan)

- **Status saat runbook ditulis — SEBAGIAN BESAR SELESAI:**
  diagnosis V2-002 membuktikan kematian encoder adalah slot **114**
  (12×/17 jam, bukan 706 seperti tertulis di PLAN §2 baris 3 dan
  EVALUASI P16 — pemetaan lama itu SALAH), dan 706 adalah orphan
  sehat tanpa feeder. [VF — V2-002, 22 Nov]
- Perbaikan R3–R8 deployed 22 Nov 07:24 (commit `56f7a6d`) + iterasi 2
  deployed ~08:20 (commit `021a9f0`: watchdog 10 dtk, queue 4096,
  rw_timeout 30 dtk; tes 20/20). [VF]
- Audio 706 dipulihkan via restart manual 08:10:28 (downtime ±48 dtk).
  [VF — D11]
- **Sisa langkah ini:** observasi 24 jam pasca-iterasi-2 masih berjalan;
  kematian #14 terjadi 08:09:50 (45 mnt pasca-deploy pertama) — target
  "0 kematian/72 jam" BELUM tercapai. [VF]
- **Verifikasi pasca (untuk menutup 0.5):** 0 kematian tak
  direncanakan/72 jam; 0 orphan; watchdog menyelamatkan (bukan
  sekadar mencatat) stall berikutnya. [PR — kriteria dari
  PLAN §6 + temuan iterasi 2]
- **Rollback:** — (diagnosis dulu). [DF — PLAN §6]

### 0.6 Janitor status-only (P4/R1)

- **Keadaan kini:** janitor mislabel 203 channel (risiko R6);
  audit V2-004 (tiket lengkap) belum dieksekusi. [VF — dashboard]
- **Target:** 0 GB file channel CONNECTED terhapus berlabel
  "unpaired" dalam audit 24 jam. [DF — PLAN §6]
- **Urutan yang benar:** eksekusi V2-004 (audit janitor) DAHULU,
  lalu implementasi status-only berdasarkan temuannya.
  [PR — V2-004 belum jalan saat runbook ditulis]
- **Rollback:** kembalikan klausa lama. → Rollback §2.6.

### Gerbang Fase 0

Scorecard baris 1, 2, 5, 6, 7, 8, 9, 11 membaik atau tetap selama
72 jam; 0 error class baru. [DF — PLAN §6] Bila gagal: rollback
langkah yang bersangkutan, diagnosis, ulangi. Lolos gerbang →
lanjut ke §0 (gerbang pembelian) → Fase 1.

---

## 2. Fase 1.1 — Beli + provisioning VPS 8

**Spesifikasi target:** Contabo Core VPS 8 — 8 vCPU / 24 GB RAM /
300 GB disk / port 600 Mbit/s. [DF — PLAN §4/§5.2; spes dari
listing, belum verifikasi panel]
**Pelajaran migrasi 17 Nov yang dipakai di sini:** provisioning
Contabo sempat TERTAHAN status 'Verification Needed' (verifikasi
manual via email) — alokasikan waktu tunggu; panel Contabo
terhalang Cloudflare Turnstile untuk browser cloud. [VF]

| # | Langkah | Verifikasi pasca |
|---|---|---|
| 1.1.1 | Beli VPS 8 bulanan (bukan kontrak 12 bln dulu); catat Order ID + IP publik + akses root di vault | Pembayaran terkonfirmasi; IP + akses root tercatat [DF — pola migrasi §3] |
| 1.1.2 | Ubuntu 24.04 (sama dengan Master — migrasi mulus); timezone `Asia/Jakarta` (scheduler memakai tanggal WIB) | `lsb_release` + `timedatectl` cocok; [VF — migrasi Fase 1: timezone Contabo awalnya Europe/Berlin dan HARUS diubah] |
| 1.1.3 | Hardening: SSH key-only (`PasswordAuthentication no`, `PermitRootLogin prohibit-password`), UFW minimal (22, 80, 443 + tailscale0), service jalan sebagai user non-root bila memungkinkan | SSH segar pasca-hardening OK; UFW active + enabled; [VF — migrasi Fase 1] |
| 1.1.4 | Install: Python 3.12, pip, venv, ffmpeg, Caddy, `cloudflared`, Tailscale, WARP, Node ≥22, git | Setiap paket terverifikasi versinya — **JANGAN percaya rc=0 dari perintah rantai** (pelajaran 17 Nov: cloudflared gagal unduh tapi rc=0 palsu dari `ln -sf`; verifikasi dengan `<tool> --version` langsung) [VF] |
| 1.1.5 | Join Tailscale + approve node baru (user); uji SSH dari mesin Muse via alias baru | Node ONLINE di tailnet; SSH via Tailscale OK [VF — migrasi: approval user 15:55, node ONLINE] |
| 1.1.6 | Ukur bandwidth port (uji paralel up/down) + CPU/disk benchmark | Port terukur; catat angka (baseline VPS 8). Ambang lolos: lihat Acceptance Test Plan T1.1 [PR] |
| 1.1.7 | Snapshot Contabo #1 (server bersih siap deploy) | [OQ] Snapshot panel 17 Nov DILEWATI (panel terhalang Cloudflare, tanpa API) — coba lagi; bila gagal, catat sebagai risiko (rollback mengandalkan rebuild, bukan snapshot) [VF] |

- **Rollback 1.1:** batalkan dalam 14 hari (hak withdrawal).
  [DF — PLAN §6] → Rollback §3.1.

---

## 3. Fase 1.2 — Deploy live service + release manifest

**Perbedaan penting vs migrasi 17 Nov:** VPS 8 BUKAN klon penuh
Master. DB tetap di Master; VPS 8 menjalankan live service +
prep worker + DR replica. [VF — PLAN §5.2]

| # | Langkah | Verifikasi pasca |
|---|---|---|
| 1.2.1 | Clone repo privat `masjavas7/MASJAVASTUBE-OS` dengan **deploy key SSH** — JANGAN pakai PAT di URL remote | `git ls-remote` OK; HEAD sama dengan Master; [VF — migrasi Fase 2] |
| 1.2.2 | Buat venv dari requirements lock; **JANGAN** jalankan skrip pembuat `.env` baru | Import smoke test OK; pelajaran 17 Nov: `torch==2.15.0+cpu` hanya ada di PyTorch CPU index (`--extra-index-url https://download.pytorch.org/whl/cpu`) [VF] |
| 1.2.3 | Salin `.env` dari Master via transfer terenkripsi, `chmod 600`, verifikasi sha256 cocok | sha256 COCOK; kunci enkripsi ada [VF — migrasi Fase 2] |
| 1.2.4 | Pasang unit systemd: `masjavas-live` (+ governor lokal) — **JANGAN start live dulu**; pasang Caddy bila perlu | `systemctl cat` unit terpasang; daemon-reload OK; unit BELUM aktif [VF — pola migrasi: unit terpasang, start manual saat uji] |
| 1.2.5 | Deploy release manifest bersama (versi yang sama dengan Master) + daftarkan worker (versi dilaporkan saat register/heartbeat) | Manifest cocok dengan Master; registrasi worker tercatat [DF — V2.0 §12; Track P7.1] |
| 1.2.6 | Konfigurasi log rotation + guard lokal (admission TX 70% port = 420 Mbps; cgroup Stage 2 4 core) | Konfigurasi terbaca dari `.env`/settings; guard tick tercatat di journal [PR — nilai dari PLAN §5.2/§5.4] |

- **[OQ]** Kredensial persis apa yang dibutuhkan VPS 8 (service
  account? kunci API? token Drive/YouTube?) — DB dan token OAuth
  channel tetap di Master; daftar kebutuhan secret per layanan
  VPS 8 belum ditulis. Harus dijawab sebelum 1.2.3.
- **Rollback 1.2:** matikan Live-01; live tetap di Master (belum ada
  stream yang pindah). → Rollback §3.2.

---

## 4. Fase 1.3 — Migrasi live canary

| # | Langkah | Verifikasi pasca |
|---|---|---|
| 1.3.1 | Pilih 1 stream canary (usul: stream dengan bitrate tipikal, bukan 114 yang masih dalam observasi pasca-perbaikan) [PR] | Stream dipilih + dicatat alasannya |
| 1.3.2 | Tugaskan shard ke Live-01 (single-owner + lease + fencing token); drain stream itu di Master (tunggu job aktif selesai, jangan bunuh) | Hanya Live-01 yang menjalankan shard itu; 0 duplikat supervisor [DF — PLAN §8; V2.0] |
| 1.3.3 | Start stream di Live-01; verifikasi RTMP langsung ke YouTube (tidak lewat Master/tunnel) + FFmpeg hidup + **broadcast benar-benar live** (bedakan FFmpeg hidup vs broadcast live) | "berhasil mengudara" di journal + verifikasi broadcast dari sisi YouTube; [DF — PLAN §8: health check broadcast] |
| 1.3.4 | Uji reconnect (matikan encoder uji → pulih otomatis) + uji TX governor (tolak stream baru di atas batas) | Reconnect pulih tanpa intervensi; governor menolak di atas 420 Mbps [DF — Track P7.2] |
| 1.3.5 | Observasi 72 jam canary; Master tetap responsif (healthz p95 <100 ms) | Scorecard baris 3, 4, 11, 12 membaik/tetap [DF — PLAN §2/§6] |
| 1.3.6 | Tambah 1–2 stream → observasi → migrasikan sisanya bertahap | Setiap batch lolos observasi sebelum batch berikut [DF — Track P7.2] |

- **Rollback 1.3:** pindahkan stream kembali ke Master (drain di
  Live-01 → start di Master → verifikasi mengudara).
  → Rollback §3.3.
- **Catatan:** Master menampung darurat ±20 stream bila Live-01
  gagal total. [DF — PLAN §8; kapasitas darurat ini belum pernah
  diuji — lihat Acceptance Test Plan T1.3-F]

---

## 5. Fase 1.4 — DR replica + watchdog

| # | Langkah | Verifikasi pasca |
|---|---|---|
| 1.4.1 | DR replica: sync malam (app + DB backup + aset kritis) Master → VPS 8 via rsync | Sync pertama lengkap; ukuran + checksum cocok [DF — PLAN §5.2] |
| 1.4.2 | Watchdog eksternal: monitor Master dari VPS 8 + alarm (heartbeat >90 dtk = stale) | Alarm uji berbunyi pada heartbeat yang sengaja diputus [DF — PLAN §5.3; V2.0 §13] |
| 1.4.3 | **Restore drill #1:** boot aplikasi dari salinan DR (DB + app jalan) di lingkungan terisolasi; ukur RPO/RTO yang TERCAPAI | RPO/RTO tercapai dicatat apa adanya — target ≤5 mnt/≤30 mnt adalah TARGET yang harus dibuktikan drill, BUKAN janji [VF — V2.0 l.179; dashboard NOT READY] |

- **Rollback 1.4:** — (non-kritis; DR tidak memengaruhi produksi
  bila dimatikan). [DF — PLAN §6]
- **[OD]** Provider + retensi storage backup (Litestream/Restic)
  belum dipilih; biaya belum dihitung — gerbang storage/cost
  (V2-009) harus lolos sebelum DR dianggap "selesai".
  [VF — dashboard NOT READY]

---

## 6. Fase 1.5–1.6 — Worker prep canary → penuh

| # | Langkah | Verifikasi pasca |
|---|---|---|
| 1.5.1 | Register Live-01 sebagai prep worker (claim via API Master; lease + heartbeat) | Registrasi + heartbeat tercatat; SQLite tetap hanya di Master [DF — Track P7.3] |
| 1.5.2 | Ingestion canary: 1 job end-to-end (klaim → proses → submit via API) | Job COMPLETED via API; 0 gangguan live; guard lokal bekerja [DF — PLAN §6] |
| 1.5.3 | Stage 2 canary: **2 slot** (bukan 4) — mulai dari 2 slot, naik hanya bila pengukuran membuktikan aman | 2 job Stage 2 DONE; CPU/RAM/disk/live-TX terukur [DF — V2.0 l.227] |
| 1.5.4 | **Uji gabungan (wajib):** Stage 2 2 slot BERJALAN BERSAMAAN dengan reassemble (±1 core) | Puncak CPU prep terukur ≤ anggaran (≈7 core total box); live tidak terganggu; kriteria detail di Acceptance Test Plan T1.5-C [VF — ADDENDUM_REASSEMBLE §5] |
| 1.5.5 | Uji worker crash + uji Master disconnect (cabut koneksi → worker pause, tanpa duplikat job; sambung lagi → lanjut) | 0 duplikat job; hasil diterima via API; fencing bekerja [DF — Track P7.3] |
| 1.5.6 | Observasi 72 jam canary | Scorecard membaik/tetap; 0 error class baru [DF — PLAN §6] |
| 1.6.1 | Naikkan ingestion concurrency bertahap + Stage 2 ke 4 slot (cgroup) | Ingestion RUNNING penuh saat backlog ada; throughput Stage 2 naik terukur; TX live tidak terganggu [DF — PLAN §6] |
| 1.6.2 | TX throttle prep: unggahan Drive dijeda saat TX live >50% | Throttle terbukti bekerja pada TX live tinggi (uji beban) [DF — PLAN §5.2; ambang 50% adalah USUL — OD PLAN §10] |

- **Rollback 1.5/1.6:** matikan worker VPS 8; prep kembali ke
  Master (turunkan ke canary dulu bila memungkinkan).
  → Rollback §3.5/§3.6.
- **Catatan:** ingestion di VPS 8 tetap bergantung pada D1
  (bot-check audio) dan P2 (kuota sync) yang belum selesai —
  runbook ini TIDAK mengklaim memperbaikinya. [VF — PLAN §9]

### Gerbang Fase 1

Scorecard SEMUA baris membaik atau tetap selama 72 jam pasca-1.6;
Master tidak lagi menjalankan live reguler. [DF — PLAN §6]

---

## 7. Fase 2 — Box ke-3 (pemicu saja; di luar runbook ini)

**Pemicu:** prep di VPS 8 terbukti menekan live ATAU Master publish
butuh seluruh 4 core ATAU armada live mendekati 60 stream (butuh box
live ke-2 — keputusan terpisah: box ke-3 untuk prep vs untuk live).
[DF — PLAN §6] **Isi:** prep pindah penuh ke compute box → pembagian
3 bagian menjadi fisik penuh. **Biaya:** +±$18,65/bln — persetujuan
anggaran terpisah. [DF — PLAN §6] Runbook Fase 2 ditulis bila
pemicu terpenuhi; bukan bagian dari dokumen ini.

---

## 8. Pelajaran migrasi 17 Nov yang diterapkan (checklist)

| # | Pelajaran (terbukti 17 Nov) | Diterapkan di |
|---|---|---|
| 1 | Timezone server baru Europe/Berlin — ubah ke Asia/Jakarta dulu | 1.1.2 |
| 2 | rc=0 palsu dari perintah rantai (cloudflared) — verifikasi versi langsung | 1.1.4 |
| 3 | `.env` salin persis + chmod 600 + sha256 cocok; JANGAN jalankan generator `.env` baru | 1.2.3 |
| 4 | DB via `VACUUM INTO` + `integrity_check` + row count cocok (jangan `cp` saat worker menulis) | 1.4.1, drill |
| 5 | Deploy key SSH, bukan PAT di URL remote | 1.2.1 |
| 6 | Worker BARU dilarang start sebelum cutover (hindari dobel publish/stream) | 1.2.4, 1.3.2 |
| 7 | rsync bulk lambat/kolaps saat uplink jenuh upload YouTube — jadwalkan sync besar di jendela sepi; jangan jalankan 2 rsync untuk file yang sama | 1.4.1 |
| 8 | DNS cutover = ganti A record (@ + www) di panel IDCloudHost/DNSCloud; propagasi ±15 mnt; user yang mengubah (Muse tidak bisa) | 1.3 (bila domain terlibat) |
| 9 | Snapshot Contabo dilewati (panel terhalang Cloudflare) — rollback mengandalkan server lama/rebuild | 1.1.7 |
| 10 | Skrip via stdin pipe gagal diam-diam (ssh memakan stdin) — jalankan skrip dari file + `< /dev/null` | semua langkah SSH |
| 11 | venv: torch CPU hanya di PyTorch CPU index | 1.2.2 |
| 12 | Verifikasi formal terpisah dari pekerjaan (run formal + rc dicatat; 1 render Stage 2 uji PASS; dekripsi 1 token OK) | semua gerbang |

---

## 9. Log eksekusi (diisi saat eksekusi)

| Tanggal/WIB | Langkah | Pelaksana | Verifikasi pasca (hasil) | Gerbang (LOLOS/GAGAL) | Catatan/rollback |
|---|---|---|---|---|---|
| | | | | | |

## 10. Pertanyaan terbuka (OQ) runbook ini

1. **[OQ-101]** Kredensial persis apa yang dibutuhkan VPS 8 per layanan?
2. **[OQ-102]** Ambang lolos uji bandwidth port 600 Mbit/s (usul: ≥420 Mbps
   sustained — sejajar cap admission 70%; belum diputuskan)?
3. **[OQ-103]** Stream canary mana yang dipilih untuk 1.3.1?
4. **[OQ-104]** Provider + retensi storage backup (Litestream/Restic)?
5. **[OQ-105]** Failover live otomatis vs manual (PLAN §10.4)?
6. **[OQ-106]** Kapasitas darurat Master ±20 stream — pernahkah diuji?
