# KONTEKS, PERMASALAHAN, SOLUSI & INSTRUKSI UNTUK AGEN ANTIGRAVITY

**Tanggal:** Selasa, 24 November 2026, 06:20 WIB
**Penulis:** Muse (agen monitoring & verifikasi)
**Status dokumen:** Siap disampaikan ke lane Antigravity
**Cakupan:** Seluruh kinerja MASJAVAS TUBE OS — Master (VPS 4 Plus) + VPS 8 (live-01)

> Dokumen ini adalah satu paket: konteks terverifikasi (§1–§2), daftar masalah
> dengan bukti (§3), solusi yang diharapkan beserta kriteria penerimaan (§4),
> saran & ide (§5), dan instruksi kerja untuk agen Antigravity (§6–§7).
> Semua angka di sini berasal dari pemeriksaan langsung ke kedua node
> (terakhir 06:18 WIB, 24 Nov 2026), bukan dari klaim commit atau log sepihak.

---

## §1 — KONTEKS: KEADAAN APLIKASI SAAT INI

### 1.1 Arsitektur yang berjalan

| Komponen | Lokasi | Status 06:18 WIB |
|---|---|---|
| Web/API + DB (control plane) | Master (100.65.206.22) | PID 866142 (restart 05:51:30), 200 / 0,009 dtk |
| Publish worker (`--lane publish`) | Master | PID 306566 (sejak 20:39:14, 23 Nov) |
| Live supervisor + 3 encoder | VPS 8 (100.108.161.98) | Supervisor PID 56297 (sejak 18:27:55, 23 Nov) |
| Prep worker (Stage 2 + ingestion) | VPS 8 | PID 408756 (restart 05:54:20), kode terbaru aktif |
| Disk Janitor (timer 30 mnt) | Master | Aktif sejak 04:04, 24 Nov; run terakhir 06:04 normal |
| Fleet sync (timer 90 dtk) | Master | Aktif; 19/20 run dalam 30 menit terakhir |
| Node telemetry (pusher 30 dtk) | VPS 8 → Master | Sampel segar (terakhir 06:15:09) |
| Web Guardian (timer 5 mnt) | Master | Aktif; semua cek terakhir OK |
| LoopGen asset cleanup (timer harian) | Master | Terpasang; run pertama 25 Nov 03:30 WIB |

**HEAD git kedua node:** `52e74db` (05:53:56). Commit pagi ini:
`254fa2a` + `0aee7ad` (Muse: paralelisasi ingestion langkah 1–4),
`0123550` + `930e764` (Antigravity: redesign dashboard telemetry),
`b41beaf` (Antigravity: orkestrasi QoS multi-node),
`b917ad1` (Antigravity: 4x concurrency + self-healing sentinel),
`52e74db` (Antigravity: auto-heal channel staging).

**Catatan penting:** web yang berjalan (restart 05:51:30) **mendahului** commit
`b917ad1` (05:51:40) dan `52e74db` (05:53:56) — perubahan API/template ingestion
dari kedua commit itu **belum aktif di web**. Prep worker (restart 05:54:20)
**sudah** menjalankan kode terbaru.

### 1.2 Angka kinerja per 06:18 WIB

| Lane | Angka kunci |
|---|---|
| **Publish** | Hari ini PUBLISHED 6, QUEUED 204, FAILED 0; 23 Nov FINAL 207 (baseline 206 terpenuhi); semaphore 0 + DB lock 0 dalam 10 jam |
| **Ingestion** | Master: COMPLETED 157, PENDING 43; VPS 8: PENDING 39, RUNNING 4, BLOCKED 2; ~12 langkah/menit; COMPLETED +5 dalam 30 menit |
| **Stage 2** | DONE 1297, FAILED 76, PARKED 216, QUEUED 7 (Master) / 0 (VPS 8), RUNNING 9; DONE hanya +1 dalam 30 menit terakhir |
| **Live** | 3/3 STREAMING (706, 114, 707); ghost-kill 0 sejak 17:39, 23 Nov; kematian encoder 16 dalam 1 jam terakhir |
| **Disk** | Master 47% (78 GB bebas, stabil); VPS 8 17% (244 GB bebas) |
| **Load** | VPS 8: 3,54/3,76/4,40 (terendah sejak kemarin); Master: 0,65/1,04/1,05 |
| **Kuota** | ~355 ribu unit dalam ~23 jam terakhir — konsisten dengan 207 publish (341,5 ribu) + operasi live; bukan anomali |
| **QA Stage 2** | 0 konkuren (cap 1 bertahan); OAuth decrypt 0 dalam 1 jam terakhir |

### 1.3 Apa yang sudah terjadi pagi ini (ringkas)

1. **03:30–03:55** — Telemetry Worker + monitoring VPS 8 dibangun dan
   diverifikasi browser; cockpit Sentinel 6-Lane dibuat sadar-node (L3 yang
   tadinya salah baca 0 kini benar membaca 3 stream).
2. **03:58–04:11** — Akar masalah disk ditemukan: Disk Janitor mati sejak
   restart worker `--lane publish` (gate lane). Janitor dipasang sebagai
   service+timer mandiri; **~50,6 GB dibersihkan** (disk 88% → 47%).
3. **04:07–04:37** — Analisa workflow multi-node menemukan split-brain DB
   (B4) + 4 masalah lain. Antigravity merespons dengan `3b06fba` (prep_worker
   masuk git, T6) dan `12f84ec` (fleet sync 90 detik, proteksi DR, janitor
   aset) — DB konvergen, ingestion bangkit dari 0 langkah.
4. **04:44–05:22** — Langkah ingestion 1–4 diimplementasikan (hapus throttle
   autopilot di node WORKER, cap QA ke 1, prioritas live nice −10,
   `--ingestion-jobs 4`). Efektivitas terbukti: langkah 0 → 2,8/menit.
5. **05:37–05:54** — Antigravity men-deploy `b41beaf` (orkestrasi QoS),
   `b917ad1` (4x concurrency + self-healing), `52e74db` (auto-heal staging).
   Ingestion melesat ke ~12 langkah/menit, PENDING akhirnya bergerak 43 → 39,
   COMPLETED +5.
6. **05:52–05:54** — 69 run Stage 2 di-FAILED massal dengan alasan
   "item hilang / tanpa drive_file_id". Antrean Stage 2 hampir kosong sejak itu.

---

## §2 — SKOR SOLIDITAS: 75 / 100

Dihitung per lane dengan bobot: Publish 25%, Live 20%, Ingestion 15%,
Stage 2 15%, Vault 10%, Janitor/Web/Multi-node 5% masing-masing.
Skor 05:49 dan 06:18 sama-sama 75 — tapi komposisinya bergeser.

| Lane | 05:49 | 06:18 | Penyebab perubahan |
|---|---|---|---|
| L1 Publish | 90 | **90** | — (bergerak baik, FAILED 0) |
| L2 Ingestion | 55 | **70** | +15 — antrean akhirnya bergerak, COMPLETED naik |
| L3 Live | 70 | **68** | −2 — kematian encoder 16/jam tak turun |
| L4 Stage 2 | 65 | **55** | −10 — output runtuh, antrean kosong |
| L5 Janitor | 90 | **90** | — |
| L6 Vault | 75 | **75** | — |
| Web/API | 85 | **83** | −2 — restart 3× hari ini |
| Multi-node | 80 | **80** | — |

**Pembacaan:** aplikasi tidak lebih solid dari sejam lalu — masalahnya
**berpindah**. Kemarin bottleneck-nya ingestion yang macet; sekarang
bottleneck-nya Stage 2 yang kehabisan bahan. Kenaikan ingestion (+15)
dimakan habis oleh kejatuhan Stage 2 (−10) dan live/web (−2 masing-masing).

**Jalur ke 83+:** (1) pasokan run LoopGen mengisi ulang antrean Stage 2
dan output kembali ≥10 DONE/jam; (2) penyebab kematian encoder ditemukan
dan laju turun di bawah 5/jam; (3) PENDING ingestion habis (39 → 0).

---

## §3 — PERMASALAHAN

### P0 — Kritis (menghentikan atau mengancam output)

#### P0-1. Stage 2 kelaparan input — antrean hampir kosong
- **Bukti:** QUEUED 79 → 7 (Master) / 0 (VPS 8) setelah pembersihan 05:52–05:54.
  RUNNING 9 tersisa. DONE hanya +1 dalam 30 menit terakhir (sebelumnya +15/jam).
  Stage 2 Active 0/3 selama ~25 menit.
- **Akar yang sudah diketahui:** 69 run di-FAILED massal karena tidak punya
  `drive_file_id` (file sumber tidak ada di Drive). Pembersihannya benar —
  run tanpa sumber tidak akan pernah sukses — tapi itu menyingkap masalah hulu:
  **kenapa 69 run bisa dibuat tanpa file sumber?**
- **Dampak bila dibiarkan:** Stage 2 menganggur total begitu 9 RUNNING selesai.
  Pipa bahan (video loop → assembly → publish) kering di hulu.
- **Yang belum diketahui:** aktor pasti pembersihan massal (kemungkinan sweep
  Antigravity saat deploy — belum terbukti); apakah ke-69 run itu bisa
  diperbaiki (sumbernya masih ada di tempat lain?) atau memang harus dibuang.

#### P0-2. Kematian encoder ~16/jam — penyebab exit 224/152 belum ketemu
- **Bukti:** 16 kematian dalam 1 jam terakhir (06:18). Total sejak 20:00,
  23 Nov: 119+ kematian. Distribusi: exit 224 = 37, 152 = 36, 146 (RTMP
  timeout) = 16, 0 = 14, 187 = 10. Semua 3 encoder berganti PID dalam sejam
  terakhir — semuanya mati dan auto-reconnect. Pola mati-bersamaan ketiga
  encoder berulang (terakhir 05:10–05:11 WIB).
- **Yang sudah diketahui:** BUKAN RTMP timeout (146 hanya 16 dari 119).
  Auto-reconnect bekerja (~30 detik), sehingga penonton tidak melihat
  gangguan berkepanjangan — tapi ketidakstabilan ini adalah masalah
  reliabilitas live nomor satu.
- **Yang belum diketahui:** mekanisme kematian 224/152. Belum ada korelasi
  yang terbukti dengan load, jaringan, atau ffmpeg. (Ini T7 dalam prompt
  19:43, 23 Nov — masih terbuka.)
- **Dampak bila dibiarkan:** setiap kematian adalah jendela stream terputus;
  pada skala 50–100 channel, laju ini berarti ratusan gangguan per hari.

#### P0-3. Ingestion belum tuntas — 39 PENDING + 2 BLOCKED + divergensi DB
- **Bukti:** PENDING turun 43 → 39 (VPS 8) tapi Master masih menunjukkan 43
  (lag sync). COMPLETED naik +5 dalam 30 menit — kemajuan nyata, tapi 39 job
  tersisa dengan durasi median ~4 jam per job berarti antrean baru habis
  dalam belasan jam. 2 job BLOCKED tidak tersentuh auto-heal (tidak memenuhi
  kriteria resume). Total job ingestion: Master 636 vs VPS 8 626 (selisih 10);
  COMPLETED 157 vs 149.
- **Dampak bila dibiarkan:** 39 channel menunggu bahan audio/thumbnail;
  divergensi DB membuat angka di dashboard tidak bisa dipercaya penuh.

### P1 — Serius (menurunkan kualitas / efisiensi)

#### P1-1. PARKED 216 — tumpukan Stage 2 yang diparkir tidak bergerak
- **Bukti:** PARKED 215 → 216 dalam 30 menit terakhir. Tidak ada mekanisme
  yang terlihat mengurai tumpukan ini (retry otomatis? triase manual?).
- **Dampak:** 216 run (potensi ~216 video) tertahan tanpa kejelasan nasib.

#### P1-2. Restart churn — web 3×, prep 3× dalam sehari
- **Bukti:** Web restart 04:47:29, 05:37:26, 05:51:30 (ketiganya bersih,
  SIGTERM). Prep restart 04:49:52, 05:38:17, 05:54:20. Pola: restart terjadi
  beberapa detik/menit SEBELUM commit mendarat (deploy-dengan-restart).
  Setiap restart web meninggalkan 1 proses worker sisa di control group
  (pola berulang, juga terlihat 03:03).
- **Dampak:** tiap restart adalah jendela risiko (publish tertunda, stream
  terganggu bila salah layanan); proses sisa menumpuk dan membingungkan
  diagnosis PID.

#### P1-3. Stream 114 paling tidak stabil; 707 belum terkonfirmasi ditonton
- **Bukti:** 114 (Ruang Galau) sempat STOPPED ~10 menit (05:10–05:20 WIB)
  lalu pulih sendiri; ini stream yang paling sering bermasalah sepanjang
  23–24 Nov. 707 (RIFFORA) berstatus STREAMING di DB, tapi status
  "sedang live" di halaman YouTube-nya belum pernah terkonfirmasi
  (bot-check menghalangi probe).
- **Dampak:** ketidakpastian apakah penonton benar-benar bisa menonton 707.

#### P1-4. Kegagalan dekripsi token OAuth (69×) — penyebab tidak pernah didiagnosis
- **Bukti:** 69 kegagalan dekripsi token untuk ketiga vault live di VPS 8,
  pertama 20:01 CET, 23 Nov. 0 dalam 1 jam terakhir — mungkin berhenti,
  mungkin dorman. Penyebab tidak pernah diselidiki.
- **Dampak bila kembali:** supervisor live tidak bisa me-refresh kredensial
  → stream tidak bisa di-relink/dirawat → siaran mati tanpa pemulihan otomatis.

### P2 — Perlu ditangani (risiko menengah / hutang teknis)

| # | Masalah | Bukti / catatan |
|---|---|---|
| P2-1 | DR restore drill belum pernah dilakukan | Backup DR terbukti valid (symlink + snapshot), tapi restore belum pernah diuji. Backup yang belum diuji restore-nya bukan backup. |
| P2-2 | Breakdown kuota per-vault belum pernah dicek | Total ~355 ribu unit/23 jam konsisten dengan operasi normal, tapi distribusi per-vault tidak diketahui — vault yang kelaparan tidak akan terlihat sampai terlambat. |
| P2-3 | ffmpeg 6.1.1 (VPS 8) vs 8.x (Master) | Ketidakcocokan versi; perilaku filter/flag bisa berbeda antar node. |
| P2-4 | Mekanisme sharding tidak terdokumentasi | `MASJAVAS_ASSIGNED_SHARDS` kosong; penugasan sebenarnya lewat flag migrasi DB — tidak ada dokumen yang menjelaskan. |
| P2-5 | `file_leases` kosong (0 baris) | Lapisan proteksi lease yang dilaporkan cockpit bersifat dormant. |
| P2-6 | PENDING task 4551 (STAGING-SLOT-028) | Semantik status PENDING belum dijelaskan. |
| P2-7 | Web menjalankan kode pra-b917ad1 | Redesign dashboard telemetry (0123550) + API/template ingestion baru (b917ad1/52e74db) belum aktif di web. |
| P2-8 | Retensi photo-assets 14 GB belum diputuskan | 215 ribu file bahan sumber; tanpa kebijakan retensi/arsip akan tumbuh tanpa batas. |
| P2-9 | Cap audio pool 5 GB vs 15 GB belum diputuskan | Kode memakai 15 GB; docstring janitor menyebut 5 GB. |

---

## §4 — SOLUSI YANG DIHARAPKAN

Setiap solusi di bawah ini disertai **kriteria penerimaan terukur**.
Sebuah perbaikan dinyatakan selesai hanya bila kriterianya terpenuhi
dengan bukti independen (angka DB + perilaku teramati), bukan hanya
"kode sudah di-deploy".

### S0-1. Pasok ulang antrean Stage 2 + cegah run yatim (untuk P0-1)
1. **Selidiki hulu:** lacak ke-69 run yang di-FAILED — dari job LoopGen mana
   mereka berasal? Apakah agen penghasil video gagal mengunggah ke Drive,
   atau tautan Drive-nya hilang di tengah jalan? Laporkan akar penyebabnya
   (bukan dugaan — dengan jejak ID job/run/file).
2. **Validasi di titik pembuatan run:** Stage 2 run TIDAK BOLEH dibuat tanpa
   `drive_file_id` yang terverifikasi ada. Bila sumber tidak ada, run jangan
   dibuat — catat sebagai kegagalan hulu di job LoopGen-nya, bukan sebagai
   run Stage 2 yang akan gagal kemudian.
3. **Perbaiki yang bisa diperbaiki:** untuk run yang sumbernya ternyata masih
   ada (mis. di `agent-results` lokal atau di folder Drive lain), tautkan
   ulang `drive_file_id`-nya dan kembalikan ke QUEUED. Untuk yang sumbernya
   benar-benar hilang, biarkan FAILED dan laporkan daftarnya.
4. **Isi ulang antrean:** pastikan LoopGen memproduksi run baru sehingga
   QUEUED kembali ke level sehat (target: ≥50 QUEUED dalam 24 jam).
- **Kriteria penerimaan:** (a) akar penyebab 69 run yatim dilaporkan dengan
  bukti; (b) validasi pembuatan run aktif (buktikan dengan 1 run baru yang
  dibuat + 1 kasus sumber-hilang yang ditolak di titik pembuatan);
  (c) QUEUED ≥ 50 dalam 24 jam; (d) DONE kembali ≥ 10/jam.

### S0-2. Temukan penyebab kematian encoder 224/152 (untuk P0-2)
1. **Instrumentasi dulu, tebakan belakangan:** setiap kematian encoder harus
   mencatat: exit code, 20 baris terakhir stderr ffmpeg, load/CPU/RAM saat
   mati, status RTMP (timeout? putus?), dan apakah kematian itu tunggal atau
   simultan dengan encoder lain. Simpan dalam tabel/log terstruktur (bukan
   hanya journal mentah).
2. **Korelasikan:** setelah 24 jam data instrumentasi, uji hipotesis secara
   berurutan — (a) tekanan memori/CPU, (b) gangguan jaringan/RTMP,
   (c) bug ffmpeg 6.1.1, (d) intervensi supervisor/sentinel, (e) OOM-killer /
   cgroup. Laporkan hipotesis yang TERBUKTI dan yang TERELIMINASI dengan data.
3. **Perbaiki berdasarkan bukti:** implementasikan perbaikan untuk mekanisme
   yang terbukti. Jangan "memperbaiki" mekanisme yang belum terbukti.
- **Kriteria penerimaan:** (a) instrumentasi aktif + 24 jam data kematian
  tercatat; (b) laporan korelasi dengan minimal 1 mekanisme terbukti atau
  3 hipotesis tereliminasi; (c) laju kematian turun di bawah 5/jam selama
  6 jam berturut-turut setelah perbaikan.

### S0-3. Tuntaskan antrean ingestion + konvergensi DB (untuk P0-3)
1. **Biarkan orkestrasi bekerja** — jangan ubah governor dinamis
   (ECO/TURBO/BALANCED) sebelum antrean habis; yang dibutuhkan sekarang
   adalah observasi, bukan tuning baru.
2. **Selidiki 2 BLOCKED:** kenapa kedua job ini tidak memenuhi kriteria
   resume auto-heal? Bila kriterianya terlalu sempit, perluas dengan hati-hati.
   Bila job-nya memang rusak, tandai dan laporkan.
3. **Perbaiki sync ingestion:** fleet sync harus menyalin transisi
   PENDING → RUNNING → COMPLETED secara penuh (saat ini Master tertinggal:
   PENDING 43 vs 39, COMPLETED 157 vs 149, total 636 vs 626). Target:
   selisih total job antar-DB = 0 dalam 1 jam setelah sinkronisasi.
- **Kriteria penerimaan:** (a) PENDING VPS 8 = 0 (atau <5) dalam 24 jam;
  (b) 2 BLOCKED terselesaikan atau dilaporkan rusak dengan bukti;
  (c) selisih total ingestion_jobs antar-DB = 0.

### S1-1. Urai PARKED 216 (untuk P1-1)
- Triase 216 run PARKED: kelompokkan berdasarkan alasan parkir; buat kebijakan
  retry otomatis untuk alasan transient (mis. IncompleteRead) dan daftar
  manual untuk alasan struktural. Laporkan pembagian kelompoknya.
- **Kriteria penerimaan:** PARKED < 100 dalam 48 jam, dengan kebijakan retry
  yang terdokumentasi.

### S1-2. Disiplin deploy — hentikan restart churn (untuk P1-2)
- Satu restart per layanan per jendela 15 menit (aturan lintas-agen yang
  sudah berlaku). Gabungkan beberapa commit menjadi SATU restart terjadwal,
  bukan restart per commit. Setelah restart web, verifikasi tidak ada proses
  worker sisa; bila ada, bersihkan dan cari penyebabnya (kemungkinan
  `KillMode`/`TimeoutStopSec` unit).
- **Kriteria penerimaan:** maksimal 2 restart web per hari ke depan;
  0 proses sisa setelah restart (verifikasi 3 restart berturut-turut).

### S1-3. Stabilkan 114 + konfirmasi 707 (untuk P1-3)
- Bandingkan konfigurasi 114 dengan 706/707 (vault, kunci stream, slot
  encoder, riwayat reconnect). Bila ada perbedaan, selaraskan. Untuk 707:
  konfirmasi status live melalui API YouTube (bukan scraping halaman —
  bot-check membuat scraping tidak bisa dipercaya).
- **Kriteria penerimaan:** 114 tanpa STOPPED selama 12 jam; 707 terkonfirmasi
  live via API (dengan broadcast ID + timestamp).

### S1-4. Diagnosis kegagalan dekripsi OAuth (untuk P1-4)
- Audit vault live di VPS 8: apakah token terdekripsi dengan benar sekarang?
  Apakah ada vault yang `master_vault_id`-nya berubah tanpa re-pair?
  (Pelajaran insiden 12 Nov: pindah vault tanpa re-pair mematahkan refresh.)
  Bila ada token yang perlu re-pair, laporkan daftarnya ke owner — re-pair
  adalah tindakan owner, bukan agen.
- **Kriteria penerimaan:** laporan audit 3 vault live (status token, hasil
  dekripsi uji, hasil refresh uji) + daftar re-pair bila ada.

### S2 — Hutang teknis (P2-1 s.d. P2-9)
- **P2-1:** jalankan DR restore drill ke database sementara (JANGAN ke DB live):
  restore snapshot DR, verifikasi integritas + jumlah baris kunci.
- **P2-2:** tambahkan breakdown per-vault ke halaman telemetry (unit per vault
  per hari + sisa kuota).
- **P2-3:** selaraskan ffmpeg VPS 8 ke 8.x (atau dokumentasikan kenapa 6.1.1
  dipertahankan + perbedaan perilaku yang relevan).
- **P2-4:** dokumentasikan mekanisme penugasan node (flag migrasi DB) di
  `ARSITEKTUR_MULTI_NODE_MASJAVAS_TUBE_OS_V2.md` atau dokumen arsitektur.
- **P2-5:** putuskan: isi dan tegakkan `file_leases`, atau hapus lapisan itu
  dari cockpit secara jujur (jangan tampilkan angka yang tidak berarti).
- **P2-6:** dokumentasikan semantik status PENDING (task 4551) di kode.
- **P2-7:** restart web TERJADWAL (satu kali, gabungkan dengan deploy lain)
  untuk mengaktifkan 0123550 + b917ad1/52e74db — dengan pemberitahuan dulu.
- **P2-8 / P2-9:** keputusan owner (retensi photo-assets, cap pool) — siapkan
  opsi + angka, jangan putuskan sendiri.

---

## §5 — SARAN & IDE (di luar perbaikan wajib)

### I-1. Peringatan dini (alerting) — celah terbesar saat ini
Tiga insiden pagi ini (disk 88%, janitor mati 8 jam, antrean Stage 2 kosong)
semuanya **tidak memicu alarm apa pun** — semuanya ditemukan manusia secara
manual. Usulan ambang peringatan (via sentinel/telegram yang sudah ada):
- Disk Master > 80% (peringatan) / > 85% (kritis).
- Janitor tidak berjalan > 45 menit.
- Stage 2 QUEUED = 0 selama > 30 menit (pipa kering).
- Ingestion PENDING tidak berubah selama > 2 jam (macet).
- Kematian encoder > 10/jam selama 2 jam berturut-turut.
- Fleet sync gagal 3× berturut-turut.
- DB Master vs VPS 8 selisih total job > 5.

### I-2. Dashboard "kesehatan pipa" satu layar
Satu panel yang menunjukkan aliran end-to-end: LoopGen memproduksi →
Stage 2 merender → Drive menyimpan → ingestion melengkapi → publish
menerbitkan — dengan angka antrean di tiap sambungan. Saat ini tiap angka
berada di halaman berbeda; kebocoran antar-tahap (seperti 69 run yatim)
tidak terlihat sampai terlambat.

### I-3. Kontrak data antar-tahap (data contract)
Tetapkan dan tegakkan di kode: setiap Stage 2 run WAJIB punya
`drive_file_id` yang terverifikasi; setiap ingestion job WAJIB punya
`channel_id` yang terpasang; setiap live task WAJIB punya stream key yang
cocok dengan broadcast-nya. Pelanggaran kontrak ditolak di titik masuk
(validate-at-entry), bukan ditemukan belakangan sebagai FAILED misterius.

### I-4. Uji beban ingestion terkendali
Dengan load VPS 8 kini rendah (3,5/4,4) dan governor dinamis aktif,
ini jendela yang tepat untuk menguji batas: naikkan konkurensi ingestion
secara bertahap (4 → 6) selama 1 jam sambil memantau load + live.
Bila load15 bertahan < 6,5 dan live 3/3, pertahankan. Bila tidak,
kembalikan — dengan data, bukan dugaan.

### I-5. Pemulihan otomatis untuk PARKED transient
Run PARKED karena IncompleteRead (jaringan) seharusnya di-retry otomatis
dengan backoff — bukan menunggu triase manual. Ini akan mengurai sebagian
besar PARKED 216 tanpa campur tangan manusia.

### I-6. Jejak audit untuk aksi massal
Aksi yang mengubah >10 baris sekaligus (seperti FAILED massal 69 run)
harus mencatat: siapa/apa pelakunya, kapan, kenapa, dan daftar ID yang
terkena — dalam satu baris audit. Pembersihan 05:52–05:54 tidak meninggalkan
jejak seperti ini, sehingga kami harus merekonstruksi dari timestamp.

### I-7. Satu restart, banyak commit (deploy batching)
Gabungkan deploy menjadi jendela terjadwal (mis. 2× sehari) dengan
satu restart per layanan. Ini menghentikan pola restart-per-commit
yang terlihat pagi ini (5 restart dalam ~1 jam).

---

## §6 — INSTRUKSI KEPADA AGEN ANTIGRAVITY

### 6.1 Urutan kerja (kerjakan berurutan, laporkan per fase)

| Fase | Pekerjaan | Target waktu | Keluaran |
|---|---|---|---|
| **Fase 0 — Diagnosis** | S0-1 langkah 1 (lacak 69 run yatim) + S0-2 langkah 1 (instrumentasi kematian encoder) + S0-3 langkah 2 (selidiki 2 BLOCKED) + S1-4 (audit vault) | 6 jam | Laporan diagnosis dengan bukti (ID, angka, log) — TANPA perubahan kode produksi |
| **Fase 1 — Perbaiki hulu** | S0-1 langkah 2–4 (validasi run + perbaikan + isi ulang antrean) + S1-1 (triase PARKED) + S0-3 langkah 3 (sync ingestion) | 24 jam | Antrean Stage 2 terisi; PARKED < 100; DB konvergen |
| **Fase 2 — Reliabilitas live** | S0-2 langkah 2–3 (korelasi + perbaikan kematian encoder) + S1-3 (stabilkan 114, konfirmasi 707) | 48 jam | Laju kematian < 5/jam selama 6 jam |
| **Fase 3 — Pengerasan** | S1-2 (disiplin deploy) + I-1 (alerting) + I-6 (audit aksi massal) + P2-1 (DR drill) + P2-2 (breakdown kuota) | 72 jam | Alert aktif; DR drill lolos; disiplin deploy terbukti |
| **Fase 4 — Hutang teknis** | P2-3 s.d. P2-9 (sisanya) | Sesuai persetujuan owner | Per item, dengan verifikasi |

### 6.2 Aturan verifikasi (mengikat — pelanggaran berarti pekerjaan belum selesai)
1. Setiap perbaikan diverifikasi dengan **bukti independen**: angka DB +
   perilaku teramati (bukan hanya "log menunjukkan sukses").
2. Untuk perbaikan live: buktikan dengan status broadcast di YouTube
   (via API), bukan hanya PID encoder atau status DB.
3. Untuk perbaikan antrean: buktikan dengan angka QUEUED/RUNNING/DONE
   sebelum vs sesudah, di KEDUA DB.
4. Laporkan juga yang GAGAL diverifikasi — jangan sembunyikan.
5. Jangan pernah mengklaim "100% RESOLVED" tanpa angka yang mendukung.

### 6.3 Aturan perubahan (mengikat)
1. **Backup dulu:** snapshot DB + backup file sebelum perubahan data/skema.
2. **Tes dulu:** tes unit/integrasi yang relevan harus hijau sebelum deploy.
3. **Satu restart per 15 menit** lintas semua agen — koordinasikan dengan
   lane lain sebelum me-restart layanan apa pun.
4. **Jangan restart live supervisor** tanpa persetujuan owner (stream sedang
   mengudara).
5. **Jangan restart worker Master** pada jam publish sibuk tanpa koordinasi.
6. Commit dengan pesan yang akurat (jangan menyapu file yang tidak terkait
   ke dalam satu commit; jangan memberi label yang salah pada perubahan).
7. Setelah deploy, amati 30 menit sebelum menyatakan berhasil.

### 6.4 Format laporan per fase
Untuk setiap fase, laporkan dalam format ini:
- **Yang dikerjakan:** daftar perubahan (file + commit hash).
- **Bukti:** angka sebelum vs sesudah + perintah yang dipakai untuk memverifikasi.
- **Yang belum selesai:** daftar eksplisit + alasan.
- **Risiko baru:** bila ada efek samping yang ditemukan.
- **Butuh owner:** keputusan/tindakan yang hanya bisa dilakukan owner.

---

## §7 — LARANGAN (JANGAN DILAKUKAN TANPA PERSETUJUAN OWNER)

1. **Jangan sentuh logika Guard 1 bypass** di `worker/ingestion.py`
   (baris ~6171–6174, "diabaikan di node WORKER") tanpa koordinasi —
   itu perbaikan yang sudah terbukti bekerja; bila governor dinamis
   membuatnya redundan, diskusikan dulu — jangan hapus diam-diam.
2. **Jangan kembalikan throttle autopilot** (cap ingestion ke 1 saat publish
   aktif) di node WORKER — throttle itu sudah terbukti usang di arsitektur
   multi-node (publish dan ingestion tidak lagi berbagi CPU).
3. **Jangan hapus file apa pun** (termasuk `photo-assets`, backup, atau
   file job) tanpa persetujuan owner + verifikasi Drive.
4. **Jangan ubah timer janitor / fleet sync / guardian** tanpa persetujuan.
5. **Jangan ubah skema DB** (tabel/kolom baru) tanpa persetujuan owner
   + rencana migrasi + rollback.
6. **Jangan sentuh konfigurasi vault/OAuth** (re-pair, pindah vault,
   ubah `master_vault_id`) — itu ranah owner.
7. **Jangan deploy ke production pada 17:00–22:30 WIB** (jendela siaran
   prime) tanpa persetujuan owner.

---

## §8 — LAMPIRAN: FAKTA KUNCI (per 06:18 WIB, 24 Nov 2026)

### A. Service & PID
| Layanan | Node | PID | Sejak |
|---|---|---|---|
| Web (`masjavas-os-web`) | Master | 866142 | 05:51:30, 24 Nov |
| Live supervisor (`masjavas-live`) | VPS 8 | 56297 | 18:27:55, 23 Nov |
| Worker publish (`masjavas-os-worker`) | Master | 306566 | 20:39:14, 23 Nov |
| Prep (`masjavas-prep`) | VPS 8 | 408756 | 05:54:20, 24 Nov |
| Janitor timer | Master | — | Aktif sejak 04:04, 24 Nov |
| Fleet sync timer | Master | — | Aktif, 90 detik |
| Node telemetry timer | VPS 8 | — | Aktif sejak 03:32:35, 24 Nov |

### B. Live stream
| Slot | Channel | Task ID | Status | Broadcast ID |
|---|---|---|---|---|
| 706 | melodi cover | 1795191337447 | STREAMING | `rlLDyJSI6ok` (live + publik terkonfirmasi 23 Nov) |
| 114 | Ruang Galau | 1795226249006 | STREAMING | `yf5NOyy1SXU` (ID ke-4) |
| 707 | RIFFORA | 1795321099635 | STREAMING | `6Yy95afrjtQ` (watchability belum terkonfirmasi) |
| 698 | RuangIrama | 1795332171262 | TASK DIHAPUS | — (dihapus MONA, 23 Nov 06:06) |

### C. Angka antrean (06:18)
- **Publish (hari ini):** PUBLISHED 6, QUEUED 204, AWAITING_MONETIZATION 1,
  SKIPPED_AUDIO_MISSING 30, FAILED 0.
- **Ingestion (Master):** BLOCKED 2, COMPLETED 157, COMPLETED_PARTIAL 35,
  PENDING 43, SUPERSEDED 399. **(VPS 8):** BLOCKED 2, COMPLETED 149,
  COMPLETED_PARTIAL 35, PENDING 39, RUNNING 4, SUPERSEDED 397.
- **Stage 2 (Master):** DONE 1297, FAILED 76, PARKED 216, QUEUED 7.
  **(VPS 8):** DONE 1296, FAILED 76, PARKED 215, RUNNING 9, QUEUED 0.

### D. File kode kunci
- `worker/ingestion.py` — Guard 1 bypass l.6171–6174; `heal_failed_ingestion_jobs`
  l.~495; `compute_adaptive_ingestion_concurrency` l.6115;
  `run_ingestion_worker_cycle` l.6237 (auto-heal tiap siklus l.6270).
- `worker/prep_worker.py` — governor dinamis (ECO/TURBO/BALANCED);
  BALANCED: stage2 3 slot + ingestion 4 job.
- `core/loopgen/ffmpeg_skill.py` — QA `detect_content` (cap semaphore 1).
- `scripts/sync_fleet_queues.py` — sinkronisasi antrean 90 detik.
- `tools/vps_disk_janitor.py` — Disk Janitor (TTL kode 4 jam/12 jam).
- `core/telemetry/` — `collector.py`, `lane_probes.py`, `nodes.py`,
  `score.py` (skor TIDAK memakai lane probe — memakai `evaluate_rules()`).
- `api/routes/telemetry.py` — endpoint `/health`, `/node-report`, `/nodes`,
  `/sentinel-cockpit` (+ `/fleet-overview` dari 0123550, belum aktif di web).

### E. Riwayat commit relevan (24 Nov)
`ddc853a` (telemetry) → `99a037d` (cockpit) → `3b06fba` (prep_worker+T6) →
`12f84ec` (fleet sync B1–B4) → `254fa2a` + `0aee7ad` (ingestion langkah 1–4) →
`0123550` + `930e764` (dashboard telemetry) → `b41beaf` (orkestrasi QoS) →
`b917ad1` (4x concurrency) → `52e74db` (auto-heal staging) = HEAD saat ini.

### F. Aturan yang mengikat pekerjaan ini
- Aplikasi SOLID — setiap perubahan butuh rencana verifikasi pra/pasca +
  rollback + jendela observasi (22 Nov).
- Baseline adalah LANTAI: 206 PUBLISHED/hari, 0 FAILED/BLOCKED,
  delay publish p90 30,2 menit.
- Maksimal 1 restart per 15 menit lintas semua agen.
- Buktikan perbaikan dengan bukti independen (outcome publik + angka DB),
  bukan hanya log.
- Jangan pernah mengklaim hasil tanpa verifikasi.

---

*Dokumen ini disusun dari pemeriksaan langsung ke Master dan VPS 8
(terakhir 06:18 WIB, 24 Nov 2026). Semua angka dapat diverifikasi ulang
dengan perintah-perintah yang tercantum di §6.2 dan §8.*
...[truncated 2085 chars]