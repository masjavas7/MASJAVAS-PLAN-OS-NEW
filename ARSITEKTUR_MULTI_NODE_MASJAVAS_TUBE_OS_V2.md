# ARSITEKTUR MULTI-NODE MASJAVAS TUBE OS — Desain Frontend & Backend

**Versi:** 1.0 — 23 November 2026, 20:02 WIB
**Status:** DRAFT untuk persetujuan owner (Abdul Aziz)
**Penulis:** Muse (berdasarkan insiden nyata 22–23 Nov 2026)
**Latar:** Rencana ke depan akan menambah VPS 8 lagi dan lagi (live-02, live-03,
...). Arsitektur ini ditentukan SEKARANG agar insiden yang terjadi pada migrasi
live-01 tidak terulang pada setiap penambahan node baru.

---

## 1. LATAR BELAKANG — Insiden yang Memotivasi Desain Ini

Semua masalah di bawah ini terjadi NYATA pada 22–23 Nov 2026 saat migrasi ke
VPS 8 (live-01). Setiap masalah dipetakan ke bagian desain yang mencegahnya.

| # | Insiden | Tanggal | Dampak | Dicegah oleh |
|---|---------|---------|--------|--------------|
| I1 | Dashboard tampil "Siaran dihentikan" padahal stream LIVE di VPS 8 | 23 Nov 19:54 | Operator MONA panik, klik "Mulai Siar" | §4.1, §6.1 |
| I2 | Auto-heal Master restart task yang sudah migrasi (encoder duplikat) | 23 Nov 16:44, 17:07 | 6 encoder untuk 3 channel | §5.2, §8 |
| I3 | Penugasan node via hack `error_message` ("migrated_to_live-01") | 23 Nov 15:51 | Tidak ada kolom penugasan resmi; mekanisme tidak terdokumentasi | §4.1, §5.1 |
| I4 | Split-brain: 2 DB (Master + VPS 8) dengan status berbeda | 23 Nov 16:43 | Master STOPPED, VPS 8 STREAMING — tidak ada kebenaran tunggal | §3.1, §4.1 |
| I5 | `MASJAVAS_ASSIGNED_SHARDS` kosong — mekanisme penugasan tidak ada | 23 Nov 16:43 | Tidak jelas siapa yang memutuskan node mana menjalankan apa | §5.1 |
| I6 | Monitor hanya cek Master — buta terhadap VPS 8 | 23 Nov 18:39 | SSH gagal = tidak ada visibilitas sama sekali | §9 |
| I7 | Prep worker (ingestion/Stage 2) tidak punya desain distribusi | 23 Nov 19:41 | Masih di Master; tidak ada rencana penempatan di N node | §7 |
| I8 | Guard hanya di 1 lapisan (`start_stream`), belum aktif di live/worker | 23 Nov 19:41 | Guard ada di kode tapi tidak berjalan (service belum restart) | §8 |
| I9 | Kematian massal encoder (8×) tanpa penyebab teridentifikasi | 22–23 Nov | Stream mati bersamaan, penyebab tidak diketahui | §9.3 |
| I10 | Kuota tercatat per-DB, tidak ada pandangan armada | 23 Nov 15:20 | Ledger stage 2 baru bekerja; tidak ada agregasi | §4.3 |

**Prinsip desain:** Setiap insiden di atas HARUS memiliki jawaban arsitektural
di dokumen ini. Bila ada insiden baru di masa depan, dokumen ini diperbarui
dengan baris baru di tabel + bagian desain yang sesuai.

---

## 2. PRINSIP ARSITEKTUR (7 prinsip mengikat)

Prinsip ini TIDAK BOLEH dilanggar oleh implementasi apa pun. Bila ada konflik
antara prinsip dan kemudahan implementasi, prinsip menang.

| # | Prinsip | Artinya |
|---|---------|---------|
| P1 | **Satu sumber kebenaran** | Master DB adalah SATU-SATUNYA tempat kebenaran status task. Node TIDAK punya DB status sendiri. |
| P2 | **Penugasan eksplisit** | Setiap task punya `assigned_node_id` yang jelas di Master DB. Tidak ada penugasan implisit, tidak ada hack string di `error_message`. |
| P3 | **Kepemilikan berbasis lease** | Hanya node pemegang lease yang valid boleh menjalankan task. Lease punya TTL + heartbeat. |
| P4 | **Mutasi hanya via API** | Node TIDAK PERNAH menulis Master DB langsung. Semua perubahan status via API Master yang terautentikasi. |
| P5 | **Gagal-aman (fail-safe)** | Bila ragu, JANGAN start. Menolak start lebih aman daripada encoder duplikat. |
| P6 | **Teramati secara default** | Setiap perubahan status tercatat (log + event) dan terlihat di dashboard. Tidak ada perubahan diam-diam. |
| P7 | **Operasi agnostik-node** | Semua operasi (start/stop/migrate/restart) bekerja sama, tidak peduli node mana yang memegang task. Operator tidak perlu tahu node mana. |

---

## 3. ARSITEKTUR SISTEM

### 3.1 Gambaran umum

```
                    ┌─────────────────────────────────┐
                    │         MASTER (Control Plane)   │
                    │  ┌──────────┐  ┌──────────────┐ │
                    │  │ Web/API  │  │  Scheduler   │ │
                    │  │ Dashboard│  │  + Governor  │ │
                    │  └────┬─────┘  └──────┬───────┘ │
                    │       │               │          │
                    │  ┌────┴───────────────┴───────┐  │
                    │  │   Master DB (SATU-SATUNYA  │  │
                    │  │   sumber kebenaran status)  │  │
                    │  └────────────┬────────────────┘  │
                    └───────────────┼───────────────────┘
                         Tailscale  │  API (HTTPS, auth)
              ┌──────────────────────┼──────────────────────┐
              │                      │                      │
     ┌────────┴────────┐    ┌────────┴────────┐    ┌───────┴─────────┐
     │  live-01 (VPS 8)│    │  live-02 (VPS 8)│    │  live-N (VPS 8) │
     │  ┌────────────┐ │    │  ┌────────────┐ │    │  ┌────────────┐ │
     │  │ Encoders   │ │    │  │ Encoders   │ │    │  │ Encoders   │ │
     │  │ Prep worker│ │    │  │ Prep worker│ │    │  │ Prep worker│ │
     │  │ Node agent │ │    │  │ Node agent │ │    │  │ Node agent │ │
     │  └────────────┘ │    │  └────────────┘ │    │  └────────────┘ │
     │  DB LOKAL:      │    │  DB LOKAL:      │    │  DB LOKAL:      │
     │  HANYA cache    │    │  HANYA cache    │    │  HANYA cache    │
     │  baca (opsional)│    │  baca (opsional)│    │  baca (opsional)│
     └─────────────────┘    └─────────────────┘    └─────────────────┘
```

### 3.2 Pembagian peran (mengikat)

| Komponen | Berjalan di | TIDAK boleh berjalan di |
|----------|-------------|-------------------------|
| Web/API + Dashboard | Master | Node live |
| Master DB (sumber kebenaran) | Master | Node live (hanya cache baca) |
| Scheduler + Governor + Publish | Master | Node live |
| Encoder live (ffmpeg RTMP) | Node live | Master (kecuali darurat, dengan persetujuan owner) |
| Prep worker (ingestion + Stage 2) | Node live | Master (setelah migrasi 1.6 selesai) |
| Node agent (heartbeat + status push) | Setiap node live | — |
| Kuota ledger | Master | Node live |

### 3.3 Node agent

Setiap node live menjalankan **node agent** (satu proses Python, systemd service):
- **Registrasi** saat boot: kirim node_id, hostname, IP, kapasitas, versi kode.
- **Heartbeat** setiap 30 detik: load, CPU, RAM, disk, jumlah encoder, status.
- **Status push:** laporkan perubahan status task ke Master API dalam < 5 detik.
- **Lease renewal:** perpanjang lease untuk task yang dipegang setiap 30 detik.
- **Perintah:** terima perintah dari Master (start/stop/migrate) via API polling
  atau long-poll (bukan Master yang SSH ke node).

**Node agent adalah SATU-SATUNYA proses di node yang berkomunikasi dengan Master.**
Encoder ffmpeg TIDAK berkomunikasi langsung dengan Master.

---

## 4. SKEMA DATABASE

### 4.1 Tabel baru: `live_nodes`

```sql
CREATE TABLE live_nodes (
    node_id         TEXT PRIMARY KEY,       -- 'live-01', 'live-02', ...
    hostname        TEXT NOT NULL,          -- 'vmi3778263'
    tailnet_ip      TEXT NOT NULL,          -- '100.108.161.98'
    public_ip       TEXT,                   -- '31.220.76.63'
    role            TEXT NOT NULL DEFAULT 'live',
                        -- 'live' | 'prep' | 'live+prep'
    max_streams     INTEGER NOT NULL DEFAULT 70,   -- kapasitas @1080p
    max_prep_slots  INTEGER NOT NULL DEFAULT 4,    -- slot Stage 2
    status          TEXT NOT NULL DEFAULT 'online',
                        -- 'online' | 'offline' | 'draining' | 'maintenance'
    git_commit      TEXT,                   -- commit yang sedang berjalan
    last_heartbeat  INTEGER,                -- epoch detik
    cpu_percent     REAL,
    mem_percent     REAL,
    disk_free_gb    REAL,
    load1           REAL,
    created_at      INTEGER NOT NULL,
    updated_at      INTEGER NOT NULL
);
```

### 4.2 Kolom baru di `live_stream_tasks`

```sql
ALTER TABLE live_stream_tasks ADD COLUMN assigned_node_id TEXT;
    -- NULL = belum ditugaskan (legacy / Master). Setelah migrasi penuh,
    -- TIDAK BOLEH NULL untuk task aktif.

ALTER TABLE live_stream_tasks ADD COLUMN lease_node_id TEXT;
    -- Node pemegang lease saat ini (harus = assigned_node_id saat STREAMING).

ALTER TABLE live_stream_tasks ADD COLUMN lease_expires_at INTEGER;
    -- Epoch detik. Lease kedaluwarsa bila node tidak heartbeat 90 detik.

ALTER TABLE live_stream_tasks ADD COLUMN migration_state TEXT;
    -- NULL | 'migrating_out' | 'migrating_in'. Mencegah operasi ganda
    -- saat migrasi berlangsung.

ALTER TABLE live_stream_tasks ADD COLUMN node_status TEXT;
    -- Status terakhir yang dilaporkan node ('STREAMING' | 'STOPPED' |
    -- 'RECONNECTING' | 'ERROR'). Dipisah dari kolom `status` (yang
    -- dikontrol Master) agar perbedaan terlihat jelas.
```

**Kolom `error_message` TIDAK BOLEH lagi dipakai untuk menandai migrasi.**
String "migrated_to_live-01" dihapus dari kode; penugasan hanya via
`assigned_node_id`.

### 4.3 Tabel baru: `node_events`

```sql
CREATE TABLE node_events (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    event_type      TEXT NOT NULL,
        -- 'node_registered' | 'node_heartbeat_lost' | 'node_recovered' |
        -- 'task_assigned' | 'task_migrated' | 'task_started' |
        -- 'task_stopped' | 'encoder_died' | 'encoder_reconnected' |
        -- 'lease_expired' | 'guard_blocked_start'
    node_id         TEXT,
    task_id         TEXT,
    detail          TEXT,                   -- JSON: PID, exit code, dsb.
    created_at      INTEGER NOT NULL
);
CREATE INDEX idx_node_events_task ON node_events(task_id, created_at);
CREATE INDEX idx_node_events_node ON node_events(node_id, created_at);
```

Tabel ini adalah **jejak audit** untuk semua perubahan status. Insiden I9
(kematian massal tanpa penyebab) dijawab dengan: setiap kematian encoder
mencatat exit code + ekor log + kondisi node saat itu.

### 4.4 Kuota (tidak berubah, dipertegas)

- `vault_quota_ledger` tetap di Master. Node TIDAK punya ledger sendiri.
- Node yang perlu memanggil YouTube API HARUS cek kuota via Master API dulu.
- Dashboard kuota menampilkan agregasi armada (bukan per-node).

---

## 5. PENUGASAN & MIGRASI

### 5.1 Penugasan task ke node

**Aturan penugasan (mengikat):**
1. Setiap task aktif punya `assigned_node_id` (tidak NULL).
2. Penugasan diputuskan oleh **Master scheduler** (bukan oleh node, bukan
   manual via DB).
3. Algoritma default: **least-loaded** — pilih node online dengan
   `(streams_aktif / max_streams)` terendah. Bila seri, pilih node dengan
   load1 terendah.
4. Owner/operator dapat **pin** task ke node tertentu (override manual,
   dicatat di `node_events`).
5. Node dengan status `draining` atau `maintenance` TIDAK menerima task baru.
6. Node dengan status `offline` (> 90 detik tanpa heartbeat) — semua task-nya
   dijadwalkan ulang ke node sehat (lihat §5.3).

### 5.2 Alur migrasi (atomic, 6 langkah)

Migrasi task dari node A ke node B HARUS mengikuti urutan ini. Tidak ada
jalan pintas.

```
1. Master set migration_state='migrating_out', assigned_node_id=B
   (task TIDAK BOLEH di-start/stop manual selama migrasi)
2. Master perintahkan node A: stop task
3. Node A konfirmasi: encoder berhenti, PID mati (verifikasi /proc)
4. Master perintahkan node B: start task (dengan lease baru)
5. Node B konfirmasi: encoder jalan, RTMP tersambung, status STREAMING
6. Master set migration_state=NULL, lease_node_id=B, catat event
```

**Bila langkah 3 atau 5 GAGAL:** migrasi dibatalkan, task kembali ke node A
(rollback otomatis), event dicatat, operator diberi tahu. TIDAK BOLEH ada
keadaan di mana task berjalan di 2 node sekaligus.

**Durasi target:** migrasi 1 task < 2 menit (termasuk verifikasi).

### 5.3 Failover node mati

Bila node tidak heartbeat selama 90 detik:
1. Master tandai node `offline`.
2. Semua lease task di node itu kedaluwarsa (lease_expires_at tercapai).
3. Master tugaskan ulang task ke node sehat (algoritma §5.1).
4. Task di-start di node baru (alur migrasi §5.2, langkah 4–6).
5. Bila TIDAK ADA node sehat dengan kapasitas: task masuk antrean
   `PENDING_NODE` (terlihat di dashboard, bukan diam-diam hilang).

**Bila node pulih (heartbeat kembali):** node TIDAK BOLEH otomatis me-restart
task lama. Task yang sudah dipindah TETAP di node baru. Node yang pulih hanya
menerima task baru via scheduler.

### 5.4 Guard multi-lapisan (mencegah I2)

Guard "task hanya boleh jalan di node penugasannya" ditegakkan di **5 lapisan**:

| Lapisan | Lokasi | Mekanisme |
|---------|--------|-----------|
| API | `POST /api/live/start` | Tolak bila `assigned_node_id != node peminta` |
| Supervisor | `start_stream()` | Tolak bila `assigned_node_id != node ini` (sudah ada di f7ee5ce, perlu diperluas ke semua node) |
| Auto-heal | `_auto_heal_live_task_runner` | Lewati task yang `assigned_node_id != node ini` |
| Scheduler | Penjadwalan harian | Jangan jadwalkan task ke node yang salah |
| Sentinel | Kill/reconnect | Jangan kill/reconnect task yang bukan milik node ini |

**Setiap penolakan guard dicatat** di `node_events` (event_type =
`guard_blocked_start`) agar terlihat di dashboard dan log.

**Tes wajib:** Setiap lapisan guard punya unit test. Test dijalankan di CI
sebelum deploy. Tidak ada deploy tanpa test guard hijau.

---

## 6. DESAIN FRONTEND

### 6.1 Dashboard `/live` — kartu task (mencegah I1)

Setiap kartu task menampilkan:

```
┌─────────────────────────────────────────────┐
│ [Nama Channel]                    [OPERATOR]│
│ ┌─────────────────────────────────────────┐ │
│ │ ● LIVE di live-01  •  uptime 3j 12m    │ │  ← badge status + node
│ │   Encoder PID 81858 • 1080p • 39% CPU  │ │  ← detail node (bila live)
│ └─────────────────────────────────────────┘ │
│ Video: ...  Audio: ...  Stream Key: 26t0…  │
│ Jadwal: ...  Auto Stop: Non-Stop (24/7)    │
│                                             │
│ [Live ↗] [Player] [Studio] [SEO] [QA] [Log]│
│ [■ Hentikan]  [↗ Migrasikan]  [🗑]          │  ← tombol sesuai status
└─────────────────────────────────────────────┘
```

**Aturan tampilan (mengikat):**
1. Badge status SELALU menunjukkan node: "● LIVE di live-01", bukan sekadar
   "LIVE". Bila task STOPPED: "○ Berhenti (terakhir di live-01)".
2. **Tombol "Mulai Siar" HANYA muncul bila task benar-benar STOPPED di semua
   node.** Bila task STREAMING di node mana pun, tombol diganti "■ Hentikan".
3. Tombol "Paksa" HANYA untuk owner, dan tetap melewati guard (§5.4).
4. Selama migrasi (`migration_state` tidak NULL): kartu menampilkan
   "↗ Memigrasi live-01 → live-02..." dan SEMUA tombol aksi dinonaktifkan.
5. Bila node penugasan `offline`: kartu menampilkan peringatan kuning
   "⚠ Node live-01 offline — failover dalam proses" (bukan merah "Siaran
   dihentikan").

### 6.2 Halaman manajemen node (BARU — `/nodes`)

Halaman baru untuk owner (bukan operator):

```
┌──────────────────────────────────────────────────────────────┐
│ Node Armada                                    [+ Tambah Node]│
├──────────┬────────┬─────────┬───────┬──────┬──────┬─────────┤
│ Node     │ Status │ Streams │ Prep  │ Load │ Disk │ Versi   │
├──────────┼────────┼─────────┼───────┼──────┼──────┼─────────┤
│ live-01  │ ● Onl  │ 3 / 70  │ 0 / 4 │ 2.83 │ 277G │ cea60b8 │
│ live-02  │ ● Onl  │ 0 / 70  │ 0 / 4 │ 0.42 │ 290G │ cea60b8 │
│ live-03  │ ○ Off  │ —       │ —     │ —    │ —    │ —       │
└──────────┴────────┴─────────┴───────┴──────┴──────┴─────────┘
Klik node → detail: daftar task, riwayat event, tombol [Drain] [Maintenance]
```

**Fitur wajib:**
- **Tambah node:** wizard (install agent → registrasi → verifikasi → aktif).
  Target: node baru siap menerima task dalam < 30 menit.
- **Drain node:** migrasikan semua task ke node lain, lalu tandai `draining`.
  Dipakai sebelum maintenance / decommission.
- **Riwayat event per node:** dari `node_events` (§4.3).

### 6.3 Ikhtisar armada (BARU — widget di dashboard utama)

```
┌─────────────────────────────────────────────────┐
│ Armada Live: 3 node online • 3/210 stream aktif │
│ [live-01 ████░░░░░░ 3/70] [live-02 ░░░░░░░░░ 0/70] │
│ Kuota hari ini: 1.240 / 10.000 unit (12%)       │
│ Event terakhir: 19:55 encoder 706 mati (exit 224)│
└─────────────────────────────────────────────────┘
```

### 6.4 Cakupan operator (tidak berubah, dipertegas)

- Operator melihat HANYA task mereka sendiri (aturan yang sudah ada).
- Operator melihat badge node (informasi), tapi TIDAK bisa menambah/menghapus
  node atau memigrasi task antar-node (itu wewenang owner).
- Operator BISA start/stop task mereka — perintah diteruskan ke node penugasan
  via Master API (operator tidak perlu tahu node mana).

---

## 7. DISTRIBUSI PREP WORKER (mencegah I7)

### 7.1 Prinsip

- Prep job (ingestion download, Stage 2 render) adalah **antrean di Master DB**.
- Setiap node menjalankan **prep worker** yang mengambil job dari antrean Master.
- Tidak ada prep worker di Master setelah migrasi penuh (Master hanya publish).

### 7.2 Antrean prep

```sql
-- Tabel yang sudah ada (operational_tasks / background_jobs) ditambah:
ALTER TABLE background_jobs ADD COLUMN assigned_node_id TEXT;
ALTER TABLE background_jobs ADD COLUMN claimed_at INTEGER;
    -- Job yang di-claim tapi tidak selesai dalam timeout (misal 30 menit)
    -- dikembalikan ke antrean (claim kedaluwarsa).
```

**Alur:**
1. Master membuat job prep (status QUEUED, `assigned_node_id` NULL).
2. Prep worker node memanggil `POST /api/prep/claim` → Master beri 1 job
   (pilih node dengan slot kosong + load terendah).
3. Worker kerjakan job, laporkan progress via `POST /api/prep/progress`.
4. Worker laporkan selesai/gagal via `POST /api/prep/complete`.
5. Bila worker mati di tengah job: claim kedaluwarsa → job kembali QUEUED →
   node lain mengambilnya.

### 7.3 Kapasitas prep per node

- Stage 2: max 4 slot per node (cgroup 4 core) — sesuai rencana yang ada.
- Ingestion: max 2 download bersamaan per node (hindari jenuh bandwidth).
- Prep TIDAK BOLEH mengganggu live: bila load node > 7,0, prep worker jeda
  (tunggu load turun). Live selalu prioritas.

### 7.4 Storage prep

- Workspace bahan (`/opt/masjavas/workspace`) disinkronkan ke setiap node
  via DR sync (mekanisme yang sudah ada, diperluas ke N node).
- Output prep (video/audio/thumbnail) diunggah ke Drive dari node yang
  mengerjakannya (bukan dikirim balik ke Master dulu).

---

## 8. KEAMANAN & KETAHANAN

1. **Autentikasi node:** Setiap node punya token API unik (disimpan di
   `/opt/masjavas-os/.env` node, TIDAK di repo). Token dicabut saat node
   dihapus.
2. **Tidak ada SSH antar-node untuk operasi rutin.** Semua komunikasi via API
   Tailscale (HTTPS). SSH hanya untuk debugging manual.
3. **Tidak ada kredensial di log.** Stream key, token API, dan URL bertanda
   tangan TIDAK PERNAH dicetak di log, laporan, atau dashboard.
4. **Restart terkoordinasi:** Aturan max 1 restart/15 menit lintas agent
   tetap berlaku, diperluas ke semua node (restart node A dan node B tidak
   boleh bersamaan bila keduanya memegang task aktif).
5. **Deploy bertahap:** Deploy kode baru ke 1 node dulu (canary), observasi
   1 jam, baru deploy ke node lain. Jangan deploy ke semua node sekaligus.
6. **Rollback:** Setiap deploy mencatat commit sebelumnya. Rollback = checkout
   commit lama + restart service (prosedur terdokumentasi, bukan improvisasi).

---

## 9. OBSERVABILITAS (mencegah I6, I9)

### 9.1 Monitoring terpusat

- Monitor (cron yang sudah ada) diperluas: cek Master + SEMUA node yang
  terdaftar di `live_nodes` (bukan hanya Master).
- Setiap node melaporkan metrik ke Master setiap 30 detik (via heartbeat).
- Dashboard `/nodes` menampilkan metrik real-time per node.

### 9.2 Alert

| Kondisi | Tingkat | Aksi |
|---------|---------|------|
| Node offline > 90 detik | KRITIS | Failover otomatis + notifikasi owner |
| Task STREAMING → STOPPED tak terduga | TINGGI | Auto-reconnect + catat event |
| Encoder mati 3× dalam 1 jam (node sama) | TINGGI | Notifikasi owner + tandai node untuk investigasi |
| Load node > 8,0 selama 10 menit | SEDANG | Jeda prep worker + notifikasi |
| Disk node < 20 GB free | SEDANG | Notifikasi owner |
| Kuota vault > 80% | SEDANG | Notifikasi owner |
| Guard memblokir start | INFO | Catat event (bukan alert — ini perilaku benar) |

### 9.3 Investigasi kematian encoder (menjawab I9)

Setiap kematian encoder HARUS mencatat (otomatis, di `node_events`):
- Exit code ffmpeg
- 20 baris terakhir log ffmpeg ("ekor log")
- Load/CPU/RAM/disk node saat kematian
- Durasi hidup encoder (uptime)
- Jumlah kematian node itu dalam 1 jam terakhir

Tanpa data ini, penyebab kematian massal TIDAK BISA diidentifikasi. Logging
ini adalah PRASYARAT sebelum menambah node baru.

---

## 10. RENCANA MIGRASI (dari keadaan sekarang ke arsitektur ini)

| Fase | Pekerjaan | Estimasi | Prasyarat |
|------|-----------|----------|-----------|
| **F0** | **Persetujuan dokumen ini** (owner) | — | Dokumen dibaca + disetujui |
| **F1** | Skema DB: `live_nodes`, kolom `assigned_node_id`/`lease_*`/`migration_state`/`node_status`, `node_events` | 1 hari | F0 |
| **F2** | Node agent (registrasi + heartbeat + status push) + API Master | 2–3 hari | F1 |
| **F3** | Backfill live-01: daftarkan sebagai node, isi `assigned_node_id` untuk 3 task, hapus hack `error_message` | 0,5 hari | F2 |
| **F4** | Guard 5 lapisan (§5.4) + unit test | 1–2 hari | F1 |
| **F5** | Dashboard `/live` baru (§6.1) + halaman `/nodes` (§6.2) + widget armada (§6.3) | 2–3 hari | F2, F4 |
| **F6** | Antrean prep terdistribusi (§7) + prep worker di live-01 | 2–3 hari | F2 |
| **F7** | Monitoring + alert terpusat (§9) | 1 hari | F2 |
| **F8** | **Proof: tambah live-02** dengan arsitektur baru (wizard §6.2), migrasikan 1 task sebagai canary | 1 hari | F1–F7 |
| **F9** | Matikan prep di Master; Master murni control plane + publish | 0,5 hari | F6, F8 |

**Total estimasi:** ~12–16 hari kerja (satu lane). Fase dapat diparalelkan
bila ada 2 lane (backend F1–F4 + frontend F5).

**Aturan migrasi:**
- Setiap fase punya kriteria selesai yang terverifikasi (bukan klaim).
- Tidak ada fase yang dimulai sebelum fase prasyarat terverifikasi.
- F8 (live-02) adalah BUKTI arsitektur bekerja — jangan tambah live-03
  sebelum live-02 stabil 72 jam.

---

## 11. ATURAN YANG MENGIKAT (ringkasan — ditempel di setiap prompt implementasi)

1. Master DB adalah satu-satunya sumber kebenaran status. Node tidak punya DB status.
2. Setiap task aktif punya `assigned_node_id`. Tidak ada penugasan via `error_message`.
3. Hanya pemegang lease valid yang boleh menjalankan task.
4. Node tidak pernah menulis Master DB langsung — hanya via API.
5. Bila ragu, jangan start. Encoder duplikat lebih buruk daripada stream berhenti sebentar.
6. Setiap perubahan status tercatat di `node_events` dan terlihat di dashboard.
7. Guard ditegakkan di 5 lapisan + unit test. Tidak ada deploy tanpa test guard hijau.
8. Dashboard menampilkan node di setiap kartu task. Tombol start hanya bila task benar-benar berhenti di semua node.
9. Deploy bertahap: 1 node canary dulu, observasi 1 jam, baru node lain.
10. Tidak ada node baru (live-03+) sebelum live-02 stabil 72 jam di arsitektur baru.

---

## 12. PERTANYAAN TERBUKA (untuk owner)

1. **Jumlah node docel:** Berapa node live yang direncanakan dalam 3 bulan ke
   depan? (Mempengaruhi kapasitas Master DB + API.)
2. **Prep di node live atau node prep khusus?** Opsi A: setiap node live juga
   menjalankan prep (lebih sederhana). Opsi B: node prep khusus terpisah dari
   node live (lebih terisolasi, biaya lebih tinggi).
3. **Failover otomatis atau manual?** Dokumen ini mengusulkan failover otomatis
   (§5.3). Bila owner lebih suka failover manual (notifikasi dulu, owner yang
   memutuskan), sebutkan.
4. **Halaman `/nodes` untuk operator?** Dokumen ini membatasi ke owner. Bila
   operator senior (misal MONA) perlu melihat status node, sebutkan.

---

## 13. RIWAYAT REVISI

| Versi | Tanggal | Perubahan |
|-------|---------|-----------|
| 1.0 | 23 Nov 2026 20:02 | Draf awal — berdasarkan insiden 22–23 Nov 2026 |
