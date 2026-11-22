# MASJAVAS PLAN OS NEW

Rencana penambahan 1 VPS 8 untuk **MASJAVAS TUBE OS** — dari satu box
(Master) ke dua box (Master + Live-01), supaya live streaming, ingestion,
dan Stage 2 tidak lagi saling berebut sumber daya di satu mesin.

## Status

**DRAFT — belum siap dieksekusi.** Gerbang pembelian belum lolos:
harga Master + VPS 8 belum terverifikasi di panel Contabo (V2-009),
biaya storage/backup belum dihitung, dan persetujuan owner belum ada.
Anggaran mengikat: total **$34,67/bln** (Master ±$16,02 + VPS 8 $18,65).

## Isi repo

- `PLAN_PENAMBAHAN_VPS8_2026-11-22.md` — rencana lengkap: konteks problem
  (semua terverifikasi 22 Nov 2026), scorecard 13 metrik, prinsip desain,
  modal semua worker, arsitektur target, phasing + gerbang verifikasi
  (Fase 0 $0 → Fase 1 VPS 8 → Fase 2 box ke-3), biaya, risiko, dan
  keputusan yang dibutuhkan.
- `ADDENDUM_REASSEMBLE_CPU_2026-11-22.md` — koreksi 22 Nov 12:10:
  modal CPU ingestion mencakup perakitan audio (reassemble, ±1 core
  puncak); total VPS 8 ≈7 core puncak, headroom ±1 core.

## Ringkasan arsitektur target

- **Master** (Contabo, 4 vCPU): control plane + publish + prep (sementara).
- **Live-01 / VPS 8** (Contabo Core VPS 8, 8 vCPU / 24 GB / 600 Mbit/s):
  live streaming + prep worker + salinan DR operasional.
- Antar-box via Tailscale ($0, sudah jalan); SQLite tetap di Master,
  worker remote lewat REST API (bukan akses DB lewat jaringan).
