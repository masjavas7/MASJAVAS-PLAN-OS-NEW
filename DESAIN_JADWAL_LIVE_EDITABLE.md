# DESAIN: Jadwal Live yang Bisa Diedit, Disimpan & Dipakai Ulang

| | |
|---|---|
| **Tanggal** | 25 Nov 2026 |
| **Status** | DRAFT — menunggu persetujuan user sebelum implementasi |
| **Pemicu** | Insiden slot 034 (SALSA): sumber audio relay `kizfrmykOkM` dihapus YouTube; operator tidak bisa mengganti URL tanpa hapus + buat ulang jadwal dari nol |
| **Ruang lingkup** | Fitur livestreaming MASJAVAS TUBE OS (`api/routes/live_stream.py`, `worker/live_supervisor.py`, `worker/live_stream_engine.py`, `web/templates/live_*.html`) |
| **Prinsip pengaman** | Baseline lantai (206 PUBLISHED/hari, 0 FAILED) tidak boleh regresi; tiap fase pakai verifikasi pre/post + rollback; live disentuh terakhir, satu stream dulu |

---

## 1. Masalah (terverifikasi dari kode, 25 Nov 2026)

1. **Tidak ada endpoint edit/update untuk live task.** `api/routes/live_stream.py` hanya punya: create (`POST /api/live/schedule`), start, stop, pause, resume, priority, delete, qa-probe, broadcast/create-auto. Pencarian `update_live_task` / `edit_live` / PUT/PATCH di `api/` + `worker/` = kosong.
2. **Editor dashboard hanya metadata.** `modalLiveMetadataEditor` di `live_dashboard.html` hanya ubah judul, deskripsi, tag, thumbnail (In-Flight SEO) via `POST /api/live/metadata/{task_id}`. Sumber video/audio, URL relay, dan jam tayang tidak bisa diubah.
3. **Satu-satunya jalan mengubah sumber = hapus + buat ulang.** Konsekuensi: kehilangan binding broadcast (`broadcast_id`), riwayat task, `reconnect_count`, dan konfigurasi harus diisi ulang dari nol — lambat (±30 menit) dan rawan salah, terutama saat insiden tengah malam.
4. **Tidak ada validasi sumber saat simpan.** URL relay yang sudah dihapus (kasus 034) baru ketahuan setelah stream gagal berulang kali, dari log — bukan saat operator menyimpan.
5. **Tidak ada konsep simpan/pakai ulang.** Setiap jadwal dibuat dari nol; tidak ada template, tidak ada salin (clone), tidak ada riwayat revisi.

**Kabar baik:** tabel `live_stream_tasks` sudah punya semua kolom yang dibutuhkan (`video_path`, `video_filename`, `video_source_type`, `video_url`, `audio_source_type`, `audio_path`, `audio_filename`, `audio_url`, `youtube_live_url`, `start_time_epoch`, `stop_time_epoch`, `auto_stop_minutes`, `repeat_mode`, `resolution`, `bitrate_kbps`, `priority`, `thumbnail_path`, `mute_video_audio`). Tidak ada perubahan skema pada tabel utama kecuali 1 kolom versi (lihat §5).

---

## 2. Tujuan & batasan

### Tujuan
- **T1.** Operator bisa mengedit jadwal live yang sudah dibuat (semua field konfigurasi) tanpa hapus + buat ulang.
- **T2.** Ganti URL relay audio pada stream yang sedang tayang **tanpa mematikan encoder** (hot-swap, downtime 0 detik).
- **T3.** Sumber yang sudah mati (video dihapus, URL tidak valid) **tidak bisa tersimpan** — ditolak saat validasi dengan pesan yang jelas.
- **T4.** Saat stream bermasalah, dashboard menunjukkan penyebab + tombol perbaikan satu klik (quick-fix).
- **T5.** Jadwal bisa disimpan sebagai template, dipakai ulang, direvisi, dan di-rollback ke versi sebelumnya.
- **T6.** Semua perubahan teraudit (siapa, kapan, apa yang berubah, dari nilai apa ke nilai apa).

### Bukan tujuan (non-goals)
- Mengubah cara kerja encoder/RTMP (di luar hot-swap relay).
- Mengubah sistem broadcast YouTube (broadcast tetap terikat ke task; edit tidak membuat broadcast baru).
- Mengubah RBAC/operator scoping (memakai yang sudah ada).
- Migrasi/failover antar-node (fitur terpisah, sudah ada).

---

## 3. Cerita pengguna

### 3.1 Insiden 034 — sebelum vs sesudah

| | Sebelum (hari ini) | Sesudah (desain ini) |
|---|---|---|
| Deteksi | Operator tahu dari log/laporan, berjam-jam kemudian | Baris task 034 memerah + banner: "Sumber relay dihapus — video kizfrmykOkM tidak tersedia" |
| Perbaikan | Hapus task → buat baru → isi ulang semua field → start → verifikasi (±30 mnt, kehilangan broadcast + riwayat) | Klik "Ganti URL relay" → tempel URL baru → simpan → tervalidasi → hot-swap (±1 mnt, siaran jalan terus, broadcast + riwayat utuh) |

### 3.2 Cerita lain
- **US-2 (ganti video):** Operator ganti video loop channel reggae ke video baru. Sistem memberi tahu "butuh restart encoder ±1 menit" + minta konfirmasi → restart terjadwal otomatis → stream kembali dengan video baru.
- **US-3 (template):** Operator menyimpan konfigurasi "Reggae 24/7" sebagai template. Bulan depan, buat 5 jadwal reggae baru dari template dalam 5 klik.
- **US-4 (salah edit):** Operator salah tempel URL relay jam 3 pagi. Buka riwayat → "Kembalikan ke versi 12:04" → konfigurasi pulih.
- **US-5 (darurat massal):** YouTube relay bermasalah global. Operator pilih 6 channel → "Alihkan semua ke audio darurat" → 1 aksi.
- **US-6 (perubahan terjadwal):** Operator menjadwalkan "ganti video besok 06:00" tanpa menyentuh siaran yang sedang jalan.

---

## 4. Gambaran solusi

Tiga lapisan baru, semuanya di atas mekanisme yang sudah ada:

```
Dashboard (live_dashboard.html + live_create.html)
  │  modal edit · panel quick-fix · pemilih template · riwayat revisi
  ▼
API web Master (api/routes/live_stream.py) — endpoint baru, pola POST seperti metadata
  │  validasi · scoping operator · optimistic locking · tulis revisi
  ▼
┌──────────────────────────────┬─────────────────────────────────────────────┐
│ Task BELUM jalan / di Master │ Task SEDANG tayang di node (VPS 8)          │
│ Edit langsung ke baris DB    │ Edit ke DB + antrean order lintas-node      │
│ Master; berlaku saat start   │ (live_task_orders) → push script → supervisor│
│ berikutnya.                  │ → hot-swap relay ATAU restart encoder.      │
└──────────────────────────────┴─────────────────────────────────────────────┘
```

**Keputusan arsitektur kunci (D1–D8):**

| ID | Keputusan | Alasan |
|---|---|---|
| D1 | Endpoint update memakai **POST** (bukan PATCH) | Mengikuti konvensi kode: `POST /api/live/metadata/{task_id}` |
| D2 | Hot-swap **hanya untuk URL relay audio** | Satu-satunya pergantian yang aman tanpa restart: manager memegang FIFO `O_RDWR` sehingga encoder tidak pernah melihat EOF (terbukti di `live_stream_engine.py::_run_loop_inner`). Video/resolusi/jadwal = restart encoder |
| D3 | Perintah lintas-node via **tabel antrean `live_task_orders`** + aksi order baru | Eksplisit, bisa diaudit, bisa retry; tidak menumpang flag `migration_state` (yang sudah terbukti rapuh — bug Fase C #1) |
| D4 | **Optimistic locking** via kolom `config_version` | Mencegah dua operator saling menimpa edit (lost update); konflik → 409 + tampilkan versi terbaru |
| D5 | Validasi URL relay memakai **`probe_media_availability`** yang sudah ada | Sudah ada (multi-region, read-only, dipakai qa-probe); ditambah probe manifest yt-dlp untuk memastikan video benar-benar live/bisa diakses |
| D6 | Template = **snapshot** (bukan referensi hidup) | Revisi template tidak mengubah task yang sudah dibuat; operator memilih sendiri kapan menerapkan (via bulk apply) |
| D7 | Revisi = **snapshot JSON lengkap** per edit | Murah (edit jarang, task ≤707), rollback trivial, audit lengkap |
| D8 | Rollback = **edit biasa dengan nilai lama** | Masuk revisi juga — jejak audit tidak pernah putus |

---

## 5. Model data

### 5.1 Perubahan pada `live_stream_tasks` (1 kolom)

```sql
ALTER TABLE live_stream_tasks ADD COLUMN config_version INTEGER NOT NULL DEFAULT 1;
-- Naik +1 di setiap edit yang berhasil. Dipakai optimistic locking (D4).
-- Tidak ada perubahan lain: semua kolom konfigurasi sudah ada.
```

### 5.2 Tabel baru: `live_schedule_templates`

```sql
CREATE TABLE live_schedule_templates (
    id              TEXT PRIMARY KEY,          -- uuid
    name            TEXT NOT NULL UNIQUE,      -- "Reggae 24/7", "Dangdut Malam"
    description     TEXT NOT NULL DEFAULT '',
    operator_name   TEXT NOT NULL,             -- pemilik template (OWNER = semua operator)
    channel_id      TEXT,                      -- NULL = template umum (pilih channel saat pakai)
    config_json     TEXT NOT NULL,             -- snapshot CreateLiveRequest (tanpa key/node)
    source_task_id  TEXT,                      -- task asal template dibuat (boleh NULL)
    use_count       INTEGER NOT NULL DEFAULT 0,
    created_at      INTEGER NOT NULL,
    updated_at      INTEGER NOT NULL
);
CREATE INDEX idx_templates_operator ON live_schedule_templates(operator_name);
```

Catatan: `stream_key` dan `node_id` **tidak** disimpan di template (key per channel, node dipilih saat buat task).

### 5.3 Tabel baru: `live_task_revisions`

```sql
CREATE TABLE live_task_revisions (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    task_id         TEXT NOT NULL,
    revision_no     INTEGER NOT NULL,          -- 1, 2, 3... per task
    config_json     TEXT NOT NULL,             -- snapshot konfigurasi SETELAH edit
    changed_fields  TEXT NOT NULL,             -- JSON: {"audio_url": {"old": "...", "new": "..."}}
    changed_by      TEXT NOT NULL,             -- operator / OWNER
    change_reason   TEXT NOT NULL DEFAULT '',  -- alasan opsional dari operator
    created_at      INTEGER NOT NULL,
    UNIQUE(task_id, revision_no)
);
CREATE INDEX idx_revisions_task ON live_task_revisions(task_id, revision_no DESC);
```

Retensi: simpan 50 revisi terakhir per task (janitor hapus yang lebih lama; revisi pertama — konfigurasi awal — selalu disimpan).

### 5.4 Tabel baru: `live_task_orders` (antrean perintah lintas-node)

```sql
CREATE TABLE live_task_orders (
    id              TEXT PRIMARY KEY,          -- uuid
    task_id         TEXT NOT NULL,
    target_node_id  TEXT NOT NULL,             -- node pemilik task (assigned_node_id)
    action          TEXT NOT NULL,             -- 'relay_swap' | 'config_restart' | 'emergency_audio' | ...
    payload_json    TEXT NOT NULL,             -- {"audio_url": "..."} / {"reason": "..."}
    status          TEXT NOT NULL DEFAULT 'PENDING',  -- PENDING | DELIVERED | DONE | FAILED | EXPIRED
    attempts        INTEGER NOT NULL DEFAULT 0,
    result_json     TEXT,                      -- hasil dari node (DONE/FAILED)
    created_by      TEXT NOT NULL,
    created_at      INTEGER NOT NULL,
    delivered_at    INTEGER,
    completed_at    INTEGER,
    expires_at      INTEGER NOT NULL           -- created_at + 10 menit; PENDING yang kedaluwarsa → EXPIRED
);
CREATE INDEX idx_orders_node_status ON live_task_orders(target_node_id, status);
```

---

## 6. API

Semua endpoint baru di `api/routes/live_stream.py`, memakai `verify_live_task_access` / `verify_channel_operator_access` yang sudah ada. Semua mengembalikan `config_version` terbaru.

### 6.1 Ringkasan endpoint

| Method & path | Fungsi | Prioritas |
|---|---|---|
| `POST /api/live/tasks/{task_id}/edit` | Edit konfigurasi task (D1) | P0 |
| `POST /api/live/tasks/{task_id}/relay-swap` | Hot-swap URL relay (STREAMING) | P0 |
| `POST /api/live/tasks/{task_id}/emergency-audio` | Alihkan ke audio darurat (STREAMING, tanpa restart) | P0 |
| `POST /api/live/validate-source` | Validasi URL relay / file video sebelum simpan | P0 |
| `GET /api/live/tasks/{task_id}/revisions` | Daftar revisi task | P1 |
| `POST /api/live/tasks/{task_id}/rollback/{revision_no}` | Kembalikan ke revisi tertentu | P1 |
| `POST /api/live/templates` | Buat template (dari task / dari nol) | P1 |
| `GET /api/live/templates` | Daftar template (scope operator) | P1 |
| `POST /api/live/templates/{template_id}/apply` | Buat task dari template | P1 |
| `PUT /api/live/templates/{template_id}` | Revisi template | P1 |
| `DELETE /api/live/templates/{template_id}` | Hapus template | P1 |
| `POST /api/live/bulk-edit` | Edit massal banyak task sekaligus | P2 |
| `POST /api/live/tasks/{task_id}/schedule-change` | Jadwalkan perubahan di waktu tertentu | P2 |
| `GET /api/live/tasks/{task_id}/orders` | Status order lintas-node untuk task | P0 (pendukung) |

### 6.2 `POST /api/live/tasks/{task_id}/edit` (inti)

**Request** (semua field opsional kecuali `config_version`; hanya field yang dikirim yang diubah):

```json
{
  "config_version": 7,
  "title": "REGGAE SANTAI 24/7",
  "video": "gdrive_video_9XyZ.mp4",
  "video_source_type": "DEPOT",
  "video_url": "",
  "audio_source_type": "YOUTUBE_URL",
  "audio_filename": "",
  "audio_url": "https://www.youtube.com/watch?v=NEW_ID",
  "mute_video_audio": true,
  "start_time": "2026-11-26T06:00:00+07:00",
  "stop_time": null,
  "auto_stop_hours": 0,
  "repeat": "daily",
  "resolution": "1080p",
  "priority": 2,
  "change_reason": "Ganti sumber relay, video lama dihapus"
}
```

**Tidak bisa diubah via edit:** `channel_id`, `stream_key`, `rtmp_url`, `broadcast_id`, `assigned_node_id` (pindah node = failover, fitur terpisah), `operator_name` (ubah operator = admin/OWNER via jalur lain).

**Respons sukses:**

```json
{
  "ok": true,
  "task_id": "1795502710790",
  "config_version": 8,
  "applied": "immediate",
  "applied_detail": "Tersimpan. Berlaku saat stream di-start berikutnya.",
  "revision_no": 12,
  "requires_restart": false,
  "order_id": null
}
```

`applied` dapat bernilai:
- `immediate` — task belum jalan (IDLE/SCHEDULED/STOPPED/ERROR): tersimpan, berlaku saat start berikutnya.
- `hot_swap` — hanya URL relay berubah pada task STREAMING: order `relay_swap` dikirim ke node.
- `restart_pending` — video/resolusi/jadwal berubah pada task STREAMING: butuh restart; respons meminta konfirmasi (lihat §6.3).
- `queued` — node tujuan tidak terjangkau saat ini; order masuk antrean, berlaku saat node kembali (maks 10 menit, lalu EXPIRED + notifikasi).

**Error:**

| Kode | Arti |
|---|---|
| 403 | Bukan operator pemilik task/channel (bukan OWNER) |
| 404 | Task tidak ditemukan |
| 409 `VERSION_CONFLICT` | `config_version` tidak cocok — task sudah diubah operator lain; respons menyertakan konfigurasi terbaru |
| 409 `TASK_MIGRATING` | Task sedang migrasi (`migration_state` tidak NULL) — edit ditolak sampai migrasi selesai |
| 422 `SOURCE_INVALID` | Validasi sumber gagal (video dihapus / URL bukan live / file tidak ada); pesan menyebut sumber persisnya |
| 422 `NO_CHANGE` | Tidak ada field yang berubah |

**Alur server (edit):**
1. Ambil task + cek akses operator (403 bila bukan pemilik).
2. Tolak bila `migration_state` tidak NULL (409).
3. Cek `config_version` (409 bila tidak cocok).
4. Validasi tiap sumber yang berubah (§7.3) — gagal → 422, tidak ada yang tersimpan (atomic).
5. Simpan snapshot revisi (D7) + naikkan `config_version` + UPDATE baris (satu transaksi).
6. Tentukan `applied` dari status task + field yang berubah (§7.1); bila perlu, buat order di `live_task_orders` (D3).
7. Kembalikan respons + `config_version` baru.

### 6.3 Konfirmasi restart

Bila edit pada task STREAMING mengubah field yang butuh restart (video, resolusi, `mute_video_audio`, jadwal stop), endpoint mengembalikan **200 dengan `requires_restart: true` + `applied: "restart_pending"`** — perubahan **sudah tersimpan di DB** tapi **belum diterapkan ke encoder**. Dashboard menampilkan: "Tersimpan. Ganti video butuh restart encoder ±1 menit. [Restart sekarang] [Nanti saja]". Tombol "Restart sekarang" memanggil `POST /api/live/tasks/{task_id}/restart` (endpoint baru, tipis: stop + start via jalur yang sudah ada) atau operator membiarkan restart terjadi pada reconnect berikutnya. Desain ini memisahkan "simpan" (aman, reversible) dari "terapkan" (berdampak siaran) — operator tidak pernah dipaksa restart di tengah insiden.

### 6.4 `POST /api/live/tasks/{task_id}/relay-swap`

Jalur cepat khusus ganti URL relay (dipakai panel quick-fix). Validasi URL dulu (§7.3), lalu UPDATE `audio_url` + revisi + order `relay_swap` ke node. Respons: `{ok, order_id, applied: "hot_swap"}`. Bila task tidak STREAMING, berperilaku seperti edit biasa (`immediate`).

### 6.5 `POST /api/live/validate-source`

```json
// Request
{"kind": "relay_url", "url": "https://www.youtube.com/watch?v=XXX"}
// atau {"kind": "video_file", "filename": "gdrive_video_X.mp4", "node_id": "live-01"}
// Respons
{"ok": true, "kind": "relay_url", "alive": true, "is_live": true,
 "title": "Reggae Live 24/7", "checked_at": 1795564000, "regions_ok": ["id", "sg", "us"]}
// Gagal:
{"ok": false, "alive": false, "reason": "VIDEO_REMOVED",
 "message": "Video ini sudah dihapus dari YouTube (alasan: lisensi/hak cipta)."}
```

Dipakai live (saat operator mengetik/tempel URL, debounce 800ms) dan wajib sebelum simpan.

### 6.6 Template & revisi

- `POST /api/live/templates` — body: `{name, description, channel_id?, config_json | source_task_id}`. Bila `source_task_id` diisi, config disalin dari task (tanpa key/node).
- `POST /api/live/templates/{id}/apply` — body: `{channel_id, key?, start_now?, node_id?}` → membuat task baru via jalur create yang sudah ada (`schedule_live_stream`), lalu `use_count += 1`.
- `POST /api/live/tasks/{id}/rollback/{revision_no}` — memuat snapshot revisi → menjalankan alur edit yang sama (validasi ulang sumber! snapshot lama bisa mengandung URL yang kini mati — bila validasi gagal, rollback ditolak dengan pesan jelas + tawarkan rollback parsial field non-sumber).

### 6.7 Bulk edit (P2)

`POST /api/live/bulk-edit` — body: `{task_ids: [...], patch: {audio_source_type: "DEPOT", ...}, change_reason: "..."}`. Batasan: hanya field yang sama untuk semua task (tidak ada `audio_url` per-task yang berbeda — untuk itu pakai edit per task). Server memproses per task dengan alur edit yang sama; respons berisi hasil per task (`ok`/`failed` + alasan). Maks 20 task per panggilan.

### 6.8 Scheduled change (P2)

`POST /api/live/tasks/{id}/schedule-change` — body: `{apply_at_epoch, patch: {...}, change_reason}`. Disimpan di tabel `live_scheduled_changes` (task_id, apply_at, patch_json, status). Supervisor (atau worker terjadwal di Master) menerapkan saat waktunya tiba via alur edit yang sama. Perubahan terjadwal bisa dibatalkan (`DELETE .../schedule-change/{id}`) sebelum waktunya.

---

## 7. Logika backend

### 7.1 Semantik edit per status task

| Status task | Video/resolusi/jadwal | URL relay audio | Judul/prioritas |
|---|---|---|---|
| IDLE / SCHEDULED / STOPPED / ERROR | Tersimpan, berlaku saat start berikut | Tersimpan, berlaku saat start berikut | Tersimpan langsung |
| STREAMING | Tersimpan + `restart_pending` (butuh konfirmasi restart) | **Hot-swap** via order (§7.2) | Tersimpan langsung (judul dipakai dashboard/metadata; tidak ganggu encoder) |
| RECONNECTING | Ditolak sementara (409 `TASK_RECONNECTING`) — tunggu STREAMING atau STOPPED | Ditolak sementara (alasan sama) | Tersimpan langsung |
| PAUSED | Seperti STOPPED | Seperti STOPPED | Tersimpan langsung |
| Migrasi berjalan | Ditolak (409 `TASK_MIGRATING`) | Ditolak | Ditolak |

### 7.2 Hot-swap URL relay (inti teknis)

**Fakta dasar (terverifikasi):** encoder membaca audio dari FIFO (`-i pipe_path`, `live_supervisor.py` ~1012). FIFO dipegang manager dengan `os.O_RDWR` (`live_stream_engine.py::_run_loop_inner`) — feeder boleh mati/hidup tanpa encoder melihat EOF. Manager me-resolve manifest dari `self.youtube_url` di tiap iterasi loop dan tiap probe 12 detik saat FALLBACK. Supervisor bisa mengambil manager yang sedang jalan via `get_live_audio_relay_manager(task_id)` (sudah dipakai di `live_supervisor.py:699, 1917`).

**Mekanisme baru yang dibutuhkan (kode):**
1. Metode baru `LiveAudioRelayManager.request_source_swap(new_url)`:
   - Set `self.youtube_url = new_url`, set flag `_swap_requested = True`.
   - Hentikan feeder yang sedang jalan (`_terminate_current_feeder()`) — loop yang sedang menunggu feeder keluar, melihat flag, dan `continue` ke resolusi PRIMARY dengan URL baru (pola sama seperti `refresh_restart` yang sudah ada untuk token refresh).
   - Bila manager sedang di loop FALLBACK: flag membuat probe berikutnya langsung memakai URL baru (tanpa menunggu 12 detik — bangunkan dengan event/flag check tiap tick 2 detik).
   - Catat `swap` di log + kembalikan status (URL baru diterima / manager tidak jalan).
2. Handler supervisor `handle_relay_swap_order(task_id, new_url)`:
   - Ambil manager via `get_live_audio_relay_manager(task_id)`; bila tidak ada (task tidak STREAMING di node ini) → laporkan FAILED `MANAGER_NOT_RUNNING` (order tidak boleh diam-diam hilang).
   - Panggil `request_source_swap(new_url)`; tunggu konfirmasi aliran (pre-warm singkat: `wait_for_flow` dengan timeout 30 dtk memakai mekanisme yang sudah ada) → laporkan DONE/FAILED ke Master via laporan status.
3. **Jalur lintas-node (D3):**
   - Edit endpoint (Master) menulis order `relay_swap` ke `live_task_orders` (status PENDING).
   - `derive_orders_for_node` (Master, `core/telemetry/node_failover.py`) diperluas: selain stop/start dari `migration_state`, ikut membaca `live_task_orders` PENDING untuk node itu → order `{"action": "relay_swap", "task_id", "order_id", "audio_url"}`.
   - `apply_node_orders` di `scripts/node_telemetry_push.py` diperluas: aksi `relay_swap` → tulis ke DB lokal (UPDATE `audio_url` + `config_version` pada baris task lokal) + beri sinyal supervisor (via file sinyal / API lokal supervisor — pilih mekanisme yang sudah dipakai supervisor untuk perintah lokal; bila belum ada, tambahkan polling ringan: supervisor mengecek tabel order lokal tiap 15 detik bersamaan dengan cek migrasi yang sudah ada).
   - Supervisor mengeksekusi → update order jadi DONE/FAILED lewat laporan status (saluran yang sudah ada) atau endpoint ack baru `POST /api/live/orders/{order_id}/ack`.
   - Order PENDING >10 menit → EXPIRED (janitor); dashboard menampilkan status order (Terkirim / Diterapkan / Gagal / Kedaluwarsa).

**Batas hot-swap:** hanya `audio_url` (dengan `audio_source_type` tetap `YOUTUBE_URL`/`YOUTUBE_LIVE_RELAY`). Mengubah *jenis* audio (misal relay → file depot) butuh restart encoder karena input ffmpeg berubah — ditangani sebagai `restart_pending`, bukan hot-swap.

### 7.3 Validasi sumber

| Sumber | Validasi | Alat |
|---|---|---|
| URL relay YouTube | Video ada + bukan dihapus/dibatasi + (bila memungkinkan) sedang live / punya manifest HLS | `probe_media_availability` (sudah ada, multi-region) + probe manifest yt-dlp ringan (tanpa unduh, timeout 20 dtk) |
| File video depot | File ada di `live_videos` node tujuan + bisa di-probe (ffprobe: durasi > 0) | Cek via node tujuan (endpoint file yang sudah ada / order validasi); bila node tidak terjangkau → simpan dengan peringatan "belum terverifikasi di node" |
| File audio depot | File ada + format audio valid | Sama seperti video |
| URL video (video_source_type=URL) | URL bisa dijangkau + Content-Type video | HEAD request, timeout 10 dtk |

Validasi berjalan **sinkron** di endpoint edit (timeout total maks 30 dtk) dan **asinkron live** di form (debounce). Kegagalan validasi = 422, tidak ada perubahan tersimpan.

### 7.4 Revisi & rollback

- Setiap edit yang berhasil (termasuk rollback dan bulk-edit per task) menulis 1 baris `live_task_revisions` dengan snapshot penuh + `changed_fields` (diff per field).
- `GET /api/live/tasks/{id}/revisions` mengembalikan daftar (nomor, waktu, operator, alasan, field yang berubah).
- Rollback = edit dengan nilai snapshot lama (validasi ulang sumber — lihat §6.6).
- Revisi pertama dibuat otomatis saat task dibuat (snapshot awal) — rollback selalu punya titik kembali.

### 7.5 Template

- Buat dari task (`source_task_id`) atau dari nol (form sama seperti create).
- `channel_id` opsional: template terikat channel (key ikut dari channel saat apply) atau umum.
- Apply = create task biasa via `schedule_live_stream` (semua guard create berlaku: admission, double-submit 409, kapasitas node).
- Revisi template tidak menyentuh task yang sudah dibuat (D6). Opsi "Terapkan ke task yang memakai template ini" = bulk-edit dengan patch dari template (P2, opsional).

### 7.6 Bulk edit (P2)

- Satu `patch` untuk banyak task; tiap task diproses dengan alur edit lengkap (akses, versi, validasi, revisi, order).
- Hasil per task dilaporkan; kegagalan satu task tidak membatalkan yang lain.
- Batas: 20 task/panggilan; OWNER saja (operator biasa hanya untuk task miliknya — tetap dicek per task).

### 7.7 Perubahan terjadwal (P2)

- Tabel `live_scheduled_changes`: `(id, task_id, apply_at_epoch, patch_json, status, created_by, created_at)`.
- Worker terjadwal (timer yang sudah ada / cron internal Master) menerapkan saat waktunya tiba via alur edit yang sama; gagal (misal task sedang migrasi) → retry 3x tiap 5 menit → FAILED + notifikasi.
- Bisa dibatalkan sebelum `apply_at`.

### 7.8 Notifikasi operator

- Saat relay manager mencatat manifest gagal berulang (sudah ada ring B10 `manifest_failure_summary`), supervisor menandai task dengan `relay_status=DEAD` + alasan (misal `VIDEO_REMOVED`) di laporan status → dashboard menampilkan banner merah di baris task + toast saat operator membuka dashboard.
- Order FAILED/EXPIRED → notifikasi di dashboard (riwayat order per task).

---

## 8. Antarmuka (dashboard)

Mengikuti pola modal yang sudah ada (`modalLiveMetadataEditor`).

1. **Tombol Edit** di tiap baris task (ikon pensil) → modal edit dengan field yang sama seperti form create (`live_create.html`), terisi nilai saat ini + indikator per field:
   - 🟢 Berlaku langsung / 🟡 Hot-swap tanpa henti / 🔴 Butuh restart ±1 menit (penjelasan di bawah field).
   - Validasi URL relay live saat mengetik (✅ hidup / ❌ dihapus + alasan).
   - Tombol "Simpan" + (bila ada perubahan restart) "Simpan & Restart" / "Simpan saja".
2. **Panel quick-fix** — muncul otomatis di baris task yang bermasalah (RECONNECTING berulang / relay mati / ERROR): menampilkan penyebab + tombol: [Ganti URL relay] [Pakai audio darurat] [Ganti video] [Restart stream].
3. **Pemilih template** — di form create: dropdown "Pakai template…" + di tiap task: "Simpan sebagai template".
4. **Riwayat revisi** — tab/modal per task: daftar revisi + tombol "Kembalikan ke versi ini" + diff field.
5. **Status order** — di baris task: kecil "Perubahan terkirim… / diterapkan ✓ / gagal ✗ (klik untuk detail)".
6. **Perubahan terjadwal (P2)** — daftar perubahan terjadwal per task + tombol batal.

---

## 9. Keamanan & peran

- Semua endpoint baru memakai `verify_live_task_access` / `verify_channel_operator_access` (OWNER = semua; operator = hanya task/channel miliknya).
- Template: operator hanya melihat template miliknya + template OWNER (ditandai "Umum"). Template OWNER bisa dipakai semua operator tapi hanya OWNER yang bisa revisi/hapus.
- Bulk-edit: tetap dicek per task (operator tidak bisa menyentuh task operator lain lewat bulk).
- `stream_key` tidak pernah ditampilkan/diedit lewat fitur ini (tetap via jalur key yang sudah ada).
- Semua perubahan mencatat `changed_by` (operator login) — audit trail.

---

## 10. Kasus tepi

| # | Kasus | Perilaku |
|---|---|---|
| E1 | Dua operator edit task yang sama bersamaan | Optimistic locking: yang kedua dapat 409 + konfigurasi terbaru; tidak ada yang tertimpa diam-diam |
| E2 | Edit saat task sedang migrasi | 409 `TASK_MIGRATING`; operator diminta tunggu migrasi selesai |
| E3 | Edit saat task RECONNECTING | Field sumber ditolak (409); judul/prioritas tetap bisa |
| E4 | Hot-swap saat node tidak terjangkau | Order PENDING; berlaku saat node kembali (maks 10 mnt) lalu EXPIRED + notifikasi |
| E5 | Hot-swap tapi manager tidak jalan di node | Order FAILED `MANAGER_NOT_RUNNING`; dashboard memberi tahu; operator bisa restart stream |
| E6 | URL relay baru ternyata mati setelah hot-swap | Manager jatuh ke FALLBACK (audio darurat) seperti biasa; dashboard menampilkan peringatan; operator ganti lagi |
| E7 | Rollback ke revisi yang URL-nya kini mati | Rollback ditolak (422) + tawarkan rollback parsial (field non-sumber saja) |
| E8 | Template dihapus padahal task dibuat darinya | Task tidak terpengaruh (snapshot, D6); `source_task_id`/`template_id` di task jadi NULL |
| E9 | Ganti video ke file yang belum ada di node tujuan | Validasi gagal (422) + petunjuk pre-copy; atau simpan dengan peringatan bila node tidak terjangkau |
| E10 | Perubahan terjadwal saat waktunya task sedang migrasi | Retry 3x/5 mnt → FAILED + notifikasi |
| E11 | Edit task yang broadcast-nya sudah berakhir/dihapus di YouTube | Edit konfigurasi tetap bisa; start berikutnya membuat broadcast baru via jalur yang sudah ada |
| E12 | Double-submit (klik simpan 2x) | Idempotency key dari frontend + guard 409 yang sudah ada dari bundle |

---

## 11. Rencana tes

| Lapisan | Tes |
|---|---|
| Unit (baru) | Edit per status (§7.1); optimistic locking (409); validasi sumber (hapus/valid/bukan-live); diff revisi; rollback; template apply (tanpa key); bulk-edit hasil per task; `request_source_swap` (flag + feeder terminate + re-resolve) |
| Integrasi (baru) | Order lintas-node end-to-end (tulis order → derive → apply → ack) dengan DB tes terisolasi; hot-swap pada relay manager sungguhan (FIFO + feeder silence, tanpa YouTube) |
| Regresi | Seluruh `tests/test_live_*.py` yang ada harus tetap hijau; khususnya `test_live_bundle_b10_pickup.py` (528 baris) + `test_live_nodes_fase_c*.py` |
| Manual (staging) | Skenario 034 end-to-end: task STREAMING → ganti URL relay → verifikasi encoder TIDAK restart (PID sama) + audio baru mengalir + sentinel tetap LIVE |
| Manual (staging) | Ganti video → konfirmasi restart → PID baru + video baru + sentinel LIVE |
| Beban | 20 task bulk-edit; 50 revisi/task (janitor) |

Target: semua tes baru hijau + 0 regresi sebelum merge. Tes memakai DB terisolasi (bukan DB live) — lihat `AGENTS.md` (isolasi tes).

---

## 12. Rencana rollout

| Fase | Isi | Verifikasi | Rollback |
|---|---|---|---|
| **Fase 0** — fondasi | Migrasi skema (1 kolom + 3 tabel + indeks); endpoint `validate-source`; tulis revisi pada create (revisi #1) | Migrasi di DB salinan; endpoint validasi mengembalikan hasil benar untuk URL hidup/mati | Drop tabel baru; kolom `config_version` tidak dipakai kode lama (aman) |
| **Fase 1** — edit task belum jalan | `POST .../edit` untuk IDLE/SCHEDULED/STOPPED/ERROR + modal edit + riwayat revisi + rollback | Edit 1 task IDLE (645) → verifikasi tersimpan + revisi tercatat + rollback bekerja | Nonaktifkan tombol edit (feature flag); data revisi tidak mengganggu kode lama |
| **Fase 2** — hot-swap relay | `request_source_swap` + handler supervisor + order `relay_swap` + quick-fix panel | Staging: task STREAMING → swap URL → PID encoder sama + audio baru + sentinel LIVE; lalu 1 stream produksi terendah | Nonaktifkan order `relay_swap` (derive tidak membaca tabel order); hot-swap kembali ke cara lama (restart) |
| **Fase 3** — restart terkonfirmasi | `restart_pending` + endpoint restart + konfirmasi UI | Ganti video 1 task → restart ±1 mnt → sentinel LIVE | Sama seperti Fase 2 |
| **Fase 4** — template | CRUD template + apply + "simpan sebagai template" | Buat template dari 034 → apply ke channel tes → task benar | Nonaktifkan menu template |
| **Fase 5** — P2 | Bulk-edit + scheduled change + janitor revisi | Bulk 3 task; scheduled change 1 task | Nonaktifkan endpoint P2 |

**Jendela aktivasi:** tiap fase butuh restart web Master (endpoint+UI) dan/atau restart live VPS 8 (supervisor/manager) — dikumpulkan per fase dan dimintakan persetujuan seperti aktivasi 1–5. **Live disentuh terakhir, satu stream terendah dulu** (Fase 2–3 diuji di 1 stream sebelum dibuka ke semua).

---

## 13. Risiko

| Risiko | Dampak | Mitigasi |
|---|---|---|
| Hot-swap merusak aliran audio (bug flag/race) | Encoder kehabisan audio → mati | FIFO tetap dipegang manager (desain yang sama seperti token-refresh yang sudah terbukti); fallback ke audio darurat otomatis; uji staging wajib sebelum produksi |
| Order lintas-node hilang/tidak pernah diterapkan | Operator mengira perubahan berlaku padahal tidak | Status order terlihat di dashboard (PENDING/DONE/FAILED/EXPIRED) + expiry 10 menit + notifikasi gagal |
| Validasi yt-dlp lambat / diblokir | Endpoint edit timeout | Timeout 20 dtk + cache hasil validasi 5 menit per URL + fallback ke probe metadata saja |
| Dua operator edit bersamaan | Lost update | Optimistic locking (D4) + 409 |
| Skema baru merusak migrasi/backup | DB tidak konsisten | Migrasi aditif saja (ADD COLUMN + CREATE TABLE); diuji di DB salinan dulu; backup sebelum migrasi |

---

## 14. Keputusan yang menunggu user

1. **Setujui desain ini?** (Bila ya: implementasi dimulai dari Fase 0–1.)
2. **Hot-swap di semua stream atau bertahap?** Rekomendasi: bertahap (1 stream terendah → semua).
3. **Siapa boleh bulk-edit?** Rekomendasi: OWNER saja untuk lintas-operator; operator biasa hanya task miliknya.
4. **Retensi revisi 50/task — cukup?** (Naikkan bila perlu; murah.)
5. **Perubahan terjadwal (P2) — bangun sekarang atau tunda?** Rekomendasi: tunda sampai P0–P1 terbukti di produksi.

---

## Lampiran A — Peta file yang disentuh

| File | Perubahan |
|---|---|
| `api/routes/live_stream.py` | Endpoint baru §6 + model request `UpdateLiveRequest` + validasi + tulis revisi/order |
| `api/routes/live_nodes.py` | `derive_orders_for_node` membaca `live_task_orders` (aksi baru) |
| `core/telemetry/node_failover.py` | (bila derive tinggal di sini) — sama seperti di atas |
| `scripts/node_telemetry_push.py` | `apply_node_orders`: aksi `relay_swap` / `config_restart` / `emergency_audio` |
| `worker/live_stream_engine.py` | `LiveAudioRelayManager.request_source_swap()` + flag swap di `_run_loop_inner` |
| `worker/live_supervisor.py` | Handler order relay_swap + cek order lokal (polling 15 dtk bersama cek migrasi) + tandai `relay_status` |
| `core/database/` (migrasi) | Skema §5 (1 kolom + 3 tabel + indeks + janitor revisi) |
| `web/templates/live_dashboard.html` | Tombol edit, modal edit, quick-fix, riwayat, status order |
| `web/templates/live_create.html` | Pemilih template |
| `tests/test_live_editable_schedule.py` (baru) | Tes §11 |
| `tests/test_live_bundle_b10_pickup.py` | Regresi (tidak diubah, harus tetap hijau) |

## Lampiran B — Fakta kode yang dipakai desain ini (terverifikasi 25 Nov 2026)

- Tidak ada endpoint edit live task; metadata memakai `POST /api/live/metadata/{task_id}` (`live_stream.py:1877`).
- `CreateLiveRequest` (`live_stream.py:220`): title, channel_id, video, video_source_type (DEPOT/URL), video_url, audio_source_type (ORIGINAL/MUTED/DEPOT/YOUTUBE_URL/DIRECT_URL), audio_filename, audio_url, mute_video_audio, key, rtmp_url, start/stop_time, auto_stop_hours, duration_min, repeat (manual/daily/weekly), operator, resolution (720p/1080p), priority 1–5, unlist_replay_on_stop, start_now, force, node_id, node_confirmed.
- URL relay = `task.audio_url` saat `audio_source_type` ∈ (YOUTUBE_URL, YOUTUBE_LIVE_RELAY, LIVE_RELAY) (`live_supervisor.py:911-912`).
- Encoder membaca audio dari FIFO (`live_supervisor.py` ~1012); manager memegang FIFO `O_RDWR` (`live_stream_engine.py::_run_loop_inner`); manager bisa diambil supervisor via `get_live_audio_relay_manager(task_id)` (`live_supervisor.py:699, 1917`).
- Order lintas-node saat ini hanya `stop`/`start` (`node_failover.py:441, 449`; `node_telemetry_push.py:209`).
- Validasi URL sudah ada: `probe_media_availability` via `POST /api/live/qa-probe` (`live_stream.py:1307`).
- Scoping operator: `verify_live_task_access` + `verify_channel_operator_access` (`live_stream.py:85-133`).
- Double-submit guard 409 sudah ada (bundle `e2779ed`).
