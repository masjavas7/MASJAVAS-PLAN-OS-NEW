# Panduan heartbeat eksternal — UptimeRobot Free (item 3.6c)

PANDUAN SETUP SAJA. Jangan buat akun/monitor tanpa keputusan koordinator +
user. UptimeRobot free: 50 monitor, interval cek 5 menit, alert email +
Telegram, tanpa kartu kredit.

## Mengapa

Monitor internal (cron Muse + SSH) kadang buta saat proxy SSH flaky.
Heartbeat eksternal memantau dari luar: bila domain tidak menjawab, user
diberi tahu walau semua jalur internal mati.

## Monitor 1 — Healthz publik (UTAMA)

| Setelan | Nilai |
|---|---|
| Monitor Type | HTTP(s) |
| Friendly Name | `MASJAVAS TUBE — healthz` |
| URL | `https://masjavastube.my.id/healthz` |
| Monitoring Interval | 5 minutes (batas paket free) |
| Keyword | `"status":"healthy"` (tipe monitor: Keyword — DOWN bila keyword hilang) |
| Alert Contacts | Email user + Telegram (opsional, lihat bawah) |

Respons yang diharapkan (terverifikasi live 25 Nov 08:36 WIB):

- HTTP **200**
- Body JSON memuat `"status":"healthy"`, `"app":"MASJAVAS TUBE OS"`,
  `"checks":{"db":true,"disk":true,"worker":true}`, `latency_ms` kecil
  (~3,5 ms internal), `disk_free_gb` ~60.

Contoh body (dipotong): `{"status":"healthy","app":"MASJAVAS TUBE OS",
"version":"4.0.0-PROD","checks":{"db":true,"disk":true,"worker":true},
"latency_ms":3.54,"details":{"telemetry_age_sec":14,"health_score":85,
"disk_free_gb":60.22,"system_disk_free_gb":60.21}}`

Catatan latensi: pengukuran eksternal dari VM Muse menunjukkan total
~1,4 dtk (didominasi egress proxy), sementara `latency_ms` internal
~3,5 ms. JANGAN pasang alert berbasis response-time ketat — pantau status
+ keyword saja.

## Monitor 2 — Halaman utama (OPSIONAL)

| Setelan | Nilai |
|---|---|
| Monitor Type | HTTP(s) |
| Friendly Name | `MASJAVAS TUBE — homepage` |
| URL | `https://masjavastube.my.id/` |
| Monitoring Interval | 5 minutes |
| Keyword | — (cukup HTTP 200/3xx; halaman login boleh redirect) |

Tujuan: membedakan "aplikasi mati" (keduanya DOWN) vs "endpoint healthz
saja bermasalah" (hanya Monitor 1 DOWN).

## Langkah setup (5–10 menit)

1. Buka https://uptimerobot.com → Sign Up (Free Plan) dengan email user.
2. Dashboard → Add New Monitor → isi sesuai tabel Monitor 1 di atas →
   Create Monitor.
3. (Opsional) Tambahkan Monitor 2.
4. My Settings → Alert Contacts → tambah email user; bila mau Telegram:
   tambah kontak Telegram, ikuti instruksi pairing bot UptimeRobot
   (@UptimeRobotBot), lalu kaitkan ke kedua monitor.
5. Uji: di daftar monitor, klik titik-tiga → "Check now" / tunggu 1 siklus
   5 menit — status harus UP. Uji DOWN: JANGAN matikan aplikasi untuk uji;
   cukup buat monitor sementara ke URL yang salah
   (`https://masjavastube.my.id/healthz-salah`) dan pastikan alert DOWN
   masuk, lalu hapus monitor sementara itu.

## Bila alert DOWN masuk

1. Cek `https://masjavastube.my.id/healthz` dari browser — 200 atau tidak.
2. Bila tidak menjawab: kemungkinan tunnel (`masjavas-os-tunnel.service`)
   mati, web service mati, atau VPS Master mati — eskalasi ke koordinator
   (restart butuh persetujuan user; JANGAN restart sepihak).
3. Bila healthz 200 tapi UptimeRobot bilang DOWN: kemungkinan keyword
   berubah (deployment mengubah body) — perbarui keyword di monitor.

## Yang TIDAK dipantau UptimeRobot (tetap via cron Muse)

- VPS 8 / kalender-web (`http://100.108.161.98:8081`) — hanya tailnet,
  tidak dapat dijangkau internet publik.
- Detail internal: publish/hari, FAILED, Stage 2 stall, disk, kuota,
  ghost-kill — dipantau cron 0.4 + alert 3.6
  (`ALERT_CRON_SPECS_3.6.md`).
- UptimeRobot free tidak punya status page publik tanpa konfigurasi
  tambahan; bila user mau status page, aktifkan terpisah (gratis).
