# INSTRUKSI IMPLEMENTASI: AGENT AI SUMOPOD + FITUR MANAJEMEN AGENT — FASE B

> Disusun 24 Nov 2026, 09:30–09:45 WIB. Untuk diteruskan owner ke lane Antigravity.
> Dokumen ini memerintahkan **Fase B1 (agent metadata) + B2 (agent klasifikasi
> jev) + B3 (fitur manajemen agent di dashboard)** — memakai model Sumopod yang
> SUDAH terverifikasi live hari ini (tes 10 panggilan + paket probe, 09:04–09:21 WIB).
> **Fase C (Applier — menulis metadata ke YouTube) TETAP DITAHAN** — agent fase ini
> hanya membuat DRAFT, tidak pernah menulis ke YouTube.
>
> Dokumen acuan: `INSTRUKSI_LANGGRAPH_ANTIGRAVITY_2026-11-24.md` (pilot Fase A —
> SUDAH SELESAI, commit `a1b9ab9`), `FUNGSI_MODEL_SUMOPOD_2026-11-24.md` (riset
> 60 model + rekomendasi), `hasil_tes_sumopod_20261124-090416.md` (tes 10 panggilan),
> `probe_hasil_20261124-091804.md` (paket probe + tindak lanjut).

---

## §1 — KONTEKS

### 1.1 Keputusan model (TERVERIFIKASI live 24 Nov 2026 — bukan estimasi)

Semua angka di bawah ini dari panggilan live nyata (bukan harga stiker).
Harga stiker MENIPU: glm-5.3-flash terlihat termurah tapi efektif Rp 21/post
karena reasoning liar (2.023–7.187 token). Model yang dipakai di sini dipilih
karena biaya EFEKTIF-nya terendah + perilakunya terverifikasi.

| Agent | Model Sumopod | Harga/1 jt token (in/out) | Toggle terverifikasi | Biaya/post | Batch 2.768 post |
|---|---|---|---|---|---|
| **metadata** (utama) | gpt-4o-mini | $0,15 / $0,60 | — (non-reasoning bawaan) | Rp 3,5 | ±Rp 9.600 |
| **metadata** (fallback murah) | gpt-6-luna | $0,10 / $0,50 | `reasoning_effort:"none"` → reasoning 0 | Rp 2,2 | ±Rp 6.000 |
| **klasifikasi** | jev-1.14.0 | $0,042 / $0,00 | endpoint `/v1/systemone` (bukan chat) | Rp 0,36 | ±Rp 1.000 |
| **judul** (jalur kualitas) | claude-haiku-5-5 | $0,10 / $0,50 | `thinking:{"type":"disabled"}` → reasoning 0; TANPA temperature | Rp 5,4 | ±Rp 15.000 |
| **qa** | jev-1.14.0 | $0,042 / $0,00 | sama dengan klasifikasi | ±Rp 0,36–0,50 | ±Rp 1.000–1.400 |

- **gpt-4o-mini:** 5/5 post OK, ~4 dtk/post, reasoning 0, max_tokens 600
  cukup. SATU TEMUAN: prompt awal membuatnya mengarang timestamps, akun
  medsos, dan placeholder di 3/5 post — prompt DIPERKERAS dan verifikasi
  ulang menunjukkan hasil BERSIH TOTAL (§8.2 — pakai prompt ini, bukan yang lama).
- **gpt-6-luna:** baru 1 video terverifikasi (output bersih tapi datar).
  Statusnya FALLBACK yang bisa dipilih di UI — tes 5 post (±Rp 11) adalah
  sub-tugas B1, BUKAN prasyarat peluncuran agent utama.
- **claude-haiku-5-5:** Bahasa Indonesia TERBAIK dari semua kandidat, tapi
  tokenizer-nya memakai ±80% token masuk lebih banyak (462 vs 256 pada video
  yang sama) dan MENOLAK parameter temperature (error 400) — jangan kirim
  temperature untuk model ini.
- **jev-1.14.0:** BUKAN penulis teks — model keputusan (ya/tidak + probabilitas,
  pilihan dari daftar, skor). Endpoint native `POST /v1/systemone` dengan
  format questions `{"type": "noul", "instructions": "..."}` — format lama
  `{"noul": {...}}` DITOLAK 400. Pertanyaan `choice` butuh field `criteria`
  sebagai DICT `{nama: deskripsi}` — BUKAN `options`, BUKAN list string
  (list DITOLAK 400 `dict_type`; menggagalkan pilot klasifikasi 10× pada
  24 Nov 10:18). Terverifikasi 200 OK, jawaban `{"noul": 0.32}`,
  biaya $0,0000127/panggilan untuk 1 pertanyaan noul; 3 pertanyaan
  (2 noul + 1 choice) $0,0000199 (±Rp 0,36)/panggilan.
- **Model yang DILARANG dipakai agent:** glm-5.3-flash (reasoning liar, mahal
  efektif), seed-2-0-mini (toggle diabaikan proxy, reasoning 1.052),
  mimo-v2.6-flash (toggle hanya sebagian, reasoning 365 — lebih mahal dari
  gpt-4o-mini), hy4-flash (thinking model; promo 50% berakhir ±29–30 Nov).
- **Sumopod TIDAK punya generate gambar/post** (60 model = teks + embedding +
  jev saja — terverifikasi dari dashboard + `/v1/models`). Gambar/thumbnail
  tetap jalur DeepInfra FLUX-1-schnell (Rp 35/gambar, keputusan 18 Nov);
  video tetap KEIN. JANGAN mencari endpoint image/post di Sumopod.

### 1.2 Keadaan aplikasi saat ini (fakta, 24 Nov 2026)

- Stack: Python 3.12.3, FastAPI 0.162.0, Uvicorn 0.54.0, Pydantic 2.14.3,
  TANPA SQLAlchemy/alembic. Venv: `/opt/masjavas-os/venv/bin/python`.
  WAJIB pakai venv ini — system python3 tidak punya fastapi.
- Pilot LangGraph Fase A SELESAI (commit `a1b9ab9`): graph audit di
  `core/audit_graph/{graph,state,nodes}.py`, checkpoint
  `/opt/masjavas-os/data/langgraph_checkpoints.db`, worker
  `worker/audit_graph_worker.py`. Titik sambung Fase B SUDAH disiapkan:
  `generate_metadata_stub(state)` di `core/audit_graph/nodes.py:192`
  (melempar `NotImplementedError("Fase B ditahan")`, diekspor
  `core/audit_graph/__init__.py`). Pekerjaan ini MENGGANTI badan stub itu —
  bukan membangun graph baru.
- Modul AI existing (PAKAI ULANG polanya, jangan tulis ulang):
  `core/ai/{description_polisher,gemini_pool,metadata_prompt,video_pipeline}.py`
  — `gemini_pool.py` adalah pola pool provider yang harus diikuti client
  Sumopod; `metadata_prompt.py` adalah pola manajemen prompt existing.
  Route existing: `api/routes/gemini.py`, `api/routes/ai_video.py`.
  Template existing: `web/templates/ai_pool.html` (UI pool AI — pola UI yang
  harus diikuti halaman manajemen agent), `web/templates/audit.html` (253 baris).
- Pola auth OWNER-only (WAJIB diikuti semua route baru — jangan ulangi celah
  P1-1 API tanpa auth): dependency di `api/routes/audit.py:30-40` — role harus
  `OWNER`, selain itu 403.
- Tabel audit existing: `video_audit`, `audit_channel_summary`,
  `video_repackage_jobs`, `video_repackage_outcomes`. Data live: 2.768 post
  terbit; 251 diaudit.
- Baseline SOLID (lantai, dari owner 22 Nov): **206 PUBLISHED/hari,
  0 FAILED/BLOCKED.** Setiap perubahan wajib membuktikan baseline tidak regresi.
- CATATAN P0-1 (masih terbuka): query Analytics belum meminta `impressions`,
  sehingga kategori A tidak pernah terpicu. Agent metadata TIDAK bergantung
  pada kategori (inputnya judul/views/durasi/kategori-apa-pun) — pilot jalan
  pada video kategori apa pun. Agent klasifikasi jev TIDAK menggantikan
  `core/audit/classifier.py` (deterministik, berbasis CTR/retensi) — jev
  menambah dimensi KUALITAS KONTEN (clickbait, kesesuaian judul, risiko),
  bukan menggantikan classifier yang sudah solid.

### 1.3 Biaya — angka konkret terverifikasi (kurs Rp 18.000/USD)

**Satuan biaya: "per post".** 1 post = 1 video YouTube yang sudah terbit =
1 panggilan API = 1 set metadata (3 judul + 1 deskripsi + 10 tag) atau
1 keputusan klasifikasi. Batch 2.768 post = backlog video terbit (sekali
jalan). Posting baru 206/hari × Rp 3,9 ≈ **Rp 800/hari** bila semua posting
baru memakai agent (metadata + klasifikasi).

| Kebutuhan | Biaya/batch 2.768 post | Biaya/post |
|---|---|---|
| Metadata (gpt-4o-mini) | ±Rp 9.600 | Rp 3,5 |
| Klasifikasi (jev-1.14.0) | ±Rp 1.000 | Rp 0,36 |
| **Total Fase B (utama)** | **±Rp 10.600** | **±Rp 3,9** |
| Total bila metadata pakai gpt-6-luna | ±Rp 7.000 | ±Rp 2,5 |
| QA (jev, opsional) | ±Rp 1.000–1.400 | ±Rp 0,36–0,50 |

Saldo AI Sumopod saat ini $5 (±Rp 90.000) — cukup untuk ±8 batch penuh
konfigurasi utama. **Aplikasi TIDAK PERNAH auto-top-up.** Bila saldo habis,
agent berhenti dengan status `NEEDS_TOPUP` dan dashboard menampilkan
peringatan — bukan error massal.

### 1.4 Ruang lingkup instruksi ini

**YA (Fase B1+B2+B3):** client provider Sumopod; 4 agent (metadata, klasifikasi,
judul, qa) sebagai node LangGraph + registry; prompt berversi; cost guard +
ledger biaya; tabel baru (aditif); API + UI manajemen agent (OWNER-only);
pilot 5 post → batch 100 → observasi 24 jam.

**BUKAN scope ini:** Fase C (Applier/edit metadata YouTube — DITAHAN, butuh
keputusan owner tersendiri); generate gambar/post (bukan Sumopod);
perubahan pada live/publish/ingestion/Stage 2/LoopGen; perubahan classifier
deterministik existing; timer otomatis agent (semua run MANUAL sampai owner
menyetujui jadwal); LangSmith.

---

## §2 — YANG SUDAH BENAR (terverifikasi, jangan diutak-atik)

1. Graph audit Fase A (`core/audit_graph/`) + checkpoint SQLite terpisah +
   titik `interrupt()` HITL setelah `summarize`. Fase B MENAMBAH node setelah
   titik itu — jangan memindahkan/menghapus interrupt.
2. Pipeline audit existing (`core/audit/collector.py`, `classifier.py`,
   `worker/audit_worker.py`) + timer audit existing.
3. Modul `core/ai/*` existing — baca dan ikuti polanya; jangan menulis ulang
   `metadata_prompt.py` / `gemini_pool.py`.
4. Worker produksi (`worker.autonomous_daemon`), live supervisor, ingestion,
   Stage 2, governor beban, sentinel — JANGAN SENTUH.
5. Baseline 206 PUBLISHED/hari, 0 FAILED/BLOCKED — lantai yang tidak boleh regresi.
6. Disiplin kunci DB: `acquire_db_write_lock` untuk tulis ke DB utama;
   `sqlite3.connect` ke DB live selalu `timeout=30`.
7. Aturan restart: maks 1 restart/15 menit lintas agent (jeda dihitung dari
   timestamp terverifikasi, bukan dari angka sebesar epoch — pelajaran 19 Nov).
8. Disiplin isolasi tes: tes memakai temp-DB, BUKAN DB live; venv
   `/opt/masjavas-os/venv/bin/python`.

---

## §3 — ARSITEKTUR

### 3.1 Client provider Sumopod (baru)

- File baru: `core/llm/sumopod.py` (paket `core/llm/` baru — sejajar
  `core/ai/`; client ini provider-agnostik di balik interface `chat()` +
  `decide()` sehingga model bisa diganti tanpa menyentuh agent).
  Ikuti pola pool/klien di `core/ai/gemini_pool.py` (retry, timeout, logging).
- Endpoint: `https://ai.sumopod.com/v1` — OpenAI-compatible.
  `chat()`: `POST /chat/completions` (model, messages, temperature,
  max_tokens, + parameter toggle opsional). `decide()`:
  `POST /v1/systemone` (model jev, state, questions — format §1.1).
- **API key: dari config terenkripsi OWNER-only yang SUDAH ADA di aplikasi
  (jalur yang sama dengan key lain) — JANGAN simpan key di plaintext, JANGAN
  simpan di tabel, JANGAN cetak di log/laporan.** Bila jalur config terenkripsi
  belum punya slot untuk Sumopod, tambahkan slotnya (bukan file .env baru).
  Key TIDAK PERNAH ditulis di dokumen/chat/laporan — owner yang mengisinya
  lewat dashboard (UI §3.7 menyediakan field input key yang langsung
  terenkripsi).
- Retry: maks 2 percobaan untuk error transient (timeout/5xx/429 dengan
  backoff). Error 4xx (auth, validasi, model tidak ada) TIDAK di-retry —
  dicatat dan agent ditandai `NEEDS_REVIEW`.
- Timeout: 120 dtk per panggilan chat (glm pernah 114 dtk; model pilihan
  ~4 dtk, jadi 120 dtk adalah batas aman, bukan target).
- Setiap panggilan WAJIB mencatat: model, prompt_tokens, completion_tokens,
  **reasoning_tokens** (dari `usage.completion_tokens_details`), finish_reason,
  latensi, biaya USD (dihitung dari tabel harga §8.1 — JANGAN hardcode harga
  di banyak tempat; satu sumber: `core/llm/prices.py`).

### 3.2 Registry agent (config + tabel)

4 agent, masing-masing SATU baris di tabel `llm_agents` (seed awal dari file
config, BUKAN hardcode di node):

| agent_id | model | temperature | max_tokens | toggle | prompt | budget/hari |
|---|---|---|---|---|---|---|
| `metadata` | gpt-4o-mini | 0,7 | 600 | — | `metadata_v1` (§8.2) | Rp 10.000 |
| `klasifikasi` | jev-1.14.0 | — | — | endpoint systemone | `klasifikasi_v1` (§8.3) | Rp 1.000 |
| `judul` | claude-haiku-5-5 | — (DILARANG kirim temperature) | 600 | `thinking:{"type":"disabled"}` | `judul_v1` | Rp 5.000 |
| `qa` | jev-1.14.0 | — | — | endpoint systemone | `qa_v1` (§8.3) | Rp 1.000 |

- Kolom `llm_agents`: `agent_id` (PK), `model`, `role` (deskripsi 1 baris),
  `params_json` (temperature/max_tokens/toggle), `prompt_id` (FK ke
  `llm_prompts`), `daily_budget_rp`, `status` (`ACTIVE`/`PAUSED`/`DISABLED`),
  `fallback_model` (untuk `metadata`: `gpt-6-luna` — hanya dipakai bila owner
  mengganti model di UI, BUKAN otomatis), `updated_at`, `updated_by`.
- Model yang bisa dipilih di UI DIBATASI pada daftar terverifikasi §1.1
  (dropdown, bukan input bebas) — mencegah owner/pengguna memilih model
  thinking yang boros tanpa sadar. Model di luar daftar = butuh penambahan
  eksplisit + probe.
- Agent `judul` bersifat OPSIONAL dan manual (owner memicu per post dari UI)
  — bukan bagian batch otomatis.

### 3.3 Wiring LangGraph (mengganti stub — mengikat)

- `core/audit_graph/nodes.py:192` — ganti badan `generate_metadata_stub`
  menjadi node `generate_metadata` yang: (1) membaca daftar video dari state
  (yang sudah diklasifikasi node `classify`); (2) untuk tiap post, memanggil
  agent `metadata` via client §3.1; (3) memvalidasi format output (harus ada
  `JUDUL:` + 3 butir, `DESKRIPSI:`, `TAG:` + 10 tag — bila tidak lengkap,
  1x retry dengan prompt yang sama, lalu tandai `NEEDS_REVIEW`); (4) menyimpan
  DRAFT ke `video_repackage_jobs` dengan status `DRAFT` (tabel existing —
  JANGAN buat tabel draft baru bila tabel ini sudah memadai; bila tidak
  memadai, tambahkan kolom, bukan tabel baru).
- Node `klasifikasi` (baru, setelah `classify` deterministik): memanggil agent
  `klasifikasi` (jev) untuk tiap post — pertanyaan noul: clickbait? judul
  sesuai isi? + choice: kategori konten. Hasilnya kolom TAMBAHAN di baris
  `video_audit` (atau tabel `llm_usage` + ringkasan di `audit_channel_summary`
  — pilih yang paling sedikit mengubah skema existing).
- Node `qa` (baru, setelah `generate_metadata`): agent `qa` (jev) memeriksa
  tiap draft — noul: "Apakah deskripsi mengandung timestamp?",
  "Apakah deskripsi mengandung akun media sosial atau placeholder?",
  "Apakah judul mengandung angka/nomor internal (slot)?". Draft dengan
  jawaban ya (probabilitas > 0,5) ditandai `NEEDS_REVIEW` + alasan —
  TIDAK PERNAH otomatis dihapus atau diperbaiki tanpa owner.
- **Titik interrupt HITL existing TETAP di antara draft dan aksi tulis apa pun.**
  Fase ini TIDAK MEMILIKI node tulis ke YouTube — graph berakhir setelah draft
  + QA. Bila ada kode yang memanggil YouTube Data API `videos.update`, fase
  ini GAGAL kriteria B-1.

### 3.4 Prompt berversi (tabel `llm_prompts`)

- Kolom: `prompt_id` (mis. `metadata_v1`), `agent_id`, `system_prompt`,
  `user_template`, `version`, `created_at`, `created_by`, `note`,
  `is_active` (satu prompt aktif per agent).
- Prompt v1 = teks TERVERIFIKASI di §8.2–§8.3 — JANGAN parafrase, JANGAN
  "perbaiki" tanpa tes. Perubahan prompt = versi baru + pilot 5 post ulang
  (kriteria B-2) sebelum diaktifkan.
- **PERUBAHAN WAJIB pada user prompt (pelajaran probe): HAPUS nomor slot
  dari user prompt.** Prompt tes lama menulis `Channel: X (slot 42)` dan
  gpt-5-nano membocorkan "Slot 42" ke judul publik. Slot adalah metadata
  internal — model tidak boleh melihatnya. User prompt v1: channel (nama
  saja), judul saat ini, views, rata-rata durasi tonton, umur video, hasil
  audit (kategori + alasan).
- `metadata_prompt.py` existing: bila sudah berisi template prompt metadata,
  prompt v1 DITAMBAHKAN sebagai template baru di sana (bukan file terpisah)
  — satu tempat untuk semua prompt metadata.

### 3.5 Cost guard (mengikat — ini inti permintaan owner)

- **Budget harian per agent** (kolom `daily_budget_rp`, default §3.2) +
  **budget harian global Rp 15.000** (satu batch penuh + margin). Sebelum
  SETIAP panggilan, client memeriksa: pengeluaran agent hari ini + estimasi
  panggilan ini ≤ budget agent DAN ≤ budget global. Bila tidak: panggilan
  DITOLAK, agent berstatus `PAUSED_BUDGET`, dashboard menampilkan peringatan.
  TIDAK ADA override otomatis — owner yang menaikkan budget di UI.
- **Alarm reasoning:** bila `reasoning_tokens > 100` pada agent yang
  seharusnya non-reasoning (metadata/judul), run ditandai `REASONING_ALARM`,
  agent di-PAUSE otomatis, dan dashboard menampilkan peringatan. (Pelajaran
  glm: reasoning liar adalah cara utama kredit terbuang diam-diam.)
- **Estimasi dry-run (wajib sebelum batch > 50 video):** UI menampilkan
  estimasi biaya = jumlah post × biaya/post agent (§8.1) + margin 10%,
  dan owner menekan "Jalankan" SETELAH melihat estimasi. Batch ≤ 50 video
  tetap menampilkan estimasi, tanpa persetujuan terpisah.
- **Kill switch global:** satu tombol di UI (OWNER-only) yang menghentikan
  SEMUA run agent dalam ≤ 10 detik (flag di DB yang dibaca tiap iterasi —
  bukan restart service). Status run yang terhenti: `STOPPED_BY_OWNER`.
- **Saldo $0:** bila Sumopod mengembalikan error saldo/insufficient balance,
  semua agent berstatus `NEEDS_TOPUP`, run berhenti bersih (bukan FAILED
  massal), dashboard menampilkan peringatan top-up. TIDAK ADA auto-top-up.

### 3.6 Tabel baru (aditif — nol perubahan skema existing)

- `llm_agents` (§3.2), `llm_prompts` (§3.4).
- `llm_usage` (ledger per panggilan — sumber kebenaran biaya): `id`,
  `run_id`, `agent_id`, `model`, `video_id` (nullable), `prompt_tokens`,
  `completion_tokens`, `reasoning_tokens`, `finish_reason`, `cost_usd`,
  `cost_rp` (kurs 18.000, dicatat saat insert), `latency_ms`, `status`,
  `created_at`. SATU baris per panggilan — tanpa agregasi prematur.
- `llm_runs`: `run_id`, `agent_id`, `triggered_by`, `scope` (slot/post list),
  `videos_total`, `videos_done`, `videos_failed`, `cost_rp_total`, `status`
  (`RUNNING`/`DONE`/`STOPPED_BY_OWNER`/`PAUSED_BUDGET`/`NEEDS_TOPUP`/
  `NEEDS_REVIEW`), `started_at`, `finished_at`.
- Index: `llm_usage(agent_id, created_at)`, `llm_usage(run_id)`.
  Semua tulis memakai `acquire_db_write_lock` (disiplin §2.6).

### 3.7 API + UI manajemen agent (OWNER-only)

- Route baru: `api/routes/agents.py` — ikuti pola `api/routes/audit.py`
  (dependency OWNER-only §1.2). Endpoint: `GET /api/agents` (daftar + status +
  biaya hari ini), `GET /api/agents/{id}` (detail + config + prompt aktif),
  `PUT /api/agents/{id}` (ubah model/params/budget/status — model hanya dari
  daftar terverifikasi), `GET /api/agents/{id}/usage` (ledger + agregasi),
  `POST /api/agents/{id}/run` (jalankan: pilot 5 / batch — dengan estimasi
  dry-run di respons), `POST /api/agents/{id}/pause|resume`,
  `POST /api/agents/kill-switch`, `GET /api/agents/runs/{run_id}`,
  `GET/PUT /api/agents/prompts` (versi prompt), `POST /api/agents/key`
  (isi/ganti API key Sumopod — langsung terenkripsi, tidak pernah
  dikembalikan ke client).
- Template baru: `web/templates/agents.html` — ikuti pola
  `web/templates/ai_pool.html` + `audit.html` (base template yang sama).
  Halaman memuat: (1) kartu per agent (status, model, biaya hari ini vs
  budget, run terakhir); (2) detail agent (config + prompt aktif + tombol
  pause/resume); (3) panel jalankan (pilih agent + scope + estimasi dry-run
  + tombol Jalankan); (4) dasbor biaya (grafik biaya/hari/agent 30 hari,
  alarm reasoning); (5) log run (tabel + filter status); (6) kill switch
  global (tombol merah, konfirmasi 2 langkah); (7) pengaturan key (field
  input key + status "tersimpan/terverifikasi" — JANGAN pernah menampilkan
  key).
- Tambahkan tautan "Agent AI" di navigasi dashboard (di samping Audit/AI Pool).

### 3.8 Observabilitas (minimal, tanpa platform baru)

- Log per run (1 baris JSON per post): run_id, agent_id, video_id, status,
  token (in/out/reasoning), cost_rp, latensi. File:
  `/opt/masjavas-os/data/logs/agents.log`.
- JANGAN memasang LangSmith / platform tracing berbayar. Dasbor biaya membaca
  `llm_usage` langsung.
- Endpoint status: `GET /api/agents/status` (OWNER-only) — run aktif, biaya
  hari ini per agent, status kill switch, status saldo (dari error terakhir
  atau cek `/v1/models` berkala — JANGAN polling saldo tiap menit).

---

## §4 — KRITERIA PENERIMAAN (definisi SELESAI — semua wajib hijau)

| # | Kriteria | Bukti yang diterima |
|---|---|---|
| B-1 | Tanpa tulis ke YouTube | `grep -r "videos.update\|youtube.*update" core/audit_graph/ core/llm/ core/agents/ worker/audit_graph_worker.py` menghasilkan NOL; graph berakhir setelah draft + QA |
| B-2 | Metadata 5 post benar + murah | Pilot 5 post (sampel yang sama dengan tes 24 Nov): 5/5 format lengkap (JUDUL 3/DESKRIPSI/TAG 10), 0 timestamp/medsos/placeholder/nomor-slot di output, biaya total ≤ Rp 25, latensi rata-rata ≤ 10 dtk/post |
| B-3 | Klasifikasi jev jalan | 5 post: jawaban bertipe (noul 0–1 + choice) tercatat, biaya total ≤ Rp 5; format request sesuai §8.3 (bukan format lama yang 400) |
| B-4 | Ledger biaya akurat | Untuk 20 panggilan acak: biaya tercatat di `llm_usage` = hitung ulang dari token × harga §8.1, selisih 0 (query §8.4) |
| B-5 | Budget cap ditegakkan | Tes dengan budget agent Rp 5: run berhenti sendiri setelah cap tercapai, agent `PAUSED_BUDGET`, 0 panggilan setelah cap, dashboard menampilkan peringatan |
| B-6 | Alarm reasoning jalan | Simulasi respons dengan reasoning_tokens 500 pada agent metadata → run ditandai `REASONING_ALARM`, agent di-pause otomatis |
| B-7 | UI + API OWNER-only, kill switch jalan | Non-OWNER mendapat 403 di semua route `/api/agents/*`; kill switch menghentikan run dalam ≤ 10 dtk dengan status `STOPPED_BY_OWNER`; key tidak pernah muncul di respons/log |
| B-8 | Tidak ganggu produksi + observasi 24 jam | Selama pilot + batch 100: load naik ≤ 0,5; 0 stream terganggu; PUBLISHED tidak turun dari baseline; 0 FAILED/BLOCKED baru; log + ledger utuh 24 jam; rollback teruji (hapus route/template/tabel baru → sistem kembali ke keadaan Fase A) |

**Aturan penilaian:** satu kriteria merah = fase BELUM selesai. Sebutkan B mana
yang hijau dan B mana yang merah, dengan bukti per B.

---

## §5 — SARAN & IDE TAMBAHAN (di luar scope — jangan dikerjakan sekarang)

1. **Agent konsep thumbnail:** menulis prompt gambar + teks overlay untuk
   FLUX-1-schnell (DeepInfra, BUKAN Sumopod) — agent teks yang sama
   (gpt-4o-mini) dengan prompt berbeda. Menunggu keputusan owner tentang
   alur thumbnail.
2. **Agent ide judul A/B:** haiku-5-5 menulis 5 varian judul per post;
   owner memilih manual. Sudah ada sebagai agent `judul` manual (§3.2) —
   ide tambahannya adalah uji A/B otomatis via YouTube Test & Compare (Fase C).
3. **Cost optimizer:** bila gpt-6-luna lolos tes 5 post, UI menawarkan
   "ganti metadata ke luna, hemat Rp 3.600/batch" — SATU klik, dengan
   catatan versi model di ledger (biaya lama vs baru terukur).
4. **Rantai QA ganda:** jev memeriksa draft mini SEBELUM HITL (sudah ada
   sebagai node `qa` §3.3) — ide lanjutannya: draft yang lolos QA dengan
   probabilitas tinggi masuk antrean prioritas HITL.
5. **Notifikasi cap/alarm:** OpenClaw sidecar / WhatsApp saat budget tercapai
   atau alarm reasoning menyala — pekerjaan terpisah, jangan dicampur.
6. **Prompt per genre:** template user prompt berbeda per klaster genre
   (koplo vs religi vs jazz) — setelah prompt v1 stabil 1 batch penuh.
7. **LangSmith (opsional, free tier):** tracing visual bila log JSON dirasa
   kurang — keputusan owner.

---

## §6 — INSTRUKSI PELAKSANAAN

**Urutan:** B0 (prasyarat) → B1 (provider + metadata + pilot 5) → B2 (jev) →
B3 (UI manajemen) → B4 (batch 100 + observasi 24 jam).

- **B0 — Prasyarat:** (a) Owner mengisi API key Sumopod lewat UI §3.7
  (atau config terenkripsi) — JANGAN mulai panggilan live tanpa key yang
  terverifikasi (`GET /v1/models` mengembalikan 60 model); (b) baca
  `FUNGSI_MODEL_SUMOPOD_2026-11-24.md` §1 + §8 dan laporan tes/probe —
  semua angka di instruksi ini berasal dari sana; (c) P0-1 TIDAK MENGHALANGI
  pilot (alasan §1.2) — tapi catat di laporan bahwa kategori A belum bisa
  terpicu sampai P0-1 selesai.
- **B1 — Provider + metadata + pilot 5:** client §3.1 + tabel §3.6 +
  node `generate_metadata` (§3.3) + prompt v1 (§8.2). Pilot 5 post
  (sampel tes 24 Nov). Buktikan B-2 + B-4. Sub-tugas opsional: tes 5 post
  gpt-6-luna (±Rp 11) — laporkan hasilnya, JANGAN mengganti model utama
  tanpa persetujuan owner.
- **B2 — Klasifikasi + QA (jev):** node `klasifikasi` + `qa` (§3.3),
  format request §8.3. Buktikan B-3. Perhatikan: `choice` butuh `criteria`
  sebagai DICT {nama: deskripsi} — bukan `options`, bukan list (list
  ditolak server 400 `dict_type` dan menggagalkan pilot klasifikasi
  10× pada 24 Nov 10:18; diperbaiki commit 253a72b).
- **B3 — UI manajemen:** route + template §3.7, cost guard §3.5, kill
  switch. Buktikan B-5 + B-6 + B-7 (tes cap memakai budget Rp 5, BUKAN
  budget produksi).
- **B4 — Batch 100 + observasi:** jalankan metadata + klasifikasi pada
  100 post (dry-run dulu, estimasi ±Rp 370), observasi 24 jam (B-8).
  Timer TETAP TIDAK ADA — semua run manual sampai owner menyetujui jadwal.

**Aturan perubahan (mengikat, dari owner 22 Nov):** aplikasi sudah SOLID —
setiap perubahan wajib rencana verifikasi pra/pasca + rollback + jendela
observasi. Jangan pernah mengorbankan bagian yang solid demi perbaikan.
Baseline (206 PUBLISHED/hari, 0 FAILED/BLOCKED) adalah lantai. Fase B
bersifat aditif — file + tabel + route baru, nol perubahan pada jalur
produksi existing kecuali badan stub §3.3.

**Aturan verifikasi (mengikat):**
- Buktikan perbaikan AKTIF dengan membandingkan waktu commit vs waktu start
  proses (service web/worker yang memuat kode baru).
- Buktikan hasil NYATA (query DB + log + ledger), bukan hanya status di memori.
- Jangan pernah mengklaim "100% SELESAI" tanpa bukti terukur per kriteria §4.
- Setiap logika baru disertai unit test minimal (client, cost guard, validasi
  format, alarm reasoning, auth route). Tes WAJIB pakai venv
  `/opt/masjavas-os/venv/bin/python` + temp-DB. Tes live (panggilan Sumopod
  nyata) hanya pada pilot 5 + batch 100 — JANGAN memicu batch penuh untuk
  pengujian.

**Format laporan per fase:** apa yang diubah (commit hash), bukti aktivasi
(PID + waktu start), bukti hasil (angka sebelum/sesudah + query verifikasi),
B mana yang hijau/merah, biaya API yang terpakai (dari `llm_usage`), apa yang
masih terbuka.

---

## §7 — LARANGAN

1. JANGAN membangun Fase C (node tulis YouTube / Applier) — keputusan owner
   belum dibuat. Agent fase ini hanya DRAFT.
2. JANGAN menyimpan/mencetak API key Sumopod di plaintext, tabel, log, laporan,
   atau chat. Key hanya di config terenkripsi + input UI §3.7.
3. JANGAN memakai model di luar daftar terverifikasi §1.1 tanpa probe +
   persetujuan owner. Khususnya JANGAN memakai glm-5.3-flash, seed-2-0-mini,
   mimo-v2.6-flash, atau hy4-flash untuk agent.
4. JANGAN memasukkan nomor slot (atau metadata internal lain) ke prompt —
   model membocorkannya ke output publik (pelajaran probe gpt-5-nano).
5. JANGAN mengubah kode live streaming, publish, ingestion, Stage 2, LoopGen,
   atau classifier deterministik existing.
6. JANGAN mengaktifkan timer/jadwal otomatis agent tanpa persetujuan eksplisit
   owner setelah B-1 s.d. B-8 hijau.
7. JANGAN memicu batch penuh (2.768 post) untuk pengujian — maks batch uji 100.
8. JANGAN mengklaim fase selesai tanpa bukti aktivasi + hasil per kriteria §4.
9. JANGAN restart live/publish worker — hanya service yang memuat kode baru
   (web/worker audit-graph) bila perlu, maks 1/15 menit lintas agent.
10. JANGAN auto-top-up saldo Sumopod dalam kondisi apa pun.
11. JANGAN mencari/memakai endpoint image/post di Sumopod — tidak ada.

---

## §8 — LAMPIRAN: FAKTA KUNCI TERVERIFIKASI

### 8.1 Tabel harga Sumopod (USD/1 jt token, dashboard 24 Nov 2026)

| Model | In | Out | Catatan |
|---|---|---|---|
| gpt-4o-mini | $0,15 | $0,60 | Non-reasoning. ~257 in + ~240 out token/post |
| gpt-6-luna | $0,10 | $0,50 | `reasoning_effort:"none"` → reasoning 0 |
| gpt-5-nano | $0,05 | $0,40 | `reasoning_effort:"minimal"` → reasoning 0; bocor "Slot 42" |
| claude-haiku-5-5 | $0,10 | $0,50 | `thinking:{"type":"disabled"}`; TANPA temperature; tokenizer +80% |
| mimo-v2.6-flash | $0,14 | $0,28 | Toggle SEBAGIAN (reasoning 365) — DILARANG |
| seed-2-0-mini | $0,10 | $0,40 | Toggle diabaikan (reasoning 1.052) — DILARANG |
| hy4-flash | $0,035 | $0,14 | Thinking; promo 50% berakhir ±29–30 Nov — DILARANG |
| glm-5.3-flash | $0,015 | $0,25 | Thinking liar (2.023–7.187) — DILARANG |
| jev-1.14.0 | $0,042 | $0,00 | `/v1/systemone`; output GRATIS; ~303 in token/panggilan |

Kurs acuan: Rp 18.000/USD. Harga wajib diverifikasi ulang di dashboard
sebelum top-up besar.

### 8.2 Prompt metadata v1 (TERVERIFIKASI — pakai persis, jangan parafrase)

**System prompt:**
> Kamu adalah ahli metadata YouTube untuk channel musik Indonesia (kompilasi
> lagu, koplo, dangdut, slow rock, akustik, nostalgia). Tulis dalam Bahasa
> Indonesia yang natural dan enak dibaca, bukan terjemahan kaku. Jangan
> mengarang nama artis, angka, atau klaim yang tidak ada di data yang
> diberikan. JANGAN tambahkan timestamps, JANGAN tambahkan akun media sosial,
> JANGAN pakai placeholder seperti [Durasi video] — hanya tulis JUDUL,
> DESKRIPSI, dan TAG sesuai format, tidak ada bagian lain.

**User prompt (template — TANPA nomor slot):**
> Buatkan metadata baru untuk video YouTube ini:
>
> Channel: {channel_name}
> Judul saat ini: {title}
> Views: {views} | Rata-rata durasi tonton: {dur_sec} detik | Umur video: {age_days} hari
> Hasil audit: kategori {category} — {category_reason}
>
> Tugas: tulis metadata yang lebih menarik klik dan SEO-friendly, tetap jujur
> dengan isi video (jangan clickbait berlebihan, jangan karang fakta).
>
> Format jawaban (ikuti persis):
> JUDUL:
> 1. ...
> 2. ...
> 3. ...
> DESKRIPSI:
> ...
> TAG:
> tag1, tag2, ..., tag10

Parameter: temperature 0,7, max_tokens 600.

### 8.3 Format jev (TERVERIFIKASI — format lain DITOLAK 400)

Request `POST https://ai.sumopod.com/v1/systemone`:
```json
{
  "model": "jev-1.14.0",
  "state": "Judul video: ... Channel: ... Views: ...",
  "questions": {
    "is_clickbait": {"type": "noul", "instructions": "Apakah judul ini clickbait?"},
    "kategori": {"type": "choice", "instructions": "Pilih kategori video ini.", "criteria": {"musik": "Video berisi musik atau lagu.", "berita": "Video berisi berita atau jurnalisme.", "lainnya": "Kategori lain."}}
  }
}
```
Respons 200 (terverifikasi): `{"model": "typesafe/jev-1.14-20261014",
"answers": {"is_clickbait": {"type": "noul", "noul": 0.32}}, "usage":
{"input_tokens": 303, "output_tokens": 23, "cost": 1.2726e-05}}`.
Perhatikan: `choice` memakai field `criteria` sebagai DICT {nama:
deskripsi} — BUKAN `options`, BUKAN list (list ditolak 400 `dict_type`).
Jawaban choice: `{"type": "choice", "choice": "musik", "confidence": 0.92,
"probabilities": {...}}`. Biaya 3 pertanyaan: $0,0000199 (±Rp 0,36).

### 8.4 Query verifikasi

- B-2 (format + biaya pilot): `SELECT count(*) FROM video_repackage_jobs
  WHERE status='DRAFT' AND created_at >= <awal_pilot>;` (harus 5) +
  `SELECT sum(cost_rp) FROM llm_usage WHERE run_id=<run_pilot>;` (≤ Rp 25).
- B-4 (ledger akurat): `SELECT u.id, u.cost_usd, (u.prompt_tokens*0.15 +
  u.completion_tokens*0.60)/1000000 AS hitung FROM llm_usage u WHERE
  u.model='gpt-4o-mini' AND abs(u.cost_usd - hitung) > 0.000001;`
  (harus 0 baris; sesuaikan harga per model dari §8.1).
- Biaya hari ini per agent: `SELECT agent_id, sum(cost_rp) FROM llm_usage
  WHERE date(created_at,'unixepoch') = date('now') GROUP BY 1;`

### 8.5 File & referensi

- Repo: `masjavas7/MASJAVASTUBE-OS`. Fase A HEAD `a1b9ab9`.
- Stub: `core/audit_graph/nodes.py:192` (`generate_metadata_stub`).
- Pola yang harus diikuti: `core/ai/gemini_pool.py` (client),
  `core/ai/metadata_prompt.py` (prompt), `api/routes/audit.py:30-40` (auth
  OWNER-only), `web/templates/ai_pool.html` (UI).
- File BARU fase ini: `core/llm/sumopod.py`, `core/llm/prices.py`,
  `core/agents/registry.py` (atau setara), `api/routes/agents.py`,
  `web/templates/agents.html`, `tests/test_agents_*.py`,
  `/opt/masjavas-os/data/logs/agents.log`, tabel `llm_agents`,
  `llm_prompts`, `llm_usage`, `llm_runs`.
- Dokumen acuan (di mesin Muse):
  `~/workspace/engineering/FUNGSI_MODEL_SUMOPOD_2026-11-24.md`,
  `~/workspace/engineering/sumopod-test/hasil_tes_sumopod_20261124-090416.md`,
  `~/workspace/engineering/sumopod-test/probe_hasil_20261124-091804.md`,
  `~/workspace/engineering/sumopod-test/test_sumopod_metadata.py` (prompt
  + pemanggil terverifikasi — rujukan implementasi client),
  `~/workspace/engineering/INSTRUKSI_LANGGRAPH_ANTIGRAVITY_2026-11-24.md`.

---

*Dokumen ini disusun dari tes live 10 panggilan + paket probe 5 toggle +
verifikasi endpoint jev + pembacaan kode aplikasi via SSH read-only
(24 Nov 2026, 09:04–09:45 WIB). Semua klaim dapat diverifikasi ulang dari
file, query, dan laporan yang tercantum.*
