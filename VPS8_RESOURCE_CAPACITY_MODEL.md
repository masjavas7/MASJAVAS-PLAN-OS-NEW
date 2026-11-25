# MODEL KAPASITAS SUMBER DAYA VPS 8 — MASJAVAS TUBE OS V2

**Status:** DRAFT — model perencanaan, BUKAN konfigurasi produksi.
**Versi:** 1.2 — 25 Nov 2026, 22:50 WIB (Cycle 26: + verdict
drift waktu M8 — laju per-stream BERGANTUNG JENDELA WAKTU
(siang ~0,12–0,13; sore/malam ~0,16–0,17); lantai perencanaan
0,175 TETAP; OQ-TCO-5 TERJAWAB, OQ-TCO-6 TERJAWAB SEBAGIAN;
revisi penuh v2.0 tetap menunggu V2-008).
**Versi:** 1.1 — 25 Nov 2026, 18:50 WIB (Cycle 25: + KOREKSI M8 —
angka live/kapasitas §1/§2/§3.1/§7.2/§7.3 SUPERSEDED; otoritas
biaya scale-up kini `MASJAVAS_TCO_MODEL.md` §7).
**Versi:** 1.0 — 22 Nov 2026.
**Ruang lingkup:** box VPS 8 / Live-01 (Contabo Core: 8 vCPU, 24 GB
RAM, 300 GB disk, port 600 Mbit/s). Fokus: CPU — dimensi yang paling
ketat. RAM, disk, dan jaringan dicakup singkat di §2.
**Sumber:** `PLAN_PENAMBAHAN_VPS8_2026-11-22.md` §1–§6 (§4 modal,
§5.2/§5.4 guard), `ADDENDUM_REASSEMBLE_CPU_2026-11-22.md`,
`v2-003/GUARD_RECOMMENDATIONS_V2-003.md`, `STATUS_REGISTRY.yaml`,
`MASJAVAS_COST_REGISTER.md`.
**Item sprint:** Repository Engineering Audit Sprint, P0 #2 (F3).

## KOREKSI M8 — 25 Nov 2026, 14:51 WIB (Cycle 24–25; diagnosis `DIAGNOSIS_M8_2026-11-25.md`)

**Yang gugur (terukur, bukan asumsi):** A2 prebuilt terukur
**0,1746 core/stream** (feeder 0,0548 + encoder 0,1199; 30 interval/
33,5 mnt, delta-ticks /proc), bukan ≤0,07; VPS 8 hanya menghantar
**~4,3–5,0 core/dtk** (vCPU 55–62 ticks/dtk, rasio 0,53–0,62,
steal=0), bukan 8,0 — kapasitas terpakai **3,0–3,5 core/box**
(70% dari yang terhantar), bukan 5,6.

**Bagian model ini yang SUPERSEDED (angka lama dipertahankan di
bawah sebagai riwayat — JANGAN dipakai untuk perencanaan):**
- §1 ringkasan ("≈7 core dari 8 vCPU → headroom ±1 core") —
  total box terukur saat ini (live + supervisor + builder) sudah
  ≈3,9–4,4 core/dtk pada kapasitas terhantar 4,3–5,0; box jenuh
  pada kapasitas YANG TERHANTAR, bukan setengah-menganggur.
- §2 baris CPU ("8 vCPU, puncak ≈7 core, headroom ±1") —
  kapasitas efektif ≈4,3–5,0 core/dtk; headroom harus dihitung
  dari 3,0–3,5 terpakai, bukan dari 8 nominal.
- §3.1 baris Live ("±0,07 core @2 stream → ±1,75 core @50 stream;
  puncak 2 core") — angka benar: 50 stream ≈ **9,3 core all-in**
  (8,75 + supervisor per box), bukan 2,0.
- §3.1 baris TOTAL ("±2,5 core (50 stream + prep penuh)" /
  "≈7 core (dengan cap §5)") — 50 stream + prep penuh TIDAK
  muat di 1 box; total 50 stream saja ≈9,3 core.
- §7.2 ("Live 50 stream 2,0; total ≈7,4 core; headroom ±0,6") —
  total yang benar ≈9,3 core live saja (belum prep); headroom
  pada 1 box = NEGATIF untuk 50 stream.
- §7.3 ("total ≈8,8 core > 8 vCPU" TANPA cap) — ambang
  overcommit harus dibaca terhadap kapasitas TERHANTAR
  (~4,3–5,0), bukan 8 nominal; skenario tanpa cap jauh lebih
  buruk dari yang tertulis.
- **Yang TIDAK gugur:** angka Stage 2 (0,13 core/slot), reassemble
  (0,9 core/job), ingestion (±0), dan struktur cap/admission
  (§5–§6) — ketiganya bukan objek M8 dan tetap berlaku sampai
  V2-008 mengukurnya.

**Otoritas biaya scale-up kini:** `MASJAVAS_TCO_MODEL.md` §7
(Cycle 25): cap = maks ~16–19 stream; 50 = 3 box $71,97;
100 = 6 box $127,92; 696 = 35–41 box $668–$781.
**Tindak lanjut:** revisi penuh model ini (v2.0) menjadi tugas
siklus berikut SETELAH hasil Opsi 1/Opsi 5 + V2-008; sampai
saat itu, §3.1/§7.2/§7.3 di bawah dibaca sebagai RIWAYAT, bukan
perencanaan. Pertanyaan terbuka: OQ-TCO-6 (rasio penghantaran
host jangka panjang — butuh verifikasi independen).

## KOREKSI M8 — LANJUTAN (Cycle 26: verdict drift waktu, 25 Nov ~20:54 WIB)

Sumber: diagnosis selisih M8 sesi koordinator D33 (laporan
~20:10 + soliditas ~20:50, `~/memory/2026-11-25.md`).

**Verdict: selisih +36% (0,120 → 0,1637) = DRIFT WAKTU, bukan
efek konfigurasi.** "A2 = 0,120" = mean jendela pagi/siang;
"Tuas 1 = 0,1637" = mean jendela sore/malam. Laju instan 088
sudah 0,1459 (15:42) dan 0,1447 (17:54) — identik; kenaikan
0,145 → ~0,17 terjadi 18:00–19:30 tanpa perubahan konfigurasi,
lalu turun ke 0,146 (19:55–20:57). H1 umur proses TERTOLAK;
H2 RTMP/TCP dominan TERKONFIRMASI (mux0:flv 62% CPU) tapi
penyebab kenaikan TAK TERBUKTI; H3 fase konten TERTOLAK
definitif; H4 host/penjadwalan berubah menurut waktu
TERKONFIRMASI sebagai penjelasan terbaik (semua encoder +
supervisor ikut naik; korelasi M8–load −0,399).

**Yang berubah di model ini:**

- Laju per-stream adalah FUNGSI JENDELA WAKTU: pagi/siang
  ≈ 0,12–0,13; sore/malam ≈ 0,16–0,17. Angka "0,1746" (A2)
  dan "0,1637" (Opsi 1, Tuas 1) keduanya adalah mean jendela
  sore/malam — JANGAN dibandingkan dengan "0,120" (mean
  pagi/siang) sebagai bukti efek konfigurasi.
- **Lantai perencanaan 0,175 core/stream TETAP** — otoritas
  biaya tetap TCO §7 (v1.2 §7.7). Opsi 1 hemat feeder ~0,055
  kotor, bersih ~0,011 dalam jendela yang sama.
- OQ-TCO-5 **TERJAWAB** (lantai tidak berubah); OQ-TCO-6
  **TERJAWAB SEBAGIAN** (kerja terhantar naik 4,38 → 4,66
  core/dtk; efisiensi per wakeup yang berubah, bukan kapasitas;
  verifikasi independen rasio host tetap menunggu).
- **D84 (owner ~20:54): TETAP 2 BOX** sampai solid 100% —
  opsi 3 box DITUNDA; cap $34,67 TETAP mengikat.
- Revisi penuh v2.0 TETAP menunggu V2-008 (benchmark belum
  jalan — butuh SSH/sesi interaktif).

## Legenda label klaim

- **[VF]** VERIFIED FACT — terverifikasi langsung (log/DB/pengukuran).
- **[DF]** DOCUMENTED FACT NOT YET REVALIDATED — tercatat di dokumen
  sumber, belum divalidasi ulang sejak ditulis.
- **[AS]** ASSUMPTION — asumsi yang dipakai model ini.
- **[PR]** PROPOSAL — usulan model ini, belum diputuskan.
- **[OQ]** OPEN QUESTION — pertanyaan terbuka, belum ada jawaban.
- **[OD]** OWNER DECISION REQUIRED — butuh keputusan owner.

**Aturan yang mengikat (sprint aturan 6):** angka concurrency final,
reserve, dan threshold adalah PROPOSAL sampai V2-008 (benchmark
anggaran sumber daya) mengukurnya. Model ini TIDAK mengunci angka
apa pun.

---

## 1. Ringkasan eksekutif

VPS 8 dibeli untuk mengurangi perebutan sumber daya — tetapi sejak
awal ia beroperasi dekat kapasitas maksimum CPU:

- Total puncak yang dihitung ulang (termasuk koreksi reassemble):
  **≈7 core dari 8 vCPU → headroom ±1 core.** [VF — addendum 12:10]
- Headroom ±1 core itu **BERSYARAT**: ia berlaku hanya bila batas
  konkurensi (§5) dan admission control (§6) ditegakkan. Tanpa
  keduanya, beban puncak dapat mencapai **≈8–9 core — melebihi
  kapasitas box** (simulasi §7.3).
- CPU adalah dimensi yang ketat. RAM (±6 GB dari 24 GB), disk
  (±155 GB dari 300 GB), dan jaringan (310 + ≤100 Mbps dari
  600 Mbps) punya headroom besar. [DF — PLAN §4]
- Kesimpulan model: **VPS 8 muat, tetapi tanpa slack.** Batas
  konkurensi dan admission control adalah bagian LOAD-BEARING dari
  desain — bukan pelengkap. Bila V2-008 mengukur puncak nyata lebih
  tinggi dari model ini, jawabannya adalah menurunkan konkurensi
  (atau menunda Fase 1.6), bukan menambah beban dengan harapan
  "nanti juga cukup".

---

## 2. Spesifikasi box + dimensi non-CPU

| Dimensi | Kapasitas | Pemakaian puncak (model) | Headroom | Status |
|---|---|---|---|---|
| CPU | 8 vCPU | ≈7 core | ±1 core | **KETAT** |
| RAM | 24 GB | ±6 GB | ±18 GB | Longgar [DF — PLAN §4] |
| Disk | 300 GB | ±155 GB | ±145 GB (kuota: sisakan ≥50 GB bebas) | Longgar [DF — PLAN §4/§5.4] |
| Jaringan | 600 Mbit/s | 310 Mbps live + ≤100 Mbps prep = ±410 Mbps | ±190 Mbps (di bawah cap admission 420 Mbps) | Cukup, dengan throttle [DF — PLAN §4] |

RAM per workload [DF — PLAN §4]: Stage 2 4 GB; ingestion gate
≥4 GB tersedia; live 1 GB; DR <1 GB; reassemble ±0,1 GB/job.
Disk [DF — PLAN §4]: Stage 2 50 GB; ingestion 100 GB (staging +
salinan pool 20,5 GB); live 5 GB; DR 100 GB.
Jaringan [VF — PLAN §1.2/§4]: 50 stream @1080p butuh ±310 Mbps
egress (±6,2 Mbps/stream); cap admission 70% port = 420 Mbps;
unggahan prep di-throttle ≤100 Mbps dan dijeda saat TX live >50%.

**Implikasi:** model ini memfokuskan seluruh analisis pada CPU.
Bila V2-008 menemukan dimensi lain menjadi ketat lebih dulu (mis.
I/O disk saat Stage 2 + DR sync bersamaan), model ini harus
direvisi — catat sebagai [OQ-201].

---

## 3. Budget CPU per workload

### 3.1 Tabel budget

> **SUPERSEDED — KOREKSI M8 (Cycle 25).** Baris Live + TOTAL di
> bawah ini riwayat (v1.0); angka perencanaan kini di TCO §7.
> Baris Stage 2 / reassemble / ingestion tetap berlaku sampai
> V2-008.

| Workload | Rata-rata | Puncak | Basis angka | Label |
|---|---|---|---|---|
| Live (s.d. 50 stream) | ±0,07 core @2 stream → ±1,75 core @50 stream | 2 core (rentang 1–3) | 0,07 core/2 stream terukur; 50 × 0,035 ≈ 1,75 | [VF] rata-rata; [DF] rentang PLAN §4 |
| LoopGen Stage 2 | ±0,13 core/slot (12,8 dtk CPU / 100,1 dtk wall) | 4 core (cgroup cap, 4 slot) | 12,8 dtk CPU vs 100,1 dtk wall/klip; cgroup max 4 core | [VF] rasio CPU/wall; [DF] cgroup PLAN §4 |
| Reassemble audio | 0,25 core (64×/24 jam × 6 mnt × 0,9 core) | ±1 core @1 job; ±1,8 core @2 job | ±0,9 core/job, ±6 mnt/job, 64×/24 jam | [VF] addendum 12:10 |
| Ingestion (non-reassemble) | ±0 | ±0 | 0,0% di 1.510 tick (I/O-bound: unduh yt-dlp + sync Drive) | [VF] bacaan governor; lihat catatan §3.2 |
| OS + daemon box (supervisor live, prep worker, governor lokal, watchdog, DR sync) | [PR] ±0,3 core | [PR] ±0,5 core | Analogi: daemon Master ±0,2 core terukur; supervisor + watchdog belum terukur di VPS 8 | [PR] — V2-008 harus mengukur |
| Web/API + control plane | — | — | TETAP DI MASTER (bukan beban VPS 8). Web 1,02 core hari ini → <0,2 core setelah Fase 0 (P15) | [VF] web 1,02 core; [DF] target PLAN §2 |
| **TOTAL VPS 8** | **±2,5 core** (50 stream + prep penuh) | **≈7 core** (dengan cap §5) | Jumlah puncak di atas, reassemble 1 job | [VF] total PLAN §4 pasca-koreksi |

### 3.2 Catatan per workload

**Live.** CPU live hampir nol per stream: 0,07 core untuk 2 stream
(±2–6% CPU per stream) [VF — PLAN §1.2]. Bottleneck live adalah
jaringan, bukan CPU. Puncak 2 core pada 50 stream mencakup supervisor
+ relay feeder audio per stream + overhead; rentang 1–3 core dari
PLAN §4 mencerminkan ketidakpastian jumlah feeder aktif. [DF]

**Stage 2.** I/O dominan, bukan CPU: 12,8 dtk CPU vs 100,1 dtk wall
per klip (maks wall 900,8 dtk) [VF — PLAN §1.4]. Artinya satu slot
Stage 2 rata-rata hanya memakai ±13% satu core — tetapi saat ffmpeg
aktif ia dapat memakan 1 core penuh (burst). Cgroup 4 core membatasi
seluruh Stage 2, bukan per slot. Konsekuensi: **yang membatasi Stage 2
adalah jumlah SLOT + I/O Drive, bukan CPU mentah** — menaikkan slot
di atas 4 tidak menambah throughput bila I/O jenuh, tetapi tetap
menambah puncak CPU. Throughput terukur: 1.011 DONE/7 hari; antrean
50 QUEUED. [VF — PLAN §1.4]

**Reassemble.** ±0,9 core/job selama ±6 menit; 64 job/24 jam →
±5,8 core-jam/hari ≈ 0,25 core rata-rata; puncak terukur ±1 core
(job berurutan, 3 job/35 mnt) [VF — addendum]. Reassemble adalah
satu-satunya beban prep yang CPU-bound secara nyata. **Koreksi yang
melahirkan model ini:** angka ini sempat tidak masuk anggaran karena
CPU-nya tercatat di bucket render/web (artefak atribusi V2-003).

**Ingestion (non-reassemble).** ±0 CPU terukur di 1.510 tick [VF].
Catatan: angka ini adalah bacaan governor dengan atribusi yang sama
yang menyesatkan soal reassemble — ingestion berjalan di dalam
`autonomous_daemon` (cmdline tanpa "ingestion") sehingga CPU-nya
tercatat lane `other` [VF — V2-003]. Yang dapat dinyatakan dengan
aman: ingestion didominasi unduh + sync (I/O-bound) dan gate-lah
yang membatasinya (5 cap bertumpuk → konkurensi 1), bukan CPU.
V2-008 + V2-003-R3 (perbaiki atribusi) harus mengonfirmasi angka ±0
ini; sampai saat itu, model memakai ±0 dengan cadangan OS/daemon
§3.1 sebagai penyerap ketidakpastian. [OQ-202]

**OS + daemon box.** Belum pernah diukur di VPS 8 (box belum ada).
Angka [PR] ±0,3/±0,5 core memakai analogi Master: daemon otonom
±0,2 core terukur [VF — PLAN §1.5], ditambah supervisor live +
governor lokal + watchdog + DR sync malam. DR sync (rsync) dijadwalkan
malam hari agar tidak bertabrakan dengan puncak prep [DF — PLAN §5.2].

**Yang TIDAK dihitung di VPS 8.** Web/API (1,02 core hari ini) dan
control plane (dispatcher + governor Master, <0,5 core) tetap di
Master [DF — PLAN §4/§5.1]. LoopGen Tahap 1/2 API-bound (±0 CPU
lokal) [DF — PLAN §4]. Jangan menggandakan angka-angka ini ke
anggaran VPS 8.

---

## 4. Reserve

Reserve adalah kapasitas yang TIDAK BOLEH dipakai workload lain —
bukan target, melainkan lantai yang dilindungi admission control.

### 4.1 Reserve Live: 2 core [PR, didukung setting yang ada]

- **Nilai yang diusulkan: 2 core** — sama dengan puncak live yang
  dimodelkan (§3.1) dan sama dengan `LIVE_STREAM_RESERVED_CORES=2`
  yang sudah terdefinisi di settings aplikasi [DF — V2-003 OQ-011].
  (Pemakaian setting itu di jalur aktif belum diaudit — OQ-011
  masih terbuka; reserve di sini adalah kebijakan model, bukan
  klaim bahwa setting itu sudah bekerja.)
- **Makna:** admission prep (reassemble/Stage 2/ingestion berat)
  DITOLAK bila CPU yang tersedia setelah reserve < kebutuhan job.
  Live tidak pernah antre di belakang prep.
- **Landasan prinsip:** "Live prioritas tertinggi di box-nya;
  co-tenant tidak pernah boleh mengganggu live; bila mengganggu,
  co-tenant dimatikan dulu" [DF — PLAN §3 prinsip 5].
- **Catatan:** pada 2 stream (hari ini), live hanya memakai 0,07
  core — reserve 2 core pada tahap awal adalah perlindungan untuk
  pertumbuhan ke 50 stream, bukan kebutuhan hari ini. V2-008
  mengukur ulang reserve ini pada 16, 30, dan 50 stream.

### 4.2 Reserve OS + daemon: 0,5 core [PR]

- **Nilai yang diusulkan: 0,5 core** (puncak §3.1) — untuk kernel,
  systemd, Tailscale, supervisor live, governor lokal, watchdog,
  dan DR sync. Tanpa reserve ini, satu lonjakan prep dapat
  membuat governor/watchdog sendiri kelaparan CPU dan gagal
  mendeteksi overload yang ia sebabkan (mode kegagalan yang
  sama dengan governor Master hari ini: 13 error loop tanpa
  eskalasi [VF — V2-003]).
- **Mekanisme yang diusulkan:** cgroup terpisah untuk
  supervisor/governor/watchdog dengan `cpu.weight` tinggi atau
  `cpu.min` — [PR], detail implementasi di tangan V2-008 +
  implementasi Fase 1.

### 4.3 Anggaran yang tersisa untuk prep

8 vCPU − 2 (live) − 0,5 (OS/daemon) = **5,5 core untuk prep
(Stage 2 + reassemble + ingestion)**. Puncak prep yang dimodelkan:
Stage 2 4 (cgroup) + reassemble 1 (1 job) + ingestion 0 = 5 core
→ muat dengan sisa 0,5 core. **Bila reassemble berjalan 2 job
bersamaan (1,8 core), puncak prep = 5,8 core > 5,5 core anggaran**
→ admission control §6 HARUS mencegah kombinasi ini saat live
penuh, atau menurunkan slot Stage 2 sementara. Inilah inti model:
angka-angka individual muat, kombinasinya belum tentu.

---

## 5. Batas konkurensi (PROPOSAL — V2-008 mengunci)

| Workload | Batas yang diusulkan | Rasional | Validasi V2-008 |
|---|---|---|---|
| Reassemble | **Maks 2 job bersamaan**; 1 job saat live ≥30 stream atau CPU pressure tinggi | 64 job × 6 mnt = 6,4 jam CPU/hari. 1 slot: antrean terburuk 6,4 jam bila 64 job datang bersamaan; 2 slot: 3,2 jam. 2 job = 1,8 core puncak — masih muat dalam anggaran prep 5,5 core bila Stage 2 ≤3 slot aktif | Ukur wall time/job pada 1 vs 2 job bersamaan (apakah 2 job saling memperlambat?); ukur puncak CPU nyata 2 job |
| Stage 2 | **Canary 2 slot** (Fase 1.5) → **penuh 4 slot** (Fase 1.6); cgroup max 4 core tetap | Sesuai PLAN §6. 4 slot × ±0,13 core rata-rata = ±0,5 core rata-rata; puncak dibatasi cgroup 4 core | Ukur throughput/klip pada 2 vs 4 slot (apakah I/O jenuh sebelum slot penuh?); ukur puncak CPU cgroup |
| Ingestion (non-reassemble) | **Gate-bound, bukan CPU-bound** — batas dari 5 cap yang ada (shed/operasional/live/RAM/load) + reformasi V2-003-R7 | CPU ±0; yang membatasi adalah gate + bot-check + sweeper | V2-003-R7 mereformasi cap sebagai satu sistem; V2-008 mengukur throughput vs konkurensi |
| Live | **Admission TX** (bukan batas CPU): tolak stream baru bila TX total + bitrate > 70% port (420 Mbps) | CPU live tidak pernah menjadi kendala [VF — PLAN §1.2] | Uji admission pada 16/30/50 stream |

**Aturan kombinasi [PR]:** Stage 2 slot aktif + reassemble job
bersamaan tidak boleh melampaui anggaran prep 5,5 core pada
perhitungan puncak (4 + 1,8 = 5,8 > 5,5). Implementasi admission:
saat reassemble ke-2 diminta dan Stage 2 sudah 4 slot aktif, tunda
reassemble ke-2 sampai satu slot Stage 2 bebas (atau sebaliknya —
tunda klaim Stage 2 baru saat 2 reassemble berjalan). Urutan
penundaan adalah kebijakan admission (§6), bukan preferensi
workload: yang ditunda adalah job yang PALING BARU diminta.

---

## 6. Admission control berbasis CPU pressure + load

### 6.1 Prinsip

1. **Guard lokal per box** [DF — PLAN §5.4]: governor VPS 8 hanya
   menjaga VPS 8. Tidak ada shed antar-box (doktrin tanpa sandera).
2. **Live tidak pernah ditolak karena prep** — admission prep yang
   mengalah, bukan live [DF — PLAN §3 prinsip 5].
3. **Tolak/tunda SEBELUM live terganggu** — admission adalah
   pencegahan, bukan pemadam. Shed reaktif (mematikan job yang
   sedang jalan) adalah lapis terakhir, bukan lapis pertama.
4. **Setiap penolakan tercatat alasannya** [DF — PLAN §6 langkah
   0.4: "setiap penahanan tercatat"] — agar admission tidak menjadi
   kotak hitam seperti governor Master hari ini (29 aksi no-op
   tanpa eskalasi [VF — V2-003]).

### 6.2 Sinyal admission

| Sinyal | Sumber | Yang diukur | Status |
|---|---|---|---|
| Load average (load1/load5/load15) | `/proc/loadavg` | Saturasi CPU box (8 vCPU → saturasi ≈ 8,0) | [DF] load gate sudah ada (load1 ≥2,0/≥3,0 di Master 4-core — NILAI INI TIDAK BERLAKU MENTAH DI VPS 8, lihat §6.3) |
| CPU pressure (PSI) | `/proc/pressure/cpu` (`some`/`full`, avg10/60/300) | Waktu tugas menunggu CPU — ukuran antrean yang lebih jujur daripada load | [OQ-203] belum pernah diukur di Master maupun VPS 8; V2-008 harus menetapkan baseline PSI |
| CPU per cgroup | cgroup Stage 2 / prep worker (`cpu.stat`, `cpu.max`) | Pemakaian aktual vs cap per workload | [PR] instrumen baru — butuh implementasi Fase 1 |
| TX live | sampler governor (EMA 300 dtk) | Egress live vs cap 420 Mbps | [VF] sampler sudah ada; V2-003 membuktikan EMA lebih jujur daripada instan 5-detik |
| Antrean job | API Master (operational_tasks) | Backlog prep yang menunggu | [DF] sudah ada |

### 6.3 Threshold (PROPOSAL — V2-008 mengunci)

| Threshold | Nilai yang diusulkan | Tindakan | Rasional |
|---|---|---|---|
| T1 — prep pause (lunak) | load15 ≥ 6,0 ATAU cpu.some avg10 > 20% (bila PSI tersedia) | Jeda klaim prep BARU (reassemble + Stage 2); job berjalan tidak disentuh | load15 6,0 pada 8 vCPU = 75% saturasi; menyisakan ruang untuk live + burst. Nilai load Master (≥2,0/≥3,0 pada 4 core ≈ 50–75%) diskalakan ke 8 vCPU |
| T2 — prep shed (keras) | load1 ≥ 7,0 ATAU cpu.some avg10 > 40% | Turunkan slot Stage 2 aktif (4→2) + tunda reassemble baru; bila masih trip 5 menit, hentikan 1 job prep termuda | 7,0/8 = 87,5% saturasi — live reserve 2 core terancam; shed prep SEBELUM live terganggu |
| T3 — live protect (darurat) | load1 ≥ 7,5 selama 2 menit ATAU live TX drop bersamaan dengan load tinggi | Matikan co-tenant prep (semua job prep non-kritis) sampai load < 6,0 | Implementasi prinsip 5 PLAN §3; darurat, harus tercatat + alert |
| TX live | TX total + bitrate stream baru > 420 Mbps (70% port) | Tolak stream baru | [DF] sudah ada di PLAN §5.2 |

**Catatan penting soal load gate lama:** ambang load1 ≥2,0/≥3,0 di
PLAN §5.4 dikalibrasi untuk Master 4-core. Menyalin angka itu
mentah ke VPS 8 (8 vCPU) akan mencekik prep pada 25–37% saturasi
— kesalahan kalibrasi yang sama jenisnya dengan anggaran TX 115
Mbps yang kedaluwarsa [VF — V2-003]. Threshold di atas diskalakan
ke 8 vCPU; V2-008 harus memvalidasi skala ini dengan beban nyata.

### 6.4 Kebijakan tolak/tunda (urutan)

Saat admission trip, urutan tindakan (dari yang paling murah):

1. **Tunda klaim baru** (prep tidak mengambil job baru; job berjalan
   selesai normal). Tidak ada pekerjaan yang hilang.
2. **Turunkan konkurensi** (slot Stage 2 4→2; reassemble 2→1).
3. **Jeda unggahan Drive prep** (TX throttle — sudah ada di PLAN
   §5.2: jeda saat TX live >50%).
4. **Hentikan job prep termuda** (shed — lapis terakhir; job kembali
   ke antrean dengan lease dilepas, bukan hilang).
5. **Tidak pernah:** mematikan/menunda live untuk memberi ruang
   kepada prep.

Setiap tindakan dicatat (alasan + sinyal + nilai saat trip) dan
dihitung (counter per hari) — agar V2-008 dan operator dapat
melihat apakah admission bekerja atau hanya menjadi pajangan
seperti governor Master hari ini.

---

## 7. Simulasi beban

Semua simulasi memakai angka §3. "Puncak" = jumlah puncak tiap
workload (konservatif — puncak jarang berbarengan persis; V2-008
mengukur korelasi sebenarnya).

### 7.1 Beban normal (hari ini: 2 stream, prep canary)

| Workload | CPU |
|---|---|
| Live 2 stream | 0,07 |
| Stage 2, 2 slot (rata-rata) | 0,26 |
| Reassemble 1 job | 0,9 |
| Ingestion | 0 |
| OS + daemon | 0,3 |
| **Total** | **±1,5 core** |

Headroom ±6,5 core. Box nyaris idle — ini kondisi Fase 1.3–1.5
(canary). Tidak ada risiko kapasitas pada tahap ini.

### 7.2 Beban puncak yang direncanakan (50 stream, prep penuh, cap ditegakkan)

> **SUPERSEDED — KOREKSI M8 (Cycle 25).** Tabel ini riwayat
> (v1.0): live 50 stream yang benar ≈9,3 core all-in; 50 stream
> + prep penuh TIDAK muat di 1 box. Lihat TCO §7.

| Workload | CPU |
|---|---|
| Live 50 stream | 2,0 |
| Stage 2, 4 slot (cgroup cap) | 4,0 |
| Reassemble 1 job | 0,9 |
| Ingestion | 0 |
| OS + daemon | 0,5 |
| **Total** | **≈7,4 core** |

Headroom ±0,6 core — LEBIH SEMPIT dari ±1 core yang dinyatakan
PLAN §4, karena model ini menambahkan reserve OS/daemon 0,5 core
yang tidak ada di penjumlahan PLAN. [PR — perbedaan ini harus
dikonfirmasi V2-008: bila OS/daemon nyata <0,5 core, headroom
kembali ke ±1 core.] Pesan model tidak berubah: **muat, tanpa
slack.**

### 7.3 Beban puncak TANPA cap (yang dicegah admission)

| Workload | CPU |
|---|---|
| Live 50 stream | 2,0 |
| Stage 2, 4 slot (cgroup cap) | 4,0 |
| Reassemble 2 job bersamaan | 1,8 |
| Ingestion (andaikan atribusi salah 0,5 core — skenario stres) | 0,5 |
| OS + daemon | 0,5 |
| **Total** | **≈8,8 core > 8 vCPU** |

**OVERCOMMIT.** Inilah skenario yang dibeli VPS 8 untuk dihindari
— dan yang akan terjadi bila batas konkurensi + admission tidak
ditegakkan. Perhatikan: skenario ini tidak membutuhkan workload
baru atau lonjakan tak terduga; ia hanya membutuhkan dua job
reassemble bersamaan + atribusi ingestion yang ternyata tidak nol.
Admission §6 harus mencegah kombinasi ini (T1 trip pada load15
≥6,0 jauh sebelum 8,8 tercapai).

### 7.4 Mode kegagalan (failure)

| Skenario | Dampak CPU | Respons model |
|---|---|---|
| Restart supervisor live (deploy/restart) | Live 0 sementara; prep tidak berubah | Admission prep tetap jalan; live kembali dalam reserve 2 core |
| Tailscale putus | Prep tidak bisa klaim job baru (CPU prep → 0); live jalan terus (RTMP langsung) | CPU turun — bukan risiko kapasitas; risiko adalah penugasan baru tertunda + heartbeat stale >90 dtk [DF — PLAN §8] |
| Stage 2 runaway (ffmpeg macet/loop) | Dibatasi cgroup 4 core | Cgroup adalah pagar terakhir; watchdog + R6 (alert) harus mendeteksi job yang melebihi wall normal (maks terukur 900,8 dtk/klip [VF]) |
| DR sync malam bertabrakan dengan prep | rsync I/O-bound; CPU kecil | Jadwal malam hari [DF — PLAN §5.2]; bila bertabrakan, DR yang mengalah (non-kritis) |
| Burst reassemble (64 job datang bersamaan) | 2 job × 0,9 = 1,8 core selama ±3,2 jam | Cap 2 job (§5) + T1/T2; antrean 62 job sisanya menunggu — keterlambatan bahan, bukan gangguan live |
| Encoder 114 mati berulang (pola V2-002) | Restart encoder ≈0 CPU | Bukan masalah kapasitas — masalah reliabilitas (P16/R3–R8). Jangan "memperbaiki" dengan menambah core |

---

## 8. Yang harus diukur V2-008 (checklist benchmark)

V2-008 adalah prasyarat mengunci SEMUA angka [PR] di model ini:

1. **Puncak CPU nyata per workload** — ukur live pada 2/16/30/50
   stream; Stage 2 pada 2/4 slot; reassemble pada 1/2 job bersamaan.
   Bandingkan dengan §3; revisi model bila selisih >20%.
2. **Korelasi puncak** — apakah puncak Stage 2 + reassemble
   berbarengan atau saling menghindar? (Menentukan apakah
   penjumlahan §7.2 terlalu konservatif.)
3. **Wall time reassemble 1 vs 2 job** — apakah 2 job bersamaan
   saling memperlambat (kontensi CPU) atau linear?
4. **Throughput Stage 2 vs slot** — 2 vs 4 slot: apakah I/O jenuh
   sebelum slot penuh? (Menentukan apakah slot ke-3/ke-4
   bermanfaat.)
5. **Baseline PSI** (`/proc/pressure/cpu`) pada beban normal/peak —
   menetapkan threshold T1/T2 yang terukur, bukan tebakan.
6. **OS + daemon nyata** di VPS 8 (supervisor + governor +
   watchdog + DR sync) — menggantikan [PR] ±0,3/±0,5 core.
7. **Atribusi ingestion nyata** (bersama V2-003-R3) — mengonfirmasi
   atau mengoreksi ±0 core [OQ-202].
8. **Uji gabungan wajib** — canary Stage 2 (2 slot) diuji BERSAMAAN
   dengan reassemble berjalan, bukan sendirian [VF — addendum §5].
9. **Uji admission** — picu T1/T2 dengan beban buatan; verifikasi
   penolakan tercatat + live tidak terganggu + pemulihan otomatis
   saat beban turun.

**Kriteria lolos V2-008 untuk model ini:** semua angka [PR] diganti
angka terukur, atau dinyatakan tetap [PR] dengan alasan + tanggal
pengukuran ulang. Model tanpa pengukuran ulang = model yang
kedaluwarsa — jangan dipakai untuk keputusan Fase 1.6.

---

## 9. Pertanyaan terbuka

| ID | Pertanyaan | Blokir |
|---|---|---|
| OQ-201 | Apakah I/O disk menjadi ketat lebih dulu saat Stage 2 + DR sync + ingestion staging bersamaan? (Model ini mengasumsikan CPU yang ketat.) | Validasi §2 |
| OQ-202 | Berapa CPU ingestion sebenarnya setelah atribusi diperbaiki (V2-003-R3)? Apakah ±0 terkonfirmasi? | Budget §3.1 |
| OQ-203 | Berapa baseline PSI CPU (`/proc/pressure/cpu`) di Master dan VPS 8? Threshold T1/T2 yang diusulkan belum punya baseline. | Threshold §6.3 |
| OQ-204 | Apakah `LIVE_STREAM_RESERVED_CORES=2` dipakai di jalur aktif mana pun? (V2-003 OQ-011.) Bila ya, reserve §4.1 sudah setengah terimplementasi. | Reserve §4.1 |
| OQ-205 | Berapa wall time reassemble pada 2 job bersamaan — linear atau melambat? | Cap §5 |
| OQ-206 | Apakah cgroup Stage 2 (4 core) juga membatasi anak proses ffmpeg + QA + unggah, atau hanya ffmpeg? | Puncak §3.1 |

---

## 10. Kaitan dengan dokumen lain

- `PLAN_PENAMBAHAN_VPS8_2026-11-22.md` §4 (modal — sumber angka
  model ini), §5.2/§5.4 (arsitektur + guard lokal), §6 (phasing:
  canary 1.5 → penuh 1.6).
- `ADDENDUM_REASSEMBLE_CPU_2026-11-22.md` — koreksi yang melahirkan
  model ini (F3).
- `v2-003/GUARD_RECOMMENDATIONS_V2-003.md` — R1 (admission TX),
  R3 (atribusi), R4 (eksekutor throttle), R7 (reformasi konkurensi
  ingestion); OQ-010/011/012.
- `STATUS_REGISTRY.yaml` — V2-008 NOT_STARTED (prasyarat angka
  final); D9/D10 AWAITING_OWNER (perbaikan Guard v2).
- `VPS8_PURCHASE_GATE.md` — gerbang pembelian; model ini memberi
  masukan "biaya kapasitas" (headroom ±1 core adalah risiko yang
  harus dicatat di keputusan pembelian, bukan alasan menunda
  pembelian bila sinyal lain mendukung).
- `VPS8_DEPLOYMENT_RUNBOOK.md` + `VPS8_ACCEPTANCE_TEST_PLAN.md` —
  langkah 1.5/1.6 + uji gabungan T1.5-C memakai model ini.
