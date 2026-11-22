# MODEL TOTAL COST OF OWNERSHIP (TCO) — MASJAVAS TUBE OS V2

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
tetap harus ditambahkan begitu ada angkanya.

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
