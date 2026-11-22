# RUNBOOK ROLLBACK VPS 8 — MASJAVAS TUBE OS V2

**Status:** DRAFT — disiapkan, BELUM dieksekusi / BELUM pernah
didrill. Dokumen pendamping: `VPS8_DEPLOYMENT_RUNBOOK.md` (langkah
maju) dan `VPS8_ACCEPTANCE_TEST_PLAN.md` (kriteria yang bila gagal
memicu rollback di sini).
**Versi:** 1.0 — 22 Nov 2026, 12:35 WIB.

## Legenda label klaim

- **[VF]** VERIFIED FACT · **[DF]** DOCUMENTED FACT NOT YET REVALIDATED ·
  **[AS]** ASSUMPTION · **[PR]** PROPOSAL · **[OQ]** OPEN QUESTION ·
  **[OD]** OWNER DECISION REQUIRED. (Definisi penuh: Deployment Runbook.)

## 1. Prinsip

1. **Rollback adalah jalur yang direncanakan, bukan kegagalan.**
   Satu metrik scorecard memburuk = gerbang gagal = rollback.
   [VF — PLAN §6; aturan perubahan 22 Nov 01:48]
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

## 3. Pemicu rollback per fase

| Pemicu | Tingkat | Tindakan | Keputusan |
|---|---|---|---|
| Satu metrik scorecard memburuk pasca-langkah (72 jam observasi) | P1 | Rollback langkah itu (§5–§6) | Owner (berdasarkan laporan) |
| ≥1 error class baru dalam 72 jam observasi | P1 | Rollback langkah itu; diagnosis sebelum ulangi | Owner |
| Kematian encoder tak direncanakan ≥1× pasca-migrasi live canary | P1 | Pindahkan stream kembali ke Master (§6.3) | Owner |
| Duplikat supervisor (2 supervisor menjalankan 1 channel) | P0 | Matikan supervisor yang salah SATU (yang bukan pemilik lease); verifikasi 1 owner | Owner; eksekusi segera setelah konfirmasi |
| Dobel publish (2 worker menerbitkan video yang sama) | P0 | Hentikan pemberian job ke worker yang salah; drain; audit duplikat di YouTube | Owner; eksekusi segera setelah konfirmasi |
| Live down (broadcast mati, bukan hanya FFmpeg mati) >5 menit | P0 | Langkah darurat §4 | [OD] persetujuan pra-otorisasi belum ada |
| Publish berhenti total (0 PUBLISHED dalam 2 jam pada jam aktif) | P0 | Langkah darurat §4 | [OD] sama seperti di atas |
| DB `integrity_check` gagal pasca-sync/DR | P0 | JANGAN pakai salinan itu; tetap di Master; diagnosis | Owner |
| Harga riil > $34,67/bln (diketahui sebelum/saat pembelian) | P1 | Hentikan pembelian; eskalasi (jangan lanjut diam-diam) | Owner |
| Tailscale putus >90 dtk (heartbeat stale) | P2 | Bukan rollback — stream jalan tetap jalan; penugasan baru tertunda; alarm; perbaiki konektivitas | Operator + lapor owner |

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
