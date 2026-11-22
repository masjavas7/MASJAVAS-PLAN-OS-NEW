# MASJAVAS PLAN OS NEW

Paket rencana deployment **MASJAVAS TUBE OS** — dari satu box (Master)
ke dua box (Master + Live-01 di VPS 8), supaya live streaming, ingestion,
dan Stage 2 tidak lagi saling berebut sumber daya di satu mesin.

## Status

**DRAFT — belum siap deploy.** Kesiapan keseluruhan: **NOT READY**
(lihat `ENGINEERING_READINESS_DASHBOARD.md`). Gerbang yang belum lolos:
harga Master + VPS 8 belum terverifikasi di panel Contabo (V2-009),
biaya storage/backup belum dihitung, Batch 1 baru 3/10, dan persetujuan
owner belum ada. Anggaran mengikat: total **$34,67/bln**
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
  (V2-001–V2-010, format 15 elemen; V2-001–V2-004 lengkap).
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

### Register + rencana pendukung (menyusul — dalam penyusunan)

- `MASJAVAS_RISK_REGISTER.md`, `MASJAVAS_OWNER_DECISION_REGISTER.md`,
  `MASJAVAS_COST_REGISTER.md`, `MASJAVAS_DEPENDENCY_GRAPH.md`
- `MASJAVAS_BACKUP_RECOVERY_PLAN.md`,
  `MASJAVAS_RELEASE_MANAGEMENT_PLAN.md`,
  `MASJAVAS_SECURITY_HARDENING_PLAN.md`,
  `MASJAVAS_OPERATIONAL_RUNBOOK.md`,
  `MASJAVAS_PRODUCTION_ACCEPTANCE_CRITERIA.md`

## Ringkasan arsitektur target

- **Master** (Contabo, 4 vCPU): control plane + publish + prep (sementara).
- **Live-01 / VPS 8** (Contabo Core VPS 8, 8 vCPU / 24 GB / 600 Mbit/s):
  live streaming + prep worker + salinan DR operasional.
- Antar-box via Tailscale ($0, sudah jalan); SQLite tetap di Master,
  worker remote lewat REST API (bukan akses DB lewat jaringan).
