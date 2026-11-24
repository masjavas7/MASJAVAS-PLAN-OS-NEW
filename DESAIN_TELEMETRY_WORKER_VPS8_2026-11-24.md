# DESAIN PERBAIKAN TELEMETRY — Monitor Worker + Kinerja VPS 8

**Tanggal:** 24 November 2026, 03:30 WIB
**Diminta oleh:** Owner (Abdul Aziz) — "perbaiki /telemetry agar bisa memantau
worker dan kinerja VPS 8"
**Status:** DESAIN — implementasi didelegasikan ke subagent implementer
**Halaman target:** https://masjavastube.my.id/telemetry

---

## 1. KEADAAN SEKARANG (terverifikasi 03:24–03:30 WIB)

### Yang sudah ada
- **Kolektor** (`core/telemetry/collector.py`, jalan tiap 30 dtk di proses web):
  menulis ke tabel `telemetry_samples` — CPU, load, RAM, disk, ukuran DB,
  antrean (queued/rendering/uploading), published_24h, failed_24h,
  worker_last_change, top process, services_ok, health_score.
  Sampel terbaru (03:23): CPU 4,5%, load 0,15, RAM 34,8%, disk 87,1%
  (18,59 GB free), queued 210, published_24h 212, failed_24h 0, skor 75.
- **Endpoint** `/api/telemetry/health` (OWNER only): metrik real-time Master
  (VPS, DB, channel, vault, task, load, top_process, worker_last_change,
  services_ok, health_score, collector, drive_volume, network).
- **Template** `telemetry.html` (986 baris): panel sentinel L1–L6, skor,
  load, CPU, antrean. Elemen `workerVal` hanya menampilkan
  "berapa lama lalu worker berubah" (`fmtAgo(worker_last_change)`).
- **Probe lane** (`core/telemetry/lane_probes.py`): probe_host_metrics,
  probe_lane_publish, probe_lane_ingestion, probe_lane_live,
  probe_lane_loopgen, probe_lane_janitor.

### Yang TIDAK ada (kesenjangan)
1. **Worker:** tidak ada PID, lane (`--lane publish`), uptime, atau laju
   error (semaphore/DB-lock) di halaman mana pun.
2. **VPS 8:** TIDAK ADA data sama sekali di telemetry — tidak ada tabel,
   endpoint, kolektor, atau tampilan untuk node live-01.
3. **Prep worker VPS 8:** tidak terpantau (Stage 2 throughput, ingestion).
4. **Kematian encoder:** tidak ada metrik (jumlah/jam, exit code) di halaman
   mana pun.

---

## 2. DESAIN

### 2.1 Prinsip
1. **Aditif, bukan penggantian.** Semua endpoint + tampilan yang ada TIDAK
   DIUBAH perilakunya. Fitur baru ditambahkan sebagai kunci/section baru.
2. **Push, bukan pull.** VPS 8 MENDORONG metrik ke Master (bukan Master yang
   SSH ke VPS 8). Web tidak pernah block menunggu VPS 8.
3. **Degradasi anggun.** Bila VPS 8 tidak melapor > 90 dtk, halaman
   menampilkan "STALE / OFFLINE" — bukan error, bukan angka palsu.
4. **Satu sumber kebenaran.** Semua data telemetry dibaca dari Master DB.
   Tidak ada query langsung ke VPS 8 dari proses web.
5. **Ringan.** Pusher VPS 8 memakai psutil + SQLite read-only + journalctl
   terhitung (bukan grep seluruh journal). Kolektor Master tidak bertambah
   berat.

### 2.2 Skema DB baru: `node_telemetry_samples`

```sql
CREATE TABLE IF NOT EXISTS node_telemetry_samples (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    node_id TEXT NOT NULL,              -- 'live-01', 'live-02', ...
    ts INTEGER NOT NULL,                -- epoch detik (waktu push)
    cpu_percent REAL NOT NULL DEFAULT 0,
    load1 REAL NOT NULL DEFAULT 0,
    load5 REAL NOT NULL DEFAULT 0,
    load15 REAL NOT NULL DEFAULT 0,
    ram_percent REAL NOT NULL DEFAULT 0,
    disk_percent REAL NOT NULL DEFAULT 0,
    disk_free_gb REAL NOT NULL DEFAULT 0,
    streams_active INTEGER NOT NULL DEFAULT 0,
    streams_total INTEGER NOT NULL DEFAULT 0,
    encoders_alive INTEGER NOT NULL DEFAULT 0,
    prep_active INTEGER NOT NULL DEFAULT 0,     -- 1 = prep worker jalan
    prep_stage2_active INTEGER NOT NULL DEFAULT 0,
    prep_ingestion_active INTEGER NOT NULL DEFAULT 0,
    encoder_deaths_1h INTEGER NOT NULL DEFAULT 0,
    rtmp_timeouts_1h INTEGER NOT NULL DEFAULT 0,
    uptime_sec INTEGER NOT NULL DEFAULT 0,
    version TEXT NOT NULL DEFAULT ''    -- git commit singkat
);
CREATE INDEX IF NOT EXISTS idx_node_tel_node_ts
    ON node_telemetry_samples(node_id, ts);
```

**Retensi:** hapus sampel > 7 hari (job pembersih yang sama dengan
`telemetry_samples`, atau DELETE berkala di endpoint push bila jumlah baris
> 20.000/node).

### 2.3 Endpoint baru di `api/routes/telemetry.py`

#### `POST /api/telemetry/node-report` (VPS 8 → Master)
- **Auth:** header `X-Node-Token` — token bersama yang disimpan di
  `.env` Master (`NODE_TELEMETRY_TOKEN`) dan `.env` VPS 8. Bila token
  tidak cocok → 401. JANGAN log token.
- **Body (JSON):** semua kolom §2.2 kecuali `id` (node_id, ts, metrik...).
- **Validasi:** node_id cocok pola `^[a-z0-9][a-z0-9-]{0,31}$`; ts dalam
  ±5 menit dari waktu Master; angka dalam rentang wajar (0–100 untuk
  persen, >= 0 untuk hitungan).
- **Perilaku:** INSERT 1 baris + DELETE sampel > 7 hari untuk node itu
  (bila perlu). Respons: `{"ok": true, "id": <rowid>}`.
- **Rate limit:** tolak bila node yang sama push < 10 dtk sejak push
  terakhir (429). Mencegah banjir.

#### `GET /api/telemetry/nodes` (halaman → Master, OWNER only)
- Mengembalikan sampel TERBARU per node + staleness:
```json
{
  "nodes": [
    {
      "node_id": "live-01",
      "ts": 1795465423,
      "age_sec": 12,
      "stale": false,
      "cpu_percent": 45.2, "load1": 8.47, "load5": 8.64, "load15": 7.58,
      "ram_percent": 5.1, "disk_percent": 16.0, "disk_free_gb": 244.0,
      "streams_active": 3, "streams_total": 3,
      "encoders_alive": 3,
      "prep_active": 1, "prep_stage2_active": 4, "prep_ingestion_active": 2,
      "encoder_deaths_1h": 16, "rtmp_timeouts_1h": 3,
      "uptime_sec": 43200, "version": "cea60b8"
    }
  ],
  "timestamp": 1795465435
}
```
- `stale` = true bila `age_sec` > 90.

### 2.4 Worker metrics di `/api/telemetry/health` (tambah kunci baru)

Tambahkan kunci `worker` ke respons (bentuk lama tidak berubah):
```json
"worker": {
  "pid": 306566,
  "uptime_sec": 24120,
  "lane": "publish",          // dari /proc/<pid>/cmdline (--lane X) atau "default"
  "status": "running",        // running | stopped | unknown
  "cmdline": "python -m worker.autonomous_daemon --lane publish",
  "cpu_percent": 12.3,        // via psutil.Process(pid), bila PID ada
  "memory_mb": 412.5
}
```
- **Sumber:** `systemctl show -p MainPID masjavas-os-worker.service` untuk
  PID (cache 60 dtk — jangan panggil systemctl tiap request); cmdline dari
  `/proc/<pid>/cmdline`; uptime dari `ActiveEnterTimestamp` atau
  `psutil.Process(pid).create_time()`.
- **Bila worker tidak jalan:** `status: "stopped"`, pid 0, field lain null.
- **Error rates (opsional, bila murah):** hitung semaphore-busy + DB-lock
  dari journal worker 1 jam terakhir, cache 5 menit (pola yang sama dengan
  `_DRIVE_VOLUME_CACHE`). Bila terlalu berat, TINGGALKAN untuk fase 2 —
  jangan memperlambat `/health`.

### 2.5 Pusher VPS 8: `scripts/node_telemetry_push.py` + timer

**Script** (Python stdlib + psutil + sqlite3, TANPA dependensi baru):
1. Baca metrik host via psutil: CPU, load, RAM, disk, uptime.
2. Baca status stream dari DB lokal (read-only, `mode=ro&timeout=5000`):
   `SELECT status, COUNT(*) FROM live_stream_tasks GROUP BY status` →
   streams_active (STREAMING), streams_total.
3. Hitung encoder hidup: `pgrep -f "ffmpeg -nostdin -stats"` (jumlah proses,
   BUKAN cmdline — jangan kirim cmdline yang mengandung stream key).
4. Status prep: `systemctl is-active masjavas-prep.service` + hitung slot
   Stage 2 aktif dari journal 5 menit terakhir (pola "Starting render") —
   ATAU baca dari log bila ada cara lebih murah. Minimal: prep_active 0/1.
5. Kematian encoder 1 jam: `journalctl -u masjavas-live.service
   --since "1 hour ago" | grep -c "mati — exit code"`.
   RTMP timeout 1 jam: pola "Connection timed out" yang sama.
   **PERHATIAN:** journalctl di VPS 8 memakai zona CET — `--since "1 hour
   ago"` relatif, aman dari masalah zona waktu.
6. Versi: `git -C /opt/masjavas-os rev-parse --short HEAD` (bila gagal, "").
7. POST JSON ke `http://100.65.206.22:8080/api/telemetry/node-report`
   dengan header `X-Node-Token` dari `/opt/masjavas-os/.env`
   (kunci `NODE_TELEMETRY_TOKEN`).
8. Timeout HTTP 10 dtk. Bila gagal: log ke stderr + exit 0 (jangan crash
   timer; kegagalan push akan terlihat sebagai STALE di halaman).

**Systemd timer** `masjavas-node-telemetry.timer` (setiap 30 dtk) +
service `masjavas-node-telemetry.service` (oneshot, User=root atau ubuntu
— pilih yang bisa baca DB + journal; root paling aman untuk journal).

**Token:** generate 32-byte hex di Master (`python3 -c "import secrets;
print(secrets.token_hex(16))"`), simpan di `.env` Master
(`NODE_TELEMETRY_TOKEN=<token>`) dan `.env` VPS 8 (kunci sama).
**JANGAN pernah mencetak token di log, laporan, atau chat.**

### 2.6 Template `telemetry.html` (tambah 2 section)

#### Section WORKER (di bawah panel skor, di atas grafik)
```
┌─────────────────────────────────────────────────────────────┐
│ 👷 Publish Worker                        [● RUNNING]        │
│ PID 306566  •  Lane: publish  •  Uptime 6j 42m              │
│ CPU 12,3%  •  RAM 412 MB                                    │
│ Hari ini: 2 PUBLISHED  •  0 FAILED  •  209 QUEUED           │
└─────────────────────────────────────────────────────────────┘
```
- Data dari `data.worker` (endpoint /health) + `data.tasks` (sudah ada).
- Badge: hijau RUNNING / merah STOPPED / abu-abu UNKNOWN.
- Auto-refresh bersamaan dengan refresh halaman yang ada (pola JS yang sama).

#### Section VPS 8 / NODE (di bawah Worker, di atas grafik)
```
┌─────────────────────────────────────────────────────────────┐
│ 🖥️ live-01 (VPS 8)              [● ONLINE • 12 dtk lalu]    │
│ Load 8,47 / 8,64 / 7,58  •  CPU 45%  •  RAM 5%  •  Disk 16% │
│ Stream 3/3  •  Encoder 3 hidup  •  Prep: Stage 2 (4) + Ing (2)│
│ Kematian encoder 16/jam  •  RTMP timeout 3/jam  •  v cea60b8│
└─────────────────────────────────────────────────────────────┘
```
- Data dari `GET /api/telemetry/nodes` (fetch terpisah, 30 dtk).
- Badge: hijau ONLINE (age < 90 dtk) / kuning STALE (90 dtk–5 mnt) /
  merah OFFLINE (> 5 mnt atau tidak ada sampel).
- Bila tidak ada node sama sekali: tampilkan "Belum ada node yang melapor
  — pusher VPS 8 belum aktif" (bukan error).
- Desain generik per-node (loop) agar live-02 dst otomatis muncul.

### 2.7 Keamanan
- `/api/telemetry/nodes` OWNER only (pola `_require_owner` yang sama).
- `/api/telemetry/node-report` token-based (bukan session). Token 32-byte hex.
- TIDAK ADA stream key / URL RTMP / URL googlevideo dalam payload apa pun.
  Pusher HANYA mengirim hitungan + metrik, bukan cmdline proses.
- Validasi input ketat di node-report (pola node_id, rentang angka, ts).

---

## 3. RENCANA IMPLEMENTASI (untuk subagent implementer)

### Urutan kerja
1. **Backup:** `cp` file yang akan diubah (route, collector bila disentuh,
   template, .env) ke `/opt/masjavas-os/data/backups/telemetry-vps8-<timestamp>/`.
   Backup DB: `sqlite3` backup via API `con.backup()` atau `cp` file DB
   (saat web jalan, pakai `.backup` sqlite3 — JANGAN `cp` mentah file WAL).
2. **Skema:** buat tabel `node_telemetry_samples` via migrasi SQL (idempotent
   `CREATE TABLE IF NOT EXISTS` + `CREATE INDEX IF NOT EXISTS`). Verifikasi
   dengan `.schema`.
3. **Backend Master:** tambah endpoint node-report + nodes + kunci worker di
   `/health` (file `api/routes/telemetry.py`). Ikuti pola yang ada
   (cache, error handling, `_require_owner`).
   **PERHATIAN:** beberapa file di `core/telemetry/` root-owned — bila perlu
   mengeditnya, `sudo chown ubuntu:ubuntu` dulu (pelajaran AGENTS.md).
   Sebisa mungkin HINDARI mengedit collector — worker metrics dibaca di
   endpoint `/health`, bukan di collector.
4. **Token:** generate + simpan di `.env` Master dan VPS 8. JANGAN cetak token.
5. **Pusher VPS 8:** tulis `scripts/node_telemetry_push.py` + unit timer.
   Uji manual 1× push (cek baris masuk ke `node_telemetry_samples` di Master).
   Aktifkan timer.
6. **Template:** tambah section Worker + Node (HTML + JS fetch). Ikuti gaya
   Tailwind yang ada.
7. **Tes:** `python -m py_compile` semua file Python yang diubah; jalankan
   test suite terkait bila ada (`tests/test_*telemetry*`).
8. **Deploy:** restart web SATU KALI (aturan max 1 restart/15 menit lintas
   agent — cek restart terakhir dulu; web terakhir restart 03:03:38 WIB).
   Verifikasi pasca-restart (lihat §4).
9. **Rollback (bila gagal):** restore file dari backup + restart web kembali.
   Tabel `node_telemetry_samples` boleh tetap ada (kosong, tidak merusak).

### Larangan
- JANGAN ubah perilaku endpoint/template yang ada (hanya tambah).
- JANGAN sentuh live service / worker service / publish.
- JANGAN cetak token / stream key di log, laporan, atau chat.
- JANGAN jalankan migrasi skema yang tidak idempotent.
- JANGAN restart web lebih dari 1× tanpa verifikasi di antaranya.

---

## 4. VERIFIKASI (wajib sebelum lapor selesai)

### Pra-deploy
- [ ] `py_compile` semua file Python yang diubah: OK.
- [ ] Test suite telemetry (bila ada): hijau.
- [ ] Backup file + DB tercatat (path + timestamp).

### Pasca-deploy (web restart)
- [ ] Web responsif: `curl http://127.0.0.1:8080/` 200 < 2 dtk.
- [ ] `/telemetry` render 200 (via browser login / curl dengan session).
- [ ] `GET /api/telemetry/nodes` mengembalikan live-01 dengan `age_sec` < 90
      (setelah pusher jalan 1 menit).
- [ ] `GET /api/telemetry/health` mengandung kunci `worker` dengan PID +
      lane yang benar (cocokkan dengan `systemctl show` worker).
- [ ] Section Worker + VPS 8 tampil di halaman (screenshot / curl HTML
      mengandung id section baru).
- [ ] Tidak ada error baru di journal web 10 menit pasca-restart.
- [ ] Publish tidak terganggu: PUBLISHED 24 Nov bertambah normal.

### Observasi 1 jam
- [ ] Sampel node masuk tiap ~30 dtk (hitung baris `node_telemetry_samples`
      bertambah).
- [ ] Tidak ada error push di journal pusher VPS 8.
- [ ] Halaman tidak melambat (load `/telemetry` < 3 dtk).

---

## 5. SARAN & IDE PERBAIKAN LAIN (dari laporan kinerja 03:20)

Untuk owner — di luar scope telemetry, disampaikan bersamaan:

| # | Saran | Alasan (bukti) | Prioritas |
|---|-------|----------------|-----------|
| S1 | **Selidiki exit code 224/152** (T7 prompt Antigravity) | 73/113 kematian encoder; ~16/jam; penyebab belum diketahui | P0 |
| S2 | **Throttle prep saat load VPS 8 > 7** | Load 8,47/8,64/7,58 — box hampir jenuh; prep + encoder berebut CPU | P0 |
| S3 | **Bersihkan disk Master (88%, 19 GB free)** | Risiko disk penuh menghentikan publish + DB | P1 |
| S4 | **DR restore drill** | Backup terbukti valid, restore belum pernah diuji | P1 |
| S5 | **Breakdown kuota per-vault di telemetry** | 9.750 unit tercatat agregat; risiko per-vault tidak terlihat | P2 |
| S6 | **Ukur throughput prep (video/jam, job/jam)** | Prep jalan tapi throughput belum terukur | P2 |
| S7 | **Setujui arsitektur multi-node** (dokumen 20:02) | Menentukan desain sebelum live-02 ditambah | P1 |
| S8 | **Alert otomatis load/disk/kematian encoder** | Saat ini hanya pantauan manual + monitor 30 mnt | P2 |
