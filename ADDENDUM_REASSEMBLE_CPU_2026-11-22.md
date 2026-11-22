# ADDENDUM — Koreksi modal CPU ingestion: reassemble audio (22 Nov 2026, 12:10–12:20 WIB)

**Status:** ADDENDUM (turunan). Dokumen beku V2.0 TIDAK diubah; addendum ini
mencatat bacaan yang benar atas klaim yang kini diketahui menyesatkan.

## 1. Koreksi (dari Bapak, main chat 12:08 WIB)

Plan VPS 8 menganggarkan ingestion **±0 CPU** dengan alasan "CPU ingestion
0,0% di semua 1.510 tick governor 24 jam". Angka itu **artefak atribusi**,
bukan bukti ingestion tidak butuh CPU:

- Perakitan audio (reassemble, `core/curation/audio_reassembler.py`) adalah
  satu wadah dengan ingestion — `worker/ingestion.py` memanggil
  `process_and_reassemble_ingestion_audio` saat mengisi audio channel.
- Tapi job-nya jalan sebagai anak proses web/daemon (juga bisa dipicu lewat
  API `POST /reassemble-audio`), dan `classify_lane` governor hanya mengenali
  cmdline berisi "ingestion" — sehingga CPU reassemble tercatat di bucket
  **render/web**, bukan ingestion. (Akar yang sama dengan temuan V2-003:
  atribusi ingestion buta; daemon otonom ±18% CPU tercatat lane `other`.)
- Bukti 22 Nov 11:33–12:06 WIB: job reassemble World_Pop 90,7% (6,5 mnt)
  lalu slot 007 Luka Senja 85,7%; governor mencatat render CPU 35–214%
  sementara ingestion 0,0%; load server 4,63 → 3,32 begitu satu job selesai.

## 2. Angka yang benar (terukur)

| Item | Nilai | Sumber |
|---|---|---|
| Frekuensi reassemble | 64×/24 jam | Plan §1.5 (basis bukti 22 Nov) |
| CPU per job | ±0,85–0,9 core, ±6 mnt/job (maks 2 thread) | Terukur 22 Nov 11:33–12:06 |
| CPU rata-rata | ±5,8 core-jam/hari ≈ **0,25 core** | 64 × 6 mnt × 0,9 ÷ 24 jam |
| CPU puncak | **±1 core** (job berurutan, 3 job/35 mnt pagi ini) | Terukur 22 Nov 11:33 |
| RAM per job | ±70 MB (≈0,1 GB) — kecil | Terukur 22 Nov 12:06 |

Yang kelewat adalah modal **CPU**, bukan RAM. Stage 2 (214% CPU + 1,1 GB,
run-1226) sudah masuk anggaran (cgroup 4 core, 4 GB).

## 3. Perbaikan di Plan VPS 8 (3 edit, TERVERIFIKASI 12:14 WIB)

File: `PLAN_PENAMBAHAN_VPS8_2026-11-22.md` (salinan engineering; salinan
`~/workspace/user/files/` disinkronkan dari salinan engineering pada
12:20 WIB — diff membuktikan satu-satunya perbedaan adalah 3 edit ini):

1. §1.3: catatan KOREKSI soal artefak atribusi governor.
2. §4: baris baru "Reassemble audio (kurasi; satu wadah ingestion)" —
   ±1 core puncak (0,25 rata-rata), 0,1 GB/job, ikut prep worker ke VPS 8
   (Fase 1).
3. Total VPS 8: ≈6 core → **≈7 core puncak; headroom menyempit ke ±1 core**
   pada puncak (8 vCPU − 7).

## 4. Dampak pada dokumen beku V2.0 (bacaan yang benar)

- `MASTER_ARCHITECTURE_..._V2.md` l.27 — "Audit modal terverifikasi:
  ingestion CPU 0,0% (1.510 tick)": benar sebagai **bacaan governor**,
  MENYESATKAN sebagai klaim sumber daya. Jangan dipakai lagi sebagai bukti
  modal ingestion.
- `MASTER_ARCHITECTURE_..._V2.md` l.40 + `IMPLEMENTATION_CHECKLIST_..._V2.md`
  l.22 — "Modal ingestion 1 core / 2 GB RAM (orchestrator §4.3)":
  **DIKONFIRMASI oleh koreksi ini.** Anggaran 1 core V2.0 ternyata sudah
  benar — reassemble-lah ~1 core itu (puncak). Koreksi Plan (§4: ingestion
  ±0 + reassemble ±1 puncak) kini SELARAS dengan V2.0. Tidak ada perubahan
  angka di V2.0 yang diperlukan; yang berubah hanya penjelasan asal
  1 core tersebut.
- `IMPLEMENTATION_CHECKLIST_..._V2.md` l.25 — "Ingestion CPU 0,0% ...
  — benar": benar sebagai bacaan governor; kualifikasi yang sama berlaku.
- RAM ingestion 2 GB di V2.0 tetap berdiri (reassemble ±0,1 GB/job).

## 5. Konsekuensi untuk program

- Headroom VPS 8 pada puncak kini ±1 core (bukan ±2). Implikasi untuk
  V2-008 (benchmark anggaran sumber daya): angka puncak prep worker harus
  memakai ±1 core reassemble, dan canary Stage 2 (2 slot) harus diuji
  BERSAMAAN dengan reassemble berjalan, bukan sendirian.
- V2-003-R2 (perbaiki atribusi lane) kini punya bukti tambahan: artefak
  yang sama sudah pernah menyesatkan satu dokumen anggaran.
- Tidak ada perubahan biaya: total $34,67/bln tidak berubah (box yang sama).
