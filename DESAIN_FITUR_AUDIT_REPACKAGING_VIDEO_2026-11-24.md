# DESAIN FITUR AUDIT & REPACKAGING VIDEO TERBIT

**Tanggal:** Selasa, 24 November 2026
**Status:** DRAFT v1.1 — menunggu persetujuan owner
**Riwayat revisi:**
- **v1.0** (06:46 WIB) — desain awal: arsitektur worker, skema tabel,
  perhitungan kuota, klasifikasi A/B/C/D, tahapan, kriteria penerimaan.
- **v1.1** (06:50 WIB) — 12 bottleneck tambahan + solusi (§13);
  audit bertingkat (§3.1, §5.4); jendela edit 15:00–17:00 (§3.1);
  targeting berdasarkan potensi kenaikan CTR (§6.3); dimensi monetisasi
  (§4.1, §6.1); ambang impresi minimum untuk vonis (§4.3); T7 sebagai
  sinyal utama (§8, §9); review berdasarkan pengecualian (§3.4, §8, §9);
  hierarki fallback generator (§7.1); risiko R-9 s.d. R-11 (§10).
**Penulis:** Muse
**Lokasi eksekusi:** VPS 8 (live-01) — worker baru `masjavas-audit`
**Skala yang diaudit:** 2.759 video terbit di 210 channel (per 06:46 WIB, 24 Nov 2026)

> **Ringkasan satu kalimat:** fitur ini mengaudit semua video yang sudah terbit,
> mengklasifikasikan tiap video berdasarkan sinyal performa nyata (impresi, CTR,
> retensi), dan HANYA mengedit ulang kemasan (judul + thumbnail + deskripsi)
> video yang sinyalnya menunjukkan masalah kemasan — dengan anggaran kuota
> sendiri, dan setiap edit diukur hasil sebelum/sesudahnya.

---

## §1 — LATAR BELAKANG & MASALAH

### 1.1 Masalah yang ingin diselesaikan

Banyak channel armada MASJAVAS TUBE yang sudah publish memiliki performa
sangat buruk — view rendah atau nol. Publish terus berjalan (207 video/hari),
tapi sebagian besar video tidak pernah ditonton. Tanpa audit, tidak ada yang
tahu:

- Video mana yang 0 view karena **tidak pernah didistribusikan** (0 impresi)?
- Video mana yang punya impresi tapi **tidak diklik** (CTR rendah)?
- Video mana yang diklik tapi **ditinggal penonton** (retensi rendah)?
- Channel mana yang seluruh videonya mati (masalah level channel)?

Ketiga penyebab itu membutuhkan tindakan yang BERBEDA. Mengedit metadata
semua video secara membabi buta tidak membedakan ketiganya — dan akan
menghabiskan kuota untuk video yang tidak akan merespons.

### 1.2 Ide mentah vs desain ini

| Ide mentah (owner) | Penilaian | Desain ini |
|---|---|---|
| Audit semua post terbit | ✅ Benar — audit adalah fondasi | Dipertahankan sebagai Fase 0 |
| Target channel view rendah/nol | ✅ Benar — populasi yang tepat | Dipertahankan, dengan triase lebih lanjut |
| Edit judul, deskripsi, tag, thumbnail | ⚠️ Sebagian benar | Hanya judul + thumbnail + deskripsi faktual; tag diprioritaskan rendah |
| Koreksi dari bahan GDrive | ⚠️ Sebagian benar | Bahan dipakai HANYA untuk akurasi isi (daftar lagu, chapter) — bukan untuk menentukan apa yang akan perform |
| "Akan kembali terindeks & didorong algoritma" | ❌ Naif — mekanisme itu tidak ada | Diganti: target nyata adalah **menaikkan CTR** pada video yang sudah punya impresi; CTR yang naik itulah yang menaikkan distribusi |
| Edit semua post | ❌ Naif — boros kuota, hasil tidak terukur | Diganti: edit HANYA kategori A (impresi ada, CTR rendah) |

### 1.3 Fakta YouTube yang mendasari desain

1. **Tidak ada "re-index boost" untuk edit metadata.** YouTube tidak
   memberi dorongan kesegaran pada video yang metadatanya diubah.
2. **Distribusi mengikuti perilaku penonton** (CTR, retensi, watch time,
   session time) — bukan kesegaran metadata.
3. **Tag hampir tidak berpengaruh** pada ranking (pernyataan resmi YouTube).
4. **Thumbnail + judul adalah tuas kemasan tertinggi** — keduanya menentukan
   CTR, dan CTR menentukan impresi lanjutan.
5. **Video dengan 0 impresi tidak akan bangkit dari edit metadata** —
   algoritma belum pernah mengujinya pada penonton, atau sudah mengujinya
   dan berhenti. Masalahnya adalah distribusi, bukan kemasan.
6. **Mengedit video yang sedang bekerja berisiko** merusak sinyal yang
   sudah berjalan — jangan sentuh yang performanya baik.

---

## §2 — PRINSIP DESAIN (mengikat)

1. **Triase berdasarkan sinyal, bukan berdasarkan view.** Tindakan ditentukan
   oleh kombinasi impresi + CTR + retensi (§6), bukan oleh "view rendah" saja.
2. **Audit dulu, edit kemudian.** Tidak ada satu pun edit sebelum audit
   Fase 0 selesai dan klasifikasi terbit.
3. **Edit hanya yang punya peluang.** Hanya kategori A (impresi ada, CTR
   rendah) yang diedit. Kategori lain dicatat dengan rekomendasi masing-masing.
4. **Kuota sendiri, tidak pernah mengambil kuota publish.** Fitur ini punya
   anggaran harian dan governor sendiri (§5). Bila kuota publish tertekan,
   fitur ini berhenti dulu.
5. **Setiap edit diukur.** Snapshot sebelum + pengukuran +7 hari dan +14 hari.
   Tanpa loop ukur, fitur ini tidak bisa diklaim berhasil.
6. **Jangan sentuh yang bekerja.** Video dengan CTR/view baik tidak diedit.
7. **Validasi di titik masuk.** Metadata baru harus lolos validasi (panjang
   judul, kecocokan isi, tidak ada klaim palsu) sebelum dikirim ke API.
8. **Bertahap dan bisa dihentikan.** Pilot kecil → ukur → skala. Setiap fase
   punya kriteria lanjut/berhenti (§9).

---

## §3 — ARSITEKTUR WORKER

### 3.1 Penempatan

- **Worker baru `masjavas-audit`** berjalan di **VPS 8** (node worker),
  sebagai systemd service + timer (bukan daemon kontinu — pekerjaan ini
  batch, bukan real-time).
- **Control plane tetap di Master:** DB (sumber kebenaran), dashboard,
  dan API. Worker VPS 8 membaca/menulis DB via API Master (prinsip P4
  arsitektur multi-node) atau via salinan DB lokal + fleet sync —
  keputusan implementasi di §3.5.
- **Jadwal (v1.1 — audit bertingkat):** berjalan di luar jam prime
  (hindari 17:00–22:30 WIB).
  - Koleksi inkremental harian (video baru + jendela ukur): 01:00–02:00 WIB.
  - Koleksi bergilir mingguan (¼ katalog): 01:00–03:00 WIB, satu hari
    per minggu.
  - Koleksi penuh bulanan: 01:00–05:00 WIB, satu hari per bulan.
  - **Edit diterapkan 15:00–17:00 WIB (jendela utama)** — kuota baru saja
    reset (15:00 WIB) dan masih sebelum jam prime live. Jendela
    06:00–10:00 WIB menjadi cadangan bila antrean tidak habis.

### 3.2 Komponen

```
┌─────────────────────────────────────────────────────────────────┐
│                    masjavas-audit (VPS 8)                        │
│                                                                  │
│  ┌──────────┐   ┌────────────┐   ┌───────────┐   ┌───────────┐  │
│  │Collector │──▶│Classifier  │──▶│Generator  │──▶│Applier    │  │
│  │(baca)    │   │(A/B/C/D)   │   │(judul/thum│   │(API write │  │
│  │          │   │            │   │bnail/desk)│   │+ governor)│  │
│  └──────────┘   └────────────┘   └───────────┘   └─────┬─────┘  │
│       ▲                                               │         │
│       │            ┌──────────────┐                   ▼         │
│       └────────────│Measurer      │◀───────── ┌─────────────┐  │
│  (snapshot +7/+14) │(before/after)│            │Reporter     │  │
│                    └──────────────┘            │(dashboard)  │  │
│                                                └─────────────┘  │
└─────────────────────────────────────────────────────────────────┘
         ▲                      │                       │
         │ YouTube Data API     │ YouTube Analytics API │ YouTube Data API
         │ + Analytics API      │ (read)                │ (videos.update,
         │ (read, murah)        │                       │  thumbnails.set)
```

| Komponen | Tugas | Biaya API |
|---|---|---|
| **Collector** | Ambil daftar video per channel (`videos.list`) + metrik per video (Analytics `reports.query`: views, impresi, CTR, retensi, sumber traffic) + metadata saat ini (judul, deskripsi, thumbnail). Video ID diambil dari `operational_tasks.youtube_video_id` (terisi 100%, 2.760/2.760 — tidak butuh `search.list`). | Audit penuh: ~56 unit (daftar) + ~210–420 unit (analitik). Audit inkremental harian: ~50–100 unit. Total audit/bulan (bertingkat): ~2.000–3.000 unit |
| **Classifier** | Terapkan aturan §6 → kategori A/B/C/D per video + ringkasan per channel | 0 (komputasi lokal) |
| **Generator** | Buat judul baru (dari pola CTR terbaik armada per genre), thumbnail baru (pipeline thumbnail yang sudah ada), deskripsi baru (daftar lagu akurat dari bahan + chapter/timestamp) | 0 API (thumbnail di-render lokal; bila memakai AI image, biaya terpisah) |
| **Applier** | Kirim `videos.update` (judul/deskripsi/tag) dan `thumbnails.set` lewat token vault pemilik video, dengan governor kuota (§5) + retry/backoff + pencatatan | 50 unit per update + 50 unit per thumbnail |
| **Measurer** | Snapshot metrik di T0 (saat edit), T+7 hari, T+14 hari; hitung delta CTR/view/impresi; tandai edit sebagai BERHASIL / NETRAL / GAGAL | ~1 unit per video per pengukuran |
| **Reporter** | Dashboard + API: ringkasan per channel, antrean edit, hasil sebelum/sesudah, anggaran kuota terpakai | 0 |

### 3.3 Integrasi dengan sistem yang sudah ada

| Sistem yang ada | Cara dipakai |
|---|---|
| `auth_tokens` (token OAuth per vault) | Applier memakai token vault pemilik channel — SAMA seperti publish. Tidak ada kredensial baru. |
| `vault_quota_ledger` | Setiap unit yang dipakai fitur ini dicatat dengan `operation_type` baru (`AUDIT_UPDATE`, `AUDIT_THUMBNAIL`) agar terpisah dari publish di laporan kuota. |
| `operational_tasks` (status PUBLISHED) | Sumber daftar video terbit + `channel_id` + tanggal publish. Collector tidak perlu meraba-raba video ID. |
| Pipeline thumbnail (ingestion/LoopGen) | Generator thumbnail memakai ulang generator yang sudah ada — bukan membangun baru. |
| Bahan GDrive (`LINK_GDRIVE_BAHAN`) | Generator deskripsi membaca daftar lagu dari bahan untuk akurasi tracklist + chapter. Baca saja, tidak mengubah bahan. |
| Fleet sync (90 detik) | Hasil audit yang ditulis ke DB VPS 8 tersinkron ke Master (atau langsung tulis via API Master — lihat §3.5). |
| Governor yang sudah ada | Applier tunduk pada governor load VPS 8 (jangan render thumbnail saat load1 > 7) + governor kuota baru (§5). |

### 3.4 Alur data end-to-end

```
1. Collector (01:00) → videos.list per channel (210 channel)
                        → Analytics per video (2.759 video)
                        → simpan snapshot ke video_audit
2. Classifier         → terapkan aturan §6 → kategori per video
                        → ringkasan per channel ke audit_channel_summary
3. Generator          → untuk kategori A: buat judul + thumbnail + deskripsi baru
                        → simpan sebagai video_repackage_jobs (status DRAFT)
4. Review (pengecualian) → owner/operator meninjau 10 DRAFT teratas
                        → (impresi terbesar) + 10 acak; sisanya auto-approve
                        → bila lolos validasi (Fase 1: review pengecualian;
                        → Fase 2+: otomatis bila pilot lolos)
5. Applier (06:00)    → ambil job APPROVED sesuai anggaran kuota hari itu
                        → videos.update + thumbnails.set (dengan backoff)
                        → catat snapshot T0 ke video_repackage_outcomes
6. Measurer (T7/T14)  → ukur ulang metrik → hitung delta → tandai hasil
                        → (T7 = sinyal utama; T14 = konfirmasi;
                        → vonis hanya bila ≥200 impresi baru sejak T0)
7. Reporter           → dashboard: berapa yang membaik / netral / memburuk
```

### 3.5 Keputusan implementasi yang masih terbuka

| # | Keputusan | Opsi | Rekomendasi awal |
|---|---|---|---|
| D-1 | Cara worker menulis DB | (a) API Master, (b) DB lokal + fleet sync | (a) API Master — sesuai prinsip P4; hindari divergensi seperti yang terjadi pada ingestion (Master 636 vs VPS 8 626) |
| D-2 | Review manual vs otomatis | (a) Selalu manual, (b) otomatis setelah pilot lolos | (b) — manual di Fase 1, otomatis di Fase 2 bila CTR pilot membaik |
| D-3 | Generator thumbnail | (a) Pipeline yang ada, (b) pipeline baru berbasis CTR | (a) dulu; (b) bila data CTR menunjukkan pola thumbnail yang jelas |
| D-4 | Generator judul | (a) Template dari best-performer armada, (b) LLM | (a) dulu — deterministik, murah, dan terbukti di armada sendiri |

---

## §4 — SKEMA TABEL

Empat tabel baru di DB Master (sumber kebenaran). Semua timestamp dalam
epoch detik UTC, konsisten dengan konvensi DB yang ada (`recorded_at`,
`updated_at` epoch).

### 4.1 `video_audit` — snapshot audit per video

```sql
CREATE TABLE IF NOT EXISTS video_audit (
    id                  INTEGER PRIMARY KEY AUTOINCREMENT,
    video_id            TEXT    NOT NULL,       -- ID video YouTube
    channel_id          TEXT    NOT NULL,       -- channel_id internal (STAGING-SLOT-xxx / real)
    slot_number         INTEGER,                -- NO armada (1-707), bila diketahui
    vault_id            TEXT,                   -- vault pemilik channel
    title               TEXT,                   -- judul saat audit
    published_at        INTEGER,                -- epoch publish
    views               INTEGER DEFAULT 0,      -- dari Analytics (lifetime)
    impressions         INTEGER DEFAULT 0,      -- dari Analytics (lifetime)
    ctr                 REAL    DEFAULT 0.0,    -- click-through rate (0.0-1.0)
    avg_view_duration   REAL    DEFAULT 0.0,    -- detik
    traffic_search      INTEGER DEFAULT 0,      -- view dari pencarian
    traffic_suggested   INTEGER DEFAULT 0,      -- view dari suggested
    traffic_browse      INTEGER DEFAULT 0,      -- view dari browse/home
    category            TEXT    NOT NULL,       -- A / B / C / D (lihat §6)
    category_reason     TEXT,                   -- alasan klasifikasi (1 baris)
    is_public           INTEGER DEFAULT 1,      -- 1 = publik, 0 = unlisted/private
    monetization_status TEXT,                   -- NOT_MONETIZED / MONETIZED / NOT_APPLICABLE (v1.1)
    is_gone             INTEGER DEFAULT 0,      -- 1 = video tidak ditemukan di API (v1.1)
    audited_at          INTEGER NOT NULL,       -- epoch audit
    UNIQUE(video_id, audited_at)
);
CREATE INDEX IF NOT EXISTS idx_audit_channel ON video_audit(channel_id);
CREATE INDEX IF NOT EXISTS idx_audit_category ON video_audit(category);
CREATE INDEX IF NOT EXISTS idx_audit_audited ON video_audit(audited_at);
```

**Catatan:** satu baris per video per audit (bukan upsert) — riwayat audit
tersimpan sehingga tren per video bisa dilacak. Retensi: snapshot lebih
dari 90 hari diarsipkan (bukan dihapus — data tren itu aset).

### 4.2 `video_repackage_jobs` — pekerjaan edit metadata

```sql
CREATE TABLE IF NOT EXISTS video_repackage_jobs (
    id                  INTEGER PRIMARY KEY AUTOINCREMENT,
    video_id            TEXT    NOT NULL,
    channel_id          TEXT    NOT NULL,
    vault_id            TEXT    NOT NULL,       -- vault yang membayar kuota
    old_title           TEXT,
    new_title           TEXT,
    old_description     TEXT,
    new_description     TEXT,
    old_thumbnail_url   TEXT,
    new_thumbnail_path  TEXT,                   -- path lokal thumbnail baru
    change_title        INTEGER DEFAULT 0,      -- 1 = judul diubah
    change_description  INTEGER DEFAULT 0,
    change_thumbnail    INTEGER DEFAULT 0,
    status              TEXT    NOT NULL DEFAULT 'DRAFT',
        -- DRAFT -> APPROVED -> APPLIED -> MEASURED
        -- DRAFT -> REJECTED | APPLIED -> FAILED
    quota_units_used    INTEGER DEFAULT 0,      -- unit yang benar-benar terpakai
    error_message       TEXT,
    created_at          INTEGER NOT NULL,
    approved_at         INTEGER,
    applied_at          INTEGER,
    approved_by         TEXT                    -- 'owner' / 'operator' / 'auto-pilot'
);
CREATE INDEX IF NOT EXISTS idx_repack_status ON video_repackage_jobs(status);
CREATE INDEX IF NOT EXISTS idx_repack_channel ON video_repackage_jobs(channel_id);
CREATE INDEX IF NOT EXISTS idx_repack_video ON video_repackage_jobs(video_id);
```

### 4.3 `video_repackage_outcomes` — hasil sebelum/sesudah

```sql
CREATE TABLE IF NOT EXISTS video_repackage_outcomes (
    id                  INTEGER PRIMARY KEY AUTOINCREMENT,
    job_id              INTEGER NOT NULL REFERENCES video_repackage_jobs(id),
    video_id            TEXT    NOT NULL,
    measured_at         INTEGER NOT NULL,       -- epoch pengukuran
    checkpoint          TEXT    NOT NULL,       -- T0 / T7 / T14
    views               INTEGER DEFAULT 0,
    impressions         INTEGER DEFAULT 0,
    ctr                 REAL    DEFAULT 0.0,
    avg_view_duration   REAL    DEFAULT 0.0,
    delta_ctr           REAL,                   -- vs T0 (NULL untuk T0)
    delta_views         INTEGER,                -- vs T0 (NULL untuk T0)
    verdict             TEXT,                   -- BERHASIL / NETRAL / GAGAL (diisi di T14)
    UNIQUE(job_id, checkpoint)
);
CREATE INDEX IF NOT EXISTS idx_outcome_job ON video_repackage_outcomes(job_id);
```

**Definisi verdict (di T14; T7 memakai definisi yang sama sebagai sinyal dini):**
- **BERHASIL:** delta CTR ≥ +0,01 (1 poin persen) ATAU delta views ≥ +50%
  dengan impresi tidak turun.
- **NETRAL:** perubahan dalam ±0,005 CTR dan ±10% views.
- **GAGAL:** delta CTR ≤ −0,01 ATAU views turun > 20%.
- **BELUM CUKUP DATA (v1.1):** impresi baru sejak T0 < 200 — vonis ditunda
  sampai ambang tercapai atau T14 tercapai (mana yang dulu). Vonis atas
  < 200 impresi tidak berarti secara statistik (noise).
- Estimasi CTR memakai perataan Bayesian untuk video berimpresi kecil:
  CTR = (klik + 10) / (impresi + 500).

### 4.4 `audit_channel_summary` — ringkasan per channel

```sql
CREATE TABLE IF NOT EXISTS audit_channel_summary (
    id                  INTEGER PRIMARY KEY AUTOINCREMENT,
    channel_id          TEXT    NOT NULL,
    slot_number         INTEGER,
    channel_name        TEXT,
    total_videos        INTEGER DEFAULT 0,
    count_a             INTEGER DEFAULT 0,      -- layak edit
    count_b             INTEGER DEFAULT 0,      -- masalah distribusi
    count_c             INTEGER DEFAULT 0,      -- masalah konten
    count_d             INTEGER DEFAULT 0,      -- janji vs isi
    total_views         INTEGER DEFAULT 0,
    avg_ctr             REAL    DEFAULT 0.0,
    recommendation      TEXT,                   -- EDIT / STRATEGI_LAIN / HENTIKAN / PANTAU
    recommendation_reason TEXT,
    audited_at          INTEGER NOT NULL,
    UNIQUE(channel_id, audited_at)
);
CREATE INDEX IF NOT EXISTS idx_channelsum_rec ON audit_channel_summary(recommendation);
```

**Nilai `recommendation`:**
- **EDIT** — punya video kategori A (≥3 video A) → masuk antrean repackaging.
- **STRATEGI_LAIN** — dominan kategori B (distribusi) → butuh strategi di
  luar metadata (playlist, promosi silang, niche ulang) — BUKAN edit.
- **HENTIKAN** — seluruh video 0 impresi + 0 view + umur > 90 hari →
  kandidat penghentian publish + realokasi kuota (keputusan owner).
- **PANTAU** — performa cukup baik → tidak ada tindakan, audit ulang berkala.

---

## §5 — PERHITUNGAN KUOTA

### 5.1 Inventaris (per 06:46 WIB, 24 Nov 2026 — terverifikasi)

| Item | Angka |
|---|---|
| Video terbit (total) | **2.759** |
| Channel dengan video terbit | **210** (dari 707 slot) |
| Rata-rata video per channel terbit | ~13 |
| Vault aktif | 37 |
| Kuota harian armada (37 × 10.000) | 370.000 unit/hari |
| Kuota harian yang dipakai publish (207 × 1.650) | ~341.550 unit/hari |
| **Sisa kuota teoritis armada** | **~28.450 unit/hari** |

> ⚠️ Sisa 28.450 unit/hari adalah angka TEORITIS (asumsi semua vault penuh
> dan seimbang). Praktiknya, distribusi vault tidak merata dan publish
> adalah prioritas — anggaran fitur ini harus jauh di bawah angka itu.

### 5.2 Biaya API (resmi YouTube Data API v3 + Analytics API)

| Operasi | Biaya | Keterangan |
|---|---|---|
| `videos.list` (baca metadata) | 1 unit / panggilan (maks 50 video) | Audit penuh: 2.759 video ≈ 56 panggilan ≈ **56 unit** |
| Analytics `reports.query` | ~1 unit / query | Per channel per set metrik: 210 channel ≈ **210–420 unit** per audit penuh |
| `videos.update` (judul/deskripsi/tag) | **50 unit** / video | Satu panggilan mengubah semua field sekaligus |
| `thumbnails.set` | **50 unit** / video | Terpisah dari videos.update |
| Pengukuran ulang (Measurer) | ~1 unit / video / checkpoint | 2 checkpoint (T7 + T14) |

### 5.3 Biaya skenario

| Skenario | Perhitungan | Total unit |
|---|---|---|
| **Audit penuh (baca saja)** | 56 + ~420 | **~500 unit** (sekali jalan — nyaris gratis) |
| Edit 1 video (judul+deskripsi) | 50 | 50 unit |
| Edit 1 video (judul+deskripsi+thumbnail) | 50 + 50 | 100 unit |
| Edit SEMUA 2.759 video (lengkap) | 2.759 × 100 | **275.900 unit** (≈ 10 hari seluruh sisa kuota armada — TIDAK REALISTIS) |
| Edit 10% video (276 video, asumsi kategori A) | 276 × 100 | 27.600 unit |
| Edit 20% video (552 video) | 552 × 100 | 55.200 unit |

### 5.4 Anggaran yang diusulkan

| Parameter | Nilai | Alasan |
|---|---|---|
| Anggaran harian fitur (total armada) | **5.000 unit/hari** (Fase 1–2) | ≈ 50 video lengkap/hari; ~18% dari sisa teoritis; tidak mengganggu publish |
| Anggaran baca audit (bertingkat, v1.1) | **~2.000–3.000 unit/bulan** | Harian inkremental ~50–100 unit + mingguan bergilir + bulanan penuh ~500 unit; turun ~80% dari audit harian penuh (~15.000/bulan) |
| Anggaran harian (bila pilot berhasil) | **10.000 unit/hari** (Fase 3) | ≈ 100 video lengkap/hari |
| Cap per vault per hari | **500 unit** | Satu vault tidak boleh kehabisan kuota karena fitur ini; publish di vault itu tetap jalan |
| Prioritas antrean | Impresi tertinggi + CTR terendah dulu | Video dengan impresi besar + CTR buruk = potensi kenaikan terbesar |
| Penghentian darurat | Berhenti bila kuota publish hari itu < 10% sisa | Publish selalu menang atas repackaging |

### 5.5 Waktu tempuh (estimasi)

| Asumsi kategori A | Jumlah video | Unit (lengkap) | Hari @ 5.000/hari | Hari @ 10.000/hari |
|---|---|---|---|---|
| 5% dari 2.759 | 138 | 13.800 | ~3 hari | ~2 hari |
| 10% dari 2.759 | 276 | 27.600 | ~6 hari | ~3 hari |
| 20% dari 2.759 | 552 | 55.200 | ~11 hari | ~6 hari |
| 30% dari 2.759 | 828 | 82.800 | ~17 hari | ~9 hari |

> Persentase kategori A yang sebenarnya HANYA diketahui setelah audit
> Fase 0. Angka di atas adalah skenario, bukan janji.

### 5.6 Governor kuota (aturan yang diimplementasikan di Applier)

1. Sebelum mengirim `videos.update`/`thumbnails.set`, Applier membaca
   `vault_quota_ledger` untuk vault pemilik video: bila pemakaian hari itu
   + biaya edit > (10.000 − cadangan publish 1.650), SKIP video itu
   (antre ke hari berikutnya).
2. Bila anggaran harian fitur (5.000/10.000) habis, Applier berhenti
   sampai reset kuota (15:00 WIB).
3. Setiap unit dicatat ke `vault_quota_ledger` dengan `operation_type`
   `AUDIT_UPDATE` / `AUDIT_THUMBNAIL` (bukan dicampur dengan publish).
4. Dashboard menampilkan pemakaian kuota fitur ini secara terpisah.

---

## §6 — KLASIFIKASI & ATURAN KEPUTUSAN

### 6.1 Kategori per video (diterapkan Classifier)

| Kategori | Kondisi (semua harus terpenuhi) | Diagnosis | Tindakan |
|---|---|---|---|
| **A** | Impresi ≥ 100 DAN CTR < 0,02 DAN umur ≥ 14 hari | Masalah kemasan — YouTube sudah menampilkan, penonton tidak mengklik | **EDIT judul + thumbnail** (+ deskripsi bila ada kesalahan faktual) |
| **B** | Impresi < 10 DAN views = 0 DAN umur ≥ 30 hari | Masalah distribusi — YouTube tidak/belum menguji video ini | TIDAK DIEDIT. Rekomendasi channel: strategi lain atau hentikan |
| **C** | Views ≥ 10 DAN retensi < 30% durasi | Masalah konten — diklik lalu ditinggal | TIDAK DIEDIT (metadata tidak memperbaiki konten). Catat untuk evaluasi konten |
| **D** | CTR ≥ 0,02 DAN views ≥ 10 DAN retensi < 30% | Janji ≠ isi — judul/thumbnail menarik klik tapi isi mengecewakan | EDIT judul/thumbnail ke arah yang LEBIH JUJUR (kurangi clickbait), bukan lebih agresif |
| **—** (tidak terkategori) | Tidak memenuhi kondisi di atas | Belum cukup data / performa cukup baik | Tidak ada tindakan; audit ulang periode berikutnya |

**Catatan ambang:** angka 100 impresi / 2% CTR / 30% retensi adalah nilai
awal yang wajar untuk channel musik kompilasi. Setelah data pilot terkumpul,
ambang ini dikalibrasi dari distribusi CTR armada sendiri (mis. CTR median
armada sebagai garis batas, bukan 2% tetap).

**Dimensi monetisasi (v1.1):** `monetization_status` dicatat per video
(§4.1) dan dipakai dalam prioritas (§6.3) — bukan dalam kategori A/B/C/D
itu sendiri. Kategori ditentukan murni oleh sinyal performa.

### 6.2 Pemeriksaan level channel (dijalankan Collector per channel)

Sebelum video diedit, periksa kondisi channel — bila ada yang salah,
perbaiki DULU (ini sering luput dan tidak bisa diperbaiki metadata):

| # | Pemeriksaan | Cara cek | Bila bermasalah |
|---|---|---|---|
| C-1 | Video benar-benar publik? | `videos.list` → `status.privacyStatus` | Bila unlisted/private tanpa sengaja → kembalikan ke publik (50 unit/video) |
| C-2 | Channel terindeks pencarian? | Cari nama channel di YouTube | Bila tidak ditemukan → masalah visibilitas channel (di luar fitur ini — laporkan) |
| C-3 | Blokir YPP / monetisasi? | Status channel di DB + dashboard | Bila diblokir → tandai, jangan buang kuota edit |
| C-4 | Thumbnail default/rusak? | Cek URL thumbnail | Bila thumbnail hilang → regenerasi (masuk kategori A) |
| C-5 | Deskripsi kosong / salah? | Cek panjang + kecocokan tracklist | Bila kosong/salah → perbaiki dari bahan (tanpa mengubah judul) |

### 6.3 Urutan prioritas antrean edit (v1.1 — berdasarkan potensi kenaikan)

1. Kategori A, diurutkan berdasarkan **skor potensi kenaikan**:
   `skor = impresi × (0,05 − CTR)` (target CTR 5% sebagai acuan) —
   video dengan impresi besar + CTR jauh di bawah target didahulukan.
   Peringkat per channel memakai jumlah skor video A-nya (bukan view
   terendah — channel view 0 hampir selalu kategori B dan tidak layak edit).
2. Kategori D (setelah A habis).
3. C-4/C-5 (thumbnail hilang / deskripsi kosong) — perbaikan kebersihan,
   biaya kecil, dilakukan bersamaan bila kuota tersisa.
4. **Dimensi monetisasi (v1.1):** channel termonetisasi didahulukan dalam
   skor yang sama (ROI pendapatan langsung); channel belum monetisasi
   diukur dari jam tayang + subscriber menuju YPP, bukan view semata.
   (91,6% video terbit saat ini NOT_MONETIZED — 2.527 dari 2.760.)

---

## §7 — GENERATOR METADATA

### 7.1 Judul baru

- **Sumber pola:** 20 video CTR tertinggi di armada sendiri per genre
  (diambil dari `video_audit`). Pola yang terbukti bekerja di armada sendiri
  lebih berharga dari template generik.
- **Aturan:**
  - Panjang ≤ 100 karakter (batas YouTube); target 50–70 karakter.
  - Pertahankan kata kunci genre + mood yang sudah ada (jangan ubah total —
    penonton yang sudah mengenal format channel tidak boleh bingung).
  - Tambahkan pembeda spesifik (mis. nama lagu utama, suasana, konteks
    waktu) — bukan superlatif kosong ("TERBAIK", "PALING ENAK").
  - DILARANG clickbait yang tidak sesuai isi (itu menciptakan kategori D).
  - DILARANG mengubah judul video yang CTR-nya sudah ≥ 5%.
- **Validasi sebelum kirim:** judul baru ≠ judul lama; tidak mengandung kata
  yang dilarang YouTube; tidak duplikat dengan judul video lain di channel
  yang sama.
- **Hierarki fallback (v1.1):** bila genre tidak punya 20 video CTR terbaik
  (genre kecil), pakai pola terbaik lintas genre → bila itu pun tidak ada,
  pakai template manual. Fallback yang dipakai dicatat per video
  (tambahkan kolom `generator_source` ke `video_repackage_jobs` saat
  implementasi).

### 7.2 Thumbnail baru

- **Pipeline:** memakai ulang generator thumbnail yang sudah ada
  (ingestion/LoopGen). Bila data CTR menunjukkan pola visual yang jelas
  (mis. thumbnail dengan teks besar perform lebih baik), pola itu
  dijadikan parameter generator.
- **Aturan:**
  - Resolusi 1280×720 (16:9), ukuran < 2 MB.
  - Kontras tinggi, teks terbaca di ukuran kecil (uji di 120px).
  - Konsisten dengan identitas visual channel (jangan ubah total gaya).
  - DILARANG memakai gambar yang menyesatkan isi video.
- **Varian:** untuk video impresi tertinggi, buat 2 varian thumbnail dan
  uji berurutan (varian 1 selama 7 hari → bila CTR tidak naik, varian 2).

### 7.3 Deskripsi baru

- **Sumber kebenaran isi:** bahan GDrive (`LINK_GDRIVE_BAHAN`) — daftar lagu
  yang sebenarnya ada di video.
- **Isi deskripsi baru:**
  1. Ringkasan 1–2 kalimat (kata kunci genre + mood).
  2. Daftar lagu lengkap + timestamp (chapter) — membantu pencarian dan UX.
  3. Tautan channel/playlist terkait (internal).
- **Tag:** isi dengan 5–10 tag yang relevan (genre, mood, nama artis/lagu)
  — usaha minimal, karena dampaknya kecil. JANGAN menghabiskan waktu
  optimasi tag.

### 7.4 Yang TIDAK diubah

- Video itu sendiri (tidak ada re-render / re-upload).
- Tanggal publish, kategori video, bahasa.
- Judul/deskripsi video yang performanya baik (CTR ≥ 5% atau views tinggi).
- Metadata channel (nama, deskripsi channel) — di luar cakupan fitur ini.

---

## §8 — TAHAPAN IMPLEMENTASI

| Fase | Pekerjaan | Durasi | Keluaran |
|---|---|---|---|
| **Fase 0 — Audit saja** | Bangun Collector + Classifier + 4 tabel + dashboard baca-saja. TIDAK ADA edit. | 1 minggu | Laporan triase 210 channel: jumlah A/B/C/D per channel + rekomendasi per channel |
| **Fase 1 — Pilot** | Bangun Generator + Applier + Measurer. Pilot pada 5–10 channel (pilih berdasarkan skor potensi §6.3). Review berdasarkan pengecualian (10 teratas + 10 acak; sisanya auto-approve bila lolos validasi). | 2 minggu (1 minggu edit + 1 minggu ukur T7) | Hasil T7 pilot: berapa % edit yang BERHASIL (T7 = sinyal utama) |
| **Fase 2 — Keputusan skala** | Evaluasi hasil T7 (utama) + T14 (konfirmasi) pilot. Bila ≥ 20% edit BERHASIL → lanjut ke skala penuh dengan review otomatis. Bila < 20% → hentikan atau revisi generator. Kalibrasi generator mingguan dari data T7. | 1 minggu | Keputusan GO / NO-GO berbasis data |
| **Fase 3 — Skala penuh** | Terapkan ke semua channel rekomendasi EDIT, dengan anggaran 10.000 unit/hari + governor. Audit ulang bulanan. | Berkelanjutan | Antrean edit berjalan + laporan hasil bulanan |
| **Fase 4 — Kontinu** | Audit berkala (bulanan) + repackaging video baru yang masuk kategori A + kalibrasi ambang dari data armada. | Berkelanjutan | Fitur menjadi bagian permanen dari operasi |

---

## §9 — KRITERIA PENERIMAAN

### Fase 0 (audit)
- [ ] Collector berhasil mengaudit ≥ 95% dari 2.759 video (sisanya dicatat
  dengan alasan: video dihapus, token kedaluwarsa, dsb.).
- [ ] Setiap video terkategori A/B/C/D/tidak-terkategori dengan alasan.
- [ ] Ringkasan per channel terbit untuk 210 channel (termasuk rekomendasi
  EDIT / STRATEGI_LAIN / HENTIKAN / PANTAU).
- [ ] Biaya audit penuh ≤ 1.000 unit (verifikasi dari `vault_quota_ledger`).
- [ ] Dashboard audit bisa dibuka owner dan menampilkan angka yang sama
  dengan DB (verifikasi silang 10 video acak).

### Fase 1 (pilot)
- [ ] ≥ 50 video kategori A diedit (judul + thumbnail) pada 5–10 channel.
- [ ] 0 video kategori B/C atau video ber-CTR baik yang teredit (verifikasi
  dari `video_repackage_jobs` vs `video_audit`).
- [ ] 0 pelanggaran anggaran kuota (verifikasi dari ledger per vault).
- [ ] Snapshot T0 tercatat untuk 100% video yang diedit.
- [ ] Hasil T7 terukur untuk 100% video yang diedit (vonis hanya bila
  ≥200 impresi baru; bila tidak → BELUM CUKUP DATA).
- [ ] Review pengecualian berjalan (10 teratas + 10 acak ditinjau manual;
  sisanya auto-approve tercatat).

### Fase 2 (keputusan skala)
- [ ] Hasil T7 (utama) + T14 (konfirmasi) pilot: persentase BERHASIL /
  NETRAL / GAGAL / BELUM CUKUP DATA dilaporkan dengan angka.
- [ ] GO bila ≥ 20% BERHASIL dan 0% GAGAL dengan penurunan views > 20%.
  NO-GO bila < 20% BERHASIL → generator direvisi atau fitur dihentikan.
  (Angka 20% adalah ambang awal — owner boleh mengubah sebelum pilot.)

### Fase 3 (skala penuh)
- [ ] Antrean edit berjalan sesuai prioritas §6.3.
- [ ] Tidak ada hari di mana kuota publish terganggu oleh fitur ini
  (verifikasi: PUBLISHED harian tidak turun di bawah baseline 206
  karena kuota — bila turun, governor terbukti gagal).
- [ ] Laporan hasil bulanan: jumlah edit + % BERHASIL/NETRAL/GAGAL.

---

## §10 — RISIKO & MITIGASI

| # | Risiko | Probabilitas | Dampak | Mitigasi |
|---|---|---|---|---|
| R-1 | Fitur menghabiskan kuota publish | Sedang | Tinggi (publish berhenti) | Governor per-vault + cadangan publish 1.650 unit + penghentian darurat (§5.6) |
| R-2 | Edit merusak video yang bekerja | Rendah (bila aturan dipatuhi) | Sedang | Larangan edit video CTR ≥ 5%; review manual di Fase 1 |
| R-3 | Hasil tidak terukur / klaim sepihak | Sedang | Sedang | Loop ukur T0/T7/T14 wajib; verdict dari data, bukan dari perasaan |
| R-4 | Token vault kedaluwarsa saat edit massal | Sedang | Sedang | Applier memeriksa token sebelum batch; job gagal-token ditandai (bukan diulang tanpa henti) |
| R-5 | YouTube rate-limit / error API | Rendah | Rendah | Backoff eksponensial; jeda antar-panggilan; antrean persisten (tidak hilang saat restart) |
| R-6 | Ekspektasi berlebihan ("semua video bangkit") | Tinggi | Sedang | Dokumen ini + laporan triase menyatakan ekspektasi nyata: hanya kategori A yang punya peluang; target 20% BERHASIL |
| R-7 | Thumbnail baru tidak lebih baik dari lama | Sedang | Rendah | Varian ganda untuk video impresi tinggi; ukur T7; kembalikan thumbnail lama bila CTR turun (rollback manual) |
| R-8 | Worker VPS 8 bersaing sumber daya dengan live/prep | Rendah | Sedang | Jadwal di luar jam prime; tunduk pada governor load (jeda bila load1 > 7) |
| R-9 | Noise statistik pada video berimpresi kecil | Tinggi | Rendah | Vonis hanya bila ≥200 impresi baru; perataan Bayesian; agregasi per channel/genre |
| R-10 | Token vault gagal saat batch edit | Sedang | Sedang | Pre-flight check token per vault; skip vault bermasalah; antrean per vault |
| R-11 | Batas atas repackaging (konten duplikatif antar-channel) | Sedang | Sedang | Audit ukur tumpang tindih konten; laporkan batas ini secara jujur; diferensiasi jangka panjang di pembuatan, bukan di repackaging |

---

## §11 — KEPUTUSAN YANG DIBUTUHKAN OWNER

| # | Keputusan | Opsi | Dampak bila tidak diputuskan |
|---|---|---|---|
| K-1 | Anggaran kuota harian fitur | 5.000 (aman) / 10.000 (agresif) / angka lain | Tidak bisa mulai Fase 1 |
| K-2 | Ambang GO/NO-GO pilot | 20% BERHASIL (usulan) / angka lain | Tidak ada kriteria objektif untuk skala |
| K-3 | Review manual vs otomatis di Fase 3 | Manual terus / otomatis setelah pilot lolos | Menentukan beban kerja operator |
| K-4 | Nasib channel rekomendasi HENTIKAN | Hentikan publish + realokasi kuota / biarkan jalan | Channel mati terus menghabiskan kuota publish |
| K-5 | Gaya thumbnail baru | Ikuti gaya channel saat ini / gaya baru berbasis CTR | Menentukan parameter generator |
| K-6 | Siapa yang membangun | Antigravity / Muse / dibagi | Menentukan penjadwalan kerja |
| K-7 | Diagnosis 497 channel belum publish (v1.1) | Lakukan bersamaan / tunda / tidak | 70% armada diam tanpa penjelasan bila tidak didiagnosis |

---

## §12 — LAMPIRAN

### A. Biaya API ringkas

| Operasi | Unit |
|---|---|
| `videos.list` (50 video) | 1 |
| Analytics `reports.query` | ~1 |
| `videos.update` | 50 |
| `thumbnails.set` | 50 |
| `videos.insert` (publish, pembanding) | 1.650 |

### B. File & tabel yang dipakai ulang

| Aset | Lokasi / nama |
|---|---|
| Token OAuth per vault | `auth_tokens` (kolom `token_status`) |
| Ledger kuota | `vault_quota_ledger` (kolom `recorded_at` epoch) |
| Daftar video terbit | `operational_tasks` (status `PUBLISHED`) |
| Info channel + vault | `channels` (`master_vault_id`, `operator_name`) |
| Generator thumbnail | Pipeline ingestion/LoopGen yang sudah ada |
| Bahan (tracklist) | `LINK_GDRIVE_BAHAN` per channel (Drive) |
| Governor load | `prep_worker.py` (pola ECO/TURBO/BALANCED) |

### C. Perintah verifikasi (untuk owner / agen verifikasi)

```sql
-- Jumlah video per kategori (setelah audit Fase 0)
SELECT category, COUNT(*) FROM video_audit
WHERE audited_at = (SELECT MAX(audited_at) FROM video_audit)
GROUP BY category;

-- Hasil edit (setelah T14)
SELECT verdict, COUNT(*) FROM video_repackage_outcomes
WHERE checkpoint = 'T14' GROUP BY verdict;

-- Kuota yang dipakai fitur ini (per hari)
SELECT date(recorded_at, 'unixepoch', '+7 hours') AS hari,
       SUM(units_consumed) AS unit_audit
FROM vault_quota_ledger
WHERE operation_type IN ('AUDIT_UPDATE', 'AUDIT_THUMBNAIL')
GROUP BY hari ORDER BY hari DESC LIMIT 7;
```

### D. Aturan yang mengikat pekerjaan ini

- Aplikasi SOLID — setiap perubahan butuh rencana verifikasi pra/pasca +
  rollback + jendela observasi.
- Baseline adalah LANTAI: 206 PUBLISHED/hari, 0 FAILED/BLOCKED.
- Publish selalu menang atas fitur ini dalam perebutan kuota.
- Jangan pernah mengklaim hasil tanpa pengukuran sebelum/sesudah.
- Maksimal 1 restart layanan per 15 menit lintas semua agen.

---

## §13 — BOTTLENECK TAMBAHAN & SOLUSI (Revisi v1.1)

Bab ini ditambahkan 06:50 WIB, 24 Nov 2026, setelah analisa bottleneck
lanjutan atas desain v1.0. Setiap bottleneck dicantumkan buktinya dan
solusinya, beserta bab yang diubah sebagai akibatnya.

### B1. Discovery video ID — BUKAN bottleneck (kabar baik)
- **Temuan:** `operational_tasks.youtube_video_id` terisi 100%
  (2.760/2.760 video terbit, per 06:48 WIB). Collector mengambil ID
  langsung dari DB — tidak butuh `search.list` (100 unit/panggilan).
- **Tindak lanjut:** tetap tangani video yang sudah mati (B10).

### B2. Audit harian penuh = pemborosan + data basi
- **Bukti:** audit penuh tiap hari ≈ 500 unit/hari (15.000/bulan) untuk
  membaca angka yang sebagian besar sama — data Analytics tertinggal
  24–48 jam.
- **Solusi:** audit bertingkat — harian inkremental (~50–100 unit),
  mingguan bergilir (¼ katalog), bulanan penuh (~500 unit).
  Total ~2.000–3.000 unit/bulan (turun ~80%).
- **Mengubah:** §3.1, §3.2, §5.4.

### B3. Targeting "mini performa" yang naif membuang kuota
- **Bukti:** channel view 0 hampir selalu kategori B (0 impresi) —
  metadata tidak bisa membantu mereka.
- **Solusi:** target berdasarkan skor potensi kenaikan
  (`impresi × (0,05 − CTR)`), bukan view terendah.
- **Mengubah:** §6.3, §8 (pemilihan channel pilot).

### B4. 91,6% video NOT_MONETIZED
- **Bukti:** 2.527 dari 2.760 video terbit berstatus NOT_MONETIZED
  (per 06:48 WIB). Menaikkan view mereka tidak menghasilkan pendapatan
  langsung — nilainya menuju syarat YPP.
- **Solusi:** `monetization_status` menjadi dimensi klasifikasi;
  channel termonetisasi didahulukan; channel belum monetisasi diukur
  dari jam tayang + subscriber.
- **Mengubah:** §4.1, §6.1, §6.3.

### B5. 497 channel belum pernah publish
- **Bukti:** hanya 210 dari 707 channel punya video terbit.
- **Solusi:** dashboard audit menampilkan kesenjangan ini eksplisit;
  diagnosis terpisah untuk 497 channel (tanpa bahan? tanpa vault?
  staging? terblokir?) — di luar cakupan fitur ini (keputusan K-7).
- **Mengubah:** §11 (K-7 baru).

### B6. Noise statistik pada channel kecil
- **Bukti:** video 50 impresi + 1 klik = CTR 2% dengan selang
  kepercayaan sangat lebar — vonis atas angka itu tidak berarti.
- **Solusi:** vonis hanya bila ≥200 impresi baru sejak T0
  (bila tidak → BELUM CUKUP DATA); perataan Bayesian;
  agregasi vonis per channel/genre.
- **Mengubah:** §4.3, §9.

### B7. Feedback loop 14 hari terlalu lambat
- **Bukti:** dengan 50 edit/hari, setelah 2 minggu ada 700 video
  dalam penerbangan tanpa vonis — generator berjalan buta.
- **Solusi:** T7 sebagai sinyal utama (CTR stabil 3–7 hari);
  T14 hanya konfirmasi; kalibrasi generator mingguan dari data T7;
  jeda pola yang sama bila batch pertama T7-nya negatif.
- **Mengubah:** §3.4, §8, §9.

### B8. Review manual = bottleneck manusia
- **Bukti:** 50 DRAFT/hari untuk ditinjau = operator lelah, review
  menjadi stempel karet.
- **Solusi:** review berdasarkan pengecualian (10 teratas + 10 acak;
  sisanya auto-approve bila lolos validasi) + UI approve massal.
- **Mengubah:** §3.4, §8, §9.

### B9. Token vault rapuh + distribusi tidak merata
- **Bukti:** 210 channel tersebar tidak merata di 37 vault; insiden
  69 kegagalan dekripsi OAuth (23 Nov) menunjukkan token bisa gagal massal.
- **Solusi:** pre-flight check token per vault sebelum batch; skip
  vault bermasalah (bukan retry tanpa henti); antrean edit per vault;
  kesehatan vault tampil di dashboard audit.
- **Mengubah:** §3.2 (Applier), §10 (R-10).

### B10. Video yang sudah mati
- **Bukti:** sebagian dari 2.760 ID mungkin sudah tidak ada
  (dihapus/di-privat/dihapus YouTube).
- **Solusi:** `videos.list` kosong → tandai GONE (`is_gone = 1`),
  verifikasi sekali lagi setelah 7 hari, lalu keluarkan permanen
  dari audit.
- **Mengubah:** §4.1.

### B11. Jendela edit vs reset kuota
- **Bukti:** kuota reset 15:00 WIB; jendela 06:00–10:00 adalah tengah
  hari-kuota (publish sudah menghabiskan sejak kemarin sore).
- **Solusi:** jendela utama 15:00–17:00 (kuota baru reset, sebelum jam
  prime); 06:00–10:00 menjadi cadangan. Governor real-time tetap
  menjadi pengaman utama.
- **Mengubah:** §3.1.

### B12. Diferensiasi konten adalah batas atas repackaging
- **Bukti:** bila 210 channel menerbitkan kompilasi nyaris identik,
  YouTube tidak akan mendistribusikan 210 salinan hal yang sama —
  sebaik apa pun judulnya.
- **Solusi:** audit mengukur tumpang tindih konten (tracklist yang sama
  di banyak channel) dan menandai channel duplikatif; tuas jangka
  panjang adalah diferensiasi saat pembuatan, bukan saat repackaging.
  Fitur ini melaporkan batas ini secara jujur.
- **Mengubah:** §10 (R-11); metrik tumpang tindih ditambahkan ke
  `audit_channel_summary` saat implementasi (kolom baru bila diperlukan).

---

*Dokumen ini disusun dari data terverifikasi (2.760 video / 210 channel,
per 06:48 WIB, 24 Nov 2026) dan biaya API resmi YouTube.
Semua angka kuota dapat diverifikasi ulang dari `vault_quota_ledger`.
Revisi v1.1 (06:50 WIB): 12 bottleneck tambahan + solusi, audit bertingkat,
targeting potensi kenaikan, dimensi monetisasi, T7 sebagai sinyal utama.*
