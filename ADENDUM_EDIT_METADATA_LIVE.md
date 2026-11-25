# ADENDUM — Edit Judul, Deskripsi & Thumbnail Live (In-Flight SEO)

| | |
|---|---|
| **Tanggal** | 25 Nov 2026 07:35 WIB |
| **Status** | DISETUJUI UNTUK DIIMPLEMENTASIKAN (perintah user: "harus dapat di edit dan disimpan") — scope tambahan implementasi jadwal editable |
| **Induk** | `DESAIN_JADWAL_LIVE_EDITABLE.md` (Fase 1–5) + `PANDUAN_GUI_MODERN_MASJAVAS.md` |

---

## 1. Temuan kunci: editornya SUDAH ADA — tapi ada 3 celah kepercayaan

Verifikasi kode 25 Nov 2026 (bukan dugaan):

| Komponen | Keadaan |
|---|---|
| Modal edit metadata | ADA — `modalLiveMetadataEditor` di `live_dashboard.html`, tombol per baris (line 564) + tombol di player monitor (2288) |
| Endpoint | ADA — `GET /api/live/metadata/{task_id}` (2944) + `POST /api/live/metadata/{task_id}` (3065), form: title, description, tags, category_id, broadcast_id, youtube_live_url, thumbnail (UploadFile) |
| Update YouTube | ADA — `update_live_broadcast_metadata` (`live_broadcaster.py:469`): `videos().update` + `thumbnails().set`, judul maks 100 karakter, deskripsi maks 5000, kategori default "10", pre-flight cek kuota |
| Skema | ADA — `live_stream_tasks` punya `title` (354), `thumbnail_path` (370), `description` (407) |
| Upload thumbnail | ADA — disimpan ke `data/live_thumbnails/thumb_{task_id}_{ts}.{ext}` |

**Tiga celah yang membuat fitur ini terasa "tidak bisa diedit/disimpan":**

### Celah 1 — Pesan sukses PALSU saat kuota habis (paling merusak kepercayaan)
`update_live_broadcast_metadata` punya jalur defer: bila kuota habis, ia mengembalikan `quota_blocked: True` + pesan "Metadata disimpan di database lokal. Sinkronisasi ke YouTube Studio ditunda". **Tapi endpoint mengabaikan flag itu** — selalu mengembalikan `success: True` + pesan "Metadata & SEO siaran langsung berhasil diperbarui di YouTube!". Operator mengira berhasil; YouTube tidak berubah.

### Celah 2 — Sync tertunda TIDAK PERNAH dicoba lagi
Grep seluruh `core/`, `worker/`, `api/` — **tidak ada retry worker, antrean, atau indikator** untuk sync yang di-defer. Edit yang tertunda kuota duduk di lokal selamanya sampai seseorang menyimpan ulang secara manual. Tidak ada badge "tertunda" di baris task.

### Celah 3 — Deskripsi & tag TIDAK disimpan di database lokal
UPDATE lokal di endpoint (3139–3146) hanya menyimpan `title`, `broadcast_id`, `youtube_live_url`, `thumbnail_path`, `updated_at`. **Kolom `description` (yang sudah ada di skema) tidak pernah ditulis; tag tidak punya kolom sama sekali.** Akibat: edit deskripsi yang ter-defer kuota HILANG TOTAL (tidak di DB, tidak di YouTube) — sambil menampilkan pesan sukses (Celah 1).

### Celah pendukung
- **Thumbnail tanpa validasi:** hanya cek ekstensi (`.jpg/.jpeg/.png/.webp`); tanpa cek dimensi (YouTube: 1280×720), ukuran (<2 MB), atau isi file — di server maupun di modal (modal hanya menampilkan thumbnail lama, tanpa pratinjau file baru). Thumbnail tidak valid gagal di YouTube dengan 500 generik.
- **Sulit ditemukan:** tombol pemicu berlabel jargon "In-Flight SEO", tombol kecil amber di antara 5–8 tombol per baris.
- **Dua permukaan edit bertabrakan:** modal metadata mengedit judul; modal edit Fase 1 (`_RECONNECTING_ALLOWED_FIELDS = ("title", "priority")`) juga mengedit judul — tanpa satu tempat yang jelas.
- **Tanpa riwayat:** tidak ada audit/rollback untuk metadata (berbeda dengan revisi jadwal Fase 1).
- **Tanpa verifikasi pasca-simpan:** tidak ada pembacaan ulang snippet untuk membuktikan YouTube benar-benar berubah.

---

## 2. Saran perbaikan

### P0 — Perbaiki kepercayaan (wajib, bagian dari implementasi ini)

| # | Saran | Implementasi |
|---|---|---|
| S1 | **Status jujur di setiap simpan** | Endpoint meneruskan `quota_blocked` ke respons; toast + badge baris: "✓ Tersinkron ke YouTube 07:35" / "⏳ Tersimpan lokal — sinkron ke YouTube tertunda (kuota), dicoba lagi otomatis". Jangan pernah klaim "berhasil di YouTube" bila belum. |
| S2 | **Antrean retry sync otomatis** | Tabel baru `live_metadata_sync_queue` (task_id, payload JSON, attempts, next_retry_at, last_error, status). Worker ringan (di supervisor Master, reuse pola `live_task_orders`): coba lagi tiap 15 menit / saat kuota reset; maks 10 percobaan lalu tandai FAILED + beri tahu operator. Badge "tertunda" hilang saat sukses. |
| S3 | **Simpan SEMUA field secara lokal** | UPDATE lokal menulis `description` + kolom baru `tags` (TEXT, CSV). DB lokal menjadi sumber kebenaran; tidak ada lagi edit yang hilang saat defer. GET memakai DB sebagai dasar, YouTube sebagai konfirmasi. |
| S4 | **Validasi thumbnail ujung-ke-ujung** | Klien: pratinjau file baru + cek dimensi/ukuran sebelum kirim. Server: cek ulang dengan Pillow (1280×720 dianjurkan, maks 2 MB, JPG/PNG); pesan galat spesifik ("Thumbnail harus 1280×720 — file ini 1920×1080"). |
| S5 | **Satukan permukaan edit** | Wizard edit Fase 1 mendapat langkah "Judul & SEO" yang MEMAKAI endpoint metadata (satu tempat, bukan dua modal). Tombol per baris diganti labelnya jadi "Edit Judul & Thumbnail" (hapus jargon "In-Flight SEO") sebagai shortcut ke langkah itu. Modal lama dipertahankan sebagai shortcut, bukan permukaan terpisah. |

### P1 — Peningkatan (setelah P0 terbukti)

| # | Saran | Nilai |
|---|---|---|
| S6 | **Riwayat metadata + rollback** | Pakai ulang `live_task_revisions` (Fase 1): tiap simpan metadata = 1 revisi; "kembalikan judul kemarin" 1 klik. |
| S7 | **Verifikasi pasca-simpan** | Setelah update sukses, baca ulang snippet dari YouTube; tampilkan "✓ Terverifikasi di YouTube 07:35" atau peringatan bila tidak cocok. |
| S8 | **Template judul/deskripsi per genre** | Variabel `{channel}`, `{tanggal}`, `{episode}` + penghitung karakter (judul 100, deskripsi 5000) + petunjuk SEO (kata kunci di 60 karakter pertama judul). |
| S9 | **Pratinjau kartu hasil pencarian** | Tampilkan thumbnail + judul seperti kartu YouTube sebelum simpan — operator melihat hasil akhirnya, bukan form mentah. |
| S10 | **Edit massal metadata** | OWNER: awalan/akhiran judul atau ganti thumbnail untuk banyak channel sekaligus (pakai mesin bulk-edit Fase 5). |
| S11 | **Perubahan metadata terjadwal** | Ganti judul/thumbnail otomatis di jam prime (pakai mesin perubahan terjadwal Fase 5). |
| S12 | **Uji A/B thumbnail** | Simpan 2 thumbnail, tukar terjadwal, bandingkan CTR lewat kolektor metrik (butuh metrik CTR — masa depan). |

---

## 3. Catatan kuota & biaya

- 1× simpan metadata = `videos.update` (50 unit) + `thumbnails.set` bila ada thumbnail baru (50 unit). Pre-flight guard yang sudah ada (`check_live_quota_budget`, margin 100) dipertahankan.
- Retry (S2) TIDAK boleh membakar kuota: cek guard dulu tiap percobaan; bila masih terblokir, tunda ke siklus berikutnya tanpa memanggil API.
- Verifikasi pasca-simpan (S7) memakai `videos.list` (1 unit) — murah.

---

## 4. Kriteria terima (P0)

- [ ] Simpan saat kuota habis → respons + UI berkata "tersimpan lokal, tertunda" (BUKAN "berhasil di YouTube"); badge ⏳ muncul di baris.
- [ ] Edit yang tertunda tersinkron otomatis saat kuota pulih (tes: simulasi guard terblokir → buka → retry sukses, badge hilang).
- [ ] Deskripsi + tag tersimpan di DB lokal (cek langsung `SELECT description, tags`).
- [ ] Thumbnail 1920×1080 / >2 MB ditolak dengan pesan spesifik, sebelum memanggil YouTube.
- [ ] Wizard edit punya langkah "Judul & SEO" yang berfungsi penuh; tombol lama jadi shortcut ke sana.
- [ ] Tes: endpoint (sukses, defer, retry, validasi thumbnail), worker retry, regresi editor lama.
- [ ] Semua UI baru mengikuti `PANDUAN_GUI_MODERN_MASJAVAS.md`.
