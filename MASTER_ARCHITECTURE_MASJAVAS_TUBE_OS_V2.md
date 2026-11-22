# MASTER ARCHITECTURE — MASJAVAS TUBE OS V2.0

**Status:** DRAFT BASELINE — menunggu audit schema, verifikasi biaya, dan persetujuan implementasi
**Tanggal:** 22 November 2026
**Arsitektur target:** Two-node + API-based orchestration + local resource governor + independent backup
**Prinsip utama:** Don't design features. Design systems.

---

## CATATAN VERIFIKASI MUSE (22 Nov 2026, 05:0x WIB)

Dokumen ini dibekukan sebagai Draft Baseline V2.0 dari naskah yang disampaikan user 22 Nov 05:00 WIB,
yang menggabungkan `PLAN_PENAMBAHAN_VPS8_2026-11-22.md`, `DAFTAR_REPO_GITHUB_RUJUKAN.md`,
dan review arsitektur Staff-level beserta koreksi operasional.

**Tiga pekerjaan pra-implementasi — status:**

1. **Isi asli `RENCANA_ORCHESTRATOR_2026-11-22.md` — SELESAI (gap ditutup).**
   File ternyata ADA di `workspace/engineering/RENCANA_ORCHESTRATOR_2026-11-22.md`
   (11.498 byte, 22 Nov 04:02 WIB) dan sudah dibaca penuh oleh Muse. Rekonstruksi G1–G12
   + 8 aturan main dalam dokumen ini TERVERIFIKASI AKURAT terhadap file aslinya:
   G1 load gate, G2 shed flag asimetris (112×/24 jam log-only), G3 RAM gate ≥4 GB
   (tersedia 3,2 GB), G4 disk gate ≥40 GB (39–41 GB saat puncak wave), G5 broadcast
   window 17:00–22:30, G6 Stage 2 yield + STUDIO_RENDER_BUSY, G7 semaphore render 2+1,
   G8 sweeper batch 2/120 dtk, G9 churn SUPERSEDED 66% (333 vs 123 COMPLETED/7 hari),
   G10 lock SQLite 31×/24 jam, G11 kuota Drive/YouTube, G12 satu event loop.
   Audit modal terverifikasi: ingestion CPU 0,0% (1.510 tick), Stage 2 12,8 dtk CPU/klip
   vs wall 100,1 dtk/klip (I/O dominan), 1.011 DONE/7 hari.
2. **Audit schema database aktual (`operational_tasks` dkk.) — BELUM.**
   Catatan awal dari memori: `operational_tasks` tidak punya kolom `slot_number`
   (pakai `channel_id`); tidak ada tabel `fleet_runs`/`job_queues` (state wave di
   `background_jobs`); tidak ada kolom `audio_progress` (progres di `result_json` +
   `ingestion_items`). Audit wajib sebelum migration apa pun.
3. **Angka biaya storage + harga VPS dari panel aktual — BELUM.**
   Harga Master ±$14,43 pra-pajak masih estimasi; total $34,67/bln belum terverifikasi
   di panel Contabo. Biaya object storage (Litestream/Restic) belum dihitung.

**Koreksi minor terhadap naskah (dicatat, isi naskah tidak diubah):**

- Modal ingestion per orchestrator §4.3: **1 core / 2 GB RAM / 100 GB disk**.
  Angka "≥4 GB" di plan adalah AMBANG GATE (RAM bebas minimum agar konkurensi naik),
  bukan konsumsi aktual. V2 §4.4 sudah membedakan ketiganya dengan benar.
- Antrean Stage 2: **50 QUEUED + 4 PENDING item** (orchestrator §2), bukan hanya 50.
- Resource Guard v2 SUDAH LIVE di produksi sejak 20 Nov 23:02 (lane Antigravity,
  commit `5b8279a` + `2e61dcc`) — governor lokal V2 harus dibangun DI ATAS implementasi
  ini, bukan dari nol. Audit kode guard v2 termasuk dalam Track A.
- KEIN: keputusan top-up DITUNDA user sejak 12 Nov ("diputuskan NANTI, setelah
  arsitektur siap"). Plan berakhir 23 Nov 2026 — fallback Tahap 1/2 adalah P0 hari ini.

**Dokumen sumber (disimpan sebagai historis, bukan dihapus):**

- `PLAN_PENAMBAHAN_VPS8_2026-11-22.md` (plan of record sebelumnya)
- `RENCANA_ORCHESTRATOR_2026-11-22.md` (doktrin + G1–G12 + 8 aturan main)
- `RANCANGAN_INFRASTRUKTUR_2_VPS_2026-11-22.md` (analisis opsi + Lampiran A/B)
- `DAFTAR_REPO_GITHUB_RUJUKAN.md` + `github-research/` (50 repo terverifikasi)
- `EVALUASI_SEBELUM_PERBAIKAN_2026-11-22.md` (register P1–P16)
- `VERIFIKASI_CPU_INGESTION_LOOPGEN_LIVE_2026-11-22.md` (peta CPU + insiden live)

---

# 0. Tujuan dan status dokumen

Dokumen ini menjadi acuan arsitektur induk MASJAVAS TUBE OS. Dokumen ini menyatukan:

1. `PLAN_PENAMBAHAN_VPS8_2026-11-22.md`
2. Rencana orchestrator dan temuan G1–G12 yang dirujuk dalam plan VPS 8
3. `DAFTAR_REPO_GITHUB_RUJUKAN.md` — 50 repository utama, catatan tambahan, dan lead susulan
4. Review arsitektur Staff-level beserta koreksi dan tambahan keputusan operasional 22 November 2026

Dokumen ini bukan instruksi untuk langsung mengubah production. Semua perubahan wajib melewati audit, rencana verifikasi, persetujuan, implementasi bertahap, dan rollback.

### Status keputusan

| Area                           | Status                                            |
| ------------------------------ | ------------------------------------------------- |
| Two-node architecture          | Direkomendasikan                                  |
| SQLite tetap di Master         | Direkomendasikan untuk tahap awal                 |
| API-based worker orchestration | Direkomendasikan                                  |
| Local resource governor        | Wajib                                             |
| Job deduplication + generation | Wajib dirancang                                   |
| Lease + fencing + idempotency  | Wajib sebelum distributed execution               |
| Deployment compatibility       | Wajib sebelum multi-worker                        |
| Independent backup             | Wajib sebelum VPS 8 menjadi bagian penting sistem |
| PostgreSQL / Redis             | Ditunda sampai ada bukti kebutuhan                |
| Pembelian VPS 8                | Menunggu demand, benchmark, dan verifikasi harga  |
| Litestream / Restic            | Menunggu keputusan storage dan biaya              |
| Prometheus / Grafana           | Bertahap; bukan prasyarat awal                    |
| VPS ketiga                     | Belum diperlukan                                  |

---

# 1. Reframed Problem

MASJAVAS bukan sekadar aplikasi FastAPI yang menjalankan beberapa background task. MASJAVAS adalah platform orkestrasi workload untuk banyak channel YouTube dengan tiga domain pekerjaan yang memiliki karakteristik sumber daya dan failure domain berbeda:

1. **PUBLISH** — download, pemrosesan, rendering, upload, dan penjadwalan.
2. **PREPARATION / CONTENT SUPPLY** — ingestion, LoopGen, Stage 1/2, Stage 2, pengisian bahan, dan sinkronisasi Drive.
3. **LIVE STREAMING** — supervisor, encoder FFmpeg, RTMP, status broadcast, dan pemulihan stream.

Masalah utama bukan hanya kekurangan CPU. Masalahnya adalah isolasi sumber daya, koordinasi kepemilikan pekerjaan, kontensi database, network saturation, job churn, dan kegagalan yang tidak selalu terdeteksi.

Sistem harus dapat menambah worker tanpa membuat worker lain kehilangan kapasitas. Worker yang mati tidak boleh menyebabkan pekerjaan selesai hilang atau dikerjakan secara tidak terkendali oleh dua worker.

**Target arsitektur:**

- Satu control plane sebagai pemegang kebenaran operasional.
- Banyak worker yang mengambil pekerjaan sesuai kapasitas.
- Setiap workload memiliki resource boundary.
- Live yang sedang berjalan tidak bergantung pada ketersediaan Master.
- Semua job dapat diaudit, dipulihkan, dan direkonsiliasi.
- Backup dapat dipulihkan meskipun Master dan VPS 8 mengalami kegagalan.

---

# 2. Kondisi Awal dan Baseline

Data berikut berasal dari pengukuran dan dokumen plan 21–22 November 2026. Angka ini adalah baseline historis, bukan jaminan kondisi saat implementasi.

| Metrik                          | Baseline                          |
| ------------------------------- | --------------------------------- |
| Publish 21 November             | 206                               |
| FAILED/BLOCKED                  | 0                                 |
| Median keterlambatan publish    | 11,6 menit                        |
| P90 keterlambatan publish       | 30,2 menit                        |
| Encoder tidak direncanakan mati | 5 kali/24 jam pada slot 706       |
| Encoder orphan                  | 1                                 |
| SQLite `database is locked`     | 31 kali/24 jam                    |
| Load Master rata-rata           | 3,5–4,2                           |
| CPU unit web                    | 1,02 core                         |
| Ingestion RUNNING saat backlog  | 2 job                             |
| Churn SUPERSEDED ingestion      | 66% dalam 7 hari                  |
| Disk Master saat puncak wave    | 72%                               |
| Healthz                         | sekitar 10 ms                     |
| Live aktif                      | 2 stream                          |
| Backlog ingestion               | 412 channel / sekitar 28.461 item |
| LoopGen Stage 2 DONE            | 1.011 dalam 7 hari                |
| LoopGen Stage 2 QUEUED          | 50                                |

### Temuan penyebab

- Web menjalankan pekerjaan FFmpeg berat in-process.
- Ingestion dan render saling memengaruhi melalui guard global.
- Governor meminta render mengalah, tetapi sebagian tindakan hanya tercatat sebagai log.
- Sweeper, Fleet Runner, dan retry dapat membuat job baru ketika job lama masih berjalan.
- SQLite menerima banyak penulisan dari beberapa aktivitas.
- Live health check dapat terganggu oleh database lock.
- Encoder dapat mati dan meninggalkan proses orphan.
- Live bandwidth lebih penting daripada CPU untuk ekspansi jumlah stream.

**Doktrin:** perbaikan yang menurunkan keberhasilan publish, meningkatkan error, atau membuat data hilang adalah perbaikan yang gagal.

---

# 3. Requirements

## 3.1 Functional requirements

1. Menjalankan publish, preparation, dan live secara terpisah.
2. Mendaftarkan worker dan kapabilitasnya.
3. Menugaskan job berdasarkan tipe, kapasitas, dan kompatibilitas worker.
4. Mencegah job aktif ganda untuk identitas pekerjaan yang sama.
5. Mendeteksi worker mati, heartbeat terlambat, dan lease kedaluwarsa.
6. Memulihkan job yang ditinggalkan worker.
7. Mencatat seluruh perubahan status job.
8. Mencegah output worker lama menimpa hasil worker baru.
9. Menyediakan dashboard status worker, job, live, kapasitas, dan error.
10. Menyediakan backup dan prosedur restore yang diuji.
11. Memastikan Master dapat berpindah server tanpa mengubah identitas logis sistem.
12. Memungkinkan live yang sudah berjalan tetap hidup saat Master tidak tersedia.

## 3.2 Non-functional requirements

| Aspek         | Target / prinsip                                                                     |
| ------------- | ------------------------------------------------------------------------------------ |
| Availability  | Live yang sudah aktif tetap berjalan saat control plane terputus                     |
| Consistency   | Strong ownership untuk claim dan assignment                                          |
| Job delivery  | At-least-once dengan idempotent execution                                            |
| Latency       | API health p95 <100 ms sebagai target awal                                           |
| Recovery      | RPO target ≤5 menit dan RTO target ≤30 menit, wajib dibuktikan melalui restore drill |
| Security      | Worker API hanya melalui jaringan privat                                             |
| Isolation     | Live mendapat prioritas tertinggi pada Live-01                                       |
| Scalability   | Tambah worker tanpa memindahkan database                                             |
| Auditability  | Semua transisi job dan keputusan scheduler dapat ditelusuri                          |
| Compatibility | Worker hanya menerima job yang kompatibel dengan release dan schema                  |
| Cost          | Tidak melewati anggaran bulanan yang disetujui tanpa keputusan baru                  |

---

# 4. System Scale dan Capacity Model

## 4.1 Infrastruktur awal

**Master — Contabo Plus 4**

- 4 vCPU
- 8 GB RAM
- 145 GB disk
- Port 500 Mbit/s
- FastAPI, control plane, SQLite, publish worker, dan sementara preparation worker

**Live-01 — rencana Contabo Core VPS 8**

- 8 vCPU
- 24 GB RAM
- 300 GB disk
- Port 600 Mbit/s
- Live supervisor, live FFmpeg, dan preparation worker

Spesifikasi dan harga aktual wajib diperiksa ulang di panel checkout sebelum pembelian.

## 4.2 Capacity budget bukan alokasi final

Estimasi plan lama:

| Komponen VPS 8  | Estimasi                                 |
| --------------- | ---------------------------------------- |
| Live            | sekitar 2 core                           |
| Stage 2         | maksimum 4 core                          |
| Ingestion       | sangat kecil berdasarkan pengukuran awal |
| DR dan watchdog | sangat kecil                             |
| Total estimasi  | sekitar 6 core                           |

Angka tersebut bukan hasil benchmark kapasitas akhir.

CPU Stage 2 yang terukur sekitar 12,8 detik CPU per klip dengan wall time sekitar 100,1 detik menunjukkan bahwa I/O dan jaringan ikut menentukan throughput. Karena itu, CPUQuota tidak boleh ditetapkan hanya berdasarkan pembagian angka yang tampak rapi.

**Aturan:** mulai Stage 2 dengan dua slot canary. Naikkan menjadi tiga atau empat hanya jika pengukuran CPU, RAM, I/O, disk, dan live TX membuktikan aman.

## 4.3 Network budget

Port VPS 8: 600 Mbps.

- Batas admission awal: 70% = 420 Mbps.
- Prep harus mengalah terhadap kebutuhan live.
- Upload Drive dijeda atau diperlambat ketika live TX melewati ambang yang ditentukan.
- Backup tidak boleh mengganggu live.
- Gunakan pengukuran TX aktual per interface, bukan hanya metadata bitrate stream.

Estimasi 50 stream membutuhkan sekitar 310 Mbps egress adalah asumsi kapasitas awal. Harus divalidasi menggunakan bitrate aktual dan headroom jaringan.

## 4.4 Disk budget wajib direkonsiliasi

Perkiraan kebutuhan lama:

- Ingestion: sekitar 100 GB.
- Stage 2: sekitar 50 GB.
- Live: sekitar 5 GB.
- DR replica: sekitar 100 GB.

Total kebutuhan sekitar 255 GB, sebelum OS, log, cache, dan ruang bebas.

Dengan disk 300 GB, ruang bebas minimum 50 GB tidak otomatis tersedia. Maka perlu:

1. Audit volume aktual tiap direktori.
2. Pisahkan data permanen, staging, cache, dan replica.
3. Tetapkan quota per workload.
4. Hitung ulang kebutuhan DR.
5. Jangan menganggap angka disk 155 GB sebagai final sebelum definisi setiap komponen disepakati.

RAM juga perlu dibedakan antara konsumsi aktual, batas cgroup, dan syarat admission. Angka "RAM tersedia minimal 4 GB" bukan berarti worker selalu menggunakan 4 GB.

---

# 5. High-Level Architecture

```
                         EXTERNAL MONITORING
                      Healthchecks.io / UptimeRobot
                                |
                                |
                       Telegram Alert Channel
                                |
                                v
+----------------------------------------------------+
|                     MASTER                         |
|                                                    |
|  FastAPI / Control Plane                           |
|  - Worker Registry                                 |
|  - Job API                                         |
|  - Scheduler / Dispatcher                          |
|  - Lease + Fencing                                 |
|  - Deployment Compatibility                        |
|  - Operational Dashboard                           |
|                                                    |
|  SQLite (single authoritative database)           |
|                                                    |
|  Publish Worker                                    |
|  Preparation Worker (temporary / fallback)         |
|                                                    |
|  systemd + cgroups + local resource governor       |
+------------------------+---------------------------+
                         |
                   Tailscale / Private API
                         |
             +-----------+------------+
             |                        |
             v                        v
+---------------------------+   +--------------------+
|          LIVE-01           |   | Future Workers     |
|                           |   |                    |
| Live Supervisor           |   | Worker Agent       |
| FFmpeg / RTMP             |   | Capability Report  |
| Local Live Governor       |   | Pull Job API       |
|                           |   |                    |
| Preparation Worker        |   |                    |
| - Ingestion               |   +--------------------+
| - LoopGen Stage 2         |
|                           |
| Local spool / artifacts   |
| Node Exporter (later)     |
+---------------------------+

Independent Backup:
Master -> encrypted off-site object storage
Master -> optional Litestream replication
Live-01 is NOT the only backup destination.
```

### Prinsip komunikasi

- SQLite tidak pernah dibuka melalui jaringan.
- Worker remote berkomunikasi melalui API.
- Drive tetap menjadi media serah-terima bahan.
- Live RTMP menuju YouTube langsung dari Live-01.
- Master tidak menjadi proxy media live.
- Worker mengambil pekerjaan secara pull-based.
- Live yang sedang berjalan tidak memerlukan Master untuk mempertahankan proses encoder lokalnya.

---

# 6. Architecture Options

## Option A — Custom REST API + SQLite

Master menyediakan endpoint register, claim, heartbeat, complete, fail, dan status.

**Kelebihan:**

- Tidak perlu Redis.
- SQLite tetap menjadi source of truth.
- Integrasi paling dekat dengan aplikasi yang sudah ada.
- Mudah menambahkan fencing, generation, dan audit event.

**Kekurangan:**

- Mekanisme queue, retry, lease, dan scheduler perlu dirawat sendiri.
- SQLite tetap single-writer.
- Perlu disiplin transaksi dan concurrency control.

## Option B — Huey

Huey memiliki backend SQLite dan implementasi task queue yang relatif ringan.

**Kelebihan:**

- Lebih sedikit kode antrean yang perlu dibangun.
- Scheduling dan retry tersedia.
- Tidak harus menambah Redis pada tahap awal.

**Kekurangan:**

- Perlu audit apakah model queue-nya sesuai dengan schema dan job lifecycle yang sudah ada.
- Jangan sampai menjadi sistem antrean kedua di samping `operational_tasks`.
- Integrasi fencing, generation, dan compatibility tetap harus dirancang.

## Option C — Redis + RQ

**Kelebihan:**

- Queue lebih matang.
- Named queue dan prioritas memudahkan pemisahan lane.
- Dapat berkembang ketika throughput meningkat.

**Kekurangan:**

- Infrastruktur dan operasional tambahan.
- Redis menjadi dependency baru.
- Perlu pengamanan, persistence, monitoring, dan prosedur recovery.
- Belum tentu menyelesaikan bottleneck SQLite pada data operasional utama.

### Keputusan

Gunakan **Option A untuk tahap pertama**, setelah audit schema. Huey menjadi alternatif yang diuji di lingkungan non-production jika custom queue terlalu mahal dirawat. Redis + RQ ditunda sampai ada metrik yang membuktikan kebutuhan.

Tidak boleh ada dua authoritative job queue.

---

# 7. Worker Registration dan Capability Model

Setiap worker harus mendaftar ke Master.

Data minimum:

```
worker_id
worker_type
hostname
release_id
schema_version
protocol_version
capabilities
capacity
status
last_heartbeat_at
registered_at
```

Contoh capability:

```
{
  "worker_type": "prep",
  "capabilities": [
    "ingestion",
    "loopgen_stage2"
  ],
  "max_concurrency": 2,
  "release_id": "2026.11.22-a",
  "protocol_version": 1
}
```

Capacity report hanya menjadi informasi scheduler. Worker wajib tetap memeriksa kondisi lokal sebelum menerima job.

Master tidak boleh mengirim job hanya karena status worker `READY`. Admission harus mempertimbangkan CPU, RAM, disk, network, dan local governor.

---

# 8. Job Lifecycle dan Orchestration

## 8.1 State machine

```
stateDiagram-v2
    [*] --> QUEUED
    QUEUED --> CLAIMED
    CLAIMED --> RUNNING
    CLAIMED --> RETRY_WAIT: claim expired
    RUNNING --> COMPLETED
    RUNNING --> FAILED
    RUNNING --> RETRY_WAIT
    RUNNING --> CANCEL_REQUESTED
    CANCEL_REQUESTED --> CANCELLED
    RETRY_WAIT --> QUEUED
    FAILED --> DEAD_LETTER
    COMPLETED --> [*]
    CANCELLED --> [*]
    DEAD_LETTER --> [*]
```

Definisi:

- `QUEUED`: siap diambil.
- `CLAIMED`: sudah dimiliki worker, belum mulai.
- `RUNNING`: sedang diproses.
- `COMPLETED`: hasil tervalidasi dan diterima.
- `FAILED`: gagal dan menunggu klasifikasi/retry.
- `RETRY_WAIT`: menunggu waktu retry.
- `CANCEL_REQUESTED`: pembatalan diminta.
- `CANCELLED`: pembatalan selesai.
- `DEAD_LETTER`: gagal permanen atau melewati batas retry.

## 8.2 Deduplication key dan generation

Gunakan identitas logis:

```
deduplication_key =
    channel_id + job_kind + generation
```

`generation` membedakan pekerjaan baru yang memang diperlukan dari pekerjaan lama yang masih berjalan.

Contoh:

```
channel=123
job_kind=INGESTION
generation=7
```

Aturan:

1. Satu job aktif untuk satu deduplication key.
2. Request job berulang tidak membuat pekerjaan baru jika job aktif yang sama sudah ada.
3. Perubahan input yang memang mengharuskan eksekusi ulang menaikkan generation.
4. Unique constraint harus mencegah dua job aktif untuk key yang sama.
5. Job lama tidak boleh menimpa hasil generation terbaru.

Jangan menganggap setiap retry sebagai generation baru. Retry tetap bagian dari logical job yang sama.

## 8.3 Lease, heartbeat, dan fencing

Lease menentukan sampai kapan worker berhak memegang pekerjaan.

Fencing token adalah nomor monotonik yang meningkat setiap kali ownership berpindah.

Misalnya:

```
Worker A claims job -> fencing_token 41
Lease Worker A expired
Worker B claims job -> fencing_token 42
Worker A returns late with token 41
Master rejects stale result
```

Heartbeat tidak memperpanjang hak worker lama jika token ownership sudah berubah.

Fencing harus divalidasi ketika worker mengirim hasil, bukan hanya saat claim.

## 8.4 Idempotency

Setiap operasi yang berpotensi menghasilkan side effect harus mempunyai idempotency key.

Contoh:

```
channel_id + job_id + operation_type
```

Untuk hasil media:

1. Worker menulis ke direktori temporary unik.
2. Worker menjalankan validasi.
3. Worker mengirim metadata dan hasil ke Master.
4. Master memeriksa fencing token dan generation.
5. Hasil diterima secara atomik.
6. File dipindahkan ke lokasi final.
7. Hasil lama masuk quarantine atau cleanup sesuai aturan retensi.

Upload ke YouTube adalah external side effect. Fencing tidak dapat membatalkan upload yang sudah diterima YouTube. Karena itu, upload harus mempunyai operation record dan reconciliation flow tersendiri.

## 8.5 Retry policy

Retry hanya untuk error yang memang dapat pulih.

| Kategori                         | Tindakan                                                   |
| -------------------------------- | ---------------------------------------------------------- |
| Network timeout                  | Retry dengan exponential backoff + jitter                  |
| HTTP 5xx transient               | Retry terbatas                                             |
| Temporary storage failure        | Retry setelah health check                                 |
| Worker crash                     | Lease recovery dan retry                                   |
| YouTube quotaExceeded            | Jangan retry otomatis; tunggu reset atau tindakan operator |
| Permission / invalid credentials | Pause dan eskalasi                                         |
| Invalid input                    | Permanent failure                                          |
| Unsupported release              | Jangan claim job                                           |
| Rate limit                       | Hormati retry-after dan budget provider                    |

Tenacity dapat digunakan untuk retry transient yang sudah diklasifikasikan. Jangan membungkus seluruh error YouTube dengan retry generik.

---

# 9. Database Design dan Migration Strategy

## 9.1 Prinsip utama

Database saat ini sudah mempunyai tabel operasional seperti:

- `operational_tasks`
- `channels`
- `auth_tokens`
- tabel terkait publish, live, dan konfigurasi lainnya

**Dilarang membuat sistem job paralel sebelum audit schema.**

Langkah pertama:

1. Dump schema tanpa data sensitif.
2. Petakan seluruh tabel job.
3. Petakan status dan transisi yang sudah ada.
4. Cari fungsi yang membuat job baru.
5. Cari sumber churn SUPERSEDED.
6. Cari seluruh penulis status job.
7. Cari indeks dan constraint yang tersedia.
8. Susun migration plan kompatibel dengan aplikasi lama.

## 9.2 Logical data model

Model berikut adalah target konseptual. Nama tabel dan kolom final menunggu audit schema.

### Worker registry

```
workers
-------
worker_id
worker_type
hostname
release_id
schema_version
protocol_version
capabilities_json
capacity_json
status
last_heartbeat_at
created_at
updated_at
```

### Job ownership

Jika `operational_tasks` sudah menjadi tabel utama, tambahkan atribut yang diperlukan ke tabel itu secara inkremental.

Konsep field:

```
deduplication_key
generation
worker_id
lease_expires_at
fencing_token
attempt_count
max_attempts
next_retry_at
idempotency_key
release_constraint
```

### Job events

```
job_events
----------
id
job_id
event_type
from_status
to_status
worker_id
fencing_token
reason_code
metadata_json
created_at
```

Event penting:

- JOB_CREATED
- JOB_DEDUPLICATED
- JOB_CLAIMED
- JOB_STARTED
- HEARTBEAT_RECEIVED
- LEASE_EXPIRED
- FENCING_REJECTED
- JOB_COMPLETED
- JOB_FAILED
- RETRY_SCHEDULED
- DEAD_LETTERED
- CANCEL_REQUESTED
- JOB_CANCELLED

### Worker metrics

Metrik periodik dapat disimpan dalam tabel ringkas atau sistem metrics terpisah. Jangan menulis setiap sampel CPU ke SQLite terlalu sering.

## 9.3 SQLite operational rules

- WAL mode.
- Transaksi pendek.
- Jangan menjalankan HTTP, Google API, atau proses FFmpeg di dalam transaksi.
- Claim job dilakukan dengan transaksi atomik.
- Hindari heartbeat yang menulis terlalu sering.
- Batch event yang aman.
- Indeks sesuai query claim dan lease.
- Monitor busy timeout dan lock frequency.
- Backup harus menggunakan mekanisme konsisten untuk SQLite aktif.

## 9.4 PostgreSQL migration trigger

Pertimbangkan PostgreSQL bila salah satu kondisi berikut terus terjadi setelah optimasi:

- `database is locked` tetap >10 kali/hari.
- P95 operasi tulis >100 ms secara konsisten.
- Claim job mulai menjadi bottleneck.
- Banyak worker menulis bersamaan dan menyebabkan antrean transaksi.
- Kebutuhan multi-region atau query operasional berkembang melampaui SQLite.

Angka ini adalah trigger evaluasi, bukan migrasi otomatis.

---

# 10. Resource Isolation dan Local Governor

Pemisahan proses tidak cukup. Setiap workload harus mendapat resource boundary.

## 10.1 Master

Kelompok layanan:

- `masjavas-os-web`
- `masjavas-os-worker-publish`
- `masjavas-os-worker-prep`
- `masjavas-os-control`
- `masjavas-os-janitor`

## 10.2 Live-01

Kelompok layanan:

- `masjavas-live`
- `masjavas-prep-worker`
- `masjavas-node-exporter` (tahap monitoring)
- `masjavas-backup-agent`

## 10.3 systemd resource controls

Gunakan sesuai hasil benchmark:

- `CPUQuota`
- `CPUWeight`
- `MemoryHigh`
- `MemoryMax`
- `IOWeight`
- `TasksMax`
- `LimitNOFILE`

Nilai final harus ditentukan melalui pengujian, bukan menyalin angka contoh secara membabi buta.

## 10.4 Priority policy

Urutan prioritas Live-01:

1. Live stream yang sudah berjalan.
2. Live supervisor dan recovery.
3. Control/health agent lokal.
4. Ingestion.
5. LoopGen Stage 2.
6. Backup dan pekerjaan non-urgent.

Jika live terancam:

- hentikan admission job baru;
- turunkan concurrency prep;
- throttle upload;
- hentikan Stage 2 jika perlu;
- jangan membunuh live yang sehat hanya karena prep membutuhkan resource.

## 10.5 Disk governor

- Sisakan ruang bebas minimum yang telah divalidasi.
- Setiap workload mempunyai batas staging.
- Cache dapat dibersihkan berdasarkan lifecycle dan retention.
- Janitor wajib memeriksa referensi database dan ownership.
- File yang meragukan masuk quarantine, bukan langsung dihapus.

## 10.6 Network governor

- Ukur TX aktual.
- Gunakan `tc` jika shaping diperlukan.
- Live mempunyai prioritas.
- Upload Drive dan backup dapat ditunda.
- Admission stream baru memakai proyeksi bitrate aktual + headroom.

---

# 11. Artifact Lifecycle dan Janitor Safety

Lifecycle:

```
TEMPORARY
    |
PROCESSING
    |
VALIDATED
    |
PUBLISHED
    |
ARCHIVED
    |
EXPIRED
```

Tambahkan status:

```
QUARANTINED
```

Janitor tidak boleh menghapus file hanya karena metadata pasangan tidak ditemukan.

Sebelum penghapusan, periksa:

1. Owner file.
2. Status job.
3. Referensi database.
4. Generation.
5. Umur file.
6. Status lifecycle.
7. Apakah file sedang digunakan proses.
8. Apakah ada hasil valid yang masih membutuhkan file tersebut.

Prosedur:

- Kandidat cleanup masuk quarantine.
- Catat alasan dan metadata.
- Tunggu grace period.
- Lakukan pemeriksaan ulang.
- Hapus hanya jika seluruh syarat terpenuhi.
- Catat hasil cleanup.

Target penting: tidak ada file channel CONNECTED yang terhapus karena salah diklasifikasikan sebagai unpaired.

---

# 12. Deployment Manager dan Version Compatibility

Ini merupakan komponen wajib sebelum worker tersebar di beberapa VPS.

Setiap deployment memiliki:

```
release_id
git_commit
schema_version
worker_protocol_version
config_version
created_at
```

Worker melaporkan identitas versi saat register dan heartbeat.

Master hanya memberi job jika:

- protocol version kompatibel;
- schema version kompatibel;
- capability tersedia;
- release memenuhi constraint job;
- worker tidak berada dalam maintenance mode.

## Deployment policy

1. Build dan test release.
2. Jalankan migration yang backward-compatible.
3. Deploy Master terlebih dahulu jika API baru tetap kompatibel dengan worker lama.
4. Deploy satu worker canary.
5. Jalankan smoke test.
6. Perluas rollout.
7. Hentikan pemberian job ke worker lama.
8. Drain job aktif sebelum restart.
9. Rollback jika acceptance gate gagal.

Gunakan expand-and-contract migration. Jangan menghapus kolom lama sebelum seluruh worker dan release menggunakan skema baru.

---

# 13. Monitoring dan Observability

## 13.1 Tahap awal: ringan dan eksternal

Sebelum memasang stack monitoring lengkap:

- External uptime monitor memeriksa Master dan Live-01.
- Healthchecks.io cloud memantau heartbeat job terjadwal.
- Telegram menerima alert.
- Worker mengirim heartbeat.
- Master mencatat job progress dan error.
- Business health endpoint membedakan HTTP sehat dari pekerjaan sehat.

Self-hosted Healthchecks tidak menjadi pilihan awal karena menambah Django dan database pada VPS yang sedang dibatasi resource.

## 13.2 Tahap berikutnya: Node Exporter + Prometheus

Node Exporter sendiri hanya mengekspor metrik. Ia membutuhkan Prometheus atau collector lain untuk mengambil dan menyimpan data.

Tahap monitoring lengkap:

- Node Exporter pada Master dan Live-01.
- Prometheus sebagai scraper.
- Grafana bila dashboard historis diperlukan.
- Alertmanager atau mekanisme alert yang teruji.
- Telegram sebagai kanal notifikasi.

Prometheus dapat ditempatkan di VPS 8 hanya setelah RAM, disk, retensi, dan failure domain dievaluasi. External watchdog tetap berada di luar kedua VPS.

## 13.3 Business health checks

Jangan hanya memeriksa HTTP 200.

Periksa:

- Publish progress.
- Job yang terlalu lama RUNNING.
- Worker heartbeat.
- Encoder process.
- FFmpeg progress.
- RTMP connection.
- YouTube broadcast state.
- Stream yang selesai tetapi FFmpeg masih hidup.
- Upload yang belum direkonsiliasi.
- Disk staging.
- Live TX.
- SQLite lock frequency.
- Backup terakhir dan hasil restore terakhir.

---

# 14. Backup, DR, dan Storage Economics

## 14.1 Pisahkan tiga hal

1. **Live operational recovery** — restart atau recovery proses live.
2. **Warm DR replica** — salinan yang mempercepat pemulihan operasional.
3. **Independent backup** — salinan terenkripsi dan versioned di luar kedua VPS.

VPS 8 tidak boleh menjadi satu-satunya lokasi backup Master. Jika kedua VPS mengalami masalah pada provider atau account yang sama, salinan di VPS 8 dapat ikut hilang.

## 14.2 Kandidat teknologi

**Litestream**

- Replikasi SQLite WAL.
- Mendukung pemulihan ke titik waktu tertentu sesuai konfigurasi.
- Bukan pengganti backup historis yang terpisah.

**Restic**

- Backup terenkripsi dan versioned.
- Retention dan repository harus dikelola.
- Perlu backend storage yang independen.

## 14.3 Storage cost gate

Sebelum adopsi:

1. Pilih backend object storage.
2. Hitung biaya per GB.
3. Hitung biaya request dan egress.
4. Estimasikan pertumbuhan database dan aset.
5. Tentukan retensi.
6. Hitung biaya Litestream.
7. Hitung biaya Restic.
8. Masukkan ke total anggaran bulanan.

Jangan menganggap object storage Contabo atau layanan lain sudah termasuk dalam anggaran VPS.

## 14.4 Target recovery

Target yang diusulkan:

- RPO ≤5 menit.
- RTO ≤30 menit.
- 7 backup harian.
- 4 backup mingguan.
- Backup bulanan sesuai kebutuhan retensi.

Semua ini target sementara sampai diuji.

Setiap bulan:

- restore database ke lingkungan terisolasi;
- verifikasi integrity;
- verifikasi jumlah record;
- verifikasi file kritis;
- uji aplikasi membaca hasil restore;
- catat waktu pemulihan.

---

# 15. Repository Adoption Strategy

Riset repository berjumlah 50 repo utama ditambah catatan tambahan. Seluruhnya adalah referensi riset; belum berarti layak dipasang di production.

## Priority A — kandidat yang relevan

| Repository    | Fungsi                 | Keputusan                                      |
| ------------- | ---------------------- | ---------------------------------------------- |
| Litestream    | SQLite WAL replication | Evaluasi setelah storage cost gate             |
| Restic        | Encrypted backup       | Evaluasi bersama backend storage               |
| Node Exporter | Host metrics           | Adopsi saat Prometheus siap                    |
| Healthchecks  | Cron/job heartbeat     | Gunakan layanan cloud terlebih dahulu          |
| Huey          | Task queue             | Uji alternatif non-production                  |
| Tenacity      | Retry                  | Gunakan hanya untuk error transient            |
| SlowAPI       | API rate limiting      | Evaluasi endpoint worker                       |
| PyBreaker     | Circuit breaker        | Gunakan pada dependency eksternal yang relevan |

## Priority B — sesuai kebutuhan

| Repository             | Fungsi                 | Keputusan                                  |
| ---------------------- | ---------------------- | ------------------------------------------ |
| MediaMTX               | Media server           | Uji terisolasi untuk live reliability      |
| SRS                    | RTMP/media server      | Alternatif benchmark                       |
| ffmpeg-normalize       | Loudness normalization | Hanya untuk kebutuhan audio                |
| Chromaprint            | Audio fingerprint      | Evaluasi bila perlu verifikasi audio       |
| python-audio-separator | Stem separation        | Hanya jika fitur pemisahan stem dibutuhkan |
| SQLAdmin               | Admin panel            | Opsional, bukan kebutuhan inti             |

## Tidak diadopsi sekarang

- Redis + RQ sebelum throughput membuktikan kebutuhan.
- LiteFS atau rqlite tanpa alasan kuat.
- OvenMediaEngine untuk kebutuhan saat ini.
- SRS tanpa benchmark dibanding MediaMTX.
- Demucs versi repository yang diarsipkan.
- Spleeter yang sudah deprecated.
- `youtube-upload` yang tidak aktif.
- `python-arq` yang berstatus maintenance-only.
- Tool yang melewati YouTube API dengan Playwright atau berisiko melanggar ToS.

## Catatan lisensi dan keamanan

- Periksa lisensi copyleft sebelum integrasi.
- `python-audio-separator` wajib menggunakan versi yang sudah memperbaiki CVE yang disebut dalam riset, minimal 0.47.2.
- Jangan memasang repository hanya berdasarkan stars.
- Setiap adopsi perlu uji kompatibilitas, resource benchmark, security review, dan rollback.

---

# 16. Workstream Mendesak yang Tidak Diselesaikan oleh VPS 8

## 16.1 KEIN — P0

Dokumen plan menyebut masa KEIN berakhir 23 November 2026.

Risiko: LoopGen Tahap 1/2 dapat berhenti jika tidak ada provider atau fallback.

Tindakan segera:

1. Audit semua pemanggilan KEIN.
2. Inventarisasi tahap yang benar-benar bergantung pada KEIN.
3. Siapkan provider alternatif yang sah.
4. Siapkan mode manual atau hold jika fallback belum tervalidasi.
5. Simpan checkpoint agar job dapat dilanjutkan.
6. Gunakan idempotency key agar fallback tidak membuat job ganda.
7. Uji satu channel canary.
8. Catat biaya dan kualitas hasil.
9. Baru aktifkan fallback bertahap.

**Jangan menunggu VPS 8.** Ini masalah dependency provider, bukan kapasitas server.

## 16.2 D1 — Bot-check ingestion

Dahono gateway dilaporkan menolak login dengan HTTP 407 meskipun akun aktif dan kuota tersedia.

Rencana pengujian:

- Eskalasi ke provider.
- Uji proxy residential yang diusulkan secara terbatas.
- Ukur success rate yt-dlp.
- Ukur latency.
- Ukur biaya per GB.
- Uji stabilitas session dan credential.
- Pastikan penggunaan sesuai ketentuan provider dan platform.

VPS 8 tidak otomatis memperbaiki bot-check.

## 16.3 P2 — Google Drive sync quota

Mekanisme kuota belum terpecahkan meskipun enam teori sudah dieliminasi.

Sebelum worker baru menjalankan Drive API dari IP kedua:

1. Catat project dan credential yang dipakai.
2. Catat endpoint dan metode API.
3. Catat request per menit.
4. Catat response code dan quota error.
5. Pisahkan retry dari request normal.
6. Uji satu worker dari IP baru.
7. Bandingkan dengan baseline Master.
8. Terapkan rate limit terpusat per credential/project bila diperlukan.

Jangan berasumsi dua IP berarti kuota bertambah. Kuota dapat melekat pada project atau resource lain.

## 16.4 Encoder slot 706 — P16

Pindah VPS tidak memperbaiki akar masalah encoder.

Wajib:

- identifikasi exit code dan signal;
- simpan stderr FFmpeg;
- simpan process tree;
- simpan input file dan parameter encoder;
- cek resource dan network;
- bedakan encoder mati, RTMP putus, dan broadcast selesai;
- pastikan tidak ada orphan;
- uji recovery;
- tambahkan business health check.

## 16.5 Master interim sekitar 17 Desember

Migrasi Master adalah proyek terpisah.

Sebelum migrasi:

- endpoint Master harus stabil;
- worker menggunakan konfigurasi endpoint, bukan IP hardcoded;
- Tailscale identity dan route diuji;
- database backup dan restore diuji;
- worker dapat reconnect;
- schema dan release compatibility diverifikasi;
- jadwal cutover dan rollback disiapkan.

Jangan menggabungkan migrasi Master, perubahan schema besar, dan perpindahan live worker dalam satu perubahan.

---

# 17. Roadmap Gabungan — Parallel Workstreams

Roadmap lama terlalu berurutan jika seluruh persiapan VPS 8 menunggu seluruh orchestration selesai. Roadmap baru menggunakan jalur paralel dengan gate yang jelas.

## Track A — Stabilization (mulai segera)

**Tujuan:** lindungi operasi yang sudah berjalan.

- A1: diagnosis encoder 706.
- A2: perbaiki orphan process.
- A3: pisahkan web dan worker berat.
- A4: systemd resource isolation.
- A5: perbaiki janitor dan quarantine.
- A6: turunkan job churn.
- A7: baseline scorecard.
- A8: siapkan fallback KEIN.

Gate:

- publish tidak memburuk;
- tidak ada data valid terhapus;
- encoder stabil;
- error baru tidak meningkat;
- resource isolation terukur.

## Track B — VPS 8 Readiness (paralel)

**Tujuan:** siapkan server tanpa mengganggu production.

- B1: verifikasi harga Contabo.
- B2: hitung storage budget.
- B3: verifikasi anggaran total.
- B4: siapkan hardening dan Tailscale.
- B5: desain systemd unit.
- B6: benchmark live dan Stage 2.
- B7: siapkan external watchdog.
- B8: siapkan disk lifecycle dan network governor.

Gate:

- harga sesuai batas anggaran;
- disk budget masuk akal;
- backup independen tersedia;
- resource limits siap;
- rollback plan siap.

## Track C — Orchestration Minimum (bersama canary)

**Tujuan:** remote worker dapat mengambil job dengan aman.

- C1: audit schema.
- C2: tentukan authoritative job table.
- C3: deduplication key + generation.
- C4: worker registration.
- C5: claim + lease.
- C6: heartbeat.
- C7: fencing token.
- C8: idempotency.
- C9: job event audit.
- C10: version compatibility.

Mulai dari satu tipe job dan satu worker canary. Jangan langsung memindahkan seluruh workload.

## Track D — VPS 8 Canary

- D1: deploy Live-01.
- D2: register worker.
- D3: uji API claim.
- D4: jalankan satu job preparation.
- D5: uji local governor.
- D6: uji live process terpisah.
- D7: uji Master disconnect.
- D8: uji lease expiry.
- D9: uji stale worker result.
- D10: uji rollback.

Naikkan concurrency hanya setelah seluruh gate lulus.

## Track E — Observability dan Recovery

- E1: external uptime monitoring.
- E2: Healthchecks cloud.
- E3: Telegram alerts.
- E4: business health checks.
- E5: Node Exporter.
- E6: Prometheus.
- E7: backup object storage.
- E8: restore drill.

## Track F — Scale dan HA

Hanya setelah data menunjukkan kebutuhan:

- PostgreSQL.
- Redis/RQ.
- Live-02.
- Worker tambahan.
- Shard rebalancing.
- Multi-region control plane.
- Automated failover.

---

# 18. Scorecard dan Acceptance Criteria

Baseline mengikuti pengukuran 21–22 November. Window pengukuran harus konsisten. Jangan membandingkan baseline 7 hari dengan hasil 72 jam seolah-olah setara.

| Metrik                  | Baseline           | Target awal              |
| ----------------------- | ------------------ | ------------------------ |
| Publish per hari        | 206                | ≥200                     |
| FAILED/BLOCKED          | 0                  | 0                        |
| P90 publish delay       | 30,2 menit         | ≤30 menit                |
| Encoder unplanned death | 5/24 jam           | ≤1/24 jam setelah P16    |
| Encoder orphan          | 1                  | 0                        |
| SQLite lock             | 31/24 jam          | <20, lalu <10            |
| Master average load     | 3,5–4,2            | <3,0, lalu <2,5          |
| Web CPU                 | 1,02 core          | <0,2 core                |
| Ingestion RUNNING       | 2                  | ≥3 saat backlog tersedia |
| Ingestion churn         | 66%                | <20% dalam window 7 hari |
| Master disk peak        | 72%                | ≤70%                     |
| Healthz p95             | sekitar 10 ms      | <100 ms                  |
| New error classes       | belum ada baseline | 0 selama observasi gate  |

### Gate policy

Setiap perubahan:

1. Pre-change snapshot.
2. Baseline.
3. Implementasi terbatas.
4. Post-change validation.
5. Observasi 72 jam untuk perubahan operasional.
6. Observasi 7 hari untuk metrik yang memang membutuhkan window 7 hari.
7. Rollback bila metrik kritis memburuk.

Tidak semua metrik bisa dinyatakan lulus hanya berdasarkan observasi 72 jam.

---

# 19. Failure Scenarios

| Failure                               | Dampak                                       | Recovery                                                       |
| ------------------------------------- | -------------------------------------------- | -------------------------------------------------------------- |
| Master down                           | Claim dan assignment baru berhenti           | Live berjalan lokal; worker tidak mengambil job baru           |
| Live-01 down                          | Live shard terputus                          | Alarm eksternal; recovery manual/box cadangan sesuai kapasitas |
| Tailscale down                        | Control plane tidak dapat menghubungi worker | Live berjalan; assignment baru ditahan                         |
| SQLite lock                           | Claim/status terlambat                       | Retry transaksi pendek, observability, optimasi writer         |
| Worker crash                          | Job tertinggal                               | Lease expiry dan requeue                                       |
| Duplicate execution                   | Side effect ganda                            | Dedup key + idempotency                                        |
| Stale worker result                   | Hasil lama menimpa baru                      | Fencing token                                                  |
| Disk full                             | Job dan encoder berisiko gagal               | Admission stop, quota, janitor quarantine                      |
| TX saturation                         | Live buffering                               | Stop prep upload, network shaping                              |
| YouTube quotaExceeded                 | API call gagal                               | Pause dan tunggu reset; jangan retry membakar kuota            |
| FFmpeg mati                           | Stream terputus                              | Supervisor recovery dan root-cause telemetry                   |
| Broadcast selesai tetapi FFmpeg hidup | Status sistem tidak konsisten                | Business health check dan reconciliation                       |
| Backup corrupt                        | Recovery gagal                               | Integrity check dan restore drill                              |
| Master migration gagal                | Worker kehilangan control plane              | Endpoint stabil, rollback, reconnect                           |

---

# 20. CAP dan Consistency Decision

MASJAVAS memilih consistency ownership yang kuat pada Master, dengan availability lokal untuk live yang sudah berjalan.

Saat Master tidak tersedia:

- Live yang sudah aktif tetap berjalan.
- Worker tidak menerima job baru.
- Worker tidak boleh membuat ownership baru secara lokal.
- Job yang belum selesai menunggu rekonsiliasi.
- Setelah koneksi pulih, Master memeriksa lease, fencing, dan job events.

Ini berarti sistem tidak berusaha mempertahankan seluruh operasi distributed saat control plane terputus.

Prioritasnya adalah mencegah dua worker mengerjakan side effect yang sama.

---

# 21. Security dan Operational Governance

- API worker hanya melalui Tailscale/private network.
- Setiap worker memiliki identity dan credential sendiri.
- Credential tidak dicatat di log.
- Rotasi token didukung.
- Rate limit per worker.
- Validasi payload dan capability.
- Batasi akses filesystem.
- Pisahkan user systemd untuk workload bila layak.
- Jangan menjalankan service sebagai root tanpa alasan.
- Firewall default deny.
- Simpan audit log.
- Deployment menggunakan release manifest.
- Semua secret dan backup dienkripsi.

---

# 22. Keputusan Arsitektur Final yang Direkomendasikan

| Keputusan            | Rekomendasi                                       |
| -------------------- | ------------------------------------------------- |
| Physical topology    | Master + Live-01                                  |
| Control plane        | FastAPI pada Master                               |
| Database             | SQLite WAL pada Master                            |
| Queue                | API + tabel job existing setelah audit            |
| Worker model         | Pull-based                                        |
| Ownership            | Lease + fencing token                             |
| Duplicate prevention | Dedup key + generation                            |
| Execution            | At-least-once + idempotency                       |
| Audit                | Job events                                        |
| Resource isolation   | systemd cgroups                                   |
| Live priority        | Local governor pada Live-01                       |
| Media handoff        | Google Drive                                      |
| Network              | Tailscale untuk control; RTMP langsung ke YouTube |
| Monitoring awal      | External uptime + Healthchecks cloud              |
| Monitoring lanjutan  | Node Exporter + Prometheus                        |
| Backup               | Independent encrypted off-site backup             |
| SQLite replication   | Litestream setelah storage gate                   |
| Backup tooling       | Restic setelah storage gate                       |
| Queue evolution      | Huey evaluation, Redis/RQ bila diperlukan         |
| Database evolution   | PostgreSQL hanya berdasarkan trigger              |
| Deployment           | Release ID + protocol/schema compatibility        |
| Scale                | Canary, lalu capacity-based expansion             |

---

# 23. Next Steps — Urutan Implementasi

## P0 — 24 jam pertama

1. Audit dan siapkan fallback KEIN sebelum 23 November.
2. Diagnosis encoder 706 dan orphan.
3. Ambil baseline CPU/RAM/disk/TX.
4. Audit janitor dan file yang berpotensi salah hapus.
5. Verifikasi status D1.
6. Siapkan pengukuran P2 Drive quota.
7. Verifikasi harga Master dan VPS 8 melalui panel.

## P1 — Stabilization + preparation

1. Pisahkan publish dan prep.
2. Keluarkan FFmpeg berat dari web.
3. Terapkan systemd resource control.
4. Perbaiki deduplication ingestion.
5. Audit dan perbaiki janitor.
6. Siapkan external watchdog.
7. Audit database schema.

## P2 — Orchestration minimum

1. Tetapkan authoritative job table.
2. Tambahkan dedup key dan generation.
3. Worker registration.
4. Claim + lease + heartbeat.
5. Fencing token.
6. Idempotency.
7. Job events.
8. Version compatibility.

## P3 — VPS 8 canary

1. Harga dan biaya storage disetujui.
2. Provisioning dan hardening.
3. Tailscale.
4. Live worker.
5. Prep worker.
6. Canary satu job.
7. Canary satu live stream.
8. Uji Master disconnect.
9. Uji worker crash.
10. Uji rollback.

## P4 — Recovery dan scale

1. Backup independen.
2. Litestream jika layak.
3. Restic jika layak.
4. Restore drill.
5. Prometheus dan Grafana jika dibutuhkan.
6. Naikkan concurrency berdasarkan benchmark.
7. Evaluasi VPS berikutnya.

---

# 24. Open Decisions — Tidak Boleh Diasumsikan

| Pertanyaan             | Keputusan yang diperlukan                      |
| ---------------------- | ---------------------------------------------- |
| Harga VPS 8            | Harga checkout aktual termasuk pajak dan setup |
| Master interim         | Provider dan tanggal migrasi final             |
| Storage backup         | Provider, kapasitas, biaya, egress             |
| RPO/RTO                | Target yang telah dibuktikan                   |
| KEIN fallback          | Provider dan prosedur yang disetujui           |
| D1                     | Hasil eskalasi dan proxy test                  |
| P2                     | Temuan kuota dan rate limit aktual             |
| Existing job schema    | Audit tabel dan migration plan                 |
| Prometheus             | Lokasi, retensi, dan biaya resource            |
| Stage 2 concurrency    | Hasil benchmark 1, 2, 3, 4 slot                |
| Live failover          | Manual atau otomatis setelah uji               |
| VPS 8 purchase trigger | Demand, capacity, dan budget gate              |

---

# 25. Definition of Done

Arsitektur V2 belum dianggap selesai hanya karena VPS 8 berhasil dinyalakan.

V2 dianggap berhasil ketika:

1. Master tetap menjadi satu-satunya pemegang kebenaran job ownership.
2. Worker dapat register, claim, heartbeat, dan menyelesaikan job.
3. Duplicate execution dapat dicegah atau direkonsiliasi.
4. Stale worker tidak dapat menimpa hasil baru.
5. Restart prep tidak mengganggu publish.
6. Beban prep tidak mengganggu live.
7. Live tetap berjalan saat Master tidak tersedia.
8. Worker yang mati dapat dipulihkan tanpa kehilangan status pekerjaan.
9. Artifact tidak dihapus tanpa verifikasi lifecycle.
10. Deployment lintas versi dapat dikendalikan.
11. Monitoring mendeteksi business failure.
12. Backup dapat dipulihkan dalam target yang telah diuji.
13. Biaya bulanan dan biaya storage diketahui.
14. Scorecard tidak memburuk setelah migrasi.
15. Semua perubahan mempunyai bukti verifikasi dan rollback.

---

# Penutup

MASJAVAS TUBE OS V2 tidak dibangun dengan menambah server sebanyak mungkin. Sistem dibangun dengan memperjelas ownership, membatasi resource, mengendalikan side effect, dan memastikan kegagalan dapat dipulihkan.

Prinsip yang mengikat:

- Tambah worker tanpa mengganggu worker lain.
- Kehilangan satu server tidak boleh menghilangkan seluruh kendali sistem.
- Master putus tidak boleh mematikan live yang sedang berjalan.
- Job gagal tidak boleh membuat hasil ganda.
- File yang belum jelas statusnya tidak boleh dihapus.
- Perubahan versi tidak boleh merusak worker yang masih berjalan.
- Backup yang belum pernah direstore belum dapat dianggap sebagai backup yang terbukti.

**Satu control plane. Banyak worker. Ownership yang jelas. Resource yang terisolasi. Recovery yang teruji.**

**END OF MASTER ARCHITECTURE V2.0 — DRAFT BASELINE**

---

## Catatan validasi dokumen

Dokumen ini sudah menggabungkan inti plan VPS 8 — termasuk pembagian tiga workload, baseline scorecard, resource budget, dan fase awal — serta rekomendasi repository dan koreksi operasional yang disampaikan. Baseline dan prinsip pembagian workload merujuk pada plan VPS 8.

Daftar repository menegaskan Huey, Litestream, RQ, MediaMTX, SRS, Node Exporter, Healthchecks, dan tool pendukung lain sebagai referensi, bukan dependensi yang otomatis harus dipasang. Risiko lisensi, proyek yang diarsipkan, dan batasan keamanan juga perlu dipertahankan dalam proses adopsi.

**Tiga pekerjaan yang harus dilakukan sebelum V2 ini menjadi acuan implementasi final:**

1. ~~Masukkan isi asli `RENCANA_ORCHESTRATOR_2026-11-22.md`~~ — **SELESAI 22 Nov 05:0x WIB.**
   File ditemukan di `workspace/engineering/` dan dibaca penuh; G1–G12 + 8 aturan main
   terverifikasi akurat (lihat Catatan Verifikasi Muse di atas).
2. Audit schema database aktual, khususnya `operational_tasks`, sebelum menyetujui migration.
3. Isi angka biaya storage dan harga VPS dari panel aktual — jangan menganggap estimasi lama sebagai tagihan final.

Rekomendasi: **bekukan dokumen ini sebagai Draft Baseline V2.0, bukan langsung sebagai instruksi deployment.**
Langkah berikutnya yang paling bernilai adalah membuat *implementation checklist* per fase yang menyebut
file kode, tabel database, systemd unit, tes pra/pasca, dan rollback untuk setiap perubahan.
