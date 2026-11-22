# KRITERIA PENERIMAAN PRODUKSI — MASJAVAS TUBE OS V2

**Status:** DOKUMEN RENCANA (Workstream H + Track P10). Baseline dan
target diambil dari scorecard V2.0 §18 + Plan §2; kriteria ini adalah
gerbang yang dipakai SETIAP fase dan SETIAP perubahan (bukan hanya
saat pembelian VPS 8).
**Sumber utama:** `MASTER_ARCHITECTURE_MASJAVAS_TUBE_OS_V2.md` §18
(Scorecard), §25 (Definition of Done), §24 (Open Decisions);
`IMPLEMENTATION_CHECKLIST_..._V2.md` Track P10, §0;
`PLAN_PENAMBAHAN_VPS8_2026-11-22.md` §2, §6; `v2-002/` (koreksi
atribusi kematian encoder); `ENGINEERING_READINESS_DASHBOARD.md`.
**Terakhir diperbarui:** 22 Nov 2026.

**Label bukti:** `[VERIFIED FACT]` / `[DOCUMENTED FACT NOT YET REVALIDATED]` /
`[ASSUMPTION]` / `[PROPOSAL]` / `[OPEN QUESTION]` /
`[OWNER DECISION REQUIRED]`.

---

## 1. Baseline "solid" — yang harus dilindungi

Baseline = kondisi terverifikasi 21–22 Nov 2026. **Setiap fase harus
mempertahankan atau memperbaiki SEMUA baris; satu baris memburuk =
gerbang gagal** (Plan §2).

| # | Metrik | Baseline (21–22 Nov) | Target Fase 0 | Target Fase 1 |
|---|---|---|---|---|
| 1 | Publish/hari | 206 (21 Nov) | ≥200, 0 FAILED | ≥200, 0 FAILED |
| 2 | FAILED/BLOCKED | 0 | 0 | 0 |
| 3 | Keterlambatan terbit p90 | 30,2 menit | ≤30 menit | ≤25 menit |
| 4 | Keterlambatan terbit median | 11,6 menit `[DOCUMENTED FACT NOT YET REVALIDATED — tidak muncul di tabel scorecard V2.0 §18/Plan §2; konfirmasi di V2-005]` | ≤ baseline | ≤ baseline |
| 5 | Kematian encoder tak direncanakan | 5×/24 jam (lihat koreksi §1.1) | ≤1×/24 jam (setelah P16) | 0×/72 jam |
| 6 | Encoder orphan | 1 | 0 | 0 |
| 7 | "database is locked"/24 jam | 31× | <20× | <10× |
| 8 | Load Master (rata-rata) | 3,5–4,2 | <3,0 | <2,5 |
| 9 | CPU unit web (rata-rata) | 1,02 core | <0,2 core | <0,2 core |
| 10 | Ingestion RUNNING saat backlog ada | 2 job | ≥3 job (konkurensi penuh) | ≥3 di Master + penuh di VPS 8 |
| 11 | Churn SUPERSEDED ingestion | 66% (7 hari) | <20% | <20% |
| 12 | Disk Master (puncak wave) | 72% | ≤70% | ≤70% |
| 13 | Healthz p95 | ±10 ms | <100 ms | <100 ms |
| 14 | Stream live aktif | 2 | 2 (stabil) | Sesuai demand, s.d. 16 → 50 |
| 15 | Error class baru pasca-fase | — (belum ada baseline) | 0 dalam 72 jam observasi | 0 dalam 72 jam observasi |

### 1.1 Koreksi atribusi kematian encoder [VERIFIED FACT — V2-002]

Plan §2 baris 3 menulis kematian "5×/24 jam (slot 706)". Audit V2-002
(OQ-007) membuktikan pemetaan itu SALAH: stream yang mati berulang
(`1795226249006`) adalah **STAGING-SLOT-114** ("RUANG GALAU"), bukan
706. Angka laju 5×/24 jam tetap dipakai sebagai baseline sampai V2-005
menetapkan baseline baru — tetapi atribusi slotnya dikoreksi.

**Diskrepansi jendela yang belum direkonsiliasi:** V2-002 menghitung
12 kematian dalam 17 jam (21 Nov 14:00 → 22 Nov 07:00), dengan interval
yang memendek hingga 9–13 menit — laju yang lebih tinggi dari baseline
5×/24 jam. Baseline mana yang benar untuk gerbang penerimaan adalah
`[OPEN QUESTION — ditetapkan oleh V2-005 baseline snapshot]`; sampai
saat itu, target "≤1×/24 jam" dan "0×/72 jam" tetap berlaku apa pun
baseline awalnya, dan perbaikan encoder 114 (commit `56f7a6d` +
`021a9f0`, observasi 24 jam berjalan) dinilai terhadap target itu.

## 2. Gate policy (mengikat)

Dari V2.0 §18 + Checklist §0.1 + Plan §6:

1. Pre-change snapshot.
2. Baseline (window yang sama dengan metrik yang dibandingkan).
3. Implementasi terbatas (canary — tidak ada perubahan massal tanpa
   canary).
4. Post-change validation.
5. Observasi 72 jam untuk perubahan operasional.
6. Observasi 7 hari untuk metrik yang memang membutuhkan window
   7 hari (churn, publish/hari). **Jangan membandingkan baseline
   7 hari dengan hasil 72 jam seolah-olah setara.**
7. Rollback bila metrik kritis memburuk.

**Satu metrik kritis memburuk = gate gagal = rollback.** Tidak ada
pengecualian "memburuk sedikit" — ambang ada di tabel §1.

## 3. Gerbang per fase (Plan §6)

### 3.1 Gerbang Fase 0 ($0, Master saja)

Scorecard baris 1, 2, 5, 6, 7, 8, 9, 11, 13 (§1: publish, FAILED,
kematian encoder, orphan, lock, load, CPU web, ingestion RUNNING,
churn, healthz, error class baru) membaik atau tetap selama 72 jam;
0 error class baru. Langkah Fase 0: pisah proses publish/prep; job
berat keluar dari web (CPU web <0,2 core; 0 anak ffmpeg di web);
klaim pull + 1-owner (churn <20% dalam 7 hari); sweeper batch 2→5;
diagnosis P16 (0 kematian/72 jam; 0 orphan); janitor status-only
(0 GB file CONNECTED terhapus berlabel "unpaired" dalam audit 24 jam).

### 3.2 Gerbang Fase 1 (VPS 8)

**Pemicu pembelian (harus SEMUA terpenuhi):** (a) Fase 0 lolos gerbang;
(b) live mendekati 12–16 stream ATAU 6 stream key sudah terisi semua +
antrean channel siap-live ≥10; (c) harga Master terverifikasi di panel
Contabo dan total ≤ $34,67/bln. `[OWNER DECISION REQUIRED — D4]`

**Gerbang pasca-1.6:** scorecard SEMUA baris membaik atau tetap selama
72 jam; Master tidak lagi menjalankan live reguler.

## 4. Kriteria khusus VPS 8

| # | Kriteria | Ambang / cara uji | Status |
|---|---|---|---|
| V1 | Live 1080p untuk semua channel | Semua stream baru + canary berjalan 1080p (keputusan user; default 720p di `.env` masih stale — V2-003-R8). | `[VERIFIED FACT — keputusan user]` |
| V2 | Kapasitas TX | 50 stream ≈ 310 Mbps egress `[ASSUMPTION — V2.0 §4.3: harus divalidasi dengan bitrate aktual + headroom]`. Admission cap 70% port = 420 Mbps (port 600 Mbps). | `[ASSUMPTION — validasi di P6.4]` |
| V3 | Prep tidak mengganggu live | TX live tidak terganggu saat prep penuh (langkah 1.6); governor menolak di atas batas; Master responsif. | `[PROPOSAL — diuji di canary 1.5/1.6]` |
| V4 | Canary live | 1–2 stream stabil 72 jam sebelum perluasan (langkah 1.3). | `[PROPOSAL — Plan §6]` |
| V5 | Canary prep | Ingestion 1 job + Stage 2 1 slot: klaim API jalan, 0 gangguan live, guard lokal bekerja (langkah 1.5). | `[PROPOSAL — Plan §6]` |
| V6 | Skala Stage 2 | 2 slot canary dulu; naik ke 3–4 HANYA bila pengukuran CPU/RAM/I-O/disk/live TX membuktikan aman (V2.0 §4.2). | `[PROPOSAL]` |
| V7 | Headroom puncak | Total puncak ≈7 core dari 8 vCPU (termasuk reassemble ±1 core — addendum 22 Nov 12:10); headroom ±1 core pada puncak. | `[DOCUMENTED FACT — Plan §4 pasca-koreksi]` |
| V8 | Port terukur | Port 600 Mbit/s terukur saat provisioning (langkah 1.1), bukan hanya klaim listing. | `[PROPOSAL — Plan §6]` |
| V9 | DR replica | Restore uji berhasil — DB + app jalan dari salinan (langkah 1.4). | `[PROPOSAL — Plan §6]` |
| V10 | Harga terverifikasi | Total riil ≤ $34,67/bln (cap mengikat); bila lebih, eskalasi — jangan lanjut diam-diam. | `[OWNER DECISION REQUIRED — V2-009]` |

## 5. Kriteria backup & recovery (Track P8.4)

- Backup berada di luar kedua VPS.
- Restore berhasil (drill bulanan — lihat
  `MASJAVAS_BACKUP_RECOVERY_PLAN.md` §6).
- RPO dan RTO DIBUKTIKAN, bukan diasumsikan (target RPO ≤5 mnt /
  RTO ≤30 mnt adalah TARGET sampai drill membuktikannya).
- Backup tidak mengganggu live.

## 6. Kriteria deployment (Track P9)

- Tidak ada job dikirim ke worker inkompatibel.
- Deployment tidak memutus semua live bersamaan.
- Rollback release dapat dilakukan tanpa merusak schema.

## 7. Final gate (Track P10) + Definition of Done (V2.0 §25)

Final gate P10 — SEMUA harus terpenuhi:

- Semua metrik kritis tidak memburuk.
- Tidak ada error class baru selama 72 jam.
- Metrik 7 hari dievaluasi dengan window 7 hari.
- Live stabil. Publish stabil.
- Job tidak hilang. Job tidak menghasilkan side effect ganda.
- Backup dapat direstore. Rollback sudah diuji.
- Biaya aktual diketahui. Runbook operasional tersedia.
- Owner menyetujui hasil.

Definition of Done V2.0 §25 (15 butir — ringkas): Master satu-satunya
pemegang kebenaran ownership; worker register/claim/heartbeat/selesai;
duplikasi dicegah/direkonsiliasi; worker basi tidak menimpa hasil baru;
restart prep tidak mengganggu publish; beban prep tidak mengganggu live;
live tetap berjalan saat Master tidak tersedia; worker mati pulih tanpa
kehilangan status; artifact tidak dihapus tanpa verifikasi lifecycle;
deployment lintas versi terkendali; monitoring mendeteksi business
failure; backup dapat dipulihkan dalam target yang teruji; biaya
bulanan + storage diketahui; scorecard tidak memburuk pasca-migrasi;
semua perubahan punya bukti verifikasi dan rollback.

## 8. Cara mengukur (per metrik)

| Metrik | Sumber ukur | Window |
|---|---|---|
| Publish/hari, FAILED/BLOCKED | DB `operational_tasks` / dashboard | 24 jam; 7 hari untuk tren |
| Delay terbit p90/median | Timestamp jadwal vs terbit per task | 7 hari (metrik 7 hari) |
| Kematian encoder | Log R6 supervisor (`Stream #... mati`) + hitungan DB | 24 jam; 72 jam untuk target Fase 1 |
| Orphan | Sweep startup R8 + `ps` / cgroup | Tiap restart + harian |
| SQLite lock | Hitung `database is locked` di log | 24 jam |
| Load / CPU web / RAM / disk | Tick governor + `uptime`/`df`/`free` | Kontinu; rata-rata harian |
| Ingestion RUNNING / churn | DB ingestion jobs (COMPLETED vs SUPERSEDED) | 7 hari untuk churn |
| Healthz p95 | Monitor eksternal / log | 24 jam |
| Live TX | Sampler TX governor (EMA 300-dtk + instan) | Kontinu |
| Error class baru | Klasifikasi error log pasca-perubahan | 72 jam observasi |

Detail implementasi pengukuran (query, dashboard, alert) adalah
Track P8.2/P8.3 — `[PROPOSAL]`.

## 9. Pertanyaan terbuka

1. Baseline kematian encoder yang benar (5×/24 jam vs 12×/17 jam) —
   `[OPEN QUESTION — V2-005]`.
2. Baseline median delay 11,6 mnt — konfirmasi sumber + window —
   `[OPEN QUESTION — V2-005]`.
3. Validasi estimasi 310 Mbps untuk 50 stream — `[OPEN QUESTION — P6.4]`.
4. Ambang "metrik kritis" vs "metrik non-kritis" untuk gate gagal —
   saat ini semua baris §1 diperlakukan kritis (Plan §2); bila owner
   ingin membedakan, itu `[OWNER DECISION REQUIRED]`.
5. Target RPO/RTO final setelah drill pertama — `[OPEN QUESTION — P8.4]`.
