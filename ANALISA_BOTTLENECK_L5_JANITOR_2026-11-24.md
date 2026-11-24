# ANALISA BOTTLENECK L5 JANITOR — Kenapa Disk Hampir Penuh

**Tanggal:** 24 November 2026, 04:00 WIB
**Diminta oleh:** Owner (Abdul Aziz) — analisa + saran perbaikan
**Koreksi:** Janitor adalah **L5** (L6 adalah Vault) — sesuai ralat owner.
**Status disk saat analisa:** 126 GB terpakai / 145 GB (88%), sisa 19 GB.

---

## 1. RINGKASAN EKSEKUTIF

Dua penyebab, bukan satu — dan ralat owner benar:

1. **AKUT: Disk Janitor MATI sejak 23 Nov 20:39 WIB.** Perubahan worker
   ke `--lane publish` (commit cea60b8, W2) tanpa sengaja menonaktifkan
   janitor — gerbang lane di kode mengecualikan 'publish'. Janitor
   terakhir jalan 20:07, lalu 0 kali sejak itu.
2. **KRONIS: Cakupan janitor terlalu sempit.** Bahkan saat hidup, janitor
   TIDAK PERNAH membersihkan 40 GB direktori LoopGen (proof, agent-results,
   photo-assets) — konsumen disk terbesar yang tidak terkelola.

---

## 2. BUKTI — JANITOR MATI (Bottleneck 1)

| Bukti | Temuan |
|-------|--------|
| Journal worker | Run terakhir Disk Janitor: **23 Nov 20:07:11** ("Pruned 130 cached files, 3.0 GB freed"). Total 39 run hari itu, semua dari PID 4131759. Setelah 20:07: **0 run**. |
| Restart worker | **23 Nov 20:39:14** — worker restart dengan cmdline `--lane publish` (terverifikasi via /proc/306566/cmdline). |
| Kode (`autonomous_daemon.py:1737-1738`) | `# Periodic VPS Disk Janitor (every 30m)` — `if lane in ('all', 'janitor') and ...` — lane 'publish' TIDAK termasuk. Tick janitor tidak pernah menyala. |
| Service mandiri | `config/systemd/masjavas-janitor.service` + `.timer` ADA di repo, tapi `systemctl list-unit-files \| grep janitor` = **KOSONG — tidak pernah di-install**. Tidak ada fallback. |
| Proses/cron | Tidak ada proses janitor berjalan, tidak ada cron janitor. |

**Seberapa efektif janitor saat hidup:** 6 run terakhir (17:07–20:07,
23 Nov) membebaskan **29,2 GB** (0,66 + 4,27 + 8,61 + 7,15 + 5,51 + 3,0 GB).
Pipeline menghasilkan cache dengan laju sangat tinggi — janitor adalah
satu-satunya yang menahan pertumbuhan workspace.

**Workspace Janitor (prune 6 jam sekali):** run terakhir tercatat 17:07
23 Nov (3 file, 338 MB). Tidak ada baris log sejak restart 20:39 — TAPI
ia hanya mencatat log bila ada file yang dihapus, jadi statusnya TIDAK
BISA dipastikan mati dari log saja. Dampaknya kecil (0,3–1,5 GB/run)
dibanding Disk Janitor (3–8,6 GB/run).

---

## 3. BUKTI — DI MANA DISK HABIS (Bottleneck 2, 3, 4)

Total 126 GB terpakai. Rincian:

| Lokasi | Ukuran | Dibersihkan janitor? |
|--------|--------|----------------------|
| `/opt/masjavas/workspace/channels` | **40 GB** | ✅ YA (target utama — tapi lihat §4) |
| `/opt/masjavas-os/data/loopgen` | **42 GB** | ❌ TIDAK PERNAH |
| └ `photo-assets` (215.403 file gambar) | 14 GB | ❌ |
| └ `proof` (1.290 png + 1.221 mp4 + 817 jpg) | 13 GB | ❌ |
| └ `agent-results` (4.832 png + 1.093 mp4) | 13 GB | ❌ |
| `/opt/masjavas/assets/global_audio_pool` | **16 GB** | ⚠️ Sebagian (cap 15 GB, lihat §5) |
| `/opt/masjavas/assets/content_id_pool` | 5,5 GB | ❌ |
| `/opt/masjavas-os/data/backups` | 3,6 GB | ❌ |
| `/var/log` + `/tmp` + lain-lain | ~5 GB | ❌ (di luar scope janitor) |

**Usia file LoopGen (penting untuk retensi):**
- `proof` > 7 hari: hanya 0,3 GB (128 file) — sebagian besar proof
  berumur < 7 hari. Akumulasi 13 GB terjadi dalam seminggu terakhir.
- `agent-results` > 7 hari: **0 file, 0 GB** — SELURUH 13 GB berumur
  < 7 hari (hasil produksi massal LoopGen baru-baru ini).
- `proof` tertua: 13 Nov 2026 (run-12/13 STAGE2-TEST).

---

## KOREKSI (24 Nov 04:10 WIB — temuan eksekusi pembersihan)

**Klaim TTL 24 jam di §4 dokumen ini SALAH.** Kode janitor ternyata memakai
TTL **4 jam bila disk penuh / 12 jam** (bukan 24 jam seperti docstring).
Buktinya: 1 run janitor (04:04:40) membebaskan 37,99 GB dan workspace
menyusut 40 GB → 1,5 GB. Workspace 40 GB BUKAN steady-state TTL — melainkan
akumulasi karena janitor mati + **33,35 GB file karantina channel UNPAIRED**
(kategori yang tidak tercakup inventaris awal). Dengan janitor hidup + TTL
4 jam, workspace seharusnya tetap kecil (~1–2 GB). Docstring janitor perlu
dikoreksi (24 jam → 4/12 jam). Koreksi ini tidak mengubah Bottleneck 1, 2,
4, 5 — hanya §4.

**Hasil pembersihan (04:04–04:09):** disk 88% → **53%** (76 GB terpakai,
69 GB bebas). Total dibebaskan ~50,6 GB: janitor 37,99 GB + agent-results
12,36 GB + proof 0,26 GB + backup 1,38 MB. Timer janitor AKTIF + enabled
(run tiap 30 menit). Log: `deletion-log-2026-11-24.md`.

## 4. BOTTLENECK 3 — TTL 24 JAM MENJAMIN WORKSPACE ~40 GB

> ⚠️ BAGIAN INI DIKOREKSI — lihat KOREKSI di atas. TTL sebenarnya 4/12 jam.

Komposisi workspace/channels (40 GB, 6.394 file):
- mp4: 19,9 GB (1.009 file) • mp3: 16,0 GB (346 file) • m4a: 2,5 GB
- jpg: 0,6 GB • png/json/txt: < 0,1 GB

**Hanya 118 file (0,6 GB) yang berumur > 24 jam.** Artinya ~38 GB
media workspace berumur < 24 jam — itulah produksi 1 hari pipeline.

Janitor hanya menghapus file > 24 jam. Dengan produksi ~38 GB/hari,
workspace akan SELALU ~40 GB (steady-state 1 hari produksi) walau
janitor sehat. Janitor hanya mencegah pertumbuhan DI ATAS 1 hari —
ia tidak pernah bisa mengecilkan workspace di bawah itu dengan TTL 24 jam.

**Implikasi:** menghidupkan kembali janitor saja TIDAK akan membebaskan
ruang besar SECARA LANGSUNG (hanya 0,6 GB yang eligible saat ini).
Manfaatnya adalah mencegah pertumbuhan lebih lanjut. Ruang besar harus
datang dari tempat lain (LoopGen, pool, backup).

---

## 5. BOTTLENECK 4 — CAP AUDIO POOL MEMBINGUNGKAN

- Docstring janitor: "prune down to <= 5 GB".
- Kode (`vps_disk_janitor.py:49`): `max_pool_bytes = 15 GB`.
- Pool saat ini: **16 GB** — di atas cap kode (15 GB), jauh di atas
  docstring (5 GB).

Bila cap yang benar adalah 5 GB, janitor yang hidup akan membebaskan
~11 GB dari pool. Bila 15 GB, hanya ~1 GB. Ambiguitas ini harus
diputuskan — kode dan dokumen tidak sepakat.

---

## 6. BOTTLENECK 5 — FILE LEASES KOSONG (perlindungan tidak aktif)

Tabel `file_leases`: **0 baris** (aktif 0, kedaluwarsa 0). Tidak ada
apa pun yang mendaftarkan lease. Tampilan cockpit "File Lease: 0 Aktif"
adalah nyata — tapi artinya lapisan perlindungan lease TIDAK BERFUNGSI.
Janitor tidak punya informasi lease untuk melindungi file yang masih
dipakai (ia hanya mengandalkan guard "tugas aktif").

---

## 7. SARAN & IDE PERBAIKAN

### P0 — Hidupkan kembali janitor (pilih satu)

| Opsi | Cara | Kelebihan | Kekurangan |
|------|------|-----------|------------|
| **A (REKOMENDASI)** | Install `masjavas-janitor.service` + `.timer` yang sudah ada di repo (timer 30 menit) | Independen dari worker/lane — selamat dari restart worker & perubahan lane di masa depan | Perlu install unit (1 perintah + enable) |
| B | Ubah gerbang lane jadi `('all', 'janitor', 'publish')` | Cepat (1 baris) | Janitor kembali bergantung pada worker — mati lagi bila worker mati/ganti lane |
| C | Worker khusus `--lane janitor` | Isolasi penuh | Boros (proses worker penuh hanya untuk janitor) |

Rekomendasi: **A + pertahankan janitor in-worker sebagai cadangan**
(dengan gerbang diperbaiki ke B). Dua lapis, saling melengkapi.

### P0 — Persingkat TTL workspace (24 jam → 6–12 jam)

File dikonsumsi publish dalam hitungan jam, bukan hari. TTL 12 jam
memotong steady-state workspace dari ~40 GB menjadi ~20 GB; TTL 6 jam
menjadi ~10 GB. Guard "tugas aktif" yang sudah ada melindungi file
yang sedang dipakai. Verifikasi dulu: tidak ada tugas yang membaca
cache > 6 jam setelah dibuat.

### P1 — Perluas cakupan janitor ke LoopGen

| Direktori | Ukuran | Usulan retensi |
|-----------|--------|----------------|
| `proof/` | 13 GB | Hapus > 7 hari (saat ini hanya 0,3 GB yang > 7 hari — kebijakan ini mencegah pertumbuhan, bukan membebaskan banyak sekarang) |
| `agent-results/` | 13 GB | Hapus setelah terdistribusi ke Drive (verifikasi: file sudah ada di Drive sebelum hapus lokal) — potensi bebas terbesar |
| `photo-assets/` | 14 GB | Perlu keputusan: arsip ke Drive/cold storage, atau retensi N hari per channel |

### P1 — Putuskan cap audio pool (5 GB vs 15 GB)

Bila 5 GB yang benar: perbaiki kode (1 baris) → bebas ~11 GB saat
janitor berikutnya jalan. Bila 15 GB: perbaiki docstring.

### P1 — Retensi backup (3,6 GB)

Aturan: simpan 7 hari terakhir ATAU N terbaru per jenis backup,
hapus sisanya otomatis. Backup telemetry/cockpit terbaru (145 MB +
kecil) dipertahankan sebagai rollback aktif.

### P2 — Observabilitas janitor (agar kematian 7,5 jam tak terulang)

- L5 probe: laporkan **run terakhir janitor + GB yang dibebaskan**
  (data sudah ada di tabel `deletion_audit` — janitor mencatat tiap
  penghapusan ke sana).
- Alert STALE bila tidak ada run janitor > 1 jam.
- Inilah yang membuat kematian janitor tidak terdeteksi: cockpit
  menampilkan disk + lease, tapi TIDAK PERNAH menampilkan apakah
  janitor masih hidup.

### P2 — Isi atau hapus file_leases

Daftarkan lease dari worker saat file dibuat/dipakai, ATAU hapus
metrik lease dari cockpit bila sistem lease memang tidak dipakai.
Metrik yang selalu 0 adalah noise.

### P2 — Alert pertumbuhan disk

Alert WARNING di 85% (sudah terlampaui), CRITICAL di 90%.
Saat ini tidak ada alert otomatis untuk disk.

---

## 8. PEMBEBASAN SEGERA (butuh persetujuan owner — penghapusan file)

| Tindakan | Potensi bebas | Risiko | Status |
|----------|---------------|--------|--------|
| Hidupkan janitor (Opsi A) | ~0,6 GB sekarang; mencegah +38 GB/hari | Rendah (guard tugas aktif) | Menunggu keputusan |
| Hapus `agent-results/` yang sudah di Drive | s.d. 13 GB | Sedang — WAJIB verifikasi Drive dulu per file | Menunggu keputusan + verifikasi |
| Cap pool 5 GB | ~11 GB | Rendah — pool adalah cache, bisa diunduh ulang | Menunggu keputusan cap |
| Hapus `proof/` > 30 hari | Kecil (< 0,3 GB) | Rendah | Bisa jalan dengan retensi |
| Bersihkan backup lama | s.d. ~2 GB | Rendah bila retensi jelas | Menunggu keputusan retensi |

**TIDAK ADA penghapusan yang dilakukan dalam analisa ini.**
Semua tindakan di atas menunggu persetujuan owner.

---

## 9. LINI WAKTU KEJADIAN

- **13 Nov:** proof tertua dibuat (run-12/13 STAGE2-TEST).
- **18 Nov:** peluncuran massal LoopGen 10 channel — agent-results
  mulai menumpuk (13 GB dalam < 7 hari terakhir).
- **23 Nov 04:28–20:07:** janitor bekerja normal (39 run, membebaskan
  puluhan GB).
- **23 Nov 19:18:** commit cea60b8 (W2: worker `--lane publish`).
- **23 Nov 20:07:11:** run Disk Janitor TERAKHIR (3,0 GB dibebaskan).
- **23 Nov 20:39:14:** worker restart dengan `--lane publish` →
  **janitor mati** (gerbang lane).
- **24 Nov 03:20:** disk 88% (19 GB bebas) — dilaporkan di laporan kinerja.
- **24 Nov 04:00:** analisa ini. Janitor sudah mati ~7,5 jam.
