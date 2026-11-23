# ENGINEERING READINESS DASHBOARD — MASJAVAS TUBE OS

**Terakhir diperbarui:** 23 Nov 2026, 14:50 WIB (Cycle 12: VPS 8 DIORDER owner 14:41 — D4 DIPUTUSKAN, gerbang pembelian 06:31 SUPERSEDED; R22/R23 diperbarui; R24–R26 baru; D12–D14 baru; kesiapan gerbang ±65% (14:16); penentu 15:00–15:10)
**Sumber:** `AUTONOMOUS_WORK_LOG.md`, `BATCH1_EXECUTION_TICKETS_V2.md`,
dokumen frozen V2.0, bukti audit V2-001.
**Sumber status resmi:** `STATUS_REGISTRY.yaml` — bila ada perbedaan
antara dashboard ini dan registry, registry yang benar.
**Label bukti:** VERIFIED FACT / DOCUMENTED FACT NOT YET REVALIDATED /
ASSUMPTION / PROPOSAL / OPEN QUESTION / OWNER DECISION REQUIRED.

## Overall readiness: NOT READY

**Alasan:** Batch 1: 1 EXECUTED (V2-001 — dengan KOREKSI: auditnya di
host yang salah, lihat R15; bukti tidak berlaku untuk target) + 2
READY_FOR_REVIEW (V2-002 + V2-003) dari 10; **10/10 tiket kini lengkap
di sisi dokumen** (eksekusi desain V2-010 menunggu V2-007 DISETUJUI,
gerbang teknis keras; TANPA migration); **VPS 8 DIBELI owner 23 Nov 14:41
(`vmi3778263`) — gerbang pembelian 06:31 SUPERSEDED oleh keputusan langsung
owner, D4 DIPUTUSKAN**; kesiapan gerbang ±65% (14:16, naik dari ±40%
13:32) — penentu 15:00–15:10 WIB (relink 114/707 + ledger tahap 2);
V2-009 kini verifikasi PASCA-BELI (harga aktual panel + storage belum
terverifikasi); keputusan provider KEIN ditutup atas asumsi owner tanpa
bukti live; risiko split-brain box lama (R14) sudah RESOLVED 07:05 WIB.
Fondasi dokumen (arsitektur + checklist frozen) kuat — yang kurang adalah
bukti eksekusi, verifikasi biaya, dan pembuktian pemulihan 15:00.

## VPS8 purchase-readiness gate (§10)

### Architecture

| Item | Status |
|---|---|
| Master/VPS8 responsibility boundaries documented | READY (V2.0 §4–§6, frozen) |
| Worker communication contract documented | READY WITH ACCEPTED RISKS (V2.0 §7–§8 desain; kontrak API detail = Workstream C, belum ditulis) |
| Database ownership and access model documented | READY WITH ACCEPTED RISKS (SQLite Master-only, V2.0; audit schema V2-007 belum jalan) |
| Job lifecycle and failure recovery documented | READY WITH ACCEPTED RISKS (V2.0 §9–§11; dedup key menunggu V2-007/V2-010) |
| Live continuity during Master outage documented | NOT READY (arah di V2.0; prosedur detail belum ada) |

### Infrastructure

| Item | Status |
|---|---|
| VPS specifications verified | NOT READY (spes Contabo VPS 8 dari listing — belum verifikasi panel) |
| Price verified from provider source | NOT READY (Master ±$16,02 = ESTIMASI; VPS8 $18,65 listing belum verifikasi panel; cap $34,67/bln mengikat — pembelian SUDAH terjadi 14:41 atas keputusan owner, V2-009 kini verifikasi pasca-beli) |
| Storage and backup costs calculated | NOT READY (provider + retensi belum dipilih) |
| Network assumptions documented | READY WITH ACCEPTED RISKS (model TX 70% port di V2.0; pengukuran Master 91–93% port 500 Mbps terverifikasi) |
| Firewall and private connectivity plan ready | NOT READY |
| Resource budget based on benchmarks or marked assumptions | READY WITH ACCEPTED RISKS (modal per worker di orchestrator; benchmark Stage 2 concurrency = V2-008) |

### Operations

| Item | Status |
|---|---|
| Production stabilization risks reviewed | READY WITH ACCEPTED RISKS (P1–P16 di EVALUASI_SEBELUM_PERBAIKAN; V2-002 + V2-003 selesai — P16 terdiagnosis, Guard v2 diaudit; V2-004 belum diaudit) |
| KEIN fallback decision presented to owner | READY WITH ACCEPTED RISKS (V2-001 EXECUTED atas asumsi owner; KOREKSI 07:05: audit V2-001 di host salah — tidak ada bukti live; matriks harga tetap valid) |
| Monitoring and alerting plan ready | NOT READY (arah: heartbeat + watchdog eksternal dulu; rencana detail belum ada) |
| Backup and restore procedure ready | NOT READY (RPO/RTO = target, belum dibuktikan restore drill) |
| Recovery targets documented and testable | NOT READY |

### Implementation

| Item | Status |
|---|---|
| Batch 1 tickets completed to readiness standard | NOT READY (1 EXECUTED dengan keterbatasan bukti + 2 READY_FOR_REVIEW dari 10; V2-004 + V2-005 + V2-006 + V2-007 + V2-008 + V2-009 + V2-010 tiket lengkap, belum dieksekusi; lihat STATUS_REGISTRY.yaml) |
| Schema audit approved before migration work | NOT READY (V2-007 belum jalan; tidak ada migration yang berjalan — aman) |
| Deployment sequence prepared | NOT READY |
| Canary procedure prepared | NOT READY |
| Rollback procedure prepared | NOT READY |
| Acceptance tests defined | NOT READY |
| Owner approvals clearly identified | READY WITH ACCEPTED RISKS (register awal di bawah; belum dokumen mandiri) |

## Workstreams (§6)

| WS | Nama | Status | Catatan |
|---|---|---|---|
| A | Production stabilization | IN_PROGRESS | V2-002 + V2-003 READY_FOR_REVIEW; V2-004 tiket lengkap (Cycle 5), menunggu eksekusi; P1–P16 terdokumentasi |
| B | KEIN fallback readiness | IN_PROGRESS | V2-001 EXECUTED dengan keterbatasan bukti (audit di host salah; Q5/Q6 ditutup atas asumsi owner 06:55 — bukan bukti live) |
| C | Orchestrator & job lifecycle | NOT_STARTED | Desain di V2.0 + orchestrator doc; kontrak API detail belum ditulis |
| D | VPS8 infrastructure readiness | IN_PROGRESS | VPS 8 DIBELI owner 14:41 (`vmi3778263`, Cloud VPS 8, EU, no setup); Fase 1 (setup + Tailscale + deploy live + canary 706 dulu) dijalankan lane Antigravity via prompt 14:52 — di luar scope cron local-only; program memantau, tidak mengeksekusi; runbook + acceptance + rollback sudah ada sebagai dokumen |
| E | Backup, monitoring, recovery | NOT_STARTED | Arah di V2.0 §12–§13; rencana detail belum ada |
| F | Deployment & release mgmt | NOT_STARTED | Manifest rilis di V2.0; rencana detail belum ada |
| G | Security & access control | NOT_STARTED | — |
| H | Documentation & execution readiness | IN_PROGRESS | 14/14 artefak §9 ada (12 selesai + di-push ke repo 22 Nov 12:37); repo MASJAVAS-PLAN-OS-NEW 21 file |

## Batch 1 tracker (§7–§8)

| ID | Tiket | Pri | Status | Bukti |
|---|---|---|---|---|
| V2-001 | KEIN dependency/fallback (audit) | P0 | EXECUTED (Q5/Q6 ditutup atas asumsi owner "KEIN aman", 06:55 WIB) | `workspace/engineering/v2-001/` (6 dokumen) |
| V2-002 | Encoder 706 root-cause | P0 | READY_FOR_REVIEW (audit 22 Nov 06:58–07:10; 12 kematian slot 114, RTMP timeout) | `workspace/engineering/v2-002/` (5 dokumen + manifest) |
| V2-003 | Resource Guard v2 audit | P0 | READY_FOR_REVIEW (audit 22 Nov 07:15–07:40; governor 0 aksi efektif, Q6 solved) | `workspace/engineering/v2-003/` (6 dokumen + manifest) |
| V2-004 | Janitor safety audit | P0 | QUEUED (tiket lengkap, diperluas Cycle 5 22 Nov 10:50 WIB; eksekusi menunggu sesi interaktif + SSH) | — |
| V2-005 | Baseline snapshot | P1 | NOT_STARTED (tiket lengkap, diperluas Cycle 6 22 Nov 14:50 WIB; eksekusi menunggu sesi interaktif + SSH) | — |
| V2-006 | Service inventory | P1 | NOT_STARTED (tiket lengkap, diperluas Cycle 7 22 Nov 18:50 WIB; eksekusi menunggu sesi interaktif + SSH) | — |
| V2-007 | Database schema audit (GERBANG TEKNIS) | P1 | NOT_STARTED (tiket lengkap, diperluas Cycle 8 22 Nov 22:50 WIB; eksekusi menunggu sesi interaktif + SSH) | — |
| V2-008 | Resource budget benchmark | P1 | NOT_STARTED (tiket lengkap, diperluas Cycle 9 23 Nov 02:50 WIB; eksekusi menunggu sesi interaktif + SSH; uji concurrency F.6 butuh materi uji ATAU persetujuan user bila menyentuh antrean produksi) | — |
| V2-009 | VPS 8 price + storage cost verification (GERBANG BIAYA) | P1 | NOT_STARTED (tiket lengkap, diperluas Cycle 10 23 Nov 06:50 WIB; eksekusi menunggu sesi interaktif + angka panel Contabo dari user — TIDAK butuh SSH; JANGAN coba browser task untuk panel, Turnstile terbukti memblokir) | — |
| V2-010 | Deduplication design proposal | P1 | NOT_STARTED (tiket lengkap, diperluas Cycle 11 23 Nov 10:50 WIB; eksekusi DESAIN menunggu V2-007 DISETUJUI — gerbang teknis keras; TANPA migration) | — |

## Risk register (awal — dari §5.5 + temuan V2-001)

| # | Risiko | Severity | Status |
|---|---|---|---|
| R1 | KEIN expiry 23 Nov tanpa fallback teruji | P0 | ACCEPTED BY OWNER (asumsi "KEIN aman", 22 Nov 06:55; Q5/Q6 tidak terverifikasi — risiko diterima user) |
| R2 | Encoder slot 114 mati 12×/17 jam (bukan 706 — P16 salah alamat) + orphan 706 | P0 | FIX DEPLOYED + ITERASI 2 — TIDAK MENAHAN: observasi 24 jam GAGAL 14:47 (B10/R20: 4 stream mati serentak; watchdog fired 14:47:11, encoder 706 tetap mati 14:47:15) |
| R3 | D1 proxy residential 407 (bot-check yt-dlp) | P1 | OPEN (keputusan user ditunda 20 Nov) |
| R4 | P2 kuota Drive — mekanisme belum diketahui | P1 | OPEN (6 teori tereliminasi) |
| R5 | SQLite lock 31×/24 jam | P1 | OPEN (V2-007 + guard audit) |
| R6 | Janitor mislabel 203 channel CONNECTED | P0 | OPEN (V2-004; janitor belum dijalankan mode hapus) — tiket audit lengkap Cycle 5 10:50 WIB (baseline reklasifikasi 40,92 GB/24 jam; OQ-013 dry-run, OQ-014 retensi `deletion_audit`, OQ-015 jalur hapus lain) |
| R7 | `shed_render` log-only 112×/24 jam | P1 | DIAGNOSED (V2-003: 29×/17 jam, tanpa eksekutor by design; governor tidak pernah pilih shed_ingestion karena atribusi buta) |
| R8 | Churn 66% (333 SUPERSEDED vs 123 COMPLETED) | P1 | OPEN |
| R9 | Stage 2 I/O-bound (12,8 dtk CPU vs 100,1 dtk wall) | P2 | OPEN (V2-008) |
| R10 | Harga Master/VPS8 + storage belum terverifikasi | P1 | OPEN (V2-009; butuh user untuk panel Contabo) |
| R11 | Dua host KEIN (`beliapi.com` vs `kein.my.id`) — kuota mungkin terpisah | P1 | RESOLVED (OQ-004: satu akun — pindah domain, modal situs 05:29) |
| R12 | `loopgen_agent_tasks` kosong — rekaman Opsi 3 tidak terlacak di DB ini | P2 | OPEN (OQ-001 → V2-007) |
| R13 | Pemetaan stream→slot bertentangan: stream #1795226249006 dicatat milik slot 706 (P16) dan milik STAGING-SLOT-114 (AUDIT_TAHAP0) — diagnosis bisa salah alamat | P1 | RESOLVED (OQ-007: stream milik STAGING-SLOT-114 — DB; V2-002 Q8) |
| R14 | Split-brain: box lama Tencent masih menjalankan web + menulis DB-nya sendiri (55 MB vs 118 MB live) — dua Master divergen | P0 | RESOLVED (R1 dieksekusi 07:05 WIB: web lama stopped+disabled, port 8080 tutup, DB statis sejak 07:05 — diverifikasi 07:09) |
| R15 | Status KEIN pada sistem LIVE tidak diketahui — audit V2-001 dilakukan di host salah (box lama, stale sejak 17 Nov); asumsi "KEIN aman" tanpa bukti live | P0 | ACCEPTED BY OWNER (user konfirmasi 07:03 WIB: lanjut V2-003, KEIN tetap dilewati — audit ulang R2 DITOLAK; status live tetap unknown atas keputusan user) |
| R16 | Governor tidak pernah mengambil aksi efektif (0/1.918 tick V2; satu-satunya aksi = shed_render no-op) | P1 | OPEN (V2-003-R4/R6) |
| R17 | Konfigurasi split guard V2/admission legacy + flag menyesatkan; alasan flag `false` tidak diketahui (OQ-010) | P1 | OPEN (V2-003-R1/R5) |
| R18 | Atribusi lane ingestion buta (daemon → lane `other`; governor tidak bisa melihat ingestion) | P1 | OPEN (V2-003-R3) |
| R19 | Orphan 706 mengudara TANPA AUDIO ±17 jam (pipe tanpa feeder sejak restart 14:42; bitrate tetap HEALTHY sehingga tak terdeteksi) | P1 | RESOLVED (restart manual 08:10:28 — audio mengalir, feeder 19,4 MB) |
| R20 | Live MATI MASSAL 22 Nov 14:47 (4 stream/21 dtk) + pemulihan tidak andal (reconnect gagal DB-lock; status desync; 706 tanpa reconnect; encoder ganda 707; relay task mati; supervisor "hard exiting") | P0 | OPEN — mekanisme TERBUKTI 15:30 (overwrite stale-`now` daemon vs supervisor; health check STREAMING-only); live PULIH 15:16 (4/4 STREAMING, audio terverifikasi 15:29) tapi TIDAK STABIL (sentinel ghost-kill 114 2×; 3 proses pemulihan berpacu; encoder ganda 114; fix sentinel belum ter-deploy) |
| R21 | Relink publik gagal di UI ("Failed to fetch") + sentinel BUTA (bot-check) membunuh siaran sehat tiap ±4 mnt (RIFFORA tayang 19:00:32–19:04:40 lalu dibunuh sentinel) | P0 | OPEN — diagnosis SELESAI 19:08 (panggilan YouTube sinkron membekukan worker 60–120 dtk; probe kena LOGIN_REQUIRED; relink tanpa pemanggil otonom); perbaikan menunggu keputusan owner |
| R22 | Kuota YouTube API 3 vault live HABIS (±21:25 22 Nov) dibakar lingkaran kill-relink; keempat live GELAP di YouTube sejak ±21:30 meski dashboard STREAMING | P0 | OPEN — REM DI-DEPLOY 03:53 23 Nov (commit 96e5ee4); lingkaran kill BERHENTI (0 kill sejak 12:39, total 757; monitor 14:39). Periodic ensurer ADA (`live_supervisor.py:1912`, 5 call site) → 114/707 seharusnya auto-relink 15:00 TANPA intervensi; cron pemulihan 15:10 menjadi verifikasi saja; ledger tahap 2 belum terverifikasi (penentu 15:00–15:10, R24) |
| R23 | Slot 698 (RUANGIRAMA) DIHAPUS 06:06 WIB 23 Nov (stop manual + DELETE, IP 103.214.112.181) — guardian mencoba membangkitkannya tiap 30 dtk | P1 | RESOLVED (asal + loop) / OPEN (nasib) — ASAL TERKONFIRMASI: MONA yang menghapus, dikonfirmasi user 09:11 ("mona yang hapus"); guardian loop MATI sejak restart live 08:48 (KONTEKS_MASALAH_TERBARU §3.4). SISA: task baru belum dibuat — menunggu keputusan MONA (D12). CATATAN POSITIF: reaper encoder ganda commit 33f1c15 TERVERIFIKASI bekerja (sensus 06:48: tepat 3 encoder, 0 duplikat) — duplikat encoder bukan lagi masalah aktif |
| R24 | Pemulihan 15:00 untuk 114/707 BELUM TERBUKTI — encoder STREAMING tapi broadcast belum ada (114 `broadcast_id` KOSONG; 707 broadcast `YFmbo1wSusM` oEmbed 404); rantai encoder→broadcast→live belum pernah dibuktikan untuk keduanya | P0 | OPEN — penentu gerbang (kesiapan ±65% → ±75–80% bila berhasil, → ±50% bila gagal); label "Mengudara (RTMP Ingest)" menyesatkan; periodic ensurer diharapkan auto-relink 15:00; verifikasi 15:00–15:10 |
| R25 | Perbaikan schema lock `cd4fb7a` BELUM BERJALAN — commit 14:02:53, tapi restart live 13:50:05 + web 14:00:57 keduanya SEBELUM commit; kode ada di disk tapi belum aktif di service mana pun; klaim "0 DB lock" TIDAK PRESIS (5 lock 13:54:59–14:00:00) | P1 | OPEN — butuh restart service terencana + persetujuan owner (D13, tenggat 24 jam); observasi 2 jam pasca-restart untuk membuktikan 0 FAILED semaphore baru |
| R26 | CPU Master JENUH — bottleneck mengikat (load 6,60/6,11/6,42 pada 4 vCPU); ffmpeg live ~165% (2 encoder) + audio_reassembler + yt-dlp ~53%; ketidakstabilan encoder BEROTASI antar task (706→707→114) → starvation sumber daya bersama; RAM BUKAN bottleneck (3,9 GB tersedia) | P0 | OPEN — menjelaskan kematian encoder berulang (R2/B10) + WEB DOWN 13:48–13:55 (web hidup tapi tak merespons ±7 mnt; restart 13:50 tidak memulihkan; live 0 stream); VPS 8 Fase 1 (offload live + prep) berjalan via Antigravity; mekanisme B10 tetap belum terpecahkan |

## Owner decision register (awal)

| # | Keputusan | Status | Bukti / tenggat |
|---|---|---|---|
| D1 | Provider KEIN: top-up / fallback / HOLD | DITUTUP — user: "ASUMSIKAN KEIN AMAN" (22 Nov 06:55); top-up/fallback tidak diputuskan, dianggap tidak perlu. KOREKSI 07:05: bukti V2-001 dari host salah — asumsi tanpa bukti live (R15). Konfirmasi user 07:03 WIB: KEIN tetap dilewati, audit ulang ditolak | Expiry 23 Nov 2026 |
| D7 | Hentikan layanan aplikasi di box lama Tencent (V2-002 R1) | DISETUJUI + DIEKSEKUSI 22 Nov 07:05 WIB (web stopped+disabled; DB statis terverifikasi) | — |
| D8 | Perbaikan encoder 114 (V2-002 R3–R8: relay audio, admission TX, cleanup relay, exit code, alert kronis, adopsi orphan) | DISETUJUI 07:05 WIB; DEPLOYED + TERVERIFIKASI 07:24 WIB (commit 56f7a6d; observasi 24 jam) | Laporan: `v2-002/ENCODER114_FIX_REPORT.md` |
| D11 | Restart manual stream 706 untuk memulihkan audio (orphan tanpa feeder — R19) | DISETUJUI + DIEKSEKUSI 08:10 WIB (downtime ±48 dtk; audio terverifikasi mengalir) | — |
| D9 | Perbaikan Guard v2 (V2-003-R1–R9; prioritaskan V2-003-R1/R2 admission TX) | MENUNGGU | `v2-003/GUARD_RECOMMENDATIONS_V2-003.md` |
| D10 | OQ-010 — siapa men-set flag `LIVE_GUARD_V2_ENABLED=false` + CPU 90 di `.env`, dan mengapa (prasyarat V2-003-R1) | MENUNGGU | `.env` diubah 21 Nov 17:58 |
| D2 | Pilot DeepInfra 3 varian (Rp 800) | MENUNGGU (ditawarkan 18 Nov) | — |
| D3 | D1 proxy residential (WaroengProxy Rp 10 rb/GB, uji 1 GB) | DITUNDA user 20 Nov — jangan tanya ulang | — |
| D4 | Pembelian VPS8 ($34,67/bln cap) | DIPUTUSKAN + DIEKSEKUSI 23 Nov 14:41 WIB (owner membeli langsung; `vmi3778263`, Cloud VPS 8 (2026), EU, no setup; gerbang pembelian 06:31 SUPERSEDED) | Prompt Fase 1: `PROMPT_ANTIGRAVITY_IMPLEMENTASI_VPS8_2026-11-23.md` (14:52) |
| D12 | Nasib slot 698 — dikembalikan (buat task live baru) atau tidak? | MENUNGGU (keputusan MONA) | MONA menghapus 698 (dikonfirmasi user 09:11); guardian loop mati 08:48; channel vault-ready |
| D13 | Jadwal restart service untuk mengaktifkan `cd4fb7a` (perbaikan schema lock) | MENUNGGU | Perbaikan ada di disk, belum berjalan; tenggat 24 jam (RENCANA §6 D3) |
| D14 | Fase 2 Antigravity (bitrate conditioner + `assigned_node_id`) | MENUNGGU | Antigravity mengklaim "siap Fase 2"; persetujuan terpisah; setelah Tahap A (RENCANA §6 D4) |
| D5 | Sweeper batch, TX-throttle threshold, failover live auto/manual, kepemilikan janitor, limit live final, provider Master pasca-Des | MENUNGGU (plan §10) | — |
| D6 | Aktivasi distribusi Stage 2 live (guard 422) + peluncuran batch massal | MENUNGGU (peluncuran = user) | — |

## Cost register (awal — VERIFIED vs ESTIMATED)

| Item | Nilai | Status | Sumber |
|---|---|---|---|
| Master Contabo Plus 4 | ±$16,02/bln (pre-tax ±$14,43) | ESTIMATED (belum verifikasi panel) | Plan VPS8 |
| VPS 8 Contabo Core | $18,65/bln + $2,66 setup | DOCUMENTED NOT YET REVALIDATED | Plan VPS8 (listing) |
| Total cap | $34,67/bln | OWNER DECISION (mengikat) | User 21–22 Nov |
| TCO vs cap (sprint 13:08) | Minimum PAS di cap ($0 headroom); expected +$1,50/bln; worst case +$4,97/bln — SEBELUM storage/domain | DIPUTUSKAN 22 Nov 13:13: cap = seluruh TCO → infrastruktur saja sudah $34,67; storage/backup harus $0 atau infrastruktur direvisi |
| DeepInfra FLUX | Rp 35,44/gambar | HISTORICAL (18 Nov) | Riset 18 Nov |
| Muse Image | Rp 180/gambar | HISTORICAL (18 Nov) | Riset 18 Nov |
| Kie.ai Seedream 5.5 | ±Rp 7,8 jt/700 ch | HISTORICAL (18 Nov) | Riset 18 Nov |
| ModelsLab Unlimited | $149/bln ±Rp 2,7 jt | HISTORICAL (18 Nov) | Riset 18 Nov |
| Storage off-site (Litestream/Restic) | Belum dihitung | OPEN | V2-009 |

## §18 baseline — yang sudah ada (22 Nov 05:18 WIB)

- `workspace/engineering/`: 88 entri. Dokumen kunci: V2.0 frozen (1.462
  baris), checklist frozen (1.076 baris), tiket Batch 1 (434 baris),
  RENCANA_ORCHESTRATOR (G1–G12), PLAN_PENAMBAHAN_VPS8, RANCANGAN_2_VPS,
  EVALUASI_SEBELUM_PERBAIKAN (P1–P16), rantai bukti bottleneck (4 dokumen),
  50 repo GitHub terverifikasi, audit ingestion/disk/thumbnail, dll.
- 14 artefak §9: 14/14 selesai 22 Nov 12:37 (12 draft paralel + review + push ke repo MASJAVAS-PLAN-OS-NEW; 21/21 file OK via konektor vault custom.github).
- Ekuivalen parsial (periksa sebelum membuat duplikat): risk register ←
  EVALUASI_SEBELUM_PERBAIKAN (P1–P16); cost ← PLAN_PENAMBAHAN_VPS8 §7 +
  docs/BIAYA_*; VPS8 runbook ← PLAN_PENAMBAHAN_VPS8 §6 + migrasi/;
  decision register ← PLAN_PENAMBAHAN_VPS8 §10.

## Next 5 actions

1. **Verifikasi 15:00–15:10 WIB (PENENTU, di luar scope cron local-only):**
   114/707 live di YouTube via periodic ensurer + ledger tahap 2 mencatat
   INSERT/BIND/TRANSITION (R24). Bila berhasil → kesiapan ±75–80%; bila
   relink gagal → kembali ±50% + diagnosis sebelum canary 114/707.
   Cron 15:10 kini verifikasi saja (bukan pemulihan).
2. Keputusan user D9/D10 (perbaikan Guard v2 + OQ-010 prasyarat) — menunggu.
3. Keputusan user D12 (nasib 698) + D13 (jadwal restart `cd4fb7a`) + D14
   (Fase 2 Antigravity) — menunggu.
4. Fase 1 VPS 8 (lane Antigravity, prompt 14:52 — di luar scope cron):
   setup + Tailscale → deploy live → canary 706 dulu → 114/707 SETELAH
   15:00 → DR replica + watchdog → worker prep canary → worker prep penuh.
   Program memantau via log + monitor, tidak mengeksekusi. JANGAN sentuh
   Master 15:00–15:15.
5. Eksekusi audit V2-004 (janitor) saat sesi interaktif tersedia — tiket
   lengkap; tangkap `deletion_audit` SEGERA (retensi belum diketahui, OQ-014).
6. Eksekusi audit V2-005 (baseline snapshot) saat sesi interaktif tersedia —
   tiket lengkap; snapshot adalah lantai gerbang Fase 0 (rollback
   satu-metrik butuh angka acuan yang sah); baca schema dulu sebelum query
   (OQ-016/017/018).
7. Eksekusi audit V2-006 (service inventory) saat sesi interaktif tersedia —
   tiket lengkap; nama unit systemd dari V2-005 F.3 wajib diverifikasi dulu
   (OQ-019); hasil V2-006 adalah input V2-007 (penulis DB, OQ-022) + V2-008
   (worker untuk benchmark) + P3.
8. Eksekusi audit V2-007 (schema audit — GERBANG TEKNIS) saat sesi interaktif
   tersedia — tiket lengkap; backup konsisten DULU lalu audit dari SALINAN
   (OQ-024); gerbang menahan V2-010 sampai disetujui.
9. Eksekusi benchmark V2-008 saat sesi interaktif tersedia — tiket lengkap;
   ukur konsumsi aktual per worker + uji concurrency Stage 2 (1 vs 2 slot) —
   F.6 HANYA dengan materi uji ATAU persetujuan user bila menyentuh antrean
   produksi; angka concurrency final menggantikan PROPOSAL capacity model.
   R26 (CPU Master jenuh, load 6,6/4 vCPU) memperkuat urgensi benchmark ini
   sebagai pembanding pasca-canary.
10. Eksekusi verifikasi V2-009 saat sesi interaktif tersedia — kini
    verifikasi PASCA-BELI: minta angka panel Contabo dari user (daftar
    periksa F.1, sekali, dengan tanggal — harga aktual VPS 8 yang dibayar +
    harga Master + renewal); F.2–F.5 (quote storage + volume backup +
    hitung ulang TCO) dapat dikerjakan dulu; verdict C-1–C-5 kini menilai
    apakah pembelian yang sudah terjadi muat cap $34,67/bln dan apa yang
    harus direvisi bila tidak.
11. Eksekusi desain V2-010 saat sesi interaktif tersedia — HANYA setelah
    V2-007 DISETUJUI (gerbang teknis keras); desain mengikuti cabang OQ-027;
    OQ-046–OQ-055 dijawab dari data V2-007 + kode; TANPA migration
    (migration = tiket L3 terpisah setelah desain disetujui user).
12. Jawab OQ-005/006 (host benar) + OQ-011/012 + OQ-013/014/015 (saat
    eksekusi V2-004) + OQ-016/017/018 (saat eksekusi V2-005) +
    OQ-019/020/021/022/023 (saat eksekusi V2-006) + OQ-024/025/026/027/028
    (saat eksekusi V2-007) + OQ-029/030/031/032/033/034/035 (saat eksekusi
    V2-008) + OQ-036/037/038/039/040/041/042/043/044/045 (saat eksekusi
    V2-009) + OQ-046/047/048/049/050/051/052/053/054/055 (saat eksekusi
    desain V2-010).
13. V2-009 F.2–F.5 dapat dikerjakan sebagai tugas lokal siklus berikut
    (quote storage/domain/monitoring dari sumber publik + hitung volume
    backup + hitung ulang TCO dengan label DF/ESTIMASI) — verdict C-1/C-2/C-4
    tetap "TIDAK DAPAT DINILAI" sampai angka panel F.1 tiba.
