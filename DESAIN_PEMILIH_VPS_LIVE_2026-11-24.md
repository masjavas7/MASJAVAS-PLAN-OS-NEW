# Desain Pemilih VPS untuk Start Live — MASJAVAS TUBE OS

Tanggal: 24 Nov 2026, 14:21 WIB. Perintah user: saat user mau start live, beri opsi
VPS mana yang menjalankan; user memilih berdasarkan VPS yang tersedia dan masih
longgar; sistem merekomendasikan yang paling sehat/optimal; arsitektur harus siap
tambah VPS baru kapan saja.

## 1. Temuan hari ini (bukti, bukan asumsi)

- Task live baru dibuat TANPA penugasan node: 3 relay otonom (WULAN/088/034)
  punya `assigned_node_id=NULL` → jalan di node yang membuatnya (Master).
  Hanya 706/114/707 yang `assigned=live-01`.
- 3 relay bahkan TIDAK ADA di DB VPS 8 — DR sync tidak mereplikasi task tanpa
  penugasan (atau filternya hanya task ter-assign).
- `live_nodes` ada (master max_streams 10, live-01 max_streams 70) tapi
  heartbeat TERAKHIR ~03:05 WIB (~11 jam lalu); cpu/mem/disk/load semua NULL.
  Pipeline telemetri MATI. `node_events` hanya 1 baris (deploy skema).
- Skema tidak sinkron: Master punya `node_telemetry_samples`, VPS 8 TIDAK.
- Token OAuth tidak bisa didekripsi di VPS 8 (181× gagal 24 Nov; 0× di Master)
  → manajemen broadcast via API lumpuh di VPS 8 (RTMP tetap jalan).
- Load VPS 8 sudah 6–7/8 vCPU — tambah 3 relay akan menjenuhkan.
- Kode telemetri pernah dibangun (commit `0bb9cb5` pusher metrik VPS 8,
  `99a037d` cockpit Sentinel node-aware, `ddc853a` monitor live-01) — Fase A
  WAJIB memeriksa dulu apa yang sudah ada sebelum menulis yang baru
  (revive > rebuild).

## 2. Prinsip arsitektur (mengikat, untuk N VPS)

1. **Master DB = satu-satunya sumber kebenaran** (`live_nodes` +
   `node_telemetry_samples`). Node lain tidak mengambil keputusan dari DB lokal.
2. **Keputusan dari data telemetri, bukan hardcoded.** Tidak ada lagi
   `if node == 'live-01'` di jalur pengambilan keputusan.
3. **Node baru = daftar saja.** Satu perintah join + heartbeat; tanpa ubah kode,
   tanpa deploy ulang Master. Kapasitas dibaca dari `live_nodes.max_streams` /
   `max_prep_slots` (kolom sudah ada).
4. **User memilih, sistem merekomendasikan.** Default = Otomatis (skor tertinggi);
   user boleh override ke node mana pun yang ONLINE.
5. **Failover manual, tidak diam-diam.** Bila node pilihan mati di tengah siaran,
   sistem MENAWARKAN pindah (dengan konfirmasi), tidak memindah sepihak.
6. **Skor transparan.** Setiap skor menampilkan alasan dalam bahasa manusia
   ("load 6,9/8 — hampir penuh", "3 dari maks 70 stream").

## 3. Telemetri per node (heartbeat 30 detik → API Master)

Setiap node mendorong: `load1/5/15, cpu%, mem%, disk_free_gb, tx_mbps,
streams_aktif, ffmpeg_slots_terpakai, encoder_deaths_1h, rtmp_timeouts_1h,
last_error, git_commit`. Disimpan ke `node_telemetry_samples` (retensi 7 hari,
agregasi per jam setelah itu) + `live_nodes` di-update (kolom terakhir).

Status node dari umur heartbeat:
- ONLINE: hb < 90 detik (hijau, bisa dipilih).
- DEGRADED: hb 90 detik – 5 menit (kuning, bisa dipilih dengan peringatan).
- OFFLINE: hb > 5 menit (merah, TIDAK bisa dipilih; tugasnya ditandai).

## 4. Skor kesehatan 0–100 (dihitung server-side di Master)

| Komponen | Bobot | Cara hitung |
|---|---|---|
| Headroom CPU | 35 | `35 * (1 - load1/vcpu)` (negatif → 0) |
| Headroom stream | 25 | `25 * (1 - streams_aktif/max_streams)` |
| Stabilitas | 20 | `20 - min(20, deaths_1h*2 + timeouts_1h)` |
| Jaringan | 10 | `10 * (1 - tx_mbps/max_tx_budget)` |
| Kesegaran data | 10 | 10 bila hb < 60 dtk, turun linear ke 0 di 5 mnt |

Ambang: **≥80 DIREKOMENDASIKAN** (hijau, badge) · **50–79 bisa dipakai**
(kuning) · **<50 hindari** (merah — hanya bila user paksa + konfirmasi) ·
**OFFLINE terkunci**. Skor dihitung ulang tiap heartbeat masuk; API selalu
mengembalikan skor + komponen + alasan teks.

Contoh hari ini (ilustrasi): VPS 8 load ~6,5/8 → headroom CPU ≈ 6,5/35 →
skor total ~55–65 (kuning) meski stream baru 3/70 — benar, karena CPU-nya
yang sempit, bukan slot stream-nya.

## 5. Alur start live baru (Fase B)

1. User klik "Mulai Siar" → modal pemilih VPS (bukan langsung start).
2. Modal menampilkan kartu per node ONLINE/DEGRADED: nama, role, skor + alasan,
   load, stream aktif/maks, kematian encoder/jam, umur heartbeat.
3. Badge "DIREKOMENDASIKAN" di skor tertinggi + tombol "Otomatis (paling sehat)"
   sebagai default terpilih.
4. User pilih node (atau Otomatis) → `assigned_node_id` ditulis saat task dibuat
   → supervisor node tujuan mengeksekusi (lease 60 detik, diperpanjang tiap
   heartbeat task; `node_status` mencerminkan status di node tujuan).
5. Mode otonom (relay otomatis): selalu Otomatis — task baru ke skor tertinggi
   saat dibuat. TIDAK PERNAH lagi `assigned_node_id=NULL`.
6. Bila node tujuan OFFLINE saat start diklik → tolak dengan pesan + tawarkan
   node rekomendasi berikutnya.

## 6. API (Fase A–B)

- `GET /api/live/nodes` — daftar node + skor + komponen + rekomendasi + alasan.
- `POST /api/live/nodes/register` — node baru mendaftar (OWNER; body: node_id,
  hostname, tailnet_ip, public_ip, role, max_streams, max_prep_slots).
- `POST /api/live/nodes/{id}/heartbeat` — node mendorong telemetri (auth token
  node via Tailscale; pola sama seperti token kalender).
- `GET /api/live/nodes/{id}/streams` — stream yang jalan di node itu.
- `POST /api/live/tasks/{id}/assign` — (Fase C) pindahkan task ke node lain
  (manual, OWNER; hanya bila task STOPPED atau dengan konfirmasi failover).

## 7. Fase pembangunan

- **Fase A — Fondasi (SEKARANG).** (1) Periksa kode telemetri yang sudah ada
  (`0bb9cb5`/`99a037d`/`ddc853a`, `core/telemetry/`) → hidupkan/perbaiki pusher
  di Master + VPS 8 sampai `live_nodes` + `node_telemetry_samples` terisi dan
  umur heartbeat < 90 detik. (2) Samakan skema: `node_telemetry_samples` di
  VPS 8. (3) Modul skor + `GET /api/live/nodes`. (4) Tes di temp DB (JANGAN
  pernah DB live), semua hijau. (5) Commit + push satu exec, file sendiri saja.
  TIDAK restart `masjavas-live` di node mana pun; TIDAK menyentuh
  `live_supervisor.py`, `live_stream_engine.py`, `live_broadcaster.py`.
- **Fase B — Pemilih UI + assignment saat create.** Modal pemilih VPS di halaman
  live; `assigned_node_id` wajib terisi di semua jalur pembuatan task (manual +
  otonom); default Otomatis.
- **Fase C — Eksekusi lintas node + failover manual.** Supervisor node tujuan
  menjalankan task; Master memantau via lease/`node_status`; endpoint assign +
  penawaran failover saat node mati.
- **Fase D — Dashboard `/nodes` + alert.** Semua node + skor + stream-nya dalam
  satu halaman; alert bila node OFFLINE atau skor < 50.

## 8. Prasyarat sebelum migrasi 3 relay ke VPS 8 (di luar Fase A)

1. Dekripsi OAuth di VPS 8 diperbaiki (kunci tidak sinkron — 181× gagal).
2. Kapasitas VPS 8 dilonggarkan (load 6–7/8; prep berbagi box dengan live).
3. Mekanisme migrasi task yang sedang jalan (Fase C) — belum ada.
