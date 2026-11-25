# AUDIT D1 — Metrik per-stream + dashboard + alert Telegram (Fase 3)

**Worker:** D1 · 25 Nov 2026 ±11:40 WIB · basis main `0bf5a3b` (E4+D2 merge)
**Aturan:** L2 — tanpa perubahan schema. Semua klaim di bawah dari `grep` + baca kode di
`origin/main`; nomor baris bisa bergeser antar-commit.

## 1. Yang SUDAH ADA (jangan diduplikasi)

| Kebutuhan D1 | Sudah ada | Lokasi (file:baris) |
|---|---|---|
| Parse baris `-stats` ffmpeg (size/time) | `parse_ffmpeg_stats_tail()` → `(size_bytes, time_sec)`; baca 64 KB ekor, pisah `\r`/`\n` | `worker/live_supervisor.py:516` |
| Progress stream dari log | `get_stream_progress(task_id)` → log `data/live_logs/live_<task>.log` | `worker/live_supervisor.py:546` |
| Bytes terkirim per stream | `get_stream_bytes_sent(pid, task_id)` — size log → CPU ticks → `/proc/<pid>/io` | `worker/live_supervisor.py:468` |
| Telemetri log (bitrate string, fps, speed, frame, health, now_playing) | `get_live_task_telemetry(task_id)` — 40 baris akhir log; health HEALTHY/DEGRADED/ERROR/YOUTUBE_OFFLINE | `worker/live_stream_engine.py:809` |
| Endpoint telemetri per task | `GET /api/live/telemetry/{task_id}` | `api/routes/live_stream.py:3908` |
| Telemetri di daftar task | `GET /api/live/tasks` lampirkan `telemetry` + `bitrate_display` + `uptime_str` + `stream_health` (STREAMING + PID hidup saja) | `api/routes/live_stream.py:922-934` |
| Snapshot berkala bitrate/fps/speed/CCV | `sample_active_live_streams()` → tabel `live_metrics_snapshots` (dedup 25 dtk, prune 7 hari) | `core/intelligence/live_metrics_collector.py:39`; schema `core/intelligence/streaming_schema.py:18` |
| Dashboard stream (CCV, tren, reconnect) | `core/live_intelligence/service.py:115` — `reconnect_count`, `started_at`, CCV, freshness | `core/live_intelligence/service.py:50-130` |
| Halaman monitor live | `web/templates/live_monitor.html` (457 baris; polling `/api/live/tasks` 2 dtk + `/api/live/capacity` 3 dtk; kolom: saluran, resolusi, prioritas, status, bitrate & fps, uptime, data terkirim, aksi) | `web/templates/live_monitor.html:199-209,297` |
| Halaman dashboard live | `web/templates/live_dashboard.html`, `live_nodes.html`, `live_command.html` | `web/templates/` |
| `reconnect_count` | Kolom SUDAH ADA di `live_stream_tasks` (`core/database/models.py:388`, migrasi `:1477`); dirawat supervisor: +1 tiap reconnect (`live_supervisor.py:2282,2332,2394`), reset setelah stabil ≥120 dtk (P11, `:2175`) | — |
| Sentinel YouTube per-stream | `core/youtube/live_health_sentinel.py` — status LIVE / WARMING_UP / UNVERIFIED / YOUTUBE_OFFLINE; vonis OFFLINE 2× berurutan + circuit breaker 60 mnt; cache `_HEALTH_CACHE` | `core/youtube/live_health_sentinel.py:52-144,307-401` |
| Sentinel node (W1–W7) | `core/telemetry/sentinel.py:300` `check_rule_w6_live_stream` (stream down); `core/telemetry/rules.py` G1–G7 (node-level, dedup DB 12 jam) | `core/telemetry/sentinel.py`, `core/telemetry/rules.py` |
| Pengirim Telegram | `send_telegram_alert()` — token/chat dari `settings` + override `system_settings` DB; cooldown in-memory 15 mnt per `dedup_key`; non-blocking, fail-open | `core/telemetry/telegram_notifier.py:31-33,96-168` |
| Alert Telegram live yang sudah ada | (a) restart kronis >3×/60 mnt, cooldown 1 jam/task — `live_supervisor.py:2571-2595` (`_record_restart_and_alert`); (b) auto-heal gagal `live_autoheal_fail_` — `live_stream_engine.py:1406`; (c) sentinel kill circuit-breaker `sentinel_kill_cb_` — `:1599`; (d) stream terputus `live_error_` — `:1690` | — |
| Telemetri endpoint umum | `GET /telemetry/events` (riwayat alert + event) | `api/routes/telemetry.py:632` |
| FIFO audio per task | Pipe `/tmp/masjavas_live_audio_<task_id>.pipe` — feeder (HLS/concat/silence) memegang pipe ini; PID feeder dapat ditemukan lewat cmdline yang memuat path pipe | `worker/live_stream_engine.py:3162,3457` |
| CPU encoder (mentah, tidak diekspos) | Ring buffer pre-mortem `_premortem` — `psutil.Process(pid).cpu_percent()` per tick 5 dtk, 12 sampel | `worker/live_supervisor.py:2406-2436` |
| Deteksi beku (tidak diekspos sbg metrik) | `_watchdog_frozen_duration()` + `progress_verdict` di `run_health_check` (size/time diam ≥30 dtk → kill/reconnect) | `worker/live_supervisor.py:2437,2104-2130` |

## 2. CELAH (yang D1 bangun — tidak ada duplikat)

1. **Frame drop (`dup`/`drop`)** — tidak di-parse di mana pun. Baris `-stats`
   memilikinya (`frame=… fps=… dup=… drop=…`), tapi `parse_ffmpeg_stats_tail`
   hanya mengambil `size=`/`time=`, dan `get_live_task_telemetry` hanya
   `bitrate/fps/speed/time/frame`.
2. **%CPU per-stream (encoder + feeder)** — tidak ada. Yang ada hanya sampler
   global + governor; `_premortem` hanya encoder dan tidak diekspos;
   `cpu_display` di `/api/live/tasks` adalah **placeholder hardcode yang salah**:
   `"< 1% (Zero-CPU Stream Copy)"` (`api/routes/live_stream.py:930`) padahal
   terukur ~0,24–0,41 core/stream (Fase 0 M).
3. **Bitrate numerik + kondisi beku sebagai metrik** — bitrate hanya string
   (`"1234kbits/s"`); tidak ada nilai `bitrate_kbps` numerik di jalur
   dashboard; watchdog mendeteksi beku tapi tidak mengeksposnya sebagai
   metrik/alert kondisi tersendiri.
4. **Satu snapshot per-stream gabungan** — bitrate/fps/speed, drop, uptime,
   reconnect, %CPU tersebar di 3 tempat (log parser, `live_metrics_snapshots`,
   `live_intelligence/service.py`); tidak ada satu endpoint/objek yang
   menggabungkan kelimanya.
5. **Alert kondisi per-stream** — yang ada hanya 4 pemicu di atas (§1).
   **Belum ada alert untuk:** encoder mati (STREAMING tapi PID mati — saat ini
   hanya di-reconcile diam-diam), bitrate 0/beku, reconnect berulang berbasis
   `reconnect_count` (yang ada berbasis hitung restart di memori), %CPU
   melewati ambang.
6. **Dashboard** — `live_monitor.html` tidak menampilkan CPU riil, drop, atau
   reconnect (kolom Reconnect hanya muncul sebagai `RETRY n/10` saat status
   RECONNECTING).

## 3. Keputusan desain D1 (L2, tanpa schema)

- Modul baru **`core/telemetry/live_stream_metrics.py`** — satu tempat untuk:
  semua ambang bernama (`D1_*`), parser baris `-stats` lengkap
  (bitrate/fps/speed/frame/dup/drop/time/size), `collect_stream_metrics()`
  (bitrate, drop, uptime, reconnect, %CPU encoder+feeder, status sentinel),
  dan `D1AlertEvaluator` (4 kondisi + cooldown per stream/kondisi).
- **Sumber data hanya yang sudah ada:** log `-stats` ffmpeg + `psutil`
  (encoder PID dari DB + feeder lewat cmdline pemuat pipe
  `/tmp/masjavas_live_audio_<task_id>.pipe`) + sentinel cache + kolom DB
  yang sudah ada (`pid`, `started_at_epoch`, `reconnect_count`,
  `bytes_sent`). **Tanpa tabel/kolom baru, tanpa tulis DB.**
- **Alert reuse** `send_telegram_alert()` (cooldown 15 mnt bawaan) +
  cooldown D1 sendiri per `(task_id, kondisi)` — tidak ada pengirim baru.
- **Dashboard:** endpoint baru `GET /api/live/metrics` (read-only, RBAC sama
  seperti `/api/live/tasks`) + 2 kolom baru di `live_monitor.html`
  (CPU Enc+Feed, Drop/Reconnect); `cpu_display` hardcode tidak diubah di API
  (konsumen lain), hanya ditimpa di tampilan monitor oleh nilai riil.
- **Wiring alert:** satu panggilan best-effort di akhir
  `LiveSupervisor.run_health_check()` (throttle 30 dtk, fail-open) — aktif
  saat restart `masjavas-live` berikutnya di VPS 8. Tidak ada service/timer
  baru.
- **Batasan yang dicatat:** metrik bersifat **node-local** — log + proses
  hanya terlihat di node yang menjalankan encoder (VPS 8). Endpoint di
  Master hanya bermakna untuk encoder lokal, sama seperti
  `get_live_task_telemetry` yang sudah ada. `cpu_percent` psutil butuh 2
  panggilan berurutan untuk nilai bermakna (panggilan pertama 0,0);
  evaluator menoleransinya (sampel pertama tidak memicu alert CPU).
