# Spek cron alert item 3.6 — SIAP DIBUAT KOORDINATOR (jangan dibuat worker)

Tiga cron alert baru + satu drill bulanan opsional, mengikuti pola cron 0.4
(`stage2-stall-alert`, 30 menit, state JSON, anti-spam 4 jam, diam saat OK
atau saat SSH gagal). Skrip sudah jadi + sudah diuji live dan simulasi.
Koordinator membuat cron-nya via `cron.add` (goal yang sama dengan 0.4:
`perbaikan-aplikasi-masjavas-tube-rencana-25-nov`, delivery chat yang sama
dengan 0.4).

Protokol skrip (semua): jalankan `python3 <skrip>`; baca baris pertama
stdout. `ALERT:` / `RECOVERED:` → kirim isi baris itu ke user (satu pesan
singkat). `OK:` / `SUPPRESSED:` / `ERROR:` → diam, jangan kirim apa pun.
Anti-spam 4 jam + status pemulihan dikelola skrip via state file — cron
tidak perlu logika tambahan. Jangan ubah apa pun di server; semua cek
read-only (df / journalctl).

---

## 1. alert-disk — Alert disk <20 GB

- **id:** `alert-disk`
- **Interval:** 60 menit, timezone Asia/Jakarta
- **Skrip:** `~/workspace/engineering/scripts/alert_disk.py`
- **State:** `~/workspace/engineering/alert_disk_state.json`
- **Ambang:** ruang kosong <20 GB di Master ATAU VPS 8.
- **Nilai saat disiapkan (25 Nov 08:37 WIB):** Master 61 GB, VPS 8 236 GB — aman.
- **Instruksi cron:** Jalankan `python3 ~/workspace/engineering/scripts/alert_disk.py`.
  Bila output diawali `ALERT:` atau `RECOVERED:`, kirim isi baris itu ke user
  di chat ini. Bila `OK:`, `SUPPRESSED:`, atau `ERROR:`, diam.

## 2. alert-quota — Alert quotaExceeded >10/jam

- **id:** `alert-quota`
- **Interval:** 30 menit, timezone Asia/Jakarta
- **Skrip:** `~/workspace/engineering/scripts/alert_quota.py`
- **State:** `~/workspace/engineering/alert_quota_state.json`
- **Ambang:** >10 baris `quotaExceeded` dalam journal 60 menit terakhir di
  Master ATAU VPS 8 (sumber: `journalctl --since '1 hour ago'`, read-only).
- **Nilai saat disiapkan (25 Nov 08:37 WIB):** Master 5/jam, VPS 8 0/jam — aman.
- **Instruksi cron:** Jalankan `python3 ~/workspace/engineering/scripts/alert_quota.py`.
  Bila output diawali `ALERT:` atau `RECOVERED:`, kirim isi baris itu ke user
  di chat ini. Bila `OK:`, `SUPPRESSED:`, atau `ERROR:`, diam.

## 3. alert-ghostkill — Alert ghost-kill >0/jam

- **id:** `alert-ghostkill`
- **Interval:** 30 menit, timezone Asia/Jakarta
- **Skrip:** `~/workspace/engineering/scripts/alert_ghostkill.py`
- **State:** `~/workspace/engineering/alert_ghostkill_state.json`
- **Ambang:** >0 kill sentinel dalam journal 60 menit terakhir di Master ATAU
  VPS 8. Penanda kill yang dieksekusi (satu-satunya): baris journal
  `[Live Watchdog Sentinel] Menghentikan ghost process #<tid> ...`
  (`worker/live_stream_engine.py:1480`). Baris "Kill ... ditahan" BUKAN kill
  dan tidak dihitung.
- **Nilai saat disiapkan (25 Nov 08:37 WIB):** Master 0, VPS 8 0 (1 jam dan
  6 jam terakhir) — aman. Total historis 785 (tidak berubah sejak 23 Nov).
- **Instruksi cron:** Jalankan `python3 ~/workspace/engineering/scripts/alert_ghostkill.py`.
  Bila output diawali `ALERT:` atau `RECOVERED:`, kirim isi baris itu ke user
  di chat ini. Bila `OK:`, `SUPPRESSED:`, atau `ERROR:`, diam.

## 4. (Opsional) restore-drill-monthly — Drill restore bulanan

- **id:** `restore-drill-monthly`
- **Interval:** bulanan, tanggal 1, 05:00 WIB (setelah backup harian 04:00)
- **Skrip:** `~/workspace/engineering/scripts/restore_drill.py`
- **Instruksi cron:** Jalankan `python3 ~/workspace/engineering/scripts/restore_drill.py`.
  Kirim ringkasan ke user HANYA bila hasil FAIL atau ERROR (sertakan baris
  HASIL + hitungan yang regresi). Bila PASS, diam; hasil tercatat di
  `~/workspace/engineering/backups/restore-drill.log`.

---

## Cara uji (sudah dijalankan 25 Nov 08:37 WIB — semua lolos)

Setiap skrip alert mendukung `--test-input '<JSON>' --state-file /tmp/<nama>.json`
untuk simulasi tanpa SSH. Siklus yang diuji per skrip: breach → `ALERT`
(exit 1); ulang → `SUPPRESSED` (exit 2); pulih → `RECOVERED` (exit 4);
ulang → `OK` (exit 0). Contoh:

```
python3 alert_disk.py      --test-input '{"master": 15.2, "vps8": 236.0}' --state-file /tmp/t_disk.json
python3 alert_quota.py     --test-input '{"master": 14, "vps8": 0}'       --state-file /tmp/t_quota.json
python3 alert_ghostkill.py --test-input '{"master": 2, "vps8": 0}'         --state-file /tmp/t_gk.json
```

Run live (SSH asli, 25 Nov 08:37 WIB): ketiga skrip `OK` — disk 61/236 GB,
quota 5/0 per jam, ghost-kill 0/0 per jam.
