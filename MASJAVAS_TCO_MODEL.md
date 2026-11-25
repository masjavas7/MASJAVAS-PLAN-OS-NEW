# MODEL TOTAL COST OF OWNERSHIP (TCO) — MASJAVAS TUBE OS V2

**Versi:** 1.1 — 25 Nov 2026, 18:50 WIB (Cycle 25: tambah §7 scale-up
pasca-M8; §3 rujuk §7; OQ-TCO-5/6 baru).
**Versi:** 1.0 — 22 Nov 2026, 13:05 WIB (sprint item #7).
**Sumber:** `PLAN_PENAMBAHAN_VPS8_2026-11-22.md` §7,
`MASJAVAS_COST_REGISTER.md`, `ENGINEERING_READINESS_DASHBOARD.md`.
**Status:** MODEL PERENCANAAN — bukan tagihan. Angka yang belum
diverifikasi tetap berlabel demikian; model ini TIDAK membuat
biaya yang belum dihitung menjadi nol.

## Legenda label

- **TERVERIFIKASI** — dibuktikan di sumber primer (panel/invoice/
  halaman pricing) pada tanggal yang dicatat.
- **ESTIMASI** — angka perkiraan; belum verifikasi panel/pricing.
- **DOCUMENTED FACT NOT YET REVALIDATED (DF)** — tercatat di
  dokumen riset; belum diverifikasi ulang.
- **BELUM DIHITUNG** — belum ada angka sama sekali. **Bukan nol
  dan bukan tercakup cap.**
- **PROPOSAL [PR]** — usulan perencanaan model ini, belum diputuskan
  owner.
- **OWNER DECISION** — angka yang ditetapkan user sebagai batas
  mengikat.

**PERINGATAN KURS:** PLAN §7 memakai ±Rp 18.300/USD (implisit:
$34,67 → Rp 634 rb); dokumen BIAYA memakai Rp 18.000/USD. Angka
Rupiah di bawah memakai Rp 18.300/USD (asumsi PLAN §7) kecuali
dinyatakan lain.

## 1. Ruang lingkup: cap ≠ TCO

Cap **$34,67/bln** adalah batas anggaran INFRASTRUKTUR (Master +
VPS 8) yang ditetapkan owner [OWNER DECISION — mengikat, PLAN §7].
TCO adalah biaya PENUH menjalankan sistem: infrastruktur + storage
+ backup + domain + layanan pendukung + API/provider + cadangan.
**Jangan menganggap angka cap otomatis mencakup biaya yang belum
dihitung** — temuan utama model ini adalah cap sudah habis oleh
infrastruktur saja (§4).

**DIPUTUSKAN 22 Nov 13:13 WIB (OQ-TCO-1 tertutup):** cap $34,67/bln
mencakup SELURUH TCO — infrastruktur + storage + backup + domain +
layanan pendukung + API/provider + cadangan. Konsekuensi:
infrastruktur saja sudah $34,67 → storage/backup harus $0 (mis.
free tier object storage untuk DB 118 MB) atau infrastruktur
direvisi; skenario expected + worst case (§3) MELEBIHI cap dan
tidak dapat diterima sebagai rencana sampai ada revisi.

## 2. Komponen biaya (per bulan, kecuali dinyatakan lain)

### 2.1 Infrastruktur — Master + VPS 8

| Item | Pra-pajak | PPN 11% | Incl. pajak | Label |
|---|---|---|---|---|
| Master (Contabo Plus 4) | $14,43 | $1,59 | ±$16,02 | **ESTIMASI** (PLAN §7 — belum verifikasi panel; V2-009) |
| Live-01 (Contabo Core VPS 8, 8 vCPU/24 GB/300 GB/600 Mbit/s) | $16,80 | $1,85 | $18,65 | **DF** (listing PLAN §7; pra-pajak dihitung: 18,65 ÷ 1,11 — belum verifikasi panel) |
| **Subtotal infrastruktur** | **$31,23** | **$3,44** | **$34,67** | **OWNER DECISION — cap mengikat** |
| Setup sekali (VPS 8) | $2,40 | $0,26 | $2,66 | **DF** (PLAN §7; pra-pajak dihitung: 2,66 ÷ 1,11) |

Aritmetika konsisten: $31,23 × 1,11 = $34,67; $14,43 × 1,11 =
$16,02; $16,80 × 1,11 = $18,65. Angka pra-pajak VPS 8 + setup
adalah HASIL HITUNG dari angka incl-tax yang didokumentasikan,
bukan harga panel — labelnya mengikuti sumbernya (DF).

### 2.2 Storage + backup off-site

| Item | Nilai | Label |
|---|---|---|
| Object storage off-site (Restic) | BELUM DIHITUNG | **BELUM DIHITUNG** (V2-009 — provider belum dipilih) |
| Retensi backup (berapa lama salinan disimpan) | BELUM DIHITUNG | **BELUM DIHITUNG** (kebijakan retensi belum diputuskan — OQ) |
| Litestream (replikasi SQLite) | BELUM DIHITUNG | **BELUM DIHITUNG** (V2-009; hanya setelah storage gate lolos) |
| Snapshot VPS (Contabo) | BELUM DIHITUNG | **BELUM DIHITUNG** (harga snapshot tidak tercatat di dokumen mana pun) |

### 2.3 Transfer data + domain + layanan pendukung

| Item | Nilai | Label |
|---|---|---|
| Transfer data / egress (di luar kuota paket, bila ada) | BELUM DIHITUNG | **BELUM DIHITUNG** (kuota trafik paket Contabo tidak tercatat — verifikasi di panel, V2-009) |
| Domain masjavastube.my.id (registrasi + perpanjangan) | BELUM DIHITUNG | **BELUM DIHITUNG** (biaya domain tidak tercatat di dokumen mana pun) |
| Agregator "Offside" (Content ID) | $18/tahun = $1,50/bln | **DF** (MEMORY.md 23 Okt 2026; perlakuan pajak tidak tercatat) |
| Monitoring eksternal (Healthchecks.io / UptimeRobot — tier gratis vs berbayar) | BELUM DIHITUNG | **BELUM DIHITUNG** (pemilihan provider belum diputuskan; tier gratis mungkin cukup — belum diverifikasi) |

### 2.4 API + provider

| Item | Nilai | Label |
|---|---|---|
| KEIN OMNI 1.1 (perpanjangan plan — berakhir 23 Nov 2026) | BELUM DIHITUNG | **BELUM DIHITUNG** (harga perpanjangan tidak tercatat; top-up ditunda user sejak 12 Nov; D1 DITUTUP) |
| Google Omni API Generate Video (Unlimited 1 bln + bonus API Image) | Rp 600.000/bln | **DF** (penawaran 21 Nov 2026 — belum ada keputusan pembelian) |
| Image-gen LoopGen 700 ch — ModelsLab flat (jalur termurah yang layak) | $149/bln (±Rp 2,7 jt) | **DF — HISTORICAL** (riset 15 Nov). **BUKAN rekomendasi beli:** aturan user 16 Nov — studi WAJIB free, berbayar DITOLAK. Dicatat sebagai riset harga saja |
| Image-gen LoopGen 700 ch — Kie.ai Seedream 5.5 (rekomendasi dokumen BIAYA) | Rp 7,8 jt (sekali, +buffer redo) | **DF — HISTORICAL** (riset 15 Nov). Sama seperti di atas — bukan rekomendasi beli |
| WaroengProxy (alternatif D1, bila diuji) | Rp 10.000/GB | **ESTIMASI** (belum diuji; D3 MENUNGGU — jangan tanya ulang) |
| DeepInfra pilot 3 varian | Rp 800 (sekali) | **PROPOSAL** (D2 MENUNGGU, ditawarkan 18 Nov) |

### 2.5 Cadangan biaya tak terduga

| Item | Nilai | Label |
|---|---|---|
| Cadangan (contingency) — usul 10% dari subtotal infrastruktur bulanan | $3,47/bln | **[PR]** PROPOSAL model ini — belum diputuskan owner. Bukan harga, melainkan asumsi perencanaan. Bila owner menolak angka ini, cadangan = BELUM DIPUTUSKAN (bukan nol) |

### 2.6 Di luar cap (persetujuan anggaran terpisah)

| Item | Nilai | Label |
|---|---|---|
| Box live ke-2 (Fase 2, bila VPS 8 rapuh) | BELUM DIHITUNG | **BELUM DIHITUNG** (di luar anggaran $34,67/bln; butuh persetujuan terpisah) |
| Kontrak 12 bulan VPS 8 (opsi hemat — BUKAN biaya tambahan, melainkan alternatif pembayaran) | VPS 8 $15,85/bln; total $31,87/bln; bayar di muka $190,21 | **ESTIMASI** (PLAN §7; $190,21 ÷ 12 = $15,85/bln — konsisten. Putuskan setelah model terbukti; bulanan dulu — KONTRAK-12 MENUNGGU) |
| Server lama Tencent | $14,40/bln — BERAKHIR 21 Nov 2026 | **DF** (tidak lagi dibayar; layanan aplikasi dihentikan 22 Nov 07:05) |

## 3. Tiga skenario vs cap $34,67/bln

Semua skenario memakai harga bulanan (bukan kontrak 12 bulan)
kecuali dinyatakan lain. "Yang diketahui" = jumlah komponen yang
punya angka; "yang belum diketahui" = komponen BELUM DIHITUNG yang
tetap harus ditambahkan begitu ada angkanya. Skenario di bawah
adalah 2-box (Master + 1 VPS 8); untuk biaya di atas 2 box
(scale-up live), lihat §7.

### Skenario MINIMUM — dua box saja

| Komponen | Per bulan | Per tahun |
|---|---|---|
| Master + VPS 8 | $34,67 | $416,04 |
| Setup sekali VPS 8 | — | $2,66 |
| **Total minimum** | **$34,67/bln** | **$418,70/thn** |
| **vs cap $34,67/bln** | **PAS di cap — headroom $0** | — |

Skenario ini TIDAK mencakup storage, backup, domain, monitoring,
API, atau cadangan apa pun. Ini adalah lantai absolut — biaya
paling kecil yang mungkin untuk menjalankan dua box.

### Skenario EXPECTED — dua box + layanan pendukung yang sudah diketahui

| Komponen | Per bulan | Per tahun |
|---|---|---|
| Master + VPS 8 | $34,67 | $416,04 |
| Agregator Offside | $1,50 | $18,00 |
| Setup sekali VPS 8 | — | $2,66 |
| Object storage + retensi + snapshot + egress + domain + monitoring | BELUM DIHITUNG | BELUM DIHITUNG |
| **Total expected (yang diketahui)** | **$36,17/bln** | **$436,70/thn** |
| **vs cap $34,67/bln** | **LEBIH $1,50/bln — SEBELUM menghitung storage/domain** | — |

### Skenario WORST CASE — expected + cadangan + kenaikan yang belum diketahui

| Komponen | Per bulan | Per tahun |
|---|---|---|
| Expected (yang diketahui) | $36,17 | $434,04 |
| Cadangan 10% [PR] | $3,47 | $41,64 |
| Setup sekali VPS 8 | — | $2,66 |
| Kenaikan harga renewal (BELUM DIHITUNG) + pertumbuhan storage (BELUM DIHITUNG) + perpanjangan KEIN (BELUM DIHITUNG) | BELUM DIHITUNG | BELUM DIHITUNG |
| **Total worst case (yang diketahui)** | **$39,64/bln** | **$478,34/thn** |
| **vs cap $34,67/bln** | **LEBIH $4,97/bln — SEBELUM menghitung yang belum diketahui** | — |

Box live ke-2 (Fase 2) TIDAK termasuk di skenario mana pun —
persetujuan anggaran terpisah, di luar cap.

### Sensitivitas: kontrak 12 bulan

Bila kontrak 12 bulan VPS 8 diambil (total $31,87/bln), skenario
EXPECTED yang diketahui menjadi $31,87 + $1,50 = **$33,37/bln**
— DI BAWAH cap sebesar $1,30/bln — tetapi storage/domain tetap
BELUM DIHITUNG dan bisa mendorong total kembali di atas cap.
Kontrak mengunci harga 12 bulan dan butuh bayar di muka $190,21;
keputusan KONTRAK-12 MENUNGGU (putuskan setelah model terbukti).
Sensitivitas ini BUKAN rekomendasi mengambil kontrak.

## 4. Temuan utama

1. **Cap habis oleh infrastruktur saja.** Subtotal Master + VPS 8
   = $34,67/bln — tepat sama dengan cap. Headroom anggaran $0.
   Setiap biaya tambahan apa pun (storage, domain, API, cadangan)
   berada DI ATAS cap, bukan di dalamnya.
2. **Skenario expected sudah melampaui cap pada biaya yang
   diketahui saja** (+$1,50/bln dari Offside) — sebelum storage
   dan domain yang belum dihitung. Ini bukan kesalahan hitung:
   cap ditetapkan untuk infrastruktur, sedangkan TCO mencakup
   seluruh sistem. Owner perlu memutuskan: (a) cap $34,67 hanya
   untuk infrastruktur (dan biaya lain dianggarkan terpisah),
   atau (b) cap adalah total TCO (maka pembelian VPS 8 bulanan
   tidak muat bersama storage — perlu kontrak 12 bulan dan/atau
   pengurangan cakupan). **[OD — OWNER DECISION REQUIRED]**
3. **Gerbang pembelian (V2-009) tidak bisa lolos sebelum storage
   dihitung.** Komponen BELUM DIHITUNG terbesar adalah object
   storage + retensi + snapshot — tanpa angkanya, total TCO
   tidak dapat dibandingkan dengan cap secara jujur.
4. **Harga renewal belum terverifikasi.** Angka $18,65/bln adalah
   listing (DF), bukan harga renewal yang dikonfirmasi panel.
   Bila harga renewal naik, skenario minimum pun melampaui cap.
5. **Tidak ada biaya yang diasumsikan nol.** Setiap komponen
   BELUM DIHITUNG dicatat sebagai utang perhitungan, bukan
   sebagai $0. Total TCO yang sebenarnya = total yang diketahui
   + semua komponen BELUM DIHITUNG (yang baru diketahui setelah
   V2-009).

## 5. Daftar verifikasi V2-009 (untuk menutup model ini)

1. Harga Master di panel Contabo (pra-pajak + incl-tax).
2. Harga VPS 8 di panel Contabo (listing + renewal).
3. Kuota trafik paket Contabo (apakah egress live 310 Mbps
   masuk kuota atau berbayar).
4. Provider + harga object storage off-site (per GB/bulan).
5. Kebijakan retensi backup + biaya retensi.
6. Harga snapshot VPS Contabo.
7. Biaya domain masjavastube.my.id (registrasi + perpanjangan).
8. Tier monitoring eksternal (gratis cukup atau berbayar).
9. Keputusan owner soal cakupan cap (§4 temuan #2) — DIPUTUSKAN
   22 Nov 13:13 WIB: cap mencakup SELURUH TCO.
10. Keputusan owner soal cadangan 10% [PR] (terima / ubah / tolak).
11. Rekonsiliasi §7 scale-up pasca-M8 — SELESAI Cycle 25
    (25 Nov 18:50 WIB); hitung ulang bila OQ-TCO-5/6 terjawab.

## 6. Pertanyaan terbuka

1. **[OQ-TCO-1 — DIPUTUSKAN 22 Nov 13:13 WIB]** Cap $34,67/bln
   mencakup SELURUH TCO (infrastruktur + storage + backup +
   domain + layanan). Infrastruktur saja sudah $34,67 →
   storage/backup harus $0 atau infrastruktur direvisi.
2. **[OQ-TCO-2]** Komposisi $190,21 bayar di muka kontrak 12 bulan
   — apakah itu murni 12× $15,85 VPS 8, atau termasuk komponen
   lain? (Aritmetika konsisten dengan 12× $15,85, tetapi
   komposisi tidak didokumentasikan.)
3. **[OQ-TCO-3]** Berapa volume data backup yang harus disimpan
   (DB 118 MB + file staging + salinan DR) — menentukan harga
   storage begitu provider dipilih.
4. **[OQ-TCO-4]** Harga perpanjangan KEIN dan harga paket Dahono
   tidak tercatat di dokumen mana pun — apakah perlu dicari
   sebelum V2-009, atau tetap BELUM DIHITUNG karena D1 DITUTUP?

## 7. Scale-up live pasca-M8 — biaya per milestone stream (REKONSILIASI Cycle 25)

**Status bagian ini:** REKONSILIASI — menggantikan angka scale-up
lama di dokumen program (RENCANA FINAL §4 + capacity model §3.1/
§7.2) dengan matematika diagnosis M8. Angka di bawah adalah
INFRASTRUKTUR SAJA (box live + Master); per §1 cap = seluruh TCO,
jadi setiap milestone di atas 1 box sudah DI ATAS cap sebelum
storage/domain dihitung — angka di bawah adalah LANTAI, bukan TCO
penuh. Storage/domain per box tambahan BELUM DIHITUNG.

### 7.1 Asumsi (sumber: `DIAGNOSIS_M8_2026-11-25.md`)

| Asumsi | Nilai | Label |
|---|---|---|
| Biaya per stream, AAC-copy (steady-state, NO 88) | 0,175 core (feeder 0,055 + encoder 0,120) | DOCUMENTED (30 interval/33,5 mnt, delta-ticks /proc; M8 §1 + §4) |
| Mean armada era AAC-copy (estimasi, bila semua stream pindah) | ~0,18–0,20 core/stream | DOCUMENTED (M8 §4.1 — 088 adalah kasus termurah; BUKAN angka terukur armada) |
| Supervisor live per box | 0,18 core | DOCUMENTED (M8 §1.1) |
| Kapasitas terpakai per box | 3,0–3,5 core (70% dari 4,3–5,0 core/dtk yang terhantar) | DOCUMENTED (M8 §5–§6; rasio penghantaran host 0,53–0,62, bisa berubah menurut waktu/host — verifikasi independen disarankan sebelum jadi konstanta) |
| Harga box live (VPS 8, bulanan) | $18,65/bln incl. pajak | DF (listing; belum verifikasi panel — V2-009 F.1) |
| Harga Master | $16,02/bln incl. pajak | ESTIMASI (belum verifikasi panel) |
| Ambang M8 0,15 core/stream | BELUM diputuskan (D34 MENUNGGU) | OWNER DECISION REQUIRED — anggaran realistis per M8 §6(b) adalah ~0,17–0,18 |

### 7.2 Tabel milestone (biaya infrastruktur saja)

| Milestone | Core feeder+encoder | All-in (+supervisor per box) | Box live | $/bln (box live + Master $16,02) | vs cap $34,67 |
|---|---|---|---|---|---|
| **16–19 stream (1 box, dalam cap)** | 2,8–3,3 | 3,0–3,5 | **1** | **$34,67 (cap)** | **PAS** |
| 50 stream | 8,75 | 9,3 | **3 (ketat) / 4 nyaman** | **$71,97** / $90,62 | LEBIH $37,30 / $55,95 |
| 100 stream | 17,5 | 18,6 | **6** | **$127,92** | LEBIH $93,25 |
| 696 stream | 121,8 | 129,4 | **35–41** | **$668–$781** | LEBIH $634–$746 |

Perhitungan: $/bln = $16,02 + N × $18,65 (N = jumlah box live).
3 box: 16,02 + 55,95 = 71,97; 4 box: 16,02 + 74,60 = 90,62;
6 box: 16,02 + 111,90 = 127,92; 35 box: 16,02 + 652,75 = 668,77;
41 box: 16,02 + 764,65 = 780,67. Tabel ini menyalin
`DIAGNOSIS_M8_2026-11-25.md` §6(b) (supervisor dihitung per box).

### 7.3 Yang SUPERSEDED oleh tabel ini

- RENCANA FINAL §4 (D28, 25 Nov 09:24): "100 stream = 2 box
  ≈ $53,32; cap cukup sampai 50 stream (bila A2 ≤0,07)" —
  **SUPERSEDED**. Asumsi dasarnya (kapasitas efektif 5,6 core/box;
  A2 ≤0,07 core/stream) gugur oleh M8: A2 terukur 0,1746 (bukan
  ≤0,07) dan kapasitas efektif terukur 3,0–3,5 core/box (bukan 5,6).
  Bahkan pada 0,15 pun 50 stream butuh 3 box ($71,97); pada 0,07
  pun butuh 2 box ($53,32).
- Capacity model §3.1 + §7.2 (v1.0): "Live 50 stream = 2,0 core;
  total puncak ≈7,4 core; headroom ±0,6" — **SUPERSEDED** (angka
  live 50 stream yang benar ≈9,3 core all-in; 50 stream + prep
  penuh TIDAK muat di 1 box — melebihi kapasitas terpakai 3,0–3,5).
  Lihat KOREKSI M8 di dokumen tersebut (Cycle 25).
- Angka "696 pasca-A2 = 12–13 box ≈ $224–243" (RENCANA FINAL
  changelog #2) — **SUPERSEDED** oleh 35–41 box $668–$781.
- Milestone "50 stream stabil dalam cap" (RENCANA FINAL §5) —
  **TIDAK BERLAKU** pada efisiensi sekarang. Target 50-dalam-cap
  butuh BOTH: per-stream ≤ ~0,07 core DAN kapasitas efektif
  ≥ ~3,7 core/box — keduanya belum terbukti (M8 §6(b)). JANGAN
  janjikan.

### 7.4 Sensitivitas: apa yang bisa mengubah tabel ini

- **Opsi 1 (hapus feeder; encoder baca prebuilt langsung):**
  harapan −0,055/stream → ~0,12. Pada 0,12: 50 stream = 6,0 +
  0,36–0,54 = 6,36–6,54 core → 2 box (bila kapasitas 3,5) atau
  3 box (bila 3,0) → $53,32–$71,97/bln — MASIH di atas cap.
  Opsi 1 TIDAK mengembalikan 50-dalam-cap; ia hanya menurunkan
  3 box → 2–3 box. Status: koordinator Opsi 5+1 berjalan (D33);
  Opsi 1 belum ada bukti jalan (Cycle 24).
  [PROPOSAL — aritmetika dari asumsi M8, bukan pengukuran baru]
- **Opsi 5 (uji diskriminan klon encoder 60 dtk):** SELESAI
  14:53:48–14:54:48, 0,1291 (ENC 0,0672 + FEED 0,0619) — uji
  60 detik, BUKAN angka produksi; jangan dipakai sebagai
  per-stream produksi. [DOCUMENTED — Gelombang 1, Cycle 24]
- **Revisi ambang D34 (0,15 → ~0,17–0,18):** mengubah ambang
  GO/NO-GO Fase 2, TIDAK mengubah jumlah box di tabel ini
  (tabel memakai 0,175 terukur). [OWNER DECISION REQUIRED]
- **Mean armada 0,18–0,20 (bukan 0,175):** 50 stream tetap 3–4
  box (pada 0,20: 10,0 + 0,54 = 10,54 core → 4 box pada
  kapasitas 3,0); 100 stream tetap 6–7 box. Rentang tabel di
  atas adalah kasus optimistis (0,175).
  [PROPOSAL — aritmetika]
- **Rasio penghantaran host (0,53–0,62):** bila verifikasi
  independen (grafik Contabo / uji beban box kosong) menunjukkan
  rasio berbeda, kapasitas terpakai 3,0–3,5 ikut berubah — tabel
  ini harus dihitung ulang. [OPEN QUESTION — OQ-TCO-6]

### 7.5 Konsekuensi untuk skenario §3

Skenario MINIMUM/EXPECTED/WORST di §3 adalah 2-box (Master +
1 VPS 8) dan TIDAK berubah. Scale-up di §7 adalah biaya TAMBAHAN
di luar skenario itu: setiap box live tambahan = +$18,65/bln
infrastruktur saja, sebelum storage/domain per box (BELUM
DIHITUNG). Karena cap = seluruh TCO (§1), milestone 50 stream
($71,97 infra saja) sudah melampaui cap sebesar $37,30/bln
SEBELUM satu dolar pun storage/domain — keputusan scale-up
adalah keputusan anggaran baru, bukan optimasi dalam cap.

### 7.6 Pertanyaan terbuka baru

5. **[OQ-TCO-5 — BARU Cycle 25]** Apakah hasil Opsi 1 / Opsi 5
   mengubah lantai per-stream 0,175? (Jawaban menunggu koordinator
   Opsi 5+1; bila Opsi 1 lolos ≤0,15, hitung ulang §7.2.)
6. **[OQ-TCO-6 — BARU Cycle 25]** Berapa rasio penghantaran host
   VPS 8 yang sebenarnya dalam jangka panjang (0,53–0,62 terukur
   1 jendela 33,5 mnt — butuh verifikasi independen via grafik
   Contabo / uji beban box kosong, BUKAN VPS 8 live)?
