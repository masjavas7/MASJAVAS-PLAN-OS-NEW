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
- `MASTER_ARCHITECTURE_MASJAVAS_TUBE_OS_V2.md` — baseline arsitektur
  V2.0 (DRAFT BASELINE, beku): topologi dua-node, orkestrasi API +
  SQLite, kepemilikan job, retry, isolasi sumber daya, backup/DR,
  monitoring, deployment.
- `IMPLEMENTATION_CHECKLIST_MASJAVAS_TUBE_OS_V2.md` — checklist
  eksekusi V2.0 (beku): track P0–P10, dependency order, template
  tiket V2-XXX, Batch 1 V2-001–V2-010.
- `BATCH1_EXECUTION_TICKETS_V2.md` — tiket eksekusi Batch 1
  (V2-001–V2-010, format 15 elemen; tiket V2-001–V2-004 lengkap —
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

### Runbook VPS 8

- `VPS8_DEPLOYMENT_RUNBOOK.md` — langkah deployment Fase 0 → 1 → 2
  dengan verifikasi tiap langkah + rujukan rollback (DRAFT).
- `VPS8_ACCEPTANCE_TEST_PLAN.md` — 30+ kasus uji dengan kriteria
  lolos/gagal terukur, dikaitkan ke scorecard 13 metrik.
- `VPS8_ROLLBACK_RUNBOOK.md` — pemicu + prosedur rollback per fase,
  RPO/RTO sebagai target, otoritas keputusan owner.

### Sprint audit repo (P0)

- `VPS8_RESOURCE_CAPACITY_MODEL.md` — model kapasitas CPU VPS 8:
  budget per workload, reserve Live 2 core + OS/daemon 0,5 core,
  batas konkurensi, admission control berbasis CPU pressure;
  puncak dihitung ulang ≈7,4 core, headroom ±0,6 core.
- `VPS8_PURCHASE_GATE.md` — gerbang pembelian yang diperkuat:
  BELI ⟺ Fase 0 lolos AND (Demand OR Risk) AND V2-009 lolos AND
  rollback ada AND persetujuan owner; 9 sinyal risiko dengan
  threshold + sumber data.

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

## Ringkasan arsitektur target

- **Master** (Contabo, 4 vCPU): control plane + publish + prep (sementara).
- **Live-01 / VPS 8** (Contabo Core VPS 8, 8 vCPU / 24 GB / 600 Mbit/s):
  live streaming + prep worker + salinan DR operasional.
- Antar-box via Tailscale ($0, sudah jalan); SQLite tetap di Master,
  worker remote lewat REST API (bukan akses DB lewat jaringan).
