# Paket Keputusan Fase C — Apply DRAFT Agent AI ke YouTube

**Tanggal:** 25 Nov 2026 08:40 WIB · **Item:** Fase 3.2 (RENCANA §7)
**Status:** SIAP DIPUTUSKAN — pilot judul + QA selesai, dry-run batch 1 selesai.
**Bukti:** salinan DB temp 25 Nov 08:34 (`/tmp/fase3_pilot.sqlite` di Master);
laporan pilot `/tmp/fase3_pilot_report.json`; batch 1 `/tmp/fase3_batch1.json`.
Kode: commit `668130c` (harness + guard double-submit).

---

## 1. Ringkasan pilot (live, temp DB — Rp 20,52)

| Pilot | Hasil |
|---|---|
| Judul, 5 post (haiku-5-5) | 5/5 DONE, 25 varian, **0 judul >100 karakter** (60–83), biaya Rp 12,58 (~Rp 2,52/post — di bawah estimasi Rp 5,4) |
| QA, 20 DRAFT (jev-1.14.0) | 18 bersih / **2 ter-flag** `ada_nomor_internal` (noul 0,91 & 0,68; maks bersih 0,08 — separasi bersih), biaya ~Rp 7,94 (~Rp 0,40/draft) |
| Flip NEEDS_REVIEW | Berfungsi: DRAFT 102 → 100, NEEDS_REVIEW 0 → 2 (di temp DB) |

**Kualitas judul — contoh (video `6fLJt4d2HAg`, views 809):**
lama: "Malaysia Viral 10 Lagu Melayu Paling Menyentuh Hati" →
baru: "10 Lagu Melayu yang Bikin Hati Tersentuh, Cocok Didengar Malam Hari" /
"Kumpulan 10 Lagu Melayu Paling Menyentuh Hati, Teman Nostalgia Masa SMA" /
"Lagu Melayu Penyentuh Hati: 10 Pilihan Terbaik untuk Kenangan Masa Muda".
Natural, nama artis/lagu dipertahankan, tanpa clickbait kasar.

**Temuan QA — 2 flag adalah FALSE POSITIVE substansi:** angka yang
di-flag adalah bagian judul lagu ("Satru 3", "1001 Macam"), BUKAN nomor
slot internal. Desain sudah benar (flag → NEEDS_REVIEW untuk review
manusia, bukan auto-reject), tapi pertanyaan QA perlu disempurnakan
("nomor slot" vs "angka dalam judul lagu") sebelum QA massal.
**Temuan metadata — 1 judul mengarang "Resmi":** draft `#9`
`_tv2Uayq50c` mengusulkan "(Lirik Resmi)" untuk channel cover —
kata "Resmi" tidak ada di judul lama. QA saat ini tidak menangkap ini.

## 2. Status DRAFT (terverifikasi dari salinan DB)

- **102 baris DRAFT, 100 video unik** — 2 video punya 2 draft aktif
  (`jO_-OFowJ8Q`, `Q7-AZuCUOyA`, dari 2 pilot 24 Nov sebelum dedupe K2).
- **Eligible Fase C: 98** (DRAFT terbaru per video; 2 NEEDS_REVIEW
  dikecualikan; 2 duplikat lama dibuang).
- **0 pernah di-apply** (`applied_at` terisi 0/102). Total spend agent
  Rp 358,52 (metadata Rp 322,06 + klasifikasi Rp 36,46); judul + QA
  sebelum pilot ini 0 run.
- 10 channel, 10 vault (maks 15/vault).

## 3. VERIFIKASI ROLLBACK — TIDAK LENGKAP ⚠️

Rencana menyebut "rollback via `old_*` + backup thumbnail". Verifikasi
di kode + data membuktikan mekanisme itu **BELUM ADA**:

| Komponen rollback | Status |
|---|---|
| `old_title` | ✅ terisi 102/102 (`_save_draft`, `core/agents/runner.py:333-365`) |
| `old_description` | ❌ **KOSONG 102/102** — `_save_draft` meng-INSERT literal `''` |
| `old_tags` | ❌ kolom tidak ada di tabel |
| `backup_thumbnail_path` | ❌ terisi 0/102; **tidak ada kode** yang mengisinya (grep kosong) |
| `old_thumbnail_url` | ❌ terisi 0/102 |
| Kode apply ke YouTube | ❌ **TIDAK ADA** — tidak ada `apply_repackage`/`fase_c` di repo; `videos().update` hanya ada di jalur live (`core/youtube/live_broadcaster.py`), bukan repackage |
| Kode rollback | ❌ **TIDAK ADA** — `auto_rollback_triggered`/`applied_at` tidak pernah ditulis kode mana pun (hanya skema) |
| `change_thumbnail` | 0/102 = 1 — draft saat ini hanya judul+deskripsi+tag |

**Implikasi:** Fase C hari ini hanya bisa me-rollback **judul** dari baris
draft. Rollback deskripsi/tag/thumbnail membutuhkan pengambilan metadata
lama dari YouTube API **sebelum** apply (dan menyimpannya) — itu pekerjaan
pembangunan applyer, bukan sekadar persetujuan. Jangan janjikan rollback
penuh sampai applyer + snapshot pra-apply dibangun dan dites.

## 4. Aturan batch yang diusulkan

1. **Ukuran batch: 10 DRAFT/batch** (10 batch untuk 98 eligible).
   Alasan: kuota kecil, blast radius kecil, review manusia masih feasible.
   Alternatif: 20/batch (5 batch) bila batch 1–2 mulus.
2. **Per batch:** persetujuan OWNER per batch (bukan sekali untuk semua);
   jeda observasi ≥24 jam antar batch untuk melihat efek (views/CTR);
   hentikan batch berikutnya bila ada 2+ masalah dalam satu batch.
3. **Urutan batch 1: views DESC** (proksi dampak terbaik yang tersedia
   hari ini). Urutan id mengelompok per channel (batch 1 by-id = 6/10
   dari Melodi Terakhir). Opsi lain: round-robin per channel.
4. **Eligibilitas per batch:** hanya DRAFT terbaru per video;
   NEEDS_REVIEW dikecualikan sampai review manusia selesai;
   duplikat lama dibuang.
5. **Kuota:** `videos.update` = 50 unit/video → batch 10 = **500 unit**,
   tersebar ~5 vault (~50–300 unit/vault); total 98 = **4.900 unit**.
   Kecil dibanding kuota harian — bukan penghambat.
6. **Pra-syarat apply (harus dibangun dulu — lihat §6):** snapshot
   metadata lama dari YouTube sebelum apply (judul+deskripsi+tag+
   thumbnail) + tulis `applied_at` + tombol rollback per batch.

## 5. Dry-run batch pertama — YANG AKAN BERUBAH (10 DRAFT, urutan id)

Dari DB temp, **tanpa eksekusi**. Ringkas (lengkap di `/tmp/fase3_batch1.json`):

| # | Video | Channel | Lama → Baru (judul) |
|---|---|---|---|
| 3 | C-cMV9BcOLU | Melodi Terakhir | Ketika Satu Pelita Menolak Padam... → Ketika Satu Pelita Tak Pernah Padam |
| 4 | mtiKyCzh5Vk | Melodi Terakhir | Indah Yastami Rembulan Malam • Akustik… → Rembulan Malam - Indah Yastami \| Akustik yang Menghanyutkan |
| 5 | 0tvnWp1MMvk | Melodi Terakhir | Luka Sekerat Rasa… Syahdu di Kedai Kopi → …Cover Syahdu di Kedai Kopi |
| 6 | jO_-OFowJ8Q | Bass Pantura | Ida Laila Koplo Pilihan Terbaik… → Koleksi Terbaik Ida Laila dalam Versi Koplo… |
| 7 | Q7-AZuCUOyA | Jiwang Ombak | Andai Kau Tau Hatiku — Slow Rock Melayu Terbaik 2026 → …Slow Rock Melayu Penuh Perasaan |
| 8 | Z2n1qcXsDSg | Melodi Terakhir | Hati Siapa Tak Luka ~ …Cover Akustik Damai → …Cover Akustik … yang Menyentuh |
| 9 | _tv2Uayq50c | Jalan Kenangan | Imam S. Arifin – Jangan Merana Lirik → …(Lirik Resmi) ⚠️ kata "Resmi" mengarang |
| 10 | tOU5dwl34gc | Bass Goyang | Denny Caknan Feat. Wawes – Dumes… → Denny Caknan & Wawes - Dumes: Melodi Nostalgia… |
| 11 | Ikw-wT5cYIs | Melodi Terakhir | Nostalgia Indah Yastami Bila Nanti… → Kenangan Manis: Cover Lagu Bila Nanti… |
| 12 | pzaLcXWDdTA | Melodi Terakhir | Air Mata Di Hari Persandinganmu… → Air Mata di Hari Persandinganmu… yang Menggetarkan |

Tiap apply juga mengganti **deskripsi (401–510 karakter baru)** dan **tag**.
Kuota batch 1: 500 unit (vault dominan MV-…B476CC: 6 video = 300 unit).
**Perubahan ini BELUM terjadi** — dry-run dari DB temp saja.

## 6. Keputusan yang dibutuhkan dari user

1. **Fase C ya/tidak?** (Keputusan utama Rencana §7 3.2.)
2. **Ukuran batch:** 10 (disarankan) atau 20 per batch?
3. **Urutan:** views DESC (disarankan) / urutan id / round-robin channel?
4. **Bangun applyer dulu?** Apply + snapshot pra-apply + rollback
   **belum ada kodenya** — perlu 1–2 hari pembangunan + tes sebelum
   batch 1 bisa dieksekusi. Setujui pembangunan itu sebagai scope
   lanjutan, atau tunda Fase C sampai applyer siap?
5. **Kata "Resmi":** keluarkan draft `#9` dari batch 1 (disarankan)
   atau biarkan?
6. **Perbaiki pertanyaan QA** (angka judul lagu vs nomor slot) sebelum
   QA massal? (Disarankan: ya — kecil, 1 prompt.)

## 7. Dependensi CTR — putusan: FASE C TIDAK PERLU MENUNGGU CTR

- Triase CTR idealnya memprioritaskan batch (video CTR rendah dulu).
- **Fakta hari ini:** 0/98 video eligible punya CTR nyata di audit
  terbaru. Satu-satunya CTR >0 di DB adalah **87 baris fabrikasi lama
  0,0455 (24 Nov 07:12, impressions = views×22)** — nilai yang sama
  yang dihapus commit `8fdfcbb`. Audit terbaru (25 Nov 05:41) CTR = 0.
- Kode CTR 1.2 (`8fdfcbb`) sudah commit, tapi audit pertama dengan CTR
  nyata belum mendarat; verifikasi live dijadwalkan malam ini 19:00.
- **Putusan:** Fase C bisa jalan dengan urutan manual sekarang
  (views DESC). Bila verifikasi 19:00 mengonfirmasi CTR nyata mengalir,
  triase-CTR dipakai mulai batch 2. Menunggu CTR untuk batch 1 hanya
  menunda tanpa manfaat (batch 1 = 10 video, risiko kecil).
- Catatan data: 87 baris CTR fabrikasi masih tersimpan di
  `video_audit` — perlu dibersihkan/ditandai agar tidak dipakai
  triase kelak (di luar scope 3.2; diteruskan sebagai temuan).

## 8. Peta kode (ringkas — lengkap di laporan worker 3.2)

- Runner: `core/agents/runner.py` — `run_metadata_agent:527`,
  `run_jev_agent:776` (klasifikasi/QA), `run_judul_agent:941`,
  `_save_draft:333`, orchestrator `_orchestrate_draft:435` (default OFF),
  `load_posts_for_scope:1081`. Header menegaskan: tidak ada tulis
  YouTube di modul ini.
- API: `api/routes/agents.py` — `run_agent:353` (dry-run wajib >50 post),
  guard 409 + pre-create baris run (baru, `668130c`).
- Ledger: `core/llm/ledger.py` — `create_run` idempotent,
  `get_running_run` (baru, `668130c`).
- UI: `web/templates/agents.html` — `startRun`/`dryRun` kini mengunci
  tombol di baris pertama (baru, `668130c`).
- Harness: `tools/pilot_agent_fase3.py` (baru). Tes:
  `tests/test_agents_fase3_pilot.py` 13/13 hijau;
  regresi `tests/test_agents_fase_b.py` 36/36 hijau.

## 9. Temuan minor (dicatat, tidak diubah)

- Docstring `run_agent` mengklaim "Kembali 202" — aktual 200 (FastAPI
  default; UI tidak terpengaruh karena cek `res.ok`). Perbaiki docstring
  atau status code saat sentuhan berikutnya.
- 2 run klasifikasi 24 Nov 03:18 gagal 5/5 sebelum pilot 5/5 berhasil
  03:24 — riwayat, bukan masalah aktif.
