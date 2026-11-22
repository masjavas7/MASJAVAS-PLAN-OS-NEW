# ORCHESTRATOR API CONTRACT v1 — MASJAVAS TUBE OS V2

**Status:** DRAFT v1.0 — 22 Nov 2026. Kontrak ini mendefinisikan API
sebelum implementasi orchestrator ditulis; ia BUKAN dokumentasi API
yang sudah berjalan. Tidak ada endpoint di bawah ini yang
terimplementasi saat dokumen ini ditulis.
**Sumber:** `MASTER_ARCHITECTURE_MASJAVAS_TUBE_OS_V2.md` §6–§9, §12,
§19–§21, §24 (FROZEN — V2.0); `RENCANA_ORCHESTRATOR_2026-11-22.md`
§4; `REPOSITORY_ENGINEERING_AUDIT_SPRINT.md` item #4 (kriteria
selesai: kasus uji stale-worker terdefinisi sebagai kasus uji wajib).
**Ruang lingkup:** API pull-based REST antara Master (control plane +
SQLite) dan worker remote (prep ingestion, prep Stage 2, live).
Implementasi mengikuti kontrak ini; penyimpangan harus dicatat
sebagai deviasi dan disetujui sebelum deploy.

## Legenda label klaim

- **[VF]** VERIFIED FACT — terverifikasi langsung (DB/log/kode/tes).
- **[DF]** DOCUMENTED FACT NOT YET REVALIDATED — tercatat di dokumen
  sumber (V2.0 / rencana orchestrator / acceptance plan), belum
  divalidasi ulang sejak ditulis.
- **[AS]** ASSUMPTION — asumsi yang dipakai kontrak ini.
- **[PR]** PROPOSAL — isi kontrak v1 ini (aturan baru yang diusulkan,
  belum diimplementasi dan belum diuji).
- **[OQ]** OPEN QUESTION — pertanyaan terbuka, belum ada jawaban.
- **[OD]** OWNER DECISION REQUIRED — butuh keputusan owner.

Aturan label untuk dokumen ini: semua yang dikutip dari V2.0 adalah
[DF]; semua yang didefinisikan BARU oleh kontrak ini (schema persis,
nilai timeout, kode error, format header) adalah [PR] — karena
kontrak ini sendiri adalah usulan, bukan fakta produksi.

---

## 1. Prinsip kontrak (mengikat)

Prinsip-prinsip ini berasal dari V2.0 dan TIDAK BOLEH dilanggar oleh
implementasi mana pun:

1. **Pull, bukan push.** Worker MENGAMBIL job saat punya kapasitas
   (klaim + lease). Master tidak pernah mendorong job ke worker.
   [DF — V2.0 §8; Rencana Orchestrator §4.2 aturan 2]
2. **SQLite hanya di Master.** Worker remote TIDAK PERNAH membuka
   database lewat jaringan; semua akses data lewat API ini.
   [DF — V2.0 §5, §9.1]
3. **Satu job aktif per deduplication key** (`channel_id +
   job_kind + generation`). Retry BUKAN generation baru; job lama
   tidak boleh menimpa hasil generation terbaru.
   [DF — V2.0 §8.2]
4. **Lease + fencing token.** Kepemilikan job ditentukan oleh lease
   dan fencing token monotonik yang naik setiap kali ownership
   berpindah. Fencing divalidasi saat worker mengirim HASIL, bukan
   hanya saat klaim. [DF — V2.0 §8.3]
5. **Idempotency.** Setiap operasi yang berpotensi menghasilkan side
   effect punya idempotency key. Submit ganda dengan key yang sama
   menghasilkan SATU hasil. [DF — V2.0 §8.4]
6. **Hasil media atomik.** Worker menulis ke direktori temporary
   unik → validasi → kirim metadata + hasil → Master memeriksa
   fencing token + generation → terima atomik → pindah ke lokasi
   final. [DF — V2.0 §8.4]
7. **Upload YouTube adalah side effect eksternal.** Fencing TIDAK
   dapat membatalkan upload yang sudah diterima YouTube. Upload
   wajib punya operation record + reconciliation tersendiri.
   [DF — V2.0 §8.4]
8. **Tailscale-only.** API worker hanya dapat dijangkau lewat
   jaringan privat (Tailscale). Tidak ada endpoint worker yang
   terekspos ke internet publik. [DF — V2.0 §21]
9. **Master tidak memberi job hanya karena status READY.** Admission
   mempertimbangkan CPU, RAM, disk, network, dan local governor;
   worker wajib memeriksa kondisi lokal sebelum menerima job.
   [DF — V2.0 §7]
10. **Tidak ada I/O jaringan di dalam transaksi DB.** Claim atomik
    memakai transaksi pendek; HTTP, Google API, dan FFmpeg TIDAK
    BOLEH berjalan di dalam transaksi. [DF — V2.0 §9.3]

---

## 2. Konvensi umum

### 2.1 Alamat dasar dan versi

- Base URL API ditentukan saat provisioning dan hanya dapat dijangkau
  lewat Tailscale. **Alamat dan port persis BELUM ditetapkan**
  [OQ-401] — kontrak ini tidak mengarangnya. Semua path di bawah
  ini relatif terhadap base URL tersebut.
- Versi API ada di path: `/v1/...`. [PR]
- `protocol_version` (integer, saat ini `1`) dilaporkan worker di
  register + heartbeat dan dipakai Master untuk keputusan
  kompatibilitas. [DF — V2.0 §7, §12; nilai awal 1 dari contoh §7]
- Perubahan yang tidak kompatibel (breaking) WAJIB menaikkan versi
  mayor (`/v2/...`) dan `protocol_version`; perubahan yang kompatibel
  (field opsional baru) tidak menaikkan versi. [PR — mengikuti
  kebijakan expand-and-contract V2.0 §12]

### 2.2 Autentikasi

- Setiap worker memiliki identitas dan kredensial sendiri.
  [DF — V2.0 §21]
- [PR] Kredensial dikirim sebagai bearer token di header:
  `Authorization: Bearer <worker-token>`. Token terikat pada satu
  `worker_id`; token worker A tidak dapat dipakai untuk bertindak
  sebagai worker B.
- [PR] Master menolak request tanpa token yang valid dengan `401`
  (`AUTH_REQUIRED` / `AUTH_INVALID`, lihat §6).
- Rotasi token didukung [DF — V2.0 §21]; prosedur penerbitan dan
  rotasi token (siapa menerbitkan, masa berlaku, cara rotasi tanpa
  downtime) [OQ-402] — belum didefinisikan di V2.0 maupun di
  rencana keamanan; harus didefinisikan sebelum implementasi.
- Kredensial TIDAK PERNAH dicatat di log. [DF — V2.0 §21]
- Rate limit per worker diberlakukan [DF — V2.0 §21]; nilai batasnya
  [OQ-403] — harus melampaui lalu lintas normal terburuk (heartbeat
  + klaim + event job) dengan margin besar, karena batas ini menjaga
  dari loop yang tak terkendali, bukan dari lalu lintas normal.

### 2.3 Format envelope

Semua response memakai envelope JSON yang sama [PR]:

```json
{
  "ok": true,
  "data": { "...": "..." },
  "error": null,
  "request_id": "req-01J..."
}
```

```json
{
  "ok": false,
  "data": null,
  "error": {
    "code": "STALE_FENCING_TOKEN",
    "message": "Fencing token 41 sudah tidak berlaku; pemilik saat ini memegang token 42.",
    "retryable": false,
    "retry_after_seconds": null
  },
  "request_id": "req-01J..."
}
```

- `request_id` dibuat Master untuk setiap request dan dipakai untuk
  korelasi log. [PR]
- `error.retryable` menyatakan apakah client BOLEH mencoba lagi;
  `retry_after_seconds` (bila ada) WAJIB dihormati. [PR — mengikuti
  kebijakan retry V2.0 §8.5]

### 2.4 Idempotency key

- Client WAJIB mengirim header `Idempotency-Key` pada operasi yang
  berpotensi side effect: `register`, `claim`, `complete`, `fail`.
  [PR — penerapan V2.0 §8.4]
- Format key yang direkomendasikan: `channel_id + job_id +
  operation_type` untuk operasi job; `worker_id + operation_type +
  client-generated-uuid` untuk operasi worker. [PR — contoh V2.0
  §8.4]
- Master menyimpan hasil operasi per idempotency key dan
  mengembalikan hasil yang SAMA untuk pengulangan key yang sama
  (tanpa menjalankan ulang side effect). Retensi key minimum 24 jam.
  [PR]
- Pengulangan `complete` dengan key yang sama setelah sukses HARUS
  mengembalikan hasil sukses yang sama (bukan error duplikat).
  [PR — diuji di T1.5-H acceptance plan]

### 2.5 Jam dan lease

- Keputusan lease (kedaluwarsa, urutan kepemilikan) memakai jam
  Master. Worker dan Master WAJIB tersinkronisasi waktu (NTP).
  [PR — V2.0 tidak menetapkan sumber jam; penetapan ini mencegah
  perselisihan lease akibat clock skew]
- Semua timestamp dalam kontrak ini memakai UTC, format ISO 8601
  (`2026-11-22T05:00:00Z`). [PR]

---

## 3. Nilai awal (PROPOSAL — validasi di canary)

Semua nilai di bawah ini adalah **[PR]** — nilai awal yang masuk
akal, BUKAN hasil benchmark. Nilai final ditetapkan setelah uji
canary dan V2-008 (benchmark); kontrak TIDAK mengunci angka final.
Alasan tiap nilai dicantumkan agar peninjau dapat menantangnya.

| Parameter | Nilai awal [PR] | Alasan |
|---|---|---|
| Lease job default | 15 menit | Lebih panjang dari celah heartbeat terburuk yang ditoleransi (5× interval), lebih pendek dari durasi job terpanjang yang wajar; heartbeat memperpanjangnya, jadi durasi job tidak dibatasi lease |
| Interval heartbeat worker | 60 detik | Cukup sering untuk deteksi worker mati dalam ~3 menit (3× miss), cukup jarang untuk tidak membebani SQLite (V2.0 §9.3 melarang heartbeat menulis terlalu sering) |
| Interval heartbeat job | 120 detik | Job aktif melaporkan progress batch (Rencana Orchestrator §4.2 aturan 7: progress di-batch, bukan tiap detik) |
| Worker dianggap STALE | 3× heartbeat terlewat (180 dtk) | Konsisten dengan ambang alarm rollback runbook: heartbeat stale >90 dtk memicu alarm [DF]; 180 dtk = ambang "worker hilang", 90 dtk = ambang "peringatan dini" |
| Timeout HTTP request (client) | 30 detik | Melebihi latensi normal API internal (healthz p95 ±10 ms [VF]) dengan margin besar; cukup pendek agar worker tidak menggantung |
| Timeout klaim (server) | 10 detik | Klaim adalah transaksi atomik pendek [DF — V2.0 §9.3]; bila melebihi ini, ada masalah DB |
| Retry API transient | maks 5 percobaan, backoff eksponensial + jitter, base 1 dtk | Mengikuti V2.0 §8.5 (retry terbatas, backoff + jitter); 5 percobaan ≈ 31 dtk total sebelum menyerah |
| Max attempts job (transient) | 3 | Nilai awal konservatif; `max_attempts` adalah field model data [DF — V2.0 §9.2], nilainya belum ditetapkan di V2.0 |
| Retensi idempotency key | 24 jam | Melebihi durasi job terpanjang yang wajar + jendela retry |

Nilai-nilai ini TIDAK BOLEH diubah diam-diam di implementasi;
perubahan nilai dicatat sebagai revisi kontrak (v1.1, dst.).

---

## 4. Model data

### 4.1 Worker

Field minimum dari V2.0 §7 [DF]; format JSON di bawah ini [PR]:

```json
{
  "worker_id": "prep-vps8-01",
  "worker_type": "prep",
  "hostname": "vmi3762689",
  "release_id": "2026.11.22-a",
  "schema_version": 3,
  "protocol_version": 1,
  "capabilities": ["ingestion", "loopgen_stage2"],
  "capacity": {
    "max_concurrency": 2,
    "cpu_cores": 4,
    "memory_mb": 4096,
    "disk_free_mb": 51200
  },
  "status": "READY",
  "last_heartbeat_at": "2026-11-22T05:00:00Z",
  "registered_at": "2026-11-22T04:00:00Z"
}
```

- `worker_type`: `prep` | `live` | `publish` [PR — mengikuti
  pembagian workload V2.0 §5 / Rencana Orchestrator §4.1].
- `capabilities`: daftar kemampuan yang dideklarasikan worker.
  Contoh V2.0 §7: `ingestion`, `loopgen_stage2`. Kosakata
  capability final menunggu audit schema (V2-007) [OQ-404].
- `status`: `READY` | `BUSY` | `DRAINING` | `MAINTENANCE` |
  `STALE` | `OFFLINE` [PR]. `STALE`/`OFFLINE` ditetapkan Master
  (bukan worker) berdasarkan heartbeat yang terlewat.
- `capacity` adalah DEKLARASI worker; Master memakainya sebagai
  informasi scheduler, bukan kebenaran mutlak. Worker wajib
  memeriksa kondisi lokal sebelum menerima job. [DF — V2.0 §7]

### 4.2 Job

```json
{
  "job_id": "job-01J...",
  "deduplication_key": "ch-123:INGESTION:7",
  "channel_id": "ch-123",
  "job_kind": "INGESTION",
  "generation": 7,
  "status": "CLAIMED",
  "worker_id": "prep-vps8-01",
  "fencing_token": 42,
  "lease_expires_at": "2026-11-22T05:15:00Z",
  "attempt_count": 1,
  "max_attempts": 3,
  "next_retry_at": null,
  "idempotency_key": "ch-123:job-01J...:complete",
  "release_constraint": ">=2026.11.22-a",
  "payload": { "...": "..." },
  "created_at": "2026-11-22T04:59:00Z",
  "updated_at": "2026-11-22T05:00:00Z"
}
```

- Field kepemilikan (`deduplication_key`, `generation`,
  `worker_id`, `lease_expires_at`, `fencing_token`,
  `attempt_count`, `max_attempts`, `next_retry_at`,
  `idempotency_key`, `release_constraint`) [DF — V2.0 §9.2].
- `status` mengikuti state machine V2.0 §8.1 [DF]: QUEUED,
  CLAIMED, RUNNING, COMPLETED, FAILED, RETRY_WAIT,
  CANCEL_REQUESTED, CANCELLED, DEAD_LETTER.
- `job_kind`: kosakata final menunggu audit schema (V2-007);
  nilai yang dipakai V2.0 sebagai contoh: `INGESTION`
  [DF — V2.0 §8.2]. Jenis lain (Stage 2, publish, live) belum
  distandarisasi [OQ-404].
- `payload` berisi input spesifik job (mis. channel, item,
  parameter render). Skema per `job_kind` didefinisikan
  terpisah per jenis job saat implementasi; kontrak v1 ini
  TIDAK mengarang skema payload. [OQ-405]
- `release_constraint` menyatakan versi release minimum yang
  boleh menjalankan job ini. [PR — penerapan V2.0 §12 "release
  memenuhi constraint job"]

### 4.3 Lease dan fencing token

- `fencing_token`: integer monotonik per job, dimulai dari 1,
  naik 1 setiap kali ownership berpindah (klaim baru setelah
  lease habis / release / fail). [PR — mekanisme dari contoh
  V2.0 §8.3: A=41 → B=42]
- Token TIDAK PERNAH dipakai ulang dan TIDAK PERNAH turun.
  [PR]
- Setiap operasi yang mengubah state job (`heartbeat`,
  `complete`, `fail`, `release`) WAJIB menyertakan fencing
  token yang dipegang worker. Master menolak operasi bila
  token tidak sama dengan token pemilik saat ini.
  [PR — penerapan "fencing divalidasi saat mengirim hasil"
  V2.0 §8.3]

### 4.4 Job events (audit trail)

Master mencatat event berikut ke `job_events` [DF — V2.0 §9.2]:
JOB_CREATED, JOB_DEDUPLICATED, JOB_CLAIMED, JOB_STARTED,
HEARTBEAT_RECEIVED, LEASE_EXPIRED, FENCING_REJECTED,
JOB_COMPLETED, JOB_FAILED, RETRY_SCHEDULED, DEAD_LETTERED,
CANCEL_REQUESTED, JOB_CANCELLED. Setiap event mencatat
`fencing_token` yang berlaku saat itu [DF — V2.0 §9.2].

---

## 5. Endpoint

### 5.1 POST /workers/register — Registrasi worker

**Tujuan:** mendaftarkan worker baru (atau mendaftarkan ulang
worker yang restart) ke Master. [DF — V2.0 §7]

**Autentikasi:** bearer token worker [PR]. Token yang dipakai
harus cocok dengan `worker_id` yang didaftarkan; bila tidak
cocok → `403 FORBIDDEN`.

**Request:**

```json
{
  "worker_id": "prep-vps8-01",
  "worker_type": "prep",
  "hostname": "vmi3762689",
  "release_id": "2026.11.22-a",
  "schema_version": 3,
  "protocol_version": 1,
  "capabilities": ["ingestion", "loopgen_stage2"],
  "capacity": {
    "max_concurrency": 2,
    "cpu_cores": 4,
    "memory_mb": 4096,
    "disk_free_mb": 51200
  }
}
```

**Response sukses (201 baru / 200 sudah ada):**

```json
{
  "ok": true,
  "data": {
    "worker_id": "prep-vps8-01",
    "status": "READY",
    "registered_at": "2026-11-22T04:00:00Z",
    "server_time": "2026-11-22T05:00:00Z",
    "heartbeat_interval_seconds": 60,
    "maintenance_mode": false
  },
  "error": null,
  "request_id": "req-01J..."
}
```

**Aturan:**

- **Idempotent:** register ulang dengan `worker_id` yang sama
  MEMPERBARUI data worker (bukan membuat duplikat); bila
  `release_id`/`protocol_version` berubah, Master mencatat
  perubahan versi. [PR]
- **Idempotency:** header `Idempotency-Key` WAJIB (§2.4).
- **Timeout:** client 30 dtk; server memproses <5 dtk (tulis
  registry singkat). [PR]
- **Retry:** kegagalan transient (5xx, timeout) boleh di-retry
  dengan backoff (§3); `400`/`403`/`409` TIDAK boleh di-retry
  tanpa memperbaiki request. [PR — V2.0 §8.5]
- **Lease/fencing:** tidak berlaku (operasi worker, bukan job).
- **Version compatibility:** Master menerima `protocol_version`
  yang didukungnya; versi yang tidak didukung → `426
  INCOMPATIBLE_PROTOCOL` dan worker TIDAK BOLEH mengklaim job.
  [PR — penerapan "unsupported release → jangan claim job"
  V2.0 §8.5]
- **Error:** `400 VALIDATION_ERROR` (field wajib hilang / nilai
  tidak valid), `401 AUTH_REQUIRED`/`AUTH_INVALID`, `403
  FORBIDDEN` (token tidak cocok dengan worker_id), `409
  WORKER_ID_CONFLICT` (worker_id sudah dipakai identitas lain),
  `426 INCOMPATIBLE_PROTOCOL`, `429 RATE_LIMITED`.

### 5.2 POST /workers/heartbeat — Heartbeat dan kapasitas

**Tujuan:** worker melaporkan dirinya hidup + kapasitas terkini.
Master memakai heartbeat untuk menetapkan STALE/OFFLINE dan
untuk admission. [DF — V2.0 §7, §12]

**Autentikasi:** bearer token worker yang sama dengan
`worker_id`. [PR]

**Request:**

```json
{
  "worker_id": "prep-vps8-01",
  "status": "READY",
  "protocol_version": 1,
  "release_id": "2026.11.22-a",
  "capacity": {
    "active_jobs": 1,
    "max_concurrency": 2,
    "cpu_percent": 45.2,
    "memory_free_mb": 2048,
    "disk_free_mb": 48000,
    "load1": 1.8
  },
  "active_job_ids": ["job-01J..."]
}
```

**Response sukses (200):**

```json
{
  "ok": true,
  "data": {
    "worker_id": "prep-vps8-01",
    "status": "READY",
    "server_time": "2026-11-22T05:01:00Z",
    "maintenance_mode": false,
    "drain_requested": false,
    "next_heartbeat_seconds": 60
  },
  "error": null,
  "request_id": "req-01J..."
}
```

**Aturan:**

- **Interval:** 60 dtk [PR — §3]. Master menandai worker STALE
  setelah 3× terlewat, OFFLINE setelah 10× terlewat.
- **Tulis DB dihemat:** heartbeat TIDAK BOLEH menulis setiap
  sampel metrik ke SQLite; metrik disimpan ringkas atau di
  sistem metrics terpisah [DF — V2.0 §9.2, §9.3]. Master cukup
  memperbarui `last_heartbeat_at` + status + kapasitas
  ringkas.
- **Idempotency:** heartbeat bersifat idempotent secara alami
  (overwrite state terakhir); header `Idempotency-Key` TIDAK
  diwajibkan. [PR]
- **Timeout:** client 30 dtk. [PR]
- **Retry:** heartbeat yang gagal TIDAK di-retry agresif —
  worker cukup mengirim heartbeat berikutnya sesuai jadwal
  (miss dihitung Master). Retry backoff hanya bila worker
  baru saja kehilangan koneksi dan ingin pulih cepat.
  [PR — mencegah badai retry saat Master sibuk]
- **Lease/fencing:** tidak berlaku.
- **`drain_requested`:** bila Master mengembalikan `true`,
  worker TIDAK BOLEH mengklaim job baru dan harus
  menyelesaikan/melepaskan job aktif (lihat §8). [PR —
  penerapan "drain job aktif sebelum restart" V2.0 §12]
- **`maintenance_mode`:** bila `true`, Master TIDAK memberi job
  ke worker ini [DF — V2.0 §12].
- **Error:** `401`, `404 WORKER_NOT_FOUND` (worker belum
  register → worker harus register ulang), `409
  VERSION_MISMATCH` (release/protocol berubah sejak register →
  worker harus register ulang), `429`.

### 5.3 POST /jobs/claim — Mengambil job

**Tujuan:** worker meminta satu job yang cocok dengan
capability-nya. Claim adalah transaksi atomik: Master memilih
job QUEUED yang memenuhi constraint, menetapkan owner +
lease + fencing token, dan mengembalikan job — semua dalam
satu transaksi pendek. [DF — V2.0 §9.3]

**Autentikasi:** bearer token worker. [PR]

**Request:**

```json
{
  "worker_id": "prep-vps8-01",
  "capabilities": ["ingestion", "loopgen_stage2"],
  "max_jobs": 1
}
```

**Response sukses (200 — ada job):**

```json
{
  "ok": true,
  "data": {
    "job": {
      "job_id": "job-01J...",
      "deduplication_key": "ch-123:INGESTION:7",
      "channel_id": "ch-123",
      "job_kind": "INGESTION",
      "generation": 7,
      "status": "CLAIMED",
      "fencing_token": 42,
      "lease_expires_at": "2026-11-22T05:15:00Z",
      "attempt_count": 1,
      "max_attempts": 3,
      "payload": { "...": "..." }
    }
  },
  "error": null,
  "request_id": "req-01J..."
}
```

**Response sukses (200 — tidak ada job):** `data: { "job": null,
"retry_after_seconds": 30 }` — worker menunggu sebelum klaim
lagi (backpressure lewat antrean, bukan polling agresif).
[PR — Rencana Orchestrator §4.2 aturan 3]

**Aturan:**

- **Satu job per klaim** (`max_jobs: 1`). Klaim batch (>1 job)
  TIDAK didukung di v1 [PR]; bila dibutuhkan, diajukan sebagai
  revisi kontrak dengan alasan kapasitas.
- **Seleksi job:** Master hanya memberikan job bila: protocol
  + schema worker kompatibel; capability tersedia; release
  memenuhi `release_constraint` job; worker tidak dalam
  maintenance mode [DF — V2.0 §12]; dan admission lokal
  (CPU/RAM/disk/network/governor) mengizinkan [DF — V2.0 §7].
  Urutan prioritas job (FIFO vs prioritas channel) [OQ-406] —
  V2.0 tidak menetapkannya; v1 memakai FIFO per `created_at`
  [PR] sampai kebijakan prioritas diputuskan.
- **Dedup:** Master TIDAK PERNAH memberikan dua job aktif
  untuk satu deduplication key (unique constraint).
  [DF — V2.0 §8.2]
- **Lease:** klaim sukses menetapkan `lease_expires_at` =
  sekarang + 15 menit [PR — §3] dan menaikkan `fencing_token`
  (+1 dari pemilik sebelumnya, atau 1 untuk klaim pertama).
- **Idempotency:** header `Idempotency-Key` WAJIB. Klaim ulang
  dengan key yang sama dalam jendela retensi mengembalikan
  job yang SAMA (bukan job kedua). [PR]
- **Timeout:** client 30 dtk; server ≤10 dtk (§3). Bila client
  timeout tanpa response, client TIDAK BOLEH mengasumsikan
  klaim gagal — ia harus memverifikasi dengan klaim ulang
  memakai idempotency key yang SAMA. [PR]
- **Retry:** 5xx/timeout boleh di-retry dengan idempotency key
  yang sama; `409`/`429` dihormati (`retry_after_seconds`).
  [PR]
- **Event:** Master mencatat `JOB_CLAIMED` (dengan fencing
  token) ke `job_events`. [DF — V2.0 §9.2]
- **Error:** `401`, `403 WORKER_NOT_READY` (maintenance /
  drain / protocol tidak kompatibel), `409 NO_JOB_AVAILABLE`
  (tidak dipakai — ketiadaan job adalah 200 dengan `job:
  null`, bukan error), `429`, `503 SERVICE_UNAVAILABLE`
  (Master dalam maintenance — worker backoff).

### 5.4 POST /jobs/{id}/heartbeat — Memperpanjang lease

**Tujuan:** worker melaporkan job masih berjalan + progress,
dan memperpanjang lease. [DF — V2.0 §8.3]

**Autentikasi:** bearer token worker pemilik job. [PR]

**Request:**

```json
{
  "worker_id": "prep-vps8-01",
  "fencing_token": 42,
  "status": "RUNNING",
  "progress_percent": 45,
  "progress_note": "stage 2/5: crossfade"
}
```

**Response sukses (200):**

```json
{
  "ok": true,
  "data": {
    "job_id": "job-01J...",
    "status": "RUNNING",
    "fencing_token": 42,
    "lease_expires_at": "2026-11-22T05:17:00Z",
    "cancel_requested": false
  },
  "error": null,
  "request_id": "req-01J..."
}
```

**Aturan:**

- **Interval:** 120 dtk [PR — §3]. Setiap heartbeat sukses
  memperpanjang lease 15 menit dari SEKARANG (bukan dari
  lease lama). [PR]
- **Fencing:** Master menolak heartbeat bila `fencing_token`
  tidak sama dengan token pemilik saat ini → `410
  STALE_FENCING_TOKEN`. Heartbeat TIDAK memperpanjang hak
  worker lama bila ownership sudah berpindah.
  [DF — V2.0 §8.3]
- **`cancel_requested`:** bila Master mengembalikan `true`,
  worker harus menghentikan pekerjaannya secara tertib dan
  memanggil `release` (atau `fail` dengan reason
  `CANCELLED_BY_MASTER`). [PR — penerapan state
  CANCEL_REQUESTED V2.0 §8.1]
- **Progress di-batch:** worker melaporkan progress pada
  heartbeat (bukan tiap detik). [DF — Rencana Orchestrator
  §4.2 aturan 7]
- **Idempotency:** heartbeat idempotent secara alami; header
  `Idempotency-Key` TIDAK diwajibkan. [PR]
- **Timeout:** client 30 dtk. [PR]
- **Retry:** heartbeat yang gagal karena transient boleh
  di-retry SEKALI dalam interval; bila tetap gagal, worker
  menunggu heartbeat berikutnya. Worker TIDAK BOLEH
  menganggap job hilang hanya karena satu heartbeat gagal.
  [PR]
- **Event:** Master mencatat `HEARTBEAT_RECEIVED` — dibatasi
  (batch/ringkas) agar tidak membanjiri `job_events`.
  [PR — mengikuti "batch event yang aman" V2.0 §9.3]
- **Error:** `401`, `403 NOT_JOB_OWNER` (worker lain yang
  memanggil), `404 JOB_NOT_FOUND`, `409 JOB_NOT_RUNNING`
  (job sudah COMPLETED/FAILED/CANCELLED/DEAD_LETTER),
  `410 STALE_FENCING_TOKEN`, `429`.

### 5.5 POST /jobs/{id}/complete — Melaporkan hasil

**Tujuan:** worker melaporkan job selesai beserta hasil.
Ini adalah titik validasi fencing yang PALING PENTING:
Master memeriksa fencing token + generation SEBELUM menerima
hasil. [DF — V2.0 §8.3, §8.4]

**Autentikasi:** bearer token worker pemilik job. [PR]

**Request:**

```json
{
  "worker_id": "prep-vps8-01",
  "fencing_token": 42,
  "result": {
    "status": "SUCCESS",
    "artifacts": [
      {
        "name": "stage2-final.mp4",
        "kind": "video",
        "location_ref": "staging/prep-vps8-01/job-01J.../stage2-final.mp4",
        "size_bytes": 52428800,
        "sha256": "9f86d081...",
        "duration_seconds": 20.0,
        "loudness_lufs": -14.03
      }
    ],
    "metrics": {
      "wall_seconds": 100.1,
      "cpu_seconds": 12.8
    }
  }
}
```

**Response sukses (200):**

```json
{
  "ok": true,
  "data": {
    "job_id": "job-01J...",
    "status": "COMPLETED",
    "fencing_token": 42,
    "accepted_at": "2026-11-22T05:12:00Z",
    "final_location": "drive/channel-123/01_VIDEOS/stage2-final.mp4"
  },
  "error": null,
  "request_id": "req-01J..."
}
```

**Aturan:**

- **Validasi fencing (WAJIB, berurutan):** (1) token sama dengan
  token pemilik saat ini; (2) generation sama dengan generation
  job saat ini; (3) status job masih CLAIMED/RUNNING. Bila (1)
  gagal → `410 STALE_FENCING_TOKEN`; bila (2) gagal → `409
  GENERATION_MISMATCH`; bila (3) gagal → `409 JOB_NOT_RUNNING`.
  [PR — penerapan V2.0 §8.3, §8.4]
- **Validasi hasil:** Master memeriksa artefak yang dilaporkan
  (keberadaan, ukuran, checksum) sebelum menerima. Artefak
  yang tidak lolos validasi → `422 RESULT_INVALID`; job
  TIDAK ditandai COMPLETED (worker boleh memperbaiki dan
  mengirim ulang selama lease masih berlaku, atau job gagal
  sesuai kebijakan retry). [PR — penerapan "hasil tervalidasi
  dan diterima" V2.0 §8.1]
- **Promosi atomik:** hasil diterima secara atomik, lalu file
  dipindahkan ke lokasi final; hasil lama masuk quarantine
  atau cleanup sesuai retensi. [DF — V2.0 §8.4]
- **Transport artefak:** `location_ref` adalah referensi ke
  artefak di staging (bukan isi file). Mekanisme transfer
  artefak besar (Drive staging vs upload langsung) BELUM
  diputuskan [OQ-407] — kontrak v1 mendefinisikan HANYA
  metadata hasil; implementasi TIDAK BOLEH mengasumsikan
  transport tertentu sampai OQ-407 diputuskan.
- **Checksum:** `sha256` WAJIB untuk setiap artefak (integritas
  hasil). [PR]
- **Idempotency:** header `Idempotency-Key` WAJIB. Pengulangan
  dengan key yang sama mengembalikan hasil sukses yang sama
  (tanpa duplikasi promosi). [PR — diuji T1.5-H]
- **Timeout:** client 60 dtk [PR — lebih panjang dari endpoint
  lain karena validasi artefak]; server memvalidasi dalam
  batas itu atau mengembalikan `202 ACCEPTED` + status
  validasi asinkron [PR — bila validasi berat, jangan blokir
  request; pola ini TBD saat implementasi, OQ-408].
- **Retry:** `410`/`409`/`422` TIDAK boleh di-retry tanpa
  memperbaiki penyebabnya. 5xx/timeout boleh di-retry dengan
  idempotency key yang SAMA. [PR]
- **Event:** Master mencatat `JOB_COMPLETED` (atau
  `FENCING_REJECTED` bila ditolak) ke `job_events`.
  [DF — V2.0 §9.2]
- **YouTube upload:** bila job ini melibatkan upload YouTube,
  `complete` TIDAK BOLEH dipanggil sebelum operation record
  upload dibuat dan reconciliation mengonfirmasi status
  upload. Detail alur upload di luar kontrak v1 ini.
  [DF — V2.0 §8.4; OQ-409]
- **Error:** `401`, `403 NOT_JOB_OWNER`, `404 JOB_NOT_FOUND`,
  `409 JOB_NOT_RUNNING`, `409 GENERATION_MISMATCH`, `410
  STALE_FENCING_TOKEN`, `422 RESULT_INVALID`, `429`.

### 5.6 POST /jobs/{id}/fail — Melaporkan kegagalan

**Tujuan:** worker melaporkan job gagal beserta klasifikasi
error. Klasifikasi menentukan retry vs dead-letter.
[DF — V2.0 §8.5]

**Autentikasi:** bearer token worker pemilik job. [PR]

**Request:**

```json
{
  "worker_id": "prep-vps8-01",
  "fencing_token": 42,
  "error": {
    "category": "TRANSIENT",
    "code": "DRIVE_TIMEOUT",
    "message": "Drive upload timeout setelah 3 percobaan",
    "retryable": true,
    "details": { "...": "..." }
  }
}
```

**Response sukses (200):**

```json
{
  "ok": true,
  "data": {
    "job_id": "job-01J...",
    "status": "RETRY_WAIT",
    "attempt_count": 2,
    "max_attempts": 3,
    "next_retry_at": "2026-11-22T05:20:00Z",
    "dead_lettered": false
  },
  "error": null,
  "request_id": "req-01J..."
}
```

**Aturan:**

- **Kategori error** (wajib salah satu) [PR — dari tabel
  V2.0 §8.5]:
  - `TRANSIENT` — network timeout, HTTP 5xx, storage sementara
    → retry dengan backoff + jitter.
  - `PERMANENT` — input tidak valid, error yang tidak akan
    pulih dengan retry → DEAD_LETTER.
  - `QUOTA` — YouTube `quotaExceeded` atau kuota provider →
    TIDAK retry otomatis; tunggu reset / tindakan operator.
  - `AUTH` — permission / kredensial tidak valid → pause +
    eskalasi.
  - `RATE_LIMIT` — hormati `retry_after_seconds`.
- **Master yang memutuskan nasib job**, bukan worker: Master
  memvalidasi kategori terhadap `attempt_count`/`max_attempts`
  dan kebijakan retry. Worker hanya MELAPORKAN; Master yang
  menetapkan RETRY_WAIT vs DEAD_LETTER. [PR — mencegah worker
  yang salah klasifikasi membakar retry/kuota]
- **`QUOTA` TIDAK PERNAH di-retry otomatis**, apa pun kata
  worker. [DF — V2.0 §8.5]
- **Fencing:** validasi token yang sama seperti `complete`;
  token basi → `410 STALE_FENCING_TOKEN` (laporan gagal dari
  worker basi DIABAIKAN — job sudah dimiliki worker lain).
  [PR]
- **Backoff retry:** Master menghitung `next_retry_at` dengan
  exponential backoff + jitter dari `attempt_count`.
  [PR — V2.0 §8.5]
- **Dead letter:** bila `attempt_count >= max_attempts` untuk
  error TRANSIENT, atau kategori PERMANENT, Master menandai
  DEAD_LETTER dan mencatat `DEAD_LETTERED`.
  [DF — V2.0 §8.1, §8.5]
- **Idempotency:** header `Idempotency-Key` WAJIB. Laporan
  gagal ganda dengan key yang sama tidak menaikkan
  `attempt_count` dua kali. [PR]
- **Timeout:** client 30 dtk. [PR]
- **Retry:** 5xx/timeout boleh di-retry dengan key yang sama;
  `410`/`409` tidak. [PR]
- **Event:** `JOB_FAILED`, lalu `RETRY_SCHEDULED` atau
  `DEAD_LETTERED`. [DF — V2.0 §9.2]
- **Error:** `401`, `403 NOT_JOB_OWNER`, `404 JOB_NOT_FOUND`,
  `400 INVALID_ERROR_CATEGORY`, `409 JOB_NOT_RUNNING`, `410
  STALE_FENCING_TOKEN`, `429`.

### 5.7 POST /jobs/{id}/release — Melepaskan job

**Tujuan:** worker melepaskan job secara sukarela SEBELUM
selesai (mis. drain, kondisi lokal memburuk, cancel yang
diminta Master). Job kembali ke antrean untuk worker lain.
[PR — tidak ada endpoint ini di V2.0; ditambahkan karena
drain (§12) dan cancel (§8.1) membutuhkannya]

**Autentikasi:** bearer token worker pemilik job. [PR]

**Request:**

```json
{
  "worker_id": "prep-vps8-01",
  "fencing_token": 42,
  "reason": "DRAIN_REQUESTED",
  "note": "Worker diminta drain sebelum restart"
}
```

**Response sukses (200):**

```json
{
  "ok": true,
  "data": {
    "job_id": "job-01J...",
    "status": "QUEUED",
    "fencing_token": 43,
    "released_at": "2026-11-22T05:13:00Z"
  },
  "error": null,
  "request_id": "req-01J..."
}
```

**Aturan:**

- **`reason` wajib salah satu:** `DRAIN_REQUESTED`,
  `CANCELLED_BY_MASTER`, `LOCAL_CONDITIONS` (CPU/disk/network
  lokal tidak memungkinkan lanjut), `WORKER_SHUTDOWN`.
  [PR]
- **Release TIDAK menaikkan `attempt_count`** (bukan kegagalan)
  dan TIDAK memicu backoff — job langsung QUEUED.
  [PR — membedakan pelepasan tertib dari kegagalan]
- **Fencing token naik** (+1) saat release, sehingga worker
  lama tidak dapat lagi mengirim hasil untuk job itu.
  [PR — konsisten dengan "token naik setiap ownership
  berpindah" V2.0 §8.3]
- **Fencing:** token basi → `410 STALE_FENCING_TOKEN`.
  [PR]
- **Idempotency:** header `Idempotency-Key` WAJIB. Release
  ganda dengan key yang sama mengembalikan hasil yang sama.
  [PR]
- **Timeout:** client 30 dtk. [PR]
- **Retry:** 5xx/timeout boleh di-retry dengan key yang sama.
  [PR]
- **Event:** Master mencatat `LEASE_EXPIRED`-setara? TIDAK —
  release adalah event tersendiri. V2.0 §9.2 tidak
  mendefinisikan event RELEASE; kontrak v1 menambahkan
  `JOB_RELEASED` ke kosakata event [PR — usulan penambahan;
  bila implementasi menolak, catat sebagai deviasi].
- **Error:** `401`, `403 NOT_JOB_OWNER`, `404 JOB_NOT_FOUND`,
  `400 INVALID_RELEASE_REASON`, `409 JOB_NOT_RUNNING`, `410
  STALE_FENCING_TOKEN`, `429`.

---

## 6. Katalog error

Semua kode di bawah ini [PR] — kosakata kontrak v1. `retryable`
menentukan apakah client boleh mencoba lagi.

| Kode | HTTP | Arti | Retryable |
|---|---|---|---|
| `VALIDATION_ERROR` | 400 | Request tidak valid (field hilang / nilai salah) | Tidak |
| `INVALID_ERROR_CATEGORY` | 400 | Kategori error `fail` tidak dikenal | Tidak |
| `INVALID_RELEASE_REASON` | 400 | Alasan `release` tidak dikenal | Tidak |
| `AUTH_REQUIRED` | 401 | Token tidak ada | Tidak (perbaiki auth) |
| `AUTH_INVALID` | 401 | Token tidak valid / kedaluwarsa | Tidak (perbaiki auth) |
| `FORBIDDEN` | 403 | Token tidak cocok dengan worker_id | Tidak |
| `NOT_JOB_OWNER` | 403 | Pemanggil bukan pemilik job | Tidak |
| `WORKER_NOT_READY` | 403 | Worker dalam maintenance / drain / protocol tidak kompatibel | Tidak (tunggu kondisi berubah) |
| `JOB_NOT_FOUND` | 404 | Job tidak ada | Tidak |
| `WORKER_NOT_FOUND` | 404 | Worker belum register | Tidak (register dulu) |
| `JOB_NOT_RUNNING` | 409 | Job sudah COMPLETED/FAILED/CANCELLED/DEAD_LETTER | Tidak |
| `GENERATION_MISMATCH` | 409 | Generation hasil ≠ generation job saat ini | Tidak |
| `WORKER_ID_CONFLICT` | 409 | worker_id dipakai identitas lain | Tidak (eskalasi) |
| `VERSION_MISMATCH` | 409 | Release/protocol berubah sejak register | Tidak (register ulang) |
| `STALE_FENCING_TOKEN` | 410 | Fencing token sudah tidak berlaku (ownership berpindah) | Tidak |
| `RESULT_INVALID` | 422 | Artefak hasil tidak lolos validasi | Tidak (perbaiki hasil) |
| `INCOMPATIBLE_PROTOCOL` | 426 | Protocol version tidak didukung Master | Tidak (upgrade worker) |
| `RATE_LIMITED` | 429 | Rate limit per worker terlampaui; hormati `retry_after_seconds` | Ya (setelah jeda) |
| `INTERNAL_ERROR` | 500 | Error server (termasuk lock SQLite yang tidak pulih setelah retry internal) | Ya (backoff) |
| `SERVICE_UNAVAILABLE` | 503 | Master dalam maintenance / overload | Ya (backoff) |

**Catatan `STALE_FENCING_TOKEN` (410):** kode ini BUKAN error
sistem — ia adalah perilaku yang BENAR saat worker basi
mencoba bertindak. Worker yang menerimanya HARUS berhenti
mengerjakan job itu dan membuang hasil lokalnya (atau
mengarantina sesuai retensi), bukan retry. [PR]

---

## 7. Fencing & stale-worker — kasus uji wajib

Bagian ini memenuhi kriteria selesai item #4 sprint: kasus uji
stale-worker terdefinisi sebagai kasus uji wajib. [PR — mekanisme
dari V2.0 §8.3; perincian langkah adalah kontrak v1]

### 7.1 Mekanisme fencing token (ringkasan)

1. Setiap job punya `fencing_token` (integer, mulai 1).
2. Setiap perpindahan ownership (klaim baru setelah lease habis /
   release / fail) menaikkan token +1. Token tidak pernah dipakai
   ulang dan tidak pernah turun.
3. Worker menerima token saat klaim dan WAJIB menyertakannya di
   setiap operasi job berikutnya (`heartbeat`, `complete`,
   `fail`, `release`).
4. Master membandingkan token yang dikirim dengan token pemilik
   saat ini. Bila tidak sama → tolak dengan `410
   STALE_FENCING_TOKEN` dan catat event `FENCING_REJECTED`.
5. Heartbeat dari worker basi TIDAK memperpanjang lease dan
   TIDAK mengembalikan ownership. Satu-satunya cara worker
   mendapatkan kembali job adalah klaim baru (dengan token baru).
6. Fencing TIDAK dapat membatalkan side effect eksternal yang
   sudah terjadi (mis. upload YouTube yang sudah diterima).
   Karena itu upload memakai operation record + reconciliation
   tersendiri (§5.5, OQ-409). [DF — V2.0 §8.4]

### 7.2 Kasus uji wajib TC-STALE-1

**Nama:** Worker basi ditolak setelah lease berpindah.

**Kondisi awal:** Master berjalan; worker A dan worker B
terdaftar (capability sama); 1 job QUEUED (`job_kind`
apa pun yang didukung kedua worker).

**Langkah:**

| # | Aksi | Hasil yang diharapkan |
|---|---|---|
| 1 | A klaim job | Sukses. A menerima `fencing_token: 41`, lease 15 menit. Event `JOB_CLAIMED` tercatat. |
| 2 | A mulai bekerja, lalu koneksi A ke Master DIPUTUS (simulasi: hentikan proses worker A, atau blokir jaringan A→Master) | A tidak dapat heartbeat. Master mencatat heartbeat A terlewat. |
| 3 | Tunggu sampai lease A habis (15 menit, atau percepat dengan lease uji yang lebih pendek — catat nilai yang dipakai) | Master menandai lease kedaluwarsa; event `LEASE_EXPIRED`; job kembali QUEUED. |
| 4 | B klaim job | Sukses. B menerima `fencing_token: 42` (naik +1). Event `JOB_CLAIMED` kedua tercatat. |
| 5 | B menyelesaikan job dan memanggil `complete` dengan token 42 | Sukses. Job COMPLETED dengan hasil B. Event `JOB_COMPLETED` tercatat (token 42). |
| 6 | A "hidup kembali" (proses di-start / jaringan pulih) dan memanggil `complete` dengan token 41 + hasil lamanya | **DITOLAK.** Response `410 STALE_FENCING_TOKEN`. Event `FENCING_REJECTED` tercatat. Hasil A TIDAK diterima, TIDAK menimpa hasil B, TIDAK dipromosikan. |
| 7 | A memanggil `heartbeat` job dengan token 41 | **DITOLAK.** `410 STALE_FENCING_TOKEN`. Lease TIDAK diperpanjang. |
| 8 | Verifikasi akhir | Tepat 1 hasil final (milik B) di lokasi final; 0 duplikat; 0 hasil A di lokasi final; `job_events` menunjukkan urutan CLAIMED(41) → LEASE_EXPIRED → CLAIMED(42) → COMPLETED(42) → FENCING_REJECTED(41). |

**Kriteria lolos:** semua langkah 1–8 sesuai harapan; tidak ada
hasil ganda; tidak ada hasil basi yang lolos. **Kriteria gagal:**
langkah 6 atau 7 diterima (hasil basi lolos), atau hasil B
tertimpa, atau event tidak lengkap.

**Kaitan acceptance plan:** kasus ini adalah pendalaman dari
T1.5-H (Fencing + idempotency) — T1.5-H menguji penolakan
token basi secara umum; TC-STALE-1 menguji rantai lengkap
dengan dua worker dan lease expiry. [DF — Acceptance Plan §6]

### 7.3 Varian wajib

| ID | Varian | Hasil yang diharapkan |
|---|---|---|
| TC-STALE-2 | A kembali dan memanggil `fail` (bukan `complete`) dengan token basi | Ditolak `410`; status job TIDAK berubah (tetap COMPLETED milik B) |
| TC-STALE-3 | A kembali dan memanggil `release` dengan token basi | Ditolak `410`; job TIDAK kembali ke antrean |
| TC-STALE-4 | A kembali SEBELUM B menyelesaikan job (job masih RUNNING milik B), lalu A memanggil `complete` dengan token 41 | Ditolak `410`; job tetap milik B; B dapat menyelesaikan normal |
| TC-STALE-5 | Dua worker mengklaim job yang sama BERSAMAAN (race) | Tepat 1 klaim sukses (transaksi atomik); klaim kedua menerima job lain atau `job: null`; 0 duplikat ownership |
| TC-STALE-6 | A memanggil `complete` dengan token BENAR tapi `generation` lama (job sudah di-requeue dengan generation baru) | Ditolak `409 GENERATION_MISMATCH`; hasil lama tidak menimpa generation baru |

### 7.4 Yang TIDAK dijamin fencing

- Fencing tidak mencegah worker basi MENGERJAKAN job (CPU
  terbuang) — ia hanya mencegah HASIL basi diterima. Pencegahan
  kerja ganda adalah tugas lease + deteksi STALE, bukan fencing.
- Fencing tidak membatalkan upload YouTube / panggilan API
  eksternal yang sudah dieksekusi worker basi. [DF — V2.0 §8.4]

---

## 8. Siklus lease & drain

**Siklus hidup lease** [PR — penerapan V2.0 §8.3, §12]:

```
Klaim → lease 15 mnt (token N)
  → heartbeat job tiap 120 dtk → lease diperpanjang 15 mnt dari sekarang
  → complete/fail/release → lease berakhir
  → (tanpa heartbeat) lease habis → LEASE_EXPIRED → job QUEUED → token N+1
```

**Drain (sebelum restart/maintenance worker):**

1. Master mengembalikan `drain_requested: true` di heartbeat
   worker, atau operator menandai worker DRAINING.
2. Worker berhenti mengklaim job baru.
3. Worker menyelesaikan job aktif yang hampir selesai, atau
   memanggil `release` untuk job yang masih lama.
4. Setelah 0 job aktif, worker boleh berhenti dengan aman.
5. Bila worker mati mendadak saat drain, lease expiry menangani
   sisanya (job kembali ke antrean). [DF — V2.0 §8.5 "worker
   crash → lease recovery dan retry"]

**Maintenance mode:** worker dalam maintenance TIDAK menerima
job baru [DF — V2.0 §12]; job aktifnya tetap berjalan sampai
selesai atau dilepas (drain), kecuali operator memutuskan
memutusnya (Level C — butuh persetujuan owner bila memengaruhi
produksi).

**Master restart:** setelah Master pulih, Master memeriksa
lease, fencing, dan job events sebelum memberi job baru
[DF — V2.0 §20]. Job yang leasenya habis selama Master mati
kembali ke antrean; job yang masih dalam lease tetap dimiliki
workernya (worker melanjutkan heartbeat normal).

---

## 9. Idempotency & dedup (ringkasan operasional)

- **Register ganda** (worker_id sama) → update, bukan duplikat.
- **Claim ganda** (idempotency key sama) → job yang sama.
- **Complete ganda** (idempotency key sama) → hasil sukses yang
  sama, promosi sekali.
- **Fail ganda** (idempotency key sama) → `attempt_count` naik
  sekali.
- **Release ganda** (idempotency key sama) → hasil yang sama.
- **Job ganda** (deduplication key sama) → TIDAK PERNAH dibuat;
  request job berulang mengembalikan job aktif yang sudah ada
  (event `JOB_DEDUPLICATED`). [DF — V2.0 §8.2]
- **Hasil ganda** (dua worker menyelesaikan job yang sama) →
  dicegah fencing + generation; hanya hasil pemilik token
  terkini yang diterima.

---

## 10. Kompatibilitas versi & deployment

- Master hanya memberi job bila protocol + schema worker
  kompatibel, capability tersedia, release memenuhi constraint
  job, dan worker tidak dalam maintenance mode.
  [DF — V2.0 §12]
- Urutan deploy: build + test → migration backward-compatible
  → Master dulu (bila API baru tetap kompatibel dengan worker
  lama) → satu worker canary → smoke test → perluas rollout →
  hentikan pemberian job ke worker lama → drain → rollback bila
  gate gagal. [DF — V2.0 §12]
- Migration memakai expand-and-contract: kolom lama TIDAK
  dihapus sebelum seluruh worker + release memakai skema baru.
  [DF — V2.0 §12]
- **Aturan kontrak:** endpoint v1 TIDAK BOLEH dihapus atau
  diubah maknanya dalam v1; perubahan breaking masuk v2.
  Field baru dalam v1 HARUS opsional dan diabaikan aman oleh
  client lama. [PR]
- **Gerbang:** TIDAK ADA migration skema sebelum V2-007
  (audit schema) disetujui. Kontrak v1 ini adalah DOKUMEN
  DESAIN — implementasinya menunggu gerbang V2-007.
  [DF — Directive Batch 1; sprint item #4]

---

## 11. Pertanyaan terbuka (OQ kontrak v1)

| ID | Pertanyaan | Menghambat |
|---|---|---|
| OQ-401 | Alamat dasar + port API worker (Tailscale)? | Implementasi + provisioning |
| OQ-402 | Prosedur penerbitan + rotasi token worker (siapa, masa berlaku, rotasi tanpa downtime)? | Implementasi auth |
| OQ-403 | Nilai rate limit per worker? | Implementasi §2.2 |
| OQ-404 | Kosakata final `job_kind` + `capabilities`? | Menunggu audit schema V2-007 |
| OQ-405 | Skema `payload` per `job_kind`? | Implementasi per jenis job |
| OQ-406 | Kebijakan prioritas klaim (FIFO vs prioritas channel)? | Implementasi §5.3 |
| OQ-407 | Mekanisme transfer artefak besar (Drive staging vs upload langsung)? | Implementasi §5.5 |
| OQ-408 | Apakah validasi hasil `complete` sinkron atau asinkron (202 + status)? | Implementasi §5.5 |
| OQ-409 | Alur operation record + reconciliation upload YouTube? | Job publish via API |
| OQ-410 | Nilai final lease/heartbeat/timeout/retry (saat ini PROPOSAL §3)? | Validasi canary + V2-008 |

---

## 12. Ketertelusuran (kontrak → V2.0)

| Bagian kontrak | Sumber V2.0 |
|---|---|
| §1 prinsip 1–2 (pull, SQLite di Master) | §5, §8, §9.1 |
| §1 prinsip 3 (dedup key + generation) | §8.2 |
| §1 prinsip 4 (lease + fencing) | §8.3 |
| §1 prinsip 5–6 (idempotency, hasil atomik) | §8.4 |
| §1 prinsip 7 (upload YouTube) | §8.4 |
| §1 prinsip 8 (Tailscale-only) | §21 |
| §1 prinsip 9 (admission) | §7 |
| §1 prinsip 10 (transaksi pendek) | §9.3 |
| §4.1–4.2 (model worker/job) | §7, §9.2 |
| §4.4 (job events) | §9.2 |
| §5.1–5.7 (endpoint) | §7, §8.1–8.5, §12 |
| §6 (katalog error) | §8.5 (kategori retry) |
| §7 (fencing + stale-worker) | §8.3, §19 |
| §8 (lease + drain) | §8.3, §8.5, §12 |
| §10 (versi + deployment) | §12 |
| §2.2 (auth, rate limit, rotasi) | §21 |
