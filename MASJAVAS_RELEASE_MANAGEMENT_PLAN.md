# RENCANA MANAJEMEN RILIS — MASJAVAS TUBE OS V2

**Status:** DOKUMEN RENCANA (Workstream F). Release manifest + kompatibilitas
versi adalah komponen WAJIB sebelum worker tersebar di beberapa VPS
(V2.0 §12). Prosedur di sini berlaku untuk SEMUA deployment produksi —
Master maupun Live-01 — setelah V2 diimplementasikan; aturan restart
(§8) berlaku SEJAK SEKARANG karena sudah terbukti oleh insiden.
**Sumber utama:** `MASTER_ARCHITECTURE_MASJAVAS_TUBE_OS_V2.md` §12
(Deployment Manager), §9.4, §10; `IMPLEMENTATION_CHECKLIST_..._V2.md`
§0 (aturan eksekusi), Track P9, Track P10; `PLAN_PENAMBAHAN_VPS8_2026-11-22.md`
§6; `v2-002/ENCODER114_FIX_REPORT.md` (pelajaran deploy 22 Nov).
**Terakhir diperbarui:** 22 Nov 2026.

**Label bukti:** `[VERIFIED FACT]` / `[DOCUMENTED FACT NOT YET REVALIDATED]` /
`[ASSUMPTION]` / `[PROPOSAL]` / `[OPEN QUESTION]` /
`[OWNER DECISION REQUIRED]`.

---

## 1. Release manifest (V2.0 §12)

Setiap deployment memiliki manifest bersama:

| Field | Isi |
|---|---|
| `release_id` | Identitas unik rilis. |
| `git_commit` | Commit yang di-deploy (contoh nyata: `56f7a6d`, `021a9f0` — perbaikan encoder 114, 22 Nov 2026). |
| `schema_version` | Versi skema database yang dipakai rilis ini. |
| `worker_protocol_version` | Versi protokol worker↔Master. |
| `config_version` | Versi konfigurasi (knob `.env`/settings yang relevan). |
| `created_at` | Waktu pembuatan rilis. |

Penyimpanan manifest `[PROPOSAL]`: file manifest di repo (satu per
rilis, berversi git) + dicatat di database saat deploy agar Master
dapat memeriksa kompatibilitas worker terhadapnya. Lokasi final
ditetapkan di Track P9.

**Aturan Checklist §0.1:** setiap deployment memiliki release ID dan
post-deployment verification. Tanpa keduanya, deployment tidak dianggap
selesai.

## 2. Pelaporan versi worker

Worker melaporkan identitas versi saat **register dan heartbeat**
(V2.0 §12). Master HANYA memberi job bila SEMUA syarat terpenuhi:

1. Protocol version kompatibel.
2. Schema version kompatibel.
3. Capability tersedia.
4. Release memenuhi constraint job.
5. Worker tidak berada dalam maintenance mode.

Kriteria penerimaan P9: **tidak ada job dikirim ke worker
inkompatibel.** Pelanggaran terhadap aturan ini adalah cacat P0 —
worker inkompatibel yang menerima job dapat menulis data dengan
skema yang salah.

## 3. Migrasi database — expand-and-contract

- Gunakan migrasi **expand-and-contract** (V2.0 §12): tambah kolom/
  tabel baru dulu; kode baru menulis ke keduanya; HANYA setelah
  seluruh worker dan release menggunakan skema baru, kolom lama
  boleh dihapus.
- Setiap perubahan database mempunyai migration DAN rollback plan
  (Checklist §0.1).
- **Larangan mengikat (Checklist §0.2):** jangan membuat tabel `jobs`
  baru sebelum audit `operational_tasks` (V2-007). Tidak ada migrasi
  skema sebelum V2-007 disetujui — gerbang teknis.
- **Larangan mengikat (Checklist §0.2):** jangan mengganti SQLite
  dengan PostgreSQL tanpa trigger yang terbukti (V2.0 §9.4).

## 4. Deployment policy (V2.0 §12)

Urutan baku setiap rilis:

1. Build dan test release.
2. Jalankan migration yang backward-compatible.
3. Deploy Master terlebih dahulu JIKA API baru tetap kompatibel
   dengan worker lama.
4. Deploy satu worker canary.
5. Jalankan smoke test.
6. Perluas rollout.
7. Hentikan pemberian job ke worker lama.
8. Drain job aktif sebelum restart.
9. Rollback jika acceptance gate gagal.

**Aturan tambahan (Checklist §0.1–§0.2):** setiap perubahan
production diuji di lingkungan terbatas terlebih dahulu; tidak ada
perubahan massal tanpa canary; jangan menjalankan FFmpeg berat di
dalam web/API; jangan melakukan migrasi Master bersamaan dengan
cutover worker.

## 5. Canary

- **Definisi:** satu worker (atau satu stream, untuk live) menerima
  rilis baru lebih dulu; sisanya tetap di rilis lama sampai canary
  lolos observasi.
- **Canary live (Plan §6 langkah 1.3):** 1–2 stream dipindah dulu →
  observasi → semua. Kriteria lolos: stream stabil 72 jam; TX
  governor menolak di atas batas; Master responsif.
- **Canary prep (Plan §6 langkah 1.5):** ingestion 1 job + Stage 2
  1 slot. Kriteria lolos: klaim API jalan; 0 gangguan live; guard
  lokal bekerja.
- **Canary Stage 2 (V2.0 §4.2):** mulai dengan DUA slot. Naik ke
  tiga/empat HANYA bila pengukuran CPU, RAM, I/O, disk, dan live TX
  membuktikan aman. CPUQuota tidak boleh ditetapkan dari pembagian
  angka yang tampak rapi — harus dari benchmark (V2-008).
- **Smoke test pasca-canary [PROPOSAL]:** healthz merespons; worker
  register + heartbeat; satu job klaim→selesai end-to-end; satu
  reconnect encoder (bila live tersentuh); tidak ada error class baru
  di log. Daftar smoke test final ditetapkan di Track P9.

## 6. Staged rollout + drain

- Rollout diperluas bertahap (canary → sebagian → semua), bukan
  sekaligus. Kriteria penerimaan P9: **deployment tidak memutus
  semua live bersamaan.**
- **Drain sebelum restart:** hentikan pemberian job ke worker lama,
  tunggu job aktif selesai (atau capai batas drain yang ditetapkan),
  BARU restart. Job yang tidak selesai dalam batas drain kembali ke
  antrean lewat lease expiry (V2.0 §8.3) — tidak boleh hilang dan
  tidak boleh menghasilkan side effect ganda (kriteria P10).
- **Maintenance mode:** worker dalam maintenance tidak menerima job
  baru (V2.0 §12, P9). Mode ini dipakai untuk drain + restart
  terencana.

## 7. Rollback

| Jenis rollback | Mekanisme |
|---|---|
| Rilis (kode) | Deploy ulang rilis sebelumnya (git revert / checkout commit sebelumnya). Kriteria P9: rollback release dapat dilakukan TANPA merusak schema. |
| Migrasi | Expand-and-contract memungkinkan rollback kode tanpa rollback skema (kolom lama masih ada sampai contract). Bila contract sudah jalan, rollback skema memerlukan migration balik yang sudah disiapkan SEBELUM deploy. |
| Per langkah Fase 0/1 | Rollback per langkah sesuai Plan §6 (contoh: 0.1 kembali ke 1 unit; 1.2 matikan Live-01, live kembali ke Master; 1.3 pindahkan stream kembali ke Master; 1.5 matikan worker VPS 8). |
| Konfigurasi `.env` | Backup `.env` (0600) sebelum perubahan — praktik yang dipakai saat deploy R4 22 Nov 2026 (`data/backups/.env.bak-encoder114-20261122`). |

**Pemicu rollback otomatis [PROPOSAL — dari gate policy V2.0 §18 +
Plan §6]:** satu metrik scorecard kritis memburuk = gerbang gagal =
rollback. Metrik dan ambang ada di
`MASJAVAS_PRODUCTION_ACCEPTANCE_CRITERIA.md`.

## 8. Aturan restart (berlaku SEJAK SEKARANG)

Aturan ini sudah terbukti oleh insiden nyata — bukan teori:

1. **Maks 1 restart per 15 menit lintas agent** (aturan lintas-agent
   yang sudah berlaku). Sebelum restart, VALIDASI perhitungan jeda:
   angka sebesar epoch (atau 0/negatif) berarti pembacaan timestamp
   GAGAL — jangan pernah melewati sleep atas dasar itu. Bila
   timestamp tidak terbaca, tunggu 15 menit penuh dari restart
   terakhir yang DIKETAHUI.
   `[VERIFIED FACT — pelanggaran 19 Nov dicatat di AGENTS.md]`
2. **Sesi SSH panjang putus ~40–60 dtk** (proxy). Restart service
   yang memakan waktu lama HARUS via `nohup ... &` + polling, dan
   sleep dilakukan di exec LOKAL sebelum polling SSH pendek — JANGAN
   sleep panjang di dalam satu sesi SSH.
   `[VERIFIED FACT — pola 19–22 Nov]`
3. **Stop service live memakan ~95 dtk** (SIGTERM timeout → SIGKILL;
   pola yang sama pada restart 14:42 dan 07:19, 22 Nov). Rencanakan
   jendela downtime sesuai angka ini, bukan asumsi restart instan.
   `[VERIFIED FACT — 22 Nov 07:19]`
4. **Verifikasi pasca-restart HARUS memeriksa status DB, bukan hanya
   proses.** Pelajaran insiden 22 Nov 07:17–07:24: watchdog web
   menandai stream 114 ERROR setelah 5× pemulihan gagal (alasan
   kegagalan TIDAK DIKETAHUI — tidak ada log Auto-Healing di journal
   mana pun); supervisor baru mengabaikan task ERROR sehingga
   "pemulihan otomatis" TIDAK terjadi; pemulihan membutuhkan flip
   status ERROR → STREAMING (1 baris DB) + reconnect. Downtime total
   ±7 menit.
   `[VERIFIED FACT — v2-002/ENCODER114_FIX_REPORT.md]`
5. **Verifikasi env proses baru** (contoh: `LIVE_MAX_TX_MBPS=500`
   dicek di `/proc/<pid>/environ` pasca-restart 07:19).
6. **Restart web adalah perubahan terpisah.** Proses web masih
   menjalankan kode lama di memori sampai di-restart (catatan 22 Nov:
   R5 belum dimuat di jalur API karena web tidak di-restart saat
   deploy live). Jadwalkan restart web di jendela maintenance
   tersendiri; jangan asumsikan deploy live = semua proses memakai
   kode baru.
7. **Jangan restart worker saat antrean berisi tugas target kemarin
   tanpa memverifikasi heal akan memulihkan dengan step terlindungi**
   (pelajaran 19 Nov: 15 tugas hilang karena heal memakai step
   `'QUEUED'` yang tidak dilindungi populasi startup; perbaikan
   commit `b120e1a`).

## 9. Post-deployment verification + observasi

- **Verifikasi langsung:** service aktif; versi proses sesuai
  manifest (cek commit/kode di proses yang berjalan); status DB
  semua stream/task sesuai ekspektasi (§8 butir 4); guard tick
  berjalan normal; healthz merespons.
- **Observasi minimal 72 jam** untuk perubahan operasional yang
  relevan (Checklist §0.1). Metrik dengan baseline 7 hari tetap
  dinilai dengan window 7 hari — jangan membandingkan baseline
  7 hari dengan hasil 72 jam seolah setara (V2.0 §18).
- **Satu metrik kritis memburuk = gate gagal = rollback** (§7).
- **Jendela observasi deploy encoder 22 Nov** (contoh berjalan):
  24 jam pasca-deploy iterasi 2 — hitungan kematian 114, watchdog
  false-positive kill, event refresh token 3,5 jam (PID feeder
  berubah), perilaku alert R7 (tidak spam).

## 10. Checklist rilis (template)

- [ ] Task ID + kondisi sebelum perubahan + daftar file yang diubah.
- [ ] Migration + rollback plan (bila ada perubahan DB).
- [ ] Konfigurasi lama dapat dipulihkan (bila ada perubahan systemd).
- [ ] Tes di lingkungan terbatas (isolasi: tes TIDAK menulis DB live).
- [ ] Release manifest lengkap (§1).
- [ ] Backup pra-perubahan (`.env` 0600 + file yang diubah).
- [ ] Persetujuan owner untuk Level C (deploy/restart produksi).
- [ ] Deploy sesuai §4 (canary dulu).
- [ ] Post-deployment verification (§9) + cek status DB (§8.4).
- [ ] Observasi 72 jam / 7 hari + catatan hasil.
- [ ] Rollback dieksekusi bila gate gagal; hasil dicatat jujur.

## 11. Pertanyaan terbuka

1. Lokasi penyimpanan release manifest (repo saja vs repo + DB) —
   `[OPEN QUESTION — Track P9]`.
2. Daftar smoke test final per jenis rilis — `[OPEN QUESTION — Track P9]`.
3. Batas waktu drain sebelum job dikembalikan ke antrean —
   `[OPEN QUESTION — nilai detik/menit belum ditetapkan]`.
4. Apakah rollback migrasi yang sudah contract memerlukan downtime
   terjadwal — `[OPEN QUESTION]`.
5. Mekanisme distribusi manifest ke worker (bersama kode vs API) —
   `[OPEN QUESTION — Track P9]`.
