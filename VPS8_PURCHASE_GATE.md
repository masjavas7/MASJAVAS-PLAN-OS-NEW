# GERBANG PEMBELIAN VPS 8 — MASJAVAS TUBE OS V2

**Status:** DRAFT — spesifikasi gerbang, BUKAN persetujuan pembelian.
Pembelian tetap butuh persetujuan owner (Level C) setelah semua
dimensi gerbang lolos.
**Versi:** 1.0 — 22 Nov 2026.
**Ruang lingkup:** memperkuat pemicu pembelian PLAN §6 (b) — dari
"jumlah stream" menjadi **demand + operational risk + verified
cost**. PLAN tidak diedit; dokumen ini adalah spesifikasi operasional
gerbang yang dirujuk `STATUS_REGISTRY.yaml` (`gates.purchase_gate`).
**Sumber:** `PLAN_PENAMBAHAN_VPS8_2026-11-22.md` §2 (scorecard), §6
(phasing + gerbang), §7 (biaya), §10 (keputusan); `STATUS_REGISTRY.yaml`;
`MASJAVAS_COST_REGISTER.md`; `v2-002/` (admission + encoder);
`v2-003/` (guard + admission palsu); `VPS8_RESOURCE_CAPACITY_MODEL.md`.
**Item sprint:** Repository Engineering Audit Sprint, P0 #3 (F4).

## Legenda label klaim

- **[VF]** VERIFIED FACT — terverifikasi langsung (log/DB/pengukuran).
- **[DF]** DOCUMENTED FACT NOT YET REVALIDATED — tercatat di dokumen
  sumber, belum divalidasi ulang sejak ditulis.
- **[AS]** ASSUMPTION — asumsi yang dipakai dokumen ini.
- **[PR]** PROPOSAL — usulan dokumen ini, belum diputuskan.
- **[OQ]** OPEN QUESTION — pertanyaan terbuka, belum ada jawaban.
- **[OD]** OWNER DECISION REQUIRED — butuh keputusan owner.

---

## 1. Ringkasan

Gerbang pembelian lama (PLAN §6) memakai tiga pemicu: (a) Fase 0
lolos, (b) live mendekati 12–16 stream ATAU 6 stream key terisi +
antrean siap-live ≥10, (c) harga Master terverifikasi + total
≤ $34,67/bln. Pemicu (b) terlalu sempit: jumlah stream bukan
satu-satunya alasan membeli box kedua — dan bukan pula bukti bahwa
box kedua adalah obat yang benar.

**Gerbang yang diperkuat:** pembelian dibenarkan bila LIMA kondisi
terpenuhi:

| # | Kondisi | Penentu |
|---|---|---|
| 1 | Fase 0 lolos gerbang (PLAN §6, tidak berubah) | Scorecard §2: baris 1, 2, 5, 6, 7, 8, 9, 11 membaik/tetap 72 jam; 0 error class baru |
| 2 | **Demand ATAU operational risk** terpenuhi (§2, §3) | Salah satu dimensi lolos — bukan keduanya |
| 3 | **Verified cost** — V2-009 lolos (§4) | Harga panel terverifikasi + storage/backup dihitung + total ≤ cap |
| 4 | Rencana rollback ada | `VPS8_ROLLBACK_RUNBOOK.md` + `VPS8_ACCEPTANCE_TEST_PLAN.md` |
| 5 | Persetujuan owner | Eksplisit, per pembelian (Level C) |

**Dua aturan asimetri (mengikat):**

- **Stream tinggi TIDAK OTOMATIS membenarkan pembelian.** Bila
  bottleneck sesungguhnya ada di aplikasi/database (lock SQLite,
  churn, gate, bot-check, bug encoder), box baru tidak
  memperbaikinya — yang dibutuhkan adalah perbaikan Fase 0/kode.
  Diagnosis atribusi (§7) wajib sebelum keputusan.
- **Stream rendah TIDAK OTOMATIS menolak pembelian.** Bila Master
  sudah gagal memenuhi SLO pada 2 stream karena ingestion dan
  publish saling mengganggu (CPU/RAM/TX jenuh), VPS 8 dapat
  dibenarkan atas dasar risiko operasional — bukan atas dasar
  jumlah stream.

**Cap biaya tetap mengikat:** total TCO (infrastruktur + storage +
backup + domain + layanan) ≤ **$34,67/bln** [OD — OWNER DECISION;
cakupan DIPUTUSKAN 22 Nov 13:13 WIB: cap mencakup seluruh TCO].
Konsekuensi: infrastruktur saja sudah $34,67 → storage/backup
harus $0 atau infrastruktur direvisi — lihat §4.3.

---

## 2. Dimensi A — Demand

Dimensi ini lolos bila SALAH SATU pemicu terpenuhi.

### 2.1 Pemicu stream (dipertahankan dari PLAN §6)

| Pemicu | Nilai | Status hari ini |
|---|---|---|
| D-1: live mendekati 12–16 stream | Stream aktif ≥12 ATAU proyeksi mencapai 12 dalam 30 hari (berdasarkan laju penambahan stream) | TIDAK — 2 stream aktif [VF] |
| D-2: 6 stream key terisi + antrean siap-live ≥10 | Semua 6 key terisi DAN antrean channel siap-live ≥10 | TIDAK — 2 stream, 6 key belum terisi [VF — PLAN §6] |

### 2.2 Pemicu backlog (baru)

| Pemicu | Threshold [PR] | Sumber data | Cara ukur |
|---|---|---|---|
| D-3: backlog ingestion membesar | Item outstanding naik selama jendela 7 hari (akhir > awal) SEMENTARA throughput ingestion tidak naik | `ingestion_items` / hitungan Drive (kolom H) | Hitung backlog awal vs akhir jendela; bandingkan dengan COMPLETED/7 hari (baseline 123/7 hari [VF]) |
| D-4: antrean Stage 2 jenuh | QUEUED > 100 bertahan ≥3 dari 7 hari | Antrean Stage 2 | Baseline 50 QUEUED [VF — PLAN §1.4]; >100 = 2× baseline |

**Syarat atribusi untuk D-3/D-4:** pemicu backlog hanya dihitung
bila throughput dibatasi SUMBER DAYA Master (CPU/RAM/TX jenuh —
lihat §3), bukan oleh gate/bot-check/churn. Bila backlog membesar
karena bot-check yt-dlp (D1) atau 5 cap gate (V2-003), yang
dibutuhkan adalah perbaikan gate/D1 — bukan VPS 8 (§7).

---

## 3. Dimensi B — Operational risk

Dimensi ini lolos bila **≥2 sinyal risiko breaching sustained**
[PR — aturan "≥2" mencegah satu noise memicu pembelian], ATAU
**publish delay (B-6) breaching + 1 sinyal lain** [PR — publish
delay adalah SLO yang paling dekat dengan pendapatan].

"Sustained" = breaching pada ≥3 dari 7 hari jendela pengukuran
(§6), kecuali dinyatakan lain.

### 3.1 Tabel sinyal

| ID | Sinyal | Threshold breach | Baseline terverifikasi | Sumber data | Cara ukur |
|---|---|---|---|---|---|
| B-1 | CPU pressure Master | load15 ≥ 4,0 selama ≥1 jam/hari [PR] | Load 3,5–4,2 [VF — PLAN §2] | `/proc/loadavg`, tick governor | Seri load15 harian; hitung jam ≥4,0 |
| B-2 | RAM available Master | < 4 GB bertahan (gate ingestion RAM trip) [PR] | ±3,6 GB bebas, sekali ukur 22 Nov 11:33 [DF]; gate ingestion: ≥4 GB tersedia [DF — PLAN §4] | `free`/`/proc/meminfo`, log gate ingestion | Seri RAM bebas harian; hitung hari gate trip |
| B-3 | Disk I/O wait | %iowait > 10% bertahan [PR, belum tervalidasi] | TIDAK ADA baseline [OQ-301] | `iostat`/`vmstat` | Sinyal NONAKTIF sampai V2-005/V2-008 menetapkan baseline |
| B-4 | Network egress Master | TX p95 > 350 Mbps (70% port) selama jendela [PR] | Spike 60 Mbps dengan 2 stream [VF — PLAN §1.2]; port 500 Mbps terukur 91–93% (uji kapasitas, bukan beban produksi) [VF] | Sampler TX governor (EMA 300 dtk) | p95 TX harian selama 7 hari |
| B-5 | Job backlog | Sama dengan D-3/D-4 (§2.2) | Backlog 412 channel / ±28.461 item [VF — sensus 20 Nov]; Stage 2 50 QUEUED [VF] | `ingestion_items`, antrean Stage 2 | Tren 7 hari |
| B-6 | Publish delay | p90 > 30 menit selama jendela 7 hari | Median 11,6 mnt, p90 30,2 mnt [VF — PLAN §1.7]; target Fase 0 ≤30 mnt [DF — PLAN §2] | Timestamp publish di DB (`operational_tasks`) | p90 harian; breach = p90 > 30 mnt pada ≥3 dari 7 hari |
| B-7 | SQLite lock | > 31×/24 jam pada ≥3 dari 7 hari (lebih buruk dari baseline) | 31×/24 jam [VF — PLAN §2]; target Fase 0 <20× [DF] | Log aplikasi (`database is locked`) | Hitungan harian |
| B-8 | Live stability | Admission rejection > 6×/24 jam bertahan, ATAU 1 penolakan menunda restart live > 60 dtk | 6 penolakan/24 jam (legacy) [VF — V2-003]; 5 penolakan palsu menunda restart 114 selama 93 dtk [VF — V2-002] | Log `live_supervisor` (admission), tick guard | Hitungan harian + durasi tunda restart |
| B-9 | (Dicadangkan) Encoder death | **BUKAN sinyal pembelian** — lihat §3.2 | 12×/17 jam, slot 114 [VF — V2-002; PLAN §2 baris 3 "5×/24 jam slot 706" SALAH, V2-005 merekonsiliasi] | — | — |

### 3.2 Catatan per sinyal

**B-1 (load).** Baseline load 3,5–4,2 sudah menyentuh saturasi
4 vCPU — sinyal ini lemah bila berdiri sendiri (Master "sibuk"
adalah kondisi normal, bukan bukti gagal SLO). B-1 hanya bermakna
bila breaching BERSAMA B-6 (publish delay) atau B-5 (backlog):
load tinggi + SLO memburuk = kapasitas tidak cukup; load tinggi +
SLO baik = box bekerja efisien.

**B-2 (RAM).** Satu-satunya ukuran RAM bebas (±3,6 GB) adalah
sekali ukur — belum ada seri harian [OQ-302]. Ambang 4 GB memakai
gate ingestion yang sudah ada di aplikasi (bukan angka baru):
bila gate RAM trip berulang, ingestion dibatasi RAM Master secara
nyata. V2-005 harus membangun seri RAM bebas harian sebelum sinyal
ini dapat diandalkan.

**B-3 (iowait).** Tidak ada baseline — sinyal ini DINONAKTIFKAN
sampai V2-005/V2-008 mengukurnya. Threshold 10% adalah [PR] tanpa
bukti dan TIDAK BOLEH dipakai untuk keputusan sebelum baseline ada.
Mencantumkannya di sini agar tidak terlupakan, bukan agar dipakai.

**B-4 (egress).** Ambang 350 Mbps = 70% port — sama dengan cap
admission yang sudah ada di PLAN §5.2 (bukan angka baru). Catatan:
pengukuran port 91–93% adalah uji kapasitas port, BUKAN beban
produksi; beban produksi dengan 2 stream hanya spike ke 60 Mbps.
Jangan mencampur keduanya.

**B-6 (publish delay).** Sinyal terkuat: publish adalah fungsi
inti dan p90 adalah SLO scorecard. Baseline p90 30,2 mnt sudah
sedikit di atas target Fase 0 (≤30 mnt) — artinya sinyal ini
hampir breaching hari ini. Bila p90 memburuk sambil load jenuh,
itu adalah bukti kapasitas yang sesungguhnya.

**B-7 (lock SQLite).** Nuansa penting: lock SQLite TIDAK DIPERBAIKI
oleh VPS 8 — database tetap di Master (V2.0 §4). Lock yang
disebabkan tulis publish/live tetap ada setelah pembelian. Lock
yang disebabkan tulis prep worker MUNGKIN berkurang bila prep
pindah (tulis lewat API, bukan DB langsung). Karena itu B-7
dihitung sebagai sinyal risiko HANYA bila atribusi (§7)
menunjukkan porsi signifikan lock berasal dari beban prep — jika
tidak, B-7 adalah argumen untuk V2-007 (audit schema), bukan
untuk pembelian.

**B-8 (live stability).** Yang diukur adalah admission rejection
dan penundaan restart — yaitu gejala TEKANAN SUMBER DAYA pada
jalur live. Kematian encoder (B-9) sengaja DIKECUALIKAN: 12
kematian slot 114 adalah masalah reliabilitas kode/RTMP (P16,
R3–R8), bukan masalah kapasitas — box baru tidak memperbaikinya
[VF — V2-002]. Menghitung kematian encoder sebagai alasan membeli
VPS 8 adalah kesalahan atribusi yang sama jenisnya dengan
menghitung "ingestion 0,0%" sebagai bukti ingestion tidak butuh CPU.

---

## 4. Dimensi C — Verified cost

Dimensi ini lolos bila **V2-009 lolos** — seluruh daftar di bawah
tercentang. V2-009 adalah gerbang biaya (STATUS_REGISTRY:
NOT_STARTED).

### 4.1 Daftar verifikasi V2-009

| # | Item | Status hari ini | Cara verifikasi |
|---|---|---|---|
| C-1 | Harga Master terverifikasi di panel Contabo | BELUM — ±$16,02/bln (±$14,43 pra-pajak) adalah ESTIMASI [DF — cost register] | User memeriksa panel Contabo (Turnstile memblokir browser cloud — butuh user) |
| C-2 | Harga VPS 8 terverifikasi di panel/listing | BELUM — $18,65/bln + $2,66 setup adalah DOCUMENTED NOT YET REVALIDATED [DF — cost register] | Panel Contabo / halaman pricing |
| C-3 | Biaya storage/backup off-site dihitung | BELUM DIHITUNG [VF — cost register] | Pilih provider + retensi (STORAGE MENUNGGU — OWNER DECISION), hitung bulanan |
| C-4 | Total TCO (infra + storage + backup + domain + layanan) ≤ $34,67/bln | BELUM DAPAT DINILAI — harga belum terverifikasi + storage BELUM DIHITUNG | Hitung ulang setelah C-1 + C-2 + C-3 |
| C-5 | Total ALL-IN (termasuk storage/backup) diketahui + diterima owner | BELUM — lihat §4.3 | Owner memutuskan setelah C-3 |

### 4.2 Cap $34,67/bln (mengikat)

Cap total TCO (infrastruktur + storage + backup + domain +
layanan) = **$34,67/bln** [OD — OWNER DECISION, ditetapkan user
21–22 Nov 2026; cakupan DIPUTUSKAN 22 Nov 13:13 WIB: cap mencakup
seluruh TCO]. Bila total riil > $34,67: eskalasi ke owner, jangan
lanjut diam-diam [DF — PLAN §7]. Kontrak 12 bulan (total
$31,87/bln, bayar di muka $190,21) adalah opsi hemat [DF —
ESTIMASI] — diputuskan SETELAH model terbukti (bulanan dulu);
bukan bagian gerbang ini.

### 4.3 Storage/backup tercakup cap (DIPUTUSKAN 22 Nov 13:13)

Cap $34,67/bln mencakup SELURUH TCO. Biaya object storage
off-site (Restic) + Litestream masih BELUM DIHITUNG [VF — cost
register §2]. Karena infrastruktur saja sudah $34,67,
storage/backup harus $0 (mis. free tier object storage untuk
DB 118 MB) atau infrastruktur direvisi (opsi hemat: kontrak
12 bulan → $31,87/bln, membebaskan ±$2,80/bln). Bila total
all-in > $34,67/bln, pembelian TIDAK lolos gerbang — eskalasi
ke owner, jangan lanjut diam-diam.

---

## 5. Aturan keputusan

### 5.1 Logika gerbang

```
BELI  ⟺  Fase0_lolos
         AND (Demand_lolos OR Risk_lolos)
         AND Cost_terverifikasi (V2-009)
         AND Rollback_ada
         AND Persetujuan_owner

Demand_lolos ⟺ D-1 OR D-2 OR (D-3/D-4 dengan atribusi sumber daya)
Risk_lolos   ⟺ (≥2 sinyal §3.1 breaching sustained)
              OR (B-6 breaching + ≥1 sinyal lain)
```

**Yang tidak berubah dari PLAN §6:** Fase 0 harus lolos dulu;
rollback plan harus ada; owner menyetujui per pembelian. Dokumen
ini hanya memperkuat pemicu (b) — tidak melemahkan pemicu lain.

### 5.2 Contoh keputusan (ilustratif — skenario hipotetis [AS])

**Contoh A — 2 stream, beli DIBENARKAN (risiko).**
Stream aktif 2; p90 publish 45 menit selama 7 hari (B-6 breach);
load15 ≥4,0 selama 2 jam/hari (B-1 breach); TX hanya 60 Mbps.
Demand gagal (D-1/D-2 tidak terpenuhi), tetapi Risk lolos
(B-6 + B-1). Atribusi: ingestion + publish berebut CPU Master
(load jenuh + SLO memburuk). Bila V2-009 lolos + rollback ada +
owner setuju → **BELI**. Alasan pembelian adalah kapasitas,
bukan jumlah stream.

**Contoh B — 14 stream, beli DITOLAK (bottleneck aplikasi).**
Stream aktif 14 (D-1 terpenuhi); tetapi load 2,5, p90 publish
22 menit, TX 200 Mbps — Master sehat. Sementara lock SQLite
45×/24 jam (B-7 breach) + churn ingestion 60%. Atribusi: lock
berasal dari tulis publish/live (DB tetap di Master) dan churn
dari logika antrean — keduanya tidak diperbaiki VPS 8. Keputusan:
**JANGAN BELI** — kerjakan V2-007 (audit schema) + V2-003-R7
(reformasi konkurensi ingestion) dulu. Membeli box untuk masalah
database adalah pemborosan yang tampak seperti kemajuan.

**Contoh C — 14 stream, beli DIBENARKAN (demand + risiko).**
Stream aktif 14 (D-1 terpenuhi); TX p95 380 Mbps (B-4 breach);
load15 ≥4,0 (B-1 breach); backlog ingestion membesar sementara
throughput tidak naik (D-3 + atribusi sumber daya). Demand dan
Risk sama-sama lolos. Bila V2-009 lolos + rollback ada + owner
setuju → **BELI**.

### 5.3 Aturan tambahan

- **Satu sinyal tidak cukup** (kecuali B-6 + 1 lain, §3): satu
  metrik yang breaching bisa jadi noise, kesalahan ukur, atau
  insiden sekali (mis. backup malam menaikkan load). Aturan "≥2"
  adalah [PR] — V2-005 dapat merevisinya dengan bukti.
- **Gerbang dievaluasi ulang tiap siklus:** kondisi yang lolos
  minggu lalu belum tentu lolos minggu ini (mis. Fase 0
  memperbaiki publish delay sehingga B-6 pulih). Keputusan
  pembelian memakai jendela pengukuran TERBARU (§6), bukan
  jendela lama.
- **Tidak ada pembelian darurat tanpa gerbang:** bila Master
  gagal total (bukan sekadar lambat), itu adalah insiden DR
  (pakai prosedur `MASJAVAS_BACKUP_RECOVERY_PLAN.md`), bukan
  alasan melewati gerbang pembelian.

---

## 6. Jendela pengukuran + paket bukti

### 6.1 Jendela

- **Jendela pengukuran: 7 hari kalender** [PR — selaras dengan
  evaluasi scorecard 7 hari di checklist P10]. "Sustained" =
  breaching pada ≥3 dari 7 hari, atau p95 selama jendela melewati
  threshold.
- **Sumber data harus seri waktu, bukan sekali ukur.** Satu
  snapshot (mis. RAM bebas sekali ukur 22 Nov) tidak cukup untuk
  menyatakan breach. Bila seri tidak ada, sinyal dinyatakan
  TIDAK DAPAT DINILAI (bukan "lolos" atau "gagal").
- **V2-005 (baseline snapshot) adalah prasyarat pengukuran yang
  dapat dipercaya** — termasuk rekonsiliasi baseline kematian
  encoder (5×/24 jam di scorecard vs 12×/17 jam V2-002) dan
  konfirmasi sumber median delay 11,6 menit [DF —
  STATUS_REGISTRY V2-005].

### 6.2 Paket bukti keputusan (checklist)

Sebelum owner diminta memutuskan, paket berikut harus lengkap:

1. Seri 7 hari untuk 9 sinyal §3.1 (atau "TIDAK DAPAT DINILAI"
   dengan alasan per sinyal).
2. Catatan atribusi §7 (bottleneck sumber daya vs aplikasi).
3. Status Fase 0 (scorecard 72 jam).
4. Hasil V2-009 (C-1–C-5) + total all-in.
5. Rujukan rollback plan + acceptance plan.
6. Rekomendasi BELI / JANGAN BELI dengan alasan per dimensi.

Tanpa paket bukti, tidak ada keputusan pembelian — "rasanya
Master sudah berat" bukan bukti.

---

## 7. Diagnosis bottleneck — yang TIDAK membenarkan pembelian

Sebelum sinyal risiko dihitung, jawab checklist atribusi ini.
Bila jawaban mengarah ke aplikasi/DB, perbaikannya adalah
Fase 0/kode — bukan VPS 8.

| Pertanyaan diagnosis | Bila YA → | Rujukan |
|---|---|---|
| Apakah lock SQLite berasal dari tulis publish/live (bukan prep)? | V2-007 audit schema; VPS 8 tidak membantu | §3.2 B-7 |
| Apakah backlog membesar karena bot-check yt-dlp (D1) atau gate ingestion? | Jalur D1 / V2-003-R7; VPS 8 tidak membantu | PLAN §9 (D1); V2-003 R7 |
| Apakah churn SUPERSEDED masih >20%? | V2-003-R7 + registry 1-owner (Fase 0 langkah 0.3) | PLAN §6 0.3 |
| Apakah live tidak stabil karena bug encoder/RTMP (P16)? | Perbaikan R3–R8 + observasi; VPS 8 tidak membantu | V2-002 |
| Apakah publish delay disebabkan kuota YouTube (quotaExceeded)? | Doktrin kuota (5 ch/vault); VPS 8 tidak membantu | Cost register / dashboard |
| Apakah throughput Stage 2 dibatasi I/O Drive, bukan CPU? | Slot/I/O — VPS 8 membantu HANYA bila port/Drive-nya yang jenuh di Master | Capacity model §3.2 |
| Apakah Master load tinggi tetapi SLO baik (p90 ≤30 mnt, 0 FAILED)? | Box bekerja efisien — bukan bukti kapasitas kurang | §3.2 B-1 |

**Prinsip (dari review, disetujui sprint):** jangan tambah VPS 8
untuk mengatasi semua bottleneck. Beberapa masalah saat ini berada
pada kode, guard, atribusi, dan job lifecycle. Infrastruktur baru
tidak otomatis memperbaikinya.

---

## 8. Pertanyaan terbuka

| ID | Pertanyaan | Blokir |
|---|---|---|
| OQ-301 | Berapa baseline %iowait Master? (Sinyal B-3 nonaktif sampai baseline ada.) | Sinyal B-3 |
| OQ-302 | Berapa seri RAM bebas harian Master? (Baru ada sekali ukur 22 Nov 11:33.) | Sinyal B-2 |
| OQ-303 | Berapa porsi lock SQLite yang berasal dari tulis prep vs publish/live? (Menentukan apakah B-7 relevan dengan pembelian.) | Sinyal B-7 |
| OQ-304 | Apakah aturan "≥2 sinyal" (§5.1) terlalu ketat/longgar? V2-005 dapat merevisinya dengan bukti 7 hari. | Logika gerbang |
| OQ-305 | Berapa laju pertumbuhan stream yang realistis (untuk proyeksi D-1 "mencapai 12 dalam 30 hari")? Belum ada data laju penambahan stream. | Pemicu D-1 |
| OQ-306 | Berapa biaya storage/backup all-in? (V2-009 / sprint item P1 #7 — TCO model.) | Dimensi C |

---

## 9. Kaitan dengan dokumen lain

- `PLAN_PENAMBAHAN_VPS8_2026-11-22.md` §6 (gerbang asli — pemicu
  (b) diperkuat dokumen ini), §2 (scorecard — sumber threshold
  B-6/B-7), §7 (biaya), §10 (keputusan pemicu pembelian #3).
- `STATUS_REGISTRY.yaml` — `gates.purchase_gate` merujuk dokumen
  ini; V2-009 NOT_STARTED; D4 (pembelian) MENUNGGU gerbang.
- `MASJAVAS_COST_REGISTER.md` — label harga (ESTIMASI vs
  TERVERIFIKASI vs BELUM DIHITUNG); §2 storage BELUM DIHITUNG.
- `VPS8_RESOURCE_CAPACITY_MODEL.md` — headroom VPS 8 ±1 core:
  pembelian memindahkan beban ke box yang juga ketat; kapasitas
  VPS 8 bukan alasan membeli lebih awal, melainkan risiko yang
  dicatat dalam keputusan.
- `VPS8_DEPLOYMENT_RUNBOOK.md` + `VPS8_ACCEPTANCE_TEST_PLAN.md` +
  `VPS8_ROLLBACK_RUNBOOK.md` — eksekusi pasca-pembelian (kondisi
  #4 gerbang).
- `REPOSITORY_ENGINEERING_AUDIT_SPRINT.md` — item P0 #3 (F4);
  kriteria selesai item ini: gerbang terukur dengan threshold +
  sumber data per sinyal; cap $34,67/bln tetap mengikat.
