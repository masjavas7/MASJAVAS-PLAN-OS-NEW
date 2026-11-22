# SLO & ERROR BUDGET — MASJAVAS TUBE OS V2

**Versi:** 1.0 — 22 Nov 2026, 13:05 WIB (sprint item #8).
**Sumber:** `PLAN_PENAMBAHAN_VPS8_2026-11-22.md` §1.7 + §2
(scorecard), `VPS8_ACCEPTANCE_TEST_PLAN.md` §1 (baseline),
`VPS8_ROLLBACK_RUNBOOK.md` §2 (RPO/RTO), `STATUS_REGISTRY.yaml`.
**Aturan dokumen ini:** target angka HANYA dari baseline yang
dapat dipercaya (scorecard + baseline solid). SLO tanpa baseline
= **OPEN QUESTION**, bukan target karangan. Target yang ditandai
[PR] adalah PROPOSAL sampai baseline-nya diukur (V2-005).

## Legenda label

- **[VF]** VERIFIED FACT — terverifikasi langsung 21–22 Nov 2026.
- **[DF]** DOCUMENTED FACT NOT YET REVALIDATED.
- **[PR]** PROPOSAL — usulan dokumen ini, belum diputuskan.
- **[OQ]** OPEN QUESTION — belum ada jawaban / belum ada baseline.
- **[OD]** OWNER DECISION REQUIRED.
- **TARGET (belum terbukti)** — target yang ditetapkan dokumen
  arsitektur tetapi belum pernah dibuktikan dengan drill/uji.

**Prinsip yang mengikat (user, 22 Nov 12:40):** baseline solid
adalah LANTAI — SLO tidak boleh menetapkan target di bawah
baseline yang sudah dicapai, dan pelanggaran SLO dinilai dengan
klasifikasi Rollback Runbook §1.1 (hard safety breach / SLO
degradation / transient noise), bukan dengan rollback otomatis.

## 1. Ringkasan SLO

| # | SLO | Target | Status target |
|---|---|---|---|
| S1 | Live availability | OPEN QUESTION (tanpa baseline %) | [OQ] |
| S2 | Publish success rate | 0 FAILED/BLOCKED per hari | [VF] baseline 21 Nov: 206/206 |
| S3 | Job completion rate | OPEN QUESTION (tanpa baseline rate) | [OQ] |
| S4 | Worker availability | OPEN QUESTION (worker remote belum ada) | [OQ] |
| S5 | API latency (healthz p95) | p95 <100 ms | [VF] baseline ±10 ms |
| S6 | Job claim latency | OPEN QUESTION (belum pernah diukur) | [OQ] |
| S7 | Recovery time (RTO) | ≤30 mnt (TARGET belum terbukti) | [DF] target V2.0 — belum drill |
| S8 | Backup freshness (RPO) | ≤5 mnt (TARGET belum terbukti) — BELUM BERLAKU (sistem backup belum ada) | [DF] target V2.0 — belum drill |

**Kejujuran yang mengikat:** 4 dari 8 SLO belum punya target angka
karena baseline-nya tidak ada. Itu BUKAN kekurangan dokumen ini —
itu fakta sistem hari ini. Menetapkan target tanpa baseline adalah
mengarang angka (dilarang aturan sprint #5).

## 2. Definisi per SLO

### S1 — Live availability

- **SLI (indikator):** persentase waktu siaran terjadwal di mana
  broadcast BENAR-BENAR live di sisi YouTube (bukan hanya proses
  FFmpeg hidup — bedakan keduanya, T1.3-B).
- **Target:** **[OQ] OPEN QUESTION.** Tidak ada pengukuran
  availability % yang pernah dilakukan. Fakta terkait [VF]:
  2 stream reguler (114, 706); encoder 114 mati 12×/17 jam
  (21–22 Nov, pra-perbaikan) dengan pemulihan otomatis; orphan
  706 mengudara tanpa audio ±17 jam tanpa terdeteksi (D11).
  Fakta-fakta ini membuktikan availability TIDAK 100%, tetapi
  tidak cukup untuk menghitung persentasenya.
- **Window pengukuran:** 30 hari kalender [PR] — setelah
  pengukuran dimulai.
- **Cara menghitung:** (menit broadcast terverifikasi live) ÷
  (menit siaran terjadwal) × 100%, per channel lalu agregat
  fleet. Verifikasi dari sisi YouTube (metode otomatis vs manual
  masih OQ-202).
- **Sumber data:** cek status broadcast YouTube + log supervisor
  (`live_<task>.log`) + journal `masjavas-live`.
- **Alert threshold:** **[OQ]** — ditetapkan setelah baseline 30
  hari pertama diukur (V2-005). Sementara itu, alert yang berlaku
  adalah pemicu Rollback Runbook §3a (live down >5 mnt).
- **Pemilik keputusan:** Owner (keputusan rollback/hold);
  operator bertugas (respons pertama + eskalasi).

### S2 — Publish success rate

- **SLI (indikator):** PUBLISHED ÷ (PUBLISHED + FAILED + BLOCKED)
  per hari. SKIPPED (mis. skip audio) TIDAK masuk penyebut —
  skip adalah masalah pasokan bahan (ingestion), bukan kegagalan
  publish; ia dilacak terpisah sebagai schedule fulfilment (§2b).
- **Target:** **0 FAILED/BLOCKED per hari** (SLI 100% pada hari
  tanpa kegagalan) [VF — scorecard PLAN §2 baris 1: "≥200,
  0 FAILED"]. Baseline 21 Nov: 206 PUBLISHED, 0 FAILED/BLOCKED
  → SLI 100% hari itu.
- **Window pengukuran:** harian (1 hari kalender WIB); agregat
  7 hari untuk tren.
- **Cara menghitung:** hitung dari tabel `operational_tasks`
  (status PUBLISHED/FAILED/BLOCKED per `target_date_wib`).
- **Sumber data:** database Master (`operational_tasks`) +
  dashboard publish harian.
- **Alert threshold:** ≥1 FAILED/BLOCKED dalam 24 jam = SLO
  degradation (HOLD + investigasi — Rollback Runbook §3.1
  baris 1). 0 PUBLISHED dalam 2 jam pada jam aktif = hard
  safety breach (publish berhenti total).
- **Pemilik keputusan:** Owner (keputusan rollback/hold);
  operator bertugas (respons pertama + eskalasi).

#### S2b — Schedule fulfilment (metrik pelengkap, bukan SLO utama)

- **SLI:** PUBLISHED ÷ tugas terjadwal per hari. Baseline
  21 Nov: 206/237 = **86,9%** (31 skip audio) [VF — PLAN §1.7].
- **Target:** **[OQ] OPEN QUESTION** — target perbaikan
  (mengurangi skip audio) bergantung pada pemulihan ingestion
  (D1 ditunda user; backlog 28.461 item). Bukan target yang
  bisa ditetapkan dari baseline hari ini.

### S3 — Job completion rate

- **SLI (indikator):** job COMPLETED ÷ (COMPLETED + FAILED) per
  7 hari, per jenis job (ingestion, Stage 2). SUPERSEDED tidak
  masuk penyebut — ia adalah churn (diukur terpisah, scorecard
  baris 9: baseline 66%, target <20%).
- **Target:** **[OQ] OPEN QUESTION.** Tidak ada baseline rate
  yang terukur. Angka throughput yang ada [VF] BUKAN rate:
  ingestion 123 COMPLETED/7 hari; Stage 2 1.011 DONE/7 hari
  (park rate 14%); churn SUPERSEDED 66%. Jumlah FAILED per
  jenis job tidak tercatat dalam baseline — tanpa penyebut
  FAILED, rate tidak dapat dihitung.
- **Window pengukuran:** 7 hari kalender [PR] — setelah
  pencatatan FAILED per jenis job tersedia.
- **Cara menghitung:** dari antrean job (`operational_tasks` /
  antrean Stage 2): COMPLETED ÷ (COMPLETED + FAILED) per jenis
  job per 7 hari.
- **Sumber data:** database Master + log antrean Stage 2.
- **Alert threshold:** **[OQ]** — ditetapkan setelah baseline
  7 hari pertama diukur (V2-005). Sementara itu, yang berlaku
  adalah ambang scorecard baris 8–9 (Rollback Runbook §3.1).
- **Pemilik keputusan:** Owner (keputusan rollback/hold);
  operator bertugas (respons pertama + eskalasi).

### S4 — Worker availability

- **SLI (indikator):** persentase waktu worker terdaftar
  responsif — heartbeat diterima dalam jendela lease yang
  berlaku (heartbeat stale >90 dtk = tidak responsif;
  Rollback Runbook §3c).
- **Target:** **[OQ] OPEN QUESTION.** Worker remote (VPS 8)
  belum ada; tidak ada baseline. Worker Master adalah proses
  lokal (bukan worker remote) — availability-nya tidak
  diukur dengan SLI ini.
- **Window pengukuran:** 30 hari kalender [PR] — setelah
  Fase 1.2 (worker terdaftar).
- **Cara menghitung:** (menit worker responsif) ÷ (menit worker
  terdaftar) × 100%, per worker.
- **Sumber data:** log heartbeat Master (`workers/heartbeat` —
  lihat ORCHESTRATOR_API_CONTRACT_V1.md bila sudah dibuat;
  sprint item #4).
- **Alert threshold:** heartbeat stale >90 dtk = alarm
  (Rollback Runbook §3c); target alert lanjutan [OQ] sampai
  baseline 30 hari tersedia.
- **Pemilik keputusan:** Owner (keputusan rollback/hold);
  operator bertugas (respons pertama + eskalasi).

### S5 — API latency

- **SLI (indikator):** p95 latensi endpoint healthz Master
  (dan, setelah dibangun, endpoint orchestrator API).
- **Target:** **p95 <100 ms** per 24 jam [VF — scorecard PLAN §2
  baris 11; baseline ±10 ms]. Target ini LEBIH LONGGAR dari
  baseline (±10 ms) — sengaja, karena target scorecard memberi
  ruang untuk beban Fase 1. Bila p95 naik dari 10 ms ke 90 ms,
  target masih terpenuhi tetapi tren memburuk harus dicatat
  (observasi, bukan pelanggaran).
- **Window pengukuran:** 24 jam (rata-rata harian p95).
- **Cara menghitung:** p95 dari semua pengukuran healthz dalam
  24 jam (sampel: cek berkala; frekuensi cek belum ditetapkan —
  OQ-SLO-1).
- **Sumber data:** log healthz Master + (nanti) log akses API
  orchestrator.
- **Alert threshold:** p95 >50 ms selama 1 jam = peringatan dini
  [PR]; p95 >100 ms bertahan >1 jam (≥4 cek tiap 15 mnt) atau
  rata-rata harian >100 ms selama 2 hari = SLO degradation
  (Rollback Runbook §3.1 baris 11).
- **Pemilik keputusan:** Owner (keputusan rollback/hold);
  operator bertugas (respons pertama + eskalasi).

### S6 — Job claim latency

- **SLI (indikator):** waktu dari job QUEUED (siap dikerjakan)
  sampai diklaim worker — p95 per 24 jam.
- **Target:** **[OQ] OPEN QUESTION.** Belum pernah diukur.
  Klaim menjadi bottleneck adalah salah satu trigger evaluasi
  PostgreSQL (V2.0) — tetapi tanpa pengukuran, tidak ada
  baseline dan tidak ada target.
- **Window pengukuran:** 24 jam [PR] — setelah pengukuran dimulai.
- **Cara menghitung:** p95 dari (waktu klaim − `schedule_time_epoch`)
  untuk job yang diklaim dalam 24 jam; dari
  `operational_tasks` (`schedule_time_epoch` + timestamp klaim).
- **Sumber data:** database Master (`operational_tasks`).
- **Alert threshold:** **[OQ]** — ditetapkan setelah baseline
  7 hari pertama diukur (V2-005/V2-007). Sementara itu, yang
  berlaku adalah ambang scorecard baris 8 (0 RUNNING + backlog
  >6 jam = SLO degradation).
- **Pemilik keputusan:** Owner (keputusan rollback/hold);
  operator bertugas (respons pertama + eskalasi).

### S7 — Recovery time (RTO)

- **SLI (indikator):** waktu dari kegagalan terdeteksi sampai
  layanan pulih, per kelas insiden (encoder mati, worker crash,
  Master API mati, Tailscale putus, disk penuh).
- **Target:** **≤30 menit** per insiden [DF — TARGET V2.0 /
  Rollback Runbook §2]. **TARGET ini BELUM TERBUKTI** — belum
  pernah ada restore drill atau uji pemulihan yang mengukurnya.
  Bukti parsial [VF]: restart manual 706 memulihkan audio dalam
  ±48 dtk (22 Nov 08:10); reconnect encoder ditargetkan ≤2 mnt
  (T1.3-C, [PR] — belum diuji). Bukti parsial ini TIDAK
  membuktikan RTO ≤30 mnt untuk semua kelas insiden.
- **Window pengukuran:** per insiden; agregat 30 hari (p95 waktu
  pemulihan) [PR].
- **Cara menghitung:** (waktu layanan pulih terverifikasi) −
  (waktu kegagalan terdeteksi), dari log insiden + journal.
  "Pulih" = verifikasi dari sisi pengguna/broadcast, bukan
  hanya proses hidup.
- **Sumber data:** log insiden (Rollback Runbook §9) + journal
  systemd + log supervisor.
- **Alert threshold:** pemulihan >15 mnt (setengah RTO) pada
  insiden yang sedang berjalan = eskalasi [PR]; pemulihan
  >30 mnt = SLO dilanggar (catat + investigasi akar masalah).
- **Pemilik keputusan:** Owner (keputusan rollback/hold +
  otoritas drill); operator bertugas (eksekusi pemulihan +
  eskalasi).

### S8 — Backup freshness (RPO)

- **SLI (indikator):** umur salinan backup terbaru yang
  TERVERIFIKASI dapat direstore (bukan hanya umur file backup —
  salinan yang belum diuji restore tidak dihitung segar).
- **Target:** **≤5 menit** [DF — TARGET V2.0 / Rollback Runbook
  §2]. **TARGET ini BELUM TERBUKTI dan SLO ini BELUM BERLAKU:**
  belum ada sistem backup off-site (Restic) maupun replikasi
  (Litestream) — keduanya tergate V2-009; DR replica adalah
  Fase 1.4 (belum dibangun). SLO ini aktif setelah restore
  drill #1 dijalankan dan RPO yang tercapai dicatat apa adanya.
- **Window pengukuran:** berkelanjutan (umur backup terbaru
  setiap saat) [PR] — setelah sistem backup ada.
- **Cara menghitung:** (sekarang) − (timestamp salinan terbaru
  yang lolos uji restore terakhir). Uji restore: drill bulanan
  (Backup Recovery Plan).
- **Sumber data:** log backup Restic/Litestream + log restore
  drill.
- **Alert threshold:** umur backup terbaru >5 mnt (setelah
  sistem ada) = alarm [PR]; backup gagal 2× berturut-turut =
  SLO degradation. Sampai sistem ada: TIDAK ADA alert —
  ketiadaan backup dicatat sebagai risiko (R-register), bukan
  sebagai pelanggaran SLO.
- **Pemilik keputusan:** Owner (keputusan provider/retensi +
  otoritas drill); operator bertugas (eksekusi backup/drill).

## 3. Error budget

Error budget = 1 − target SLO, dalam window pengukurannya.
Budget berlaku HANYA untuk SLO yang punya target angka (S2, S5,
S7). SLO dengan target OPEN QUESTION tidak punya budget sampai
targetnya ditetapkan.

| SLO | Budget | Kebijakan bila budget habis [PR] |
|---|---|---|
| S2 Publish success rate | **0** (target 0 FAILED/hari — setiap FAILED menghabiskan budget hari itu) | Setiap FAILED = SLO degradation: HOLD perubahan non-esensial sampai akar masalah teridentifikasi + owner memutuskan. Budget 0 berarti SLO ini adalah aturan keras, bukan target statistik |
| S5 API latency | p95 boleh >100 ms selama <1 jam/hari tanpa melanggar (budget = selisih antara ambang alert 1 jam dan target harian) — definisi kasar [PR]; definisi presisi menunggu data p95 historis | Budget habis (p95 >100 ms >1 jam dalam 24 jam) = SLO degradation: HOLD + investigasi |
| S7 Recovery time | Setiap insiden yang pulih >30 mnt menghabiskan budget bulan itu [PR] (1 pelanggaran = budget habis — RTO adalah target per insiden, bukan statistik) | Budget habis = freeze perubahan non-esensial + review prosedur pemulihan oleh owner |

**Catatan:** budget S2 = 0 adalah konsekuensi jujur dari target
scorecard (0 FAILED). Bila owner menginginkan budget >0 untuk
publish (mis. toleransi 1 FAILED/minggu), target S2 harus diubah
dulu oleh owner — dokumen ini tidak mengubah target scorecard
secara sepihak. [OD]

## 4. Keterkaitan dengan dokumen lain

- **Rollback Runbook §1.1 + §3 + §3.1** — klasifikasi pelanggaran
  SLO (hard safety breach / SLO degradation / transient noise)
  dan ambang per metrik scorecard. Dokumen SLO ini mendefinisikan
  TARGET; runbook mendefinisikan TINDAKAN saat target dilanggar.
- **Acceptance Test Plan §1 + §1.1** — baseline dan klasifikasi
  hasil uji; uji T1.3-C (reconnect ≤2 mnt) dan T1.4-C (restore
  drill #1) adalah bukti yang akan mengisi S7/S8.
- **STATUS_REGISTRY.yaml** — V2-005 (baseline snapshot) adalah
  tiket yang akan mengukur baseline untuk S1/S3/S4/S6.
- **Backup Recovery Plan** — prosedur drill yang membuktikan
  S8 (RPO ≤5 mnt) dan S7 (RTO ≤30 mnt).

## 5. Pertanyaan terbuka

1. **[OQ-SLO-1]** Frekuensi cek healthz untuk S5 (tiap berapa
   menit) — menentukan kualitas sampel p95.
2. **[OQ-SLO-2]** Metode verifikasi broadcast YouTube otomatis
   untuk S1 (sama dengan OQ-202 di Acceptance Test Plan).
3. **[OQ-SLO-3]** Apakah owner menginginkan budget >0 untuk S2
   (toleransi FAILED) — bila ya, target scorecard "0 FAILED"
   perlu diubah dulu oleh owner.
4. **[OQ-SLO-4]** Jadwal pengukuran baseline 30 hari untuk S1/S4
   dan 7 hari untuk S3/S6 — siapa yang menjalankan, dan kapan
   dimulai (terkait V2-005)?
5. **[OQ-SLO-5]** Jadwal restore drill #1 (sama dengan OQ-304) —
   menentukan kapan S7/S8 mulai berlaku sebagai SLO yang
   terukur, bukan TARGET.
