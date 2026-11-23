# MASJAVAS OWNER DECISION REGISTER — MASJAVAS TUBE OS V2

**Versi:** 1.1 — 23 Nov 2026, 14:50 WIB (Cycle 12: D4 DIPUTUSKAN +
dieksekusi owner 14:41; D12–D14 baru dari RENCANA_KEDEPAN §6; pemetaan
nomor RENCANA D1–D6 dicatat — JANGAN disamakan dengan nomor register ini).
**Sumber:** `ENGINEERING_READINESS_DASHBOARD.md` (§ owner decision register),
`PLAN_PENAMBAHAN_VPS8_2026-11-22.md` (§10),
`EVALUASI_SEBELUM_PERBAIKAN_2026-11-22.md` (§6), temuan V2-001–V2-003,
MEMORY.md / ALIGNMENT_SYNTHESIS.md (untuk item yang hanya tercatat di sana —
dilabeli).

## Legenda

- **DIPUTUSKAN** — user menyetujui; pelaksanaan mungkin sudah selesai atau
  masih berjalan (lihat kolom Bukti).
- **MENUNGGU** — belum ada keputusan user. Jangan tanya ulang di luar
  register ini kecuali ada pemicu baru (aturan: keputusan yang berulang
  tidak dijawab tidak ditanyakan lagi secara proaktif).
- **DITUTUP** — tidak diputuskan secara substantif, tetapi ditutup atas
  arahan user (mis. asumsi).
- **DITOLAK** — user menolak usulan secara eksplisit.
- **VERIFIED FACT** — ada bukti di dokumen/log yang dikutip.
  **DOCUMENTED FACT NOT YET REVALIDATED** — tercatat di memori/dokumen lain,
  belum diverifikasi ulang di program ini.

## 1. Keputusan D1–D11 (dari dashboard)

| ID | Keputusan | Status | Tanggal | Bukti / dokumen |
|---|---|---|---|---|
| D1 | Provider KEIN: top-up / fallback / HOLD | DITUTUP | 22 Nov 06:55 WIB | User: "ASUMSIKAN KEIN AMAN"; konfirmasi 07:03 "lanjut V2-003, KEIN tetap dilewati". KOREKSI: bukti V2-001 dari host salah — asumsi tanpa bukti live (R15). Expiry 23 Nov 2026 |
| D2 | Pilot DeepInfra 3 varian (Rp 800) | MENUNGGU | Ditawarkan 18 Nov 2026 | Belum dijawab user. Satu-satunya fallback image-gen yang sudah dihargai per-varian (Rp 35,44/gambar FLUX) |
| D3 | D1 proxy residential (WaroengProxy Rp 10.000/GB, uji 1 GB) | MENUNGGU | Ditunda user 20 Nov 2026 | **Jangan tanya ulang** (aturan eksplisit). Dahono 407 terbukti sisi provider (21 Nov). Alternatif belum diuji |
| D4 | Pembelian VPS 8 (cap $34,67/bln mengikat) | DIPUTUSKAN + DIEKSEKUSI | Dibeli owner 23 Nov 2026 14:41 WIB | Screenshot panel Contabo: `vmi3778263` — Cloud VPS 8 (2026), Running, EU/Hub Europe, no setup. Gerbang pembelian 06:31 SUPERSEDED oleh keputusan langsung owner ("lanjut fase berikutnya tanpa menunggu"). Prompt implementasi Fase 1: `PROMPT_ANTIGRAVITY_IMPLEMENTASI_VPS8_2026-11-23.md` (14:52; canary 706 dulu, 114/707 SETELAH 15:00; JANGAN sentuh Master 15:00–15:15). IP publik TIDAK dicatat di dokumen program (lihat MEMORY.md). Verifikasi harga aktual panel + storage $0 tetap terbuka (V2-009, kini verifikasi pasca-beli) |
| D12 | Nasib slot 698 — dikembalikan (buat task live baru) atau tidak? | MENUNGGU | — | MONA menghapus 698 (dikonfirmasi user 09:11, "mona yang hapus"); guardian loop mati sejak restart live 08:48; channel vault-ready (vault MV-E02321CFBE9C3F73); keputusan MONA (RENCANA §6 D2 — nomor RENCANA, bukan register) |
| D13 | Jadwal restart service untuk mengaktifkan `cd4fb7a` (perbaikan schema lock) | MENUNGGU | — | Perbaikan ada di disk tapi belum berjalan (restart live 13:50:05 + web 14:00:57 sebelum commit 14:02:53); tenggat 24 jam (RENCANA §6 D3 — nomor RENCANA, bukan register) |
| D14 | Fase 2 Antigravity (bitrate conditioner + `assigned_node_id`) | MENUNGGU | — | Antigravity mengklaim "siap Fase 2"; owner belum menyetujui; persetujuan terpisah; setelah Tahap A (RENCANA §6 D4 — nomor RENCANA, bukan register) |
| D5 | Bundle PLAN §10 #4–#7: sweeper batch, ambang TX-throttle, failover live auto/manual, kepemilikan janitor, limit live final, provider Master pasca-Des | MENUNGGU | — | Rincian per item di §2 (PLAN10-4 s.d. PLAN10-7) |
| D6 | Aktivasi distribusi Stage 2 live (guard 422) + peluncuran batch massal | MENUNGGU | — | Peluncuran = keputusan user semata. CATATAN: distribusi live Stage 2 tercatat AKTIF sejak 19 Nov 11:08 (DOCUMENTED FACT NOT YET REVALIDATED — ALIGNMENT_SYNTHESIS); keputusan guard-422 perlu re-review |
| D7 | Hentikan layanan aplikasi di box lama Tencent (V2-002 R1) | DIPUTUSKAN | Disetujui + dieksekusi 22 Nov 07:05 WIB | Web lama stopped+disabled; port 8080 tutup; DB statis terverifikasi 07:09. Split-brain berakhir (R14 RESOLVED) |
| D8 | Perbaikan encoder 114 (V2-002 R3–R8: relay audio, admission TX, cleanup relay, exit code, alert kronis, adopsi orphan) | DIPUTUSKAN | Disetujui 07:05 WIB; deployed + terverifikasi 07:24 WIB; iterasi 2 ~08:20 WIB | Commit 56f7a6d + 021a9f0 (tes 20/20); observasi 24 jam. Laporan: `v2-002/ENCODER114_FIX_REPORT.md` |
| D9 | Perbaikan Guard v2 (V2-003-R1–R9; prioritaskan R1/R2 admission TX) | MENUNGGU | — | `v2-003/GUARD_RECOMMENDATIONS_V2-003.md`. Prasyarat R1 = D10 |
| D10 | OQ-010 — siapa men-set `LIVE_GUARD_V2_ENABLED=false` + CPU 90 di `.env`, dan mengapa | MENUNGGU | — | `.env` diubah 21 Nov 17:58; alasan tidak diketahui. Prasyarat V2-003-R1 (aktifkan admission governor) |
| D11 | Restart manual stream 706 untuk memulihkan audio (orphan tanpa feeder — R19) | DIPUTUSKAN | Disetujui + dieksekusi 22 Nov 08:10 WIB | Downtime ±48 dtk; audio terverifikasi mengalir (feeder 19,4 MB). R19 RESOLVED |

**CATATAN PENOMORAN (Cycle 12):** `RENCANA_KEDEPAN_PASCA_PERBAIKAN_2026-11-23.md`
§6 memakai nomor D1–D6 sendiri — JANGAN disamakan dengan nomor register ini.
Pemetaan: RENCANA-D1 = D4 (beli VPS 8 — DIPUTUSKAN 14:41); RENCANA-D2 = D12;
RENCANA-D3 = D13; RENCANA-D4 = D14; RENCANA-D5 = D9; RENCANA-D6 = STORAGE
(§4 di bawah, masih MENUNGGU).

## 2. Keputusan PLAN §10 (plan penambahan VPS 8)

| ID | Keputusan | Status | Tanggal | Bukti / dokumen |
|---|---|---|---|---|
| PLAN10-1 | Setuju plan ini sebagai plan of record (menggantikan rancangan 2 VPS)? | MENUNGGU | — | Belum ada catatan persetujuan eksplisit di dokumen yang dibaca. Catatan: V2.0 memakai plan ini sebagai acuan |
| PLAN10-2 | Mulai Fase 0 sekarang (semua $0; 6 langkah §6)? | MENUNGGU | — | Fase 0 = Master saja, tanpa pembelian. Termasuk pemindahan job berat keluar dari web (P15) |
| PLAN10-3 | Pemicu pembelian VPS 8: ambang 12–16 stream (atau nilai lain)? | MENUNGGU | — | Terkait D4. Demand saat ini: 2–3 stream jalan; 6 stream key |
| PLAN10-4 | Nilai operasional: sweeper batch (usul 5), ambang TX-throttle prep (usul jeda saat TX live >50%), failover live otomatis vs manual | MENUNGGU | — | Bagian dari D5 |
| PLAN10-5 | Kepemilikan janitor: ikut unit publish (usul) atau layanan sendiri? | MENUNGGU | — | Bagian dari D5; terkait V2-004 |
| PLAN10-6 | Limit live final: global + per operator (usul 60/20; kini 16) | MENUNGGU | — | Bagian dari D5 |
| PLAN10-7 | Master interim: tetap Contabo melewati 17 Des 2026 atau pindah provider? | MENUNGGU | — | Bagian dari D5. Contabo interim ±1 bulan (support sangat lambat). Keputusan pindah terpisah dari plan VPS 8 |

## 3. Keputusan EVALUASI §6 (evaluasi sebelum perbaikan)

| ID | Keputusan | Status | Tanggal | Bukti / dokumen |
|---|---|---|---|---|
| EV6-1 | Urutan perbaikan final (setuju / ubah matriks §5?) | MENUNGGU | — | Matriks §5 adalah MASUKAN evaluasi, bukan keputusan |
| EV6-2 | Pembagian 3 bagian (PUBLISH / ISI BAHAN / LIVE) — implementasikan? Bentuk deploy: 3 proses di 1 box dulu, atau langsung pisah box? | DIPUTUSKAN (rencana) / MENUNGGU (bentuk deploy) | Rencana diputuskan user 22 Nov 02:50 WIB | R4 dua-arah SUPERSEDED oleh rencana 3 bagian. Rencana deploy terperinci dibuat SETELAH keputusan bentuk deploy |
| EV6-3 | P4 — perbaiki janitor sekarang? | MENUNGGU | — | Tiket audit V2-004 lengkap (Cycle 5); eksekusi menunggu sesi interaktif |
| EV6-4 | P2 — siapa yang bisa cek Cloud Console proyek terdampak (pemilik proyek belum diketahui)? Atau izinkan audit jalur fallback kredensial? | MENUNGGU | — | 6 teori penyebab sudah dieliminasi; hanya langkah diagnostik sampai penyebab ditemukan |
| EV6-5 | P6 — eskalasi Dahono sudah dikirim? | MENUNGGU | — | D1 ditunda 20 Nov, dieksekusi 21 Nov, terbukti terblokir sisi Dahono (gateway 407, akun Starter aktif). Status eskalasi ke support belum tercatat jelas — OPEN QUESTION |
| EV6-6 | P1 — tambah slot render ke-3, atau jadwalkan render panjang ke jendela sepi (00:00–06:00), atau biarkan dulu? | MENUNGGU | — | Saturasi lunak masih tertangani (206/206 terbit) |
| EV6-7 | P7 — desain throttle render seperti apa (tunda akuisisi tugas baru vs throttle ffmpeg)? | MENUNGGU | — | Usulan desain: tunda akuisisi tugas baru (lembut); terkait D9 |
| EV6-8 | P15+P16 — diagnosis + perbaikan encoder live dan pemindahan job berat keluar dari web: sebelum Fase 0, bersamaan, atau sesudahnya? | DIPUTUSKAN (P16) / MENUNGGU (P15) | P16: diagnosis V2-002 + perbaikan D8 deployed 22 Nov 07:24–08:20 | P16 selesai via D8 (dengan koreksi pemetaan: 114, bukan 706). P15 menunggu PLAN10-2 (Fase 0) |

## 4. Keputusan lain yang tercatat (di luar D1–D11 / §10 / §6)

| ID | Keputusan | Status | Tanggal | Bukti / dokumen |
|---|---|---|---|---|
| V2-002-R2 | Audit ulang KEIN di host benar (re-audit V2-001) | DITOLAK | 22 Nov 06:55 + 07:03 WIB | User menolak 2× ("lewati kein", "KEIN tetap dilewati"). JANGAN tawarkan lagi kecuali user mengangkatnya |
| TR-720p | Trade-off 720p vs 1080p untuk live (VPS 8 muat 134 stream @720p vs 70 @1080p; 50 channel muat VPS 6 total $26,01/bln) | MENUNGGU | Diajukan user 21 Nov 2026 | 1080p untuk semua tetap keputusan berlaku (DOCUMENTED FACT NOT YET REVALIDATED — MEMORY.md). Angka 720p adalah opsi terbuka, bukan keputusan |
| STORAGE | Provider + retensi storage off-site (Litestream/Restic) | MENUNGGU | — | Biaya BELUM DIHITUNG (V2-009). RPO ≤5 mnt / RTO ≤30 mnt adalah TARGET yang harus dibuktikan restore drill, bukan janji |
| KONTRAK-12 | Kontrak 12 bulan VPS 8 (total $31,87/bln, bayar di muka $190,21) | MENUNGGU | — | PLAN §7: putuskan setelah model terbukti (bulanan dulu) |
| OMNI-600 | Paket Google Omni API Generate Video (Unlimited 1 bulan, maks 1 paralel, Rp 600.000 + bonus API Image Unlimited) | MENUNGGU | Dievaluasi user 21 Nov 2026 | Belum ada keputusan pembelian (DOCUMENTED FACT NOT YET REVALIDATED — MEMORY.md) |
| KEIN-IMG | Harga KEIN IMAGE per gambar + apakah boleh paralel (kini serial 78,7 dtk/gambar) | MENUNGGU | — | Tanyakan ke owner KEIN (rekomendasi BIAYA doc). Harga/gambar TIDAK DIKETAHUI |
| WEB-RESTART | Restart `masjavas-os-web.service` agar R5 berlaku untuk start via API (proses web masih kode lama) | MENUNGGU | — | Follow-up `v2-002/ENCODER114_FIX_REPORT.md`; jadwalkan di jendela maintenance berikutnya |
| TCO-1 | Cakupan cap $34,67/bln: seluruh TCO (infrastruktur + storage + backup + domain + layanan) | DIPUTUSKAN | 22 Nov 13:13 WIB | User memilih "Cap untuk seluruh TCO" (widget, sprint audit). Konsekuensi: infrastruktur saja sudah $34,67 → storage/backup harus $0 atau infrastruktur direvisi; skenario expected/worst TCO melebihi cap |

## 5. Ringkasan status

| Status | Jumlah | ID |
|---|---|---|
| DIPUTUSKAN | 5 (+2 sebagian) | D4, D7, D8, D11, TCO-1; EV6-2 (rencana saja — bentuk deploy menunggu), EV6-8 (P16 saja — P15 menunggu) |
| MENUNGGU | 28 | D2, D3, D5, D6, D9, D10, D12, D13, D14, PLAN10-1 s.d. PLAN10-7, EV6-1, EV6-3 s.d. EV6-7, TR-720p, STORAGE, KONTRAK-12, OMNI-600, KEIN-IMG, WEB-RESTART |
| DITUTUP | 1 | D1 |
| DITOLAK | 1 | V2-002-R2 |

Keputusan paling mendesak menurut program (PROPOSAL, bukan keputusan user):
D9/D10 (Guard v2 — prasyarat proteksi live), D12/D13 (nasib 698 + restart
`cd4fb7a`), PLAN10-2 (Fase 0 $0), EV6-4 (diagnosa P2).
