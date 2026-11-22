# MASJAVAS RISK REGISTER — MASJAVAS TUBE OS V2

**Versi:** 1.0 — 22 Nov 2026, 12:35 WIB (dibuat dari register awal dashboard +
EVALUASI_SEBELUM_PERBAIKAN P1–P16 + PLAN §8).
**Sumber:** `ENGINEERING_READINESS_DASHBOARD.md` (§ register risiko),
`EVALUASI_SEBELUM_PERBAIKAN_2026-11-22.md` (P1–P16),
`PLAN_PENAMBAHAN_VPS8_2026-11-22.md` (§8), temuan V2-001–V2-003.

## Legenda label

- **VERIFIED FACT** — terverifikasi langsung (DB/log/kode/tes) 21–22 Nov 2026.
- **DOCUMENTED FACT NOT YET REVALIDATED** — tercatat di dokumen/catatan lain,
  belum diverifikasi ulang di program ini.
- **PENILAIAN** — penilaian kualitatif penulis register (dampak/kemungkinan),
  BUKAN fakta terukur. Dinilai dari bukti yang dikutip di baris yang sama.
- **ASSUMPTION / OPEN QUESTION / OWNER DECISION REQUIRED** — sesuai artinya.
- Status: OPEN / DIAGNOSED / FIX DEPLOYED / RESOLVED / ACCEPTED BY OWNER.

## 1. Risiko R1–R19 (dari dashboard)

| ID | Risiko | Dampak (PENILAIAN) | Kemungkinan (PENILAIAN) | Status | Mitigasi | Keputusan terkait |
|---|---|---|---|---|---|---|
| R1 | KEIN expiry 23 Nov tanpa fallback teruji (kuota 5.550; plan berakhir 23 Nov 2026) | Tinggi — Tahap 1/2 LoopGen API-bound; tanpa KEIN, suplai gambar/video berhenti | Tinggi — tanggal kedaluwarsa pasti; yang tak pasti hanya dampaknya (status live unknown, R15) | ACCEPTED BY OWNER (asumsi "KEIN aman", 22 Nov 06:55) | Tidak ada (diputuskan user). Fallback terdokumentasi di V2-001 (DeepInfra FLUX Rp 35,44/gbr; Muse Image Rp 180/gbr) bila sewaktu-waktu dibutuhkan | D1 (DITUTUP) |
| R2 | Encoder slot 114 mati berulang (12×/17 jam 21–22 Nov; #13 ~07:17, #14 08:09:50) + orphan 706 | Sedang — stream putus berulang; pemulihan otomatis ±15 dtk–7 mnt; downtime kumulatif kecil tapi berulang | Tinggi — observasi 24 jam GAGAL 14:47 (B10: 4 stream mati serentak, 706 exit 146; watchdog fired 14:47:11 tapi encoder tetap mati 14:47:15 — deteksi tanpa penyelamatan) | FIX DEPLOYED + ITERASI 2 — TIDAK MENAHAN (56f7a6d 07:24 + 021a9f0 ~08:20; kematian massal 14:47 membuktikan mekanisme kematian + pemulihan belum terpecahkan; lihat R20) | R3–R8 + iterasi 2 sudah deployed; perbaikan berikut menunggu investigasi B10 + keputusan owner | D8 (DIPUTUSKAN), D11 |
| R3 | D1 proxy residential 407 — bot-check yt-dlp; tier residential + bulk kosong, hanya WARP hidup | Tinggi — pasokan audio = pembatas publish #2; skip audio 3 → 31/hari (20–21 Nov); NO 171/172 audio 0/30 | Tinggi — sudah terjadi; gateway Dahono menolak akun Starter aktif (sisa 0,89 GB) | OPEN (keputusan user ditunda 20 Nov) | Eskalasi Dahono (user-side); alternatif WaroengProxy Rp 10.000/GB (belum diuji); pool derivation sebagai mitigasi parsial; JANGAN isi kredensial mati ke DB | D3 (MENUNGGU — jangan tanya ulang) |
| R4 | P2 kuota Drive/sync — mekanisme belum diketahui; sync statistik gagal 300×/24 jam (403 quotaExceeded, 42 channel, 15 vault) | Sedang — sync statistik gagal (data dashboard salah); publish TIDAK terdampak (206 terbit) | Tinggi — sedang berlangsung; 36 channel gagal 2 hari kuota berturut-turut | OPEN (6 teori tereliminasi; diagnosa saja, DILARANG menebak) | Langkah diagnostik: cek Cloud Console 1 proyek terdampak ATAU audit jalur fallback kredensial | EVALUASI §6 #4 (MENUNGGU) |
| R5 | SQLite "database is locked" 31×/24 jam (live 24×); health check supervisor terdegradasi (traceback l.878, 920) | Rendah–Sedang — deteksi encoder mati bisa terlambat; belum ada outage yang diatribusikan ke lock | Tinggi — sedang berlangsung (31×/24 jam) | OPEN (V2-007 + guard audit) | Audit schema V2-007; pemisahan worker (P3) mengurangi tulis lintas lane; target <10×/24 jam | D7-bagian (V2-007 gerbang teknis) |
| R6 | Janitor salah label 203 channel CONNECTED sebagai unpaired (klausa `OR channel_id LIKE 'STAGING-SLOT-%'`); 40,92 GB/24 jam file CONNECTED terhapus berlabel unpaired | Tinggi — file channel terbit terhapus prematur (TTL 1 jam, bukan 12 jam); risiko retry/re-upload 12 jam pasca-publish + audit menyesatkan | Tinggi — pola terbukti (12/12 sampel video `21.mp4` dari channel CONNECTED); klausa masih aktif | OPEN (V2-004; tiket audit lengkap Cycle 5 10:50 WIB) | Audit V2-004 (OQ-013 dry-run, OQ-014 retensi `deletion_audit`, OQ-015 jalur hapus lain); perbaiki klausa filter + pencocokan eksak; JANGAN jalankan mode hapus sebelum perbaikan | EVALUASI §6 #3 (MENUNGGU) |
| R7 | `shed_render` tanpa eksekutor (log-only by design, kode l.1174–1183); 112×/24 jam era legacy, 29×/17 jam era V2 | Sedang — live tak terlindungi saat render memuncak; pada 2 stream dampak belum terasa (live ±6% CPU); prasyarat skala ke 16 stream | Tinggi — terjadi setiap render mendominasi CPU | DIAGNOSED (V2-003) | V2-003-R4/R6: bangun eksekutor throttle render (tunda akuisisi tugas baru, jangan bunuh render jalan); observasi osilasi | D9 (MENUNGGU) |
| R8 | Churn ingestion 66% (333 SUPERSEDED vs 123 COMPLETED/7 hari) | Sedang — 2/3 kerja ingestion terbuang; antrean + beban tulis sia-sia | Tinggi — sedang berlangsung | OPEN | Desain dedup V2-010 (menunggu V2-007); `deduplication_key` + satu job aktif per key (V2.0) | V2-010 |
| R9 | Stage 2 I/O-bound (12,8 dtk CPU vs 100,1 dtk wall/klip; maks 900,8 dtk) | Rendah — soal throughput, bukan kegagalan; 1.011 DONE/7 hari tetap jalan | Tinggi — terukur (n=1.011) | OPEN (V2-008) | Benchmark V2-008 (konkurensi + jaringan Drive, bukan CPU mentah); canary 2 slot | V2-008 |
| R10 | Harga Master/VPS 8 + storage belum terverifikasi (Master ±$14,43 pra-pajak = ESTIMASI; storage BELUM DIHITUNG) | Sedang — total riil bisa > cap mengikat $34,67/bln; pembelian tanpa verifikasi = komitmen buta | Sedang — harga listing VPS 8 terdokumentasi; harga Master hanya estimasi | OPEN (V2-009; butuh user untuk panel Contabo — Turnstile blokir browser cloud) | Verifikasi panel sebelum beli; eskalasi bila total > cap, jangan lanjut diam-diam | D4 (MENUNGGU gerbang) |
| R11 | Dua host KEIN (`beliapi.com` vs `kein.my.id`) — kuota mungkin terpisah | — (resolved) | — | RESOLVED (OQ-004: satu akun — pindah domain, modal situs 05:29) | — | D1 |
| R12 | `loopgen_agent_tasks` kosong — rekaman Opsi 3 tidak terlacak di DB ini | Rendah–Sedang — pekerjaan Opsi 3 tak terlacak di DB; audit + dedup buta terhadap lane ini | Tinggi — kondisi saat ini | OPEN (OQ-001 → V2-007) | Audit schema V2-007 (tentukan di mana state Opsi 3 tinggal) | V2-007 |
| R13 | Pemetaan stream→slot bertentangan (stream #1795226249006 dicatat milik 706 di P16 vs milik STAGING-SLOT-114 di AUDIT_TAHAP0) | — (resolved; dampak historis: diagnosis salah alamat) | — | RESOLVED (OQ-007: stream milik STAGING-SLOT-114 — DB; V2-002 Q8) | P16 dikoreksi (lihat §2); verifikasi pemetaan via DB sebelum diagnosis | D8 |
| R14 | Split-brain: box lama Tencent masih menjalankan web + menulis DB-nya sendiri (55 MB vs 118 MB live) | — (resolved; dampak historis: dua Master divergen) | — | RESOLVED (R1 dieksekusi 07:05 WIB: web lama stopped+disabled, port 8080 tutup, DB statis — diverifikasi 07:09) | — | D7 (DIPUTUSKAN) |
| R15 | Status KEIN pada sistem LIVE tidak diketahui — audit V2-001 di host salah (box lama, stale sejak 17 Nov); asumsi "KEIN aman" tanpa bukti live | Tinggi — keputusan D1 bertumpu pada asumsi; bila KEIN live bermasalah, tidak ada peringatan dini | Tidak dapat dinilai — itulah inti risikonya (OPEN QUESTION yang diputuskan untuk tidak dijawab) | ACCEPTED BY OWNER (konfirmasi 07:03 WIB: audit ulang R2 DITOLAK) | Tidak ada (diputuskan user). Dicatat agar tidak dianggap terverifikasi | D1 (DITUTUP) |
| R16 | Governor tidak pernah mengambil aksi efektif (0/1.918 tick V2; satu-satunya aksi = shed_render no-op) | Sedang — seluruh proteksi governor saat ini hanya log; live bergantung pada headroom, bukan governor | Tinggi — terukur 17 jam (98,5% tick = none) | OPEN (V2-003-R4/R6) | Bangun eksekutor shed_render; perbaiki atribusi (R18) agar shed_ingestion bisa dipilih | D9 (MENUNGGU) |
| R17 | Konfigurasi split: guard loop = V2, admission = legacy (`LIVE_GUARD_V2_ENABLED=false`, hanya dibaca di `can_start` l.444); alasan flag `false` tidak diketahui (OQ-010) | Sedang — admission pakai TX instan 5-detik (5 penolakan 06:15 yang tak perlu terjadi dengan EMA); flag menyesatkan operator | Tinggi — konfigurasi saat ini; 6 penolakan admission/24 jam, semua wording legacy | OPEN (V2-003-R1/R5) | Jawab OQ-010 dulu (prasyarat); aktifkan admission governor sebagai langkah terpisah setelah observasi R4 | D10 (MENUNGGU — prasyarat R1) |
| R18 | Atribusi lane ingestion buta (`classify_lane` hanya kenali cmdline "ingestion"; daemon otonom ±18% CPU tercatat lane `other` → `ingestion_cpu_pct` selalu 0,0%) | Sedang — governor selalu pilih shed_render (buntu); anggaran sumber daya bisa salah baca (preseden: Plan VPS 8 menganggarkan ingestion ±0 CPU — dikoreksi 12:10) | Tinggi — terukur (1.510/1.510 tick 0,0%) | OPEN (V2-003-R3) | Perbaiki klasifikasi lane (kenali daemon + anak prosesnya); V2-003-R2 | D9 (MENUNGGU) |
| R19 | Orphan 706 mengudara TANPA AUDIO ±17 jam (pipe tanpa feeder sejak restart 14:42; bitrate tetap HEALTHY sehingga tak terdeteksi) | — (resolved; dampak historis: 17 jam siaran tanpa audio, tak terdeteksi karena metrik video sehat) | — | RESOLVED (restart manual 08:10:28 — audio mengalir, feeder 19,4 MB; downtime ±48 dtk) | R8 sweep orphan (log + rekomendasi restart manual); health check broadcast (bedakan FFmpeg hidup vs broadcast live) | D11 (DIPUTUSKAN) |
| R20 | Live MATI MASSAL + pemulihan tidak andal (22 Nov 14:47: 4 stream mati/21 dtk; reconnect gagal DB-lock; status desync 114/698; 706 tanpa reconnect; encoder ganda 707; relay task mati; supervisor "hard exiting") | Tinggi — seluruh live padam serentak; dari sudut dashboard hanya 1/4 tampil STREAMING; live 24/7 tidak dapat diandalkan sebagai produk | Tinggi — terjadi 1× (14:47) ±6,5 jam pasca-iterasi 2; penyebab bersama belum terpecahkan; 5 titik kegagalan pemulihan terverifikasi | OPEN (dicatat 15:15 atas instruksi owner; investigasi + perbaikan menunggu keputusan owner) | Investigasi penyebab bersama (korelasi TX/jaringan); reconnect tahan DB-lock; status atomik; satu jalur pemulihan; teardown relay; selidiki sinyal supervisor | — (keputusan baru; belum bernomor) |

## 2. Risiko dari P1–P16 (yang belum tercakup R1–R19)

Pemetaan: P2→R4, P4→R6, P6→R3, P7→R7 (dicatat di §1, tidak diduplikasi).
P16 DIKOREKSI (lihat catatan di bawah tabel).

| ID | Risiko | Dampak (PENILAIAN) | Kemungkinan (PENILAIAN) | Status | Mitigasi | Keputusan terkait |
|---|---|---|---|---|---|---|
| P1 | Render jenuh lunak: 37,3 jam kerja render/hari, 2 slot = 18,6 jam dinding > jendela publish 17 jam; 7 tugas terlambat >2 jam, maks 566 mnt | Sedang — keterlambatan terbit; saturasi LUNAK (206/206 ber-bahan lengkap tetap terbit) | Tinggi — terukur 21 Nov (n=206; render rata-rata 651 dtk, p90 1.396 dtk) | OPEN | Opsi: slot render ke-3 (naikkan CPU puncak — hati-hati, load sudah 4–5,5), jadwalkan render panjang ke 00:00–06:00, atau biarkan (masih tertangani) | EVALUASI §6 #6 (MENUNGGU) |
| P3 | Satu proses worker untuk semua lane (publish, ingestion, LoopGen, janitor, catchup — 1 proses Python + 1 event loop, PID 778998) | Tinggi — satu crash/stall menghentikan publish DAN persiapan sekaligus; risiko struktural utama | Rendah–Sedang — belum ada crash total tercatat; ini risiko struktural, bukan insiden berjalan (PENILAIAN) | OPEN (rencana 3 bagian DIPUTUSKAN user 22 Nov 02:50; implementasi menunggu) | Pembagian 3 bagian: PUBLISH / ISI BAHAN / LIVE (V2.0 Track; §7 EVALUASI). Risiko perbaikan TERTINGGI — wajib backup, tes temp-DB, deploy bertahap, rollback, observasi 24–72 jam | EVALUASI §6 #2 (rencana DIPUTUSKAN; bentuk deploy MENUNGGU) |
| P5 | 673/707 channel masih ber-ID `STAGING-SLOT-*` (termasuk 203 CONNECTED); placeholder tak pernah diganti saat pairing | Sedang — akar P4 + ambiguitas join channel↔token↔Drive↔log di banyak tempat | Tinggi — kondisi saat ini (hitungan DB langsung) | OPEN | Migrasi foreign key hati-hati (backup penuh, migrasi di salinan DB dulu, verifikasi join, jendela maintenance). Risiko migrasi TINGGI; sesudah P4 | — (bukan prasyarat P1–P3) |
| P8 | Plafon live 16 + hanya 6 stream key (2 stream jalan; 16 stream ≈ 67 Mbps TX + ±48% CPU live) | Sedang — skala live terhambat di 16; di atas itu butuh VPS 8 | Tinggi — plafon sudah di depan mata (limit config 16 sejak 21 Nov 17:58) | OPEN | Bertahap (+2–4 stream, observasi); P7 sebagai prasyarat; setup stream key per channel (operator-side); VPS 8 untuk >16 | D4 (VPS 8), PLAN §10 #6 (limit final) |
| P9 | Anomali kuota vault 48 + 284 pasca-reset (tracker klaim 10.000/10.000 + 0 pemakaian, tapi 7 probe API gagal 15:03–17:21; kontrol 245/270/645 OK) | Rendah — sync thumbnail 2 slot tertunda (48: 12/30, 284: 14/30) | Tinggi — pola teramati 21 Nov (DOCUMENTED FACT NOT YET REVALIDATED — per catatan chat lain, belum diverifikasi ulang) | OPEN | Diagnosa P2 dulu (kemungkinan satu akar); sync bertahap (±10/akun/hari) setelah probe pulih; jangan re-render tanpa badge | EVALUASI §6 #4 |
| P10 | 4 channel TOKEN EXPIRED (NO 42, 47, 52, 58 — sisa insiden rebalance vault 12 Nov) | Rendah — 4/707 channel tak bisa terbit sampai re-pair | Tinggi — kondisi tercatat (DOCUMENTED FACT NOT YET REVALIDATED — per catatan 21 Nov) | OPEN | Re-pair operator (login ulang OAuth via dashboard) — satu-satunya jalan; re-enkripsi saja tidak cukup | Operator-side |
| P11 | Dip publish 19 Nov: hanya 88 PUBLISHED (91 tugas) vs ±180–237 hari lain; penyebab belum diketahui (tugas yang TIDAK ADA, bukan gagal — 0 FAILED) | Rendah–Sedang — penyebab tak diketahui; bisa terulang tanpa disadari | Tidak dapat dinilai — BELUM DISELIDIKI (ditemukan 22 Nov 02:00) | OPEN (diagnosa saja) | Diagnosa mandiri: cek log populasi 19 Nov + git log/deploy hari itu | — |
| P12 | Kebersihan: (a) aplikasi Node "fana" berbagi box, fungsi tak diketahui; (b) 12 traceback di log live/24 jam belum dibaca; (c) 28 RUNNING di `loopgen_jobs` belum dipastikan aktif vs klaim usang | Rendah — observasi/kebersihan, bukan kegagalan (fana ~0% CPU, ±2–3% RAM) | Terjadi — temuan sampingan 22 Nov (BELUM DISELIDIKI) | OPEN | Identifikasi fana; baca + klasifikasi 12 traceback; verifikasi 28 RUNNING (yang usang dibersihkan statusnya — hati-hati agar tidak membunuh job aktif) | — |
| P13 | Slot 266: thumbnails/set 403 (channel butuh verifikasi nomor telepon di YouTube Studio) | Rendah — 1 channel | Tinggi — 4× error 403 terkonfirmasi 21 Nov 17:50 (DOCUMENTED FACT NOT YET REVALIDATED) | OPEN | Aksi operator (verifikasi telepon) — bukan kode, bukan rate limit 429 | Operator-side |
| P14 | REGEN-48-SUBFIX1 stale RUNNING 3% (di-kill 21 Nov 15:06; pengganti SUBFIX2 sudah COMPLETED 100%) | Rendah — status menyesatkan; dapat menahan slot/lease | Sedang — BELUM DIVERIFIKASI 22 Nov (grep terputus broken pipe 01:30) | OPEN | Verifikasi proses benar-benar mati, lalu tandai FAILED bila masih stale | — |
| P15 | Web menjalankan job ffmpeg berat di dalam prosesnya (uvicorn; reassemble audio, thumb regen, fleet run, AI video pipeline) — unit web rata-rata 1,02 core; 2 anak ffmpeg @ ±98% | Sedang — web = konsumen CPU #2 setelah render; risiko responsivitas dashboard/API; job berat substansinya pekerjaan prep/publish | Tinggi — TERKONFIRMASI (cgroup + snapshot + grep kode; reassemble 64×/24 jam) | OPEN (bagian dari Fase 0 pemisahan proses) | Pindahkan eksekusi ke worker publish/prep; web kembali HTTP murni (<0,2 core); antrean/status job tetap terlacak dari dashboard. Catatan: modal reassemble (±1 core puncak) kini masuk anggaran VPS 8 (addendum 22 Nov 12:10) | EVALUASI §6 #8; PLAN §10 #2 (Fase 0) |

**KOREKSI P16 (VERIFIED FACT, V2-002):** P16 sebagaimana tertulis SALAH
ALAMAT. Stream #1795226249006 adalah milik **STAGING-SLOT-114** (bukan 706);
encoder yang mati berulang adalah encoder 114 (→ R2). Encoder orphan
(PID 1435089, parent PID 1) adalah encoder **706** (→ R19, resolved via D11).
Jangan pakai P16 sebagai sumber pemetaan stream→slot. Status: P16 sebagai
dokumen DICATAT-DIKOREKSI; risikonya hidup di R2/R13/R19.

## 3. Risiko tambahan dari PLAN §8 (belum tercakup §1–§2)

| ID | Risiko | Dampak (PENILAIAN) | Kemungkinan (PENILAIAN) | Status | Mitigasi | Keputusan terkait |
|---|---|---|---|---|---|---|
| S1 | Satu Live Box = SPOF 50 stream | Tinggi — satu box mati = semua live mati | Rendah — belum ada kegagalan box live tercatat (PENILAIAN) | OPEN | Master menampung darurat ±20 stream; box live ke-2 bila rapuh (persetujuan anggaran terpisah) | D4 (Fase 2) |
| S2 | Sharding salah → 2 supervisor berebut 1 channel | Tinggi — duplikasi siaran / perebutan state | Rendah — belum ada sharding; risiko muncul saat implementasi (PENILAIAN) | OPEN | Single-owner + lease + fencing token; canary dulu | V2.0 Track C |
| S3 | Tailscale putus → VPS 8 kehilangan kendali | Sedang — stream jalan tetap jalan; penugasan baru tertunda | Rendah — Tailscale sudah jalan stabil (PENILAIAN) | OPEN | Alarm heartbeat >90 dtk; stream tidak bergantung pada control plane untuk tetap mengudara | V2.0 Track E |
| S4 | Broadcast auto-complete tak terdeteksi (FFmpeg hidup, broadcast mati) | Sedang — downtime tak terlihat; preseden R19 (17 jam tanpa audio, metrik video HEALTHY) | Sedang — preseden nyata R19 (PENILAIAN) | OPEN | Health check broadcast (bedakan FFmpeg hidup vs broadcast benar-benar live) | V2.0 Track E |

## 4. Pertanyaan terbuka (OPEN QUESTION)

- OQ-005/OQ-006 (V2-001): perilaku fallback foto saat AUTH error; perilaku
  ingestion 8K-bg saat KEIN gagal — QUEUED-FOR-INTERACTIVE, re-base di host
  benar bila dikejar.
- OQ-010: siapa men-set `LIVE_GUARD_V2_ENABLED=false` + CPU 90 di `.env`
  dan mengapa (prasyarat V2-003-R1) — D10.
- OQ-011: apakah `LIVE_APP_RESERVED_*` / `LIVE_STREAM_RESERVED_CORES`
  masih dipakai? OQ-012: apakah jendela broadcast Turbo Mode aktif?
- OQ-013/014/015 (V2-004): dry-run janitor; retensi `deletion_audit`
  (tangkap SEGERA — belum diketahui); jalur hapus lain.
- P11: penyebab dip publish 19 Nov belum diselidiki.
- P9/P10/P13/P14: status per catatan 21 Nov, belum diverifikasi ulang.
