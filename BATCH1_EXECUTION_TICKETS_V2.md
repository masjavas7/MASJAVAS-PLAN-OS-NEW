# BATCH 1 EXECUTION TICKETS — MASJAVAS TUBE OS V2

**Dokumen acuan (frozen):** `MASTER_ARCHITECTURE_MASJAVAS_TUBE_OS_V2.md` (Draft Baseline),
`IMPLEMENTATION_CHECKLIST_MASJAVAS_TUBE_OS_V2.md` (v1.0)
**Versi:** 1.7 — V2-007 diperluas Cycle 8, 22 Nov 22:50 WIB (cron local-only)
**Versi:** 1.6 — V2-006 diperluas Cycle 7, 22 Nov 18:50 WIB (cron local-only)
**Versi:** 1.5 — V2-005 diperluas Cycle 6, 22 Nov 14:50 WIB (cron local-only)
**Versi:** 1.4 — status memakai kosakata direktif (QUEUED → NOT_STARTED); sumber status resmi: STATUS_REGISTRY.yaml
**Versi:** 1.3 — V2-001 + V2-002 + V2-003 + V2-004 lengkap; V2-005–V2-010 stub antrean
**Tanggal:** 22 November 2026 (v1.1: V2-002 diperluas Cycle 2, 06:50 WIB; v1.3: V2-004 diperluas Cycle 5, 10:50 WIB)
**Status:** Batch 1 = AUDIT / READ-ONLY. Tiket-tiket ini adalah artefak TURUNAN —
tidak mengubah checklist frozen, tidak mengubah urutan dependency, tidak menganggap
P2+ siap dieksekusi. Setiap tiket mencantumkan file/module, pertanyaan teknis,
perintah audit, bukti, kriteria lulus/gagal, dampak gagal, batas tindakan, dan
pihak penyetuju — sesuai format §13 checklist.

**Aturan batch:**

- Audit-only: tidak ada perubahan schema, kode, systemd, kredensial, atau job.
- L1 (diagnostik baca-saja) otomatis; L2/L3 tetap butuh persetujuan sesuai mandat.
- Satu tiket selesai = bukti terlampir + keputusan lulus/gagal tercatat, bukan
  sekadar "sudah diperiksa".
- V2-010 tidak boleh migration sebelum V2-007 disetujui (aturan checklist §14).

---

# TIKET V2-001 — KEIN DEPENDENCY / FALLBACK DECISION (AUDIT)

**Judul:** Audit dependensi KEIN + keputusan fallback sebelum expiry 23 Nov 2026
**Prioritas:** P0 — CRITICAL (tenggat besok)
**Workstream:** P0 Operational Stabilization (§1 checklist P0.1)
**Owner:** Muse (audit) — keputusan provider: user
**Status:** EXECUTED (audit selesai 22 Nov 2026 05:20 WIB; Q5/Q6 DITUTUP
atas instruksi user 06:55 WIB — "ASUMSIKAN KEIN AMAN". Asumsi ini DICATAT,
BUKAN terverifikasi: sisa kuota + jam expiry + harga top-up tetap unknown
karena dashboard terhalang Turnstile. Bila asumsi ini kelak terbukti salah,
tiket dapat dibuka kembali dari bukti yang sudah tersimpan.)
**Dibuat:** 22 Nov 2026, 05:1x WIB
**Dependency:** tidak ada (bisa mulai segera)
**Diblokir oleh:** tidak ada — membutuhkan akses baca VPS (SSH Tailscale) +
  akses baca dashboard KEIN (browser)

## A. Masalah

Plan KEIN berakhir **23 November 2026 (besok)**. LoopGen Tahap 1 (gambar via
model `kein_image`) dan Tahap 2 Opsi 1/2 (video via OMNI 1.1) bergantung pada
KEIN. Tanpa fallback yang sudah diuji, LoopGen Tahap 1/2 berhenti saat plan
kedaluwarsa. Top-up ditunda user sejak 12 Nov ("diputuskan NANTI, setelah
arsitektur siap") — arsitektur V2.0 kini dibekukan, jadi keputusan jatuh tempo
hari ini.

**Fakta terverifikasi (bukan asumsi):**

| Fakta | Nilai | Sumber |
|---|---|---|
| Akun KEIN | `user_c4fb2943` | MEMORY.md |
| Plan | 24 Okt – 23 Nov 2026; kuota 5.550 | MEMORY.md |
| Model video | OMNI 1.1, text-only, ratio 16:9 | `custom.kein/CONNECTOR.md` |
| Model gambar | `kein_image`; uji 14 Nov: 78,7 dtk/gambar, PNG 2752×1536 | PROACTIVE_PREFERENCES.md |
| Throughput | Serial 1 task (semua playlist) | MEMORY.md |
| Rate limit provider | 10 req/menit; 10 slot paralel (provider) | `custom.kein/CONNECTOR.md` |
| Riwayat task | Hanya 24 jam — task lebih tua tidak bisa dicek/unduh | `custom.kein/CONNECTOR.md` |
| Endpoint kuota | TIDAK ADA — kuota hanya bisa dibaca dari dashboard | `custom.kein/CONNECTOR.md` |
| Idempotency | Header `Idempotency-Key`; di body server tolak 422 tanpa task | `kein_api_loop_pack/SKILL.md` |
| Kunci VPS | Hanya via OWNER-only encrypted dashboard config; kunci konektor Muse TIDAK bisa dipakai VPS | MEMORY.md |
| Opsi 3 workers | Muse AI (Satu Digital, LUTHFI, AGUNG) — jalur terpisah dari KEIN | MEMORY.md |
| Cron 5-menit | DISABLED sampai top-up + 1 live test | MEMORY.md |
| Kebijakan submit | Submit dari turn proactivity handoff DITOLAK 403 `proactivity_read_only_preparation` — audit/submit hanya dari main/side chat | TOOLS.md |

**Dua pekerjaan berbeda (jangan dicampur):**

1. **Keputusan bisnis/operasional** — pilih fallback (atau top-up) dari harga
   yang sudah tersedia. Ini keputusan user.
2. **Audit teknis** — temukan seluruh pemanggilan KEIN, dependensi, jalur
   kegagalan, dan perilaku saat kredensial/layanan kedaluwarsa. Ini isi tiket ini.

Memilih fallback belum otomatis berarti sistem aman dialihkan.

## B. Target

1. Inventaris lengkap pemanggilan KEIN (file + fungsi + model + jalur).
2. Daftar job KEIN in-flight + status checkpoint/resume masing-masing.
3. Perilaku sistem saat KEIN mengembalikan 401/403/expired terdokumentasi
   dari KODE (bukan dugaan): retry? park? fail? bakar kuota?
4. Sisa kuota + waktu expiry pasti + throughput yang masih mungkin sebelum expiry.
5. Konfirmasi apakah Opsi 3 sepenuhnya independen dari KEIN.
6. Matriks keputusan top-up vs fallback vs HOLD (biaya + kualitas + waktu).
7. Desain mode HOLD yang aman (hentikan submit baru tanpa membunuh job
   berjalan, tanpa duplikat saat resume).

## C. Prasyarat

- [ ] Akses SSH baca ke VPS Master (Tailscale, alias `masjavas-vps`).
- [ ] Akses baca dashboard KEIN (`kein.my.id`) — via browser; bila butuh login,
      user yang login (kredensial TIDAK diminta di chat).
- [ ] Konektor `custom.kein` tersedia (status check saja — tanpa kuota).
- [ ] Dokumen harga fallback 18 Nov tersedia:
      `engineering/docs/BIAYA_GENERATE_IMAGE_700_1000_CHANNEL.md`,
      `engineering/docs/REPO_GITHUB_GENERATE_IMAGE_API_KEY.md`.
- [ ] Tidak ada submit KEIN yang sedang berjalan dari lane lain saat audit
      (koordinasi: audit ini read-only, tapi submit bersamaan mengubah angka).

## D. Scope pemeriksaan (file/module)

| # | Area | Lokasi / cara periksa |
|---|---|---|
| D1 | Pemanggilan KEIN di kode VPS | `grep -rn "kein" /opt/masjavas-os --include="*.py"` (baca-saja); catat file + baris + fungsi + model yang dipakai |
| D2 | Jalur Tahap 1 (gambar) | Modul LoopGen Tahap 1 — provider KEIN vs Gemini; kapan `kein_image` dipilih; fallback yang sudah ada di kode (bila ada) |
| D3 | Jalur Tahap 2 (video) | Modul LoopGen Tahap 2 Opsi 1/2 — pemanggilan OMNI 1.1; bedakan dari Opsi 3 (agent Muse AI) |
| D4 | Kredensial VPS | Keberadaan + path config terenkripsi SAJA — JANGAN tampilkan nilai key, JANGAN buka isi config terenkripsi |
| D5 | Job in-flight | DB read-only: `loopgen_jobs`, `loopgen_agent_tasks`, `loopgen_stage2_runs` — hitung per status + provider; catat task_id KEIN terbaru (ingat: riwayat hanya 24 jam) |
| D6 | Error handling | Kode penanganan 401/403/429/expired di jalur KEIN — retry? backoff? park? dead-letter? |
| D7 | Checkpoint/resume | Apakah tiap job menyimpan checkpoint (prompt, idempotency key, task_id, output parsial)? Bisakah resume tanpa duplikat? |
| D8 | Konektor Muse | `custom.kein/bin/kein.py status <task_id>` pada SATU task terbaru (bukti akses, tanpa kuota) |
| D9 | Dashboard KEIN | Sisa kuota + tanggal/jam expiry + status plan (browser; screenshot/angka) |
| D10 | Dokumen fallback | Harga + batasan teknis tiap opsi (lihat §F.6) — JANGAN riset ulang dari nol |

## E. Pertanyaan teknis yang harus dijawab

| # | Pertanyaan | Bukti yang menjawab |
|---|---|---|
| Q1 | Jalur kode mana saja yang memanggil KEIN? (file + fungsi + model) | Output grep D1 + daftar D2/D3 |
| Q2 | Berapa job KEIN in-flight saat ini (per status + provider)? | Hitungan DB D5 |
| Q3 | Apa yang terjadi saat KEIN mengembalikan 401/403/expired? | Kutipan kode error handling D6 (file:baris) |
| Q4 | Apakah tiap job punya checkpoint + idempotency key yang memungkinkan resume tanpa duplikat? | D7 — ya/tidak per jalur + lokasi penyimpanan key |
| Q5 | Berapa sisa kuota + kapan tepatnya expiry? | D9 dashboard |
| Q6 | Berapa throughput masih mungkin sebelum expiry? (sisa jam × serial 1 task × 78,7 dtk/gambar atau 3–5 mnt/video) | Hitungan dari Q5 |
| Q7 | Apakah Opsi 3 (Muse AI workers) sepenuhnya independen dari KEIN? | Konfirmasi dari D3 + `loopgen_agent_tasks` (provider field) |
| Q8 | Apakah fallback DeepInfra kompatibel dengan pipeline? (prompt, resolusi 1344×768 vs 2752×1536, kelipatan 16, QA frame) | D10 + catatan teknis: native 1080p harus 1920×1088 |
| Q9 | Bagaimana mode HOLD bekerja? (hentikan submit baru, job berjalan lanjut, resume tanpa duplikat) | Desain HOLD di §G (berbasis Q3 + Q4) |
| Q10 | Biaya per keputusan (top-up vs DeepInfra vs Muse Image vs HOLD)? | Matriks §G dari D10 |

## F. Perintah / metode audit (executable, read-only)

**F.1 — Grep pemanggilan KEIN di VPS (baca-saja):**

```bash
ssh -F /home/hatch/.ssh/config masjavas-vps \
  'grep -rn -i "kein" /opt/masjavas-os --include="*.py" | head -100'
```

Catat: file, nomor baris, fungsi, model (`omni-1.1` / `kein_image` / lain).

**F.2 — Hitung job LoopGen in-flight (DB read-only, timeout 30 dtk):**

Fase 2a — temukan file DB + cek CLI (satu panggilan dengan F.1):

```bash
ssh -F /home/hatch/.ssh/config masjavas-vps \
  'find /opt/masjavas-os -maxdepth 3 \( -name "*.db" -o -name "*.sqlite*" \) 2>/dev/null; which sqlite3 || echo NO-SQLITE3-CLI'
```

Fase 2b — hitung per status (mode read-only; ganti `<DB>` dengan path hasil 2a).
Bila `sqlite3` CLI tersedia:

```bash
ssh -F /home/hatch/.ssh/config masjavas-vps \
  'sqlite3 "file:<DB>?mode=ro" "SELECT status, COUNT(*) FROM loopgen_jobs GROUP BY status;"'
```

Bila CLI tidak tersedia, pakai python read-only:

```bash
ssh -F /home/hatch/.ssh/config masjavas-vps \
  'python3 -c "import sqlite3; c=sqlite3.connect(\"file:<DB>?mode=ro\", uri=True, timeout=30); print(list(c.execute(\"SELECT status, COUNT(*) FROM loopgen_jobs GROUP BY status\")))"'
```

Ulangi untuk `loopgen_agent_tasks` (tambah kolom provider bila ada) dan
`loopgen_stage2_runs`. Catat task_id KEIN terbaru (≤24 jam) untuk F.4.
**JANGAN** menjalankan UPDATE/DELETE/INSERT apa pun. Bila DB terkunci
(`database is locked`), tunggu 30 dtk dan coba sekali lagi; bila tetap
terkunci, catat sebagai temuan (jangan paksa).

**F.3 — Periksa error handling + checkpoint di kode (baca file):**

```bash
ssh -F /home/hatch/.ssh/config masjavas-vps \
  'grep -rn -i "401\|403\|unauthorized\|expired\|idempotency" /opt/masjavas-os/worker/loopgen* /opt/masjavas-os/worker/*kein* 2>/dev/null | head -60'
```

Sesuaikan path dengan hasil F.1. Baca fungsi penanganan error sampai tuntas
(jangan simpulkan dari nama fungsi saja).

**F.4 — Bukti akses konektor (tanpa kuota, SATU task terbaru saja):**

```bash
python3 ~/workspace/custom-connectors/custom.kein/bin/kein.py status <task_id_terbaru>
```

Hanya bila ada task_id ≤24 jam dari F.2. Bila tidak ada, catat "tidak ada
task KEIN dalam 24 jam terakhir" — itu juga temuan.

**F.5 — Dashboard KEIN (browser):**

Buka `https://kein.my.id` dashboard, catat: sisa kuota, tanggal/jam expiry,
status plan. Bila butuh login, user yang login (jangan minta kredensial).

**F.6 — Matriks fallback (dari dokumen 18 Nov — JANGAN riset ulang):**

| Opsi | Harga | Catatan teknis |
|---|---|---|
| Top-up KEIN | (cek dashboard/harga plan saat ini) | Jalur tanpa perubahan kode; serial 1 task tetap |
| DeepInfra FLUX-1-schnell | Rp 35,44/gambar (4 step @1344×768); armada penuh Rp 837 rb | Wajib kelipatan 16; native 1080p = 1920×1088; pilot 3 varian Rp 800 sudah ditawarkan, belum dijawab user |
| Muse Image API resmi | Rp 180/gambar | Kualitas + edit/refine |
| Kie.ai Seedream 5.5 | ≈ Rp 7,8 jt/700 ch; Rp 11,2 jt/1.000 ch (buffer redo 15%) | — |
| ModelsLab Unlimited | $149/bln ≈ Rp 2,7 jt flat | — |
| HOLD (tanpa provider) | Rp 0 + opportunity cost | LoopGen Tahap 1/2 berhenti; Opsi 3 jalan terus (bila Q7 = ya) |

**DILARANG dalam tiket ini:**

- `kein.py submit` (membakar kuota) atau submit apa pun ke KEIN.
- Menampilkan, menyalin, atau mencatat nilai API key / token / password.
- Mengubah kode, config, DB, systemd, atau job (UPDATE/DELETE/kill/restart).
- Top-up / pembelian / langganan apa pun (keputusan + aksi user).
- Menjalankan audit dari turn proactivity handoff (submit diblokir 403;
  audit ini read-only, tapi jalankan dari main/side chat agar konsisten).

## G.0 — Evidence manifest (wajib diisi saat eksekusi)

Setiap Q1–Q10 wajib punya satu baris manifest. Tanpa timestamp + artefak
tersimpan, bukti task KEIN (riwayat hanya 24 jam) bisa hilang sebelum
keputusan provider dibuat. Isi manifest BERSAMAAN dengan eksekusi, bukan
di akhir.

| Question ID | Command | Timestamp (WIB) | Exit code | Evidence | Finding | Confidence | Impact |
|---|---|---|---|---|---|---|---|
| Q1 | F.1 grep kein + baca penuh 2 klien | 05:12–05:14 | 0 | `KEIN_CALLSITES_V2-001.md` | 2 klien/2 host, 1 throttle, 4 jalur + 2 fallback agent | Confirmed | Fallback harus mencakup 2 host |
| Q2 | F.2 DB read-only (2 skrip) | 05:15–05:16 | 0 | `KEIN_JOBS_INFLIGHT_V2-001.md` | KEIN idle sejak 17 Nov 16:24; 0 job gambar in-flight; 0 task ID KEIN | Confirmed | Expiry tidak membunuh job in-flight |
| Q3 | F.3 grep + baca classify/recovery/quota-block | 05:13–05:16 | 0 | `KEIN_ERROR_PATHS_V2-001.md` | 4 ErrorKind + rantai quota-block lengkap; AUTH tanpa retry/fallback foto | Confirmed | Expiry 402/403-kuota ditangani; 401 = item gagal |
| Q4 | F.3 + baca checkpoint/resume | 05:14–05:16 | 0 | `KEIN_ERROR_PATHS_V2-001.md` | Video: resume tanpa duplikat; gambar: tanpa task_id, resume via exists_count | Confirmed/Partial | Resume video aman |
| Q5 | F.5 dashboard browser | DB 05:15; browser pending | DB 0 | `KEIN_QUOTA_EXPIRY_V2-001.md` | Plan 24 Okt–23 Nov, kuota awal 5.550; sisa kuota menunggu dashboard | Partial | Keputusan top-up butuh angka dashboard |
| Q6 | Hitungan dari Q5 | 05:20 | — | `KEIN_QUOTA_EXPIRY_V2-001.md` | Teoretis ±860–1.900 gambar; AKTUAL 0 (jalur idle) | Partial | Tidak ada kejar tayang kuota |
| Q7 | F.1 + F.2 + baca agent_*.py | 05:13–05:16 | 0 | `KEIN_JOBS_INFLIGHT_V2-001.md` | Video agent independen; gambar agent jatuh ke KEIN; `loopgen_agent_tasks` kosong | Partial | Opsi 3 bukan pengganti KEIN penuh |
| Q8 | D10 + baca kontrak provider | 05:16–05:20 | — | `KEIN_DECISION_MATRIX_V2-001.md` | Kontrak kompatibel; resolusi/kualitas belum teruji; butuh L3 + pilot | Partial | Pilot DeepInfra Rp 800 sebelum komitmen |
| Q9 | Desain HOLD dari Q3+Q4 | 05:20 | — | `KEIN_DECISION_MATRIX_V2-001.md` | HOLD sudah ada tanpa kode baru; biaya hari ini NOL | Confirmed | HOLD = opsi default aman |
| Q10 | Matriks D10 + Q5/Q6 | 05:20 | — | `KEIN_DECISION_MATRIX_V2-001.md` | Rekomendasi: jangan top-up membabi buta (pemakaian nol) | Partial | Keputusan user menunggu Q5 |

Manifest lengkap + perintah gagal dicatat jujur:
`workspace/engineering/v2-001/EVIDENCE_MANIFEST_V2-001.md`.

Aturan manifest:

- **Confidence:** `Confirmed` (bukti langsung dari kode/DB/dashboard),
  `Partial` (bukti tidak lengkap / hanya satu sumber), `Unknown` (tidak dapat
  diverifikasi + alasannya dicatat). `Unknown` yang eksplisit LEBIH BAIK
  daripada dugaan yang ditulis seolah fakta.
- **Evidence:** path file artefak atau output inline. Bukti TIDAK BOLEH berisi
  nilai kredensial — bila output mengandung key/token, redaksi SEBELUM disimpan.
- **Impact:** satu kalimat — apa arti temuan ini untuk fallback dan HOLD.
- Manifest yang sudah terisi disimpan di `workspace/engineering/v2-001/EVIDENCE_MANIFEST_V2-001.md`
  dan direkap ke tabel di atas saat tiket selesai.

## G. Bukti yang harus dilampirkan (deliverable tiket)

1. `KEIN_CALLSITES_V2-001.md` — daftar Q1 (file:baris + fungsi + model).
2. `KEIN_JOBS_INFLIGHT_V2-001.md` — hitungan Q2 + daftar task_id terbaru.
3. `KEIN_ERROR_PATHS_V2-001.md` — kutipan kode Q3 + jawaban Q4 (checkpoint?).
4. `KEIN_QUOTA_EXPIRY_V2-001.md` — angka Q5 + hitungan Q6.
5. `KEIN_DECISION_MATRIX_V2-001.md` — matriks Q10 + rekomendasi + desain HOLD (Q9).
6. Semua file di `workspace/engineering/v2-001/` (buat foldernya saat eksekusi).

## H. Acceptance criteria (lulus / gagal)

**LULUS bila SEMUA terpenuhi:**

- [ ] Q1–Q10 terjawab dengan bukti (bukan dugaan); yang tidak bisa dijawab
      dicatat eksplisit sebagai "TIDAK DAPAT DIVERIFIKASI + alasannya".
- [ ] Tidak ada job KEIN in-flight yang tidak tercatat status + checkpoint-nya.
- [ ] Error path 401/403/expired dikutip dari kode (file:baris), bukan asumsi.
- [ ] Matriks keputusan lengkap (biaya + kualitas + waktu + risiko per opsi).
- [ ] Desain HOLD siap dieksekusi bila keputusan provider belum ada saat expiry.
- [ ] Nol submit KEIN, nol perubahan kode/DB/config, nol kredensial tercatat.

**GAGAL bila salah satu terjadi:**

- Ada jalur pemanggilan KEIN yang tidak ditemukan (grep tidak lengkap / path salah).
- Job in-flight tidak bisa dihitung (DB tidak terbaca — eskalasi, jangan ditebak).
- Error path tidak ditemukan di kode dan tidak ada yang bisa menjelaskan
  perilaku sistem saat expiry (→ risiko "berhenti tanpa jejak").
- Keputusan provider diminta tanpa matriks biaya/kualitas (→ user memutuskan buta).

## I. Dampak jika gagal / jika tiket tidak dikerjakan

- LoopGen Tahap 1/2 berhenti 23 Nov tanpa rencana — 412 channel backlog
  (±28.461 item) kehilangan jalur suplai gambar/video AI.
- Job in-flight tanpa checkpoint hilang atau duplikat saat fallback dinyalakan.
- Keputusan top-up vs fallback dibuat tanpa angka (risiko bayar mahal untuk
  jalur yang ternyata tidak kompatibel, atau HOLD terlalu lama).
- Opsi 3 (Muse AI) mungkin jadi satu-satunya jalur — kapasitasnya perlu
  dihitung terpisah (di luar tiket ini).

## J. Batas tindakan yang diizinkan

- **BOLEH:** semua perintah §F (read-only); membaca kode/config/DB/dokumen;
  status check 1 task (F.4); membuka dashboard (F.5); menulis 5 dokumen bukti §G.
- **TIDAK BOLEH:** submit/generate KEIN; top-up/beli/langganan; ubah kode/config/
  DB/systemd/job; tampilkan/simpan kredensial; batalkan/mulai-ulang job;
  menghubungi provider atas nama user.
- **Keputusan provider (top-up / fallback / HOLD) = USER.** Tiket ini hanya
  menyajikan bukti + rekomendasi.

## K. Pihak yang menyetujui langkah berikutnya

| Langkah berikut | Penyetuju |
|---|---|
| Top-up KEIN / beli fallback | User (keputusan + pembayaran) |
| Implementasi fallback (kode/config) | User (L3 — perubahan integrasi provider) |
| Aktivasi mode HOLD | User (keputusan operasional; eksekusi L2 bila hanya flag/config non-destruktif) |
| Lanjut ke V2-002 dst. | Tidak butuh persetujuan baru (Batch 1 audit) |

## L. Estimasi & catatan eksekusi

- Estimasi: ±1–2 jam (termasuk tunggu SSH + dashboard).
- Jalankan dari main/side chat (bukan proactivity handoff).
- Tangkap task_id KEIN terbaru SEGERA — riwayat hanya 24 jam; setiap jam
  berlalu, bukti task lama hilang permanen.
- Bila dashboard KEIN butuh login dan user tidak tersedia, kerjakan D1–D4 +
  D6–D8 + D10 dulu; D5/D9 menyusul (tandai sebagai PENDING di bukti).

---

# TIKET V2-002 — ENCODER 706 ROOT-CAUSE INVESTIGATION (AUDIT)

**Judul:** Diagnosis akar penyebab encoder live slot 706 mati mendadak 5×/24 jam + 1 encoder orphan
**Prioritas:** P0 — CRITICAL | **Workstream:** P0.2 (Track A — Stabilization: A1 diagnosis encoder 706, A2 perbaiki orphan process)
**Owner:** Muse (audit) — perbaikan: user (L2/L3, tergantung temuan)
**Status:** READY_FOR_REVIEW (audit selesai 22 Nov 2026 07:10 WIB; 5 dokumen bukti + manifest di `workspace/engineering/v2-002/`; temuan utama: 12 kematian encoder SLOT 114 — bukan 706 — akibat RTMP timeout; + koreksi lintas-tiket: V2-001 mengaudit host yang salah)
**Dibuat:** 22 Nov 2026 (stub) — **Diperluas:** 22 Nov 2026 06:50 WIB (Cycle 2, local-only)
**Dependency:** log insiden (tersedia di VPS — journal live 24 jam + `ps`)
**Diblokir oleh:** tidak ada — membutuhkan akses baca VPS (SSH Tailscale) + baca kode `live_supervisor.py`

## A. Masalah

Encoder stream slot 706 mati mendadak **5× dalam 24 jam** — pola berulang,
bukan satu insiden. Supervisor me-restart otomatis (±15 dtk) setiap kali,
sehingga stream kembali — tetapi akar penyebab tidak diketahui, dan setiap
kematian adalah jeda siaran + risiko stream tidak kembali bila supervisor
sendiri terdegradasi. Terpisah: satu encoder **orphan** berjalan tanpa
pengawas — bila ia mati, tidak ada yang me-restart-nya.

**Fakta terverifikasi (bukan asumsi):**

| Fakta | Nilai | Sumber |
|---|---|---|
| Kematian encoder 706 | 21 Nov 15:08, 15:41, 17:29, 19:23; 22 Nov 02:54 (5×/24 jam) | `EVALUASI_SEBELUM_PERBAIKAN_2026-11-22.md` P16; `VERIFIKASI_CPU_..._2026-11-22.md` §5 |
| Bukti journal | 5× "Proses FFmpeg mati mendadak" untuk stream #1795226249006, journal live 24 jam | P16 |
| Auto-restart supervisor | ±15 dtk setiap kematian | P16; VERIFIKASI_CPU §5 |
| Orphan | PID 1435089, jalan 18 jam 15 mnt, parent PID 1; selamat dari restart supervisor 21 Nov 14:42 | P16; VERIFIKASI_CPU §5 |
| Health check terdegradasi | Traceback DB-locked di `live_supervisor.py:878, 920` (→ `aiosqlite execute`); 26× "database is locked" di journal live 24 jam — iterasi health check terlewat, deteksi encoder mati bisa terlambat | VERIFIKASI_CPU §5 |
| Stream yang jalan | 2 stream live: STAGING-SLOT-706 + STAGING-SLOT-114; ±4,2 Mbps TX per stream | EVALUASI l.254; `RENCANA_VPS_LIVE_2026-11-21.md` l.13 |
| File sumber live | 1080p (1920×1080, 24fps, H.264 0,59–5,05 Mbps), loop ~20 dtk, ~10 MB/file, total 44 MB | `RENCANA_VPS_LIVE_2026-11-21.md` l.13 |
| Aturan frozen | V2.0 §16.4 mewajibkan: identifikasi exit code + signal; simpan stderr FFmpeg; simpan process tree; simpan input file + parameter encoder; cek resource + network; bedakan encoder mati / RTMP putus / broadcast selesai; pastikan tidak ada orphan; uji recovery; tambahkan business health check | `MASTER_ARCHITECTURE_MASJAVAS_TUBE_OS_V2.md` §16.4 |
| Konteks | Pindah box (VPS 8) TIDAK memperbaiki ini — masalah di level stream | P16 |

**Asumsi yang BELUM terverifikasi (jangan diperlakukan sebagai fakta):**
kemungkinan audio pipe atau file sumber (P16: "kemungkinan") — ini hipotesis
awal, bukan diagnosis.

**OPEN QUESTION (ditemukan saat perluasan tiket — wajib dijawab saat eksekusi):**
OQ-007 — `AUDIT_TAHAP0_LIVE200_2026-11-21.md` l.38 memetakan stream
#1795226249006 ke **STAGING-SLOT-114**, sementara P16 + VERIFIKASI_CPU
memetakannya ke **slot 706**. Salah satu pemetaan salah (atau stream ID
dipakai ulang). Eksekusi tiket WAJIB memverifikasi pemetaan stream→slot dari
konfigurasi supervisor/DB sebelum menyimpulkan apa pun tentang slot 706.

**Dua pekerjaan berbeda (jangan dicampur):**

1. **Diagnosis (tiket ini)** — temukan exit code, signal, dan rantai penyebab
   tiap kematian + mengapa orphan lolos dari supervisor.
2. **Perbaikan** — tergantung temuan (perbaikan kode supervisor, parameter
   FFmpeg, file sumber, atau health check). Perbaikan adalah tiket terpisah
   (L2/L3) dan TIDAK termasuk dalam tiket audit ini.

## B. Target

1. Exit code + signal tiap kematian (5 peristiwa) terdokumentasi dari log.
2. Klasifikasi tiap kematian: FFmpeg mati vs RTMP putus vs broadcast selesai
   (V2.0 §16.4).
3. Stderr FFmpeg tiap kematian tersimpan sebagai bukti (bukan ringkasan).
4. Process tree + status orphan (PID 1435089 — masih ada / sudah mati /
   sudah diadopsi) terdokumentasi.
5. Input file + parameter encoder slot 706 tercatat (file mana, perintah
   FFmpeg lengkap).
6. Korelasi tiap kematian dengan resource (CPU/RAM/disk/TX) + network saat
   insiden — atau dinyatakan eksplisit TIDAK ADA korelasi yang ditemukan.
7. Peran DB-lock di health check (`live_supervisor.py:878, 920`) dalam
   keterlambatan deteksi dinilai dari kode + journal.
8. Rekomendasi perbaikan terurut (dengan level persetujuan masing-masing)
   + rencana uji recovery.
9. OQ-007 terjawab: pemetaan stream→slot yang benar.

## C. Prasyarat

- [ ] Akses SSH baca ke VPS Master (Tailscale, alias `masjavas-vps`).
- [ ] Journal live 24 jam masih tersedia (sebelum rotasi menghapusnya —
      tangkap SEGERA; eksekusi tiket ini sensitif waktu).
- [ ] Akses baca kode `live_supervisor.py` (path di VPS — temukan via F.1).
- [ ] Tidak ada restart/kill encoder atau service live selama audit
      (koordinasi: audit ini read-only, tapi restart bersamaan mengubah
      angka + menghilangkan orphan).
- [ ] Baseline V2-005 (bila sudah ada) sebagai pembanding resource normal.

## D. Scope pemeriksaan (file/module)

| # | Area | Lokasi / cara periksa |
|---|---|---|
| D1 | Journal live 24 jam | `journalctl` unit live — temukan nama unit dulu (`systemctl list-units \| grep -i live`); saring "Proses FFmpeg mati mendadak" + stream #1795226249006 + rentang 21 Nov 15:00–22 Nov 03:30 |
| D2 | Kematian per peristiwa | Untuk tiap dari 5 timestamp: exit code, signal, baris log ±5 menit, stderr FFmpeg |
| D3 | Stderr FFmpeg | Lokasi log stderr (dari konfigurasi supervisor / unit systemd) — simpan verbatim per peristiwa |
| D4 | Process tree + orphan | `ps -ef --forest` + `ps -o pid,ppid,etime,args -p 1435089` (bila PID masih ada; bila tidak, catat sebagai temuan + cari orphan baru); daftar SEMUA proses ffmpeg + parent-nya |
| D5 | Supervisor + health check | Baca `live_supervisor.py` sekitar l.878, 920 (DB-lock) + logika restart + deteksi kematian + adopsi/orphan handling (bila ada) |
| D6 | Input file + parameter encoder | File sumber slot 706 (path + ffprobe: durasi, codec, integritas) + perintah FFmpeg lengkap dari supervisor/config |
| D7 | Resource + network saat insiden | CPU/RAM/disk/TX + koneksi RTMP pada tiap timestamp kematian (dari metrik yang tersedia di VPS + journal); bandingkan dengan baseline normal |
| D8 | Pemetaan stream→slot (OQ-007) | Konfigurasi supervisor / DB: stream ID mana milik slot 706 vs slot 114; catat pemetaan yang benar |
| D9 | Riwayat restart supervisor | Restart 21 Nov 14:42 — mengapa orphan selamat; apakah restart supervisor membunuh/mengadopsi anak prosesnya |
| D10 | Pola kematian | Interval antar kematian (33 mnt, 108 mnt, 114 mnt, 391 mnt) — periodik? terkait jadwal job lain (sync Drive, ingestion, Stage 2)? |

## E. Pertanyaan teknis yang harus dijawab

| # | Pertanyaan | Bukti yang menjawab |
|---|---|---|
| Q1 | Berapa exit code + signal tiap dari 5 kematian? | D1 + D2 — angka per peristiwa, bukan satu angka gabungan |
| Q2 | Tiap kematian: FFmpeg mati, RTMP putus, atau broadcast selesai? | D2 + D3 (stderr + pesan penutup koneksi) |
| Q3 | Apa isi stderr FFmpeg pada tiap kematian? (baris terakhir sebelum mati) | D3 — kutipan verbatim |
| Q4 | Mengapa orphan PID 1435089 lolos dari supervisor + restart 14:42? (mekanisme: double-fork? setsid? supervisor tidak melacak PID?) | D4 + D5 + D9 |
| Q5 | Apakah DB-lock di health check (`live_supervisor.py:878, 920`) memperlambat deteksi kematian? (jeda deteksi vs restart ±15 dtk) | D5 + D1 (timestamp deteksi vs timestamp kematian) |
| Q6 | Apakah ada korelasi kematian dengan lonjakan CPU/RAM/disk/TX atau job lain (sync Drive, ingestion, Stage 2)? | D7 + D10 |
| Q7 | Apakah file sumber / parameter encoder slot 706 bermasalah? (file korup, durasi, audio pipe) | D6 — hasil ffprobe + perintah FFmpeg |
| Q8 | Pemetaan stream→slot yang benar (OQ-007)? | D8 |
| Q9 | Apa rekomendasi perbaikan terurut + level persetujuan masing-masing? | Sintesis Q1–Q8 |

## F. Perintah / metode audit (executable, read-only)

**F.1 — Temukan unit live + path kode (discovery, satu panggilan):**

```bash
ssh -F /home/hatch/.ssh/config masjavas-vps \
  'systemctl list-units --type=service | grep -i live; find /opt/masjavas-os -maxdepth 3 -name "live_supervisor.py" 2>/dev/null'
```

Catat nama unit + path file. Semua perintah berikut memakai hasil ini
(`<UNIT>` = nama unit, `<SUP>` = path `live_supervisor.py`).

**F.2 — Journal kematian encoder (baca-saja; tangkap SEGERA sebelum rotasi):**

```bash
ssh -F /home/hatch/.ssh/config masjavas-vps \
  'journalctl -u <UNIT> --since "2026-11-21 14:00" --until "2026-11-22 04:00" --no-pager | grep -i "ffmpeg mati\|1795226249006\|database is locked" | head -120'
```

Simpan output LENGKAP (bukan hanya grep) ke file bukti bila memungkinkan —
`journalctl ... > /tmp/v2-002-journal-706.txt` lalu unduh. Catat exit code /
signal bila tercatat di log; bila TIDAK tercatat, itu temuan (Q1 Partial).

**F.3 — Process tree + orphan (baca-saja):**

```bash
ssh -F /home/hatch/.ssh/config masjavas-vps \
  'ps -ef --forest | grep -i -E "ffmpeg|live_supervisor" | grep -v grep; echo ---; ps -o pid,ppid,etime,args -p 1435089 || echo ORPHAN-1435089-GONE'
```

Bila PID 1435089 sudah tidak ada, catat kapan ia hilang (dari journal) +
cari orphan ffmpeg lain (`ps -ef | awk '$3==1 && /ffmpeg/'`).

**F.4 — Baca health check + logika restart (baca file):**

```bash
ssh -F /home/hatch/.ssh/config masjavas-vps \
  'sed -n "850,950p" <SUP>'
```

Baca sampai tuntas: penanganan DB-lock, deteksi kematian, restart, adopsi
orphan (bila ada). Jangan simpulkan dari nama fungsi saja.

**F.5 — Input file + parameter encoder slot 706:**

```bash
ssh -F /home/hatch/.ssh/config masjavas-vps \
  'grep -n "706" <SUP> | head -40'
```

Sesuaikan dengan hasil F.1. Catat perintah FFmpeg lengkap + path file
sumber; jalankan `ffprobe` read-only pada file sumber (durasi, codec, error).

**F.6 — Korelasi resource + pemetaan stream→slot:**

- Resource: pakai metrik yang tersedia di VPS pada tiap timestamp kematian
  (journal + snapshot V2-005 bila ada); catat metode + sumber per angka.
- OQ-007: baca konfigurasi supervisor / DB yang memetakan stream ID ke slot;
  catat pemetaan benar untuk #1795226249006.

**DILARANG dalam tiket ini:**

- Restart / kill / stop encoder, supervisor, atau service live apa pun.
- Mengubah kode, config, systemd unit, parameter FFmpeg, atau file sumber.
- Mengadopsi / membunuh orphan secara manual (diagnosis dulu — adopsi
  adalah perbaikan, tiket terpisah).
- Menjalankan benchmark / load test di box live selama audit.
- Menyimpulkan akar penyebab dari SATU kematian saja (butuh pola 5 peristiwa).

## G.0 — Evidence manifest (wajib diisi saat eksekusi)

Setiap Q1–Q9 wajib punya satu baris manifest. Isi manifest BERSAMAAN dengan
eksekusi, bukan di akhir — journal bisa terotasi dan orphan bisa hilang.

| Question ID | Command | Timestamp (WIB) | Exit code | Evidence | Finding | Confidence | Impact |
|---|---|---|---|---|---|---|---|
| Q1 | F.2 journal + grep `poll()` | 07:02–07:08 | 0 | `v2-002/ENCODER706_DEATHS_V2-002.md` | Exit code TIDAK PERNAH direkam (health check hanya cek PID) | Unknown (by design) | R6 |
| Q2 | F.2 journal + stderr log stream | 07:02–07:06 | 0 | `v2-002/ENCODER706_DEATHS_V2-002.md` | 12 kematian slot 114: RTMP "Connection timed out" | Confirmed | Akar penyebab teridentifikasi |
| Q3 | F.3 stderr verbatim | 07:06 | 0 | `v2-002/ENCODER706_DEATHS_V2-002.md` | "Error muxing a packet" + "Error writing trailer: Connection timed out"; speed 0,97–0,99x | Confirmed | Menunjuk rantai audio |
| Q4 | F.3 + F.4 + F.2 (restart 14:42) | 06:58–07:04 | 0 | `v2-002/ENCODER706_ORPHAN_V2-002.md` | Orphan 1435089 (task 706) sehat 21,5 jam; mekanisme `start_new_session=True`; tetap termonitor via PID DB | Confirmed | Risiko rendah; R8 |
| Q5 | F.4 + F.2 (jeda deteksi) | 07:06–07:09 | 0 | `v2-002/ENCODER706_HEALTHCHECK_V2-002.md` | 8 lock, NOL korelasi dengan kematian; dampak ≤1 tick | Confirmed (dieliminasi) | DB-lock bukan penyebab |
| Q6 | F.6 resource | 07:06–07:09 | 0 | `v2-002/ENCODER706_CORRELATION_V2-002.md` | CPU/RAM/disk normal; OOM nihil; admission tolak 5× reconnect #12 (TX 108–200 vs guard 29–59) | Confirmed | R4; downtime #12 93 dtk |
| Q7 | F.5 + perintah FFmpeg + DB | 07:00–07:06 | 0 | `v2-002/ENCODER706_CORRELATION_V2-002.md` | File bersih; audio relay HLS tersangka (hipotesis) | Partial | R3 |
| Q8 | F.6 pemetaan stream→slot | 07:00 | 0 | `v2-002/ENCODER706_RECOMMENDATIONS_V2-002.md` | #1795226249006 = STAGING-SLOT-114; P16 salah | Confirmed | Koreksi judul/dokumen |
| Q9 | Sintesis Q1–Q8 | 07:10 | — | `v2-002/ENCODER706_RECOMMENDATIONS_V2-002.md` | 8 rekomendasi (R1–R8) + rencana uji recovery | Confirmed | Perbaikan butuh user (L2/L3) |

Aturan manifest: sama seperti V2-001 §G.0 — `Confirmed` (bukti langsung dari
log/kode/ps), `Partial` (bukti tidak lengkap / hanya satu sumber), `Unknown`
(tidak dapat diverifikasi + alasannya dicatat). `Unknown` yang eksplisit
LEBIH BAIK daripada dugaan yang ditulis seolah fakta. Bukti TIDAK BOLEH berisi
nilai kredensial.

## G. Bukti yang harus dilampirkan (deliverable tiket)

1. `ENCODER706_DEATHS_V2-002.md` — Q1 + Q2 + Q3 (per peristiwa: timestamp,
   exit code, signal, klasifikasi, stderr verbatim).
2. `ENCODER706_ORPHAN_V2-002.md` — Q4 (process tree, status orphan,
   mekanisme lolos dari supervisor).
3. `ENCODER706_HEALTHCHECK_V2-002.md` — Q5 (kutipan kode health check +
   penilaian jeda deteksi).
4. `ENCODER706_CORRELATION_V2-002.md` — Q6 + Q7 (resource saat insiden,
   ffprobe file sumber, perintah FFmpeg).
5. `ENCODER706_RECOMMENDATIONS_V2-002.md` — Q9 (rekomendasi terurut +
   level persetujuan + rencana uji recovery) + jawaban OQ-007 (Q8).
6. Semua file di `workspace/engineering/v2-002/` (buat foldernya saat eksekusi).

## H. Acceptance criteria (lulus / gagal)

**LULUS bila SEMUA terpenuhi:**

- [ ] Q1–Q9 terjawab dengan bukti (bukan dugaan); yang tidak bisa dijawab
      dicatat eksplisit sebagai "TIDAK DAPAT DIVERIFIKASI + alasannya".
- [ ] Exit code + signal tercatat per peristiwa (5/5) — atau dinyatakan
      eksplisit log tidak mencatatnya (temuan, bukan kegagalan audit).
- [ ] Stderr FFmpeg tiap kematian tersimpan verbatim (atau dinyatakan
      eksplisit tidak tersedia + lokasinya dicari).
- [ ] Status orphan terdokumentasi (masih ada / hilang / diadopsi) +
      mekanisme lolosnya dijelaskan dari kode.
- [ ] OQ-007 terjawab: pemetaan stream→slot yang benar tercatat.
- [ ] Rekomendasi perbaikan terurut + level persetujuan + rencana uji recovery.
- [ ] Nol restart/kill/ubah selama audit; stream live tidak terganggu oleh audit.

**GAGAL bila salah satu terjadi:**

- Journal sudah terotasi sebelum ditangkap dan tidak ada salinan (→ bukti
  5 kematian hilang; eskalasi, jangan ditebak dari ingatan).
- Klasifikasi kematian (Q2) disimpulkan tanpa stderr / tanpa exit code.
- Orphan tidak diperiksa (status + mekanisme lolos tidak diketahui).
- OQ-007 tidak dijawab — kesimpulan tentang slot 706 bisa salah alamat.
- Rekomendasi perbaikan ditulis tanpa membedakan diagnosis vs perbaikan
  (→ perbaikan dieksekusi tanpa tiket L2/L3).

## I. Dampak jika gagal / jika tiket tidak dikerjakan

- Encoder 706 terus mati 5×/hari tanpa diketahui penyebabnya — setiap
  kematian = jeda siaran; risiko stream tidak kembali bila supervisor
  terdegradasi DB-lock saat kematian terjadi.
- Orphan tanpa pengawas: bila mati, tidak ada yang me-restart-nya — stream
  hilang diam-diam sampai ada yang menyadari.
- Desain sharding VPS 8 (V2.0 §16.4: state health-check LOKAL di box live)
  dirancang tanpa memahami kegagalan ini — risiko membawa pola yang sama
  ke box baru.
- Perbaikan ditebak (mis. ganti file sumber / naikkan resource) tanpa bukti —
  biaya + downtime tanpa jaminan sembuh.

## J. Batas tindakan yang diizinkan

- **BOLEH:** semua perintah §F (read-only); membaca kode/config/journal/DB/
  dokumen; `ffprobe` read-only pada file sumber; menulis 5 dokumen bukti §G.
- **TIDAK BOLEH:** restart/kill/stop encoder, supervisor, atau service live;
  ubah kode/config/systemd/parameter FFmpeg/file sumber; adopsi/bunuh orphan;
  benchmark/load test di box live; eksekusi perbaikan apa pun (tiket terpisah).
- **Perbaikan (pasca-diagnosis) = USER** (L2/L3 sesuai temuan — lihat §K).

## K. Pihak yang menyetujui langkah berikutnya

| Langkah berikut | Penyetuju |
|---|---|
| Perbaikan parameter FFmpeg / file sumber (bila Q7 menunjuk ke sana) | User (L2 — perubahan perilaku encoder live) |
| Perbaikan supervisor / health check / adopsi orphan (bila Q4/Q5 menunjuk ke sana) | User (L3 — perubahan kode supervisor live) |
| Uji recovery (matikan encoder sengaja untuk uji restart) | User (L3 — mengganggu siaran live; jadwalkan di luar jam prime) |
| Penambahan business health check (V2.0 §16.4) | User (L2/L3 — tergantung implementasi) |
| Lanjut ke V2-003 dst. | Tidak butuh persetujuan baru (Batch 1 audit) |

## L. Estimasi & catatan eksekusi

- Estimasi: ±1–1,5 jam (termasuk tunggu SSH + baca kode; journal + ps cepat).
- Jalankan dari main/side chat (bukan proactivity handoff — konsistensi
  dengan V2-001).
- **Tangkap journal + status orphan SEGERA** — journal bisa terotasi, orphan
  bisa mati/hilang; setiap jam berlalu, bukti hilang permanen.
- Bila PID 1435089 sudah tidak ada saat eksekusi, JANGAN anggap masalah
  selesai — catat kapan hilang + apakah ada orphan baru + tetap jawab Q4
  dari kode (mekanisme lolos).
- Bila exit code/signal TIDAK tercatat di log, catat sebagai temuan
  (kekurangan observability — rekomendasi: log exit code + stderr permanen),
  bukan sebagai kegagalan audit.
- Koordinasi dengan lane lain: jangan audit bersamaan dengan restart
  supervisor / deployment — beri tahu sebelum mulai.

---

# TIKET V2-003 — RESOURCE GUARD V2 AUDIT

**Judul:** Audit implementasi Resource Guard v2 — apakah governor benar-benar
mengatur, atau hanya mencatat
**Prioritas:** P0 — CRITICAL
**Workstream:** A (Production Stabilization) / P0.3
**Owner:** Muse (audit) — perbaikan: user (L2/L3)
**Status:** READY_FOR_REVIEW (audit selesai 22 Nov 2026 07:40 WIB; 6 dokumen bukti + manifest di `workspace/engineering/v2-003/`; temuan utama: governor 0 aksi efektif/17 jam; konfigurasi split guard V2/admission legacy; diskrepansi TX SOLVED — instan vs EMA)
**Dibuat:** 22 Nov 2026 (stub) — **Diperluas:** 22 Nov 2026 07:15 WIB (Cycle 4, sesi interaktif)
**Dependency:** akses baca source + journal di VPS (host BENAR: `masjavas-vps-baru` — pelajaran F-E V2-002)

## A. Masalah

Resource Guard v2 live sejak 20 Nov 23:02 (lane Antigravity, commit
`5b8279a`, `4fff60a`, `2e61dcc`) dengan janji: governor lokal berbasis
atribusi yang melindungi live streaming dari perebutan resource oleh
pekerja latar (ingestion, render, Stage 2). Tetapi dua sinyal
menunjukkan governor mungkin tidak mengatur apa-apa:

1. **`shed_render` log-only 112×/24 jam** — sinyal "kurangi beban render"
   dicatat tanpa eksekutor yang diketahui. Bila tidak ada yang
   mengeksekusi, governor adalah termometer, bukan termostat.
2. **Ingestion CPU 0,0% dalam 1.510 tick governor** — ingestion tidak
   pernah berjalan saat governor mengawasi. Apakah gate ingestion
   (load/RAM/disk) mencekik ingestion sampai mati — guard yang saling
   menyandera, persis yang dilarang doktrin G1–G12?

V2-002 menambah satu anomali: admission control menolak reconnect atas
proyeksi TX 108–200 Mbps sementara guard tick melaporkan 29–59 Mbps
pada menit yang sama — dua pengukuran TX dalam satu governor yang tidak
sepakat.

## B. Target

1. Inventarisasi LENGKAP: metrik yang diukur, threshold/knob + nilai +
   lokasi definisi, dan aksi yang tersedia.
2. Jawab: aksi apa saja yang BENAR-BENAR diambil governor dalam 24 jam
   terakhir (pause? shed? deny? — hitung dari journal, bukan dari klaim
   kode).
3. Lacak `shed_render`/`shed_load` ujung-ke-ujung: siapa memicu, siapa
   menerima, siapa mengeksekusi (atau mengapa tidak ada eksekutor).
4. Petakan G1–G12 (RENCANA_ORCHESTRATOR) ke implementasi: mana yang
   terwujud di kode, mana yang hanya di dokumen.
5. Nilai gate ingestion: kondisi eksak + apakah gate menjelaskan CPU
   ingestion 0,0% (starvation) atau ada penyebab lain.
6. Rekonsiliasi anomali TX admission vs guard tick (temuan V2-002).
7. Nilai failure mode governor: apa yang terjadi bila sampler/governor
   error (apakah guard terlewat diam-diam?).
8. Rekomendasi terurut + level persetujuan masing-masing.

## C. Prasyarat

- [ ] Akses SSH baca ke `masjavas-vps-baru` (host BENAR — verifikasi
      hostname `vmi3762689` + disk 145G sebelum audit).
- [ ] Journal `masjavas-live.service` 24 jam tersedia.
- [ ] Dokumen `RENCANA_ORCHESTRATOR_2026-11-22.md` (G1–G12) dibaca.

## D. Scope pemeriksaan (file/module)

| # | Area | Lokasi / cara periksa |
|---|---|---|
| D1 | Governor live | `worker/live_supervisor.py` (host baru, 1462 baris): sampler (l.79–149), `_get_governor` (l.280), `can_start` (l.431), `signal_worker_shed_load` (l.1081), `run_resource_guard` (l.1098), `_run_resource_guard_legacy` (l.1207) |
| D2 | Modul governor | Temukan modul/class governor yang dipakai `_get_governor` (`grep` import) + baca `admission_check` |
| D3 | Jalur shed | `grep -rn "shed" --include="*.py"` di repo: pemicu → penerima → eksekutor (atau ketiadaan eksekutor) |
| D4 | Gate ingestion | `worker/ingestion.py`: kondisi gate load/RAM/disk/window + efeknya pada klaim/concurrency |
| D5 | Knob/threshold | `settings.py` (+ file settings lain): semua `LIVE_*`, `GUARD_*`, gate ingestion — nilai + komentar |
| D6 | Bukti aksi 24 jam | Journal `masjavas-live.service` 24 jam: hitung guard tick, `action=*`, shed, deny, pause, resume |
| D7 | Doktrin G1–G12 | `RENCANA_ORCHESTRATOR_2026-11-22.md` vs implementasi D1–D5 |

## E. Pertanyaan teknis yang harus dijawab

| # | Pertanyaan | Bukti yang menjawab |
|---|---|---|
| Q1 | Aksi apa yang diambil governor dalam 24 jam terakhir (hitung per jenis aksi)? | D6 — angka dari journal |
| Q2 | `shed_render`/`shed_load`: siapa memicu, siapa menerima, siapa mengeksekusi — dan mengapa log-only? | D3 + D1 (l.1081) |
| Q3 | G1–G12: mana yang terwujud di kode, mana yang tidak? | D7 vs D1–D5 |
| Q4 | Daftar lengkap threshold/knob + nilai + lokasi — mana yang kedaluwarsa (TX 115? RAM 4 GB vs 3,2 tersedia?)? | D5 |
| Q5 | Gate ingestion: kondisi eksak + apakah gate menjelaskan CPU ingestion 0,0%? | D4 + D6 |
| Q6 | Mengapa admission TX (108–200 Mbps) ≠ guard tick TX (29–59 Mbps) pada menit yang sama? | D2 + D1 (sampler vs governor snapshot) |
| Q7 | Failure mode governor: apa terjadi bila sampler/governor error? Apakah guard terlewat diam-diam? | D1 (`run_forever` exception handling + `run_resource_guard` try/except) |
| Q8 | Rekomendasi perbaikan terurut + level persetujuan masing-masing? | Sintesis Q1–Q7 |

## F. Perintah / metode audit (executable, read-only, host `masjavas-vps-baru`)

**F.1 — Verifikasi host + struktur governor (satu panggilan):**

```bash
ssh -F /home/hatch/.ssh/config masjavas-vps-baru \
  'hostname; df -h / | tail -1; wc -l /opt/masjavas-os/worker/live_supervisor.py /opt/masjavas-os/worker/ingestion.py; grep -n "import.*governor\|from.*governor" /opt/masjavas-os/worker/live_supervisor.py'
```

**F.2 — Baca guard loop + shed + sampler:**

```bash
ssh -F /home/hatch/.ssh/config masjavas-vps-baru \
  'sed -n "1081,1210p" /opt/masjavas-os/worker/live_supervisor.py'
```

(Lanjutkan `sed` untuk l.79–149 sampler dan l.1207–1350 legacy bila perlu.)

**F.3 — Lacak shed di seluruh repo:**

```bash
ssh -F /home/hatch/.ssh/config masjavas-vps-baru \
  'grep -rn "shed" /opt/masjavas-os --include="*.py" | grep -v tests | head -40'
```

**F.4 — Knob + gate ingestion:**

```bash
ssh -F /home/hatch/.ssh/config masjavas-vps-baru \
  'grep -n "LIVE_\|GUARD_" /opt/masjavas-os/config/settings.py | head -40; echo ---; grep -n "load\|ram\|RAM\|disk\|gate" /opt/masjavas-os/worker/ingestion.py | grep -i -E "threshold|limit|gate|>=|>|free" | head -30'
```

**F.5 — Bukti aksi 24 jam (journal):**

```bash
ssh -F /home/hatch/.ssh/config masjavas-vps-baru \
  'journalctl -u masjavas-live.service --since "2026-11-21 07:00" --no-pager | grep -c "Guard tick"; journalctl -u masjavas-live.service --since "2026-11-21 07:00" --no-pager | grep -o "action=[a-z_]*" | sort | uniq -c; journalctl -u masjavas-live.service --since "2026-11-21 07:00" --no-pager | grep -ci "shed"; journalctl -u masjavas-live.service --since "2026-11-21 07:00" --no-pager | grep -ci "pause_stream\|ditolak admission"'
```

**F.6 — Modul governor + `admission_check`:** baca file modul governor
(dari hasil F.1) — fungsi `admission_check` + sumber metrik TX-nya.

**DILARANG dalam tiket ini:**

- Mengubah threshold/knob/settings apa pun (L3).
- Me-restart service live/worker/web.
- Memicu shed/pause/deny secara manual untuk "menguji" (itu perubahan
  perilaku produksi — butuh tiket perbaikan + persetujuan).
- Menyimpulkan governor "bekerja" dari keberadaan kode saja — butuh
  bukti AKSI di journal.

## G.0 — Evidence manifest (wajib diisi saat eksekusi)

| Question ID | Command | Timestamp (WIB) | Exit code | Evidence | Finding | Confidence | Impact |
|---|---|---|---|---|---|---|---|
| Q1 | F.5 journal | 07:20–07:30 | 0 | `v2-003/GUARD_ACTIONS_V2-003.md` | 1.918 tick V2: 1.886 none, 29 shed_render, 0 aksi efektif; 91 sinyal shed SEMUA pra-14:42 (legacy); guard=V2, admission=legacy (split) | Confirmed | Governor termometer |
| Q2 | F.3 + F.2 | 07:15–07:25 | 0 | `v2-003/GUARD_SHED_V2-003.md` | shed_ingestion jalur lengkap tapi tak pernah dipilih (0/1.918); shed_render tanpa eksekutor (by design); atribusi ingestion buta | Confirmed | R3 + R4 (scope V2-003) |
| Q3 | D7 vs D1–D5 | 07:15–07:35 | 0 | `v2-003/GUARD_G1G12_MAP_V2-003.md` | 7 terwujud (G4 diperbaiki), G6/G7 Partial, G9 churn 67% bertahan, G10/G11 tidak ditangani, G12 sebagian | Confirmed (sebagian Partial) | Peta lengkap |
| Q4 | F.4 settings | 07:15–07:30 | 0 | `v2-003/GUARD_G1G12_MAP_V2-003.md` | Inventaris lengkap; TX 115 kedaluwarsa, CPU 90 inert, 720p stale, flag V2 menyesatkan | Confirmed | R2, R5, R8, R9 |
| Q5 | F.4 ingestion gates | 07:15–07:35 | 0 | `v2-003/GUARD_GATES_V2-003.md` | CPU 0,0% = artefak atribusi (daemon → "other"); 5 cap → konkurensi 1 permanen; 0 stage berat/24 jam | Confirmed | R7 |
| Q6 | F.6 governor + F.2 sampler | 07:20–07:35 | 0 | `v2-003/GUARD_GATES_V2-003.md` | SOLVED: admission legacy pakai TX instan 5 dtk, guard pakai EMA 300 dtk, sampler SAMA; + anggaran 115 kedaluwarsa | Confirmed | R1 + R2 |
| Q7 | F.2 run_forever + guard | 07:25–07:35 | 0 | `v2-003/GUARD_GATES_V2-003.md` | Loop selamat (13 skip/24 jam, tercatat tanpa alert); snapshot basi → NONE; degradasi sampler nyaris tak terlihat | Confirmed | R6 |
| Q8 | Sintesis Q1–Q7 | 07:40 | — | `v2-003/GUARD_RECOMMENDATIONS_V2-003.md` | 9 rekomendasi (R1–R9 scope V2-003 — tabrakan label dengan V2-002, pakai prefix V2-003-R#) + rencana uji + OQ-009/010/011 | Confirmed | Perbaikan butuh user (L2/L3) |

## G. Bukti yang harus dilampirkan (deliverable tiket)

1. `GUARD_ACTIONS_V2-003.md` — Q1 (aksi governor 24 jam, hitungan journal).
2. `GUARD_SHED_V2-003.md` — Q2 (jalur shed ujung-ke-ujung).
3. `GUARD_G1G12_MAP_V2-003.md` — Q3 + Q4 (peta doktrin + inventaris threshold).
4. `GUARD_GATES_V2-003.md` — Q5 + Q6 + Q7 (gate ingestion, anomali TX, failure mode).
5. `GUARD_RECOMMENDATIONS_V2-003.md` — Q8 (rekomendasi terurut + level).
6. `EVIDENCE_MANIFEST_V2-003.md` — manifest Q1–Q8.
7. Semua file di `workspace/engineering/v2-003/` (buat foldernya saat eksekusi).

## H. Acceptance criteria (lulus / gagal)

**LULUS bila SEMUA terpenuhi:**

- [ ] Q1–Q8 terjawab dengan bukti; yang tidak bisa dijawab dicatat
      eksplisit sebagai "TIDAK DAPAT DIVERIFIKASI + alasannya".
- [ ] Hitungan aksi governor 24 jam berasal dari journal (bukan klaim kode).
- [ ] Jalur shed dilacak sampai ujung (eksekutor ditemukan ATAU
      ketiadaannya dibuktikan dengan grep menyeluruh).
- [ ] G1–G12 dipetakan satu-per-satu ke kode (ada/tidak + lokasi).
- [ ] Inventaris threshold lengkap + penilaian kedaluwarsa per knob.
- [ ] Rekomendasi terurut + level persetujuan.

**GAGAL bila:** menyimpulkan governor bekerja tanpa bukti aksi; mengubah
threshold; me-restart service; mengaudit host yang salah (verifikasi
host dulu — pelajaran F-E).

## Batas tindakan yang diizinkan

- **BOLEH:** semua perintah §F (read-only); membaca kode/log/dokumen;
  menulis 6 dokumen bukti.
- **TIDAK BOLEH:** mengubah settings/threshold; restart service;
  memicu aksi governor manual; mengaudit `masjavas-vps` (box lama).

---

# TIKET V2-004 — JANITOR SAFETY AUDIT

**Judul:** Audit keselamatan janitor disk — semua kondisi penghapusan file,
klasifikasi unpaired, dan proteksi file channel CONNECTED
**Prioritas:** P0 — CRITICAL | **Workstream:** P0.4 (Track A — Stabilization)
**Owner:** Muse (audit) — perbaikan: user (L2/L3, tergantung temuan)
**Status:** NOT_STARTED (tiket lengkap — eksekusi menunggu sesi interaktif + SSH)
**Dibuat:** 22 Nov 2026 (stub) — **Diperluas:** 22 Nov 2026 10:50 WIB
(Cycle 5, cron local-only)
**Dependency:** backup DB konsisten membantu verifikasi (V2-005/V2-007
membantu, BUKAN blocker); audit ini read-only sehingga bisa jalan sebelum
keduanya
**Diblokir oleh:** tidak ada — membutuhkan akses baca VPS (SSH Tailscale,
host BENAR `masjavas-vps-baru` — pelajaran F-E V2-002) + baca kode
`tools/vps_disk_janitor.py`

## A. Masalah

Janitor disk (`tools/vps_disk_janitor.py`) adalah satu-satunya proses yang
MENGHAPUS file lokal secara otomatis — dan ia salah mengklasifikasi.
Klausa `OR channel_id LIKE 'STAGING-SLOT-%'` menarik 203 channel CONNECTED
(yang `channel_id`-nya masih placeholder STAGING) ke dalam set kunci
"unpaired", sehingga file mereka dihapus setelah **1 jam** (jalur unpaired),
bukan **12 jam** (TTL normal), dan audit penghapusan salah label. Pencocokan
substring memperparah: kunci pendek bisa cocok dengan folder channel lain
(mis. `slot_2` ⊂ `slot_266`).

Ini BUKAN teori: reklasifikasi audit 24 jam (22 Nov 01:40 WIB) membuktikan
40,92 GB file channel CONNECTED terhapus berlabel "unpaired", dan 12/12
sampel video `21.mp4` berasal dari channel CONNECTED yang sudah PUBLISHED
hari itu. Dampak operasional sejauh ini ringan (lease melindungi tugas aktif;
penghapusan terjadi pasca-publish) — tetapi jendela risiko nyata: retry,
re-upload, atau verifikasi yang butuh file lokal dalam 12 jam pasca-publish.
Dan janitor berjalan tiap 30 menit, jadi setiap jam berlalu menambah file
yang terhapus dengan klasifikasi salah.

**Fakta terverifikasi (bukan asumsi):**

| Fakta | Nilai | Sumber |
|---|---|---|
| Query kunci unpaired | `SELECT channel_id, channel_name, slot_number FROM channels WHERE status != 'CONNECTED' OR channel_id LIKE 'STAGING-SLOT-%'` | `LAPORAN_BOTTLENECK_UNPAIRED_RENDER_2026-11-22.md` Temuan 2 (kode ~baris 100–120) |
| Total baris `channels` | 707 (237 CONNECTED, 470 UNPAIRED) | DB langsung 22 Nov 01:40 |
| Baris `STAGING-SLOT-*` | 673/707 | DB langsung 22 Nov 01:40 |
| CONNECTED ber-ID STAGING | 203 slot (0 slot duplikat); contoh slot 266 = `('STAGING-SLOT-266', 'Tangis Senja', 'CONNECTED')` | DB langsung 22 Nov 01:40 |
| Penghapusan 24 jam | 3.409 file / 121,14 GB (`deletion_audit`); `JANITOR_UNPAIRED_QUARANTINE_CLEAN` 3.303 file / 112,52 GB | `deletion_audit` 22 Nov |
| Reklasifikasi 24 jam | Benar-benar UNPAIRED 65,2 GB (1.307 file); sebenarnya CONNECTED 40,92 GB (1.823 file); folder tanpa baris channel (mis. `Andrew`) 6,4 GB (173 file) | Reklasifikasi audit 22 Nov |
| Sampel video | 12/12 `21.mp4` yang dihapus sebagai "unpaired" berasal dari channel CONNECTED yang PUBLISHED 21 Nov (slot 266, 285, 315, 51, 32, 79, 280, 352, 263, 69, 120, 246) | Sampel audit 22 Nov |
| Aturan TTL | Normal 12 jam (4 jam bila disk >75%); jalur unpaired 1 jam; proteksi lease | `tools/vps_disk_janitor.py` (via laporan bottleneck) |
| Interval janitor | 1800 dtk (30 menit) | `EVALUASI_SEBELUM_PERBAIKAN` l.107 |
| Disk saat evaluasi | 72% (41 GB bebas); janitor aktif | EVALUASI l.19 |
| Pengecualian cache | `_shared_cache` (0,33 GB) + `global_audio_pool` dikecualikan janitor secara resmi | `DIAGNOSIS_DISK_2026-11-20.md` l.260 |
| Scheduler BUKAN sumber | `operational_tasks` 21–22 Nov untuk channel UNPAIRED: 0 baris; populasi + acquire terfilter CONNECTED (`scheduler.py:88,980`) | Laporan bottleneck Temuan 1 |
| Status checklist | P0.4 (frozen): audit seluruh kondisi hapus; inventarisasi referensi file di DB; identifikasi file proses aktif + status CONNECTED; tambah cek owner/job/lifecycle/umur; quarantine + grace period; catat tiap kandidat hapus; uji orphan aman / CONNECTED tak boleh hapus / file dipakai FFmpeg | `IMPLEMENTATION_CHECKLIST_..._V2.md` §P0.4 |
| Kepemilikan | Janitor membersihkan file dua pipeline (publish + pra-publish) — kepemilikan pasca-pemisahan worker masih keputusan terbuka (D5) | EVALUASI l.140 |

**Asumsi yang BELUM terverifikasi (jangan diperlakukan sebagai fakta):**
apakah proteksi lease menutupi SEMUA jalur hapus (atau ada jalur yang
melewatinya); apakah quarantine + grace period sudah ada di kode (nama
`JANITOR_UNPAIRED_QUARANTINE_CLEAN` menyiratkan jalur quarantine, tapi
perilakunya belum dibaca); apakah janitor punya mode dry-run.

**OPEN QUESTION (ditemukan saat perluasan tiket — wajib dijawab saat eksekusi):**
- OQ-013 — Apakah janitor punya mode dry-run / simulasi? (Bila tidak ada,
  audit hanya bisa membaca kode + `deletion_audit`, tidak bisa menguji
  klasifikasi tanpa risiko.)
- OQ-014 — Berapa lama baris `deletion_audit` dipertahankan (retensi)?
  Bila retensi pendek, bukti reklasifikasi 24 jam bisa hilang — tangkap
  SEGERA saat eksekusi.
- OQ-015 — Apakah ada jalur hapus lain selain janitor (mis. auto-prune
  guard, cleanup manual, `find -delete` di cron) yang ikut menghapus file
  lokal? (DIAGNOSIS_DISK menyebut "trigger auto-prune" di bawah ambang
  disk kritis — perlu dipetakan.)

**Dua pekerjaan berbeda (jangan dicampur):**

1. **Audit (tiket ini)** — inventarisasi SEMUA kondisi penghapusan,
   reklasifikasi bukti hapus, dan penilaian proteksi (lease, quarantine,
   grace period, cek owner/job/lifecycle/umur).
2. **Perbaikan** — tergantung temuan (kemungkinan besar: filter
   status-only + pencocokan eksak, R1 laporan bottleneck; plus quarantine
   + grace period bila belum ada). Perbaikan adalah tiket terpisah
   (L2/L3) dan TIDAK termasuk dalam tiket audit ini. Catatan: P4
   (perbaikan janitor) adalah perbaikan MANDIRI yang tidak menunggu P5
   (migrasi 673 ID STAGING → ID YouTube asli, risiko TINGGI).

## B. Target

1. Inventaris LENGKAP semua kondisi penghapusan file di janitor
   (file + fungsi + baris + kondisi + TTL + jalur: normal / unpaired /
   tekanan disk / quarantine).
2. Reklasifikasi penghapusan 24 jam (dan 7 hari bila data tersedia):
   berapa GB/file channel CONNECTED terhapus berlabel unpaired —
   angka yang bisa dibandingkan dengan baseline 40,92 GB.
3. Penilaian proteksi lease: kondisi eksak + apakah ada file leased /
   tugas aktif yang pernah terhapus (bukti dari `deletion_audit` +
   `operational_tasks`).
4. Penilaian pencocokan folder: substring vs eksak — daftar kunci yang
   berpotensi tabrakan lintas slot (mis. `slot_2` ⊂ `slot_266`) +
   berapa folder yang terdampak.
5. Perilaku TTL di bawah tekanan disk: ambang 75% → 4 jam — kapan
   terakhir terpicu, seberapa sering, file apa yang kena.
6. Quarantine + grace period: ada di kode atau tidak; bila ada, berapa
   lama + file apa yang masuk; bila tidak ada, catat sebagai gap vs
   P0.4.
7. Inventaris referensi file di DB: tabel + kolom apa saja yang
   menunjuk file lokal (`ingestion_items.local_path` dkk.) — dasar
   uji orphan.
8. File yang masih digunakan proses aktif: apakah janitor bisa menghapus
   file yang sedang dibaca FFmpeg / live / Stage 2? (cek dari kode +
   bukti hapus).
9. Peta jalur hapus lain (OQ-015): auto-prune guard, cron, cleanup
   manual — bila ada.
10. Rekomendasi perbaikan terurut (dengan level persetujuan
    masing-masing) + rencana uji (orphan aman dihapus / CONNECTED tak
    boleh dihapus / file dipakai FFmpeg).

## C. Prasyarat

- [ ] Akses SSH baca ke `masjavas-vps-baru` (host BENAR — verifikasi
      hostname `vmi3762689` + disk 145G sebelum audit; pelajaran F-E).
- [ ] Kode `tools/vps_disk_janitor.py` + unit/timer systemd-nya ditemukan
      (F.1).
- [ ] `deletion_audit` 24 jam masih tersedia (OQ-014 — tangkap SEGERA;
      audit ini sensitif waktu seperti V2-002).
- [ ] Backup DB konsisten tersedia ATAU dibuat sebelum audit membaca
      `deletion_audit` + `channels` (read-only, tapi backup melindungi
      bila eksekusi menemukan anomali yang perlu ditelusuri ulang).
- [ ] Baseline V2-005 (bila sudah ada) sebagai pembanding disk/resource
      normal.
- [ ] Koordinasi: janitor berjalan tiap 30 menit — audit TIDAK menghentikan
      janitor (read-only), tapi SEMUA angka harus dicatat dengan timestamp
      snapshot karena janitor terus menghapus selama audit berjalan.

## D. Scope pemeriksaan (file/module)

| # | Area | Lokasi / cara periksa |
|---|---|---|
| D1 | Kode janitor | `tools/vps_disk_janitor.py` (baca penuh): semua kondisi hapus, query unpaired, pencocokan folder, TTL, lease, quarantine, grace period, cek owner/job/lifecycle/umur (ada/tidak per item P0.4) |
| D2 | Unit + jadwal janitor | `systemctl list-units` + `list-timers` (cari janitor); interval, user, cara trigger; apakah ada mode dry-run / flag simulasi di unit atau kode |
| D3 | Query unpaired + hitungan DB | DB read-only: `channels` (status × STAGING), bandingkan dengan angka 22 Nov 01:40 (707 / 237 / 470 / 673 / 203) — masih sama atau bergeser? |
| D4 | `deletion_audit` | Skema tabel + hitungan 24 jam per label/jalur + reklasifikasi ulang (join ke `channels` by status saat ini + status saat hapus bila tercatat); sampel file CONNECTED yang terhapus (nama + slot + waktu publish) |
| D5 | Proteksi lease | Kode lease (definisi, siapa yang memegang, berapa lama) + cek `deletion_audit` vs `operational_tasks` aktif: adakah file tugas aktif yang terhapus? |
| D6 | Pencocokan folder | Kode pencocokan (substring `key in folder_name` vs eksak/batas kata); enumerasi kunci tabrakan (awalan slot pendek vs slot panjang); hitung folder yang cocok oleh >1 kunci |
| D7 | TTL + tekanan disk | Kode ambang disk (>75% → 4 jam; ambang kritis lain?); riwayat pemicu dari log janitor; `df` saat audit |
| D8 | Referensi file di DB | Grep skema + kode: tabel/kolom yang menyimpan path lokal (`ingestion_items.local_path`, `loopgen_distribution`, `operational_tasks`, dll.); hitung referensi yatim (path di DB tapi file sudah tidak ada) vs file yatim (file ada tapi tanpa referensi) |
| D9 | File proses aktif | `ps` + `lsof` (bila tersedia) saat audit: file apa yang sedang dibaca FFmpeg/live/Stage 2; bandingkan dengan kandidat hapus janitor (apakah ada irisan?) |
| D10 | Jalur hapus lain (OQ-015) | Grep repo: `unlink`/`remove`/`rmtree`/`shutil` di luar janitor; cron/timer lain yang menghapus; auto-prune guard (DIAGNOSIS_DISK l.284) |

## E. Pertanyaan teknis yang harus dijawab

| # | Pertanyaan | Bukti yang menjawab |
|---|---|---|
| Q1 | Apa SEMUA kondisi penghapusan file di janitor? (daftar lengkap + lokasi kode) | D1 — kutipan kode per kondisi |
| Q2 | Berapa GB/file channel CONNECTED terhapus berlabel unpaired dalam 24 jam terakhir (dan 7 hari bila data ada)? | D3 + D4 — reklasifikasi dengan timestamp snapshot |
| Q3 | Apakah proteksi lease menutupi semua jalur hapus? Adakah file tugas aktif/leased yang terhapus? | D5 — kode lease + join `deletion_audit` vs tugas aktif |
| Q4 | Seberapa besar kerusakan pencocokan substring? (kunci tabrakan + folder terdampak) | D6 — enumerasi kunci + folder |
| Q5 | Bagaimana TTL berperilaku di bawah tekanan disk (ambang 75% → 4 jam)? Kapan terakhir terpicu? | D7 — kode + log janitor + `df` |
| Q6 | Apakah quarantine + grace period ada di kode? (berapa lama, file apa yang masuk, apa yang terjadi sesudahnya?) | D1 — bila tidak ada, gap vs P0.4 |
| Q7 | Inventaris referensi file di DB lengkap? Berapa referensi yatim vs file yatim? | D8 — skema + hitungan |
| Q8 | Bisakah janitor menghapus file yang sedang dipakai FFmpeg/live/Stage 2? (bukti kode + irisan kandidat hapus vs file terbuka) | D1 + D9 |
| Q9 | Apakah ada jalur hapus lain selain janitor? (OQ-015) | D10 — grep + cron/timer |
| Q10 | Apa rekomendasi perbaikan terurut + level persetujuan masing-masing? | Sintesis Q1–Q9 (termasuk R1 status-only + pencocokan eksak; quarantine/grace bila gap) |

## F. Perintah / metode audit (executable, read-only, host `masjavas-vps-baru`)

**F.1 — Verifikasi host + temukan janitor (satu panggilan):**

```bash
ssh -F /home/hatch/.ssh/config masjavas-vps-baru \
  'hostname; df -h / | tail -1; find /opt/masjavas-os -maxdepth 3 -name "*janitor*" 2>/dev/null; systemctl list-units --type=service --type=timer | grep -i janitor'
```

Catat hostname + path janitor + nama unit/timer. Semua perintah berikut
memakai hasil ini (`<JAN>` = path `tools/vps_disk_janitor.py`).

**F.2 — Baca kode janitor penuh (baca file):**

```bash
ssh -F /home/hatch/.ssh/config masjavas-vps-baru \
  'wc -l <JAN>; grep -n "STAGING-SLOT\|unpaired\|TTL\|ttl\|lease\|quarantine\|grace\|unlink\|remove\|rmtree\|dry" <JAN> | head -80'
```

Baca fungsi penghapusan sampai tuntas (jangan simpulkan dari nama fungsi
saja). Catat SEMUA kondisi hapus + nilai TTL + ambang disk.

**F.3 — Hitung ulang klasifikasi channel (DB read-only, timeout 30 dtk):**

```bash
ssh -F /home/hatch/.ssh/config masjavas-vps-baru \
  'python3 -c "import sqlite3; c=sqlite3.connect(\"file:<DB>?mode=ro\", uri=True, timeout=30); print(list(c.execute(\"SELECT status, COUNT(*) FROM channels GROUP BY status\"))); print(list(c.execute(\"SELECT COUNT(*) FROM channels WHERE channel_id LIKE '\''STAGING-SLOT-%'\''\"))); print(list(c.execute(\"SELECT COUNT(*) FROM channels WHERE status='\''CONNECTED'\'' AND channel_id LIKE '\''STAGING-SLOT-%'\''\")))"'
```

(Ganti `<DB>` dengan path hasil discovery V2-001/V2-002; bila DB terkunci,
tunggu 30 dtk dan coba sekali lagi; bila tetap terkunci, catat sebagai
temuan, jangan paksa.)

**F.4 — Reklasifikasi `deletion_audit` 24 jam (DB read-only):**

```bash
ssh -F /home/hatch/.ssh/config masjavas-vps-baru \
  'python3 -c "import sqlite3; c=sqlite3.connect(\"file:<DB>?mode=ro\", uri=True, timeout=30); print(list(c.execute(\"SELECT label, COUNT(*), ROUND(SUM(size_bytes)/1e9,2) FROM deletion_audit WHERE deleted_at >= datetime('\''now'\'','\''-1 day'\'') GROUP BY label\")))"'
```

Sesuaikan nama kolom/tabel dengan skema aktual (cek `.schema
deletion_audit` dulu). Reklasifikasi: join tiap baris terhapus ke
`channels` (status saat ini + status saat hapus bila tercatat) dan pisahkan
CONNECTED vs benar-benar UNPAIRED vs tanpa baris channel. Simpan sampel
10 file CONNECTED (nama + slot + waktu hapus + waktu publish).

**F.5 — Lease + file proses aktif (baca-saja):**

```bash
ssh -F /home/hatch/.ssh/config masjavas-vps-baru \
  'ps -ef | grep -i -E "ffmpeg|janitor" | grep -v grep | head -20; echo ---; grep -n "lease" <JAN> | head -30'
```

Bila `lsof` tersedia, daftar file terbuka oleh ffmpeg/live dan bandingkan
dengan pola kandidat hapus janitor.

**F.6 — Jalur hapus lain (OQ-015):**

```bash
ssh -F /home/hatch/.ssh/config masjavas-vps-baru \
  'grep -rn "os.unlink\|os.remove\|shutil.rmtree\|Path(.*).unlink" /opt/masjavas-os --include="*.py" | grep -v tests | grep -v janitor | head -30; echo ---; systemctl list-timers --all | grep -i -E "clean|prune|janitor|disk"'
```

**DILARANG dalam tiket ini:**

- Menjalankan janitor dalam mode hapus (atau memicu satu siklus hapus
  apa pun — termasuk "uji" dengan file dummy di direktori produksi).
- Mengubah klausa unpaired / TTL / lease / pencocokan (L3 — perbaikan
  janitor butuh persetujuan + rencana verifikasi; lihat §K).
- Menghapus file apa pun secara manual selama audit (termasuk file yang
  "jelas orphan" — penghapusan adalah perbaikan, tiket terpisah).
- Menghentikan/menunda timer janitor (mengubah perilaku produksi).
- Menyimpulkan janitor "aman" dari tidak adanya keluhan — butuh bukti
  reklasifikasi (Q2) + bukti proteksi (Q3/Q8).

## G.0 — Evidence manifest (wajib diisi saat eksekusi)

Setiap Q1–Q10 wajib punya satu baris manifest. Isi manifest BERSAMAAN
dengan eksekusi, bukan di akhir — `deletion_audit` bisa terotasi (OQ-014)
dan janitor terus menghapus tiap 30 menit, jadi angka snapshot cepat basi.

| Question ID | Command | Timestamp (WIB) | Exit code | Evidence | Finding | Confidence | Impact |
|---|---|---|---|---|---|---|---|
| Q1 | F.2 baca janitor penuh | | | | | | |
| Q2 | F.3 + F.4 reklasifikasi | | | | | | |
| Q3 | F.5 lease + join audit | | | | | | |
| Q4 | F.2 pencocokan + enumerasi | | | | | | |
| Q5 | F.2 + log janitor + df | | | | | | |
| Q6 | F.2 quarantine/grace | | | | | | |
| Q7 | D8 inventaris referensi | | | | | | |
| Q8 | F.5 ps/lsof + kode | | | | | | |
| Q9 | F.6 jalur hapus lain | | | | | | |
| Q10 | Sintesis Q1–Q9 | | | | | | |

Aturan manifest: sama seperti V2-001 §G.0 — `Confirmed` (bukti langsung
dari kode/DB/log), `Partial` (bukti tidak lengkap / hanya satu sumber),
`Unknown` (tidak dapat diverifikasi + alasannya dicatat). `Unknown` yang
eksplisit LEBIH BAIK daripada dugaan yang ditulis seolah fakta. Bukti
TIDAK BOLEH berisi nilai kredensial.

## G. Bukti yang harus dilampirkan (deliverable tiket)

1. `JANITOR_DELETE_CONDITIONS_V2-004.md` — Q1 + Q5 + Q6 (semua kondisi
   hapus + TTL/tekanan disk + quarantine/grace).
2. `JANITOR_RECLASSIFICATION_V2-004.md` — Q2 (reklasifikasi 24 jam/7 hari
   + sampel file CONNECTED + perbandingan baseline 40,92 GB).
3. `JANITOR_LEASE_AND_INUSE_V2-004.md` — Q3 + Q8 (proteksi lease + file
   proses aktif).
4. `JANITOR_MATCHING_AND_REFERENCES_V2-004.md` — Q4 + Q7 (tabrakan
   substring + inventaris referensi file DB).
5. `JANITOR_RECOMMENDATIONS_V2-004.md` — Q9 + Q10 (jalur hapus lain +
   rekomendasi terurut + level persetujuan + rencana uji) + jawaban
   OQ-013/014/015.
6. `EVIDENCE_MANIFEST_V2-004.md` — manifest Q1–Q10.
7. Semua file di `workspace/engineering/v2-004/` (buat foldernya saat
   eksekusi).

## H. Acceptance criteria (lulus / gagal)

**LULUS bila SEMUA terpenuhi:**

- [ ] Q1–Q10 terjawab dengan bukti; yang tidak bisa dijawab dicatat
      eksplisit sebagai "TIDAK DAPAT DIVERIFIKASI + alasannya".
- [ ] Semua kondisi penghapusan terinventarisasi dari kode (bukan dari
      ingatan/dokumen lama) — termasuk kondisi yang TIDAK terdokumentasi
      di P0.4 bila ditemukan.
- [ ] Reklasifikasi 24 jam dihitung ulang dari `deletion_audit` (bukan
      mengutip angka 22 Nov 01:40) — dengan timestamp snapshot.
- [ ] Proteksi lease dinilai dari kode + bukti hapus (bukan klaim).
- [ ] Quarantine + grace period dinyatakan ADA (dengan parameter) atau
      TIDAK ADA (gap vs P0.4).
- [ ] Rekomendasi perbaikan terurut + level persetujuan + rencana uji
      (orphan aman / CONNECTED tak boleh hapus / file dipakai FFmpeg).
- [ ] Nol file dihapus, nol kode/config diubah, nol timer dihentikan
      selama audit; janitor terus berjalan normal.

**GAGAL bila salah satu terjadi:**

- `deletion_audit` sudah terotasi sebelum ditangkap dan tidak ada salinan
  (→ bukti reklasifikasi hilang; eskalasi, jangan ditebak dari angka lama).
- Klasifikasi unpaired dinilai tanpa membaca kode janitor aktual
  (→ rekomendasi bisa salah sasaran seperti P16 V2-002).
- Proteksi lease / file-digunakan dinilai tanpa bukti (→ perbaikan bisa
  membuka lubang hapus file aktif).
- Rekomendasi perbaikan ditulis tanpa membedakan diagnosis vs perbaikan
  (→ perbaikan dieksekusi tanpa tiket L2/L3).
- Audit dilakukan di host yang salah (verifikasi host dulu — pelajaran F-E).

## I. Dampak jika gagal / jika tiket tidak dikerjakan

- File channel CONNECTED terus dihapus setelah 1 jam (bukan 12 jam) —
  setiap siklus janitor (30 menit) menambah file yang hilang prematur;
  retry/re-upload/verifikasi pasca-publish bisa gagal karena file lokal
  sudah tidak ada.
- Audit penghapusan terus salah label (40,92 GB/24 jam tercatat sebagai
  "unpaid/unpaired") — keputusan kapasitas disk + prioritas cleanup
  dibuat dari angka yang menyesatkan.
- Tanpa inventaris kondisi hapus + referensi file, perbaikan janitor
  (R1 status-only) bisa salah sasaran: ada kemungkinan kondisi hapus
  lain yang belum terpetakan ikut menghapus file CONNECTED.
- Desain sharding VPS 8 (V2.0: janitor per box + kepemilikan file dua
  pipeline) dirancang tanpa memahami perilaku hapus aktual — risiko
  membawa bug klasifikasi ke box baru.

## J. Batas tindakan yang diizinkan

- **BOLEH:** semua perintah §F (read-only); membaca kode/config/journal/
  DB/dokumen; `ps`/`lsof` baca-saja; menulis 6 dokumen bukti §G.
- **TIDAK BOLEH:** menjalankan janitor mode hapus; mengubah klausa/TTL/
  lease/pencocokan; menghapus file manual; menghentikan/menunda timer
  janitor; benchmark/load test di box live; eksekusi perbaikan apa pun
  (tiket terpisah).
- **Perbaikan (pasca-diagnosis) = USER** (L2/L3 sesuai temuan — lihat §K).
  Perbaikan yang sudah diusulkan di dokumen (R1 laporan bottleneck:
  filter status-only + pencocokan eksak) TETAP butuh persetujuan +
  rencana verifikasi sebelum dieksekusi — audit ini memverifikasi
  apakah R1 memang cukup, atau ada kondisi hapus lain yang juga harus
  diperbaiki.

## K. Pihak yang menyetujui langkah berikutnya

| Langkah berikut | Penyetuju |
|---|---|
| Perbaikan filter unpaired → status-only + pencocokan eksak (R1) | User (L3 — perubahan logika hapus file produksi; butuh rencana verifikasi + observasi disk 24 jam pasca-deploy) |
| Penambahan quarantine + grace period (bila Q6 = tidak ada) | User (L3 — perubahan perilaku hapus; desain quarantine dulu) |
| Perbaikan proteksi lease / file-digunakan (bila Q3/Q8 menemukan lubang) | User (L3 — menyentuh keselamatan file aktif) |
| Uji hapus (orphan aman / CONNECTED tak boleh hapus / file dipakai FFmpeg) | User (L3 — uji melibatkan penghapusan; jadwalkan + backup dulu) |
| Lanjut ke V2-005 dst. | Tidak butuh persetujuan baru (Batch 1 audit) |

## L. Estimasi & catatan eksekusi

- Estimasi: ±1–1,5 jam (termasuk tunggu SSH + baca kode janitor penuh +
  reklasifikasi DB; grep + hitungan cepat).
- Jalankan dari main/side chat (bukan proactivity handoff — konsistensi
  dengan V2-001).
- **Tangkap `deletion_audit` + hitungan channel SEGERA** — retensi audit
  belum diketahui (OQ-014); setiap jam berlalu, bukti hilang permanen
  dan janitor terus menghapus (siklus 30 menit).
- **Verifikasi host dulu** (`masjavas-vps-baru`, hostname `vmi3762689`,
  disk 145G) — pelajaran F-E V2-001: audit di host salah menghasilkan
  kesimpulan invalid.
- Bila `deletion_audit` sudah tidak menyimpan 24 jam penuh saat eksekusi,
  JANGAN ekstrapolasi dari angka 22 Nov 01:40 — catat jendela data yang
  tersedia + reklasifikasi dari jendela itu + tandai Q2 Partial.
- Angka-angka di §A adalah BASELINE 22 Nov 01:40 WIB (DOCUMENTED FACT
  NOT YET REVALIDATED) — eksekusi WAJIB menghitung ulang; bila angka
  bergeser (mis. 203 CONNECTED sudah berubah karena pairing baru),
  laporkan angka baru + selisihnya, jangan menimpa baseline di §A.
- Koordinasi dengan lane lain: jangan audit bersamaan dengan perbaikan
  janitor / deploy / restart — beri tahu sebelum mulai.


---

# TIKET V2-005 — BASELINE SNAPSHOT (AUDIT)

**Judul:** Snapshot baseline kinerja sistem (CPU/RAM/disk/IO/TX/load per proses
+ metrik publish + lock + live) — lantai pengukuran untuk semua gerbang Fase 0
**Prioritas:** P1 — CRITICAL | **Workstream:** P1.1 | **Owner:** Muse (audit)
**Status:** NOT_STARTED (tiket lengkap — eksekusi menunggu sesi interaktif + SSH)
**Dibuat:** 22 Nov 2026 (stub) — **Diperluas:** 22 Nov 2026 14:50 WIB
(Cycle 6, cron local-only)
**Dependency:** akses baca VPS (SSH Tailscale, host BENAR `masjavas-vps-baru`
— pelajaran F-E V2-002); V2-007 membantu interpretasi status job, BUKAN
blocker (snapshot ini memakai query yang sudah dikenal, bukan schema baru)
**Diblokir oleh:** tidak ada — membutuhkan akses baca VPS + baca DB read-only
+ baca journal live/worker

## A. Masalah

Semua gerbang Fase 0 (PLAN_PENAMBAHAN_VPS8 §2 scorecard 13 metrik) dan semua
klaim perbaikan diukur TERHADAP baseline — tetapi baseline yang dipakai hari
ini adalah angka-angka yang tersebar di 4 dokumen berbeda, dihitung pada
hari dan dengan metode yang berbeda, dan sebagian sudah terbukti salah alamat:

1. **Angka kematian encoder bertentangan:** scorecard memakai "5×/24 jam
   slot 706" (P16), sementara V2-002 membuktikan **12 kematian/17 jam
   slot 114** — bukan 706 — plus kematian #13 (~07:17) dan #14 (08:09:50).
   Baseline live yang benar belum pernah ditulis dalam satu dokumen.
2. **Sumber median delay belum dikonfirmasi:** "median delay 11,6 mnt,
   p90 30,2 mnt" (21 Nov) dikutip dari EVALUASI l.52–54, tetapi metode
   hitungnya (kolom DB mana, jendela hari mana, apakah termasuk tugas
   skip-audio) belum pernah dicatat — angka ini adalah LANTAI pengikat
   (user 22 Nov 12:40: baseline = lantai, satu metrik memburuk = gerbang
   gagal = rollback), jadi metodenya harus dapat diulang.
3. **Angka resource tersebar:** load 3,6–5,5, render 252% CPU, DB-lock
   31×/24 jam, disk 72% — masing-masing dari dokumen berbeda, tanpa
   satu snapshot koheren (per proses + per unit systemd + per lane)
   yang diambil dalam SATU jendela waktu yang sama.
4. **Churn belum di-baseline-kan:** 123 COMPLETED vs 333 SUPERSEDED
   (66% churn, 7 hari) dikutip dari RENCANA_ORCHESTRATOR — belum pernah
   dihitung ulang dari DB live sebagai bagian dari snapshot baseline.

Tanpa snapshot ini, setiap klaim "tidak memburuk" pasca-perubahan tidak
dapat dibuktikan — dan aturan rollback satu-metrik (sprint 13:08) tidak
punya angka acuan yang sah.

**Fakta terverifikasi (bukan asumsi):**

| Fakta | Nilai | Sumber |
|---|---|---|
| Publish 21 Nov | 206 PUBLISHED, 0 FAILED, 0 BLOCKED (237 tugas; +31 skip audio) | `EVALUASI_SEBELUM_PERBAIKAN_2026-11-22.md` l.16, l.52; `PLAN_PENAMBAHAN_VPS8_2026-11-22.md` §1.7 |
| Render 21 Nov (n=206) | rata-rata 651 dtk, median 463 dtk, p90 1.396 dtk, maks 2.979 dtk | EVALUASI l.52–54 |
| Delay terbit vs jadwal 21 Nov | median 11,6 mnt, p90 30,2 mnt, maks 566 mnt; 7 tugas terlambat >2 jam | EVALUASI l.53–54 |
| Publish 15–21 Nov | 15–18 Nov: 178–186/hari; 20–21 Nov: 198–206/hari | EVALUASI l.317 |
| Load | 3,6–5,5; didominasi render (rata-rata 252% CPU pada 58 tick governor, ingestion 0%) | EVALUASI l.107–108 |
| DB-lock | 31×/24 jam (live 24×); 26× "database is locked" di journal live 24 jam | EVALUASI l.108–109; VERIFIKASI_CPU l.119 |
| Disk | 72% (41 GB bebas) 22 Nov | EVALUASI l.19 |
| Worker | SATU proses Python + SATU event loop (`masjavas-os-worker`, PID 778998); lane tick: ingestion 10 dtk, loopgen 30 dtk, janitor 1800 dtk | EVALUASI l.103–108 |
| Live | 2 stream jalan (slot 114 + 706); ±4,2 Mbps TX per stream; TX live ±8–9 Mbps; TX box melonjak ke 60 Mbps (03:33–03:36) | EVALUASI l.254; VERIFIKASI_CPU l.96, l.206 |
| Semaphore render | render 2, longform 1, upload 3 (default kode) | EVALUASI l.55 |
| DB | Sudah WAL + busy_timeout 30 dtk | EVALUASI l.118 |
| Churn 7 hari | 123 COMPLETED vs 333 SUPERSEDED (66% churn) | RENCANA_ORCHESTRATOR (DOCUMENTED FACT NOT YET REVALIDATED — hitung ulang saat eksekusi) |
| Kematian encoder (koreksi) | 12×/17 jam slot 114 (bukan 706); #13 ~07:17, #14 08:09:50 | V2-002 (READY_FOR_REVIEW) + Cycle 4 addendum |
| Scorecard gerbang | 13 metrik PLAN §2 (baseline vs target Fase 0/Fase 1) | PLAN_PENAMBAHAN_VPS8 §2 |

**Asumsi yang BELUM terverifikasi (jangan diperlakukan sebagai fakta):**
angka churn 7 hari (123/333) belum dihitung ulang dari DB live; metode
hitung median/p90 delay belum dicatat; angka load/disk di atas adalah
cuplikan satu titik waktu, bukan rata-rata jendela.

**OPEN QUESTION (ditemukan saat perluasan tiket — wajib dijawab saat eksekusi):**
OQ-016 — dari kolom/ query DB mana median delay 11,6 mnt + p90 30,2 mnt
dihitung (apakah termasuk 31 tugas skip-audio; jendela hari = 21 Nov saja
atau 7 hari)? OQ-017 — angka churn 123 COMPLETED / 333 SUPERSEDED dihitung
dari tabel apa (`operational_tasks`? tabel lain?) dan jendela 7 hari yang
mana? OQ-018 — apakah ada snapshot baseline versi lama yang tersimpan
(cek `workspace/engineering/` + VPS) yang bisa dipakai sebagai pembanding
tren, atau snapshot ini adalah yang pertama?

**Dua pekerjaan berbeda (jangan dicampur):**

1. **Snapshot (tiket ini)** — ukur dan catat keadaan SEKARANG dengan
   metode yang dapat diulang; JANGAN mengubah apa pun.
2. **Perbandingan pasca-perubahan** — memakai snapshot ini sebagai acuan;
   pekerjaan terpisah (tiap perubahan membandingkan dirinya ke snapshot
   ini; bukan isi tiket ini).

## B. Target

1. Snapshot resource SATU jendela waktu: CPU/RAM/disk/IO/TX/load per
   proses + per unit systemd + per lane (publish/ingestion/LoopGen/live/
   janitor/web) — dengan timestamp + metode tiap angka.
2. Hitung ulang metrik publish dari DB: publish/hari (7 hari terakhir,
   per hari), delay vs jadwal (median/p90/maks, metode OQ-016 dicatat),
   render (rata-rata/median/p90/maks), FAILED/BLOCKED/QUEUED per hari.
3. Hitung ulang lock: "database is locked" per 24 jam (total + per
   service: live vs worker vs web) + metode hitung.
4. Hitung ulang churn 7 hari: COMPLETED vs SUPERSEDED (OQ-017) + status
   job lain per hari.
5. Baseline live: stream aktif, kematian encoder (rekonsiliasi P16 vs
   V2-002 — angka benar per slot), orphan yang masih ada, TX per stream.
6. Snapshot tersimpan sebagai dokumen BERVERSI (`BASELINE_MASJAVAS_V2_<tanggal>.md`)
   + query/perintah yang dipakai, sehingga snapshot BERIKUTNYA dapat
   diulang dengan metode identik.
7. Scorecard PLAN §2 terisi kolom baseline dari snapshot ini (bukan dari
   kutipan dokumen lama) — atau dicatat eksplisit metrik mana yang belum
   bisa diisi + alasannya.

## C. Prasyarat

- [ ] Akses SSH baca ke `masjavas-vps-baru` (host BENAR — verifikasi
      hostname `vmi3762689` + disk 145G sebelum snapshot, pelajaran F-E).
- [ ] Jendela snapshot ditetapkan di awal (mis. 24 jam terakhir +
      cuplikan "saat ini") dan dicatat di dokumen — semua angka merujuk
      ke jendela yang sama.
- [ ] Tidak ada restart/deploy/benchmark bersamaan selama snapshot
      (koordinasi: snapshot read-only, tapi restart mengubah angka).
- [ ] Baseline 21 Nov (EVALUASI l.52–54) dibaca sebagai pembanding —
      BUKAN sebagai pengganti hitung ulang.

## D. Scope pemeriksaan (file/module)

| # | Area | Lokasi / cara periksa |
|---|---|---|
| D1 | Resource per proses | `ps aux --sort=-%cpu` + `top -b -n1` + `free -h` + `df -h` + `iostat` (bila ada) — satu cuplikan + rata-rata load 1/5/15 mnt |
| D2 | Unit systemd | `systemctl list-units --type=service` (unit aplikasi: worker, live, web, janitor, timer) + `systemctl status` per unit + uptime masing-masing |
| D3 | Lane worker | Tick per lane (ingestion/loopgen/janitor/Stage 2/publish) dari journal worker + atribusi CPU per lane (bila tersedia — catat bila tidak bisa, lihat V2-003 soal atribusi buta) |
| D4 | Publish 7 hari | DB read-only `operational_tasks`: hitung per hari per status + delay vs jadwal (kolom waktu yang dipakai dicatat — OQ-016) + durasi render |
| D5 | Lock 24 jam | `journalctl` (worker + live + web) grep "database is locked" per unit + DB `audit_logs` bila relevan; catat metode hitung |
| D6 | Churn 7 hari | DB read-only: COMPLETED vs SUPERSEDED per hari (OQ-017) + status lain |
| D7 | Live baseline | Stream aktif + TX per stream + kematian encoder per slot (rekonsiliasi P16 vs V2-002) + orphan (cek `ps`, bandingkan dengan V2-002) |
| D8 | Disk + staging | `df -h` + ukuran direktori staging/cache/pool audio (20,5 GB pool audio — DOCUMENTED, verifikasi saat eksekusi) + tren pertumbuhan bila ada data historis |
| D9 | Web/API | `healthz` (status + latensi) + TX box saat ini + jumlah koneksi |
| D10 | Scorecard PLAN §2 | Isi 13 metrik dari D1–D9 (atau tandai belum terisi + alasan) |

## E. Pertanyaan teknis yang harus dijawab

| # | Pertanyaan | Bukti yang menjawab |
|---|---|---|
| Q1 | Berapa CPU/RAM/disk/IO/TX/load per proses + per unit systemd pada jendela snapshot? | D1 + D2 — angka + timestamp + metode |
| Q2 | Berapa publish/hari 7 hari terakhir (per hari, per status) + delay median/p90/maks dengan metode yang dicatat? | D4 (OQ-016) |
| Q3 | Berapa durasi render rata-rata/median/p90/maks (7 hari + 21 Nov pembanding)? | D4 |
| Q4 | Berapa "database is locked" per 24 jam (total + per service)? | D5 |
| Q5 | Berapa COMPLETED vs SUPERSEDED 7 hari (churn %) + status job lain? | D6 (OQ-017) |
| Q6 | Baseline live yang benar: stream aktif, kematian encoder per slot (rekonsiliasi P16 vs V2-002), orphan, TX per stream? | D7 |
| Q7 | Berapa ukuran staging/cache/pool + tren disk (apakah 41 GB bebas masih berlaku)? | D8 |
| Q8 | Apakah snapshot ini dapat diulang (query + perintah tersimpan, jendela tercatat)? | D10 + dokumen §G (metode lengkap) |
| Q9 | Scorecard PLAN §2: metrik mana terisi dari snapshot ini, mana yang masih kosong + mengapa? | D10 |
| Q10 | Apakah ada snapshot baseline lama untuk tren (OQ-018)? | Pencarian workspace + VPS |

## F. Perintah / metode audit (executable, read-only, host `masjavas-vps-baru`)

**F.1 — Verifikasi host + cuplikan resource (satu panggilan):**

```bash
ssh -F /home/hatch/.ssh/config masjavas-vps-baru \
  'hostname; uptime; free -h; df -h / /opt 2>/dev/null | tail -3; ps aux --sort=-%cpu | head -25'
```

Catat hostname `vmi3762689` + disk 145G (bila tidak cocok — STOP, host salah).

**F.2 — Unit systemd + status (baca-saja):**

```bash
ssh -F /home/hatch/.ssh/config masjavas-vps-baru \
  'systemctl list-units --type=service --state=running | grep -i -E "masjavas|live|worker|web|janitor"; systemctl list-timers --all | grep -i -E "masjavas|janitor"'
```

**F.3 — Lock 24 jam (journal, baca-saja):**

```bash
ssh -F /home/hatch/.ssh/config masjavas-vps-baru \
  'for u in masjavas-live.service masjavas-os-worker.service masjavas-web.service; do echo "== $u"; journalctl -u $u --since "24 hours ago" --no-pager 2>/dev/null | grep -c "database is locked"; done'
```

Sesuaikan nama unit dengan hasil F.2. Simpan output lengkap bila memungkinkan.

**F.4 — Hitung publish + delay + render + churn (DB read-only, timeout 30 dtk):**

```bash
ssh -F /home/hatch/.ssh/config masjavas-vps-baru \
  'python3 - <<PYEOF
import sqlite3
c = sqlite3.connect("file:<DB>?mode=ro", uri=True, timeout=30)
# GANTI <DB> dengan path hasil discovery (V2-001 F.2 / V2-007);
# kolom waktu + status dicatat apa adanya — JANGAN menebak nama kolom,
# baca schema dulu (PRAGMA table_info(operational_tasks)).
print(list(c.execute("SELECT status, COUNT(*) FROM operational_tasks GROUP BY status")))
PYEOF'
```

Query delay/render/churn ditulis SETELAH membaca nama kolom sebenarnya
(`PRAGMA table_info`) — catat query final verbatim di dokumen bukti.
**JANGAN** UPDATE/DELETE/INSERT. Bila DB terkunci, tunggu 30 dtk, coba
sekali lagi; bila tetap terkunci, catat sebagai temuan.

**F.5 — Live baseline (baca-saja):**

```bash
ssh -F /home/hatch/.ssh/config masjavas-vps-baru \
  'ps -ef | grep -i -E "ffmpeg|live_supervisor" | grep -v grep | head -20; ss -tlnp 2>/dev/null | head -20'
```

Bandingkan dengan temuan V2-002 (stream aktif, orphan, TX per stream).

**F.6 — Disk + staging (baca-saja):**

```bash
ssh -F /home/hatch/.ssh/config masjavas-vps-baru \
  'du -sh /opt/masjavas-os/data/* 2>/dev/null | sort -h | tail -15; df -h'
```

Sesuaikan path dengan hasil discovery. Catat ukuran pool audio + staging.

**DILARANG dalam tiket ini:**

- Mengubah konfigurasi, threshold, semaphore, atau jadwal job apa pun
  ("snapshot tidak mengubah sistem" — perubahan adalah tiket terpisah).
- Menjalankan benchmark / load test / uji concurrency (itu V2-008).
- Me-restart service/worker/web/live untuk "membersihkan" angka.
- Menghitung baseline dari DOKUMEN LAMA saja tanpa hitung ulang dari
  DB/log live (kutipan lama = pembanding, bukan pengganti).
- Menyimpulkan tren dari SATU cuplikan (snapshot = satu titik + 7 hari
  hitung ulang, bukan proyeksi).

## G.0 — Evidence manifest (wajib diisi saat eksekusi)

Setiap Q1–Q10 wajib punya satu baris manifest. Isi manifest BERSAMAAN
dengan eksekusi, bukan di akhir — snapshot adalah titik waktu; angka
"saat ini" berubah tiap jam.

| Question ID | Command | Timestamp (WIB) | Exit code | Evidence | Finding | Confidence | Impact |
|---|---|---|---|---|---|---|---|
| Q1 | F.1 + F.2 | | | `BASELINE_MASJAVAS_V2_<tanggal>.md` | | | |
| Q2 | F.4 publish/delay | | | `BASELINE_MASJAVAS_V2_<tanggal>.md` | | | |
| Q3 | F.4 render | | | `BASELINE_MASJAVAS_V2_<tanggal>.md` | | | |
| Q4 | F.3 lock | | | `BASELINE_MASJAVAS_V2_<tanggal>.md` | | | |
| Q5 | F.4 churn | | | `BASELINE_MASJAVAS_V2_<tanggal>.md` | | | |
| Q6 | F.5 live | | | `BASELINE_MASJAVAS_V2_<tanggal>.md` | | | |
| Q7 | F.6 disk | | | `BASELINE_MASJAVAS_V2_<tanggal>.md` | | | |
| Q8 | Metode §G | | | `BASELINE_MASJAVAS_V2_<tanggal>.md` | | | |
| Q9 | D10 scorecard | | | `BASELINE_MASJAVAS_V2_<tanggal>.md` | | | |
| Q10 | Pencarian snapshot lama | | | `BASELINE_MASJAVAS_V2_<tanggal>.md` | | | |

Aturan manifest: sama seperti V2-001 §G.0 — `Confirmed` (bukti langsung
dari DB/log/ps), `Partial` (bukti tidak lengkap / hanya satu sumber),
`Unknown` (tidak dapat diverifikasi + alasannya dicatat). `Unknown` yang
eksplisit LEBIH BAIK daripada dugaan yang ditulis seolah fakta. Bukti
TIDAK BOLEH berisi nilai kredensial. **Setiap angka di snapshot WAJIB
mencantumkan: nilai + jendela waktu + metode/query + sumber.**

## G. Bukti yang harus dilampirkan (deliverable tiket)

1. `BASELINE_MASJAVAS_V2_<tanggal>.md` — Q1–Q10 (snapshot berversi:
   resource per proses/unit/lane + publish 7 hari + delay + render +
   lock + churn + live + disk + scorecard PLAN §2 + metode hitung ulang).
2. `EVIDENCE_MANIFEST_V2-005.md` — manifest Q1–Q10 (bila dipisah dari
   dokumen baseline; bila digabung, catat di sini).
3. Semua file di `workspace/engineering/v2-005/` (buat foldernya saat eksekusi).

## H. Acceptance criteria (lulus / gagal)

**LULUS bila SEMUA terpenuhi:**

- [ ] Q1–Q10 terjawab dengan bukti; yang tidak bisa dijawab dicatat
      eksplisit sebagai "TIDAK DAPAT DIVERIFIKASI + alasannya".
- [ ] Setiap angka baseline mencantumkan nilai + jendela + metode/query.
- [ ] Publish/delay/render/churn dihitung ulang dari DB live (bukan
      dikutip dari dokumen lama) — atau dinyatakan eksplisit bila DB
      tidak terbaca + alasannya.
- [ ] Rekonsiliasi kematian encoder P16 vs V2-002 tercatat (angka benar
      per slot).
- [ ] Scorecard PLAN §2 terisi dari snapshot ini (atau metrik kosong
      dicatat + alasannya).
- [ ] Snapshot dapat diulang: query + perintah tersimpan verbatim.
- [ ] Nol perubahan sistem selama snapshot.

**GAGAL bila salah satu terjadi:**

- Baseline ditulis dari kutipan dokumen lama tanpa hitung ulang dari
  DB/log live.
- Angka tanpa jendela waktu / tanpa metode (tidak dapat diulang).
- Rekonsiliasi encoder (P16 vs V2-002) tidak dilakukan — gerbang memakai
  angka salah alamat.
- Snapshot diambil di host yang salah (verifikasi host dulu — pelajaran F-E).
- Perubahan sistem terjadi selama snapshot tanpa dicatat.

## I. Dampak jika gagal / jika tiket tidak dikerjakan

- Aturan rollback satu-metrik tidak punya angka acuan yang sah — setiap
  perubahan pasca-baseline tidak dapat dibuktikan "tidak memburuk".
- Gerbang Fase 0 (scorecard 13 metrik) memakai angka dari dokumen lama
  yang sebagian sudah terbukti salah alamat (kematian encoder).
- V2-008 (benchmark) tidak punya pembanding — "lebih cepat/lambat dari
  apa?" tidak terjawab.
- Capacity model VPS 8 (puncak ≈7,4 core, headroom ±0,6) memakai angka
  PROPOSAL, bukan baseline terukur — risiko salah ukuran box.

## J. Batas tindakan yang diizinkan

- **BOLEH:** semua perintah §F (read-only); membaca kode/config/journal/
  DB/dokumen; `healthz`; menulis dokumen bukti §G.
- **TIDAK BOLEH:** mengubah config/threshold/semaphore/jadwal; benchmark/
  load test; restart service; eksekusi perbaikan apa pun; mengambil
  snapshot di `masjavas-vps` (box lama).
- **Perbandingan pasca-perubahan = pekerjaan terpisah** (memakai snapshot
  ini sebagai acuan, bukan isi tiket ini).

## K. Pihak yang menyetujui langkah berikutnya

| Langkah berikut | Penyetuju |
|---|---|
| Penetapan snapshot ini sebagai baseline resmi gerbang Fase 0 | User (menyetujui angka + metode; snapshot ini hanya menyajikan bukti) |
| Perubahan apa pun yang dibandingkan ke baseline ini | User (L2/L3 sesuai perubahan — lihat tiket perubahannya) |
| Snapshot ulang (baseline baru) | Tidak butuh persetujuan baru bila metode sama; bila metode berubah, catat sebagai versi baru + beri tahu user |
| Lanjut ke V2-006 dst. | Tidak butuh persetujuan baru (Batch 1 audit) |

## L. Estimasi & catatan eksekusi

- Estimasi: ±1–1,5 jam (termasuk tunggu SSH + baca schema + hitung ulang).
- Jalankan dari main/side chat (bukan proactivity handoff — konsistensi
  dengan V2-001).
- **Tangkap cuplikan "saat ini" (F.1/F.2/F.5/F.6) dalam SATU sesi
  berurutan** — angka resource berubah tiap menit; jeda antar perintah
  dicatat.
- **Baca schema dulu sebelum query (F.4):** nama kolom `operational_tasks`
  sudah terbukti menyesatkan (`schedule_time` tidak ada — yang benar
  `schedule_time_epoch` + `target_date_wib`; `updated_at` epoch INTEGER —
  lihat AGENTS.md 12 Nov). JANGAN menebak nama kolom.
- Bila DB terkunci saat F.4, catat sebagai temuan (lock itu sendiri data
  baseline — Q4) — jangan paksa.
- Koordinasi dengan lane lain: jangan snapshot bersamaan dengan restart
  supervisor / deployment / benchmark — beri tahu sebelum mulai.
- Snapshot ini adalah versi pertama yang berversi; snapshot berikutnya
  memakai nama file tanggal baru (jangan timpa file ini).

---

# TIKET V2-006 — SERVICE INVENTORY (AUDIT)

**Judul:** Inventaris lengkap layanan sistem — systemd service + timer +
cron + proses Python + FFmpeg + background task FastAPI + scheduler +
sweeper + Fleet Runner + Resource Guard + janitor + upload/Drive sync —
dengan peta dependency antar-service dan daftar port publik Master
**Prioritas:** P1 — CRITICAL | **Workstream:** P1.2 | **Owner:** Muse (audit)
**Status:** NOT_STARTED (tiket lengkap — eksekusi menunggu sesi interaktif + SSH)
**Dibuat:** 22 Nov 2026 (stub) — **Diperluas:** 22 Nov 2026 18:50 WIB
(Cycle 7, cron local-only)
**Dependency:** akses baca VPS (SSH Tailscale, host BENAR `masjavas-vps-baru`
— pelajaran F-E V2-002); V2-005 membantu interpretasi beban per unit,
BUKAN blocker (inventaris ini memakai perintah baca langsung, bukan angka
baseline); hasil tiket ini adalah INPUT wajib V2-007 (peta penulis DB),
V2-008 (benchmark per worker), dan rencana pembagian 3 bagian P3
**Diblokir oleh:** tidak ada — membutuhkan akses baca VPS + baca file unit
systemd + baca journal + baca kode (lokasi proses)

## A. Masalah

Tidak ada satu pun dokumen yang memuat daftar lengkap apa saja yang
berjalan di Master live — dan setiap kali ada insiden, tim menemukan
proses yang tidak ada di peta:

1. **Peta proses tidak lengkap:** yang diketahui hanya "SATU proses
   worker + SATU event loop" (EVALUASI P3) dan "live supervisor proses
   terpisah" (V2-003 G12). Nama unit systemd yang sebenarnya, timer,
   cron, background task FastAPI, dan proses pendukung (tunnel, relay,
   guardian, sentinel) belum pernah dicatat dalam satu inventaris.
2. **Tiga proses pemulihan berpacu tanpa pemilik tunggal (B10, 22 Nov
   15:30):** supervisor systemd + proses manual 650871 + daemon
   sama-sama mencoba memulihkan stream 14:47 — sentinel ghost-kill
   114 2×, encoder ganda 114, feeder orphan purba. Tanpa inventaris
   yang memetakan SIAPA pemilik pemulihan tiap stream, perbaikan
   struktural "SATU pemilik pemulihan" (menunggu keputusan owner)
   tidak dapat dirancang.
3. **Nama unit systemd belum terverifikasi:** V2-005 F.3 memakai nama
   `masjavas-live.service`, `masjavas-os-worker.service`,
   `masjavas-web.service` — nama-nama itu ASUMSI (tiket menulis
   "sesuaikan nama unit dengan hasil F.2"). Bila nama itu salah,
   perintah audit V2-005 F.3 mengembalikan nol dan lock 24 jam
   terhitung salah.
4. **Jalur hapus / tulis DB tidak terpetakan ke proses:** V2-007 (peta
   penulis DB) dan V2-004 (janitor) sama-sama butuh jawaban "proses
   mana yang menulis/menghapus apa" — jawaban itu hanya bisa datang
   dari inventaris ini.
5. **Port publik Master belum didaftar:** pertanyaan terbuka dari
   rencana pendukung (STATUS_REGISTRY.yaml V2-006 note) — port apa
   saja yang terbuka ke publik di Master, dan layanan apa yang
   mendengarkannya, belum pernah dicatat.

Tanpa inventaris ini, V2-007 tidak tahu proses mana yang diaudit,
V2-008 tidak tahu worker mana yang di-benchmark, dan rencana
pembagian 3 bagian (P3, keputusan user 22 Nov 02:50) tidak punya
daftar unit yang harus dipisah.

**Fakta terverifikasi (bukan asumsi):**

| Fakta | Nilai | Sumber |
|---|---|---|
| Worker | SATU proses Python + SATU event loop (`masjavas-os-worker`, PID 778998); modul `worker.autonomous_daemon` | EVALUASI l.103–108; V2-003 G12 map |
| Lane tick worker | ingestion 10 dtk, loopgen 30 dtk, janitor 1800 dtk, dst. | EVALUASI l.107 |
| Live supervisor | Proses TERPISAH `python -m worker.live_supervisor`, PID 2052794, sejak 14:42 21 Nov | V2-003 G12 map |
| Web/API | uvicorn; port 8080 (terverifikasi TERTUTUP di box lama saat R1) | V2-002 fix plan R1 (07:05) |
| Governor | Modul `core/resource_governor` (arbiter/policy/measurement); guard loop = V2, admission = legacy | V2-003 (Cycle 4) |
| Janitor | Interval 30 menit; TTL normal 12 jam (4 jam bila disk >75%); jalur unpaired 1 jam; proteksi lease | V2-004 (Cycle 5) |
| FFmpeg per stream | 1 encoder + 1 feeder (+ relay audio) per stream; supervisor kini tepat 2 anak ffmpeg per stream (pasca-deploy R3–R8) | V2-002 fix report (Cycle 4) |
| Pemulihan berpacu (B10) | 3 proses: supervisor systemd + manual 650871 + daemon; guardian hanya di daemon; sentinel ghost-kill 114 2× (15:24:43 + 15:27:31) | Addendum 22 Nov 15:30 |
| Rencana pembagian 3 bagian | PUBLISH (scheduler+unduh+render+upload) / ISI BAHAN (ingestion+LoopGen+branding) / LIVE (supervisor+encoder+relay+governor); web/API + DB + tunnel = layanan bersama | EVALUASI P3 (keputusan user 02:50) |
| DB | WAL + busy_timeout 30 dtk | EVALUASI l.118 |
| Cron aplikasi box lama | TIDAK ADA cron aplikasi (terverifikasi saat R1) | V2-002 fix plan R1 (07:05) |
| Tunnel | Tunnel lama mati, tunnel baru aktif (pasca-R1) | V2-002 fix plan R1 (07:05) |
| Cron KEIN 5-menit | DISABLED sampai top-up + 1 live test | MEMORY.md / V2-001 |

**Fakta yang BELUM terverifikasi (jangan diperlakukan sebagai fakta):**
nama unit systemd sebenarnya (`masjavas-live.service` dsb. adalah asumsi
V2-005 F.3 — wajib diverifikasi F.2 tiket ini); apakah ada timer/cron di
host live selain tick in-process janitor; proses apa yang menjalankan
guardian dan sentinel (unit systemd / thread daemon / proses mandiri);
daftar port publik Master; proses mana saja yang menulis DB (→ V2-007).

**OPEN QUESTION (ditemukan saat perluasan tiket — wajib dijawab saat eksekusi):**
OQ-019 — apa nama unit systemd SEBENARNYA di host live (apakah
`masjavas-live.service` / `masjavas-os-worker.service` /
`masjavas-web.service` dari V2-005 F.3 benar, atau nama lain)? OQ-020 —
proses apa yang menjalankan guardian dan sentinel — unit systemd, thread
di dalam daemon, atau proses mandiri — dan siapa pemilik pemulihan tiap
stream (B10: tiga pemulih berpacu)? OQ-021 — apakah ada cron / systemd
timer di host live selain tick in-process (EVALUASI l.292 menyebut "Cron
sync" — apa itu, dan apakah masih berjalan)? OQ-022 — proses mana saja
yang membuka file DB untuk tulis (input peta penulis V2-007) — dan apakah
ada penulis DB di luar worker + live supervisor + web? OQ-023 — setiap
proses FFmpeg yang berjalan: encoder / feeder / relay / orphan — siapa
parent-nya (supervisor / daemon / yatim) dan stream mana yang dilayaninya?

**Dua pekerjaan berbeda (jangan dicampur):**

1. **Inventaris (tiket ini)** — catat APA yang berjalan, SIAPA parent-nya,
   APA dependency-nya, dan PORT apa yang didengarkan; JANGAN mengubah
   apa pun.
2. **Pemisahan / perbaikan struktur** — rencana pembagian 3 bagian P3,
   SATU pemilik pemulihan (B10), dan perubahan unit systemd adalah
   pekerjaan terpisah yang MEMAKAI inventaris ini sebagai input.

## B. Target

1. Daftar lengkap unit systemd aplikasi di host live: nama unit, status
   (active/enabled/failed), uptime, perintah start (ExecStart), dan
   Restart-policy masing-masing — dengan nama unit TERVERIFIKASI
   (OQ-019 terjawab, asumsi V2-005 F.3 dikoreksi atau dikonfirmasi).
2. Daftar lengkap timer systemd + cron (system + user) di host live —
   termasuk jawaban OQ-021 ("Cron sync" itu apa).
3. Daftar lengkap proses Python aplikasi yang berjalan: PID, PPID,
   cmdline lengkap, uptime, %CPU/%MEM saat cuplikan — dipetakan ke
   unit/lane (worker daemon, live supervisor, web, guardian, sentinel,
   dan proses lain yang ditemukan).
4. Daftar lengkap proses FFmpeg: PID, PPID, cmdline (stream/slot yang
   dilayani), peran (encoder/feeder/relay), dan status orphan vs
   terkelola — OQ-023 terjawab; bandingkan dengan temuan V2-002
   (orphan 706, feeder yatim).
5. Peta dependency antar-service: siapa memanggil/membutuhkan siapa
   (worker → DB; live supervisor → DB + ffmpeg + relay; web → DB;
   governor → lane; janitor → Drive/DB; guardian/sentinel →
   supervisor/daemon) — sebagai diagram + tabel.
6. Daftar port yang didengarkan (TCP/UDP) + proses pemiliknya, dengan
   penanda mana yang publik vs localhost-only (OQ port publik dari
   rencana pendukung terjawab).
7. Identifikasi proses yang berbagi SATU event loop (`worker.autonomous_daemon`)
   vs proses yang sudah terpisah (live supervisor) — titik kontensi
   tunggal tersisa dicatat eksplisit untuk rencana P3.
8. Identifikasi pemilik pemulihan tiap stream (OQ-020): supervisor
   systemd / daemon / guardian / sentinel / manual — dan konflik
   kepemilikan yang ditemukan (B10).
9. Inventaris tersimpan sebagai dokumen BERVERSI
   (`SERVICE_INVENTORY_V2_<tanggal>.md`) + perintah yang dipakai,
   sehingga inventaris BERIKUTNYA dapat diulang dengan metode identik.
10. Input untuk V2-007 (daftar penulis DB potensial — OQ-022) dan V2-008
    (daftar worker yang akan di-benchmark) tercatat eksplisit sebagai
    handoff di akhir dokumen bukti.

## C. Prasyarat

- [ ] Akses SSH baca ke `masjavas-vps-baru` (host BENAR — verifikasi
      hostname `vmi3762689` + disk 145G sebelum inventaris, pelajaran F-E).
- [ ] Tidak ada restart/deploy/benchmark bersamaan selama inventaris
      (koordinasi: inventaris read-only, tapi restart mengubah PID,
      parent, dan uptime).
- [ ] Temuan V2-002 (encoder/feeder/relay/orphan) dan V2-003 (daemon vs
      supervisor) dibaca sebagai pembanding — BUKAN sebagai pengganti
      pencatatan langsung.
- [ ] Cuplikan diambil dalam SATU sesi berurutan (PID dan parent
      berubah tiap menit; jeda antar perintah dicatat).

## D. Scope pemeriksaan (file/module)

| # | Area | Lokasi / cara periksa |
|---|---|---|
| D1 | Unit systemd | `systemctl list-units --type=service` + `list-timers --all` + `systemctl status` per unit aplikasi + `cat` file unit (`/etc/systemd/system/masjavas*` — path diverifikasi dulu, jangan ditebak) |
| D2 | Cron | `crontab -l` (root + ubuntu) + `ls /etc/cron*` + `systemctl list-timers` — jawab OQ-021 |
| D3 | Proses Python | `ps -ef` + `ps aux --sort=-%cpu` — PID/PPID/cmdline/uptime per proses aplikasi; petakan ke unit (D1) dan lane |
| D4 | Proses FFmpeg | `ps -ef \| grep ffmpeg` — PID/PPID/cmdline per proses; peran encoder/feeder/relay/orphan; bandingkan V2-002 |
| D5 | Event loop bersama | Cmdline `worker.autonomous_daemon` vs `worker.live_supervisor` — lane apa saja di dalam daemon (ingestion/loopgen/janitor/Stage 2/publish) dari journal + cmdline |
| D6 | Pemulihan (guardian/sentinel) | Cari proses/thread guardian + sentinel: `ps` + grep kode (`grep -rn "guardian\|sentinel" /opt/masjavas-os --include="*.py"`) + journal — OQ-020 |
| D7 | Port | `ss -tlnp` + `ss -ulnp` — port + proses pemilik + alamat bind (publik vs localhost) |
| D8 | Penulis DB | `lsof` / `/proc/<pid>/fd` untuk file DB — proses mana membuka DB tulis (OQ-022, handoff V2-007); catat bila `lsof` tidak tersedia + alternatif yang dipakai |
| D9 | File unit + config start | Isi file unit (ExecStart, Restart, WorkingDirectory, EnvironmentFile) + perintah start manual yang masih dipakai (proses 650871 B10 — bagaimana ia di-start?) |
| D10 | Dependency map | Rangkai D1–D9 menjadi diagram (mermaid) + tabel dependency; tandai titik kontensi tunggal + konflik kepemilikan pemulihan |

## E. Pertanyaan teknis yang harus dijawab

| # | Pertanyaan | Bukti yang menjawab |
|---|---|---|
| Q1 | Apa nama unit systemd aplikasi SEBENARNYA + status/uptime/Restart-policy masing-masing? | D1 (OQ-019) |
| Q2 | Timer systemd + cron apa saja yang ada di host live (termasuk "Cron sync")? | D2 (OQ-021) |
| Q3 | Proses Python aplikasi apa saja yang berjalan (PID/PPID/cmdline/uptime/CPU/MEM) dan ke unit/lane mana masing-masing dipetakan? | D3 |
| Q4 | Proses FFmpeg apa saja yang berjalan — peran, parent, stream yang dilayani, orphan vs terkelola? | D4 (OQ-023) |
| Q5 | Lane apa saja yang masih berbagi SATU event loop daemon vs sudah proses terpisah? | D5 |
| Q6 | Siapa menjalankan guardian + sentinel, dan siapa pemilik pemulihan tiap stream (apakah ada konflik kepemilikan)? | D6 (OQ-020) |
| Q7 | Port apa saja yang didengarkan, oleh proses apa, dan mana yang publik? | D7 |
| Q8 | Proses mana saja yang membuka file DB untuk tulis? | D8 (OQ-022) |
| Q9 | Bagaimana tiap unit di-start (file unit vs manual) — dan proses manual 650871 (B10) di-start dari mana? | D9 |
| Q10 | Apakah inventaris ini dapat diulang + apakah handoff ke V2-007/V2-008/P3 tercatat? | D10 + dokumen §G |

## F. Perintah / metode audit (executable, read-only, host `masjavas-vps-baru`)

**F.1 — Verifikasi host + cuplikan proses (satu panggilan):**

```bash
ssh -F /home/hatch/.ssh/config masjavas-vps-baru \
  'hostname; uptime; ps aux --sort=-%cpu | head -30'
```

Catat hostname `vmi3762689` + disk 145G (bila tidak cocok — STOP, host salah).

**F.2 — Unit systemd + timer (baca-saja):**

```bash
ssh -F /home/hatch/.ssh/config masjavas-vps-baru \
  'systemctl list-units --type=service --all | grep -i -E "masjavas|live|worker|web|janitor|guardian|sentinel"; echo ---TIMERS---; systemctl list-timers --all | grep -i -E "masjavas|janitor|sync"'
```

JANGAN mengasumsikan nama unit dari V2-005 F.3 — catat nama SEBENARNYA
apa adanya (OQ-019). Bila tidak ada unit bernama `masjavas*`, perluas
grep ke semua unit `active` dan saring manual.

**F.3 — Proses lengkap + FFmpeg + port (baca-saja):**

```bash
ssh -F /home/hatch/.ssh/config masjavas-vps-baru \
  'ps -ef | grep -i -E "python|ffmpeg|uvicorn|node" | grep -v grep; echo ---PORTS---; ss -tlnp 2>/dev/null; ss -ulnp 2>/dev/null'
```

Simpan output lengkap — ini tulang punggung inventaris (Q3/Q4/Q7).

**F.4 — File unit + cron (baca-saja):**

```bash
ssh -F /home/hatch/.ssh/config masjavas-vps-baru \
  'ls /etc/systemd/system/ | grep -i -E "masjavas|live|worker|web|janitor"; echo ---CRON---; crontab -l 2>&1; sudo crontab -l 2>&1 | head -20; ls /etc/cron.d/ 2>/dev/null'
```

Baca isi tiap file unit yang ditemukan (`cat`) — ExecStart + Restart +
WorkingDirectory dicatat verbatim. `sudo` hanya bila perlu dan hanya
untuk baca.

**F.5 — Guardian/sentinel + penulis DB (baca-saja):**

```bash
ssh -F /home/hatch/.ssh/config masjavas-vps-baru \
  'grep -rn -i "guardian\|sentinel" /opt/masjavas-os --include="*.py" -l | head; echo ---DB-FD---; ls -l /proc/*/fd 2>/dev/null | grep -i "\.db" | head -20'
```

Sesuaikan path kode/DB dengan hasil discovery (V2-001 F.2 / V2-007).
Bila `ls /proc/*/fd` tidak memberi hasil, pakai `lsof` bila tersedia —
catat metode yang dipakai.

**F.6 — Status per unit + journal ringkas (baca-saja):**

```bash
ssh -F /home/hatch/.ssh/config masjavas-vps-baru \
  'for u in $(systemctl list-units --type=service --state=running --no-legend | awk "{print \$1}" | grep -i -E "masjavas|live|worker|web"); do echo "== $u"; systemctl status $u --no-pager | head -12; done'
```

Sesuaikan daftar unit dengan hasil F.2 — JANGAN hardcode nama unit dari
V2-005.

**DILARANG dalam tiket ini:**

- Mengubah / me-restart / me-stop service, timer, atau proses apa pun
  ("inventaris tidak mengubah sistem").
- Menjalankan benchmark / load test / uji concurrency (itu V2-008).
- Membunuh proses orphan / feeder yatim yang ditemukan (itu perbaikan
  terpisah — catat saja).
- Menyimpulkan dependency dari DOKUMEN LAMA saja tanpa verifikasi
  `ps`/`systemctl`/`ss` live (dokumen lama = pembanding, bukan pengganti).
- Mengasumsikan nama unit dari tiket lain (V2-005 F.3) tanpa verifikasi
  F.2 — nama unit yang salah membuat seluruh inventaris salah alamat
  (pelajaran F-E V2-002).

## G.0 — Evidence manifest (wajib diisi saat eksekusi)

Setiap Q1–Q10 wajib punya satu baris manifest. Isi manifest BERSAMAAN
dengan eksekusi, bukan di akhir — PID dan parent berubah tiap menit.

| Question ID | Command | Timestamp (WIB) | Exit code | Evidence | Finding | Confidence | Impact |
|---|---|---|---|---|---|---|---|
| Q1 | F.2 + F.6 | | | `SERVICE_INVENTORY_V2_<tanggal>.md` | | | |
| Q2 | F.2 + F.4 | | | `SERVICE_INVENTORY_V2_<tanggal>.md` | | | |
| Q3 | F.1 + F.3 | | | `SERVICE_INVENTORY_V2_<tanggal>.md` | | | |
| Q4 | F.3 | | | `SERVICE_INVENTORY_V2_<tanggal>.md` | | | |
| Q5 | F.3 + F.5 | | | `SERVICE_INVENTORY_V2_<tanggal>.md` | | | |
| Q6 | F.5 + F.6 | | | `SERVICE_INVENTORY_V2_<tanggal>.md` | | | |
| Q7 | F.3 | | | `SERVICE_INVENTORY_V2_<tanggal>.md` | | | |
| Q8 | F.5 | | | `SERVICE_INVENTORY_V2_<tanggal>.md` | | | |
| Q9 | F.4 + F.6 | | | `SERVICE_INVENTORY_V2_<tanggal>.md` | | | |
| Q10 | D10 + dokumen §G | | | `SERVICE_INVENTORY_V2_<tanggal>.md` | | | |

Aturan manifest: sama seperti V2-001 §G.0 — `Confirmed` (bukti langsung
dari `systemctl`/`ps`/`ss`/file unit), `Partial` (bukti tidak lengkap /
hanya satu sumber), `Unknown` (tidak dapat diverifikasi + alasannya
dicatat). `Unknown` yang eksplisit LEBIH BAIK daripada dugaan yang
ditulis seolah fakta. Bukti TIDAK BOLEH berisi nilai kredensial.
**Setiap entri inventaris WAJIB mencantumkan: nama + PID/unit +
timestamp cuplikan + metode.**

## G. Bukti yang harus dilampirkan (deliverable tiket)

1. `SERVICE_INVENTORY_V2_<tanggal>.md` — Q1–Q10 (inventaris berversi:
   unit + timer/cron + proses Python + FFmpeg + event loop + guardian/
   sentinel + port + penulis DB + file unit + diagram dependency
   mermaid + handoff V2-007/V2-008/P3).
2. `SERVICE_DEPENDENCY_MAP_V2.md` — diagram + tabel dependency
   antar-service (bila dipisah dari dokumen inventaris; bila digabung,
   catat di sini).
3. `EVIDENCE_MANIFEST_V2-006.md` — manifest Q1–Q10 (bila dipisah dari
   dokumen inventaris; bila digabung, catat di sini).
4. Semua file di `workspace/engineering/v2-006/` (buat foldernya saat eksekusi).

## H. Acceptance criteria (lulus / gagal)

**LULUS bila SEMUA terpenuhi:**

- [ ] Q1–Q10 terjawab dengan bukti; yang tidak bisa dijawab dicatat
      eksplisit sebagai "TIDAK DAPAT DIVERIFIKASI + alasannya".
- [ ] Nama unit systemd TERVERIFIKASI dari `systemctl` live (OQ-019) —
      asumsi V2-005 F.3 dikonfirmasi atau dikoreksi secara eksplisit.
- [ ] Setiap proses tercatat dengan PID + PPID + cmdline + timestamp
      cuplikan (dapat diulang).
- [ ] Peta dependency antar-service ada (diagram + tabel) dan mencakup
      guardian/sentinel + pemilik pemulihan (OQ-020).
- [ ] Daftar port + proses pemilik + status publik/localhost tercatat (Q7).
- [ ] Handoff ke V2-007 (penulis DB), V2-008 (worker untuk benchmark),
      dan P3 (unit untuk pembagian 3 bagian) tercatat eksplisit.
- [ ] Nol perubahan sistem selama inventaris.

**GAGAL bila salah satu terjadi:**

- Inventaris ditulis dari dokumen lama tanpa verifikasi `ps`/`systemctl`/
  `ss` live.
- Nama unit diambil dari asumsi tiket lain tanpa verifikasi F.2.
- Proses FFmpeg orphan / feeder yatim tidak dicatat (temuan V2-002
  terlewat).
- Pemilik pemulihan (guardian/sentinel/supervisor/daemon) tidak
  dipetakan — B10 tidak bisa diperbaiki dari inventaris ini.
- Inventaris diambil di host yang salah (verifikasi host dulu —
  pelajaran F-E).
- Perubahan sistem terjadi selama inventaris tanpa dicatat.

## I. Dampak jika gagal / jika tiket tidak dikerjakan

- V2-007 (audit schema) tidak tahu proses mana yang menulis DB — peta
  penulis DB ditebak dari kode, bukan dari proses live.
- V2-008 (benchmark) tidak tahu worker mana yang diukur — modal vs
  konsumsi aktual tidak dapat dipetakan ke unit.
- Rencana pembagian 3 bagian (P3) tidak punya daftar unit yang harus
  dipisah — pemisahan dirancang dari ingatan, bukan inventaris.
- Perbaikan struktural B10 (SATU pemilik pemulihan) tidak dapat
  dirancang — konflik kepemilikan guardian/sentinel/supervisor/daemon
  tetap tersembunyi.
- V2-005 F.3 (lock per service) memakai nama unit yang mungkin salah —
  angka lock per service tidak sah.

## J. Batas tindakan yang diizinkan

- **BOLEH:** semua perintah §F (read-only); membaca kode/config/file
  unit/journal/DB (baca file descriptor saja, bukan isi data);
  `systemctl status/list-units`; `ss`; menulis dokumen bukti §G.
- **TIDAK BOLEH:** mengubah/me-restart/me-stop service/timer/proses;
  membunuh orphan/feeder yatim; mengubah config/cron; benchmark/load
  test; mengambil inventaris di `masjavas-vps` (box lama).
- **Perbaikan struktur (P3, B10) = pekerjaan terpisah** (memakai
  inventaris ini sebagai input, bukan isi tiket ini).

## K. Pihak yang menyetujui langkah berikutnya

| Langkah berikut | Penyetuju |
|---|---|
| Penetapan inventaris ini sebagai peta layanan resmi (acuan V2-007/V2-008/P3) | User (menyetujui daftar + peta; inventaris ini hanya menyajikan bukti) |
| Perubahan unit systemd / pemisahan proses (P3) / SATU pemilik pemulihan (B10) | User (L2/L3 sesuai perubahan — lihat tiket perubahannya) |
| Inventaris ulang (versi baru) | Tidak butuh persetujuan baru bila metode sama; bila metode berubah, catat sebagai versi baru + beri tahu user |
| Lanjut ke V2-007 dst. | Tidak butuh persetujuan baru (Batch 1 audit) |

## L. Estimasi & catatan eksekusi

- Estimasi: ±1–1,5 jam (termasuk tunggu SSH + baca file unit + petakan parent).
- Jalankan dari main/side chat (bukan proactivity handoff — konsistensi
  dengan V2-001).
- **Tangkap cuplikan proses (F.1/F.3) dalam SATU sesi berurutan** —
  PID/parent berubah tiap menit; jeda antar perintah dicatat.
- **JANGAN hardcode nama unit dari V2-005 F.3** — verifikasi dulu lewat
  F.2; bila nama berbeda, koreksi eksplisit di dokumen bukti + beri tahu
  agar V2-005 F.3 diperbarui.
- Bila `lsof` tidak tersedia saat F.5, pakai `/proc/<pid>/fd` (sudah di
  perintah) — catat metode yang dipakai.
- Koordinasi dengan lane lain: jangan inventaris bersamaan dengan restart
  supervisor / deployment / benchmark — beri tahu sebelum mulai.
- Inventaris ini adalah versi pertama yang berversi; inventaris berikutnya
  memakai nama file tanggal baru (jangan timpa file ini).

---

# TIKET V2-007 — DATABASE SCHEMA AUDIT (GERBANG TEKNIS)

**Judul:** Audit schema database aktual — dump schema lengkap + audit
`operational_tasks`/`channels`/`auth_tokens`/publish/live/LoopGen + indeks
+ constraint + status/transisi job + fungsi pembuat/pengubah status +
peta penulis DB + query penyebab lock + keputusan perluas-vs-tabel-baru
untuk dedup key
**Prioritas:** P1 — CRITICAL | **Workstream:** P1.3 | **Owner:** Muse (audit)
— keputusan schema/migration: user
**Status:** NOT_STARTED (tiket lengkap — eksekusi menunggu sesi interaktif + SSH)
**Dibuat:** 22 Nov 2026 (stub) — **Diperluas:** 22 Nov 2026 22:50 WIB
(Cycle 8, cron local-only)
**Dependency:** backup SQLite konsisten (prasyarat KERAS — JANGAN audit
tanpa backup); input V2-006 (peta penulis DB, OQ-022) membantu, BUKAN
blocker; V2-005 membantu interpretasi status job, BUKAN blocker
**Diblokir oleh:** tidak ada — membutuhkan akses baca VPS (SSH Tailscale,
host BENAR `masjavas-vps-baru` — pelajaran F-E V2-002) + baca kode
(lokasi schema + fungsi job)

## A. Masalah

Tidak ada migration/schema baru yang boleh jalan sebelum audit ini selesai
dan disetujui (checklist P1.3 + §14 — GERBANG TEKNIS) — tetapi schema
aktual belum pernah di-dump, sehingga gerbang ini menahan dua keputusan
berikutnya tanpa dasar bukti:

1. **Keputusan dedup (V2-010) tidak bisa diambil tanpa audit:** V2.0
   §9.3 menyatakan "jika `operational_tasks` sudah menjadi tabel utama,
   tambahkan atribut yang diperlukan ke tabel itu secara inkremental" —
   tetapi apakah `operational_tasks` memang dapat diperluas (kolom
   dedup key + generation + lease + fencing token) atau justru perlu
   tabel baru, hanya bisa dijawab dari schema + indeks + constraint
   aktual, bukan dari ingatan.
2. **Nama kolom sudah terbukti menyesatkan:** `operational_tasks` TIDAK
   punya `schedule_time` — yang benar `schedule_time_epoch` (INTEGER
   epoch UTC) + `target_date_wib` (TEXT 'YYYY-MM-DD'); `updated_at`
   epoch INTEGER, bukan string datetime (AGENTS.md 12 Nov). Query yang
   menebak nama kolom menghasilkan angka salah — audit WAJIB membaca
   schema dulu sebelum query apa pun.
3. **`loopgen_agent_tasks` KOSONG padahal worker Opsi 3 produktif
   19–21 Nov (OQ-001):** rekaman tugas agent tidak terlacak di DB ini —
   tidak diketahui apakah tugas agent dicatat di tabel lain, jalur
   tulisnya mati, atau tabel ini memang tidak pernah dipakai. Tanpa
   audit, V2-010 bisa merancang dedup di atas tabel yang tidak terisi.
4. **`operational_tasks` MAX(`updated_at`) 20 Nov ~09:05 (OQ-002):**
   jalur publish mungkin tidak menyentuh `updated_at` — artinya status
   dan transisi job tidak dapat dipercaya sebagai cerminan pekerjaan
   nyata sampai fungsi pembuat/pengubah status diaudit dari kode +
   data.
5. **Lock SQLite 31×/24 jam (R5) + 8 lock tanpa korelasi waktu
   (V2-002):** query penyebab lock belum diidentifikasi. Audit indeks +
   pola tulis adalah prasyarat diagnosis lock — dan lock adalah risiko
   langsung bagi migrasi apa pun (V2-010) yang menambah tulis.

Tanpa audit ini, V2-010 merancang dedup di atas asumsi, V2-008 tidak
punya peta tabel/metrik untuk benchmark, dan rencana pembagian 3 bagian
(P3) tidak tahu tabel mana yang dimiliki tiap bagian.

**Fakta terverifikasi (bukan asumsi):**

| Fakta | Nilai | Sumber |
|---|---|---|
| Host benar | `masjavas-vps-baru` (Contabo vmi3762689, disk 145G); DB live ±118 MB vs box lama 55 MB | V2-002 F-E (Cycle 3) |
| `operational_tasks` — kolom waktu | `schedule_time_epoch` INTEGER epoch detik UTC; `target_date_wib` TEXT 'YYYY-MM-DD'; `updated_at` epoch detik INTEGER | AGENTS.md 12 Nov (pola query andal) |
| `operational_tasks` — catatan awal | Tanpa `slot_number` (pakai `channel_id`); tanpa tabel `fleet_runs`/`job_queues`; tanpa kolom `audio_progress` | Stub V2-007 (DOCUMENTED NOT YET REVALIDATED — wajib diverifikasi F.3) |
| DB mode | WAL + busy_timeout 30 dtk | EVALUASI l.118 |
| Dedup key (frozen) | `channel_id + job_kind + generation`; satu job aktif per key; retry ≠ generation baru | V2.0 §8.2 |
| `channels` | 707 slot (237 CONNECTED / 470 UNPAIRED); 673 STAGING-SLOT-*; 203 CONNECTED ber-ID STAGING | V2-004 baseline (DOCUMENTED NOT YET REVALIDATED) |
| `auth_tokens` — ikatan vault | Token terenkripsi AES-256-GCM, AAD `CHANNEL=<id>::VAULT=<master_vault_id>`; `master_vault_id` ada di `channels` DAN `auth_tokens` | Insiden vault 12 Nov (AGENTS.md) |
| `deletion_audit` | Ada (bukti reklasifikasi janitor) — retensi belum diketahui | V2-004 Q2 / OQ-014 |
| Lock | 31×/24 jam (R5); V2-002: 8 lock, nol korelasi waktu dengan kematian encoder | EVALUASI; V2-002 |
| LoopGen live | Distribusi Stage 2 live sejak 19 Nov 11:08 — final langsung ke folder 01_VIDEOS channel | ALIGNMENT_SYNTHESIS |
| Isolasi tes | Jalur tulis-live `init_schema()` BELUM diperbaiki/dibuktikan — klaim isolasi tes tidak boleh bersandar pada niat saja | ALIGNMENT_SYNTHESIS (boundary repair) |

**Fakta yang BELUM terverifikasi (jangan diperlakukan sebagai fakta):**
daftar tabel lengkap + kolom/tipe/default tiap tabel; indeks + unique
constraint aktual; nilai status yang benar-benar ada di data (vs yang
didefinisikan kode); fungsi pembuat/pengubah status job; peta penulis DB
lengkap (handoff V2-006 OQ-022); query penyebab lock; apakah
`operational_tasks` dapat diperluas untuk dedup key + generation + lease
+ fencing token; apakah ada tabel `schema_version`/riwayat migrasi.

**OPEN QUESTION (ditemukan saat perluasan tiket — wajib dijawab saat eksekusi):**
OQ-024 — apakah koneksi aplikasi (`init_schema()` atau sejenisnya)
menulis ke DB saat audit membaca — dan apakah dump dari SALINAN backup
aman dari tulis itu? OQ-025 — tabel + kolom apa saja yang BENAR-BENAR
ada di DB live (daftar lengkap vs daftar stub — termasuk tabel yang
tidak disebut di stub)? OQ-026 — indeks apa yang hilang di jalur query
panas, dan apakah itu penyebab lock R5? OQ-027 — dapatkah
`operational_tasks` menampung dedup key + generation + lease + fencing
token (kolom apa yang kurang), atau perlu tabel baru? OQ-028 — nilai
status + transisi apa yang BENAR-BENAR terjadi di data (distinct status
per tabel + contoh transisi), vs yang didefinisikan kode?

**Dua pekerjaan berbeda (jangan dicampur):**

1. **Audit (tiket ini)** — catat schema + penulis + lock + jawaban
   perluas-vs-tabel-baru; JANGAN migration apa pun.
2. **Desain dedup + migration (V2-010)** — memakai hasil audit ini
   sebagai input; baru boleh mulai TEKNIS setelah hasil V2-007
   disetujui (aturan checklist §14).

## B. Target

1. Schema dump LENGKAP dari salinan backup DB live: semua tabel +
   kolom + tipe + default + PK/FK/unique/index — tersimpan sebagai
   dokumen BERVERSI (dapat diulang dengan metode identik).
2. Audit `operational_tasks`: kolom aktual (koreksi/konfirmasi catatan
   awal stub), nilai status yang ada di data, transisi yang terjadi,
   dan jawaban apakah tabel ini dapat diperluas untuk dedup (OQ-027).
3. Audit `channels` + `auth_tokens`: kolom kunci, ikatan vault ganda,
   dan konsistensi ID (STAGING vs CONNECTED — bandingkan V2-004).
4. Audit tabel publish/live/LoopGen: `ingestion_jobs`,
   `loopgen_jobs`, `loopgen_stage2_runs`, `loopgen_distribution`,
   `loopgen_agent_tasks` (OQ-001), `background_jobs`, `audit_logs`,
   `deletion_audit`.
5. Audit indeks + unique constraint: celah yang relevan dengan lock
   (OQ-026) dan dengan dedup key (OQ-027).
6. Inventaris fungsi pembuat job + pengubah status (grep kode) +
   peta transisi status per tabel (kode vs data — OQ-028, OQ-002).
7. Peta penulis DB: proses mana menulis tabel apa (dari V2-006
   OQ-022 + verifikasi kode) — termasuk penulis di luar worker +
   live supervisor + web bila ada.
8. Identifikasi query penyebab lock (pola indeks hilang + journal
   lock + transaksi panjang di kode) — R5.
9. OQ-001/OQ-002/OQ-024–OQ-028 terjawab dengan bukti; yang tidak
   bisa dijawab dicatat eksplisit sebagai "TIDAK DAPAT DIVERIFIKASI
   + alasannya".
10. Handoff ke V2-010 (keputusan perluas-vs-tabel-baru + kolom yang
    dibutuhkan dedup key/generation/lease/fencing) dan ke V2-008
    (tabel + metrik untuk benchmark) tercatat eksplisit di akhir
    dokumen bukti.

## C. Prasyarat

- [ ] Akses SSH baca ke `masjavas-vps-baru` (host BENAR — verifikasi
      hostname `vmi3762689` + disk 145G sebelum audit, pelajaran F-E).
- [ ] **Backup konsisten DULU (F.2) — JANGAN audit tanpa backup.**
      Audit membaca SALINAN backup, bukan file DB live (jalur tulis
      `init_schema()` belum terbukti aman — OQ-024).
- [ ] Temuan V2-002 (F-E host salah), V2-004 (janitor/`deletion_audit`),
      V2-006 (peta penulis DB, OQ-022) dibaca sebagai pembanding —
      BUKAN pengganti dump langsung.
- [ ] V2.0 §8.2 (dedup key + generation) + §9.3 (perluas-vs-tabel-baru)
      dibaca sebelum menilai OQ-027.
- [ ] Tidak ada restart/deploy/benchmark/migration bersamaan selama
      audit (koordinasi: tidak ada migration yang berjalan — aman —
      tapi restart mengubah mtime/ukuran DB dan merusak perbandingan
      backup).

## D. Scope pemeriksaan (file/module)

| # | Area | Lokasi / cara periksa |
|---|---|---|
| D1 | Backup + discovery DB | Lokasi file DB (hasil discovery V2-001 F.2 — JANGAN tebak path); ukuran + mtime + journal mode (`PRAGMA journal_mode`); backup konsisten (F.2) |
| D2 | Schema dump | `.schema` + `sqlite_master` dari SALINAN (F.3) — semua tabel + kolom + tipe + default + PK/FK/unique/index |
| D3 | `operational_tasks` | Kolom aktual; `COUNT` per status + `DISTINCT job_kind`; `MAX(updated_at)` + distribusi `target_date_wib`; contoh agregat (TANPA data sensitif) — OQ-002/OQ-027/OQ-028 |
| D4 | `channels` + `auth_tokens` | Kolom kunci; hitung STAGING vs CONNECTED; `master_vault_id` di kedua tabel; JANGAN tampilkan nilai token (hitung/skema saja) |
| D5 | Tabel publish/live | Skema + status distinct + baris per status (agregat saja) |
| D6 | Tabel LoopGen | `ingestion_jobs`, `loopgen_jobs`, `loopgen_stage2_runs`, `loopgen_distribution`, `loopgen_agent_tasks` (OQ-001: kosong atau tidak, dan sejak kapan), `background_jobs` |
| D7 | Tabel audit | `audit_logs`, `deletion_audit` (retensi — OQ-014 V2-004; hitung baris tertua/terbaru) |
| D8 | Indeks + constraint | `PRAGMA index_list`/`index_info` per tabel + unique constraint di dump; bandingkan dengan kolom yang dipakai di `WHERE`/`JOIN` query panas (OQ-026) |
| D9 | Fungsi job + penulis DB | Grep kode: pembuat job (`INSERT INTO operational_tasks` dkk.), pengubah status (`UPDATE ... SET status`), pembuka koneksi DB; peta penulis dari V2-006 OQ-022 diverifikasi dari kode |
| D10 | Lock + keputusan dedup | Journal grep "database is locked" 24 jam + transaksi panjang di kode; rangkai D1–D9 menjadi jawaban perluas-vs-tabel-baru (OQ-027) + handoff V2-010/V2-008 |

## E. Pertanyaan teknis yang harus dijawab

| # | Pertanyaan | Bukti yang menjawab |
|---|---|---|
| Q1 | Apa schema LENGKAP DB live (semua tabel + kolom + tipe + PK/FK/unique/index) — dan apakah daftar stub sudah lengkap (OQ-025)? | D1 + D2 (F.3) |
| Q2 | `operational_tasks`: kolom aktual apa saja, status + `job_kind` apa yang ada di data, dan apakah `updated_at` bergerak mengikuti publish (OQ-002)? | D3 (F.4) |
| Q3 | `channels` + `auth_tokens`: kolom kunci apa saja, dan apakah ikatan vault ganda konsisten (OQ dari insiden 12 Nov)? | D4 (F.4) |
| Q4 | Tabel publish/live/LoopGen: skema + status + apakah `loopgen_agent_tasks` benar-benar kosong (OQ-001)? | D5 + D6 (F.4) |
| Q5 | Indeks + unique constraint: apa yang ada, apa yang hilang di jalur query panas (OQ-026)? | D8 (F.6) |
| Q6 | Fungsi apa saja yang membuat job dan mengubah status — dan apakah transisi di kode cocok dengan data (OQ-028)? | D9 (F.5) |
| Q7 | Proses mana saja yang menulis DB, tabel apa yang ditulis masing-masing (OQ-022 dari V2-006)? | D9 + D1 (F.5) |
| Q8 | Query/transaksi apa yang menyebabkan lock SQLite 31×/24 jam (R5)? | D8 + D10 (F.6) |
| Q9 | Dapatkah `operational_tasks` diperluas untuk dedup key + generation + lease + fencing token, atau perlu tabel baru (OQ-027)? | D2 + D3 + D8 (F.3/F.4/F.6) |
| Q10 | Apakah audit ini dapat diulang + apakah handoff ke V2-010/V2-008 tercatat? | D10 + dokumen §G |

## F. Perintah / metode audit (executable, read-only, host `masjavas-vps-baru`)

**F.1 — Verifikasi host + discovery DB (satu panggilan):**

```bash
ssh -F /home/hatch/.ssh/config masjavas-vps-baru \
  'hostname; df -h /opt/masjavas-os 2>/dev/null | tail -1; ls -la /opt/masjavas-os/*.db /opt/masjavas-os/**/*.db 2>/dev/null | head; find /opt/masjavas-os -maxdepth 3 -name "*.db" 2>/dev/null | head'
```

Catat hostname `vmi3762689` + disk 145G (bila tidak cocok — STOP, host
salah). **JANGAN tebak path DB** — pakai hasil discovery (V2-001 F.2)
dan catat path yang dipakai di semua perintah berikut sebagai
`<DB>`; bila ada LEBIH DARI SATU file `.db`, catat semuanya dan
tanyakan/putuskan mana DB utama sebelum lanjut (pelajaran F-E).

**F.2 — Backup konsisten + verifikasi backup (baca-saja terhadap live):**

```bash
ssh -F /home/hatch/.ssh/config masjavas-vps-baru \
  'sqlite3 <DB> ".backup /tmp/v2-007-backup.db" && sqlite3 /tmp/v2-007-backup.db "PRAGMA integrity_check; PRAGMA journal_mode; SELECT count(*) FROM sqlite_master WHERE type=\"table\";" && ls -la /tmp/v2-007-backup.db'
```

Bila `sqlite3` CLI tidak tersedia, pakai `VACUUM INTO
'/tmp/v2-007-backup.db'` lewat `python3 -c` dengan `timeout=30`
(AGENTS.md 19 Nov: koneksi langsung ke DB live bisa
`database is locked` — selalu `timeout=30`). Backup GAGAL =
STOP — jangan audit tanpa backup. Hapus salinan dari `/tmp` VPS
seusai audit (seperti skrip V2-001).

**F.3 — Schema dump dari SALINAN (bukan file live):**

```bash
ssh -F /home/hatch/.ssh/config masjavas-vps-baru \
  'sqlite3 /tmp/v2-007-backup.db ".schema" > /tmp/v2-007-schema.sql; sqlite3 /tmp/v2-007-backup.db "SELECT type,name,sql FROM sqlite_master ORDER BY type,name;" | head -100; wc -l /tmp/v2-007-schema.sql'
```

Simpan `/tmp/v2-007-schema.sql` lengkap — ini tulang punggung audit
(Q1/Q9). **Semua query berikut memakai salinan
`/tmp/v2-007-backup.db`, BUKAN `<DB>` live.**

**F.4 — Agregat status + kolom kunci (dari salinan; baca schema dulu):**

```bash
ssh -F /home/hatch/.ssh/config masjavas-vps-baru \
  'sqlite3 /tmp/v2-007-backup.db "SELECT status, count(*) FROM operational_tasks GROUP BY status; SELECT job_kind, count(*) FROM operational_tasks GROUP BY job_kind LIMIT 30; SELECT datetime(max(updated_at), \"unixepoch\"), datetime(min(updated_at), \"unixepoch\") FROM operational_tasks; SELECT count(*) FROM loopgen_agent_tasks; SELECT count(*) FROM channels WHERE channel_id LIKE \"STAGING-SLOT-%\";"'
```

**Baca schema (F.3) dulu sebelum query (F.4)** — nama kolom sudah
terbukti menyesatkan (`schedule_time` tidak ada — AGENTS.md 12 Nov).
Sesuaikan nama kolom/tabel dengan dump; bila tabel di F.4 tidak ada
di dump, catat sebagai temuan (OQ-025), jangan paksa query.
**TANPA data sensitif:** agregat + hitung saja — JANGAN `SELECT *`
dari `auth_tokens` atau tampilkan nilai token/kunci.

**F.5 — Fungsi pembuat/pengubah status + penulis DB (baca kode):**

```bash
ssh -F /home/hatch/.ssh/config masjavas-vps-baru \
  'grep -rn "INSERT INTO operational_tasks\|INSERT INTO loopgen_\|UPDATE operational_tasks SET\|SET status" /opt/masjavas-os --include="*.py" | head -40; echo ---CONN---; grep -rn "sqlite3.connect\|aiosqlite.connect\|init_schema" /opt/masjavas-os --include="*.py" | head -30'
```

Sesuaikan path kode dengan hasil discovery. Catat file + nomor baris
tiap fungsi (dapat diulang). Bandingkan peta penulis dengan V2-006
OQ-022 — cocok atau bertentangan, catat eksplisit.

**F.6 — Indeks + bukti lock (baca-saja):**

```bash
ssh -F /home/hatch/.ssh/config masjavas-vps-baru \
  'sqlite3 /tmp/v2-007-backup.db "SELECT name FROM sqlite_master WHERE type=\"index\" ORDER BY name;" ; echo ---LOCKS---; journalctl --since "24 hours ago" 2>/dev/null | grep -i -c "database is locked"; echo ---INDEX-INFO---; sqlite3 /tmp/v2-007-backup.db "PRAGMA index_list(operational_tasks);"'
```

Bila `journalctl` tidak memberi hasil, pakai grep log aplikasi yang
ditemukan saat discovery — catat metode yang dipakai. Ulangi
`PRAGMA index_list` untuk tiap tabel D3–D7 yang relevan dengan lock.

**DILARANG dalam tiket ini:**

- Migration schema apa pun (ALTER/CREATE/DROP) — itu V2-010, setelah
  V2-007 disetujui (aturan checklist §14).
- Menulis ke DB live dalam bentuk apa pun — termasuk koneksi yang
  memicu `init_schema()` (jalur tulis-live belum terbukti aman —
  OQ-024); audit HANYA dari salinan backup (F.2/F.3).
- Menampilkan / menyalin nilai token, kunci, atau kredensial apa pun
  (hitung + skema saja).
- Query ke DB live langsung bila terkunci — pakai salinan (F.2).
- Menyimpulkan status/transisi dari KODE saja tanpa data (atau
  sebaliknya) — Q2/Q6 wajib keduanya (OQ-002/OQ-028).
- Mengambil schema di `masjavas-vps` (box lama) — host salah membuat
  seluruh audit salah alamat (pelajaran F-E V2-002).

## G.0 — Evidence manifest (wajib diisi saat eksekusi)

Setiap Q1–Q10 wajib punya satu baris manifest. Isi manifest BERSAMAAN
dengan eksekusi, bukan di akhir — backup dan schema berubah tiap jam.

| Question ID | Command | Timestamp (WIB) | Exit code | Evidence | Finding | Confidence | Impact |
|---|---|---|---|---|---|---|---|
| Q1 | F.3 | | | `DATABASE_SCHEMA_AUDIT_V2_<tanggal>.md` | | | |
| Q2 | F.4 | | | `DATABASE_SCHEMA_AUDIT_V2_<tanggal>.md` | | | |
| Q3 | F.4 | | | `DATABASE_SCHEMA_AUDIT_V2_<tanggal>.md` | | | |
| Q4 | F.4 | | | `DATABASE_SCHEMA_AUDIT_V2_<tanggal>.md` | | | |
| Q5 | F.6 | | | `DATABASE_SCHEMA_AUDIT_V2_<tanggal>.md` | | | |
| Q6 | F.5 | | | `DATABASE_SCHEMA_AUDIT_V2_<tanggal>.md` | | | |
| Q7 | F.5 | | | `DATABASE_WRITER_MAP_V2.md` | | | |
| Q8 | F.6 | | | `DATABASE_SCHEMA_AUDIT_V2_<tanggal>.md` | | | |
| Q9 | F.3 + F.4 + F.6 | | | `DATABASE_MIGRATION_PLAN_V2.md` | | | |
| Q10 | D10 + dokumen §G | | | `DATABASE_SCHEMA_AUDIT_V2_<tanggal>.md` | | | |

Aturan manifest: sama seperti V2-001 §G.0 — `Confirmed` (bukti langsung
dari dump schema salinan backup + agregat + grep kode), `Partial` (bukti
tidak lengkap / hanya satu sumber), `Unknown` (tidak dapat diverifikasi +
alasannya dicatat). `Unknown` yang eksplisit LEBIH BAIK daripada dugaan
yang ditulis seolah fakta. Bukti TIDAK BOLEH berisi nilai kredensial.
**Setiap entri schema WAJIB mencantumkan: nama tabel/kolom + timestamp
backup + metode (salinan backup, bukan live).**

## G. Bukti yang harus dilampirkan (deliverable tiket)

1. `DATABASE_SCHEMA_AUDIT_V2_<tanggal>.md` — Q1–Q8 + Q10 (schema dump
   berversi: semua tabel + kolom + indeks + constraint + status
   distinct + jawaban OQ-001/OQ-002/OQ-024–OQ-028).
2. `DATABASE_WRITER_MAP_V2.md` — peta penulis DB (Q7): proses →
   tabel yang ditulis → fungsi/koneksi (verifikasi V2-006 OQ-022
   dari kode).
3. `DATABASE_MIGRATION_PLAN_V2.md` — Q9 (RENCANA saja, BUKAN
   eksekusi): keputusan perluas-vs-tabel-baru + kolom yang dibutuhkan
   (dedup key + generation + lease + fencing token) + backward
   compatibility + urutan migration yang diusulkan untuk V2-010.
   Eksekusi migration menunggu V2-010 + persetujuan user.
4. `EVIDENCE_MANIFEST_V2-007.md` — manifest Q1–Q10 (bila dipisah dari
   dokumen audit; bila digabung, catat di sini).
5. Semua file di `workspace/engineering/v2-007/` (buat foldernya saat eksekusi).

## H. Acceptance criteria (lulus / gagal)

**LULUS bila SEMUA terpenuhi:**

- [ ] Q1–Q10 terjawab dengan bukti; yang tidak bisa dijawab dicatat
      eksplisit sebagai "TIDAK DAPAT DIVERIFIKASI + alasannya".
- [ ] Schema dump lengkap dari SALINAN backup (bukan file live) +
      `integrity_check` backup tercatat (OQ-024 terjawab).
- [ ] Catatan awal stub dikonfirmasi atau dikoreksi eksplisit
      (`slot_number` / `fleet_runs` / `job_queues` / `audio_progress`).
- [ ] OQ-001 (`loopgen_agent_tasks` kosong) + OQ-002 (`updated_at`
      20 Nov) terjawab dari data, bukan dugaan.
- [ ] Keputusan perluas-vs-tabel-baru (OQ-027) tercatat dengan alasan
      schema (kolom + indeks + constraint yang mendasarinya).
- [ ] Peta penulis DB ada dan dicocokkan dengan V2-006 OQ-022.
- [ ] Handoff ke V2-010 (kolom dedup yang dibutuhkan) dan V2-008
      (tabel/metrik benchmark) tercatat eksplisit.
- [ ] Nol perubahan schema + nol tulis ke DB live selama audit;
      salinan backup dihapus dari `/tmp` VPS seusai audit.

**GAGAL bila salah satu terjadi:**

- Audit diambil di host yang salah (verifikasi host dulu —
  pelajaran F-E).
- Audit membaca file DB live langsung tanpa backup (risiko tulis
  `init_schema()` + lock).
- Migration apa pun dijalankan selama audit.
- Nilai token/kunci tampil di bukti.
- Status/transisi disimpulkan dari kode saja atau dari data saja
  (Q2/Q6 butuh keduanya).
- Keputusan perluas-vs-tabel-baru tidak berdasar schema dump.
- `loopgen_agent_tasks` kosong tidak dijelaskan (OQ-001 terlewat).

## I. Dampak jika gagal / jika tiket tidak dikerjakan

- V2-010 (dedup) tidak bisa mulai TEKNIS — gerbang teknis (§14)
  menahan desain + migration; V2.0 §9.3 tidak bisa diputuskan.
- V2-008 (benchmark) tidak tahu tabel/metrik mana yang diukur —
  modal vs konsumsi aktual tidak dapat dipetakan ke beban DB.
- Rencana pembagian 3 bagian (P3) tidak tahu tabel mana yang dimiliki
  tiap bagian — pemisahan dirancang dari ingatan, bukan schema.
- Lock R5 (31×/24 jam) tetap tanpa penyebab — migrasi apa pun yang
  menambah tulis berisiko memperparah lock tanpa disadari.
- Status job tetap tidak dapat dipercaya (OQ-002) — baseline V2-005
  dan gerbang Fase 0 memakai angka yang mungkin basi.

## J. Batas tindakan yang diizinkan

- **BOLEH:** semua perintah §F (read-only terhadap live; tulis HANYA
  file backup + schema dump di `/tmp` VPS); membaca kode/config/
  journal/DB salinan; `sqlite3`/`PRAGMA` terhadap SALINAN;
  menulis dokumen bukti §G.
- **TIDAK BOLEH:** migration apa pun; menulis ke DB live (termasuk
  via `init_schema()`); menampilkan nilai token/kunci; mengambil
  schema di `masjavas-vps` (box lama); `SELECT *` dari `auth_tokens`.
- **Desain dedup + migration (V2-010) = pekerjaan terpisah** (memakai
  hasil audit ini sebagai input, bukan isi tiket ini).

## K. Pihak yang menyetujui langkah berikutnya

| Langkah berikut | Penyetuju |
|---|---|
| Penetapan audit schema ini sebagai gerbang teknis lolos (V2-007 APPROVED) | User (menyetujui schema dump + jawaban OQ-027; audit ini hanya menyajikan bukti) |
| Desain dedup + migration (V2-010) | User (L2/L3 sesuai perubahan — lihat tiket V2-010; TIDAK boleh mulai teknis sebelum V2-007 disetujui) |
| Migration schema apa pun | User (L3 — setelah V2-007 disetujui + V2-010 selesai) |
| Audit schema ulang (versi baru) | Tidak butuh persetujuan baru bila metode sama; bila metode berubah, catat sebagai versi baru + beri tahu user |
| Lanjut ke V2-008 + V2-009 | Tidak butuh persetujuan baru (Batch 1 audit) |

## L. Estimasi & catatan eksekusi

- Estimasi: ±1–1,5 jam (termasuk tunggu SSH + backup + baca schema + grep kode).
- Jalankan dari main/side chat (bukan proactivity handoff — konsistensi
  dengan V2-001).
- **Backup DULU (F.2), baru dump (F.3), baru query (F.4)** — urutan ini
  melindungi DB live; JANGAN dibalik.
- **Baca schema dulu sebelum query (F.4):** nama kolom sudah terbukti
  menyesatkan (`schedule_time` tidak ada — AGENTS.md 12 Nov). JANGAN
  menebak nama kolom/tabel.
- Bila DB terkunci saat F.2, catat sebagai temuan (lock itu sendiri data
  — R5/Q8) — jangan paksa; coba lagi sekali dengan `timeout=30`.
- Koordinasi dengan lane lain: jangan audit bersamaan dengan restart
  supervisor / deployment / benchmark — beri tahu sebelum mulai.
- Audit ini adalah versi pertama yang berversi; audit berikutnya memakai
  nama file tanggal baru (jangan timpa file ini).


---

# TIKET V2-008 — RESOURCE BUDGET BENCHMARK (STUB)

**Prioritas:** P1 | **Workstream:** P1.4 | **Status:** NOT_STARTED
**Dependency:** V2-005 baseline

**Scope:** ukur konsumsi aktual per worker (ingestion, Stage 2, live,
publish, web/API) — CPU, peak RAM, disk staging/cache, TX; uji Stage 2
concurrency 1 lalu 2 (3–4 hanya bila aman); bedakan modal vs konsumsi
aktual vs ambang gate.
**Batas:** benchmark ringan yang tidak mengganggu publish/live; uji
concurrency Stage 2 butuh persetujuan bila menyentuh antrean produksi.

---

# TIKET V2-009 — VPS 8 PRICE + STORAGE COST VERIFICATION (STUB — GERBANG BIAYA)

**Prioritas:** P1 | **Workstream:** P6.1 + P6.2 | **Status:** NOT_STARTED
**Dependency:** panel Contabo + quote object storage

**Scope:** verifikasi harga Master aktual + VPS 8 aktual + pajak + setup fee
+ periode billing + cancellation policy di panel Contabo; hitung total vs
cap $34,67/bln; hitung biaya object storage (DB size + pertumbuhan +
retensi + egress + request) untuk Litestream + Restic.
**Praktis:** panel Contabo dilindungi Cloudflare Turnstile yang TIDAK BISA
dilewati browser cloud Muse — verifikasi harga butuh user cek sendiri di
browsernya atau ambil alih browser live.
**Batas:** verifikasi saja; JANGAN beli/langganan apa pun.

---

# TIKET V2-010 — DEDUPLICATION DESIGN PROPOSAL (STUB)

**Prioritas:** P1 | **Workstream:** P3.2 (desain saja) | **Status:** NOT_STARTED
**Dependency:** V2-007 DISETUJUI (keras — tidak bisa mulai teknis sebelumnya)

**Scope:** rancang `deduplication_key` (`channel_id + job_kind + generation`)
+ aturan generation + unique constraint + status aktif, BERDASARKAN hasil
audit schema V2-007. Proposal desain + migration plan (backward-compatible) —
TANPA eksekusi migration.
**Aturan checklist §14:** V2-010 belum boleh melakukan migration sebelum
hasil V2-007 disetujui.
**Batas:** desain + dokumen saja; JANGAN sentuh schema.

---

# BATCH 1 TRACKER

> Sumber status resmi: `STATUS_REGISTRY.yaml`. Tabel ini ringkasan registry.

| ID | Tiket | Prioritas | Status | Bukti |
|---|---|---|---|---|
| V2-001 | KEIN dependency/fallback decision (audit) | P0 | EXECUTED (audit 22 Nov 05:12–05:20 DI HOST SALAH — bukti tidak berlaku untuk target; Q5/Q6 ditutup atas asumsi owner "KEIN aman", 06:55 WIB) | `workspace/engineering/v2-001/` (6 dokumen) |
| V2-002 | Encoder 706 root-cause investigation | P0 | READY_FOR_REVIEW (audit 22 Nov 06:58–07:10; 12 kematian slot 114, RTMP timeout) | `workspace/engineering/v2-002/` (5 dokumen + manifest) |
| V2-003 | Resource Guard v2 audit | P0 | READY_FOR_REVIEW (audit 22 Nov 07:15–07:40; governor 0 aksi efektif, Q6 solved) | `workspace/engineering/v2-003/` (6 dokumen + manifest) |
| V2-004 | Janitor safety audit | P0 | NOT_STARTED (tiket lengkap, diperluas Cycle 5 22 Nov 10:50 WIB; eksekusi menunggu sesi interaktif + SSH) | — |
| V2-005 | Baseline snapshot | P1 | NOT_STARTED (tiket lengkap, diperluas Cycle 6 22 Nov 14:50 WIB; eksekusi menunggu sesi interaktif + SSH) | — |
| V2-006 | Service inventory | P1 | NOT_STARTED (tiket lengkap, diperluas Cycle 7 22 Nov 18:50 WIB; eksekusi menunggu sesi interaktif + SSH) | — |
| V2-007 | Database schema audit (GERBANG TEKNIS) | P1 | NOT_STARTED (tiket lengkap, diperluas Cycle 8 22 Nov 22:50 WIB; eksekusi menunggu sesi interaktif + SSH) | — |
| V2-008 | Resource budget benchmark | P1 | NOT_STARTED (stub) | — |
| V2-009 | VPS 8 price + storage cost verification (GERBANG BIAYA) | P1 | NOT_STARTED (stub) | — |
| V2-010 | Deduplication design proposal | P1 | NOT_STARTED (stub, menunggu V2-007) | — |

**Urutan eksekusi yang disarankan:** V2-001 → V2-002 → V2-003/V2-004/V2-005/
V2-006 paralel (jalur audit teknis + operasional) → V2-007 → V2-008 + V2-009
paralel → V2-010 (setelah V2-007 disetujui).

**END — BATCH 1 EXECUTION TICKETS V2.0 v1.0**
