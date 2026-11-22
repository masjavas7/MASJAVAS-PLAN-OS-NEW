# IMPLEMENTATION CHECKLIST — MASJAVAS TUBE OS V2

**Dokumen acuan:** `MASTER_ARCHITECTURE_MASJAVAS_TUBE_OS_V2.md`
**Versi:** 1.0
**Tanggal:** 22 November 2026
**Status:** READY FOR ENGINEERING EXECUTION — dengan audit schema dan verifikasi biaya sebagai gerbang wajib
**Prinsip:** Satu perubahan pada satu waktu. Ukur sebelum dan sesudah. Tidak ada perubahan production tanpa rollback.

---

## CATATAN VERIFIKASI & GROUNDING MUSE (22 Nov 2026, 05:1x WIB)

Dokumen ini dibekukan dari naskah yang disampaikan user 22 Nov 05:05 WIB. Naskah tersebut
disusun tanpa akses langsung ke file workspace; Muse memverifikasi koreksi-koreksinya
terhadap memori + dokumen engineering dan menambahkan grounding berikut. Isi naskah
tidak diubah; catatan ini adalah lampiran verifikasi.

**Koreksi baseline — TERVERIFIKASI:**

- Resource Guard v2 live sejak 20 Nov 23:02 — benar (commit `5b8279a` + `4fff60a` +
  `2e61dcc`; service live + web restart 23:02:22).
- Modal ingestion 1 core / 2 GB RAM; ≥4 GB adalah ambang gate — benar (orchestrator §4.3).
- Antrean Stage 2: 50 QUEUED + 4 PENDING — benar (orchestrator §2).
- Stage 2: 12,8 dtk CPU vs 100,1 dtk wall/klip — benar (n=1.011 DONE, 7 hari).
- Ingestion CPU 0,0%; churn 66% (333 vs 123); `shed_render` 112× log-only — benar.

**Grounding file/kode aktual (untuk konversi ke tiket per file/module):**

| Area | Lokasi aktual |
|---|---|
| Resource Guard v2 | `live_supervisor.py` (+417/−59), `ingestion.py` (+22), `settings.py` (+10 knob); commit `5b8279a`, `4fff60a`, `2e61dcc` |
| Janitor bug | `tools/vps_disk_janitor.py` — klausa `OR channel_id LIKE 'STAGING-SLOT-%'` (673/707 baris masih STAGING-SLOT-*) |
| Ingestion gates | `worker/ingestion.py` — `run_ingestion_worker_cycle(max_jobs=3)`, `compute_adaptive_ingestion_concurrency` (max 4), claim gate load1 >2,8; sweeper batch_size=2/120 dtk |
| Live traceback DB-lock | `live_supervisor.py:878, 920` (health check) |
| Stage 2 driver | `/tmp/stage2-driver-continuous.py` (standalone, sudah terpisah 3+ hari) |
| Daemon (satu loop) | `worker.autonomous_daemon` — lane publish + ingestion + loopgen satu event loop asyncio |
| Tabel job | `operational_tasks` (tanpa `slot_number`, pakai `channel_id`), `background_jobs` (state wave), `ingestion_jobs`, `loopgen_jobs`, `loopgen_stage2_runs`, `loopgen_distribution`, `loopgen_agent_tasks`, `audit_logs` (`decided_by`, `INGESTION_JOB_CREATED`) |
| Services | `masjavas-os-web.service`, `masjavas-os-worker.service`, `masjavas-os-tunnel` |
| KEIN | Konektor `custom.kein` (Muse); kunci image di OWNER-only encrypted dashboard config; pemanggilan di kode VPS — audit V2-001 butuh akses baca kode VPS |

**Klarifikasi status (penting):**

- Batch 1 (V2-001–V2-010) adalah **audit/read-only**. Status "READY FOR ENGINEERING
  EXECUTION" berlaku untuk Batch 1; implementasi P2+ (split proses, cgroup, migration,
  API orchestration) baru boleh jalan setelah V2-007 lolos + gerbang biaya lolos.
- **Batas mandat otonom:** schema/migration, systemd, firewall, infra produksi = L3 —
  butuh persetujuan user per kasus. L1 (diagnostik baca-saja) otomatis; L2 (fix risiko
  rendah + backup + tes) otomatis. Checklist ini tidak menggantikan persetujuan L3.
- **V2-009 verifikasi harga panel:** `my.contabo.com` dilindungi Cloudflare Turnstile
  yang TIDAK BISA dilewati browser cloud Muse (terverifikasi 17 Nov) — verifikasi harga
  Master/VPS 8 butuh user cek sendiri di browsernya atau ambil alih browser live.
- **P1.3 audit schema — tambahan tabel:** selain yang disebut naskah, audit juga
  `ingestion_jobs`, `loopgen_jobs`, `loopgen_stage2_runs`, `loopgen_distribution`,
  `loopgen_agent_tasks`, `background_jobs`, `audit_logs` (jejak `decided_by`:
  `AUTOPILOT_STARTUP` vs `AUTONOMOUS_SWEEPER` — penting untuk P3.1 sumber churn).
- **V2-001 KEIN — opsi fallback sudah ada harganya** (riset 18 Nov, side chat):
  DeepInfra FLUX-1-schnell Rp 35,44/gambar; Muse Image API Rp 180/gambar;
  Kie.ai Seedream 5.5 ≈ Rp 7,8 jt/700 channel; ModelsLab Unlimited $149/bln.
  Pilot DeepInfra 3 varian (Rp 800) sudah ditawarkan, keputusan user belum dijawab.
  Jangan riset ulang dari nol — pakai dokumen `engineering/docs/BIAYA_GENERATE_IMAGE_700_1000_CHANNEL.md`
  dan `REPO_GITHUB_GENERATE_IMAGE_API_KEY.md`.

---

# 0. Aturan eksekusi yang mengikat

## 0.1 Aturan perubahan

- Setiap task memiliki ID.
- Setiap task mempunyai kondisi sebelum perubahan.
- Setiap task mempunyai daftar file yang akan diubah.
- Setiap perubahan database mempunyai migration dan rollback plan.
- Setiap perubahan systemd mempunyai konfigurasi lama yang dapat dipulihkan.
- Setiap perubahan production diuji di lingkungan terbatas terlebih dahulu.
- Setiap deployment memiliki release ID.
- Setiap deployment mempunyai post-deployment verification.
- Observasi minimal 72 jam untuk perubahan operasional yang relevan.
- Metrik dengan baseline 7 hari tetap dinilai menggunakan window 7 hari.
- Satu metrik kritis memburuk berarti gate gagal.
- Tidak ada perubahan massal tanpa canary.

## 0.2 Larangan selama implementasi

- Jangan mengganti SQLite dengan PostgreSQL tanpa trigger yang terbukti.
- Jangan menambahkan Redis hanya karena sistem mulai memakai worker remote.
- Jangan membuat tabel `jobs` baru sebelum audit `operational_tasks`.
- Jangan menjalankan dua authoritative scheduler.
- Jangan membuka file SQLite melalui jaringan.
- Jangan menjalankan FFmpeg berat di dalam web/API.
- Jangan membiarkan prep mengganggu live.
- Jangan menghapus file berdasarkan kondisi "metadata tidak ditemukan" saja.
- Jangan retry otomatis untuk YouTube `quotaExceeded`.
- Jangan mengaktifkan VPS 8 sebelum harga dan biaya storage diverifikasi.
- Jangan melakukan migrasi Master bersamaan dengan cutover worker.

---

# 1. TRACK P0 — Operational Stabilization

**Prioritas:** CRITICAL
**Tujuan:** melindungi operasi yang sedang berjalan sebelum perubahan arsitektur besar.

## P0.1 — KEIN dependency decision

- Audit seluruh pemanggilan KEIN.
- Identifikasi fungsi LoopGen Tahap 1/2 yang bergantung pada KEIN.
- Catat job yang sedang berjalan dan checkpoint terakhir.
- Putuskan top-up atau provider fallback.
- Jika belum ada keputusan provider, siapkan mode HOLD yang aman.
- Pastikan job yang dihentikan dapat dilanjutkan.
- Pastikan fallback tidak menciptakan job duplikat.
- Uji satu channel canary.
- Catat biaya per eksekusi.
- Catat tingkat keberhasilan dan kualitas hasil.

**Acceptance criteria**

- Tidak ada job yang hilang akibat pergantian provider.
- Tidak ada eksekusi ganda.
- Tahap 1/2 dapat dilanjutkan dari checkpoint.
- Keputusan provider terdokumentasi.

**Rollback**

- Kembalikan ke mode HOLD.
- Jangan membuat job baru sampai dependency siap.

## P0.2 — Encoder 706 dan orphan process

- Ambil log FFmpeg lengkap.
- Catat exit code dan signal.
- Catat stderr.
- Catat process tree.
- Catat parameter FFmpeg dan input file.
- Catat CPU, RAM, disk I/O, dan TX ketika insiden.
- Bedakan FFmpeg mati, RTMP putus, dan YouTube broadcast complete.
- Verifikasi supervisor tidak meninggalkan proses orphan.
- Tambahkan recovery yang tidak membuat supervisor ganda.
- Uji restart encoder.
- Uji broadcast yang sudah complete.
- Uji koneksi RTMP putus dan tersambung kembali.

**Acceptance criteria**

- 0 orphan process.
- Maksimal 1 unplanned encoder death/24 jam pada tahap awal.
- Target berikutnya: 0 unplanned death selama 72 jam.
- Broadcast health terdeteksi secara terpisah dari PID FFmpeg.

## P0.3 — Resource Guard v2 audit

Resource Guard v2 sudah berjalan sejak 20 November 23:02. Jangan mengganti implementasi yang sudah live tanpa audit.

- Identifikasi repository dan commit implementasi Resource Guard v2.
- Inventarisasi seluruh input metrik.
- Inventarisasi seluruh threshold.
- Inventarisasi seluruh aksi governor.
- Identifikasi aksi yang benar-benar mengeksekusi perubahan.
- Identifikasi aksi yang hanya menulis log.
- Verifikasi `shed_render`.
- Verifikasi siapa yang menerima perintah shed.
- Verifikasi siapa yang menjalankan tindakan shed.
- Pastikan governor tidak mematikan workload yang salah.
- Petakan guard G1–G12 terhadap implementasi saat ini.
- Tandai guard yang perlu dihapus, dilokalkan, atau dipertahankan.

**Acceptance criteria**

- Setiap guard mempunyai owner.
- Setiap guard mempunyai aksi yang dapat diverifikasi.
- Tidak ada guard saling sandera.
- Tidak ada keputusan governor yang hanya tampak bekerja karena log.

## P0.4 — Janitor safety

- Audit seluruh kondisi penghapusan file.
- Inventarisasi referensi file dalam database.
- Identifikasi file yang masih digunakan proses aktif.
- Identifikasi status channel CONNECTED.
- Tambahkan pengecekan owner.
- Tambahkan pengecekan job status.
- Tambahkan pengecekan lifecycle.
- Tambahkan pengecekan umur file.
- Tambahkan quarantine.
- Tambahkan grace period sebelum penghapusan permanen.
- Catat setiap kandidat penghapusan.
- Uji file orphan yang benar-benar aman dihapus.
- Uji file CONNECTED yang tidak boleh dihapus.
- Uji file yang sedang digunakan FFmpeg.

**Acceptance criteria**

- Tidak ada file CONNECTED terhapus karena klasifikasi unpaired.
- File yang meragukan masuk quarantine.
- Penghapusan dapat diaudit dan ditelusuri.

---

# 2. TRACK P1 — Baseline, Inventory, dan Audit Schema

**Prioritas:** CRITICAL
**Tujuan:** mengetahui kondisi nyata sebelum mengubah struktur sistem.

## P1.1 — Baseline snapshot

- Ambil snapshot CPU per process dan per systemd unit.
- Ambil RAM usage dan available memory.
- Ambil disk usage per direktori.
- Ambil disk I/O.
- Ambil TX/RX per interface.
- Ambil load average.
- Hitung SQLite lock per 24 jam.
- Hitung job SUPERSEDED per 7 hari.
- Hitung job COMPLETED per 7 hari.
- Catat ingestion RUNNING dan PENDING.
- Catat Stage 2 QUEUED dan PENDING.
- Catat publish per hari.
- Catat keterlambatan publish median dan p90.
- Catat encoder death dan orphan.
- Catat stream aktif.
- Simpan hasil sebagai baseline versioned.

**Deliverable:** `BASELINE_MASJAVAS_V2_<tanggal>.md`

## P1.2 — Service inventory

- Inventarisasi seluruh systemd service.
- Inventarisasi timer dan cron.
- Inventarisasi process Python.
- Inventarisasi seluruh pemanggilan FFmpeg.
- Inventarisasi background task di FastAPI.
- Inventarisasi scheduler.
- Inventarisasi sweeper.
- Inventarisasi Fleet Runner.
- Inventarisasi Resource Guard.
- Inventarisasi janitor.
- Inventarisasi proses upload dan Google Drive sync.
- Petakan dependency antarservice.
- Identifikasi proses yang masih berbagi event loop.

**Deliverable:** `SERVICE_INVENTORY_V2.md`

## P1.3 — Database schema audit

**Tidak boleh membuat schema baru sebelum tahap ini selesai.**

- Backup SQLite menggunakan mekanisme konsisten.
- Simpan schema dump.
- Inventarisasi seluruh tabel.
- Audit `operational_tasks`.
- Audit `channels`.
- Audit `auth_tokens`.
- Audit tabel publish.
- Audit tabel live.
- Audit tabel LoopGen.
- Audit seluruh indeks.
- Audit unique constraint.
- Audit status dan transisi job.
- Identifikasi semua fungsi pembuat job.
- Identifikasi semua fungsi pengubah status job.
- Identifikasi semua penulis database.
- Identifikasi query yang menyebabkan lock.
- Petakan hubungan channel, task, file, dan output.
- Cari kemungkinan deduplication key yang sudah tersedia.
- Tentukan apakah `operational_tasks` dapat diperluas.
- Tentukan migration yang backward-compatible.

**Deliverables**

- `DATABASE_SCHEMA_AUDIT_V2.md`
- `DATABASE_WRITER_MAP_V2.md`
- `DATABASE_MIGRATION_PLAN_V2.md`

**Acceptance criteria**

- Satu authoritative job table telah ditentukan.
- Tidak ada job system paralel.
- Semua status dan transisi lama dipahami.
- Migration dapat dilakukan tanpa kehilangan data.

## P1.4 — Resource budget audit

Modal acuan:

| Worker    | CPU                                 | RAM         | Disk          |
| --------- | ----------------------------------- | ----------- | ------------- |
| Publish   | 2–3 core                            | 4 GB        | 60 GB + bahan |
| Ingestion | sekitar 1 core sebagai modal desain | 2 GB        | 100 GB        |
| Stage 2   | 1 core/render; cgroup max 4 core    | 4 GB        | 50 GB         |
| Live      | 1–3 core                            | 1 GB        | 5 GB          |
| Web/API   | target <0,2 core                    | ukur aktual | ukur aktual   |

Catatan:

- Ingestion 1 core / 2 GB adalah modal perencanaan.
- RAM available ≥4 GB merupakan gate yang berbeda.
- Angka ini bukan konsumsi CPU/RAM aktual yang sudah terbukti.

Checklist:

- Ukur ingestion aktual.
- Ukur Stage 2 aktual.
- Ukur live aktual.
- Ukur publish aktual.
- Ukur web/API aktual.
- Ukur peak RAM.
- Ukur disk staging.
- Ukur disk cache.
- Ukur network TX.
- Uji Stage 2 concurrency 1.
- Uji Stage 2 concurrency 2.
- Siapkan uji concurrency 3 dan 4 hanya jika aman.

**Acceptance criteria**

- Modal dan konsumsi aktual dibedakan.
- Tidak ada CPU quota final yang ditetapkan tanpa benchmark.
- Headroom RAM dan disk terukur.

---

# 3. TRACK P2 — Process Separation dan Local Resource Governor

**Prioritas:** HIGH
**Tujuan:** menghilangkan kompetisi tidak terkendali antarworkload di Master.

## P2.1 — Split publish dan prep

- Identifikasi entry point publish.
- Identifikasi entry point prep.
- Pisahkan menjadi dua systemd unit.
- Pisahkan log.
- Pisahkan restart policy.
- Pisahkan environment/configuration.
- Pisahkan health check.
- Pastikan web tidak membuat child FFmpeg.
- Pastikan restart prep tidak mematikan publish.
- Pastikan restart publish tidak mematikan prep.
- Uji reboot server.

**Acceptance criteria**

- Dua unit berjalan secara independen.
- Restart salah satu unit tidak menghentikan unit lainnya.
- CPU web target <0,2 core.
- Tidak ada child FFmpeg dari web/API.

## P2.2 — systemd cgroup controls

- Buat konfigurasi resource control untuk web/API.
- Buat konfigurasi resource control publish.
- Buat konfigurasi resource control prep.
- Terapkan `CPUWeight`.
- Terapkan `CPUQuota` jika benchmark mendukung.
- Terapkan `MemoryHigh`.
- Terapkan `MemoryMax`.
- Terapkan `IOWeight`.
- Terapkan `TasksMax`.
- Verifikasi nilai efektif melalui systemd/cgroup.
- Uji kondisi CPU pressure.
- Uji kondisi RAM pressure.
- Uji proses mencapai batas.
- Uji restart dan rollback unit.

**Acceptance criteria**

- Publish mendapat prioritas sesuai desain.
- Prep tidak dapat menghabiskan seluruh resource Master.
- Web/API tidak terpengaruh proses FFmpeg.
- Batas efektif terbukti, bukan hanya konfigurasi tertulis.

## P2.3 — Local governor

- Petakan Resource Guard v2 ke local governor.
- Tentukan metrik lokal per box.
- Tentukan threshold load.
- Tentukan threshold RAM.
- Tentukan threshold disk.
- Tentukan threshold TX.
- Tentukan aksi admission stop.
- Tentukan aksi concurrency reduction.
- Tentukan aksi pause upload.
- Tentukan aksi pause Stage 2.
- Pastikan tidak ada remote guard yang mematikan workload box lain.
- Tambahkan reason code untuk setiap keputusan.
- Tambahkan hysteresis agar governor tidak bolak-balik.
- Uji load tinggi dan pemulihan normal.

**Acceptance criteria**

- Setiap governor mengendalikan resource lokalnya.
- Live mempunyai prioritas tertinggi pada Live-01.
- Semua tindakan governor dapat diamati.

---

# 4. TRACK P3 — Job Deduplication dan Churn Reduction

**Prioritas:** CRITICAL
**Tujuan:** menghentikan pekerjaan terbuang akibat job SUPERSEDED.

## P3.1 — Audit sumber churn

- Audit sweeper.
- Audit Fleet Runner.
- Audit retry handler.
- Audit scheduler.
- Audit manual trigger.
- Audit status SUPERSEDED.
- Petakan urutan event untuk satu channel.
- Identifikasi job baru yang dibuat ketika job lama masih RUNNING.
- Identifikasi retry yang salah dianggap job baru.
- Buat contoh reproduksi churn.

## P3.2 — Deduplication key

- Tentukan canonical `channel_id`.
- Tentukan `job_kind`.
- Tentukan aturan generation.
- Definisikan `deduplication_key`.
- Tentukan status yang dianggap aktif.
- Tentukan unique constraint untuk satu job aktif.
- Buat migration.
- Uji request job berulang.
- Uji dua scheduler bersamaan.
- Uji retry.
- Uji generation baru.
- Uji job lama yang terlambat selesai.

Format konseptual:

`channel_id + job_kind + generation`

**Acceptance criteria**

- Satu active job per logical key.
- Retry tidak membuat generation baru.
- Generation baru hanya dibuat bila input memang berubah.
- Churn SUPERSEDED turun dari 66% menjadi <20% dalam window 7 hari.

## P3.3 — Sweeper dan throughput

- Audit batch sweeper.
- Uji batch 2.
- Uji batch 5.
- Ukur throughput channel/menit.
- Ukur beban database.
- Ukur jumlah job duplikat.
- Pastikan tidak ada starvation.
- Pastikan guard menahan hanya workload lokal.
- Verifikasi backlog 412 channel dan sekitar 28.461 item diproses sesuai admission policy.

**Acceptance criteria**

- Target feed sekitar 2,5 channel/menit dapat diuji.
- Job aktif tidak digandakan.
- Tidak menambah SQLite lock secara signifikan.

---

# 5. TRACK P4 — API-based Orchestration Minimum

**Prioritas:** HIGH
**Tujuan:** worker remote mengambil job secara aman tanpa akses langsung ke SQLite.

## P4.1 — API contract

- Definisikan worker registration endpoint.
- Definisikan worker heartbeat endpoint.
- Definisikan job claim endpoint.
- Definisikan job start endpoint.
- Definisikan job completion endpoint.
- Definisikan job failure endpoint.
- Definisikan cancellation endpoint.
- Definisikan worker status endpoint.
- Definisikan API error code.
- Definisikan request timeout.
- Definisikan idempotency behavior.
- Definisikan protocol version.
- Dokumentasikan OpenAPI contract.

## P4.2 — Worker registry

- Definisikan `worker_id`.
- Definisikan `worker_type`.
- Definisikan hostname.
- Definisikan capabilities.
- Definisikan capacity.
- Definisikan status.
- Definisikan release ID.
- Definisikan schema version.
- Definisikan protocol version.
- Implementasikan register.
- Implementasikan heartbeat.
- Implementasikan stale worker detection.
- Uji worker reconnect.

## P4.3 — Claim dan lease

- Tentukan lease duration.
- Tentukan heartbeat interval.
- Tentukan stale threshold.
- Implementasikan atomic claim.
- Implementasikan lease renewal.
- Implementasikan lease expiry.
- Implementasikan recovery.
- Pastikan worker hanya claim saat kapasitas lokal tersedia.
- Pastikan Master tidak mengirim job secara push.
- Uji worker crash setelah claim.
- Uji worker crash ketika RUNNING.
- Uji Master restart.

## P4.4 — Fencing token

- Tentukan sumber monotonik token.
- Naikkan token setiap ownership berpindah.
- Simpan token pada job.
- Simpan token pada worker claim.
- Validasi token pada heartbeat.
- Validasi token pada completion.
- Tolak hasil dengan token lama.
- Catat `FENCING_REJECTED`.
- Uji worker lama kembali setelah lease expired.

## P4.5 — Idempotency dan output

- Definisikan idempotency key per operasi.
- Buat temporary output directory unik.
- Validasi output sebelum publish.
- Validasi generation.
- Validasi fencing token.
- Implementasikan atomic promotion.
- Implementasikan reconciliation untuk upload YouTube.
- Pastikan hasil lama tidak menimpa hasil baru.
- Uji completion request yang terkirim dua kali.
- Uji worker mati setelah output dibuat tetapi sebelum completion.

## P4.6 — Job events

- Tentukan event type.
- Tentukan metadata yang aman disimpan.
- Catat claim.
- Catat heartbeat.
- Catat lease expiry.
- Catat retry.
- Catat failure reason.
- Catat fencing rejection.
- Catat completion.
- Catat cancellation.
- Buat query audit per job.
- Pastikan event tidak menyimpan secret.

**Acceptance criteria P4**

- Satu worker dapat register.
- Worker dapat claim satu job.
- Worker dapat heartbeat.
- Worker dapat menyelesaikan job.
- Worker crash dapat dipulihkan.
- Stale worker ditolak.
- Duplicate request tidak menggandakan side effect.
- Semua transisi dapat ditelusuri.

---

# 6. TRACK P5 — Retry, Rate Limit, dan External API Safety

**Prioritas:** HIGH

## P5.1 — Error classification

- Klasifikasikan network timeout.
- Klasifikasikan HTTP 5xx.
- Klasifikasikan HTTP 429.
- Klasifikasikan quotaExceeded.
- Klasifikasikan invalid credential.
- Klasifikasikan permission denied.
- Klasifikasikan invalid input.
- Klasifikasikan permanent failure.
- Definisikan retryable vs non-retryable.
- Definisikan batas attempt.
- Definisikan exponential backoff.
- Tambahkan jitter.
- Hormati `Retry-After`.
- Implementasikan DEAD_LETTER.

## P5.2 — Google Drive P2

- Catat project dan credential.
- Catat endpoint dan method.
- Catat request rate.
- Catat quota error.
- Catat response code.
- Uji worker dari satu IP tambahan.
- Bandingkan dengan baseline.
- Tentukan rate limit per project/credential.
- Pastikan retry tidak memperparah kuota.
- Dokumentasikan hasil uji.

## P5.3 — D1 ingestion

- Catat status eskalasi Dahono.
- Uji proxy alternatif secara terbatas jika disetujui.
- Ukur success rate.
- Ukur latency.
- Ukur biaya per GB.
- Uji stabilitas session.
- Pastikan credential tidak masuk log.
- Pastikan penggunaan mematuhi ketentuan provider dan platform.
- Bandingkan hasil dengan baseline.

**Acceptance criteria**

- Error quota tidak masuk retry loop.
- Error permanen tidak menghabiskan attempt.
- Retry transient tidak membuat job duplikat.
- P2 dan D1 mempunyai hasil pengujian terpisah.

---

# 7. TRACK P6 — VPS 8 Readiness dan Financial Gate

**Prioritas:** HIGH, tetapi pembelian menunggu gerbang.

## P6.1 — Verifikasi biaya

- Buka panel Contabo.
- Verifikasi harga Master aktual.
- Verifikasi harga VPS 8 aktual.
- Verifikasi pajak.
- Verifikasi setup fee.
- Verifikasi periode billing.
- Verifikasi cancellation/withdrawal policy.
- Hitung total bulanan.
- Pastikan total tidak melewati batas $34,67/bulan yang disetujui.
- Catat bukti harga dan tanggal verifikasi.

## P6.2 — Storage economics

- Tentukan object storage provider.
- Hitung database size.
- Hitung pertumbuhan harian.
- Hitung aset kritis.
- Tentukan retention harian.
- Tentukan retention mingguan.
- Hitung egress.
- Hitung request cost.
- Hitung Litestream cost.
- Hitung Restic cost.
- Hitung total storage bulanan.
- Dapatkan persetujuan biaya.

## P6.3 — Disk reconciliation

- Audit ingestion staging.
- Audit Stage 2 working directory.
- Audit live assets.
- Audit DR replica.
- Audit OS dan logs.
- Audit cache.
- Hitung free space minimum.
- Pastikan headroom minimal 50 GB.
- Tetapkan quota per workload.
- Tetapkan cleanup retention.

## P6.4 — Network benchmark

- Verifikasi port aktual.
- Ukur throughput.
- Ukur latency ke YouTube.
- Ukur TX saat live.
- Ukur TX saat upload Drive.
- Uji kombinasi live + prep.
- Uji throttle.
- Uji admission control 70% port.
- Uji prep pause ketika live TX melewati 50% port.
- Catat hasil.

## P6.5 — Pembelian gate

VPS 8 hanya boleh dibeli jika:

- Fase 0 lolos.
- Demand mendekati 12–16 stream; atau enam stream key terisi dan antrean channel siap-live ≥10.
- Harga Master telah diverifikasi.
- Total biaya ≤$34,67/bulan.
- Storage dan backup cost sudah dihitung.
- Rollback plan tersedia.
- Owner menyetujui pembelian.

---

# 8. TRACK P7 — VPS 8 Provisioning dan Canary

**Prioritas:** setelah P6 lulus.

## P7.1 — Provisioning

- Provision VPS 8.
- Verifikasi CPU/RAM/disk.
- Verifikasi bandwidth.
- Update OS.
- Buat user service non-root.
- Konfigurasi SSH.
- Konfigurasi firewall.
- Join Tailscale.
- Verifikasi koneksi privat.
- Deploy release manifest.
- Deploy worker agent.
- Deploy live service.
- Deploy local governor.
- Konfigurasi systemd.
- Konfigurasi log rotation.

## P7.2 — Live canary

- Pilih 1 stream.
- Pindahkan ke Live-01.
- Verifikasi RTMP direct ke YouTube.
- Verifikasi FFmpeg.
- Verifikasi broadcast state.
- Verifikasi stream progress.
- Verifikasi reconnect.
- Verifikasi TX governor.
- Verifikasi Master tetap responsif.
- Observasi 72 jam.
- Lanjutkan ke 1–2 stream tambahan jika gate lulus.
- Migrasikan sisanya secara bertahap.

**Acceptance criteria**

- Live stabil 72 jam.
- Tidak ada duplicate supervisor.
- TX governor menolak stream baru ketika batas tercapai.
- Master tidak lagi menjalankan live reguler setelah migrasi lengkap.

## P7.3 — Prep worker canary

- Register Live-01 sebagai prep worker.
- Uji satu ingestion job.
- Uji satu Stage 2 job.
- Mulai dengan dua slot Stage 2.
- Verifikasi local CPU/RAM/disk guard.
- Verifikasi TX throttle.
- Verifikasi Drive handoff.
- Verifikasi job events.
- Verifikasi fencing.
- Verifikasi idempotency.
- Uji worker crash.
- Uji Master disconnect.
- Observasi 72 jam.

**Acceptance criteria**

- Tidak ada gangguan live.
- Tidak ada duplicate job.
- Hasil worker diterima melalui API.
- SQLite tetap hanya berada di Master.

## P7.4 — Scale preparation worker

- Naikkan ingestion concurrency secara bertahap.
- Ukur ingestion throughput.
- Naikkan Stage 2 dari dua ke tiga slot bila aman.
- Uji empat slot hanya jika metrik mendukung.
- Ukur CPU time per klip.
- Ukur wall time per klip.
- Ukur disk I/O.
- Ukur network TX.
- Pastikan live tetap mendapat prioritas.
- Catat throughput sebelum dan sesudah.

---

# 9. TRACK P8 — Backup, Monitoring, dan Recovery

## P8.1 — External watchdog

- Pilih Healthchecks.io cloud atau UptimeRobot.
- Konfigurasi Master heartbeat.
- Konfigurasi Live-01 heartbeat.
- Konfigurasi job progress heartbeat.
- Konfigurasi Telegram alert.
- Uji Master down.
- Uji Live-01 down.
- Uji heartbeat terlambat.
- Uji alert recovery.

## P8.2 — Business health

- Cek publish progress.
- Cek job RUNNING terlalu lama.
- Cek encoder process.
- Cek FFmpeg progress.
- Cek RTMP connection.
- Cek YouTube broadcast state.
- Cek disk staging.
- Cek live TX.
- Cek SQLite lock.
- Cek worker stale.
- Cek backup freshness.

## P8.3 — Prometheus stack

- Tentukan apakah monitoring host metrics sudah membutuhkan Prometheus.
- Tentukan lokasi Prometheus.
- Hitung RAM.
- Hitung disk retention.
- Deploy Node Exporter.
- Deploy Prometheus.
- Uji scrape.
- Uji alert.
- Tambahkan Grafana hanya jika dashboard historis diperlukan.

Node Exporter tidak dianggap monitoring lengkap sebelum ada collector.

## P8.4 — Backup

- Tentukan backend object storage.
- Buat repository Restic.
- Konfigurasi encryption.
- Konfigurasi retention.
- Uji backup SQLite konsisten.
- Evaluasi Litestream pada salinan database.
- Periksa dampak tabel `_litestream_lock`.
- Aktifkan replikasi setelah pengujian.
- Backup aplikasi dan konfigurasi kritis.
- Backup secret dengan mekanisme aman.
- Uji restore.
- Ukur RPO.
- Ukur RTO.
- Dokumentasikan restore runbook.

**Acceptance criteria**

- Backup berada di luar kedua VPS.
- Restore berhasil.
- RPO dan RTO dibuktikan, bukan diasumsikan.
- Backup tidak mengganggu live.

---

# 10. TRACK P9 — Deployment Compatibility

- Definisikan `release_id`.
- Definisikan `git_commit`.
- Definisikan `schema_version`.
- Definisikan `worker_protocol_version`.
- Tambahkan compatibility check.
- Worker melaporkan versi saat register.
- Master menolak job yang tidak kompatibel.
- Tambahkan maintenance mode.
- Tambahkan drain mode.
- Tambahkan rolling deployment.
- Uji worker versi lama dengan Master baru.
- Uji worker versi baru dengan Master lama jika didukung.
- Uji rollback release.
- Uji migration expand-and-contract.

**Acceptance criteria**

- Tidak ada job dikirim ke worker inkompatibel.
- Deployment tidak memutus semua live bersamaan.
- Rollback release dapat dilakukan tanpa merusak schema.

---

# 11. TRACK P10 — Scorecard dan Final Acceptance

Gunakan baseline historis 21–22 November sebagai referensi. Window harus sama dengan metrik yang dibandingkan.

| Metrik            | Baseline   | Target Fase 0 | Target Fase 1    |
| ----------------- | ---------- | ------------- | ---------------- |
| Publish/hari      | 206        | ≥200          | ≥200             |
| FAILED/BLOCKED    | 0          | 0             | 0                |
| Publish delay p90 | 30,2 menit | ≤30 menit     | ≤25 menit        |
| Encoder death     | 5/24 jam   | ≤1/24 jam     | 0/72 jam         |
| Encoder orphan    | 1          | 0             | 0                |
| SQLite lock       | 31/24 jam  | <20           | <10              |
| Master load       | 3,5–4,2    | <3,0          | <2,5             |
| Web CPU           | 1,02 core  | <0,2          | <0,2             |
| Ingestion RUNNING | 2          | ≥3            | Sesuai kapasitas |
| Churn             | 66%        | <20%          | <20%             |
| Master disk peak  | 72%        | ≤70%          | ≤70%             |
| Healthz p95       | ±10 ms     | <100 ms       | <100 ms          |

## Final gate

- Semua metrik kritis tidak memburuk.
- Tidak ada error class baru selama 72 jam.
- Metrik 7 hari dievaluasi dengan window 7 hari.
- Live stabil.
- Publish stabil.
- Job tidak hilang.
- Job tidak menghasilkan side effect ganda.
- Backup dapat direstore.
- Rollback sudah diuji.
- Biaya aktual diketahui.
- Runbook operasional tersedia.
- Owner menyetujui hasil.

---

# 12. Dependency Order dan Parallel Execution

Pekerjaan dapat berjalan paralel, tetapi dependency berikut tidak boleh dilanggar.

| Workstream              | Bisa mulai                      | Bergantung pada             |
| ----------------------- | ------------------------------- | --------------------------- |
| P0 KEIN                 | Segera                          | Keputusan provider          |
| P0 Encoder              | Segera                          | Log dan bukti insiden       |
| P0 Resource Guard audit | Segera                          | Akses implementasi saat ini |
| P0 Janitor              | Segera                          | Backup dan audit file       |
| P1 Baseline             | Segera                          | Akses metrics               |
| P1 Schema audit         | Segera                          | Backup database             |
| P2 Process separation   | Setelah inventory               | P1 service inventory        |
| P2 cgroups              | Setelah baseline                | P1 resource audit           |
| P3 Dedup                | Setelah schema audit            | P1 schema                   |
| P4 API orchestration    | Setelah authoritative job model | P1 schema                   |
| P5 Retry policy         | Paralel dengan P4               | Error classification        |
| P6 VPS readiness        | Paralel dengan P1–P5            | Harga dan biaya             |
| P7 Canary               | Setelah P4 minimum + P6 gate    | API, security, budget       |
| P8 Monitoring           | Paralel dengan P6/P7            | External endpoint           |
| P9 Compatibility        | Sebelum worker production       | Release manifest            |
| P10 Final acceptance    | Setelah canary                  | Semua gate                  |

---

# 13. Implementation Ticket Template

Gunakan format ini untuk setiap task engineering.

## Task ID: V2-XXX

**Judul:**
**Prioritas:** P0 / P1 / P2 / P3
**Workstream:**
**Owner:**
**Status:** NOT STARTED / IN PROGRESS / BLOCKED / REVIEW / DONE

### A. Masalah

Jelaskan kondisi saat ini dan bukti.

### B. Target

Jelaskan kondisi yang ingin dicapai.

### C. Prasyarat

- Backup tersedia
- Baseline tersedia
- Dependency siap
- Owner menyetujui perubahan

### D. Scope perubahan

Daftar file, modul, konfigurasi, tabel, dan systemd unit yang akan diubah.

### E. Implementasi

- Perubahan kode
- Perubahan schema
- Perubahan konfigurasi
- Perubahan systemd
- Logging dan metrics

### F. Test

- Unit test
- Integration test
- Failure test
- Regression test
- Canary test

### G. Acceptance criteria

Tuliskan hasil yang dapat diukur.

### H. Rollback

Tuliskan langkah mengembalikan kondisi sebelumnya.

### I. Post-deployment observation

- 1 jam
- 24 jam
- 72 jam
- 7 hari jika metrik memerlukan window 7 hari

### J. Evidence

- Commit ID:
- Release ID:
- Log:
- Metrics:
- Test report:
- Rollback result:

### K. Final decision

- ACCEPT
- ROLLBACK
- NEEDS MORE OBSERVATION

---

# 14. First Execution Batch — 10 Task Pertama

Urutan yang saya rekomendasikan untuk mulai bekerja:

| ID     | Task                                    | Prioritas | Dependency            |
| ------ | --------------------------------------- | --------- | --------------------- |
| V2-001 | KEIN dependency/fallback decision       | P0        | Tidak ada             |
| V2-002 | Encoder 706 root-cause investigation    | P0        | Log                   |
| V2-003 | Resource Guard v2 audit                 | P0        | Source implementation |
| V2-004 | Janitor safety audit                    | P0        | DB/file inventory     |
| V2-005 | Baseline snapshot                       | P1        | Metrics               |
| V2-006 | Service inventory                       | P1        | Server access         |
| V2-007 | Database schema audit                   | P1        | Consistent backup     |
| V2-008 | Resource budget benchmark               | P1        | Baseline              |
| V2-009 | VPS 8 price + storage cost verification | P1        | Panel/storage quote   |
| V2-010 | Deduplication design proposal           | P1        | Schema audit          |

**Aturan:** V2-010 belum boleh melakukan migration sebelum hasil V2-007 disetujui.

---

# 15. Definition of Done — V2 Implementation

V2 dinyatakan selesai jika:

- Master menjadi satu-satunya authoritative control plane.
- SQLite hanya berada di Master.
- Publish, prep, dan live terisolasi.
- Resource Guard mempunyai eksekutor nyata.
- Worker dapat register dan melaporkan capabilities.
- Worker mengambil job melalui pull API.
- Deduplication mencegah job aktif ganda.
- Lease dan fencing mencegah stale ownership.
- Idempotency mencegah side effect ganda.
- Job events menyediakan audit trail.
- Version compatibility melindungi deployment.
- Live tetap berjalan saat Master terputus.
- Prep tidak mengganggu live.
- Janitor tidak menghapus file valid.
- Monitoring mendeteksi business failure.
- Backup off-site berhasil direstore.
- RPO/RTO sudah dibuktikan.
- Scorecard memenuhi target.
- Runbook recovery dan rollback tersedia.
- Biaya operasional aktual diketahui.

**Prinsip penutup:**

Jangan mengukur keberhasilan dari banyaknya task yang selesai diimplementasikan. Ukur dari berkurangnya kegagalan, hilangnya pekerjaan duplikat, terjaganya live, dan kemampuan sistem pulih tanpa kehilangan hasil kerja.

**END — IMPLEMENTATION CHECKLIST V2.0**

---

## Urutan kerja yang disarankan

Jangan langsung membagi seluruh checklist ke banyak agent. Mulai dengan **Batch 1: V2-001 sampai V2-010**. Ada tiga jalur paralel yang aman:

- **Operasional:** KEIN, encoder 706, janitor.
- **Audit teknis:** Resource Guard v2, baseline, service inventory, schema.
- **Persiapan infrastruktur:** verifikasi harga, storage, dan capacity benchmark.

Setelah V2-007 selesai, baru putuskan perubahan schema dan mulai V2-010 secara teknis. Ini mencegah kita membangun sistem antrean kedua di atas `operational_tasks` yang mungkin sebenarnya sudah dapat diperluas.

**Keputusan paling penting:** VPS 8 readiness boleh berjalan paralel, tetapi pembelian dan pemindahan workload tetap menunggu gate. Plan sumber menetapkan pembelian saat demand mendekati 12–16 stream atau enam stream key terisi dengan antrean siap-live minimal 10, serta total biaya tidak melampaui $34,67/bulan.

Riset repository juga harus tetap diperlakukan sebagai referensi: Huey, Litestream, Restic, Node Exporter, dan tool lain belum otomatis menjadi dependency production. Semua adopsi harus melalui uji terpisah.

Langkah lanjutan yang paling tepat adalah mengubah 10 task pertama ini menjadi **implementation tickets per file/module**, dimulai dari V2-001 dan V2-002, tanpa melakukan perubahan production sebelum bukti pra-perubahan tersedia.
