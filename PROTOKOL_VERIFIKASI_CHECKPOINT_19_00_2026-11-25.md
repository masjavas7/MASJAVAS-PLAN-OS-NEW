# Protokol Verifikasi 2 Checkpoint — 25 Nov 2026

Verifikasi live perbaikan CTR audit (commit `8fdfcbb`, Fase 1.2/1.3) + governor
kuota, dan task 5142 (NO 267 Rock Kenangan). Dokumen ini + 2 skrip disiapkan
Fase 3 (worker persiapan verifikasi) 25 Nov 2026 ~08:32–08:50 WIB.
Semua cek READ-ONLY. TIDAK ada auto-fix di protokol ini.

## 0. Ringkasan prakondisi (hasil pre-check 08:32–08:39 WIB)

| # | Prakondisi | Hasil |
|---|-----------|-------|
| 1 | VPS 8 HEAD mengandung `8fdfcbb` | ✅ YA — HEAD `23de8d1`, tree bersih untuk file audit; service oneshot → run berikut memakai kode ini |
| 2 | Master HEAD mengandung `8fdfcbb` | ✅ YA — HEAD `173adec` (jalur trigger API) |
| 3 | Timer run audit 19:00 WIB | ❌ **TIDAK ADA** — `masjavas-audit.timer` = `18:00 UTC` = **01:00 WIB** harian; next fire 19:00 CET 25 Nov = 01:00 WIB 26 Nov. Run 19:00 WIB HANYA bisa via trigger manual (lihat §5 GAP-1) |
| 4 | Task 5142 ada + terjadwal | ✅ Ada, QUEUED — tapi jadwal **19:35 WIB** (epoch 1795610100), BUKAN 19:30; harmonizer memindahkan 19:30→19:35 pukul 08:22 WIB (kuota vault habis sampai reset 15:00) |
| 5 | Armada 25 Nov (08:38 WIB) | PUBLISHED 25, QUEUED 186, SKIPPED_AUDIO_MISSING 31, AWAITING_MONETIZATION 1, **FAILED 1** (task 5254, slot 472, "Token tidak ditemukan", 08:36:22), BLOCKED 0 |

> ⚠️ **GAP-1 (kritis):** tanpa keputusan trigger manual, TIDAK AKAN ADA run audit
> 19:00 WIB dan verifikasi 19:05 akan melaporkan `C1 FAIL — batch baru tidak ditemukan`.
> Keputusan trigger = koordinator/user (masuk keranjang persetujuan, §5).
> ⚠️ **GAP-2:** checkpoint "5142 terbit 19:30" sudah basi — jadwal live 19:35 WIB.
> Verdict terbit paling awal 20:05 WIB (jadwal + grace 30 mnt).
> ⚠️ **Catatan lantai SOLID:** FAILED armada sudah 1 hari ini (5254, bukan 5142).
> T4 (FAILED=0) akan FAIL malam ini kecuali 5254 pulih/di-retry sebelum 19:35.
> Jangan samakan T4 dengan verdict 5142 (T1–T3).

## 1. Skrip

| Skrip | Fungsi |
|---|---|
| `~/workspace/engineering/scripts/verify_audit_1900.py` | Verifikasi run audit (pre/post/dry-run) |
| `~/workspace/engineering/scripts/verify_task_5142.py` | Verifikasi task 5142 (pre/post) |

Dijalankan dari VM Muse (bukan dari server). Satu perintah per run.
Exit: `0` = semua PASS, `1` = ada FAIL, `2` = PENDING / UNKNOWN (lihat §4).
Durasi normal ~10–60 dtk; bila SSH VPS 8 flaky bisa ~3 mnt (retry 3×45 dtk,
lalu cek journal jadi UNKNOWN, cek Master tetap jalan).

## 2. Perintah persis

```bash
cd ~/workspace/engineering/scripts

# --- Checkpoint 1: run audit ---
python3 verify_audit_1900.py --pre        # pre-check kesiapan (kapan saja)
python3 verify_audit_1900.py --post       # verifikasi pasca-run (19:05+; ideal 19:20+)
python3 verify_audit_1900.py --dry-run    # uji skrip vs data baseline 01:00 (harus FAIL)

# --- Checkpoint 2: task 5142 ---
python3 verify_task_5142.py --pre         # status + jadwal saat ini
python3 verify_task_5142.py --post        # verdict pasca-jadwal (19:35+; final 20:05+)
python3 verify_task_5142.py --post --grace-minutes 45   # bila perlu grace lebih panjang
```

**Urutan malam ini (bila trigger manual 19:00 disetujui):**

1. ~18:55 — `verify_audit_1900.py --pre` (konfirmasi ulang HEAD masih mengandung
   `8fdfcbb`; repo bisa berubah sejak pre-check pagi — Master restart 08:40,
   VPS 8 sempat pull).
2. 19:00 — trigger run (keputusan koordinator/user, BUKAN bagian skrip ini).
3. 19:05 — `verify_audit_1900.py --post` → ekspektasi PENDING (run ~14 mnt,
   baseline 809,7 dtk) atau C1 FAIL bila run belum menyimpan. **Bukan verdict.**
4. **19:20** — `verify_audit_1900.py --post` → **verdict audit**.
5. 19:35 — `verify_task_5142.py --post` → ekspektasi PENDING (task baru terpicu).
6. **20:05** — `verify_task_5142.py --post` → **verdict 5142** (jadwal + grace 30 mnt).

## 3. Cek, ambang PASS/FAIL, nilai yang diharapkan

### 3.1 Audit — `--post` (scope: batch `audited_at >= 2026-11-25 19:00 WIB`)

| Cek | PASS bila | Baseline 01:00 WIB (yang harus diperbaiki) | Harapan run 19:00 |
|-----|-----------|--------------------------------------------|-------------------|
| C1 batch baru terdeteksi | >0 baris `video_audit` baru (Master otoritatif, VPS 8 silang) | 2.577 baris, batch 01:13:54 WIB, 53 vault | ~2.500–2.600 baris baru |
| C2 CTR distinct > 2 | nilai CTR distinct > 2 | **1** (hanya 0.0) | puluhan–ratusan (CTR Analytics nyata) |
| C3 max CTR | > 0 dan ≠ persis 0,0455 | **0,0** (agregat hari: persis 0,0455) | > 0, nilai riil bervariasi |
| C4 kategori A dan B | A > 0 DAN B > 0 | A=0, B=0 | A>0, B>0 (kategori dari CTR riil) |
| C5 ledger tercatat | ≥1 baris `AUDIT_READ` di `vault_quota_ledger` dalam jendela run | **0 baris** (gagal catat, readonly DB) | >0 baris + units >0 |
| C6 governor | maks `AUDIT_READ`/vault sejak tengah malam PT ≤ 1000 unit | n/a (0 tercatat) | semua vault ≤1000; marker `AUDIT_CAP`/`AUDIT_QUOTA_EXCEEDED` boleh ada (0 unit) |
| C7 quotaExceeded | hitungan journal < 144 (baseline) | **144** | jauh < 144 (governor + stop per-channel) |
| C8 selesai | journal memuat `Audit Selesai` | ✅ 809,7 dtk, 209 ch sukses, 0 gagal | ✅ + ringkasan serupa |

Catatan C6: PASS saja tidak bermakna bila C5 FAIL (0 pemakaian selalu ≤ cap).
C6 hanya valid bersama C5 PASS. Cap 1000 = `MAX_DAILY_AUDIT_UNITS_PER_VAULT`
(10% kuota harian/vault, `core/audit/collector.py:69`); hari kuota = tengah
malam Pacific Time.

### 3.2 Task 5142 — `--post` (jadwal DIBACA LIVE dari DB; saat ini 19:35 WIB)

| Cek | PASS bila | Nilai pre-check 08:38 WIB |
|-----|-----------|---------------------------|
| T1 status | `PUBLISHED` | QUEUED (step AUTO_RETRY) |
| T2 video id | `youtube_video_id` terisi (+ URL) | kosong |
| T3 waktu terbit | `updated_at` ≥ jadwal | — (jadwal 19:35 WIB) |
| T4 armada FAILED | FAILED = 0 | **1** (task 5254, slot 472, token hilang, 08:36:22) |
| T5 armada BLOCKED | BLOCKED = 0 | 0 |

Konteks 5142: NO 267 Rock Kenangan, judul `Rock Kenangan - 2026-11-25`,
track GROWTH, worker `vps-node-01`. Riwayat: gagal 07:06 WIB
`database is locked` → AUTO_RETRY; harmonizer geser ke 19:30 lalu 19:35 WIB
karena kuota vault `MV-E279614148457BF3` habis sampai reset 15:00 WIB.
Task ini TIDAK ADA di DB VPS 8 (max id 4768) — verifikasi HANYA di Master.

### 3.3 Exit code & status

- `0` semua PASS. `1` ada FAIL. `2` PENDING (run/task belum selesai dalam
  grace window) atau UNKNOWN (VPS 8 tak terjangkau → C7/C8 tidak bisa dinilai;
  ulangi cek, JANGAN anggap PASS).
- PENDING bukan FAIL: ulangi sesuai §2. UNKNOWN bukan FAIL: ulangi cek.

## 4. Diagnosis bila FAIL (tanpa auto-fix — laporkan ke koordinator)

### Audit

- **C1 FAIL (batch baru tidak ada):**
  1. Apakah run 19:00 benar dipicu? (tidak ada jadwal otomatis — GAP-1).
     Cek `systemctl status masjavas-audit.service` + `journalctl -u
     masjavas-audit.service --since "2026-11-25 13:00" --until "2026-11-25 14:30"`
     di VPS 8 (jam CET = WIB−6).
  2. Bila journal memuat `Beban host tinggi` → run DI-SKIP load gate
     (load1 > 90% vCPU). Load VPS 8 sempat 8,95/8 core pukul ~08:41 WIB.
     Diagnosis: catat load saat 19:00; JANGAN paksa run (live streaming prioritas).
  3. Bila service aktif >25 mnt tanpa batch → run menggantung; laporkan,
     jangan kill (koordinator yang putuskan).
- **C2/C3/C4 FAIL (CTR masih palsu):** CTR nyata gagal diambil — kemungkinan
  Analytics API gagal / fallback 0.0 (`core/audit/collector.py` baris ~377).
  Cek journal run untuk error Analytics + hitung berapa baris `impressions > 0`
  di batch baru. Bila SEMUA impressions=0 → sumber Analytics gagal total.
- **C5 FAIL (ledger 0):** pencatatan ledger masih gagal (dulu: readonly DB).
  Cek journal untuk `Gagal mencatat ledger kuota audit`. Bedakan dari C6:
  C5 FAIL + C6 PASS = ledger tidak jalan, BUKAN governor bekerja.
- **C6 FAIL (vault >1000):** governor tidak menahan — cek apakah run memakai
  kode `8fdfcbb` (ulang `--pre` P1/P2) dan apakah ada `AUDIT_CAP` marker.
- **C7 FAIL (quotaExceeded ≥144):** badai berulang — governor + stop
  per-channel tidak efektif. Bandingkan jumlah vault kena cap (marker
  `AUDIT_QUOTA_EXCEEDED`) vs total vault batch.
- **C8 FAIL (tanpa `Audit Selesai`):** run tidak selesai normal — baca ekor
  journal 20 baris terakhir service untuk baris error terakhir.
- **C7/C8 UNKNOWN:** SSH VPS 8 flaky (terjadi saat persiapan: rc=3 beruntun,
  pulih setelah jeda 45–60 dtk). Tunggu 1 mnt, ulangi `--post` sekali.

### Task 5142

- **T1 FAIL setelah 20:05 (status bukan PUBLISHED):** baca `status`,
  `current_step`, `error_message` task 5142 + 3 entri terakhir
  `execution_logs`. Bila `database is locked` lagi → pola gagal 07:06 berulang
  (Fase 2.5 `0eb8bab` seharusnya menangani — laporkan sebagai regresi kandidat).
  Bila status QUEUED dan jadwal BERUBAH lagi → harmonizer menggeser (baca
  catatan HARMONIZE terakhir; skrip sudah menampilkan jadwal live).
- **T1 PASS tapi T2 FAIL:** status PUBLISHED tanpa video id → inkonsistensi
  tulis; cek `youtube_video_url` + log eksekusi.
- **T4 FAIL:** sebutkan task FAILED-nya (id + slot + error), JANGAN samakan
  dengan 5142. Per 08:38 WIB sudah ada 1 (5254/slot 472/token hilang).
- **T3 FAIL (updated_at < jadwal):** `updated_at` tidak bergerak saat publish —
  kemungkinan status berubah tanpa eksekusi baru; cek log eksekusi.

## 5. Celah prakondisi & kandidat keranjang persetujuan (L3 — TIDAK dieksekusi worker ini)

1. **GAP-1 — Trigger run audit 19:00 WIB.** Opsi (koordinator/user yang pilih):
   (a) `POST /api/audit/run-phase0` (OWNER-only, `api/routes/audit.py:209`,
   background pipeline; Master sudah mengandung `8fdfcbb`);
   (b) `systemctl start masjavas-audit.service` di VPS 8 (oneshot, kode repo).
   Catatan: run ~14 mnt (baseline 809,7 dtk) + load gate (skip bila load1 >
   90% vCPU). Alternatif: geser checkpoint ke run terjadwal 01:00 WIB 26 Nov
   (timer yang ada) dan verifikasi 01:20 WIB.
2. **GAP-2 — Jadwal 5142 = 19:35 WIB, bukan 19:30.** Checkpoint 19:30 basi;
   verdict final 20:05 WIB. Bila harmonizer menggeser lagi, skrip membaca
   jadwal live — cron 19:35 tetap valid sebagai snapshot, verdict ikut jadwal
   + grace.
3. **Task 5254 FAILED (slot 472, token tidak ditemukan, 08:36:22 WIB).**
   Di luar scope verifikasi ini; lantai 0 FAILED sudah ternoda hari ini.
   Perlu tindak lanjut terpisah (re-pair token / retry) — BUKAN oleh skrip ini.
4. **Tidak ada perubahan timer/cron yang dilakukan worker ini.** Bila
   koordinator ingin run audit 19:00 WIB menjadi JADWAL TETAP, itu perubahan
   unit systemd (L3) — perlu persetujuan terpisah.

## 6. Spek cron (TEKS SAJA — koordinator yang membuat cron-nya)

Cron Muse (one-time, 25 Nov 2026, zona Asia/Jakarta). Setiap run: jalankan
perintah, laporkan PASS/FAIL + angka ke koordinator. Jangan buat cron
berulang — ini checkpoint satu malam.

```
# CRON-A1 — 19:05 WIB: verifikasi audit (cek awal; ekspektasi PENDING/C1-belum-selesai)
Perintah: cd ~/workspace/engineering/scripts && python3 verify_audit_1900.py --post
Kondisi:  hanya bermakna bila trigger manual 19:00 disetujui & dijalankan.
          Tanpa trigger, hasil ekspektasi: C1 FAIL (tidak ada batch baru).
Tindak lanjut: apapun hasilnya, JANGAN simpulkan verdict dari run ini.

# CRON-A2 — 19:20 WIB (DIREKOMENDASIKAN, tambahan): verdict audit
Perintah: cd ~/workspace/engineering/scripts && python3 verify_audit_1900.py --post
Alasan:   run audit ~14 mnt; batch baru mendarat ~19:14. Inilah verdict C1–C8.

# CRON-T1 — 19:35 WIB: verifikasi task 5142 (snapshot; ekspektasi PENDING)
Perintah: cd ~/workspace/engineering/scripts && python3 verify_task_5142.py --post
Kondisi:  jadwal live 19:35 WIB — task baru terpicu; PENDING = normal.

# CRON-T2 — 20:05 WIB (DIREKOMENDASIKAN, tambahan): verdict task 5142
Perintah: cd ~/workspace/engineering/scripts && python3 verify_task_5142.py --post
Alasan:   jadwal 19:35 + grace 30 mnt; setelah ini non-PUBLISHED = FAIL.
```

Opsional (bila koordinator mau): pre-check ulang ~18:55 WIB
(`verify_audit_1900.py --pre && verify_task_5142.py --pre`) untuk konfirmasi
HEAD + jadwal masih sama sebelum checkpoint.

## 7. Bukti uji skrip (25 Nov 2026, VM Muse)

- `verify_audit_1900.py --pre` (08:38 WIB): exit 0 — P1/P2/P4/P5 PASS,
  P3 GAP jadwal, P6 baseline 2.577 baris / CTR distinct 1 / max 0,0 / A=0 / B=0.
- `verify_audit_1900.py --dry-run` (08:38 WIB): exit 1 — **5 FAIL
  (C2, C3, C4, C5, C7)** pada data baseline 01:00; C1/C8 PASS; C6 PASS trivial
  (0 unit ≤ cap — tidak bermakna tanpa C5). Membuktikan skrip mendeteksi
  masalah lama: CTR distinct=1, max=0,0, A=B=0, ledger 0 baris,
  quotaExceeded=144.
- `verify_audit_1900.py --post` (08:44 WIB, belum ada run): exit 1 —
  C1 FAIL "0 baris audited_at >= 19:00 WIB" (benar: belum ada run).
- `verify_task_5142.py --pre` (08:38 WIB): exit 0 — QUEUED, jadwal live
  19:35 WIB, harmonizer 08:22, armada FAILED=1.
- `verify_task_5142.py --post` (08:39 WIB, sebelum jadwal): exit 2 —
  PENDING dengan verdict final 20:05 WIB.
- Temuan uji: SSH VPS 8 flaky (rc=3 kosong, beruntun, pulih ±60 dtk) —
  skrip sudah retry 3×45 dtk + fail-soft (C7/C8 UNKNOWN, cek Master tetap jalan).
