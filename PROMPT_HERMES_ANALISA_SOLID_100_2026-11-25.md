# PROMPT UNTUK HERMES — ANALISA MENUJU APLIKASI SOLID 100%

**Tanggal:** 25 November 2026, 14:30 WIB
**Dari:** Muse (atas instruksi owner, Abdul Aziz)
**Tujuan:** Hermes menganalisa aplikasi MASJAVAS TUBE OS dan menyusun rencana terukur menuju solid 100% — dari skor 66/100, kesiapan NOT READY (±65%).
**Sifat tugas:** ANALISA SAJA. Bukan perbaikan, bukan eksekusi.

---

## 0. PERANMU — BACA DULU

- Kamu adalah **Creative Director Agent** — otak analisa aplikasi, BUKAN eksekutor.
- Tugas ini **hanya menganalisa**. Jangan ubah kode, database, konfigurasi, jadwal, atau me-restart layanan apa pun.
- Semua outputmu adalah **DRAFT terstruktur ke Director Inbox**. Owner yang memutuskan mana yang dieksekusi.
- **Larangan keras:** tidak memanggil YouTube API; tidak menulis ke DB produksi (kecuali tabel inbox); tidak mengubah jadwal publish, kuota, vault, atau YPP; tidak restart/deploy; tidak menghapus file; tidak membaca `/opt/masjavas-os/.env` atau secret apa pun.
- **Aturan mengikat yang berlaku untuk semua analisamu:**
  - *Change rule:* setiap perubahan harus hati-hati dan terverifikasi (rencana verifikasi pre/post + rollback + jendela observasi) — jangan pernah mengusulkan perbaikan yang mengorbankan bagian yang sudah solid.
  - *Evaluasi-first:* analisa + verifikasi dulu sebelum ada keputusan perbaikan.
  - *Brand priority:* MASJAVAS adalah satu-satunya brand prioritas.
  - *Fakta vs dugaan:* pisahkan dengan tegas. Dugaan tanpa bukti ditulis sebagai dugaan, bukan kesimpulan.

---

## 1. DEFINISI "SOLID 100%" — YANG KAMU TUJU

- **Skor soliditas 100/100** = 9 komponen, masing-masing bernilai 100: publish, live, LoopGen/Stage 2, ingestion, audit, Agent AI, Kalender, Hermes, infra/keamanan.
- **Gerbang kesiapan: READY** (saat ini NOT READY, ±65%).
- **Scorecard 13 metrik** di `PLAN_PENAMBAHAN_VPS8_2026-11-22.md` §2 (baseline vs target): satu metrik memburuk = gerbang gagal = rollback.
- **100% bukan janji tanpa syarat.** Untuk tiap komponen kamu WAJIB merumuskan kriteria terukur + jendela observasi, contoh:
  - Publish: 0 FAILED selama 7 hari berturut-turut DAN p90 delay ≤ 30 menit.
  - Live: 0 ghost-kill selama 7 hari DAN 0 stream bisu DAN 0 restart supervisor tanpa pelaku selama 14 hari.
  - Stage 2: 0 antrean > 12 jam DAN PARKED < 5% dari total item.
  - Keamanan: 0 PIN duplikat, 0 PAT lama aktif, 0 endpoint tanpa auth.
- Bila suatu kriteria 100% **tidak realistis** dalam waktu dekat, katakan terus terang dan usulkan **target antara yang terukur** (mis. 85 dulu, lalu 100) beserta syaratnya.
- **Jangan campur dua angka skor yang ada di log:** baseline tugas ini adalah **66/100 (Cycle 23)**. Ada angka lain, **75/100 (dihitung 05:49 + 06:18)**, yang memakai komposisi berbeda dan dokumennya sendiri menegaskan angka itu BERBEDA dari kesiapan gerbang ±65%. Pakai 66/100 sebagai titik awal; sebut 75/100 hanya bila relevan, dengan penjelasan perbedaannya.

---

## 2. KEADAAN AWAL — FAKTA PER 25 NOV 2026, 14:25 WIB

Angka di bawah ini adalah **titik awal, bukan kebenaran final**. Tugas pertamamu (Fase A) adalah memverifikasi ulang semuanya dengan bukti langsung. Bila ada yang sudah berubah saat kamu bekerja, pakai angka terbarumu dan catat perbedaannya.

| # | Komponen | Skor kini | Fakta kunci (25 Nov 14:25 WIB) |
|---|----------|-----------|----------------------------------|
| 1 | Publish | 88 | 94 PUBLISHED hari ini, 1 FAILED — slot 494 (Grey Ballads), pola BARU: 401 `youtubeSignupRequired` saat upload, healer retry 2× gagal identik → butuh re-pair OAuth. 24 Nov final 210 PUBLISHED, 23 Nov 207 — lantai 206 terlampaui 2 hari berturut-turut. Healer bekerja (task 5254 pulih, #409 akhirnya retry setelah ~19 jam). |
| 2 | Live | 65 | 8/8 STREAMING; ghost-kill 0 — jendela ke-39 berturut-turut, total tetap 785 sejak 22 Nov. B10 (kematian encoder massal) TERBUKTI hilang — akar IPv6 RTMP tidak stabil, fix paksa-IPv4 aktif. Fase 0 (A1/A2/A3/E4 + D1) aktif di VPS 8 sejak 12:12. TAPI: **3 dari 8 siaran BISU** (NO 114 Ruang Galau, NO 88 Rock Religi, NO 34 Ballad Jingga — relay HLS mati total, cookie yt-dlp kedaluwarsa / E5); **5× restart supervisor VPS 8 hari ini** (semua perintah, bukan crash, pelaku belum teridentifikasi — tiap restart mematikan 8/8 encoder, auto-reconnect pulih <3 mnt); load VPS 8 10,87/9,75/9,55 (jenuh, di atas 8 core). |
| 3 | LoopGen / Stage 2 | 45 | Stage 2: DONE 1404, PARKED 285, FAILED 24, QUEUED 2; antrean tertua 44,3 jam (id 1594, ch504, 23 Nov 17:25); bergerak sangat lambat (+1 dalam ~30 mnt). **3 dari 6 worker LoopGen stalled 2,5+ jam** (AGUNG video, MUSE GAMBAR 1, MUSE VIDEO 1) + LUTHFI MUSE 7 diam ~15,5 jam, 0 DONE. Laju video fleet −31% vs 24 Nov (27 vs 39). Klaim channel: 306 COMPLETED / 347 EXPIRED / 3 ACTIVE (rasio EXPIRED 52,9%). |
| 4 | Ingestion | 60 | Total 671 job, 0 FAILED, PENDING 0; #405/#406/#407 COMPLETED (A30/30 T30/30). TAPI #542 diam ~51 jam (eskalasi ke-36). |
| 5 | Audit | 45 | **CTR masih fabrikasi** (selalu tepat 0,0455 — query tidak meminta impressions, impressions = views×22). 87 baris CTR fabrikasi perlu dibersihkan; timer audit GAP-1 (19:00 CET = 01:00 WIB, tidak ada run 19:00 WIB tanpa trigger manual). |
| 6 | Agent AI (Sumopod) | 75 | Fase B live (B1 metadata, B2 klasifikasi, B3 UI `/agents`). **Fase C (agen menulis metadata ke YouTube) DITAHAN — agen hanya membuat draft.** |
| 7 | Kalender Live | 70 | **kalender-web MATI** (unit systemd inactive, proses detached sejak 14:40 WIB 24 Nov, tanpa auto-restart). Endpoint `/api/kalender/events` tanpa auth (GET → 200, R61). Seed me-reset state (20 CANDIDATE + 1 SCHEDULED + 1 REJECTED, bukan 21 SCHEDULED). |
| 8 | Hermes | 70 | Bridge daemon aktif di Master (panel `/hermes`). Konsumen inbox 1.956 baris, loop BELUM diregistrasi. |
| 9 | Infra / keamanan | 55 | **5 PIN operator SAMA dengan PIN produksi** (hash cocok — wajib rotasi). **PAT GitHub lama MASIH AKTIF** + auth GitHub dari Master rusak (push/pull gagal sejak 24 Nov). Janitor hapus 7 file audio in-flight slot 406 (guard 2.3 diduga hanya melindungi RUNNING, bukan PENDING pasca-restart). **Gelombang `quotaExceeded` (P2): mekanisme BELUM terpecahkan — 6 teori sudah gugur; jangan menebak.** Satu vault (MV-252AB8E353B2421B) sisa 310 unit. Backup DB harian sempat gagal 2× (alias Tencent mati) — sudah diperbaiki, OK 04:28 (156 MB, integrity ok). |

**Risiko tercatat:** R57 (slot 494 FAILED 401) · R58 (janitor hapus audio #406) · R59 (restart supervisor tanpa pelaku) · R60 (web crash-loop 09:11–09:17, RESOLVED + aturan baru: migrasi skema wajib diuji di salinan DB live) · R61 (endpoint Kalender tanpa auth). **Keranjang 12 item Fase 3** menunggu keputusan owner (rotasi PAT + 5 PIN, penetapan vault ~459 channel, Fase C, Kalender F4, aktivasi Hermes, dst).

---

## 3. SUMBER YANG WAJIB DIBACA

**Dokumen (knowledge base):**
- `~/workspace/engineering/EVALUASI_SEBELUM_PERBAIKAN_2026-11-22.md` — register 14 problem (P1–P16) + matriks prioritas.
- `~/workspace/engineering/VERIFIKASI_BOTTLENECK_2026-11-22.md` + `ANALISA_ARSITEKTUR_BOTTLENECK_2026-11-22.md` — bukti bottleneck + yang belum terpecahkan.
- `~/workspace/engineering/PLAN_PENAMBAHAN_VPS8_2026-11-22.md` — plan of record, **§2 scorecard 13 metrik**, §7 budget mengikat **$34,67/bln** (Master ±$16,02 + VPS 8 $18,65).
- `~/workspace/engineering/RENCANA_PERBAIKAN_LIVE_STREAMING_2026-11-25_v2.md` + `PROMPT_AGENT_IMPLEMENTASI_LIVE_STREAMING_2026-11-25_v2.md` — rencana live terbaru (Fase 0 berjalan).
- `~/workspace/engineering/AUTONOMOUS_WORK_LOG.md` (Cycle 23) + `STATUS_REGISTRY.yaml` + Risk Register v2.2 + Decision Register v2.1.
- `~/workspace/engineering/docs/DESAIN_HERMES_CREATIVE_DIRECTOR_2026-11-24.md` — desain peranmu sendiri (prinsip, larangan, kontrak inbox).
- `~/memory/2026-11-25.md` — log harian terbaru (monitor tiap 30 mnt, insiden, koreksi).
- `~/workspace/engineering/PROMPT_ANTIGRAVITY_PERBAIKAN_MENYELURUH_2026-11-23.md` — contoh format prompt perbaikan yang dipakai lane lain.

**Sumber live (HANYA baca):** repo `/opt/masjavas-os` (baca file + `git log`), SQLite **read-only** (`/opt/masjavas-os/data/masjavas_tube.sqlite`), `systemctl status/show`, `journalctl`, `uptime`/`ps`, endpoint aplikasi (healthz, `/api/telemetry/nodes`, dashboard). Bila suatu sumber tidak bisa diakses, catat sebagai keterbatasan — jangan mengarang angkanya.

---

## 4. LANGKAH ANALISA — KERJAKAN BERURUTAN

### Fase A — Verifikasi independen (hari 1)
Verifikasi ulang **setiap klaim di §2** dengan bukti langsung (log / DB / API / SSH read-only). Beri label tiap klaim: **TERKONFIRMASI** / **TERBANTAH** / **BELUM DAPAT DIVERIFIKASI** (+ alasan). Jangan percaya klaim lane lain tanpa bukti — sudah terbukti ada klaim yang salah (contoh: "loop split-brain berhenti" ternyata masih ada 1 kill; "Fase 0 belum aktif" ternyata sudah aktif sejak 12:12).

### Fase B — Akar masalah
Untuk tiap komponen, bedakan **gejala vs akar** (5 Whys / fishbone). Untuk mekanisme yang **belum terbukti** (terutama P2 `quotaExceeded`), tulis tegas **"MEKANISME BELUM DIKETAHUI"** + langkah diagnosis yang dibutuhkan — **JANGAN menebak perbaikan**. Menebak perbaikan untuk mekanisme yang belum dipahami adalah pelanggaran change rule.

### Fase C — Gap analysis menuju 100%
Untuk tiap komponen: apa yang kurang menuju 100 + bukti + **ambang terukur** (§1). Sebutkan juga apa yang SUDAH solid dan harus dipertahankan (jangan sampai rencana perbaikan merusaknya).

### Fase D — Prioritisasi
Susun prioritas dengan **RICE atau dampak×usaha**, lengkap dengan urutan + dependensi antar-item. Pisahkan **quick wins** (risiko rendah, dampak cepat) dari **perbaikan fondasi** (risiko tinggi, butuh jendela observasi). Perhatikan batasan: max 1 restart / 15 menit lintas lane; live disentuh terakhir, satu stream terendah dulu; tes hanya di DB salinan / DB sementara.

### Fase E — Rencana remediasi per item
Untuk tiap item perbaikan, tulis: (a) perbaikan yang diusulkan, (b) **pemilik** (lane yang mengeksekusi: Antigravity / koordinator / operator / owner), (c) estimasi usaha, (d) risiko + rollback, (e) **kriteria verifikasi** (pre/post + jendela observasi + bukti yang harus ditunjukkan), (f) biaya ($/Rp, terhadap cap $34,67/bln).

### Fase F — Risiko baru + batas pengetahuan
Daftarkan risiko yang BELUM tercatat di register. Lalu tulis dengan jujur **apa yang TIDAK bisa kamu jawab** dari data yang ada + data/akses apa yang dibutuhkan untuk menjawabnya.

---

## 5. FORMAT OUTPUT — WAJIB

1. **Laporan utama** (Bahasa Indonesia):
   - Ringkasan eksekutif — 1 halaman: skor kini, 3 temuan terbesar, 3 tindakan pertama.
   - Tabel komponen: skor kini → target → gap → bukti → ambang 100% terukur.
   - Daftar temuan bernomor (**S-01, S-02, …**): tiap temuan berisi deskripsi + bukti (file:baris / log / waktu cek) + dampak + akar (atau "belum diketahui").
   - Rencana prioritas (Fase D+E) + scorecard pemantau 100% (metrik, cara ukur, frekuensi cek).
2. **Setiap klaim faktual WAJIB bersumber** (dokumen + baris, query DB, baris log + timestamp, atau endpoint + waktu cek). **0 klaim tanpa sumber.**
3. Simpan laporan ke **Director Inbox sebagai DRAFT** (skema `AnalysisBrief`: judul, ringkasan, tabel komponen, temuan, rencana, sumber). Status awal DRAFT — tidak dikonsumsi aplikasi sebelum owner menyetujui.
4. **Tanpa placeholder** (`[isi]`, `contoh`, `lorem`, `TODO`) — entri berisi placeholder ditolak validasi deterministik.

---

## 6. ATURAN JUJUR — PEGANG TEGUH

- Bila 100% tidak realistis untuk suatu komponen dalam waktu dekat, katakan + beri target antara yang terukur.
- Bila dua sumber bertentangan, laporkan konfliknya — jangan memilih diam-diam.
- Jangan klaim perbaikan yang belum diuji. Jangan salin klaim laporan lain tanpa verifikasi sendiri.
- Bila tugas ini butuh lebih dari satu sesi, pecah jadi batch dan laporkan progres per batch — jangan diam lalu muncul dengan laporan setengah jadi tanpa jejak.

---

## 7. KRITERIA SELESAI TUGAS INI

- [ ] Semua klaim §2 terverifikasi / terbantah / dinyatakan belum dapat diverifikasi, dengan bukti.
- [ ] Tiap komponen punya ambang 100% yang terukur + jendela observasi.
- [ ] Rencana prioritas lengkap: pemilik + estimasi + risiko + rollback + kriteria verifikasi + biaya per item.
- [ ] 0 klaim faktual tanpa sumber. 0 placeholder.
- [ ] Laporan masuk Director Inbox sebagai DRAFT + ringkasan eksekutif disampaikan ke owner.

**Estimasi waktu:** sebutkan sendiri di awal pekerjaanmu (bila >1 sesi, pecah jadi batch).

---

## CARA MENJALANKAN PROMPT INI

Kirim prompt ini ke Hermes lewat Telegram setelah instalasi di Master selesai (perintah go + token bot + model Direktur masih menunggu owner). Prompt ini adalah **tugas analisa pertama** Hermes — sebelum pilot NicheBrief/EventPack/WorkOrder di desain Creative Director. Hasilnya (DRAFT AnalysisBrief) menjadi dasar keputusan owner untuk perbaikan menuju solid 100%.
