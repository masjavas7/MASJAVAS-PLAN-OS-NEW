# MASJAVAS PLAN OS NEW

Paket rencana deployment **MASJAVAS TUBE OS** — dari satu box (Master)
ke dua box (Master + Live-01 di VPS 8), supaya live streaming, ingestion,
dan Stage 2 tidak lagi saling berebut sumber daya di satu mesin.

## Status

**DRAFT — belum siap deploy.** Kesiapan keseluruhan: **NOT READY**
(lihat `ENGINEERING_READINESS_DASHBOARD.md`). Gerbang yang belum lolos:
harga Master + VPS 8 belum terverifikasi di panel Contabo (V2-009),
biaya storage/backup belum dihitung, Batch 1: 1 EXECUTED (dengan
keterbatasan bukti) + 2 READY_FOR_REVIEW dari 10 (sumber status resmi:
`STATUS_REGISTRY.yaml`), dan persetujuan owner belum ada.
Anggaran mengikat: total **$34,67/bln**
(Master ±$16,02 + VPS 8 $18,65).

Repo ini dibangun berkelanjutan sampai siap deploy: setiap dokumen di
sini adalah turunan dari program readiness — bukan instruksi deployment
sampai semua gerbang lolos.

## Isi repo

### Rencana + arsitektur (inti)

- `PLAN_PENAMBAHAN_VPS8_2026-11-22.md` — rencana penambahan VPS 8:
  konteks problem (terverifikasi 22 Nov 2026), scorecard 13 metrik,
  prinsip desain, modal semua worker, arsitektur target, phasing +
  gerbang verifikasi (Fase 0 $0 → Fase 1 VPS 8 → Fase 2 box ke-3),
  biaya, risiko, keputusan yang dibutuhkan.
- `ARSITEKTUR_MULTI_NODE_MASJAVAS_TUBE_OS_V2.md` — desain arsitektur multi-node v1.0 (DRAFT, 23 Nov 2026, menunggu persetujuan owner): 10 insiden nyata 22–23 Nov → 7 prinsip mengikat (satu sumber kebenaran Master DB, penugasan eksplisit, lease, mutasi via API, fail-safe, teramati, agnostik-node); skema DB baru (`live_nodes`, kolom lease/migrasi, `node_events`); rencana migrasi F0–F9 (~12–16 hari kerja); 4 pertanyaan terbuka untuk owner.
- `MASTER_ARCHITECTURE_MASJAVAS_TUBE_OS_V2.md` — baseline arsitektur
  V2.0 (DRAFT BASELINE, beku): topologi dua-node, orkestrasi API +
  SQLite, kepemilikan job, retry, isolasi sumber daya, backup/DR,
  monitoring, deployment.
- `IMPLEMENTATION_CHECKLIST_MASJAVAS_TUBE_OS_V2.md` — checklist
  eksekusi V2.0 (beku): track P0–P10, dependency order, template
  tiket V2-XXX, Batch 1 V2-001–V2-010.
- `BATCH1_EXECUTION_TICKETS_V2.md` — tiket eksekusi Batch 1
  (V2-001–V2-010, format 15 elemen; 10/10 tiket lengkap di sisi dokumen —
  status eksekusi di `STATUS_REGISTRY.yaml`).
- `STATUS_REGISTRY.yaml` — SATU-SATUNYA sumber status resmi tiket +
  gerbang. `ticket_status` (tiket lengkap/stub) dipisah dari
  `work_status` (pekerjaan sebenarnya); keterbatasan bukti dicatat
  di field `limitation`, bukan di nama status.
- `REPOSITORY_ENGINEERING_AUDIT_SPRINT.md` — sprint audit + perbaikan
  repo (8 item P0–P1, dimulai 22 Nov 13:02 atas review repo).
- `ENGINEERING_READINESS_DASHBOARD.md` — dashboard kesiapan: tracker
  Batch 1, checklist gerbang, register risiko + keputusan, register
  biaya, 5 aksi berikut.
- `ADDENDUM_REASSEMBLE_CPU_2026-11-22.md` — koreksi 22 Nov 12:10:
  modal CPU ingestion mencakup perakitan audio (reassemble, ±1 core
  puncak); total VPS 8 ≈7 core puncak, headroom ±1 core.
- `KONTEKS_MASALAH_BOTTLENECK_DAN_SOLUSI.md` — konteks masalah,
  9 bottleneck terverifikasi (B1–B9), 15 problem terstruktur
  (PR-01–PR-15), dan solusi yang diharapkan per fase (Fase 0/1/2+).
- `RIWAYAT_PEMBAHASAN_DAN_LANGKAH.md` — riwayat pembahasan program
  readiness 22 Nov 2026: timeline, pembahasan per topik, keputusan
  owner D1–D11, langkah berikut, dan yang ditolak (agar tak diulang).
- `KONTEKS_PROBLEM_SOLUSI_INSTRUKSI_ANTIGRAVITY_2026-11-24.md` — konteks
  + masalah + solusi + instruksi untuk agen Antigravity (24 Nov 2026,
  06:20): skor soliditas aplikasi 75/100, 3 P0 + 4 P1 + 9 P2 dengan
  kriteria penerimaan terukur, 7 ide, instruksi 5 fase (Diagnosis 6 jam
  → Hulu 24 jam → Live 48 jam → Pengerasan 72 jam → Hutang teknis) +
  7 larangan; status: siap diteruskan owner (D18).

### Runbook VPS 8

- `VPS8_DEPLOYMENT_RUNBOOK.md` — langkah deployment Fase 0 → 1 → 2
  dengan verifikasi tiap langkah + rujukan rollback (DRAFT).
- `VPS8_ACCEPTANCE_TEST_PLAN.md` — 30+ kasus uji dengan kriteria
  lolos/gagal terukur, dikaitkan ke scorecard 13 metrik (v1.1:
  gerbang memakai klasifikasi rollback 3 kategori).
- `VPS8_ROLLBACK_RUNBOOK.md` — pemicu + prosedur rollback per fase,
  RPO/RTO sebagai target, otoritas keputusan owner (v1.1: aturan
  satu-metrik SUPERSEDED — diganti hard breach / SLO degradation /
  transient noise + tabel ambang 13 metrik).

### Sprint audit repo

- `VPS8_RESOURCE_CAPACITY_MODEL.md` — model kapasitas CPU VPS 8:
  budget per workload, reserve Live 2 core + OS/daemon 0,5 core,
  batas konkurensi, admission control berbasis CPU pressure;
  puncak dihitung ulang ≈7,4 core, headroom ±0,6 core.
- `VPS8_PURCHASE_GATE.md` — gerbang pembelian yang diperkuat:
  BELI ⟺ Fase 0 lolos AND (Demand OR Risk) AND V2-009 lolos AND
  rollback ada AND persetujuan owner; 9 sinyal risiko dengan
  threshold + sumber data.
- `MASJAVAS_TCO_MODEL.md` — model total cost of ownership:
  3 skenario vs cap $34,67/bln; temuan utama: cap habis oleh
  infrastruktur saja (minimum PAS di cap, expected +$1,50/bln,
  worst case +$4,97/bln — sebelum storage/domain).
- `MASJAVAS_SLO_ERROR_BUDGET.md` — 8 SLO + error budget; hanya
  3 punya target angka dari baseline, 4 masih OPEN QUESTION.
- `ORCHESTRATOR_API_CONTRACT_V1.md` — kontrak API orchestrator
  pull-based REST v1: 7 endpoint lengkap (schema, auth, idempotency,
  lease, fencing token), katalog error 20 kode, kasus uji wajib
  TC-STALE-1 (worker basi ditolak).
- `VPS8_FAILURE_INJECTION_TEST_PLAN.md` — 11 skenario uji kegagalan
  terencana (Master mati, Tailscale putus, worker restart, SQLite
  terkunci, disk penuh, FFmpeg mati, output sebagian, publish
  timeout, stale worker, backup/restore gagal).

### Register

- `MASJAVAS_RISK_REGISTER.md` — risiko R1–R19 + P1–P16, dampak dan
  mitigasi, status per risiko.
- `MASJAVAS_OWNER_DECISION_REGISTER.md` — keputusan owner D1–D11 +
  item terbuka dari plan §10.
- `MASJAVAS_COST_REGISTER.md` — semua angka biaya dengan label
  TERVERIFIKASI / ESTIMASI / BELUM DIHITUNG; total $34,67/bln
  sebagai batas mengikat.

### Rencana pendukung

- `MASJAVAS_BACKUP_RECOVERY_PLAN.md` — backup off-site, retensi,
  restore drill, RPO/RTO sebagai target.
- `MASJAVAS_RELEASE_MANAGEMENT_PLAN.md` — release manifest, versi,
  canary, rollback release.
- `MASJAVAS_SECURITY_HARDENING_PLAN.md` — akses, kredensial, firewall,
  audit.
- `MASJAVAS_OPERATIONAL_RUNBOOK.md` — operasi harian, eskalasi,
  prosedur insiden.
- `MASJAVAS_PRODUCTION_ACCEPTANCE_CRITERIA.md` — kriteria penerimaan
  produksi per fase.
- `MASJAVAS_DEPENDENCY_GRAPH.md` — diagram dependensi (mermaid) +
  propagasi kegagalan.
- `SARAN_PERBAIKAN_INGESTION_PARALEL_VPS8_2026-11-24.md` — analisa +
  8 saran perbaikan ingestion (24 Nov 2026, 04:45): throttle autopilot
  kedaluwarsa, governor mengerem, QA Stage 2 makan ~3,1 core; langkah
  1–4 sudah diimplementasikan + terverifikasi 04:47–05:22 (PENDING
  43→39, Ingestion Steps ~17/mnt).
- `ANALISA_BOTTLENECK_L5_JANITOR_2026-11-24.md` — analisa bottleneck
  janitor L5 (24 Nov 2026, 04:00): kenapa disk hampir penuh
  (126 GB/145 GB); dua penyebab + ralat owner L5/L6.
- `ANALISA_WORKFLOW_BOTTLENECK_MULTINODE_2026-11-24.md` — analisa
  workflow + bottleneck multi-node (24 Nov 2026, 04:15): penempatan
  file Master vs VPS 8; premis owner sebagian benar, sebagian belum
  terjadi.
- `DESAIN_COCKPIT_SENTINEL_NODE_2026-11-24.md` — desain Cockpit
  Sentinel 6-Lane sadar-node + janitor otonom (24 Nov 2026, 03:47):
  panel `/telemetry` menampilkan VPS tempat kerja + janitor otonom.
- `DESAIN_TELEMETRY_WORKER_VPS8_2026-11-24.md` — desain perbaikan
  telemetry (24 Nov 2026, 03:30): `/telemetry` memantau worker +
  kinerja VPS 8.
- `DESAIN_FITUR_AUDIT_REPACKAGING_VIDEO_2026-11-24.md` — desain fitur
  audit & repackaging video terbit (DRAFT v1.1, 24 Nov 2026, 06:50):
  worker `masjavas-audit`, triase A/B/C/D, audit bertingkat hemat kuota
  (~2.000–3.000 unit/bulan), 12 bottleneck (§13); menunggu persetujuan
  + keputusan K-1 s.d. K-7 (AUDIT-REPACK).
- `ANALISA_FITUR_AUDIT_INSTRUKSI_ANTIGRAVITY_2026-11-24.md` — analisa
  fitur audit Fase 0 yang sudah live + instruksi perbaikan untuk
  Antigravity (24 Nov 2026, 07:41): temuan P0-1 (CTR fabrikasi),
  P0-2 (split-brain VPS 8), P1-1 (API tanpa auth), P1-2, P1-3,
  P2-1 s.d. P2-7 — masing-masing bukti + dampak + perbaikan +
  kriteria penerimaan; larangan: JANGAN bangun Fase 1 sebelum K-1–K-7
  (→ R40).
- `INSTRUKSI_LANGGRAPH_ANTIGRAVITY_2026-11-24.md` — instruksi Pilot
  Fase A LangGraph untuk Antigravity (24 Nov 2026, 08:08–08:30):
  StateGraph deterministik + checkpoint SQLite + HITL, read-only,
  0 LLM, Rp 0; kriteria penerimaan K-1 s.d. K-7; prasyarat mengikat
  P0-1 + P0-2 hijau sebelum run live pertama; Fase B/C DITAHAN
  (D19).
- `INSTRUKSI_AGENT_AI_SUMOPOD_ANTIGRAVITY_2026-11-24.md` — instruksi
  implementasi Fase B agent AI Sumopod (24 Nov 2026, 09:30): B1 agent
  metadata (gpt-4o-mini, prompt diperkeras terverifikasi), B2
  klasifikasi + QA jev-1.14.0, B3 UI manajemen agent (budget cap,
  kill switch, OWNER-only); Fase C (tulis ke YouTube) TETAP DITAHAN
  (D20).
- `DESAIN_PEMILIH_VPS_LIVE_2026-11-24.md` — desain pemilih VPS saat
  start live (24 Nov 2026, 14:17): modal pemilih + skor kesehatan
  0–100 (headroom CPU 35, headroom stream 25, stabilitas 20, jaringan
  10, kesegaran data 10), ambang ≥80 DIREKOMENDASIKAN / 50–79 bisa
  dipakai / <50 hindari / OFFLINE terkunci, default "Otomatis (paling
  sehat)"; Fase A terimplementasi commit `66b3ec7` (14:21–14:38,
  `GET /api/live/nodes`, 17/17 tes) — telemetri node pulih (R42).
- `DESAIN_NOTIFIKASI_LIVE_PROFESIONAL_2026-11-24.md` — desain
  notifikasi live profesional (24 Nov 2026, 14:29): ganti ~62
  `alert()`/`confirm()` di `live_dashboard.html`/`live_create.html`
  dengan toast progres + modal konfirmasi bermerek + badge LIVE;
  terimplementasi commit `b63b733` (14:29–14:43, 62→0, JS 6/6,
  render 200); 2 bug pra-eksis ditemukan (escapeHtml undefined,
  logo.svg 403 — R46).
- `RENCANA_PERBAIKAN_APLIKASI_2026-11-25.md` — rencana perbaikan
  aplikasi v1.4 (25 Nov 2026, 05:02 → 06:15): skor aplikasi 66/100;
  LIVESTREAMING prioritas 1 (perintah user 05:30), Fase 1–3 non-live
  DITUNDA sampai livestreaming beres; koordinator livestreaming
  SELESAI 06:11 (15/15 item kode + 110 tes hijau, HEAD `b6ceb80`);
  migrasi 6/6 selesai 05:30 (Master 0 ffmpeg, DB 6/6 STREAMING);
  klaster kematian ~05:32 + 06:09 + 06:18 (B10, → R54) (D25).
- `DESAIN_JADWAL_LIVE_EDITABLE.md` — desain jadwal live yang bisa
  diedit, disimpan & dipakai ulang (25 Nov 2026, 06:40–06:42, 14
  bagian + 2 lampiran): endpoint edit POST + hot-swap URL relay tanpa
  matikan encoder (`LiveAudioRelayManager.request_source_swap`) +
  antrean order lintas-node `live_task_orders` + optimistic locking
  `config_version` + template + riwayat revisi & rollback 1 klik;
  validasi sumber saat simpan; rollout 6 fase; implementasi
  DISETUJUI 06:43 (koordinator Fase 0–5 berjalan; keputusan §14
  default) (D27).
- `PANDUAN_GUI_MODERN_MASJAVAS.md` — panduan GUI modern (25 Nov 2026,
  BERLAKU untuk semua UI baru): 1 aksen emerald + 4 warna status;
  tipografi 14/12/12-mono, teks <12px dilarang; maks 2 tombol per
  baris + menu ⋯; form create → wizard 4 langkah; migrasi bertahap
  5 tahap (D27).
- `RENCANA_1080P_4_STREAM_2026-11-25.md` — rencana migrasi 720p →
  1080p 4 stream live (25 Nov 2026, ~05:40): RENCANA, belum
  dieksekusi, menunggu persetujuan per stream; encoder memakai
  `-c:v copy` (output = resolusi file sumber, bukan kolom DB);
  2 dari 4 sumber SUDAH 1080p (label DB salah); VPS 8 JENUH saat
  rencana dibuat (load 14,3/8, 13 kematian encoder/jam).

### Dokumen baru Cycle 23 (25 Nov 2026, 10:50 WIB)

- `DIAGNOSIS_FITUR_LIVE_STREAMING.md` — diagnosis fitur live
  streaming pasca-Fase 3 + pasca-D1–D6 (375 baris): 8/8 STREAMING,
  3 stream BISU (NO 114, 88, 34), relay HLS 0/6, A2 blokir E5.
- `RENCANA_PERBAIKAN_LIVE_STREAMING_2026-11-25_FINAL.md` —
  rencana FINAL perbaikan live streaming (197 baris, disetujui
  user 09:24): urutan A2 → A3 → A1 per distribusi terukur; cap
  $34,67/bln hanya cukup sampai 50 stream (D28).
- `PROMPT_AGENT_IMPLEMENTASI_LIVE_STREAMING_2026-11-25_FINAL.md`
  — prompt agen implementasi FINAL (103 baris, disetujui 09:24);
  Fase 0 berjalan: A3 `51a7824`, A1 `7826bbb`, A2 `6dc5ad7`,
  E4 `7f4f13e` — SEMUA belum aktif (D28).
- `FASE0_BASELINE_2026-11-25.md` — baseline Fase 0 (09:30–09:48,
  hanya-baca): 8/8 STREAMING, publish 30/0, load VPS 8 7,39/7,37/7,29.
- `DIAGNOSIS_MONA_LIVE_MANUAL_DAN_AUDIO_GDRIVE.md` — diagnosis
  MONA 698 + audio GDrive (07:59–08:10): tugas live GDrive di VPS 8
  WAJIB pre-copy sebelum Start; 698 LIVE 09:03 (D32).
- `ADENDUM_EDIT_METADATA_LIVE.md` — adendum edit metadata live:
  3 celah terverifikasi (pesan sukses PALSU, sinkron tertunda
  tidak pernah diulang, edit deskripsi tertunda HILANG TOTAL).
- `RENCANA_PERBAIKAN_LIVE_STREAMING_2026-11-25_v2.md` +
  `PROMPT_AGENT_IMPLEMENTASI_LIVE_STREAMING_2026-11-25_v2.md` —
  revisi v2 rencana + prompt (disetujui user 09:14; digantikan FINAL).
- `LAPORAN_PAIRING_PRIORITAS_2026-11-25.md` — laporan pairing
  prioritas: CONNECTED 245, UNPAIRED 462 (459 tanpa vault);
  60 vault, 2 over-cap, 54 slot kosong (Fase 3.1).
- `SPESIFIKASI_MONITORING_PAIRING_MINGGUAN.md` — spesifikasi
  monitoring pairing mingguan (Fase 3.1).
- `ALERT_CRON_SPECS_3.6.md` + `PANDUAN_UPTIMEROBOT_HEARTBEAT_3.6.md`
  — 3 spesifikasi alert cron + panduan UptimeRobot heartbeat (Fase 3.6).
- `PAKET_KEPUTUSAN_FASE_C_2026-11-25.md` — paket keputusan Fase C:
  98 eligible, 10/batch, 4.900 unit; Fase C apply BELUM ADA KODE.
- `PROTOKOL_VERIFIKASI_CHECKPOINT_19_00_2026-11-25.md` — protokol
  verifikasi checkpoint 19:00 (audit timer berjalan 01:00 WIB,
  bukan 19:00 WIB — R51).

Catatan: `LAPORAN_ATRIBUSI_PUSH_REBASE_2026-11-25.md` DITAHAN dari
repo publik — memuat IP publik operator; butuh keputusan owner/redaksi
sebelum dipublikasikan (R57–R61 baru, D28–D32 baru, lihat dashboard).

### Dokumen baru Cycle 24 (25 Nov 2026, 14:50 WIB)

- `DIAGNOSIS_M8_2026-11-25.md` — diagnosis M8 SELESAI + terverifikasi
  (14:51): A2 prebuilt GAGAL 0,1746 core/stream (feeder 0,0548 +
  encoder 0,1199), 25/30 interval >0,15; biaya = kernel+wakeup, bukan
  kerja data ffmpeg; VPS 8 hanya menghantar ~4,3–5,0 core/dtk (bukan
  8,0) — matematika rollout ditulis ulang (cap = maks ~16–19 stream;
  50 = 3 box $71,97; 100 = 6 box $127,92; 696 = 35–41 box $668–$781).
- `EVALUASI_B1_2026-11-25.md` — evaluasi B1 SELESAI (12:00–12:35,
  read-only): rekomendasi opsi (a) pindah prep_worker ke Master
  BERTAHAP (Stage 2 dulu, ingestion setelah soak 24–48 jam); opsi (b)
  tidak bisa capai 50 stream secara aritmetika; prep VPS 8 praktis
  mati (0,12 core, 0 ffmpeg, 50.412 yield/jam, 49 DONE/24 jam);
  aktivasi = L3, menunggu persetujuan user.
- `AUDIT_D1_METRIK_PER_STREAM_2026-11-25.md` — audit D1 Fase 3:
  metrik per-stream + dashboard + alert Telegram (yang sudah ada vs
  celah: frame drop dup/drop tidak di-parse, %CPU per-stream tidak
  diekspos, `cpu_display` placeholder hardcode yang salah).
- `PROMPT_HERMES_ANALISA_SOLID_100_2026-11-25.md` — prompt instruksi
  untuk Hermes menganalisa aplikasi menuju solid 100% (14:32):
  peran analyser-only, definisi solid 100% terukur per komponen +
  scorecard 13 metrik, keadaan awal 66/100, 6 fase kerja, output
  AnalysisBrief DRAFT.

DITAHAN dari repo publik siklus ini (memuat IP publik — butuh keputusan
owner/redaksi sebelum dipublikasikan): `PROMPT_CHAT_UTAMA_PERBAIKAN_
NONSTOP_SOLID_100_2026-11-25.md` (dokumen kanonis mandat D33, memuat IP
publik Master) + `PROGRESS_IMPLEMENTASI_LIVE_STREAMING.md` (memuat IP
publik Master + IP publik operator). KOREKSI mengikat siklus ini:
angka ghost-kill 785 TANPA SUMBER di DB/kode (berhenti kutip sebagai
fakta keras); klaim "TERBANTAH" Hermes soal PIN salah sasaran (rotasi
tetap diperlukan; rotasi PIN OWNER-ONLY per aturan 15:03).
R62–R67 baru, D33–D34 baru, OQ-101–OQ-105 baru (OQ-105 TERJAWAB,
OQ-099 + OQ-056 TERJAWAB SEBAGIAN).

## Ringkasan arsitektur target

- **Master** (Contabo, 4 vCPU): control plane + publish + prep (sementara).
- **Live-01 / VPS 8** (Contabo Core VPS 8, 8 vCPU / 24 GB / 600 Mbit/s):
  live streaming + prep worker + salinan DR operasional.
- Antar-box via Tailscale ($0, sudah jalan); SQLite tetap di Master,
  worker remote lewat REST API (bukan akses DB lewat jaringan).
