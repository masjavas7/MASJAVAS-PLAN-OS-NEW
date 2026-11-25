# Diagnosis: MONA Gagal Live Manual + Opsi Audio via Link GDrive

| | |
|---|---|
| **Tanggal** | 25 Nov 2026 08:05 WIB |
| **Status** | Diagnosis selesai (berbasis bukti) — perbaikan diteruskan ke implementasi |
| **Pelapor** | User (MONA operator) |

---

## 1. Diagnosis — apa yang sebenarnya terjadi (bukti, bukan dugaan)

### Rantai bukti (semua terverifikasi)

| # | Bukti | Sumber |
|---|---|---|
| 1 | MONA login 07:51:08 WIB | `audit_logs` id 8566 |
| 2 | 07:51:49 + 07:52:14 — 2× `POST /api/live/channel/auto-key` 200 OK dari browsernya (IP 103.214.112.181) | journal uvicorn |
| 3 | Setelah itu: **TIDAK ADA** `POST /api/live/schedule` (buat task) maupun `POST /api/live/start` — form tidak pernah ter-submit | journal uvicorn 07:45–08:05 |
| 4 | 8 dari 13 channel MONA TIDAK punya stream key tersimpan (153, 697, 699, 700, 701, 702, 704, 705). Yang punya key: 114, 698, 703, 706, 707 | tabel `channels` |
| 5 | `key` adalah field WAJIB (`Field(...)`) di `CreateLiveRequest` — form tidak bisa disimpan tanpa key | `api/routes/live_stream.py:220` |
| 6 | `auto_resolve_channel_stream_key` (`live_broadcaster.py:1796`): Tier 1 key tersimpan → Tier 2 ambil/buat via YouTube API (butuh scope OAuth + kuota) → Tier 3 minta input manual / upgrade otorisasi | kode |
| 7 | Kuota vault sedang error pagi ini (burst `quotaExceeded` 07:22:32 di journal worker, puluhan channel) | journal worker |
| 8 | Tidak ada `guard_blocked_start` 2 jam terakhir; tidak ada task ERROR/STOPPED 2 jam terakhir; 3 task MONA (706/114/707) STREAMING normal | DB |
| 9 | Create/start TIDAK menulis audit log — kegagalan client-side tidak meninggalkan jejak apa pun | grep `audit` di `live_stream.py` |

### Kesimpulan
MONA membuka form buat-live, menekan tombol isi-key-otomatis **2× (pola khas "tidak berhasil, coba lagi")**, key tidak kunjung terisi, dan form tidak pernah bisa di-submit. Penyebab paling mungkin: **channel yang dipilih tidak punya key tersimpan (8 dari 13 channelnya), dan pengambilan key otomatis via YouTube API gagal** — entah karena kuota vault habis (terbukti error pagi ini) atau token OAuth kurang scope (Tier 3: butuh upgrade otorisasi yang tidak bisa ia selesaikan sendiri).

**Yang belum bisa dibuktikan dari server:** channel mana yang ia pilih + apa pesan yang muncul di layarnya (body respons auto-key tidak tercatat di log akses). Bila Bapak tahu channelnya, saya bisa pastikan 100%. Namun perbaikan di §2 menutup SEMUA cabang penyebab ini, apa pun channelnya.

### Celah sistemik yang terungkap
- **C1 — Form tidak memberi tahu syarat di muka.** Operator baru tahu key wajib setelah mengisi form. Tidak ada indikator "channel ini belum punya key" di pemilih channel.
- **C2 — Auto-key gagal secara diam-diam.** HTTP 200 tanpa key yang usable = operator tidak tahu apakah key tidak ada, kuota habis, atau butuh otorisasi ulang. Tidak ada pesan yang bisa ditindaklanjuti.
- **C3 — Kegagalan tidak tercatat.** Tanpa audit + tanpa log submit, diagnosis harus merekonstruksi dari log akses mentah. Seharusnya setiap percobaan buat/mulai tercatat (siapa, channel apa, gagal di langkah apa).
- **C4 — DIRECT_URL diterima API tapi diabaikan supervisor.** `CreateLiveRequest` menerima `audio_source_type=DIRECT_URL`, tapi rantai resolve audio di `live_supervisor.py:1091-1260` tidak punya cabang DIRECT_URL — task seperti itu jalan tanpa suara (jatuh ke MUTED) tanpa peringatan.
- **C5 — Audio hanya dari 4 sumber** (YouTube URL, depot, audio asli video, bisu). Audio MONA yang tersimpan di GDrive tidak punya jalan masuk — inilah yang diminta user di §3.

---

## 2. Saran perbaikan live manual (M1–M5)

| # | Saran | Implementasi |
|---|---|---|
| M1 | **Checklist pra-syarat di form** | Pemilih channel menampilkan status per channel: ✅ key tersimpan / ⚠️ belum ada key (+ tombol "Siapkan key") / ❌ kuota vault habis. Form tidak bisa lanjut sampai semua syarat hijau; tiap item yang merah menjelaskan CARA memperbaikinya. |
| M2 | **Auto-key jujur + bisa ditindaklanjuti** | Respons membedakan: `key_found` / `no_key_quota` ("Kuota vault habis — coba lagi setelah reset / pakai key manual") / `no_key_scope` ("Butuh otorisasi ulang — [tombol upgrade]") / `no_key_manual` ("Tempel key dari YouTube Studio → [panduan 3 langkah]"). Jangan pernah 200-kosong. |
| M3 | **Jalur key manual yang jelas** | Endpoint `save-key` sudah ada — tampilkan di form sebagai opsi utama bila auto-key gagal: field tempel key + tautan "Buka YouTube Studio → Go Live → salin key" + tombol simpan. |
| M4 | **Audit + log setiap percobaan** | `audit_logs` mencatat: buat task (sukses/gagal + alasan), mulai/hentikan, auto-key (hasil tier). Kegagalan client-side (validasi form) juga dikirim sebagai event ringan agar bisa didiagnosis tanpa menebak. |
| M5 | **Mode "cek kesiapan" satu klik** | Tombol di dashboard per channel: "Cek siap live?" → memeriksa key, video, audio, kuota vault, kapasitas node — menampilkan laporan ✅/❌ dalam 1 layar sebelum operator mengisi form. |

---

## 3. Opsi audio via link GDrive (G1–G7) — desain fitur

### Prinsip
Pakai ulang mesin yang SUDAH ADA, jangan bangun baru:
- Video sudah bisa dari GDrive: `gdrive://` di `live_supervisor.py:1063` → `download_gdrive_video("https://drive.google.com/uc?id=...")`.
- Endpoint import sudah ada tapi VIDEO saja: `POST /api/live/download-gdrive` (`live_stream.py:3140`).
- Parser link Drive sudah ada: `core/ingestion/pipeline.py:966` (`drive.google.com/file/d/...` + `gdrive://`) dan `worker/ingestion.py:2660` (Drive API `files().get_media`).

### Desain
| # | Item | Detail |
|---|---|---|
| G1 | **Mode audio baru `GDRIVE_URL`** | Tambah ke enum `audio_source_type` (backend + tab UI di form create/edit, sejajar YOUTUBE_URL/DEPOT/ORIGINAL/MUTED). Field: link Drive. |
| G2 | **Cabang supervisor** | Di rantai resolve audio (`live_supervisor.py:1091`), tambah cabang `GDRIVE_URL`: parse file ID dari link (share link, `uc?id=`, `gdrive://`) → unduh via `download_gdrive_video`-equivalent untuk audio (atau Drive API `get_media` bila file privat + kredensial tersedia) → simpan ke depot audio (cache per file ID, jangan unduh ulang) → pakai sebagai `audio_path` (loop seperti DEPOT). |
| G3 | **Perbaiki DIRECT_URL sekalian** | Tambah cabang DIRECT_URL yang hilang (C4): ffmpeg `-i <url>` langsung untuk URL audio http(s) biasa. Satu perbaikan, dua mode audio baru yang berfungsi. |
| G4 | **Validasi saat tempel link ("Cek link")** | Tombol cek: parse ID → HEAD/probe → tampilkan nama file, durasi, ukuran, codec (ffprobe). Tolak dengan pesan jelas: "File tidak bisa diakses — ubah ke Anyone with the link" (403), "Bukan file audio" (bukan mp3/m4a/wav/aac/flac/ogg), "Terlalu besar (>500 MB)". |
| G5 | **Import ke depot (opsional)** | Perluas `/api/live/download-gdrive` dengan `kind=audio|video` agar audio Drive bisa masuk ke dropdown DEPOT (dipakai ulang banyak channel, unduh sekali). |
| G6 | **Cache + dedup** | Nama file cache `gdrive_<file_id>.<ext>`; bila file sudah ada + ukuran cocok → pakai cache (hemat bandwidth + waktu start). Tombol "unduh ulang" bila file di Drive diperbarui. |
| G7 | **Keamanan & batas** | Hanya unduh dari `drive.google.com` / `*.googleusercontent.com` (tolak host lain); batas ukuran 500 MB; timeout unduh; hapus file parsial saat gagal; jangan log isi link bertoken. |

### Kuota & biaya
- Unduhan Drive TIDAK memakai kuota YouTube (0 unit). Drive API `get_media` memakai kuota Drive (longgar). Tidak ada biaya tambahan.

### Kriteria terima
- [ ] Tempel link Drive mp3 → "Cek link" menampilkan durasi/ukuran → simpan → stream jalan dengan audio itu (tes end-to-end dengan file Drive publik).
- [ ] Link privat / salah → pesan galat spesifik sebelum submit.
- [ ] DIRECT_URL kini berfungsi (tes: URL mp3 langsung → audio terdengar).
- [ ] File yang sama tidak diunduh 2× (cache hit terverifikasi di log).
- [ ] M1–M4 implemented (checklist, auto-key jujur, key manual jelas, audit percobaan).
- [ ] UI mengikuti `PANDUAN_GUI_MODERN_MASJAVAS.md`.

---

## 4. KOREKSI 08:09 WIB — penyebab sebenarnya: batas 100 karakter judul (bukan key)

Screenshot MONA (08:08:07 + 08:08:29, `live/create`) membuktikan hipotesis §1 SALAH untuk percobaan ini:

| Fakta dari screenshot | Arti |
|---|---|
| Channel: **[Slot #698] RuangIrama — Op: MONA** | 698 PUNYA key tersimpan |
| "Kunci Otomatis Terhubung (Database Server Vault): tm4d-****-****-c6qk", field key terisi `tm4d-8r3e-xd8y-uvew-c6qk` | **Auto-key BERHASIL — key bukan masalahnya** |
| Toast error: **"Gagal. body.title: String should have at most 100 characters"** | Submit ditolak server (422 Pydantic) karena `title` > 100 karakter |
| Judul: "PMGC 2026 LIVE \| PUBG Mobile Global Championship Istanbul — Live Scores,..." | Jauh di atas 100 karakter |
| Panel kanan: "24/7 Radio Loop Siap — **0 Lagu**" | Playlist audio channel ini kosong (jebakan kedua — lihat T5) |
| Preview: URL video `https://drive.google.com/file/` + audio `-g1-got3iA0.m4a` | Video sudah dari GDrive; audio .m4a |

### Mekanisme presisi (terverifikasi kode)
- Form punya DUA field judul: `autoBroadcastTitle` (line 225, **punya** `maxlength="100"` + label "Maks 100 Karakter") dan **Judul Tugas Internal** (line 370, **TANPA maxlength** — satu-satunya `maxlength` di seluruh form 2136 baris adalah di field broadcast).
- Payload submit mengirim judul internal sebagai `body.title` (line 1916, 2049) — dan server memvalidasi `title` dengan `max_length=100` (`CreateLiveRequest`).
- Jadi: MONA mengetik judul panjang di field internal (tanpa pembatas), form membiarkannya, server menolak mentah-mentah dengan pesan Inggris mentah Pydantic tanpa petunjuk perbaikan.
- Akar desain: judul internal dan judul broadcast YouTube **disatukan dalam satu field** — padahal batas 100 karakter adalah batas YouTube, bukan batas judul internal. Broadcaster sendiri memotong diam-diam (`[:100]`, `live_broadcaster.py:1529, 1697`) saat membuat broadcast dari `task.title`.

### Saran perbaikan (T1–T6) — scope tambahan implementasi
| # | Saran | Detail |
|---|---|---|
| T1 | **Samakan pembatas di field internal** | `maxlength="100"` + penghitung live "87/100" (kuning di 90+, merah di 100) di Judul Tugas Internal — perbaikan 1 baris yang mencegah kegagalan ini persis. |
| T2 | **Pisahkan judul internal dari judul broadcast** | Perbaikan struktural: `title` internal (bebas, maks 200) + `broadcast_title` (100, batas YouTube) sebagai field terpisah; broadcaster memakai `broadcast_title`, fallback ke internal yang dipotong. Form menampilkan keduanya + pratinjau. |
| T3 | **Terjemahkan 422 ke Bahasa + bisa ditindaklanjuti** | Jangan pernah tampilkan `body.title: String should have at most...` ke operator. Format: "Judul tugas internal terlalu panjang (132 karakter, maks 100). Persingkat 32 karakter." + auto-scroll ke field + highlight merah. Berlaku untuk SEMUA galat validasi, bukan hanya judul. |
| T4 | **Tombol "pendekkan otomatis"** | Bila judul kepanjangan, tawarkan pemotongan cerdas 1 klik (pertahankan kepala judul + buang ekor; tampilkan hasil sebelum diterapkan). |
| T5 | **Peringatkan playlist 0 lagu** | Bila mode audio butuh playlist dan channel punya 0 lagu, tampilkan peringatan sebelum submit ("Channel ini belum punya daftar lagu — pilih sumber audio lain atau isi dulu"). Jangan biarkan stream jalan tanpa audio. |
| T6 | **Tampilkan node tujuan + kapasitas** | Form menunjukkan tujuan ("Tujuan: VPS 8 / live-01 — 7/70 stream") agar operator tahu ke mana live-nya pergi dan apakah masih ada kapasitas. |

### Kriteria terima (T1–T6)
- [ ] Judul internal >100 karakter tidak bisa di-submit; penghitung tampil live; pesan Bahasa yang jelas + field ter-highlight.
- [ ] T2: judul internal panjang (≤200) + judul broadcast ≤100 tersimpan dan dipakai dengan benar (broadcast YouTube memakai broadcast_title).
- [ ] T5: channel 0 lagu memunculkan peringatan sebelum submit.
- [ ] T6: node tujuan + kapasitas tampil di form.
- [ ] Tes: submit judul 101 karakter ditolak di klien (tanpa memanggil server); 422 server diterjemahkan; regresi form create.

---

## 5. BUG DUPLIKAT 08:31 WIB — tugas TERBUAT tapi UI menampilkan error (double-submit + guard 409)

### Rantai bukti (terverifikasi)
1. **Task ADA dan SCHEDULED:** `#1795570268004`, 'RUANGIRAMA — SIARAN SANTAI 24/7', channel STAGING-SLOT-698, operator MONA, node `live-01`, `audio_source_type=GDRIVE_URL`, audio_url Drive `1YmVtzPOKZ...`, video `gdrive_video_1844C6SeW-.mp4`, created 08:31:08. Satu-satunya task slot 698.
2. **Dua 409 beruntun:** `POST /api/live/schedule` 08:31:08 → 409 Conflict dan 08:31:26 → 409 (IP browser MONA). TIDAK ADA POST /api/live/schedule sukses di jendela 08:30:30–08:32:00 dari IP mana pun — jejak POST pertama yang berhasil tidak tertangkap di log akses (kemungkinan terpotong rate-limit journald; keberadaan task + 2× 409 membuktikan urutannya: create pertama berhasil di detik yang sama, create kedua ditolak guard).
3. **Guard** (`api/routes/live_stream.py:1531-1542`, bundle LIVE, sudah ada): tolak create duplikat channel+judul+operator dalam 10 menit terakhir → 409 "Tugas duplikat: ... sudah dibuat N detik lalu (task #...). Gunakan tombol Start pada tugas yang sudah ada." Backstop UNIQUE constraint line 1869.
4. **Tombol SUDAH dikunci saat submit** (btn.disabled=true di 6 tempat, line 1789/1972/2021/2083/2248/2397; form `onsubmit=preventDefault`, tombol `type=button` + onclick `submitUniversalForm`) — double-fire bukan dari klik ganda polos; kemungkinan kunci dipasang terlambat (setelah await validasi/auto-key) atau dua jalur submit. Mekanisme double-fire pastinya belum terbukti — perbaikan harus menutup SEMUA jalurnya (idempotency key), bukan menebak satu jalur.

### Jadi yang MONA alami
Klik "MULAI SIARAN SEKARANG" → request pertama BERHASIL membuat task → request kedua (double-fire) ditolak 409 → **UI menampilkan error 409 dan menyembunyikan keberhasilan** → MONA mengira gagal total. Padahal tugasnya sudah ada, SCHEDULED, tinggal tekan Start.

### Temuan pendamping (dari log yang sama)
- **Frontend/backend tidak sinkron:** template baru (T2, M1, G1) sudah tayang di browser MONA — counter 31/200 + 100/100, checklist, tab Google Drive terlihat di screenshot 08:32 — tapi backend web masih kode 06:17 (PID 3246819, NRestarts=0). Buktinya: `POST /api/live/audit-event` → **405** (2×, 08:28:14) dan `POST /api/live/validate-source` → **405** (08:29:10) — endpoint baru belum ada di backend yang jalan. Ini juga menjelaskan checklist "Pemeriksaan gagal — lanjutkan manual."
- **GDRIVE_URL tersimpan tapi belum bisa jalan:** `audio_source_type` adalah string bebas di model API, jadi backend lama menyimpannya tanpa protes — tapi supervisor VPS 8 yang jalan (restart 07:22, kode pra-G1) TIDAK punya cabang GDRIVE_URL → bila task di-Start sekarang, audio akan BISU (jatuh ke MUTED). **Restart supervisor adalah gerbang sebelum MONA menekan Start.**
- `POST /api/live/download-gdrive` → 200 (08:29:21) — unduhan Drive bekerja (endpoint lama, video-only); cache `gdrive_video_1YmVtzPOKZ.mp4` (49,19 MB). CATATAN: file cache bernama `gdrive_video_*.mp4` — perlu dipastikan G1 menangani file .mp4 sebagai sumber audio (ekstrak trek audio) bila memang itu file yang MONA pakai sebagai audio.

### Saran perbaikan (D1–D6)
| # | Saran | Detail |
|---|---|---|
| D1 | **Kunci tombol di BARIS PERTAMA handler** | `btn.disabled=true` + flag `isSubmitting` sebelum await apa pun di `submitUniversalForm`; audit semua jalur submit ganda (klik + Enter + retry otomatis). |
| D2 | **Idempotency key** | Form membuat UUID per percobaan create; server dedup berdasarkan key dan mengembalikan task yang sudah dibuat (200) bila key sama — klik ganda jadi tidak berbahaya apa pun jalurnya. |
| D3 | **409 → UX sukses, bukan error** | Bila server menjawab duplikat, UI JANGAN tampilkan toast merah — tampilkan kartu "Tugas sudah dibuat 08:31:08" + tombol [Lihat tugas] [Mulai sekarang] yang langsung membuka/menyalakan task yang ada. Pesan error sekarang sudah menyuruh "Gunakan tombol Start" tapi tidak memberi jalan ke sana. |
| D4 | **Guard abaikan task yang GAGAL** | Dedup hanya cocokkan task berstatus hidup/pending (SCHEDULED/STREAMING/PAUSED); task FAILED/ERROR/CANCELLED TIDAK boleh memblokir percobaan ulang — atau tawarkan "hapus yang gagal & buat baru". Ini menjawab keluhan user secara harfiah: "job pertama gagal tapi dideteksi duplikat". |
| D5 | **409 sertakan task_id + deep link** | Body respons 409 memuat `task_id` + URL tugas agar UI bisa bertindak (prasyarat D3). |
| D6 | **Restart web + supervisor untuk menutup kesenjangan frontend/backend** | Template baru sudah tayang tanpa restart; backend + supervisor butuh restart agar endpoint baru (audit-event, validate-source), broadcast_title, dan cabang GDRIVE_URL aktif. Ini keranjang restart gabungan lintas lane — keputusan user. |

### Cara buka blokir MONA (sekarang)
1. JANGAN buat tugas baru — task #1795570268004 sudah ada (SCHEDULED, node live-01, audio GDrive tersimpan).
2. Setujui restart gabungan (web + supervisor VPS 8) → mengaktifkan backend GDrive audio + semua perbaikan.
3. Setelah restart terverifikasi, MONA tekan **Start** pada task #1795570268004 di dashboard. JANGAN Start sebelum restart (audio akan bisu).

### Epilog 08:53–09:03 WIB — video GDrive butuh pre-copy ke node (SUDAH TERATASI)
- 08:53: MONA tekan Start → task ERROR di DB lokal VPS 8: video `gdrive_video_1844C6SeW-.mp4` tidak ditemukan di storage node (file hasil unduhan `download-gdrive` tersimpan di cache MASTER, bukan di VPS 8). Master tetap SCHEDULED (ERROR lokal tidak langsung tersinkron).
- 08:54–09:00: pre-copy video (52,6 MB) + audio GDrive (49,2 MB) Master→VPS 8 via relay VM (sha256 cocok); status lokal kembali SCHEDULED (sync Master menimpa ERROR); MONA Start lagi → **STREAMING 09:00, 8/8 live**.
- **Pelajaran sistemik:** tugas video GDrive yang node-nya VPS 8 WAJIB pre-copy file video+audio ke node SEBELUM Start (aturan yang sama dengan pre-copy migrasi Fase C). Perbaikan ke depan (di luar D1–D6): supervisor node sebaiknya mengunduh sendiri dari URL GDrive bila file lokal tidak ada (cabang video gdrive:// sudah ada untuk video — pastikan dipakai saat video_path lokal hilang), atau endpoint create melakukan pre-copy otomatis saat node ≠ Master.

### Kriteria terima (D1–D6)
- [ ] Klik ganda / double-fire hanya membuat 1 task; percobaan kedua mengembalikan task yang sama (200), bukan 409.
- [ ] Duplikat sejati menampilkan kartu "tugas sudah ada" + tombol aksi, bukan toast merah.
- [ ] Retry setelah task FAILED tidak diblokir guard.
- [ ] Setelah restart: audit-event + validate-source 200; checklist berjalan; task GDRIVE_URL di-Start dengan audio terdengar.
- [ ] Tes: double-submit simultan (2 request bersamaan) → 1 task; regresi guard 10 menit.
