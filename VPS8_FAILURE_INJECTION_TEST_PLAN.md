# RENCANA UJI INJEKSI KEGAGALAN VPS 8 — MASJAVAS TUBE OS V2

**Status:** DRAFT v1.0 — 22 Nov 2026. Rencana ini menguji sistem
saat kondisi BURUK (bukan hanya saat normal). Dokumen ini adalah
rencana; eksekusinya adalah perubahan yang memengaruhi layanan
dan membutuhkan persetujuan owner per skenario (Level C).
**Sumber:** `MASTER_ARCHITECTURE_MASJAVAS_TUBE_OS_V2.md` §18–§21
(FROZEN — V2.0); `VPS8_ACCEPTANCE_TEST_PLAN.md` (baseline §1,
uji T1.4–T1.5, guard G-1–G-8); `VPS8_ROLLBACK_RUNBOOK.md` (§1–§4:
prinsip, target pemulihan, pemicu, langkah darurat);
`ORCHESTRATOR_API_CONTRACT_V1.md` (§7: kasus uji stale-worker);
`REPOSITORY_ENGINEERING_AUDIT_SPRINT.md` item #5 (kriteria
selesai: tiap skenario punya kondisi awal, simulasi, hasil yang
diharapkan, metrik, kriteria lulus/gagal, prosedur pemulihan).
**Kaitan dengan acceptance plan:** rencana ini MELENGKAPI, bukan
menggantikan, acceptance plan. Beberapa skenario tumpang tindih
dengan uji T1.5-F/G/H — di sana dirujuk eksplisit agar tidak ada
uji ganda yang tidak disadari.

## Legenda label klaim

- **[VF]** VERIFIED FACT — terverifikasi langsung (DB/log/kode/tes).
- **[DF]** DOCUMENTED FACT NOT YET REVALIDATED — tercatat di dokumen
  sumber, belum divalidasi ulang sejak ditulis.
- **[AS]** ASSUMPTION — asumsi yang dipakai rencana ini.
- **[PR]** PROPOSAL — usulan dalam rencana ini, belum diputuskan.
- **[OQ]** OPEN QUESTION — pertanyaan terbuka, belum ada jawaban.
- **[OD]** OWNER DECISION REQUIRED — butuh keputusan owner.

---

## 1. Baseline acuan (scorecard 13 metrik)

Semua skenario diukur terhadap baseline "solid" ini. Bila sebuah
uji membuat metrik memburuk DI LUAR ambang yang ditetapkan
skenario itu, uji tersebut GAGAL dan pemicu rollback berlaku.

| # | Metrik | Baseline (21–22 Nov) | Sumber |
|---|---|---|---|
| 1 | Publish/hari | 206, 0 FAILED/BLOCKED | [VF] PLAN §2 |
| 2 | Keterlambatan terbit p90 | 30,2 mnt (median 11,6 mnt) | [VF] PLAN §2 |
| 3 | Kematian encoder tak direncanakan | **KOREKSI:** 12×/17 jam, slot **114** — BUKAN "5×/24 jam slot 706" (pemetaan lama SALAH, dibuktikan V2-002) | [VF] V2-002 |
| 4 | Encoder orphan | 0 (706 dipulihkan 22 Nov 08:10) | [VF] D11 |
| 5 | "database is locked"/24 jam | 31× | [VF] PLAN §2 |
| 6 | Load Master (rata-rata) | 3,5–4,2 | [VF] PLAN §2 |
| 7 | CPU unit web (rata-rata) | 1,02 core | [VF] PLAN §1.5 |
| 8 | Ingestion RUNNING saat backlog ada | 2 job | [VF] PLAN §2 + V2-003 |
| 9 | Churn SUPERSEDED ingestion | 66% (7 hari) | [VF] PLAN §2 + V2-003 |
| 10 | Disk Master (puncak wave) | 72% | [VF] PLAN §2 |
| 11 | Healthz p95 | ±10 ms | [VF] PLAN §2 |
| 12 | Stream live aktif | 2 reguler (114, 706); limit 16 | [VF] |
| 13 | Error class baru pasca-fase | 0 dalam 72 jam observasi | [DF] PLAN §2 (target) |

**Baseline tambahan:** Stage 2: 12,8 dtk CPU/klip, wall 100,1
dtk/klip, maks 900,8 dtk; 1.011 DONE/7 hari. Reassemble: 64×/24
jam, ±0,9 core/job, ±6 mnt/job, ±70 MB/job, puncak ±1 core.
[VF — Acceptance Plan §1]

---

## 2. Aturan umum uji injeksi

1. **Canary dulu, produksi belakangan.** Setiap skenario dijalankan
   PERTAMA di lingkungan canary/terisolasi (Fase 1.2–1.5). Uji
   terhadap produksi penuh HANYA setelah lolos di canary DAN
   disetujui owner per skenario. [PR]
2. **Satu kegagalan pada satu waktu.** Jangan menginjeksikan dua
   kegagalan bersamaan — penyebab tidak dapat diatribusikan.
   [PR]
3. **Kriteria abort.** Hentikan uji SEGERA (dan pulihkan) bila:
   live down >5 menit; publish berhenti total (0 PUBLISHED
   dalam 2 jam pada jam aktif); duplikat supervisor terdeteksi;
   dobel publish terdeteksi. Ambang ini sama dengan pemicu P0
   rollback runbook §3. [DF — Rollback §3]
4. **Snapshot pra-uji.** Sebelum tiap skenario: catat scorecard
   saat itu, status job aktif, status stream, dan hash/ukuran
   DB (bila relevan). Tanpa snapshot, "memburuk" tidak dapat
   dibuktikan. [PR — mengikuti gate policy V2.0 §18]
5. **Jendela observasi.** 72 jam untuk perubahan operasional;
   7 hari untuk metrik yang membutuhkan window 7 hari (churn).
   [DF — V2.0 §18]
6. **Aturan rollback saat ini vs revisi.** Rollback runbook §1
   saat ini memakai aturan "satu metrik memburuk = gerbang
   gagal = rollback" — sprint F2 mengonfirmasi aturan ini
   terlalu sensitif (spike CPU 3% sesaat dari backup akan
   memicu rollback) dan item #6 sprint akan merevisinya
   menjadi 3 kategori (hard safety breach / SLO degradation /
   transient noise). Sampai item #6 selesai, skenario di
   dokumen ini memakai **ambang eksplisit per skenario**
   (bukan aturan satu-metrik mentah) agar hasil uji tidak
   ditentukan oleh noise. [DF — Rollback §1; DF — Sprint F2]
7. **Data tidak pernah dihapus saat uji.** Injeksi kegagalan
   TIDAK BOLEH menghapus data produksi; prosedur pemulihan
   mengembalikan ROUTING dan LAYANAN, bukan menghapus data.
   [DF — Rollback §1 prinsip 3; L4]
8. **Drain sebelum mematikan.** Kecuali skenario secara
   eksplisit mensimulasikan kematian mendadak (crash), matikan
   layanan dengan drain: hentikan pemberian job baru, tunggu
   job aktif selesai (atau lease habis), baru matikan.
   [DF — Rollback §1 prinsip 6; V2.0 §12]
9. **Semua hasil dicatat apa adanya.** Uji yang gagal adalah
   TEMUAN, bukan kegagalan dokumen — catat angkanya, perbaiki,
   uji ulang. Jangan pernah menandai "lolos" untuk uji yang
   tidak benar-benar dijalankan. [PR — prinsip yang sama
   dengan catatan T0.5 acceptance plan]

---

## 3. Skenario

### FI-01 — Master API mati saat worker menjalankan job

**Kondisi awal:** 1+ job RUNNING di worker prep VPS 8 (via API);
1+ stream live aktif di Live-01; Master API merespons normal
(healthz 200). [PR — setup uji]

**Simulasi kegagalan:** hentikan layanan API Master (bukan
seluruh box — hanya control plane API) selama 10 menit,
sementara job worker masih berjalan dan stream live masih
mengudara. [PR]

**Hasil yang diharapkan** [DF — V2.0 §19, §20]:

- Stream live yang sudah aktif TETAP berjalan (availability
  lokal live).
- Worker TIDAK mengambil job baru (tidak ada klaim baru).
- Worker TIDAK membuat ownership baru secara lokal.
- Job yang sedang berjalan: worker melanjutkan pekerjaan
  lokalnya; heartbeat/complete GAGAL (Master tidak dapat
  dijangkau) — worker mencatat kegagalan dan TIDAK menganggap
  job hilang.
- Setelah API pulih: Master memeriksa lease, fencing, dan job
  events; job yang leasenya masih berlaku dilanjutkan worker
  pemiliknya; job yang leasenya habis selama outage kembali
  ke antrean dan dapat diklaim ulang (token naik).

**Metrik yang diamati:** status stream live (mengudara/tidak);
jumlah klaim baru selama outage (harus 0); jumlah heartbeat
gagal; waktu rekonsiliasi pasca-pulih; scorecard #1, #2, #12.

**Kriteria lolos:** live tidak terganggu (0 kematian encoder
selama uji); 0 klaim baru selama outage; 0 duplikat ownership;
semua job RUNNING ter-rekonsiliasi dalam 15 menit pasca-pulih
(dilanjutkan ATAU di-requeue — tercatat jelas yang mana);
0 job hilang tanpa jejak. **Gagal bila:** live ikut mati;
worker membuat ownership lokal; job hilang; duplikat hasil.

**Prosedur pemulihan:** start ulang API Master → verifikasi
healthz 200 → periksa `job_events` (lease/fencing) →
rekonsiliasi manual bila ada job yang statusnya tidak jelas
→ lapor owner bila ada job yang tidak ter-rekonsiliasi
otomatis. Bila API tidak pulih dalam 30 menit (RTO target),
ikuti langkah darurat rollback runbook §4. [DF — Rollback §2, §4]

**Kaitan:** pendalaman T1.5-G (Master disconnect) — T1.5-G
menguji putusnya koneksi; FI-01 menguji matinya API dengan
job RUNNING + rekonsiliasi pasca-pulih. [DF — Acceptance §6]

### FI-02 — Tailscale terputus

**Kondisi awal:** sama seperti FI-01; konektivitas
Master↔VPS 8 normal; heartbeat worker tercatat teratur.

**Simulasi kegagalan:** putuskan konektivitas Tailscale
Master↔VPS 8 selama 10 menit (mis. hentikan daemon Tailscale
di salah satu sisi, atau blokir rute). [PR]

**Hasil yang diharapkan** [DF — V2.0 §19; Rollback §3]:

- Stream live TETAP jalan (tidak bergantung pada control
  plane untuk siaran yang sudah aktif).
- Penugasan baru DITAHAN (worker tidak dapat klaim; Master
  tidak dapat memberi job).
- Heartbeat stale >90 dtk memicu ALARM (watchdog).
- Worker pause klaim baru TANPA error berulang (backoff,
  bukan badai retry).
- Setelah konektivitas pulih: klaim lanjut normal; heartbeat
  kembali; lease/fencing di-rekonsiliasi.

**Metrik yang diamati:** status stream; waktu alarm pertama
(target ≤2,5 menit — T1.4-B); jumlah percobaan klaim selama
putus (harus menurun/backoff, bukan konstan); waktu pulih
klaim normal; scorecard #12.

**Kriteria lolos:** live tidak terganggu; alarm berbunyi
≤2,5 menit; 0 badai retry (laju percobaan klaim menurun
setelah 3× gagal); klaim normal ≤5 menit pasca-pulih; 0 job
hilang. **Gagal bila:** live ikut mati; alarm tidak berbunyi
atau >5 menit; klaim berulang error tanpa backoff.

**Prosedur pemulihan:** pulihkan Tailscale → verifikasi
heartbeat kembali → periksa lease yang kedaluwarsa selama
putus → job yang ter-requeue diproses normal → tutup alarm.
Ini BUKAN rollback (P2) — stream jalan tetap jalan.
[DF — Rollback §3]

**Kaitan:** T1.5-G (Master disconnect) + T1.4-B (watchdog +
alarm). [DF — Acceptance §5, §6]

### FI-03 — Worker restart saat lease aktif

**Kondisi awal:** 1+ job RUNNING di worker prep VPS 8; lease
aktif (belum kedaluwarsa); worker lain (atau Master) tersedia
untuk mengambil alih.

**Simulasi kegagalan:** matikan paksa proses worker prep
(`kill -9` setara crash — TANPA drain) saat job berjalan.
[PR — simulasi crash, bukan restart tertib]

**Hasil yang diharapkan** [DF — V2.0 §8.5, §19]:

- Job yang terinterupsi TIDAK hilang: setelah lease habis,
  job kembali ke antrean (event `LEASE_EXPIRED`) dan dapat
  diklaim worker lain / worker yang sama setelah restart.
- Fencing token naik pada klaim berikutnya.
- Hasil parsial worker lama (di direktori temporary) TIDAK
  dipromosikan; dibersihkan sesuai retensi.
- Worker yang restart mendaftar ulang (register idempotent)
  dan dapat bekerja normal.

**Metrik yang diamati:** waktu job kembali ke antrean
(≈ sisa lease); waktu job selesai oleh worker pengganti;
jumlah duplikat job (harus 0); jumlah hasil ganda (harus 0);
scorecard #8, #9.

**Kriteria lolos:** 0 duplikat job; 0 hasil ganda; job
terinterupsi selesai oleh worker pengganti dalam 2× durasi
normal job itu; worker restart dapat register + klaim normal.
**Gagal bila:** duplikat job; hasil ganda; job hilang tanpa
jejak; worker tidak dapat register ulang.

**Prosedur pemulihan:** start ulang worker → verifikasi
register + heartbeat → periksa job yang ter-requeue →
bersihkan direktori temporary job yang terinterupsi →
verifikasi 0 duplikat ownership.

**Kaitan:** T1.5-F (Worker crash) — FI-03 menambahkan
verifikasi fencing token naik + hasil parsial tidak
dipromosikan. [DF — Acceptance §6]

### FI-04 — SQLite terkunci

**Kondisi awal:** Master berjalan normal; beban tulis normal;
frekuensi lock terukur (baseline 31×/24 jam).

**Simulasi kegagalan:** buat kontensi tulis buatan — jalankan
penulis konkuren berat ke DB Master selama 10 menit (mis.
skrip yang menulis batch event + klaim simultan dari 2
worker), sambil mengukur lock. [PR — beban sintetis; siapkan
skrip SEBELUM uji, jangan bebani produksi dadakan, sesuai
catatan acceptance plan §7]

**Hasil yang diharapkan** [DF — V2.0 §9.3, §19]:

- Klaim/status TERLAMBAT (bukan gagal total): transaksi
  pendek me-retry dengan busy timeout dan akhirnya berhasil.
- TIDAK ADA klaim ganda akibat retry (transaksi atomik).
- TIDAK ADA data rusak (integrity_check OK pasca-uji).
- Frekuensi lock tercatat (observability).

**Metrik yang diamati:** jumlah "database is locked"/jam
selama uji vs baseline; latensi klaim p95; jumlah klaim gagal
total (harus 0); hasil `integrity_check`; scorecard #5, #11.

**Kriteria lolos:** 0 klaim gagal total; 0 klaim ganda;
integrity_check OK; latensi klaim p95 <1 dtk selama uji
(nilai ambang [PR] — V2.0 §9.4 memakai trigger evaluasi
p95 tulis >100 ms konsisten untuk PostgreSQL; ambang 1 dtk
di sini adalah batas uji, bukan trigger migrasi).
**Gagal bila:** ≥1 klaim gagal total; klaim ganda;
integrity_check gagal; Master tidak responsif >60 dtk.

**Prosedur pemulihan:** hentikan beban sintetis → verifikasi
klaim normal → jalankan `integrity_check` → catat frekuensi
lock pasca-uji → bila lock tetap >10/hari pasca-optimasi,
catat sebagai temuan menuju trigger evaluasi PostgreSQL
(V2.0 §9.4 — trigger EVALUASI, bukan migrasi otomatis).

**Catatan:** baseline lock 31×/24 jam SUDAH di atas trigger
evaluasi >10/hari — artinya evaluasi PostgreSQL adalah
pekerjaan yang sudah jatuh tempo secara prinsip, BUKAN
keputusan yang diambil uji ini. Uji ini mengukur perilaku
saat lock, bukan memutuskan migrasi. [VF — baseline;
DF — V2.0 §9.4]

### FI-05 — Disk penuh

**Kondisi awal:** VPS 8 berjalan; disk bebas di atas ambang
gate (40 GB — G-4); job prep aktif; stream live aktif.

**Simulasi kegagalan:** isi disk VPS 8 secara terkendali
(file dummy di direktori scratch) sampai bebas <40 GB, lalu
<10 GB. Lakukan bertahap (40 GB → 20 GB → 10 GB) dengan jeda
observasi. [PR]

**Hasil yang diharapkan** [DF — V2.0 §10.5, §19; Acceptance
G-4, G-8]:

- Pada <40 GB bebas: klaim BARU di box itu ditolak (disk
  gate); job yang sedang berjalan TIDAK dibunuh.
- Pada <10 GB bebas: admission berhenti total untuk box itu;
  alarm disk berbunyi.
- Stream live TIDAK terganggu (live hanya butuh ±5 GB file
  loop; disk governor memprioritaskan live).
- Janitor TIDAK menghapus file channel CONNECTED secara
  otomatis untuk membebaskan ruang (janitor status-only —
  T0.6; penghapusan butuh audit + keputusan).

**Metrik yang diamati:** disk bebas per ambang; jumlah klaim
ditolak per ambang; status stream live; status job aktif
(selesai normal / ditahan); scorecard #10, #12.

**Kriteria lolos:** gate bereaksi di SETIAP ambang (klaim
ditolak <40 GB); 0 job aktif dibunuh; live tidak terganggu;
0 file CONNECTED terhapus; alarm berbunyi <10 GB.
**Gagal bila:** gate tidak bereaksi; job aktif dibunuh;
live terganggu; file CONNECTED terhapus; tidak ada alarm.

**Prosedur pemulihan:** hapus file dummy → verifikasi disk
bebas kembali >40 GB → verifikasi klaim diterima lagi →
audit: 0 file produksi terhapus selama uji → tutup alarm.
Bila disk penuh terjadi di PRODUKSI (bukan uji), ikuti
janitor quarantine + audit volume (V2.0 §4.4, §10.5) — JANGAN
hapus file produksi tanpa audit.

**Kaitan:** G-4 (disk gate ingestion), G-8 (kuota disk
co-tenant), T1.5-D (guard lokal). [DF — Acceptance §6, §7]

### FI-06 — Proses FFmpeg mati (encoder)

**Kondisi awal:** stream live aktif (canary atau reguler);
encoder FFmpeg berjalan; supervisor live + watchdog R3b aktif.

**Simulasi kegagalan:** matikan paksa proses encoder FFmpeg
satu stream (simulasi kematian encoder — BUKAN matikan
supervisor). [PR]

**Hasil yang diharapkan** [DF — V2.0 §19; temuan V2-002]:

- Supervisor mendeteksi kematian (health check PID) dan
  memulai pemulihan otomatis.
- Stream kembali mengudara (reconnect) — target pemulihan
  ≤5 menit (ambang P0 rollback: live down >5 menit).
  [DF — Rollback §3]
- Watchdog R3b (bila kematian didahului stall feeder)
  mencatat stall + tindakan yang diambil.
- TIDAK ADA orphan: setelah pemulihan, tepat 1 encoder per
  stream; tidak ada encoder tanpa feeder (pelajaran 706:
  orphan tanpa audio ±17 jam).
- Exit code + ekor stderr kematian TERCATAT (R6 — perbaikan
  yang sudah live sejak commit `56f7a6d`).

**Metrik yang diamati:** waktu deteksi kematian; waktu
stream kembali mengudara; jumlah encoder per stream
pasca-pulih (harus 1); keberadaan feeder audio pasca-pulih
(verifikasi audio mengalir, bukan hanya video);
keterbacaan log kematian (exit code tercatat?);
scorecard #3, #4, #12.

**Kriteria lolos:** stream pulih ≤5 menit; tepat 1 encoder
+ 1 feeder pasca-pulih; audio terverifikasi mengalir;
exit code + stderr tercatat; 0 orphan.
**Gagal bila:** pulih >5 menit; orphan tercipta; audio
tidak mengalir (video-only tanpa terdeteksi — kegagalan
yang sama dengan insiden 706); exit code tidak tercatat.

**Prosedur pemulihan:** bila pemulihan otomatis gagal:
verifikasi status DB task (pelajaran ERROR-race 22 Nov
07:19 — supervisor mengabaikan task berstatus ERROR);
perbaiki status bila perlu; start manual stream; verifikasi
"berhasil mengudara" + audio mengalir; bila stream tidak
pulih, pindahkan ke Master (rollback §6.3) dan lapor owner.

**Catatan kejujuran:** baseline kematian encoder adalah
12×/17 jam (slot 114) — area ini adalah yang PALING LEMAH
di sistem. Uji ini TIDAK BOLEH dijalankan pada stream
produksi reguler sebelum canary membuktikan pemulihan
otomatis bekerja; observasi 24 jam pasca-iterasi-2 (commit
`021a9f0`) masih berjalan saat rencana ini ditulis.
[VF — V2-002; DF — Acceptance §1]

**Kaitan:** T0.5 (encoder stabil pasca-P16), T1.3-F
(failover live). [DF — Acceptance §2, §4]

### FI-07 — Output file terbuat sebagian

**Kondisi awal:** job Stage 2 (atau ingestion) RUNNING di
worker prep; artefak output sedang ditulis ke direktori
temporary unik.

**Simulasi kegagalan:** hentikan worker di tengah penulisan
output (kill saat file ~50% tertulis), lalu start ulang
worker dan biarkan job di-requeue + dikerjakan ulang.
[PR]

**Hasil yang diharapkan** [DF — V2.0 §8.4]:

- File sebagian TIDAK PERNAH dipromosikan ke lokasi final.
- `complete` dengan artefak yang tidak lolos validasi
  (ukuran/checksum tidak cocok) DITOLAK (`422
  RESULT_INVALID`).
- File sebagian dibersihkan (atau dikarantina) sesuai
  retensi — tidak menumpuk di staging tanpa batas.
- Job yang dikerjakan ulang menghasilkan SATU hasil final
  yang valid (tidak ada 2 file final dari 1 job).

**Metrik yang diamati:** jumlah file di lokasi final pasca-
uji (harus 1, valid); jumlah file sebagian yang tersisa di
staging (harus 0 setelah cleanup); hasil validasi checksum;
scorecard #13 (error class baru).

**Kriteria lolos:** tepat 1 file final valid; 0 file
sebagian di lokasi final; 0 file sebagian tersisa di staging
pasca-cleanup; `complete` dengan artefak tidak valid
ditolak.
**Gagal bila:** file sebagian lolos ke lokasi final; 2 file
final dari 1 job; file sebagian menumpuk tanpa cleanup.

**Prosedur pemulihan:** hapus/karantina file sebagian →
verifikasi lokasi final hanya berisi hasil valid → jalankan
ulang job bila belum selesai → audit staging (tidak ada
sisa).

**Kaitan:** T1.5-H (idempotency) — FI-07 menguji sisi
VALIDASI artefak, T1.5-H menguji sisi DEDUP submit.
[DF — Acceptance §6]

### FI-08 — Publish timeout

**Kondisi awal:** job publish siap (video final valid);
kuota vault mencukupi; koneksi ke YouTube normal.

**Simulasi kegagalan:** buat upload YouTube timeout di
tengah jalan — simulasi dengan (a) memutus jaringan ke
YouTube di tengah upload (uji jaringan), atau (b) memakai
endpoint mock yang tidak pernah merespons (uji integrasi,
DIUTAMAKAN bila tersedia — tidak membakar kuota).
[PR — metode (b) diutamakan karena tidak memakai kuota
produksi]

**Hasil yang diharapkan** [DF — V2.0 §8.4, §8.5, §19]:

- Upload yang timeout TIDAK dianggap sukses dan TIDAK
  dianggap gagal secara pasti — statusnya TIDAK DIKETAHUI
  sampai reconciliation.
- Operation record upload dipakai untuk reconciliation:
  periksa apakah video sudah ada di YouTube SEBELUM
  memutuskan retry.
- TIDAK ADA dobel publish (video yang sama terbit 2×).
- Bila error adalah `quotaExceeded`: TIDAK ADA retry
  otomatis — pause dan tunggu reset / tindakan operator.
- Job publish TIDAK menggantung selamanya: timeout
  didefinisikan, setelah itu job masuk RETRY_WAIT atau
  FAILED dengan klasifikasi yang benar.

**Metrik yang diamati:** jumlah video terbit di YouTube
untuk job itu (harus ≤1); status operation record
(tercatat?); waktu reconciliation; jumlah retry upload
(harus 0 bila quotaExceeded); scorecard #1, #2.

**Kriteria lolos:** ≤1 video terbit; operation record
lengkap; reconciliation memutuskan dengan benar (retry
hanya bila video BELUM ada); 0 retry saat quotaExceeded;
job tidak menggantung >30 menit tanpa status jelas.
**Gagal bila:** dobel publish; retry membakar kuota saat
quotaExceeded; job menggantung tanpa batas; operation
record hilang.

**Prosedur pemulihan:** jalankan reconciliation manual
(cek YouTube) → bila video sudah ada: tandai job COMPLETED
dengan video_id yang ada (jangan upload ulang) → bila
belum ada: retry 1× dengan backoff → bila retry gagal:
FAILED + eskalasi operator → audit kuota vault yang
terpakai.

**Catatan:** mekanisme reconciliation upload YouTube
adalah OQ-409 di kontrak API — FI-08 TIDAK DAPAT lolos
penuh sebelum OQ-409 diputuskan dan diimplementasikan.
Sampai saat itu, FI-08 dicatat "DIBLOKIR OQ-409" bila
reconciliation belum ada, bukan "lolos".
[OQ — Kontrak API §11]

### FI-09 — Worker lama menyelesaikan job setelah lease berpindah (stale worker)

**Kondisi awal:** worker A dan B terdaftar (capability sama);
1 job QUEUED.

**Simulasi kegagalan:** jalankan kasus uji TC-STALE-1 dari
kontrak API §7.2 secara lengkap: A klaim (token 41) →
putuskan A → lease habis → B klaim (token 42) → B selesai →
A kembali dan mengirim hasil lama dengan token 41.
[PR — skenario ini SAMA dengan TC-STALE-1; FI-09 adalah
eksekusi TC-STALE-1 sebagai uji injeksi kegagalan formal
dengan metrik + pemulihan]

**Hasil yang diharapkan** [DF — V2.0 §8.3, §19]:

- Hasil A DITOLAK (`410 STALE_FENCING_TOKEN`); event
  `FENCING_REJECTED` tercatat.
- Hasil B tetap menjadi SATU-SATUNYA hasil final.
- Heartbeat A dengan token 41 juga ditolak; lease TIDAK
  diperpanjang untuk A.
- Urutan `job_events` lengkap dan benar (CLAIMED 41 →
  LEASE_EXPIRED → CLAIMED 42 → COMPLETED 42 →
  FENCING_REJECTED 41).

**Metrik yang diamati:** response code `complete` A (harus
410); jumlah hasil final (harus 1, milik B); kelengkapan
`job_events`; scorecard #13.

**Kriteria lolos:** semua asersi TC-STALE-1 (§7.2 kontrak)
terpenuhi; varian TC-STALE-2 s.d. TC-STALE-6 juga lolos
(bila dijalankan dalam sesi yang sama).
**Gagal bila:** hasil A diterima (apa pun caranya); hasil B
tertimpa; event tidak lengkap.

**Prosedur pemulihan:** tidak ada pemulihan sistem yang
dibutuhkan bila uji lolos (penolakan adalah perilaku benar).
Bila uji GAGAL (hasil basi lolos): HENTIKAN pemberian job
ke worker yang terlibat; audit `job_events` + lokasi final
(hasil mana yang dipromosikan?); promosikan hasil yang benar
secara manual bila perlu; tandai sebagai temuan P0 —
fencing adalah pengaman terakhir terhadap hasil ganda;
jangan lanjutkan canary sampai diperbaiki dan TC-STALE-1
lolos.

**Kaitan:** TC-STALE-1 s.d. TC-STALE-6 (Kontrak API §7);
T1.5-H (Fencing + idempotency). [DF — Acceptance §6]

### FI-10 — Backup gagal

**Kondisi awal:** jadwal backup berjalan (Restic dan/atau
Litestream — keduanya tergate V2-009; skenario ini berlaku
untuk mana pun yang aktif); backup terakhir sukses; RPO
target ≤5 menit. [DF — V2.0 §14; Rollback §2]

**Simulasi kegagalan:** buat backup GAGAL — putuskan akses
ke backend storage saat backup berjalan (atau isi backend
sampai kuota habis, di lingkungan uji). [PR]

**Hasil yang diharapkan:**

- Kegagalan backup TERDETEKSI dan memicu ALARM (backup
  freshness check — V2.0 §13.3 business health checks).
- Backup berikutnya (jadwal berikutnya) mencoba lagi secara
  otomatis; TIDAK ADA backup yang diam-diam dilewati.
- Operasi produksi (publish/live/prep) TIDAK terganggu oleh
  kegagalan backup.
- RPO target DINYATAKAN TERANCAM (bukan diklaim terpenuhi)
  selama tidak ada backup sukses — kejujuran target
  [DF — Rollback §2].

**Metrik yang diamati:** waktu alarm pertama; jumlah backup
yang dilewati tanpa alarm (harus 0); selisih waktu backup
sukses terakhir (backup age); gangguan terhadap publish/
live (harus 0); scorecard #1, #12.

**Kriteria lolos:** alarm ≤15 menit setelah kegagalan;
0 backup dilewati tanpa alarm; backup berikutnya sukses
setelah backend pulih; 0 gangguan produksi.
**Gagal bila:** tidak ada alarm; backup dilewati diam-diam;
produksi terganggu; backup age >1 jam tanpa eskalasi.

**Prosedur pemulihan:** pulihkan akses backend → jalankan
backup manual segera → verifikasi backup sukses +
integrity check → catat backup age maksimum yang terjadi
(RPO aktual selama insiden) → tutup alarm → bila backend
tidak pulih, eskalasi ke owner (pemilihan backend
cadangan adalah keputusan owner — V2.0 §24).

**Catatan:** provider + biaya object storage off-site MASIH
terbuka (V2-009) — FI-10 hanya dapat dijalankan penuh
setelah backend backup dipilih dan aktif. Sampai saat itu,
FI-10 dicatat "DIBLOKIR V2-009". [OD — V2.0 §24; Sprint #7]

### FI-11 — Restore backup gagal

**Kondisi awal:** salinan backup/DR tersedia (dari T1.4-A:
sync + checksum cocok, `integrity_check` OK); lingkungan
terisolasi untuk restore drill siap.

**Simulasi kegagalan:** jalankan restore drill #1 (T1.4-C)
dengan salinan yang RUSAK — gunakan salinan backup yang
sengaja dikorupsi (1 byte diubah di file DB salinan), atau
salinan dari backup yang gagal sebagian (FI-10).
[PR — korupsi buatan HANYA pada salinan, TIDAK PERNAH pada
DB produksi]

**Hasil yang diharapkan** [DF — V2.0 §19; Rollback §3]:

- Restore drill MENDETEKSI korupsi (`integrity_check`
  gagal / checksum tidak cocok) SEBELUM aplikasi boot dari
  salinan itu.
- Salinan yang korup TIDAK PERNAH dipakai — sistem tetap di
  Master (DB produksi tidak tersentuh).
- Drill dicatat GAGAL dengan angka yang jelas (bukan
  "lolos" dan bukan disembunyikan).
- RPO/RTO DINYATAKAN BELUM TERBUKTI (bukan diklaim
  terpenuhi).

**Metrik yang diamati:** hasil `integrity_check` salinan
(harus GAGAL terdeteksi); apakah aplikasi boot dari salinan
korup (harus TIDAK); waktu deteksi; status DB produksi
(tidak tersentuh); scorecard #13.

**Kriteria lolos:** korupsi terdeteksi sebelum boot; 0 boot
dari salinan korup; DB produksi tidak tersentuh; drill
tercatat gagal dengan alasan yang jelas.
**Gagal bila:** korupsi TIDAK terdeteksi (aplikasi boot
dari salinan korup); DB produksi tersentuh; drill dicatat
lolos padahal salinan korup.

**Prosedur pemulihan:** buang salinan korup (salinan, bukan
data produksi) → jalankan backup baru yang bersih →
verifikasi `integrity_check` + checksum → ulangi restore
drill dengan salinan bersih → catat RPO/RTO yang TERCAPAI
apa adanya → bila 2 drill berturut-turut gagal, eskalasi
ke owner (strategi backup perlu ditinjau ulang).

**Kaitan:** T1.4-C (Restore drill #1) — FI-11 adalah drill
dengan salinan korup; T1.4-C adalah drill dengan salinan
bersih. Keduanya wajib lolos (dengan maknanya masing-
masing) sebelum DR dinyatakan selesai. [DF — Acceptance §5]

---

## 4. Matriks skenario → scorecard + rollback

| ID | Skenario | Scorecard yang diamati | Pemicu rollback bila gagal |
|---|---|---|---|
| FI-01 | Master API mati | #1, #2, #12 | Job hilang / duplikat ownership (P0) |
| FI-02 | Tailscale putus | #12 | Live ikut mati (P0); alarm tidak berbunyi (P1) |
| FI-03 | Worker restart | #8, #9 | Duplikat job / hasil ganda (P0) |
| FI-04 | SQLite terkunci | #5, #11 | Klaim gagal total / integrity gagal (P0) |
| FI-05 | Disk penuh | #10, #12 | File CONNECTED terhapus (P0); gate tidak bereaksi (P1) |
| FI-06 | FFmpeg mati | #3, #4, #12 | Pulih >5 mnt (P0); orphan / audio hilang (P0) |
| FI-07 | Output sebagian | #13 | File sebagian lolos ke final (P1) |
| FI-08 | Publish timeout | #1, #2 | Dobel publish (P0); retry bakar kuota (P1) |
| FI-09 | Stale worker | #13 | Hasil basi diterima (P0 — hentikan canary) |
| FI-10 | Backup gagal | #1, #12 | Backup dilewati tanpa alarm (P1) |
| FI-11 | Restore gagal | #13 | Korupsi tidak terdeteksi (P0) |

Tingkat P0/P1 mengikuti rollback runbook §3. "P0" berarti
hentikan dan pulihkan segera; "P1" berarti rollback langkah
terkait + diagnosis sebelum mengulang. [DF — Rollback §3]

---

## 5. Urutan & gerbang pelaksanaan

**Urutan yang disarankan** [PR — dari risiko terendah ke
tertinggi, dan dari prasyarat ke lanjutan]:

| Tahap | Skenario | Prasyarat |
|---|---|---|
| Canary awal (Fase 1.2–1.5) | FI-07, FI-09, FI-03 | Kontrak API terimplementasi (TC-STALE-1); worker prep canary jalan |
| Canary lanjut (Fase 1.5) | FI-01, FI-02, FI-04 | T1.5-F/G lolos; watchdog aktif (T1.4-B) |
| Pra-live-canary (sebelum 1.3) | FI-06, FI-05 | Supervisor + watchdog R3b terverifikasi; disk gate aktif (G-4) |
| Pra-operasi-penuh (Fase 1.6) | FI-08, FI-10, FI-11 | OQ-409 diputuskan (FI-08); backend backup aktif (FI-10/11, V2-009) |

**Gerbang:**

- FI-09 HARUS lolos sebelum worker prep canary diperluas
  (fencing adalah pengaman terakhir hasil ganda).
- FI-06 HARUS lolos di canary sebelum live canary (Fase 1.3)
  diperluas — encoder adalah area terlemah (baseline
  12×/17 jam).
- FI-10/FI-11 TIDAK DAPAT dijalankan sebelum V2-009
  (backend backup) diputuskan — dicatat DIBLOKIR, bukan
  dilewati diam-diam.
- FI-08 TIDAK DAPAT lolos penuh sebelum OQ-409
  (reconciliation upload) diputuskan — dicatat DIBLOKIR.

---

## 6. Log uji (diisi saat eksekusi)

| ID | Tanggal | Pelaksana | Hasil | Angka kunci | Tindak lanjut |
|---|---|---|---|---|---|
| FI-01 | | | | | |
| FI-02 | | | | | |
| FI-03 | | | | | |
| FI-04 | | | | | |
| FI-05 | | | | | |
| FI-06 | | | | | |
| FI-07 | | | | | |
| FI-08 | | | | | |
| FI-09 | | | | | |
| FI-10 | | | | | |
| FI-11 | | | | | |

Kolom "Hasil": LOLOS / GAGAL / DIBLOKIR (dengan alasan) /
TIDAK TERUJI (dengan alasan). "TIDAK TERUJI" bukan lolos.

---

## 7. Pertanyaan terbuka (OQ rencana uji ini)

| ID | Pertanyaan | Menghambat |
|---|---|---|
| OQ-501 | Siapa yang mengeksekusi injeksi (operator? engineer? skrip otomatis?) dan di mana skrip beban/injeksi disimpan? | Semua skenario |
| OQ-502 | Lingkungan terisolasi untuk restore drill + injeksi — sudah tersedia, atau harus dibuat? (Berkaitan OQ-204 acceptance: skrip beban sintetis) | FI-04, FI-05, FI-11 |
| OQ-503 | Bagaimana memverifikasi "broadcast benar-benar live" dari sisi YouTube secara otomatis? (OQ-202 acceptance) | FI-01, FI-02, FI-06 |
| OQ-504 | Pra-otorisasi langkah darurat P0 (OQ-301 rollback runbook) — apakah owner menyetujui eksekusi darurat tanpa konfirmasi per kejadian? | Pemulihan FI-01, FI-06 |
| OQ-505 | Ambang latensi klaim p95 <1 dtk (FI-04) — apakah ambang ini tepat, atau harus diselaraskan dengan trigger PostgreSQL 100 ms? | FI-04 |
| OQ-506 | Metode simulasi publish timeout (mock vs putus jaringan) — apakah endpoint mock tersedia? | FI-08 |
| OQ-507 | Jadwal restore drill #1 (OQ-304 rollback runbook) — kapan dijalankan pertama kali? | FI-11 |
