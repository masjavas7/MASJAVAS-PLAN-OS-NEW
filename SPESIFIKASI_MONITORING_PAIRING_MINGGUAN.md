# Spesifikasi Monitoring Pairing Mingguan — MASJAVAS TUBE (Fase 3.1d)

Status: SPEK SAJA — cron BELUM dibuat (keputusan koordinator/user).
Tanggal: 2026-11-25. Generator: `scripts/pairing_priority.py` (read-only, re-runnable).

## 1. Tujuan

Pantau CONNECTED/UNPAIRED per operator tiap minggu terhadap target
(RENCANA §7 3.1: +20 channel paired/minggu, interim 300 CONNECTED dalam
3 minggu dari 243). Baseline 25 Nov 2026 08:34 WIB: CONNECTED 243,
UNPAIRED 464 (AZIZ 240, ARIEF 126, AGUS 30, HAMIDAH/SALSA/WULAN 15,
BILA 13, LUTHFI 10, MONA 0).

## 2. Definisi cron (JANGAN dibuat sebelum disetujui)

- Jadwal: **Senin 08:00 WIB** (`0 8 * * 1`, zona Asia/Jakarta).
- Lokasi: cron di mesin Muse (bukan di Master — tidak perlu deploy file
  ke server produksi; generator dijalankan via SSH pipe seperti sekarang).
- Perintah (satu wrapper, tanpa tulis DB):
  ```
  cd ~/workspace/engineering && \
  ssh -F /home/hatch/.ssh/config masjavas-vps-baru \
    '/opt/masjavas-os/venv/bin/python - --format json' \
    < scripts/pairing_priority.py \
    > pairing_snapshots/$(date +%F).json
  ```
  disusul langkah diff (lihat §3). Snapshot disimpan di
  `~/workspace/engineering/pairing_snapshots/YYYY-MM-DD.json`.
- Keluaran tiap run: 1 baris ringkasan ke
  `~/workspace/engineering/pairing_weekly.log` + pesan ringkasan untuk
  koordinator (format §4). Tidak ada penulisan ke DB produksi, tidak ada
  pesan otomatis ke operator (draf pesan tetap lewat user/koordinator).

## 3. Logika diff + ambang alert

Bandingkan snapshot minggu ini vs minggu lalu (per operator):
- `delta_connected` per operator dan fleet.
- Alert (untuk koordinator, bukan auto-aksi):
  - CONNECTED fleet TURUN vs minggu lalu (ada unpair — cek sebab).
  - `delta_connected == 0` dua minggu berturut-turut untuk operator
    dengan backlog UNPAIRED > 0 (stagnan — butuh dorongan).
  - Slot vault kosong (cap-5) < 20 → vault baru wajib disiapkan
    (baseline 25 Nov: 55 slot ≈ 2,8 minggu pada +20/minggu).
  - Vault mana pun > cap-5 (baseline: 2 vault, MV-252AB8E353B2421B dan
    MV-57D170A57C9BA7EE, masing-masing 6 channel).
- Pencapaian target: `delta_connected >= 20` = target tercapai.

## 4. Format ringkasan mingguan (untuk koordinator)

```
Pairing minggu <tanggal>: CONNECTED <n> (<delta> vs minggu lalu),
UNPAIRED <n>. Target +20/minggu: <tercapai/belum, selisih>.
Per operator: AZIZ <c>/<t> (<delta>), ARIEF ..., dst.
Slot vault kosong: <n> (ambang alert <20).
Operator stagnan 2 minggu: <nama / tidak ada>.
```

## 5. Draf pesan dorongan ke operator (teks untuk user — belum dikirim)

### 5a. Pengumuman umum (grup operator)

> Tim MASJAVAS TUBE — update pairing: dari 707 channel, 243 sudah
> CONNECTED, 464 masih UNPAIRED. Target kita +20 channel paired per
> minggu. Daftar prioritas per operator sudah disiapkan (urut nomor
> slot). Mohon pairing via dashboard — pilih vault dulu, lalu hubungkan
> akun YouTube. Channel yang sudah punya vault bisa langsung dipairing.
> Terima kasih!

### 5b. AZIZ (240 UNPAIRED — terbanyak, 51,7% backlog fleet)

> Pak AZIZ — 240 channel Bapak masih UNPAIRED (dari 251). Target minggu
> ini: 10 channel. Catatan: semua 240 belum punya vault, jadi sebelum
> pairing, vault perlu ditetapkan dulu oleh koordinator. Daftar 10
> pertama minggu ini (urut nomor slot): <diisi dari laporan prioritas>.
> Mohon kabari bila ada kendala login/akun.

### 5c. ARIEF (126 UNPAIRED)

> Mas ARIEF — 126 channel masih UNPAIRED (dari 143). Target minggu ini:
> 5 channel. Sama seperti AZIZ, channel-channel ini belum punya vault —
> koordinator akan menetapkan vault dulu, lalu Mas ARIEF pairing via
> dashboard. Daftar 5 pertama: <diisi dari laporan prioritas>.

### 5d. AGUS (30 UNPAIRED, 70 sudah CONNECTED — progres terbaik kedua)

> Mas AGUS — 70 dari 100 channel sudah CONNECTED, terbaik kedua di
> fleet. Tinggal 30 lagi; target minggu ini 1 channel dulu (prioritas
> ke AZIZ/ARIEF karena backlog mereka jauh lebih besar). Terima kasih
> atas progresnya!

### 5e. Operator rotasi — BILA (13), HAMIDAH (15), LUTHFI (10), SALSA (15), WULAN (15)

> <Nama> — tinggal <n> channel lagi menuju 40/40 CONNECTED. Target
> minggu ini: 1 channel. Daftar prioritas sudah disiapkan (urut nomor
> slot). Semangat!

### 5f. MONA (13/13 CONNECTED — 100%)

> Mbak MONA — 13/13 channel sudah CONNECTED, 100%! Terima kasih.
> Tidak ada target pairing minggu ini; mohon bantu jaga agar tidak ada
> channel yang ter-unpair.

## 6. Keputusan user yang dibutuhkan

1. **Target mingguan**: +20/minggu (RENCANA §7 3.1) atau lain? Pada
   +20/minggu, 464 backlog habis dalam ~23 minggu (±5,5 bulan).
   Interim 300 CONNECTED dalam 3 minggu butuh +19/minggu — konsisten.
2. **Pembuatan vault baru**: 55 slot kosong habis dalam ~2,8 minggu
   pada +20/minggu; total 464 backlog butuh ~93 vault pada cap-5
   (60 terdaftar, 55 slot kosong → ~84 vault baru sampai backlog habis).
   Siapa yang membuat project GCP + OAuth client baru, dan dengan tempo
   apa? (L3/infra — masuk keranjang persetujuan bila dieksekusi.)
3. **Penetapan vault untuk P2 (460 channel)**: pekerjaan OWNER/
   koordinator sebelum operator bisa pairing — disetujui sebagai tugas
   koordinator? (Menulis `master_vault_id` ke DB = tulis DB produksi —
   butuh persetujuan; BUKAN bagian dari tugas 3.1 ini.)
4. **Cron monitoring §2**: disetujui dibuat? (Jadwal Senin 08:00 WIB,
   read-only, tanpa pesan otomatis ke operator.)
5. **Perbaikan label UI** (kecil, bukan L3): modal pairing masih
   menulis "Batas Kuota 6 Ch / Vault" padahal aturan mengikat cap-5
   (web/templates/modals/modal_channel_account.html:118). Perbaiki
   label + 2 vault over-cap (6 ch) — ditangani kapan? Kandidat
   keranjang persetujuan (perubahan kode 1 baris + re-assign 2 channel).

## 7. Catatan verifikasi widget (3.1c — ringkasan)

Widget dashboard pairing (api/main.py:404-434) menghitung langsung dari
DB: `COUNT(*)` + `SUM(status='CONNECTED')`, unpaired = total − connected;
operator non-OWNER difilter ke slotnya sendiri. Hitungan independen
dari DB (707/243/464) COCOK dengan logika widget — widget benar dari
sisi kode. Dua catatan: (a) bila query DB gagal, template memakai
fallback hardcode 327 untuk OWNER (web/templates/dashboard.html:177) —
nilai usang (yang benar 464); (b) widget hanya menampilkan hitungan,
TIDAK ada daftar prioritas — daftar prioritas adalah hasil baru tugas
3.1 ini (`LAPORAN_PAIRING_PRIORITAS_2026-11-25.md`).
