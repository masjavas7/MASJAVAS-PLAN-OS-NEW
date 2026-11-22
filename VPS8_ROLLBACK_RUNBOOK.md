# RUNBOOK ROLLBACK VPS 8 — MASJAVAS TUBE OS V2

**Status:** DRAFT — disiapkan, BELUM dieksekusi / BELUM pernah
didrill. Dokumen pendamping: `VPS8_DEPLOYMENT_RUNBOOK.md` (langkah
maju) dan `VPS8_ACCEPTANCE_TEST_PLAN.md` (kriteria yang bila gagal
memicu rollback di sini).
**Versi:** 1.1 — 22 Nov 2026, 13:05 WIB (sprint item #6: aturan
rollback satu-metrik diganti klasifikasi 3 kategori; lihat §1.1
+ §3 + §3.1. Isi lain tidak berubah).

## Legenda label klaim

- **[VF]** VERIFIED FACT · **[DF]** DOCUMENTED FACT NOT YET REVALIDATED ·
  **[AS]** ASSUMPTION · **[PR]** PROPOSAL · **[OQ]** OPEN QUESTION ·
  **[OD]** OWNER DECISION REQUIRED. (Definisi penuh: Deployment Runbook.)

## 1. Prinsip

1. **Rollback adalah jalur yang direncanakan, bukan kegagalan.**
   Keputusan rollback memakai klasifikasi 3 kategori (§1.1 +
   §3 + §3.1): hanya *hard safety breach* yang memicu rollback
   darurat; *SLO degradation* menahan rollout sampai owner
   memutuskan; *transient noise* hanya diamati.

   **SUPERSEDED — aturan lama:** "Satu metrik scorecard memburuk
   = gerbang gagal = rollback" [PLAN §2/§6; aturan perubahan
   22 Nov 01:48] **tidak berlaku lagi sejak 22 Nov 2026 13:05
   WIB** (sprint item #6; temuan review F2 TERKONFIRMASI).
   Alasan: aturan lama tidak membedakan noise sesaat dari
   degradasi nyata — spike CPU 3% selama 1 menit karena backup
   akan memicu rollback penuh (false positive). Prinsip
   konservatif dipertahankan: hard safety breach tetap memicu
   rollback darurat, dan SLO degradation menahan rollout sampai
   owner memutuskan. **Catatan:** PLAN §2 masih menyatakan aturan
   lama — rekonsiliasi plan menjadi tugas berikut, bukan bagian
   item #6 (lihat OQ-306).
2. **Keputusan rollback ada di owner.** Eksekusi rollback adalah
   perubahan produksi (Level C) — butuh persetujuan owner per
   kejadian, KECUALI langkah darurat §4 yang sudah disetujui
   sebelumnya (lihat §4 — status persetujuannya masih [OD]).
3. **Data tidak pernah dihapus saat rollback.** Rollback
   mengembalikan ROUTING dan LAYANAN, bukan menghapus data.
   Penghapusan data produksi adalah L4 (dilarang). [VF — Directive]
4. **DB tetap di Master selama VPS 8** — rollback VPS 8 tidak
   pernah butuh rollback DB, selama tidak ada migration skema
   yang berjalan. Bila ada migration: expand-and-contract
   (kolom lama dipertahankan sampai semua worker memakai skema
   baru), sehingga rollback rilis tidak merusak skema.
   [DF — V2.0 §12; Track P9]
5. **Satu perubahan pada satu waktu; ukur sebelum dan sesudah.**
   [DF — Checklist §0]
6. **Drain sebelum mematikan apa pun:** hentikan pemberian job
   baru, tunggu job aktif selesai (atau lease habis), baru
   matikan layanan. Jangan bunuh job yang sedang berjalan
   kecuali langkah darurat §4. [DF — V2.0 §12]

## 1.1 Klasifikasi degradasi (otoritas: dokumen ini §3 + §3.1)

Semua ambang di bawah ini adalah **[PR]** (PROPOSAL) sampai
V2-008 mengukurnya — jangan kunci angka final sebelum benchmark
(aturan sprint #6). Baseline yang dipakai adalah scorecard
Acceptance Test Plan §1 / PLAN §2.

| Kategori | Definisi | Tindakan | Prioritas |
|---|---|---|---|
| **(a) Hard safety breach** | Kejadian yang merusak keamanan/kebenaran sistem: live down, double publish, duplikat supervisor, kehilangan data, publish berhenti total, disk kritis, kerusakan sistemik (≥3 error class baru/72 jam, ≥3 kematian encoder/72 jam) | Stop/rollback sesuai prosedur darurat §4; lapor owner segera (≤30 mnt) | P0 |
| **(b) SLO degradation** | Satu metrik scorecard memburuk melewati ambang kategorinya (§3.1) — memburuk SIGNIFIKAN dan BERKELANJUTAN (melewati threshold + durasi + jumlah sampel), bukan sesaat | HOLD rollout (tidak ada langkah baru); investigasi; owner memutuskan rollback / fix-forward / terima dengan pemantauan. TIDAK ada rollback otomatis | P1 |
| **(c) Transient noise** | Fluktuasi singkat, kecil, dan pulih sendiri — di bawah ambang kategori (b) (mis. spike CPU 3% selama 1 menit karena backup) | Observasi + catat di log; jangan hold, jangan rollback | P2 |

**Aturan eskalasi noise:** noise yang sama berulang ≥3× dalam
72 jam naik kelas menjadi SLO degradation (pola = sinyal, bukan
noise). **Aturan penentu kategori:** bila satu kejadian cocok
dengan dua kategori, pakai kategori yang lebih keras (a > b > c).
**Jendela observasi fase tetap 72 jam** (PLAN §6) — klasifikasi
ini mengubah CARA menilai hasil observasi, bukan durasinya.

## 2. Target pemulihan (TARGET, bukan janji)

| Target | Nilai | Status |
|---|---|---|
| RPO (data hilang maksimal) | ≤5 menit | [DF — V2.0 l.179] TARGET — wajib dibuktikan restore drill; BELUM pernah dibuktikan [VF — dashboard NOT READY] |
| RTO (layanan pulih maksimal) | ≤30 menit | [DF — V2.0 l.179] TARGET — sama seperti di atas |
| Pemindahan 1 stream kembali ke Master | ≤5 menit | [PR — Acceptance Test Plan T1.3-F; belum diuji] |

**Kejujuran yang mengikat:** sampai restore drill #1 (Deployment
Runbook §5, langkah 1.4.3) dijalankan dan RPO/RTO tercapai
dicatat, TIDAK ADA klaim bahwa target ini terpenuhi. Drill yang
gagal adalah temuan, bukan kegagalan dokumen — catat angkanya,
perbaiki, drill ulang.

## 3. Pemicu rollback per fase (diklasifikasi §1.1)

### 3a. Hard safety breach (P0) — stop/rollback prosedur darurat §4

| Pemicu | Tindakan | Keputusan |
|---|---|---|
| Live down (broadcast mati, bukan hanya FFmpeg mati) >5 menit | Langkah darurat §4 | [OD] persetujuan pra-otorisasi belum ada |
| Publish berhenti total (0 PUBLISHED dalam 2 jam pada jam aktif) | Langkah darurat §4 | [OD] sama seperti di atas |
| Dobel publish (2 worker menerbitkan video yang sama) | Hentikan pemberian job ke worker yang salah; drain; audit duplikat di YouTube | Owner; eksekusi segera setelah konfirmasi |
| Duplikat supervisor (2 supervisor menjalankan 1 channel) | Matikan supervisor yang salah SATU (yang bukan pemilik lease); verifikasi 1 owner | Owner; eksekusi segera setelah konfirmasi |
| DB `integrity_check` gagal pasca-sync/DR | JANGAN pakai salinan itu; tetap di Master; diagnosis | Owner |
| Kehilangan data (file channel CONNECTED terhapus/terlabel salah oleh janitor; salinan DR rusak) | Hentikan proses penyebab; pulihkan dari cadangan; diagnosis sebelum langkah apa pun | Owner; eksekusi segera setelah konfirmasi |
| Disk Master/VPS 8 >90% terpakai | Hentikan klaim baru; bersihkan staging; eskalasi darurat (disk penuh = layanan berhenti) | Owner; eksekusi segera setelah konfirmasi |
| ≥3 error class baru dalam 72 jam observasi | Rollback langkah itu; diagnosis sebelum ulangi (kerusakan sistemik) | Owner |
| Kematian encoder tak direncanakan ≥3× dalam 72 jam pasca-migrasi live canary | Pindahkan stream kembali ke Master (§6.3); diagnosis | Owner |

### 3b. SLO degradation (P1) — HOLD rollout, investigasi, keputusan owner

| Pemicu | Tindakan | Keputusan |
|---|---|---|
| Satu metrik scorecard memburuk melewati ambang kategorinya (§3.1) | HOLD (tidak ada langkah baru); investigasi; owner memutuskan rollback (§5–§6) / fix-forward / terima dengan pemantauan | Owner (berdasarkan laporan) |
| ≥1 error class baru dalam 72 jam observasi | HOLD; diagnosis; owner memutuskan rollback atau fix-forward sebelum ulangi | Owner |
| Kematian encoder tak direncanakan 1–2× dalam 72 jam pasca-migrasi live canary (dengan auto-recovery ≤2 mnt) | HOLD; investigasi rantai kematian (log R6 + relay + governor); owner memutuskan pindahkan ke Master (§6.3) atau lanjut observasi | Owner |
| Harga riil > $34,67/bln (diketahui sebelum/saat pembelian) | Hentikan pembelian; eskalasi (jangan lanjut diam-diam) | Owner |

### 3c. Transient noise (P2) — observasi, jangan rollback

| Pemicu | Tindakan | Keputusan |
|---|---|---|
| Fluktuasi metrik di bawah ambang §3.1 yang pulih sendiri (mis. spike CPU singkat karena backup/reassemble) | Catat di log observasi; pantau; bila berulang ≥3×/72 jam → naik ke SLO degradation (§1.1) | Operator |
| Tailscale putus >90 dtk (heartbeat stale) | Bukan rollback — stream jalan tetap jalan; penugasan baru tertunda; alarm; perbaiki konektivitas | Operator + lapor owner |

### 3.1 Ambang per metrik scorecard (otoritas threshold)

Baseline: Acceptance Test Plan §1 / PLAN §2 [VF]. Semua ambang
di bawah ini **[PR]** (PROPOSAL) sampai V2-008 mengukurnya.
"Durasi" = berapa lama kondisi harus bertahan; "Sampel" = berapa
pengukuran independen yang harus mengonfirmasi sebelum klasifikasi
berlaku. Satu sampel sesaat TIDAK PERNAH cukup untuk (b).

| # | Metrik (baseline) | Transient noise (c) — observasi | SLO degradation (b) — HOLD + investigasi | Hard safety breach (a) |
|---|---|---|---|---|
| 1 | Publish/hari (206; target ≥200, 0 FAILED) | 200–205/hari dalam 1 hari; pulih keesokan hari | <200/hari selama 2 hari berturut-turut (2 sampel harian); ATAU ≥1 FAILED/BLOCKED dalam 24 jam (1 sampel — baseline 0 FAILED, jadi 1 kejadian sudah signifikan) | 0 PUBLISHED dalam 2 jam pada jam aktif (publish berhenti total — §3a) |
| 2 | Keterlambatan terbit p90 (30,2 mnt; target ≤30/≤25 mnt) | p90 30–35 mnt dalam 1 hari, pulih keesokan hari | p90 >35 mnt (≥15% di atas baseline) selama 2 hari berturut-turut (2 sampel harian) | — (delay bukan safety breach; eskalasi bila delay menyebabkan publish berhenti) |
| 3 | Kematian encoder tak direncanakan (baseline TERKOREKSI 12×/17 jam slot 114 [VF — V2-002]; target 0×/72 jam) | — (setiap kematian diselidiki; tidak ada kategori noise untuk kematian) | 1–2 kematian/72 jam dengan auto-recovery ≤2 mnt (sampel = kejadian; durasi = jendela 72 jam) | ≥3 kematian/72 jam; ATAU 1 kematian dengan pemulihan >5 mnt (live down — §3a) |
| 4 | Encoder orphan (0 setelah D11; target 0) | — (setiap orphan diselidiki) | ≥1 orphan terdeteksi (1 sampel; investigasi ≤24 jam) | Orphan menyebabkan live-tanpa-audio atau duplikat supervisor (dinilai via pemicu §3a yang relevan) |
| 5 | "database is locked"/24 jam (31×; target <20/<10) | 31–40× dalam 1 hari, pulih keesokan hari | >40×/24 jam (≈30% di atas baseline) selama 2 hari berturut-turut (2 sampel harian) | — (lock adalah gejala; eskalasi bila lock menyebabkan publish gagal/data tidak tersimpan — dinilai via pemicu §3a) |
| 6 | Load Master rata-rata (3,5–4,2; target <3,0/<2,5) | Load >5 selama <1 jam lalu pulih (mis. gelombang reassemble/backup) | Rata-rata harian >4,5 (di atas rentang baseline) selama 2 hari berturut-turut (2 sampel harian) | — |
| 7 | CPU unit web rata-rata (1,02 core; target <0,2) | Spike >2 core selama <30 mnt (1 gelombang job) | Rata-rata harian >1,2 core (di atas baseline) selama 2 hari berturut-turut (2 sampel harian) | — |
| 8 | Ingestion RUNNING saat backlog ada (2; target ≥3) | 1–2 RUNNING (setara baseline — bukan regresi) | 0 RUNNING padahal backlog ada, selama >6 jam (sampel: cek tiap 1 jam, ≥6 cek berturut-turut) | — (ingestion macet tidak menghentikan live/publish; eskalasi ke owner bila >24 jam) |
| 9 | Churn SUPERSEDED ingestion (66%/7 hari; target <20%) | Fluktuasi harian (metrik ini diukur per 7 hari — 1 hari bukan sampel yang sah) | >75% dalam jendela 7 hari (1 sampel jendela; butuh 7 pengukuran harian untuk menghitungnya) | — |
| 10 | Disk Master puncak wave (72%; target ≤70%) | Puncak 72–80% selama gelombang yang mengering sendiri (<24 jam) | >80% bertahan >24 jam (sampel: cek tiap 6 jam, ≥4 cek berturut-turut) | >90% (disk penuh mengancam layanan — §3a) |
| 11 | Healthz p95 (±10 ms; target <100 ms) | p95 50–100 ms selama <1 jam, lalu pulih | p95 >100 ms bertahan >1 jam (sampel: cek tiap 15 mnt, ≥4 cek berturut-turut) ATAU rata-rata harian >100 ms selama 2 hari | — |
| 12 | Stream live aktif (2; limit 16) | Perubahan jumlah stream mengikuti demand — bukan metrik degradasi; tidak ada ambang noise/SLO | — | Kehilangan stream dinilai sebagai live down (§3a), bukan dari angka ini |
| 13 | Error class baru pasca-fase (target 0/72 jam) | — (setiap error class baru diselidiki) | ≥1 error class baru/72 jam (1 sampel; investigasi sebelum langkah berikut) | ≥3 error class baru/72 jam (kerusakan sistemik — §3a) |

**Catatan tabel:** metrik #3 memakai baseline TERKOREKSI V2-002
(12×/17 jam slot 114), BUKAN "5×/24 jam slot 706" di PLAN §2
(pemetaan lama SALAH). Target 0×/72 jam tetap berlaku karena
target mengukur keadaan pasca-perbaikan, bukan keadaan rusak
pra-perbaikan. Metrik #12 tidak punya ambang karena jumlah
stream mengikuti demand, bukan kesehatan sistem.

## 4. Langkah darurat (P0) — [PR] butuh persetujuan pra-otorisasi owner

**Status:** prosedur ini DIUSULKAN. Belum ada persetujuan owner
untuk mengeksekusinya tanpa konfirmasi per kejadian. Sampai owner
menyetujui, setiap eksekusi tetap butuh konfirmasi owner dulu.
[OD]

**Kondisi pakai:** live down >5 menit ATAU publish berhenti total
(lihat §3). **Tujuan:** kembalikan layanan ke keadaan terakhir
yang diketahui baik (Master saja) dalam ≤30 menit (RTO target).

1. **Hentikan pemberian job baru** ke VPS 8 (prep worker + live
   canary) — 1 perintah, tanpa mematikan proses dulu.
2. **Drain cepat:** tunggu job aktif selesai maksimal 5 menit;
   job yang belum selesai dilepas leasenya (kembali ke antrean
   Master — TIDAK dibunuh paksa kecuali menggantung >5 menit).
3. **Pindahkan stream live kembali ke Master:** start stream di
   Master (supervisor Master) → verifikasi "berhasil mengudara"
   → baru matikan stream di Live-01. Urutan ini mencegah
   jendela tanpa siaran. [PR — urutan dari pola cutover migrasi
   17 Nov: layanan baru jalan dulu, layanan lama dimatikan
   setelah verifikasi]
4. **Matikan layanan VPS 8** yang bermasalah (`masjavas-live`,
   prep worker) — DR replica + watchdog boleh tetap jalan.
5. **Verifikasi:** healthz Master 200; 1 job publish selesai;
   stream mengudara (verifikasi broadcast, bukan hanya proses);
   0 error di journal 15 menit.
6. **Lapor owner SEGERA** (lihat §8): apa yang terjadi, apa yang
   sudah dilakukan, keadaan kini, dan apa yang masih perlu
   diputuskan (diagnosis + jadwal coba ulang).

## 5. Rollback Fase 0 (per langkah)

| Langkah | Prosedur rollback | Verifikasi rollback selesai |
|---|---|---|
| 0.1 Pisah proses | Kembalikan ke 1 unit `masjavas-os-worker` (nonaktifkan unit baru, aktifkan unit lama) | 1 proses jalan; publish normal 24 jam |
| 0.2 Job keluar dari web | Kembalikan route lama (git revert commit terkait) + restart web | CPU web kembali ke baseline ±1,02 core; reassemble jalan lagi via web |
| 0.3 Klaim pull | Kembalikan logika antrean lama (git revert) | Churn kembali ke baseline (66%) — dicatat, bukan target |
| 0.4 Sweeper batch 5 | Kembalikan batch 2 | Feed kembali ±1 channel/menit |
| 0.5 Encoder (P16) | — (diagnosis dulu; perbaikan R3–R8 sudah live — rollback kode hanya bila terbukti memperburuk, via git revert + restart supervisor) | Bila revert: kematian kembali dipantau dari baseline lama |
| 0.6 Janitor status-only | Kembalikan klausa lama | Janitor kembali berperilaku lama; audit 24 jam |

Semua rollback Fase 0: [DF — PLAN §6, kolom Rollback].

## 6. Rollback Fase 1 (per langkah)

### 6.1 — Pembelian/provisioning (1.1)

- **Dalam 14 hari pembelian:** batalkan VPS 8 (hak withdrawal);
  hapus node Tailscale VPS 8; hapus kredensial/alias terkait.
  [DF — PLAN §6]
- **Setelah 14 hari:** tidak ada refund otomatis [AS — syarat
  Contabo belum dibaca; konfirmasi saat eksekusi]; matikan saja
  VPS 8 dan biarkan kontrak bulanan berakhir (biaya maksimal
  1 bulan $18,65). [PR]
- **Verifikasi:** Master tidak berubah (tidak ada layanan yang
  sempat pindah); scorecard tetap baseline.

### 6.2 — Deploy live service (1.2)

- Matikan `masjavas-live` di VPS 8 (belum ada stream yang pindah,
  jadi tidak ada dampak siaran). Hapus registrasi worker dari
  Master bila sudah terdaftar. [DF — PLAN §6]
- **Verifikasi:** Master menjalankan live seperti biasa; 0 stream
  di Live-01.

### 6.3 — Migrasi live canary (1.3)

1. Pilih stream yang dikembalikan (semua yang sudah pindah).
2. Drain di Live-01: hentikan pemberian shard baru; tunggu
   stream aktif selesai/dihentikan dengan rapi.
3. Start stream di Master (supervisor Master) → verifikasi
   "berhasil mengudara" + broadcast live.
4. Matikan stream di Live-01; verifikasi 0 duplikat supervisor
   (1 owner per channel — cek lease).
5. Verifikasi 30 menit: 0 error journal; publish normal.
   [DF — PLAN §6: "Pindahkan stream kembali ke Master"]

### 6.4 — DR replica + watchdog (1.4)

- Non-kritis: matikan sync + watchdog bila bermasalah; produksi
  tidak terpengaruh. [DF — PLAN §6]
- **JANGAN hapus salinan DR** yang sudah ada — salinan itu adalah
  cadangan; penghapusan backup adalah L4 (dilarang).
  [VF — Directive]

### 6.5 — Prep worker canary (1.5)

- Matikan worker prep VPS 8 (drain dulu: tunggu job aktif selesai
  atau lease habis); prep kembali ke Master (unit prep Master
  yang sudah ada sejak Fase 0). [DF — PLAN §6]
- **Verifikasi:** ingestion + Stage 2 jalan dari Master; 0 job
  hilang (audit antrean: semua job COMPLETED/QUEUED tercatat);
  0 duplikat hasil.

### 6.6 — Prep penuh (1.6)

- Turunkan dulu ke canary (Stage 2 → 2 slot, ingestion →
  konkurensi 1) bila masalahnya beban; bila masalahnya
  fungsional, matikan penuh seperti 6.5. [DF — PLAN §6:
  "Turunkan ke canary / matikan"]
- **Verifikasi:** sama seperti 6.5 + scorecard kembali ke
  baseline Fase 0.

### 6.7 — DNS/routing (bila domain terlibat)

- VPS 8 TIDAK mengubah DNS domain utama (domain tetap menunjuk
  Master/Contabo Plus 4). [VF — arsitektur target PLAN §5]
- Bila suatu langkah mengubah routing/DNS apa pun (mis. subdomain
  Live-01): kembalikan A record ke nilai sebelum perubahan;
  propagasi ±15 mnt (TTL 15 mnt); verifikasi via curl domain +
  healthz. Pola ini TERBUKTI di cutover migrasi 17 Nov
  (flip-back A record ke IP lama). [VF — migrasi]
- Perubahan DNS hanya bisa dilakukan user (akses panel
  IDCloudHost/DNSCloud) — Muse tidak bisa. [VF — migrasi]

## 7. Setelah rollback

1. **Stabilisasi 72 jam** pada keadaan pra-perubahan; scorecard
   dipantau seperti gerbang fase. [DF — PLAN §6]
2. **Catatan akar masalah:** tulis apa yang memicu rollback,
   bukti (log/angka), dan hipotesis — JANGAN ulangi langkah
   yang sama tanpa perubahan. [PR — aturan perubahan 22 Nov]
3. **Kriteria coba ulang:** akar masalah teridentifikasi +
   perbaikan terverifikasi diuji + persetujuan owner baru.
   [PR]
4. **Perbarui dokumen:** log eksekusi di Deployment Runbook §9,
   log uji di Acceptance Test Plan §9, dan log rollback di §9
   dokumen ini — dengan angka sebenarnya, bukan ringkasan.
5. **Keputusan yang tertunda dicatat** di
   `MASJAVAS_OWNER_DECISION_REGISTER.md` (bila sudah dibuat;
   bila belum, di work log). [PR]

## 8. Komunikasi

| Situasi | Kepada | Isi | Waktu |
|---|---|---|---|
| Rollback P1 direncanakan | Owner | Pemicu (metrik + angka), langkah yang di-rollback, estimasi durasi, dampak ke operator | Sebelum eksekusi |
| Rollback P0/darurat | Owner + operator | Apa yang terjadi, apa yang sudah dilakukan, keadaan kini, apa yang masih diputuskan | Segera setelah layanan pulih (≤30 mnt) |
| Rollback selesai | Owner | Keadaan akhir (scorecard), akar masalah sementara, rencana coba ulang | ≤24 jam |
| Operator terdampak | Operator (via owner) | Jendela + dampak (mis. "live kembali ke Master, tidak ada tindakan dari operator") | Sebelum/saat eksekusi |

**Template pesan singkat (P0):** "Rollback VPS 8: [pemicu + angka].
Tindakan: [langkah]. Keadaan kini: [live/publish normal/tidak].
Keputusan yang dibutuhkan: [isi]."

## 9. Log rollback (diisi saat eksekusi)

| Tanggal/WIB | Pemicu (metrik + angka) | Langkah yang di-rollback | Pelaksana | Durasi | RPO/RTO tercapai | Keadaan akhir |
|---|---|---|---|---|---|---|
| | | | | | | |

## 10. Pertanyaan terbuka (OQ) runbook ini

1. **[OQ-301]** Apakah owner menyetujui pra-otorisasi langkah
   darurat §4 (eksekusi tanpa konfirmasi per kejadian bila
   live down >5 mnt / publish berhenti total)?
2. **[OQ-302]** Syarat pembatalan Contabo setelah 14 hari —
   berapa biaya maksimal yang tertinggal?
3. **[OQ-303]** Siapa yang mengeksekusi rollback bila owner
   tidak dapat dihubungi dalam jendela P0 (delegasi? atau
   tunggu)?
4. **[OQ-304]** Kapan restore drill #1 dijadwalkan (menentukan
   apakah RPO/RTO ≤5/≤30 mnt terbukti sebelum Fase 1.4
   dinyatakan selesai)?
5. **[OQ-305]** Saluran komunikasi ke operator saat rollback
   (grup mana, siapa yang mengirim)?
6. **[OQ-306]** PLAN §2 masih menyatakan aturan lama ("satu baris
   memburuk = gerbang gagal") — kapan plan direkonsiliasi dengan
   klasifikasi 3 kategori dokumen ini? Sampai direkonsiliasi,
   dokumen ini + Acceptance Test Plan adalah otoritas aturan
   gerbang/rollback.
