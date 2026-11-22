# MASJAVAS COST REGISTER — MASJAVAS TUBE OS V2

**Versi:** 1.0 — 22 Nov 2026, 12:35 WIB.
**Sumber:** `PLAN_PENAMBAHAN_VPS8_2026-11-22.md` (§7),
`docs/BIAYA_GENERATE_IMAGE_700_1000_CHANNEL.md` (riset 15 Nov 2026),
`ENGINEERING_READINESS_DASHBOARD.md` (§ cost register awal),
MEMORY.md (item yang hanya tercatat di sana — dilabeli).

## Legenda label harga

- **TERVERIFIKASI** — harga dibuktikan di sumber primer (panel/invoice/
  halaman pricing) pada tanggal yang dicatat.
- **ESTIMASI** — angka perkiraan; belum verifikasi panel/pricing.
- **DOCUMENTED FACT NOT YET REVALIDATED** — tercatat di dokumen riset/
  memori; belum diverifikasi ulang (harga berubah cepat — cek sebelum bayar).
- **BELUM DIHITUNG** — belum ada angka sama sekali.
- **OWNER DECISION** — angka yang ditetapkan user sebagai batas mengikat.

**PERINGATAN KURS:** PLAN §7 memakai ±Rp 18.300/USD (implisit: $34,67 →
Rp 634 rb), sedangkan BIAYA doc memakai Rp 18.000/USD. Selisih kecil
(±1,7%) — angka Rupiah di bawah memakai asumsi dokumen sumbernya
masing-masing.

## 1. Infrastruktur — Master + VPS 8 (mengikat)

| Item | Nilai | Label | Sumber / catatan |
|---|---|---|---|
| Master Contabo Plus 4 | ±$16,02/bln (pra-pajak ±$14,43) | **ESTIMASI** | PLAN §7. **Harga Master ±$14,43 pra-pajak adalah ESTIMASI — belum verifikasi panel Contabo.** Verifikasi wajib sebelum pembelian VPS 8 (V2-009) |
| VPS 8 Contabo Core (8 vCPU / 24 GB / 300 GB / 600 Mbit/s) | $18,65/bln + $2,66 setup sekali | DOCUMENTED FACT NOT YET REVALIDATED | PLAN §7 (listing). Belum verifikasi panel (Turnstile blokir browser cloud — butuh user) |
| **TOTAL (Master + VPS 8)** | **$34,67/bln (±Rp 634 rb)** | **OWNER DECISION — mengikat** | Ditetapkan user 21–22 Nov 2026. Bila total riil > $34,67: eskalasi, jangan lanjut diam-diam |
| Setup sekali VPS 8 | $2,66 (±Rp 49 rb) | DOCUMENTED FACT NOT YET REVALIDATED | PLAN §7 |
| Tambahan bersih vs hari ini | +$4,25/bln | DOCUMENTED FACT NOT YET REVALIDATED | Tencent $14,40/bln berakhir 21 Nov 2026 |
| Server lama Tencent (berakhir) | $14,40/bln | DOCUMENTED FACT NOT YET REVALIDATED | Berakhir 21 Nov 2026; layanan aplikasi dihentikan 22 Nov 07:05 (D7) |
| Kontrak 12 bulan VPS 8 (opsi hemat) | Total $31,87/bln; bayar di muka $190,21 | ESTIMASI | PLAN §7. Putuskan setelah model terbukti (bulanan dulu) — KONTRAK-12 MENUNGGU |
| Alternatif ditolak: Plus 4 → Plus 8 | +€21,50/bln | DOCUMENTED FACT NOT YET REVALIDATED | PLAN §7. Ditolak: lebih mahal, tanpa isolasi; Contabo tidak menjual add-on port |
| Alternatif 720p: VPS 6 untuk 50 channel | Total $26,01/bln | ESTIMASI | Trade-off 720p (TR-720p MENUNGGU). VPS 8 muat 134 stream @720p vs 70 @1080p. 1080p tetap keputusan berlaku |
| Box live ke-2 (Fase 2, bila VPS 8 rapuh) | BELUM DIHITUNG | BELUM DIHITUNG | Di luar anggaran $34,67/bln; butuh persetujuan anggaran terpisah |

## 2. Storage + backup off-site

| Item | Nilai | Label | Sumber / catatan |
|---|---|---|---|
| Object storage off-site (Restic) | BELUM DIHITUNG | **BELUM DIHITUNG** | V2-009. VPS 8 BUKAN satu-satunya lokasi backup — backup off-site terenkripsi independen wajib (V2.0) |
| Litestream (replikasi SQLite) | BELUM DIHITUNG | **BELUM DIHITUNG** | V2-009. Hanya setelah storage gate lolos |
| Provider + retensi storage | Belum diputuskan | OWNER DECISION REQUIRED | STORAGE MENUNGGU. RPO ≤5 mnt / RTO ≤30 mnt adalah TARGET (buktikan via restore drill), bukan janji |

## 3. Generate image LoopGen (riset 15 Nov 2026 — HISTORICAL)

Asumsi: 30 gambar/channel → 700 ch = 21.000 gambar; 1.000 ch = 30.000
gambar. Kurs Rp 18.000/USD. Semua harga = **DOCUMENTED FACT NOT YET
REVALIDATED** (riset web Nov 2026 — verifikasi di halaman pricing
masing-masing sebelum top-up besar). Total di bawah SUDAH termasuk buffer
redo 15% (gambar gagal QA), kecuali dinyatakan lain.

| Jalur | Per gambar | 700 ch (21.000 gbr) | 1.000 ch (30.000 gbr) | Catatan |
|---|---|---|---|---|
| Seedream 5.5 via Kie.ai (1,5K) — REKOMENDASI UTAMA | $0,018 (~Rp 324) | Rp 7,8 jt | Rp 11,2 jt | PAYG, kredit tidak hangus, gagal generate tidak ditagih; 80 kredit gratis uji coba; tanpa buffer: Rp 6,8 jt / Rp 9,7 jt |
| Seedream 5.5 via Kie.ai (2K) | $0,0315 | Rp 13,7 jt | Rp 19,6 jt | Tanpa buffer: Rp 11,9 jt / Rp 17,0 jt |
| Seedream 5.5 via WaveSpeed (1K) | $0,02 (~Rp 360) | — | — | Terima PayPal; kredit $1 gratis |
| FLUX.2 Pro (BFL/fal, 1080p) | $0,045 | Rp 19,6 jt | Rp 27,9 jt | 1MP = $0,03. Fotorealisme terbaik (LM Arena Elo 1.265) |
| GPT Image 2.6 (APIMODELS, 1K) | $0,02 (~Rp 360) | Rp 8,7 jt | Rp 12,4 jt | Teks/grafis terbaik |
| DeepInfra FLUX | Rp 35,44/gambar | — | — | Dashboard (HISTORICAL 18 Nov). Pilot 3 varian Rp 800 — D2 MENUNGGU |
| Muse Image | Rp 180/gambar | — | — | Dashboard (HISTORICAL 18 Nov) |
| ModelsLab Unlimited (FLUX/SDXL open-source) | Flat $149/bln (~Rp 2,7 jt) | Rp 2,7 jt | Rp 2,7 jt | Tanpa batas per gambar; reseller; rate limit belum terverifikasi; perlu pilot kualitas + throughput |
| Self-host FLUX di GPU Dahono A40 | ~Rp 70/gambar | ~Rp 1,5 jt (58 jam GPU) | ~Rp 2,1 jt (83 jam GPU) | Rp 25.000/jam GPU; asumsi 10 dtk/gambar; BUTUH bangun inferensi lokal (proyek build, bukan langganan) |
| Nano Banana 2.1 / Gemini (1K) | $0,0336 (~Rp 605) | Rp 12,7 jt | Rp 18,1 jt | **TIDAK BISA DIPAKAI** — billing Gemini dibatalkan user 14 Nov |
| Nano Banana Pro (1K/2K) | $0,134 (~Rp 2.412) | — | — | Mahal; tidak disarankan untuk volume |
| KEIN IMAGE (kein_image) | **TIDAK DIKETAHUI** | — | — | Sudah LIVE di aplikasi; serial 78,7 dtk/gambar (700 ch = 19,1 hari nonstop); retensi output 1 jam |

**Tidak cocok (tercatat agar tidak diangkat lagi):** Midjourney ($10/bln,
tanpa API publik — Discord saja); Sumopod/Dahono AI (model teks saja);
Adobe Firefly ($9,99/bln, bukan untuk volume API massal).

## 4. Proxy, video API, dan layanan lain

| Item | Nilai | Label | Sumber / catatan |
|---|---|---|---|
| WaroengProxy (alternatif D1) | Rp 10.000/GB | ESTIMASI | Termurah hasil riset; BELUM DIUJI. D3 MENUNGGU (jangan tanya ulang) |
| Dahono residential Starter 1 GB | Harga tidak tercatat | BELUM DIHITUNG | Paket AKTIF (sisa 0,89 GB) tapi gateway 407 — terblokir sisi provider |
| DeepInfra pilot 3 varian | Rp 800 (sekali) | PROPOSAL | D2 MENUNGGU (ditawarkan 18 Nov) |
| Google Omni API Generate Video (Unlimited 1 bln, maks 1 paralel, + bonus API Image Unlimited) | Rp 600.000/bln | DOCUMENTED FACT NOT YET REVALIDATED | Penawaran 21 Nov 2026, belum ada keputusan pembelian (MEMORY.md). Throughput ±5 mnt/video → ±240–270 video/hari realistis |
| Agregator "Offside" (Content ID) | $18/tahun | DOCUMENTED FACT NOT YET REVALIDATED | MEMORY.md 23 Okt 2026. Musik Suno Pro didaftarkan ke agregator ini |
| KEIN OMNI 1.1 (plan 24 Okt–23 Nov 2026) | Biaya perpanjangan TIDAK TERCATAT | BELUM DIHITUNG | Kuota 5,550; berakhir 23 Nov 2026. Top-up ditunda user sejak 12 Nov; D1 DITUTUP ("ASUMSIKAN KEIN AMAN") |

## 5. Ringkasan komitmen biaya

| Komitmen | Nilai | Status |
|---|---|---|
| Cap total infrastruktur | $34,67/bln | OWNER DECISION — mengikat |
| Pengeluaran berjalan hari ini (Master saja, estimasi) | ±$16,02/bln | ESTIMASI (harga Master belum verifikasi panel) |
| Tambahan bila VPS 8 dibeli | +$18,65/bln + $2,66 setup | Menunggu gerbang D4 |
| Storage/backup off-site | BELUM DIHITUNG | V2-009 — blokir gerbang pembelian |
| Image-gen 700 channel (jalur termurah yang layak: ModelsLab flat) | ±Rp 2,7 jt/bln | HISTORICAL — belum diputuskan; aturan user: studi WAJIB free, berbayar DITOLAK (16 Nov) — semua jalur §3 di atas adalah berbayar |
| Image-gen 700 channel (rekomendasi BIAYA doc: Kie.ai Seedream) | Rp 7,8 jt (sekali, +buffer) | HISTORICAL — rekomendasi dokumen, belum diputuskan user |

**Catatan aturan user (16 Nov 2026, DISETUJUI):** studi skill/provider
WAJIB SEMUA FREE; opsi berbayar DITOLAK. Semua jalur image-gen §3 adalah
berbayar — dicatat sebagai riset harga, BUKAN rekomendasi pembelian.
Keputusan pembelian apa pun tetap butuh persetujuan eksplisit user (Level C).
