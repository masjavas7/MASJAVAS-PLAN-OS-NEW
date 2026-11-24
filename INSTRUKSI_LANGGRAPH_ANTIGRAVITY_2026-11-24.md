# INSTRUKSI IMPLEMENTASI: INTEGRASI LANGGRAPH (AGENT AI OTONOM) KE MASJAVAS TUBE OS — PILOT FASE A

> Disusun 24 Nov 2026, 08:08–08:30 WIB. Untuk diteruskan owner ke lane Antigravity.
> Dokumen ini HANYA memerintahkan **Pilot Fase A (read-only, tanpa LLM, Rp 0)**.
> Fase B (LLM) dan Fase C (penulisan metadata) DITAHAN — menunggu keputusan owner.
>
> Dokumen acuan: `KOLEKSI_REPO_AGENT_AI_OTONOM_24_7.md` (riset 30+ repo, 24 Nov),
> `DESAIN_FITUR_AUDIT_REPACKAGING_VIDEO_2026-11-24.md` v1.1 (desain audit),
> `ANALISA_FITUR_AUDIT_INSTRUKSI_ANTIGRAVITY_2026-11-24.md` (perbaikan audit P0–P2,
> disampaikan 07:41 WIB — prasyarat pilot ini).

---

## §1 — KONTEKS

### 1.1 Keputusan: LangGraph

Owner memilih **LangGraph** sebagai framework integrasi agent AI ke aplikasi,
berdasarkan riset 30+ repo GitHub (terverifikasi via GitHub API, 24 Nov 2026):

- Repo: `langchain-ai/langgraph` — ★45.319, Python, MIT, push terakhir 24 Nov 2026.
- Paket: `langgraph` **1.2.24** (PyPI, 20 Nov 2026, wheel 264,6 kB, pure Python)
  + `langgraph-checkpoint-sqlite` **3.1.3** (PyPI, 30 Okt 2026, 41,8 kB).
- Alasan: se-stack penuh (Python 3.12.3 + FastAPI + Pydantic, tanpa runtime baru);
  checkpoint/resume menjawab luka nyata restart 19 Nov (15 tugas hilang karena
  step pemulihan tidak terlindungi); human-in-the-loop native cocok dengan
  doktrin L1–L4 owner; campuran deterministik + agentic sesuai alur MASJAVAS
  (pipeline deterministik + keputusan agentic di titik tertentu).
- Yang BUKAN dipilih untuk inti aplikasi: OpenClaw (sidecar TypeScript + WhatsApp
  — pelengkap nanti, bukan inti), CrewAI (lebih lemah di durability/checkpoint),
  AutoGen (DIARSIPKAN — jangan dipakai).

### 1.2 Keadaan aplikasi saat ini (fakta, 24 Nov 2026)

- Stack: Python 3.12.3, FastAPI 0.162.0, Uvicorn 0.54.0, Pydantic 2.14.3,
  TANPA SQLAlchemy/alembic. Venv: `/opt/masjavas-os/venv/bin/python`.
  WAJIB pakai venv ini — system python3 tidak punya fastapi.
- Service: `masjavas-os-web` (dashboard :8080), `masjavas-os-worker`
  (`worker.autonomous_daemon` — mesin produksi, PALING kritis),
  `masjavas-os-tunnel` (Cloudflare), Caddy (TLS :443). Arsitektur Master + VPS 8
  (live-01). Repo: `masjavas7/MASJAVASTUBE-OS`, HEAD 3ecf76f (07:32 WIB).
- Baseline SOLID (lantai, dari owner 22 Nov): **206 PUBLISHED/hari,
  0 FAILED/BLOCKED.** Setiap perubahan wajib membuktikan baseline tidak regresi.
- Fitur Audit Fase 0 (dibangun Antigravity 07:07–07:32 WIB hari ini):
  `api/routes/audit.py` (194 baris), `core/audit/collector.py` (305),
  `core/audit/classifier.py` (96), `worker/audit_worker.py` (84),
  `web/templates/audit.html` (253). Timer VPS 8: OnCalendar 18:00 UTC = 01:00 WIB
  harian, oneshot, Nice=15, User=ubuntu. Tabel (Master): `video_audit`,
  `audit_channel_summary`, `video_repackage_jobs`, `video_repackage_outcomes`.
  Data live: 2.768 video terbit; 251 diaudit (20 channel, ~07:12 WIB);
  kategori A/B/C/D = 0; rekomendasi PANTAU 20; biaya 15 unit.
- **PRASYARAT MENGIKAT:** perbaikan P0-1 (CTR difabrikasi) dan P0-2 (split-brain
  sync VPS 8 → Master) dari dokumen audit 07:41 HARUS selesai dan terverifikasi
  SEBELUM pilot ini berjalan di data live. Graph di atas classifier yang salah
  hanya meng-checkpoint hasil yang salah. Pilot BOLEH dikembangkan paralel
  (kode + tes), tapi run live pertama menunggu P0-1 + P0-2 hijau.

### 1.3 Biaya — jawaban atas pertanyaan owner (terverifikasi 24 Nov 2026)

**Framework-nya GRATIS.** LangGraph + checkpoint-sqlite open-source MIT —
tanpa lisensi, tanpa langganan, tanpa biaya per run. LangSmith (platform
monitoring LangChain) TIDAK diperlukan untuk pilot ini.

**Satu-satunya komponen berbayar adalah panggilan LLM API** (bayar per token,
hanya saat agent memanggil model). Dan **Pilot Fase A ini memanggil LLM
NOL kali** — semua node deterministik, memakai ulang `core/audit/*` yang sudah
ada. Maka biaya pilot = **Rp 0**. Rinciannya:

| Komponen | Biaya pilot | Keterangan |
|---|---|---|
| LangGraph 1.2.24 + checkpoint-sqlite 3.1.3 | Rp 0 | MIT, `pip install`, tanpa lisensi |
| Panggilan LLM | Rp 0 | TIDAK ADA panggilan LLM di pilot (dilarang §7.4) |
| YouTube API (Analytics + Data) | Rp 0 | Gratis dalam kuota 10.000 unit/hari/vault; pilot ≤ 15 unit/20 channel (baseline run existing) |
| VPS / infra | Rp 0 tambahan | Jalan di VPS 8 yang sudah ada; proses Python ringan, Nice=15 |
| Penyimpanan checkpoint | Rp 0 | File SQLite lokal, kilobyte per channel |

**Biaya NANTI, bila owner menyetujui Fase B (LLM untuk penulisan metadata)** —
estimasi per video (asumsi 2.000 token masuk + 700 token keluar per video;
kurs acuan Rp 18.000/USD; harga dicek dari halaman pricing resmi Google/Anthropic
per 12–19 Nov 2026; hitungan diverifikasi):

| Model | Per video | Batch penuh 2.768 video | 1.000 video/bulan |
|---|---|---|---|
| Gemini 2.5 Flash-Lite | Rp 9 | Rp 23.916 | Rp 8.640 |
| Gemini 3.1 Flash-Lite | Rp 28 | Rp 77.227 | Rp 27.900 |
| Gemini 3.8 Flash (s.d. 31 Des 2026) | Rp 74 | Rp 205.524 | Rp 74.250 |
| Gemini 4 Hydrogen | Rp 148 | Rp 411.048 | Rp 148.500 |
| Claude Haiku 5.5 | Rp 10 | Rp 27.403 | Rp 9.900 |
| Claude Sonnet 5.5 / Gemini 4 Argon (s.d. 30 Nov) | Rp 198 | Rp 548.064 | Rp 198.000 |

Catatan biaya (wajib dibaca sebelum Fase B): (a) Model Flash/Flash-Lite dan
Gemini 4 Hydrogen punya **free tier** (Rp 0 dalam limit rate per proyek di
AI Studio) — Fase B bisa mulai dari free tier sebelum keluar uang; (b) Bila
melewati free tier, prepay minimal $5; (c) **Kenaikan harga terjadwal:**
Gemini 4 Argon/Carbon naik 2× mulai 1 Des 2026 ($4/$20), Gemini 3.8/3.6 Flash
naik 2× mulai 1 Jan 2027 ($1.50/$7.50); (d) Batch API diskon 50%;
(e) Google memakai konten free-tier untuk meningkatkan produknya — metadata
audit bukan data sensitif, tapi dicatat; (f) Rekomendasi: Fase B memakai
Gemini 3.1 Flash-Lite (free tier dulu, fallback berbayar termurah) karena
MASJAVAS sudah memakai Gemini. Keputusan model + anggaran adalah keputusan
owner — BUKAN bagian instruksi ini.

### 1.4 Ruang lingkup instruksi ini

**HANYA Pilot Fase A:** bungkus pipeline audit existing (collector → classifier)
menjadi satu LangGraph StateGraph dengan checkpoint SQLite, retry, governor
kuota, dan titik human-in-the-loop — read-only, tanpa LLM, tanpa aksi tulis ke
YouTube, tanpa perubahan pada worker audit existing (yang tetap jalan seperti
biasa). Pilot dibuktikan pada 5 channel, lalu 20 channel. Selesai = K-1 s.d.
K-7 (§4) hijau + observasi 24 jam.

**BUKAN scope ini:** Fase B (node LLM untuk judul/deskripsi/tag), Fase C
(Applier/edit metadata YouTube), perubahan pada live/publish/ingestion/Stage 2,
OpenClaw/sidecar WhatsApp, LangSmith.

---

## §2 — YANG SUDAH BENAR (terverifikasi, jangan diutak-atik)

1. Pipeline audit existing: `core/audit/collector.py`, `core/audit/classifier.py`,
   `worker/audit_worker.py`, `api/routes/audit.py`, tabel `video_audit` +
   `audit_channel_summary`. Pilot MEMAKAI ULANG modul ini sebagai node graph —
   jangan tulis ulang logika koleksi/klasifikasi.
2. Timer + service audit existing (`masjavas-audit.service` + `.timer`, VPS 8,
   01:00 WIB). Pilot memakai service + timer TERPISAH (§3.6).
3. Skema 4 tabel audit + pencatatan kuota `AUDIT_READ` di `vault_quota_ledger`
   (1 unit/panggilan — benar, jangan diubah).
4. Worker produksi (`worker.autonomous_daemon`), live supervisor, ingestion,
   Stage 2, governor beban, sentinel — JANGAN SENTUH.
5. Baseline 206 PUBLISHED/hari, 0 FAILED/BLOCKED — lantai yang tidak boleh regresi.
6. Disiplin kunci DB: `acquire_db_write_lock` untuk tulis ke DB utama; checkpoint
   graph WAJIB file SQLite terpisah (§3.3) agar tidak menambah perebutan kunci.
7. Aturan restart: maks 1 restart/15 menit lintas agent (jeda dihitung dari
   timestamp terverifikasi, bukan dari angka sebesar epoch — pelajaran 19 Nov).

---

## §3 — ARSITEKTUR PILOT

### 3.1 Graph (StateGraph, satu graph audit)

```
START → load_channel → quota_check → collect → classify → summarize → END
              ↑_______________| (retry, maks 3x, backoff — lihat §3.5)
```

- **State** (TypedDict, eksplisit — jangan state dict bebas):
  `slot: int`, `channel_id: str`, `channel_name: str`,
  `videos: list[dict]`, `audit_rows: list[dict]`,
  `category_counts: dict`, `units_used: int`, `attempt: int`,
  `errors: list[str]`, `started_at: str`, `finished_at: str`.
- **Node `load_channel`**: baca 1 channel dari DB (slot, `youtube_channel_id`,
  daftar `youtube_video_id` dari `live_stream_tasks`/tabel video terbit —
  kolom ini terisi 100%, 2.760/2.760, tanpa discovery).
- **Node `quota_check`** (governor): baca `vault_quota_ledger` untuk vault audit;
  bila proyeksi run ini melewati guard 8.400 unit, graph BERHENTI dengan status
  `DEFERRED_QUOTA` (bukan FAILED) dan checkpoint tersimpan — run berikutnya
  melanjutkan, bukan mengulang. Jangan pernah menghabiskan kuota demi pilot.
- **Node `collect`**: panggil fungsi koleksi existing dari
  `core/audit/collector.py` (TANPA duplikasi logika). Satu panggilan Analytics
  per video; catat `AUDIT_READ` ke ledger seperti kode existing.
- **Node `classify`**: panggil `core/audit/classifier.py` existing (SETELAH
  perbaikan P0-1 — kategori memakai CTR asli + retensi relatif).
- **Node `summarize`**: tulis baris `video_audit` + `audit_channel_summary`
  memakai jalur tulis existing (kunci tulis DB yang sama); hitung
  `units_used` total run.
- **Satu thread per channel**: `thread_id = "audit:<slot>:<YYYY-MM-DD>"`.
  `checkpoint_ns` default. Satu channel = satu thread = resume independen —
  kegagalan 1 channel tidak menghalangi 19 channel lain.

### 3.2 Tanpa LLM di pilot (mengikat)

Tidak ada node LLM, tidak ada import `langchain` model client, tidak ada API
key LLM di config pilot. Bila ada kode yang memanggil LLM, pilot GAGAL kriteria
K-1. Titik sambung Fase B ditandai dengan SATU fungsi stub
`generate_metadata_stub()` yang melempar `NotImplementedError("Fase B ditahan")`
— agar wiring Fase B nanti hanya mengganti badan fungsi ini.

### 3.3 Checkpoint (SQLite, file TERPISAH — mengikat)

- Paket: `langgraph-checkpoint-sqlite` 3.1.3, `AsyncSqliteSaver`
  (worker async; sync `SqliteSaver` hanya untuk tes).
- File: `/opt/masjavas-os/data/langgraph_checkpoints.db` — file BARU, terpisah
  dari DB utama aplikasi. **JANGAN memakai DB utama sebagai checkpoint store.**
  Alasan: DB utama sudah diperebutkan worker/web (pernah `database is locked`);
  checkpoint di file sendiri menghilangkan satu sumber lock + membuat rollback
  pilot semudah menghapus satu file.
- Koneksi checkpoint WAJIB `timeout=30`. Backup file checkpoint masuk ke
  rutinitas backup DB yang sudah ada (file kecil, kilobyte per channel).
- Buktikan checkpoint BEKERJA (bukan hanya terpasang): kriteria K-3
  (kill mid-run → resume) adalah kriteria terpenting pilot ini.

### 3.4 Human-in-the-loop (titik jeda, dibuktikan lewat tes)

- Sisipkan SATU titik `interrupt()` SETELAH node `summarize`, SEBELUM node
  tulis Fase B manapun (node tulis belum ada di pilot — titik ini menjaga
  arsitektur agar Fase B tidak bisa menulis tanpa melewati jeda).
- Di pilot, karena tidak ada node tulis, resume setelah interrupt langsung ke
  END + log `HITL_OK`. Buktikan lewat tes: jalankan graph sampai interrupt →
  verifikasi state tersimpan di checkpoint → resume dengan keputusan
  `{"approved": true}` → graph selesai; dan resume `{"approved": false}` →
  graph berakhir `DECLINED` tanpa efek samping. Keduanya harus hijau (K-4).
- JANGAN membangun UI approval di pilot ini. Approval UI adalah pekerjaan Fase C
  (butuh keputusan owner tentang siapa approver + di mana tampil).

### 3.5 Retry, kegagalan, dan status (mengikat)

- Retry per node: maks 3 percobaan, backoff 5 dtk → 30 dtk → 120 dtk.
  Retry HANYA untuk error transient (timeout/5xx/SSL). Error deterministik
  (auth gagal, video tidak ditemukan, data tidak valid) TIDAK di-retry —
  dicatat di `state["errors"]` dan channel ditandai `NEEDS_REVIEW`.
- Status akhir thread: `DONE`, `DEFERRED_QUOTA`, `NEEDS_REVIEW`, `DECLINED`.
  TIDAK ADA status FAILED massal tanpa sebab (pelajaran FAILED 76 massal
  24 Nov pagi: setiap status non-DONE wajib membawa alasan 1 baris).
- Satu channel gagal TIDAK menggagalkan run: graph lanjut ke channel
  berikutnya; ringkasan run melaporkan DONE x / NEEDS_REVIEW y / DEFERRED z.

### 3.6 Service + timer (terpisah dari audit existing)

- File baru: `config/systemd/masjavas-audit-graph.service` +
  `masjavas-audit-graph.timer`. Oneshot, `User=ubuntu`, `Nice=15`
  (sama seperti audit worker existing — jangan berebut CPU dengan live/publish).
- **Timer TIDAK diaktifkan di pilot.** Pilot dijalankan MANUAL
  (`systemctl start masjavas-audit-graph.service` + argumen
  `--max-channels 5`, lalu `--max-channels 20`). Timer harian hanya diaktifkan
  SETELAH K-1 s.d. K-7 hijau + persetujuan eksplisit owner.
- Entry point baru: `worker/audit_graph_worker.py` (CLI: `--max-channels`,
  `--slots`, `--resume`). JANGAN mengubah `worker/audit_worker.py`.
- Dependensi: tambahkan `langgraph==1.2.24` dan
  `langgraph-checkpoint-sqlite==3.1.3` ke requirements (pin eksak — jangan
  rentang longgar). Instal di venv `/opt/masjavas-os/venv`. Verifikasi
  `pip check` bersih (tanpa konflik dependensi) sebagai bagian K-1.

### 3.7 Observabilitas pilot (minimal, tanpa platform baru)

- Log per run (1 baris JSON per channel): slot, thread_id, status, units_used,
  durasi, percobaan, alasan bila non-DONE. File:
  `/opt/masjavas-os/data/logs/audit_graph.log`.
- JANGAN memasang LangSmith / platform tracing berbayar di pilot. Bila tracing
  dibutuhkan, cukup log JSON + query checkpoint (`checkpointer.list()`).
- Endpoint status opsional (BOLEH, bukan wajib): `GET /api/audit-graph/status`
  (OWNER-only, mengikuti pola auth `/audit` — jangan ulangi celah P1-1 API
  tanpa auth). Bila dibangun, sertakan: run terakhir, DONE/NEEDS_REVIEW/
  DEFERRED, units_used, checkpoint terakhir per thread.

---

## §4 — KRITERIA PENERIMAAN (definisi SELESAI — semua wajib hijau)

| # | Kriteria | Bukti yang diterima |
|---|---|---|
| K-1 | Instalasi bersih, tanpa LLM | `pip check` bersih di venv; `grep -r "ChatOpenAI\|ChatAnthropic\|ChatGoogleGenerativeAI\|langchain_openai\|langchain_anthropic" worker/audit_graph_worker.py core/audit_graph/ ` menghasilkan NOL; `python -c "import langgraph"` sukses di venv |
| K-2 | Hasil IDENTIK dengan worker existing | Pilot 5 channel → bandingkan `video_audit` + `audit_channel_summary` hasil graph vs hasil worker existing untuk 5 channel yang sama: selisih 0 baris + 0 nilai kategori/skor (diff query terlampir §8). Bila berbeda, graph yang salah — jangan "memperbaiki" classifier untuk mencocokkan graph |
| K-3 | Resume setelah restart/kill (KRITERIA UTAMA) | Bunuh proses graph di tengah run 20 channel (kill -9 setelah ≥5 channel DONE) → jalankan ulang `--resume` → run selesai dengan: 0 channel diproses ganda (cek `audited_at` + thread checkpoint), 0 channel hilang (20/20 DONE/NEEDS_REVIEW/DEFERRED), units_used total ≤ baseline run tanpa kill + 10% |
| K-4 | Human-in-the-loop terbukti | Tes otomatis: interrupt → resume approved → selesai; interrupt → resume declined → `DECLINED` tanpa efek samping. Kedua skenario hijau + log `HITL_OK` |
| K-5 | Kuota tidak boros | Satu run 20 channel memakai ≤ 15 unit (baseline run existing 20 channel) + toleransi 0 unit untuk overhead graph. Graph TIDAK BOLEH menambah panggilan YouTube apa pun |
| K-6 | Tidak mengganggu produksi | Selama pilot 20 channel: load VPS 8 naik ≤ 0,5 (load1); 0 stream live terganggu; PUBLISHED hari itu tidak turun dari baseline; 0 FAILED/BLOCKED baru di lane publish |
| K-7 | Laporan + observasi 24 jam | Laporan per fase (§6) lengkap; checkpoint file + log utuh 24 jam setelah run 20 channel; tidak ada error berulang di log; rollback teruji (hapus service + file checkpoint → sistem kembali ke keadaan sebelum pilot, worker existing tetap jalan) |

**Aturan penilaian:** satu kriteria merah = pilot BELUM selesai. Jangan
menyebut "95% selesai" — sebutkan K mana yang hijau dan K mana yang merah,
dengan bukti per K.

---

## §5 — SARAN & IDE TAMBAHAN (di luar scope pilot — jangan dikerjakan sekarang)

1. **Fase B (LLM metadata):** node `generate_metadata` menggantikan stub §3.2 —
   1 panggilan LLM per video kategori A/B (judul + deskripsi + tag koma).
   Model rekomendasi: Gemini 3.1 Flash-Lite (free tier dulu; fallback Rp 28/video
   — lihat tabel §1.3). WAJIB: budget cap harian token + ledger biaya per run +
   persetujuan owner atas model dan anggaran SEBELUM satu panggilan pun.
2. **Fase C (Applier + approval UI):** node tulis YouTube SETELAH interrupt §3.4;
   UI approval di dashboard (OWNER-only); setiap edit diukur dampaknya 7/30 hari
   (desain audit v1.1 §3.5–§3.6). Keputusan owner K-1 s.d. K-7 (desain audit)
   belum dibuat — JANGAN mulai.
3. **Store/memori antar-run:** `langgraph-store` (SQLite) untuk memori jangka
   panjang graph (mis. "channel ini sudah 3x NEEDS_REVIEW karena auth") —
   berguna setelah pilot stabil, bukan sekarang.
4. **Pola yang sama untuk lane lain (NANTI, satu per satu, masing-masing dengan
   pilot + kriteria sendiri):** retry ingestion (job macet NO 482/NO 456),
   retry Stage 2 (PARKED 216), janitor antrean. JANGAN generalisasi prematur —
   satu graph audit yang terbukti dulu.
5. **LangSmith (opsional, free tier):** tracing visual bila log JSON dirasa
   kurang — keputusan owner, bukan kebutuhan pilot.
6. **OpenClaw sidecar (jalur terpisah):** gateway WhatsApp + monitor proaktif
   di samping app — pekerjaan berbeda, tim berbeda, JANGAN dicampur ke pilot ini.

---

## §6 — INSTRUKSI PELAKSANAAN

**Urutan:** Fase 0 (prasyarat) → Fase 1 (instalasi + tes) → Fase 2 (graph +
pilot 5 channel) → Fase 3 (pilot 20 channel + observasi 24 jam).

- **Fase 0 — Prasyarat (TANPA ini, jangan jalankan run live):**
  P0-1 + P0-2 dokumen audit 07:41 selesai + terverifikasi (CTR asli, sync
  VPS 8 → Master jalan). Pengembangan kode pilot BOLEH paralel dengan Fase 0,
  tapi run live pertama menunggu Fase 0 hijau. Bila P0-1/P0-2 belum selesai
  saat kode pilot siap, laporkan "kode siap, menunggu prasyarat" — jangan
  menjalankan pilot di atas data yang diketahui salah.
- **Fase 1 — Instalasi + tes (tanpa run live):** pin dependensi (§3.6),
  `pip check` bersih, tulis graph + stub + tes (K-1, K-4), tes checkpoint di
  temp-DB (BUKAN DB live — disiplin isolasi tes mengikat).
- **Fase 2 — Pilot 5 channel:** run manual `--max-channels 5`, buktikan K-2
  (identik dengan worker existing) + K-5 (kuota). Bila K-2 merah, hentikan
  dan laporkan — jangan lanjut ke 20 channel.
- **Fase 3 — Pilot 20 channel + kill-test:** run manual `--max-channels 20`,
  buktikan K-3 (kill → resume) + K-6 (tidak ganggu produksi), observasi 24 jam
  (K-7). Timer TETAP nonaktif sampai owner menyetujui aktivasi.

**Aturan perubahan (mengikat, dari owner 22 Nov):** aplikasi sudah SOLID —
setiap perubahan wajib rencana verifikasi pra/pasca + rollback + jendela
observasi. Jangan pernah mengorbankan bagian yang solid demi perbaikan.
Baseline (206 PUBLISHED/hari, 0 FAILED/BLOCKED) adalah lantai. V2.0 bersifat
aditif — pilot ini aditif (file + service baru, nol perubahan pada jalur
produksi existing).

**Aturan verifikasi (mengikat):**
- Buktikan perbaikan AKTIF dengan membandingkan waktu commit vs waktu start
  proses (service audit-graph).
- Buktikan hasil PUBLIK/nyata (query DB + log + file checkpoint), bukan hanya
  status di memori.
- Jangan pernah mengklaim "100% SELESAI" tanpa bukti terukur per kriteria §4.
- Setiap logika baru disertai unit test minimal (graph, retry, governor,
  HITL). Tes WAJIB pakai venv `/opt/masjavas-os/venv/bin/python` + temp-DB.

**Catatan teknis:** file-file audit saat ini root-owned —
`sudo chown ubuntu:ubuntu` sebelum mengedit. Jangan commit file yang tidak
terkait. Jangan sentuh kode live/publish/ingestion/Stage 2. `sqlite3.connect`
ke DB live selalu `timeout=30`. Jangan sleep panjang di dalam satu sesi SSH
(putus ~40–60 dtk) — sleep lokal dulu, baru jalankan ssh pendek.

**Format laporan per fase:** apa yang diubah (commit hash), bukti aktivasi
(PID + waktu start), bukti hasil (angka sebelum/sesudah + query verifikasi),
K mana yang hijau/merah, apa yang masih terbuka.

---

## §7 — LARANGAN

1. JANGAN membangun Fase B (node LLM) atau Fase C (Applier/edit metadata) —
   keputusan owner belum dibuat. Pilot ini read-only + tanpa LLM.
2. JANGAN mengubah kode live streaming, publish, ingestion, Stage 2, atau
   worker audit existing (`worker/audit_worker.py`) dalam pilot ini.
3. JANGAN memakai DB utama aplikasi sebagai checkpoint store — file terpisah
   (§3.3) adalah syarat mutlak.
4. JANGAN memanggil LLM apa pun di pilot — biaya pilot Rp 0 adalah kriteria
   (K-1 memeriksa tidak adanya import client LLM).
5. JANGAN memicu audit penuh berulang untuk pengujian — setiap run penuh
   ≈ 500 unit. Uji dengan `--max-channels` kecil (5, lalu 20).
6. JANGAN mengklaim pilot selesai tanpa bukti aktivasi + hasil per kriteria §4.
7. JANGAN restart live/publish worker — hanya service audit-graph (baru, belum
   produksi) yang bebas di-restart; restart web bila perlu, maks 1/15 menit
   lintas agent.
8. JANGAN mencetak stream key, token OAuth, API key LLM, atau URL bertanda
   tangan di log/laporan.
9. JANGAN mengaktifkan timer `masjavas-audit-graph.timer` tanpa persetujuan
   eksplisit owner setelah K-1 s.d. K-7 hijau.
10. JANGAN menghapus data audit yang sudah ada (251 baris) atau checkpoint
    worker existing — histori berguna untuk perbandingan K-2.

---

## §8 — LAMPIRAN: FAKTA KUNCI

- Repo: `masjavas7/MASJAVASTUBE-OS`, HEAD 3ecf76f (07:32:11 WIB, 24 Nov 2026).
- Stack: Python 3.12.3, FastAPI 0.162.0, Uvicorn 0.54.0, Pydantic 2.14.3,
  tanpa SQLAlchemy/alembic. Venv: `/opt/masjavas-os/venv/bin/python`.
- Paket pilot: `langgraph==1.2.24` (PyPI 20 Nov 2026), `langgraph-checkpoint-sqlite==3.1.3` (PyPI 30 Okt 2026). GitHub: `langchain-ai/langgraph` ★45.319, MIT.
- File audit existing: `api/routes/audit.py` (194), `core/audit/collector.py`
  (305), `core/audit/classifier.py` (96), `worker/audit_worker.py` (84),
  `web/templates/audit.html` (253), `config/systemd/masjavas-audit.service` +
  `.timer` (VPS 8, OnCalendar 18:00 UTC = 01:00 WIB, oneshot, Nice=15).
- File BARU pilot: `worker/audit_graph_worker.py`, `core/audit_graph/`
  (graph + state + nodes), `config/systemd/masjavas-audit-graph.service` +
  `.timer` (timer NONAKTIF), `/opt/masjavas-os/data/langgraph_checkpoints.db`,
  `/opt/masjavas-os/data/logs/audit_graph.log`, tes di `tests/test_audit_graph*.py`.
- Tabel: `video_audit`, `audit_channel_summary`, `video_repackage_jobs`,
  `video_repackage_outcomes` (Master); ledger kuota `vault_quota_ledger`
  (`operation_type='AUDIT_READ'`, 1 unit/panggilan).
- Data live: 2.768 video terbit; 251 diaudit (20 channel); A/B/C/D = 0
  (SEBELUM P0-1); PANTAU 20; biaya run 20 channel = 15 unit.
- Query verifikasi K-2 (identik): bandingkan `SELECT slot, category, score FROM
  video_audit WHERE audited_at >= <run_graph> ORDER BY slot;` vs hasil run
  worker existing untuk slot yang sama — selisih harus 0.
- Query verifikasi K-3 (tanpa ganda): `SELECT youtube_video_id, count(*) c FROM
  video_audit WHERE date(audited_at,'unixepoch') = '<tanggal>' GROUP BY 1
  HAVING c > 1;` — harus 0 baris.
- Query verifikasi K-5 (kuota): `SELECT sum(units_consumed) FROM
  vault_quota_ledger WHERE operation_type='AUDIT_READ' AND created_at >=
  <awal_run>;` — run 20 channel ≤ 15 unit.
- Dokumen acuan:
  `~/workspace/engineering/KOLEKSI_REPO_AGENT_AI_OTONOM_24_7.md` (riset repo),
  `~/workspace/engineering/DESAIN_FITUR_AUDIT_REPACKAGING_VIDEO_2026-11-24.md`
  v1.1 (763 baris, desain audit + §13 bottleneck),
  `~/workspace/engineering/ANALISA_FITUR_AUDIT_INSTRUKSI_ANTIGRAVITY_2026-11-24.md`
  (404 baris, perbaikan P0–P2 = prasyarat pilot ini).

---

*Dokumen ini disusun dari riset repo terverifikasi API + pembacaan dokumen
audit + harga API resmi (24 Nov 2026, 08:08–08:30 WIB). Semua klaim dapat
diverifikasi ulang dari file, query, dan tautan yang tercantum.*
