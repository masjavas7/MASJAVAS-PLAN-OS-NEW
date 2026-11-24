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

## Ringkasan arsitektur target

- **Master** (Contabo, 4 vCPU): control plane + publish + prep (sementara).
- **Live-01 / VPS 8** (Contabo Core VPS 8, 8 vCPU / 24 GB / 600 Mbit/s):
  live streaming + prep worker + salinan DR operasional.
- Antar-box via Tailscale ($0, sudah jalan); SQLite tetap di Master,
  worker remote lewat REST API (bukan akses DB lewat jaringan).
