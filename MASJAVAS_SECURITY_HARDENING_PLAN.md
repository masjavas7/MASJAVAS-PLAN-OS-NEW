# RENCANA HARDENING KEAMANAN — MASJAVAS TUBE OS V2

**Status:** DOKUMEN RENCANA (Workstream G). Keadaan kini didokumentasikan
dari bukti yang ada; langkah hardening yang belum dilakukan ditandai
sebagai proposal dengan pemilik/tingkat persetujuan.
**Sumber utama:** `MASTER_ARCHITECTURE_MASJAVAS_TUBE_OS_V2.md` §21
(Security dan Operational Governance), §5, §14; `TOOLS.md` (akses VPS);
`MEMORY.md` + `AGENTS.md` (insiden token, rebalance vault, deploy key);
`ENGINEERING_READINESS_DASHBOARD.md` (status gerbang).
**Terakhir diperbarui:** 22 Nov 2026.

**PERINGATAN:** dokumen ini TIDAK BOLEH berisi nilai kredensial apa pun
(token, password, kunci, PIN). Pelanggaran terhadap aturan ini adalah
insiden keamanan — sama seperti eksposur token yang dicatat di §5.

**Label bukti:** `[VERIFIED FACT]` / `[DOCUMENTED FACT NOT YET REVALIDATED]` /
`[ASSUMPTION]` / `[PROPOSAL]` / `[OPEN QUESTION]` /
`[OWNER DECISION REQUIRED]`.

---

## 1. Keadaan kini (terverifikasi)

### 1.1 Akses jaringan dan SSH

| Item | Keadaan |
|---|---|
| SSH ke VPS | Hanya via Tailscale (tailnet). SSH outbound langsung (port 22) DIBLOKIR platform. `[VERIFIED FACT — TOOLS.md, 21 Okt 2026]` |
| Master live | `masjavas-vps-baru` (Contabo vmi3762689, tailnet 100.65.206.22). `[VERIFIED FACT]` |
| Box lama | `masjavas-vps` (Tencent, 100.115.180.126) — layanan aplikasi DIHENTIKAN 22 Nov 07:05 (R1); box masih berjalan (SSH/Tailscale/VNC/RDP infra tidak disentuh). `[VERIFIED FACT]` |
| Autentikasi SSH | Key auth. `PasswordAuthentication off` (server hardened). `[VERIFIED FACT — TOOLS.md, 21 Okt 2026]` |
| Firewall | UFW deny-inbound kecuali `tailscale0`. `[VERIFIED FACT — TOOLS.md, 21 Okt 2026]` |
| Akses publik | Dashboard/API publik via domain `masjavastube.my.id` + tunnel (tunnel ACTIVE, diverifikasi 22 Nov 07:05). Daftar port publik yang terbuka di luar tunnel BELUM diaudit. `[OPEN QUESTION — bagian dari V2-006 inventory]` |

### 1.2 Kredensial dan secret

| Kelas kredensial | Penyimpanan | Catatan |
|---|---|---|
| Secret aplikasi (`.env`) | `/opt/masjavas-os/.env` di Master; backup 0600 di `data/backups/` | JANGAN PERNAH dibaca utuh — grep satu key saja. `[VERIFIED FACT — praktik 22 Nov]` |
| Token OAuth YouTube (per channel) | Tabel `auth_tokens`, dienkripsi AES-256-GCM dengan AAD `CHANNEL=<id>::VAULT=<master_vault_id>` | Pindah vault = update `channels` + `auth_tokens` + re-enkripsi + **re-pair** (pelajaran insiden 12 Nov — §6). `[VERIFIED FACT]` |
| Kunci KEIN | Hanya di encrypted dashboard config (OWNER-only) | Kredensial KEIN di connector Muse TIDAK bisa dipakai VPS. `[VERIFIED FACT]` |
| OWNER PIN | Tidak dicatat di mana pun (teks) | Aturan: secret yang disampaikan di chat tidak pernah direkam sebagai teks. `[VERIFIED FACT — aturan berlaku]` |
| Service Account Google Drive | Mekanisme service account (izin editor pada folder) | Verifikasi share HARUS cocokkan permission ID, bukan filter email (pelajaran 13 Nov: `permissions list` tidak mengembalikan `emailAddress`). `[VERIFIED FACT — AGENTS.md]` |
| Token GitHub (PAT, scope repo) | Secure Vault via konektor `custom.github` (terhubung 22 Nov 12:34, login masjavas7; helper `bin/github_api.py` + `bin/sync_plan_repo.py`, terverifikasi GET /user 200 + sync 8/8) | Jalur token adalah SATU-SATUNYA jalur GitHub yang dipakai (keputusan user: "hanya jalur token"); konektor GitHub OAuth tidak terhubung dan tidak dipakai. `[VERIFIED FACT — MEMORY.md 22 Nov 12:34]` |
| Token Telegram | Notifier ada (`core/telemetry/telegram_notifier.py`) | Detail penyimpanan token tidak diaudit di dokumen ini. `[OPEN QUESTION]` |
| Kunci Tailscale / auth | Tidak didokumentasikan di sini | Inventarisasi + prosedur re-auth bila kunci hilang. `[OPEN QUESTION]` |

### 1.3 Service dan filesystem

- Service aplikasi berjalan sebagai user non-root (worker jalan sebagai
  `ubuntu`) `[VERIFIED FACT — AGENTS.md, 19 Nov]`. Beberapa file menjadi
  root-owned karena lane lain menjalankan perintah via sudo — sebelum
  patch/compile, cek `ls -la` + `chown` bila perlu (pelajaran 19 Nov).
- Backup `.env` dibuat dengan mode 0600 `[VERIFIED FACT — praktik deploy
  22 Nov]`.
- Tabel `audit_logs` ada di database `[DOCUMENTED FACT NOT YET
  REVALIDATED — grounding checklist]`; retensi dan cakupannya belum
  diaudit `[OPEN QUESTION]`.

## 2. Prinsip tata kelola (V2.0 §21)

Aturan yang mengikat untuk V2 (dan untuk Master kini sejauh dapat
diterapkan tanpa mengganggu produksi):

1. API worker hanya melalui Tailscale/private network.
2. Setiap worker memiliki identity dan credential sendiri.
3. Credential tidak dicatat di log.
4. Rotasi token didukung.
5. Rate limit per worker.
6. Validasi payload dan capability.
7. Batasi akses filesystem.
8. Pisahkan user systemd untuk workload bila layak.
9. Jangan menjalankan service sebagai root tanpa alasan.
10. Firewall default deny.
11. Simpan audit log.
12. Deployment menggunakan release manifest.
13. Semua secret dan backup dienkripsi.

## 3. Rencana hardening

### 3.1 Jaringan dan akses (prioritas P1)

| # | Langkah | Status |
|---|---|---|
| S1 | Audit port publik Master + daftar layanan yang mendengarkan (bagian dari V2-006 service inventory). | `[PROPOSAL — belum jalan]` |
| S2 | Konfirmasi UFW default-deny masih berlaku pasca-migrasi Contabo (aturan terakhir diverifikasi di box Tencent, 21 Okt). | `[OPEN QUESTION — revalidasi diperlukan]` |
| S3 | Tailscale: inventarisasi device yang tergabung + kunci auth; prosedur revoke device yang tidak dikenal. | `[PROPOSAL]` |
| S4 | VPS 8 (saat provisioning): terapkan baseline yang sama SEBELUM service live — SSH key-only, UFW deny-inbound kecuali Tailscale, tanpa password auth. | `[PROPOSAL — Track P7.1]` |
| S5 | API worker (V2): autentikasi per-worker (identity + credential sendiri), rate limit per worker, validasi payload (V2.0 §21 butir 2, 5, 6). | `[PROPOSAL — Track P4]` |

### 3.2 Kredensial dan rotasi (prioritas P0 untuk item terpapar)

| # | Langkah | Status |
|---|---|---|
| S6 | Cabut PAT GitHub read-only repo skill (terpapar di chat 4×, sejak 20 Nov). Buat baru dengan scope minimal bila masih dibutuhkan. | `[OWNER DECISION REQUIRED — perintah sudah disampaikan berulang kali, belum dijalankan user; JANGAN nagih di luar konteks rotasi]` |
| S7 | Rotasi token GitHub yang tertulis di chat 22 Nov 12:25 (dipakai transien untuk pembuatan repo `MASJAVAS-PLAN-OS-NEW`; saran rotasi disampaikan 12:30). Token GitHub kini tersimpan di vault via `custom.github` (12:34). Apakah token vault adalah nilai yang sama dengan yang tertulis di chat TIDAK DIKETAHUI — jangan asumsikan; bila sama, rotasi tetap disarankan karena sudah terpapar. | `[OPEN QUESTION — konfirmasi user]` |
| S8 | Ganti deploy key read-only repo app dengan mekanisme push yang benar (insiden 11 Nov: push ditolak — "key marked as read only"; PAT lama 11 Nov yang seharusnya di-revoke ternyata masih aktif). | `[OPEN QUESTION — status kini belum diverifikasi]` |
| S9 | Kebijakan rotasi berkala: token API eksternal (KEIN, dll.) dirotasi terjadwal; tanggal rotasi dicatat di register (bukan nilainya). | `[PROPOSAL]` |
| S10 | Larangan keras: kredensial TIDAK PERNAH di chat, dokumen, log, URL, atau repo git. Pelanggaran = cabut + rotasi segera. | `[ATURAN BERLAKU]` |

### 3.3 Service dan data (prioritas P2)

| # | Langkah | Status |
|---|---|---|
| S11 | Pisahkan user systemd per workload bila layak (V2.0 §21 butir 8; unit yang diusulkan di §10.1–10.2). | `[PROPOSAL — Track P2.1]` |
| S12 | Audit kepemilikan file di tree aplikasi (root-owned vs ubuntu) + tetapkan pemilik standar. | `[PROPOSAL]` |
| S13 | Audit retensi + cakupan `audit_logs`; pastikan event keamanan (login, perubahan izin, deploy) tercatat. | `[PROPOSAL]` |
| S14 | Backup secret hanya via repository terenkripsi (lihat `MASJAVAS_BACKUP_RECOVERY_PLAN.md` §3). | `[PROPOSAL — Track P8.4]` |
| S15 | Validasi: tidak ada secret di log aplikasi (grep berkala untuk pola token/kunci di `data/logs/`). | `[PROPOSAL]` |

## 4. Model ancaman ringkas [PROPOSAL]

| Ancaman | Dampak | Mitigasi yang ada / direncanakan |
|---|---|---|
| Kredensial terpapar (chat/log/repo) | Akses tidak sah ke GitHub / API / akun | S6–S10 (rotasi + larangan); S15 (grep log). |
| Akses SSH tidak sah | Kendali penuh atas box | Key-only + Tailscale-only + UFW (§1.1). |
| Satu akun/provider untuk kedua VPS | Insiden provider/akun melumpuhkan keduanya | Backup independen di luar kedua VPS (rencana backup §1). |
| Worker nakal / tertukar identitas | Job salah, data tertimpa | Identity per worker + fencing token (V2.0 §8.3, §21). |
| Penghapusan file salah (janitor) | Bahan channel hilang | Quarantine + 8 cek pra-hapus (V2.0 §11); mode status-only sampai V2-004 lolos. |
| Token OAuth channel bocor/rusak | Publish channel gagal | Enkripsi AAD per vault; rebalance hanya dengan prosedur 4 langkah + re-pair (§6). |

## 5. Insiden keamanan yang dipelajari

1. **Deploy key read-only (11 Nov 2026).** Push dari VPS ditolak
   GitHub ("key marked as read only"). Pelajaran: bedakan key
   baca-saja vs tulis; jangan asumsikan push bisa dilakukan dari
   host mana pun tanpa menguji autentikasinya.
2. **Token repo terpapar 4× (19–20 Nov 2026).** PAT read-only repo
   skill tertulis di chat (termasuk chat James). Perintah cabut +
   buat baru disampaikan berulang kali; sampai 22 Nov belum
   dijalankan user. Pelajaran: token yang sudah terpapar dianggap
   TERBONGKAR — cabut, jangan "dipakai dulu sementara".
3. **Vault rebalance mematahkan token (12 Nov 2026).** Mengubah
   `channels.master_vault_id` saja mematahkan dekripsi (AAD
   terikat vault); re-enkripsi saja tidak cukup karena refresh
   token terikat OAuth client vault lama (`unauthorized_client`).
   Pelajaran: perubahan identitas kredensial = prosedur penuh
   (update 2 tabel + re-enkripsi + re-pair) + verifikasi ACTIVE
   per channel sebelum dinyatakan selesai.
4. **PAT klasik di chat (22 Nov 2026).** User memberi PAT di chat
   untuk pembuatan repo. Dipakai transien via env per-perintah
   (API saja), tidak disimpan, `gh` tidak di-login. Setelah itu
   (12:34) user menyelesaikan kartu brankas → konektor
   `custom.github` terhubung (vault, scope repo). Keputusan user:
   jalur token adalah SATU-SATUNYA jalur GitHub yang dipakai
   ("hanya jalur token"); konektor GitHub OAuth tidak dipakai.
   Pelajaran: token di chat tetap pengecualian atas pilihan
   eksplisit user — dan harus dirotasi sesudahnya bila nilainya
   sama dengan yang tersimpan (S7).

## 6. Aturan perubahan kredensial (mengikat)

- Perubahan apa pun pada kredensial/auth adalah **Level C**
  (persetujuan owner) menurut Master Autonomous Engineering Directive.
- Jangan pernah mengubah `master_vault_id` tanpa prosedur 4 langkah
  (§1.2) + verifikasi dekripsi + verifikasi satu refresh/task sukses
  pasca-pindah per channel yang dipindah.
- Jangan pernah memindahkan channel antar vault tanpa izin operator
  (doktrin kuota 18 Nov).
- Setelah rotasi token apa pun: verifikasi layanan yang memakai token
  itu masih berfungsi (satu panggilan/uji nyata), lalu catat tanggal
  rotasi di log — tanpa nilainya.

## 7. Pertanyaan terbuka

1. Daftar port publik Master saat ini — `[OPEN QUESTION — V2-006]`.
2. Apakah UFW default-deny masih berlaku di box Contabo —
   `[OPEN QUESTION — revalidasi]`.
3. Inventaris device Tailscale + prosedur revoke —
   `[OPEN QUESTION]`.
4. Penyimpanan token Telegram + cakupan `audit_logs` —
   `[OPEN QUESTION]`.
5. Status deploy key / PAT repo app saat ini (S8) —
   `[OPEN QUESTION — belum diverifikasi]`.
6. Mekanisme penyimpanan kunci enkripsi backup —
   `[OPEN QUESTION — lihat rencana backup §3]`.
