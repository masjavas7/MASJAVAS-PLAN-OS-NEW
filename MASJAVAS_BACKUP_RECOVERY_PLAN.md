# RENCANA BACKUP & RECOVERY — MASJAVAS TUBE OS V2

**Status:** DOKUMEN RENCANA (Workstream E). Belum ada implementasi backup
off-site; semua prosedur di sini adalah rancangan sampai V2-009
(gerbang biaya) lolos dan Track P8 dieksekusi.
**Sumber utama:** `MASTER_ARCHITECTURE_MASJAVAS_TUBE_OS_V2.md` §14
(Backup, DR, Storage Economics), §5, §19; `IMPLEMENTATION_CHECKLIST_..._V2.md`
Track P8 (P8.4); `PLAN_PENAMBAHAN_VPS8_2026-11-22.md` §6 (langkah 1.4).
**Terakhir diperbarui:** 22 Nov 2026.

**Label bukti:** `[VERIFIED FACT]` / `[DOCUMENTED FACT NOT YET REVALIDATED]` /
`[ASSUMPTION]` / `[PROPOSAL]` / `[OPEN QUESTION]` /
`[OWNER DECISION REQUIRED]`.

---

## 1. Prinsip (V2.0 §14.1)

Pisahkan tiga hal — ketiganya BUKAN pengganti satu sama lain:

| # | Lapisan | Fungsi |
|---|---|---|
| 1 | **Live operational recovery** | Restart / recovery proses live (supervisor, watchdog). |
| 2 | **Warm DR replica** | Salinan di VPS 8 yang mempercepat pemulihan operasional (Plan §6 langkah 1.4). |
| 3 | **Independent backup** | Salinan terenkripsi dan versioned **di luar kedua VPS**. |

**Aturan mengikat [PROPOSAL — dari V2.0 §14.1, disetujui sebagai desain]:**
VPS 8 TIDAK BOLEH menjadi satu-satunya lokasi backup Master. Bila kedua
VPS mengalami masalah pada provider atau akun yang sama (keduanya Contabo),
salinan di VPS 8 dapat ikut hilang.

**Aturan kedua [V2.0 §14.3]:** jangan menganggap object storage Contabo
atau layanan lain sudah termasuk dalam anggaran VPS. Biaya storage
dihitung terpisah dan masuk total anggaran bulanan.

## 2. Cakupan backup — apa yang dibackup

| Item | Lokasi / sumber | Ukuran | Catatan |
|---|---|---|---|
| Database SQLite | `/opt/masjavas-os/data/masjavas_tube.sqlite` | 118 MB `[VERIFIED FACT — 22 Nov 2026]` | Satu-satunya sumber kebenaran job ownership. Backup harus konsisten (lihat §3). |
| File `.env` | `/opt/masjavas-os/.env` | Tidak dicatat | Berisi secret. Backup HANYA dalam bentuk terenkripsi; JANGAN pernah menulis nilainya di dokumen/log/chat. `[VERIFIED FACT — lokasi]` |
| Kode aplikasi | `/opt/masjavas-os/` (checkout git) | Tidak dicatat | Remote: repo privat `masjavas7/MASJAVASTUBE-OS`. Backup = git remote + checkout lokal. `[VERIFIED FACT — remote]` |
| Backup lokal yang sudah ada | `/opt/masjavas-os/data/backups/` | Tidak dicatat | Berisi backup `.env` (0600) + backup file pra-patch. Disalin ke off-site sebagai lapisan tambahan, bukan pengganti backup terstruktur. `[VERIFIED FACT — keberadaan]` |
| Staging ingestion | Direktori staging ingestion | ±100 GB `[DOCUMENTED FACT NOT YET REVALIDATED — estimasi V2.0 §4.4, bukan final]` | Termasuk salinan pool ±20,5 GB (Plan §4). Perlu audit volume aktual sebelum angka final (V2.0 §4.4). |
| Kerja + cache Stage 2 | Direktori kerja Stage 2 | ±50 GB `[DOCUMENTED FACT NOT YET REVALIDATED — estimasi V2.0 §4.4]` | Termasuk data LoopGen ±30 GB (Plan §4). |
| File loop live | Direktori file video live | ±5 GB `[DOCUMENTED FACT NOT YET REVALIDATED — estimasi V2.0 §4.4]` | File video yang diputar encoder. |
| Log operasional | `data/live_logs/`, `data/logs/` | Tidak dicatat | Bukti forensik (log kematian encoder, relay). Retensi log belum ditetapkan `[OPEN QUESTION]`. |
| Konfigurasi systemd | Unit `masjavas-os-*`, `masjavas-live` | Kecil | Setiap perubahan systemd harus punya konfigurasi lama yang dapat dipulihkan (Checklist §0.1). |
| Salinan DR (VPS 8) | VPS 8 (rencana) | ±100 GB `[DOCUMENTED FACT NOT YET REVALIDATED — estimasi V2.0 §4.4]` | Salinan app + DB + aset (Plan §4). Bukan backup independen (lihat §1). |

**Total estimasi kebutuhan disk VPS 8 ±255 GB** (sebelum OS, log, cache,
ruang bebas) `[DOCUMENTED FACT NOT YET REVALIDATED — V2.0 §4.4]` —
dengan disk 300 GB, ruang bebas minimum 50 GB TIDAK otomatis tersedia.
Rekonsiliasi disk adalah Track P6.3 dan prasyarat sebelum angka backup
dinyatakan final.

**Yang TIDAK masuk backup off-site [PROPOSAL]:** cache yang dapat
dibangun ulang (thumbnail cache, unduhan sementara yt-dlp), file
`__pycache__`, dan artefak TEMPORARY yang belum tervalidasi (V2.0 §11).
Daftar eksklusi final ditetapkan saat implementasi P8.4.

## 3. Restic — backup terenkripsi dan versioned [PROPOSAL]

Teknologi kandidat dari V2.0 §14.2. Rincian implementasi adalah Track P8.4.

- **Repository:** satu repository Restic per lingkungan (produksi).
  Backend object storage independen — provider BELUM DIPILIH
  `[OWNER DECISION REQUIRED — bagian dari V2-009]`.
- **Enkripsi:** repository terenkripsi. Kunci enkripsi disimpan
  TERPISAH dari repository dan dari kedua VPS (prinsip: kunci tidak
  boleh hilang bersama data yang dilindunginya). Mekanisme penyimpanan
  kunci yang tepat (vault owner) `[OPEN QUESTION — diputuskan bersama
  pemilihan backend]`.
- **Backup SQLite konsisten:** backup TIDAK BOLEH menyalin file DB
  yang sedang ditulis begitu saja. Metode kandidat: API backup
  bawaan SQLite (`sqlite3.Connection.backup()`) atau `VACUUM INTO`
  ke file snapshot, lalu Restic membackup snapshot itu. Metode final
  diuji di P8.4 ("Uji backup SQLite konsisten").
- **Retensi (target V2.0 §14.4):** 7 backup harian, 4 backup mingguan,
  backup bulanan sesuai kebutuhan retensi. Retensi final menyesuaikan
  biaya storage (gerbang §8).
- **Jadwal [PROPOSAL]:** backup harian di luar jam sibuk publish/live;
  backup TIDAK BOLEH mengganggu live (kriteria penerimaan P8.4).
  Jendela jam yang tepat ditetapkan setelah pengukuran beban
  (diketahui: wave publish + broadcast window 17:00–22:30).
- **Backup secret:** "Backup secret dengan mekanisme aman" (P8.4).
  `.env` dan kredensial lain hanya masuk repository terenkripsi;
  TIDAK PERNAH masuk repo git atau dokumen.

## 4. Litestream — replikasi WAL (opsional) [PROPOSAL]

Teknologi kandidat dari V2.0 §14.2. Status: OPSIONAL, dievaluasi setelah
Restic berjalan.

- **Fungsi:** replikasi SQLite WAL ke object storage; mendukung
  pemulihan ke titik waktu tertentu sesuai konfigurasi.
- **Batasan tegas [V2.0 §14.2]:** Litestream BUKAN pengganti backup
  historis yang terpisah. Ia tidak menggantikan Restic.
- **Prasyarat pengujian (P8.4):** evaluasi pada SALINAN database dulu;
  periksa dampak tabel `_litestream_lock`; aktifkan replikasi hanya
  setelah pengujian.
- **Biaya:** dihitung terpisah dalam gerbang storage (§8).
  Litestream + Restic keduanya TERGATE persetujuan storage/biaya —
  tidak ada yang diaktifkan sebelum V2-009 lolos.

## 5. Prosedur restore

### 5.1 Restore database (skenario utama)

1. Ambil snapshot Restic terbaru yang lolos integrity check
   (atau titik waktu Litestream bila tersedia).
2. Restore ke **lingkungan terisolasi** — JANGAN menimpa DB live
   sebelum verifikasi selesai (V2.0 §14.4).
3. Verifikasi integrity (`PRAGMA integrity_check`).
4. Verifikasi jumlah record tabel kunci (bandingkan dengan catatan
   pra-insiden bila ada; baseline jumlah record ditetapkan saat
   implementasi P8.4 `[OPEN QUESTION — baseline belum ada]`).
5. Verifikasi file kritis ikut ter-restore.
6. Uji aplikasi membaca hasil restore (mode baca saja dulu).
7. Bila lolos: jadwalkan cutover dengan owner (Level C — perubahan
   produksi). Catat waktu pemulihan aktual untuk mengukur RTO.

### 5.2 Restore aplikasi + konfigurasi

1. Checkout kode dari git remote (commit sesuai release manifest).
2. Restore `.env` dari backup terenkripsi (dekripsi di host target;
   JANGAN pernah menampilkan nilainya).
3. Restore unit systemd + konfigurasi lokal.
4. Verifikasi service aktif + healthz merespons + guard tick berjalan.

### 5.3 Restore DR dari VPS 8 (bila Master down)

Prosedur detail mengikuti Plan §6 langkah 1.4 ("Restore uji berhasil —
DB + app jalan dari salinan"). Catatan: DR replica mempercepat pemulihan
operasional tetapi BUKAN pengganti backup independen (§1). Prosedur
failover live (manual vs otomatis) masih `[OPEN QUESTION — Plan §10
keputusan #4, V2.0 §24]`.

## 6. Restore drill — bulanan (V2.0 §14.4)

Setiap bulan, tanpa kecuali:

1. Restore database ke lingkungan terisolasi.
2. Verifikasi integrity.
3. Verifikasi jumlah record.
4. Verifikasi file kritis.
5. Uji aplikasi membaca hasil restore.
6. **Catat waktu pemulihan** (bukti RTO).

Hasil drill dicatat di `AUTONOMOUS_WORK_LOG.md` (atau log operasional
yang ditetapkan kemudian) dengan: tanggal, snapshot yang dipakai,
waktu restore, hasil tiap verifikasi, dan temuan. Drill yang gagal
adalah P1 — dilaporkan ke owner, bukan didiamkan.

## 7. Target recovery — TARGET, bukan janji

| Target | Nilai | Status |
|---|---|---|
| RPO (Recovery Point Objective) | ≤5 menit | `[PROPOSAL — TARGET V2.0 §14.4, BELUM DIBUKTIKAN]` |
| RTO (Recovery Time Objective) | ≤30 menit | `[PROPOSAL — TARGET V2.0 §14.4, BELUM DIBUKTIKAN]` |

RPO/RTO adalah target sementara sampai diuji (V2.0 §14.4: "Semua ini
target sementara sampai diuji"). Kriteria penerimaan P8.4: "RPO dan RTO
dibuktikan, bukan diasumsikan." Pengukuran dilakukan pada drill pertama
yang berhasil (P8.4: "Ukur RPO. Ukur RTO.") — bila hasil ukur tidak
mencapai target, target direvisi berdasarkan bukti, bukan dipertahankan
sebagai janji.

## 8. Storage cost gate (V2.0 §14.3) — prasyarat adopsi

Sebelum Restic/Litestream diadopsi, delapan langkah ini harus selesai
(bagian dari V2-009):

1. Pilih backend object storage.
2. Hitung biaya per GB.
3. Hitung biaya request dan egress.
4. Estimasikan pertumbuhan database dan aset.
5. Tentukan retensi.
6. Hitung biaya Litestream.
7. Hitung biaya Restic.
8. Masukkan ke total anggaran bulanan (cap mengikat $34,67/bln —
   total riil di atas cap = eskalasi, jangan lanjut diam-diam).

Status saat ini: **NOT READY** (dashboard: "Storage and backup costs
calculated — NOT READY; provider + retensi belum dipilih").
`[VERIFIED FACT — status dashboard 22 Nov 10:50 WIB]`.

## 9. Risiko

| Risiko | Mitigasi |
|---|---|
| Backup corrupt → recovery gagal | Integrity check + restore drill bulanan (V2.0 §19). |
| Kunci enkripsi hilang | Kunci disimpan terpisah dari repository (mekanisme final OPEN QUESTION §3). |
| Backup mengganggu live | Jadwal off-peak + throttle; kriteria P8.4 melarang gangguan live. |
| Kedua VPS di provider/akun sama | Backup independen di luar kedua VPS (§1). |
| DB berubah selama backup | Snapshot konsisten via API backup/VACUUM INTO (§3), bukan salin file mentah. |
| Retensi tidak cukup untuk forensik | Retensi ditetapkan bersama biaya (§8); log operasional masuk cakupan (§2). |

## 10. Pertanyaan terbuka

1. Provider object storage + biaya per GB/request/egress — `[OWNER DECISION REQUIRED — V2-009]`.
2. Mekanisme penyimpanan kunci enkripsi Restic — `[OPEN QUESTION]`.
3. Baseline jumlah record untuk verifikasi restore — `[OPEN QUESTION — ditetapkan di P8.4]`.
4. Retensi log operasional — `[OPEN QUESTION]`.
5. Failover live manual vs otomatis — `[OPEN QUESTION — Plan §10 #4]`.
6. Angka volume aktual tiap direktori (audit P6.3) — `[OPEN QUESTION — estimasi §2 belum final]`.
