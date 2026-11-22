# RENCANA UJI PENERIMAAN VPS 8 — MASJAVAS TUBE OS V2

**Status:** DRAFT — disiapkan, BELUM dieksekusi. Dokumen pendamping:
`VPS8_DEPLOYMENT_RUNBOOK.md` (langkah) dan `VPS8_ROLLBACK_RUNBOOK.md`
(prosedur mundur bila uji gagal).
**Versi:** 1.0 — 22 Nov 2026, 12:35 WIB.
**Aturan gerbang:** satu metrik scorecard memburuk = gerbang gagal =
rollback. Setiap fase: observasi 72 jam, 0 error class baru.
[VF — PLAN §2/§6; aturan perubahan 22 Nov 01:48]

## Legenda label klaim

- **[VF]** VERIFIED FACT · **[DF]** DOCUMENTED FACT NOT YET REVALIDATED ·
  **[AS]** ASSUMPTION · **[PR]** PROPOSAL · **[OQ]** OPEN QUESTION ·
  **[OD]** OWNER DECISION REQUIRED. (Definisi penuh: Deployment Runbook.)

## 1. Baseline "solid" (acuan semua uji)

| # | Metrik | Baseline (21–22 Nov) | Sumber |
|---|---|---|---|
| 1 | Publish/hari | 206 (21 Nov), 0 FAILED/BLOCKED | [VF] PLAN §2 |
| 2 | Keterlambatan terbit p90 (median 11,6 mnt) | 30,2 mnt | [VF] PLAN §2 |
| 3 | Kematian encoder tak direncanakan | **KOREKSI:** 12×/17 jam, slot **114** — BUKAN "5×/24 jam slot 706" seperti tertulis di PLAN §2 baris 3 (pemetaan lama SALAH, dibuktikan V2-002). Target: ≤1×/24 jam → 0×/72 jam | [VF] V2-002; target [DF] PLAN §2 |
| 4 | Encoder orphan | 0 (706 sudah dipulihkan 22 Nov 08:10; sebelumnya 1) | [VF] D11 |
| 5 | "database is locked"/24 jam | 31× | [VF] PLAN §2 |
| 6 | Load Master (rata-rata) | 3,5–4,2 | [VF] PLAN §2 |
| 7 | CPU unit web (rata-rata) | 1,02 core | [VF] PLAN §1.5 |
| 8 | Ingestion RUNNING saat backlog ada | 2 job (konkurensi terkunci 1 oleh 5 cap bertumpuk — V2-003) | [VF] PLAN §2 + V2-003 |
| 9 | Churn SUPERSEDED ingestion | 66% (7 hari; V2-003: 67% masih terjadi) | [VF] PLAN §2 + V2-003 |
| 10 | Disk Master (puncak wave) | 72% | [VF] PLAN §2 |
| 11 | Healthz p95 | ±10 ms | [VF] PLAN §2 |
| 12 | Stream live aktif | 2 reguler (114, 706); limit 16 | [VF] |
| 13 | Error class baru pasca-fase | 0 dalam 72 jam observasi | [DF] PLAN §2 (target) |

**Baseline tambahan (untuk uji prep):** Stage 2: 12,8 dtk CPU/klip,
wall 100,1 dtk/klip (I/O dominan), max 900,8 dtk; 1.011 DONE/7 hari.
[VF] Reassemble: 64×/24 jam, ±0,9 core/job, ±6 mnt/job, ±70 MB/job,
puncak ±1 core (job berurutan). [VF — ADDENDUM_REASSEMBLE]
**Perbaikan yang sudah live saat baseline ini ditulis:** encoder
R3–R8 (commit `56f7a6d`) + iterasi 2 (commit `021a9f0`); observasi
24 jam pasca-iterasi-2 masih berjalan; kematian #14 tercatat
08:09:50 (45 mnt pasca-deploy pertama). [VF]

**Cara membaca tabel uji:** kolom "Lolos bila" adalah kriteria
keberhasilan yang terukur; "Gagal bila" adalah pemicu rollback
(lihat Rollback Runbook). Semua pengukuran dicatat di log uji (§6).

---

## 2. Uji Fase 0 ($0, Master saja)

| ID | Uji (langkah) | Lolos bila | Gagal bila | Scorecard |
|---|---|---|---|---|
| T0.1 | Pisah proses publish vs prep (0.1) | 2 unit jalan; restart prep tidak mengganggu 1 job publish yang sedang berjalan (selesai normal); 0 publish gagal selama uji | Publish gagal/terlambat >p90 selama restart prep | 1, 2 |
| T0.2 | Job berat keluar dari web (0.2) | CPU web rata-rata 24 jam <0,2 core; 0 anak ffmpeg di bawah proses web (cek 3×/hari) | CPU web ≥0,2 core; ≥1 anak ffmpeg di web | 6, 7 |
| T0.3 | Klaim pull + 1-owner (0.3) | Churn SUPERSEDED <20% dalam 7 hari; 0 duplikat job (channel, kind) aktif bersamaan | Churn ≥20%; ≥1 duplikat terdeteksi | 9 |
| T0.4 | Sweeper batch 5 + guard lokal (0.4) | Feed ≥2 channel/menit (rata-rata 24 jam); 100% penahanan guard tercatat alasannya di log (audit sampel 50 baris) | Feed <2/mnt; ≥1 penahanan tanpa alasan | 8 |
| T0.5 | Encoder stabil pasca-P16 (0.5) | 0 kematian tak direncanakan/72 jam; 0 orphan; watchdog R3b menyelamatkan ≥1 stall (feeder restart + encoder tetap hidup) | ≥1 kematian; ≥1 orphan; watchdog mencatat stall tapi encoder tetap mati | 3, 4 |
| T0.6 | Janitor status-only (0.6) | 0 GB file channel CONNECTED terhapus berlabel "unpaired" dalam audit 24 jam (audit pasca-V2-004) | ≥1 file CONNECTED terhapus/terlabel salah | — (R6) |
| T0.G | Gerbang Fase 0 | Baris 1, 2, 5, 6, 7, 8, 9, 11 membaik/tetap 72 jam; 0 error class baru | Satu baris memburuk; ≥1 error class baru | 1,2,5,6,7,8,9,11 |

**Catatan T0.5:** kriteria "watchdog menyelamatkan stall" adalah
[PR] — berasal dari temuan iterasi 2 (watchdog fired 08:09:42 tapi
encoder tetap mati 6 dtk kemudian). Bila tidak ada stall alami
dalam 72 jam, uji ini dicatat "TIDAK TERUJI (tidak ada stall)",
bukan lolos.

---

## 3. Uji provisioning VPS 8 (Fase 1.1)

| ID | Uji | Lolos bila | Gagal bila |
|---|---|---|---|
| T1.1-A | Spesifikasi box | CPU 8 vCPU, RAM 24 GB, disk 300 GB terbaca sistem (lscpu/free/df); OS Ubuntu 24.04; timezone Asia/Jakarta | Spes tidak cocok; timezone bukan WIB |
| T1.1-B | Bandwidth port | [PR] Uji paralel up/down mencapai ≥420 Mbps sustained (sejajar cap admission 70% port 600 Mbps) — ambang ini USUL, belum diputuskan owner [OD]. Minimal: angka terukur + dicatat apa adanya | Tidak terukur; atau <420 Mbps tanpa penjelasan (→ eskalasi, bukan lolos diam-diam) |
| T1.1-C | Hardening | SSH key-only (password login ditolak); UFW aktif (22, 80, 443 + tailscale0); service non-root | Satu cek gagal |
| T1.1-D | Konektivitas privat | Node Tailscale ONLINE; SSH via Tailscale OK dari mesin Muse; latensi Master↔VPS 8 tercatat | Node tidak ONLINE; SSH gagal |
| T1.1-E | Dependensi | python3.12, ffmpeg, Caddy, cloudflared, Tailscale, WARP, Node ≥22, git — masing-masing `--version` OK (bukan rc rantai) | Satu paket gagal verifikasi versi |

---

## 4. Uji live canary (Fase 1.2–1.3)

| ID | Uji | Lolos bila | Gagal bila |
|---|---|---|---|
| T1.2-A | Deploy live service | Unit `masjavas-live` terpasang + governor lokal terkonfigurasi (admission 420 Mbps); release manifest cocok dengan Master; worker terdaftar + versi dilaporkan | Manifest tidak cocok; registrasi gagal |
| T1.3-A | RTMP direct | Stream canary mengudara ke YouTube langsung dari Live-01 (bukan lewat Master/tunnel); journal "berhasil mengudara" + PID tercatat | RTMP lewat Master; tidak mengudara |
| T1.3-B | FFmpeg hidup vs broadcast live | Broadcast terverifikasi dari sisi YouTube (bukan hanya proses FFmpeg hidup) — bedakan keduanya secara eksplisit | Hanya proses yang dicek; atau broadcast tidak live padahal FFmpeg hidup |
| T1.3-C | Reconnect | Encoder dimatikan (uji) → supervisor pulih otomatis ≤2 menit; reconnect_count tercatat; 0 intervensi manual | Tidak pulih otomatis; atau pulih >5 menit |
| T1.3-D | TX governor | Stream baru ditolak saat TX total + bitrate >420 Mbps (uji dengan penawaran stream tiruan/melebihi batas); penolakan tercatat alasannya | Stream diterima di atas batas; atau ditolak tanpa alasan tercatat |
| T1.3-E | Master responsif | healthz p95 <100 ms selama canary berjalan; publish normal | healthz p95 ≥100 ms; publish terganggu |
| T1.3-F | Failover darurat ke Master | [PR] 1 stream canary dipindahkan kembali ke Master (drain Live-01 → start Master) dalam ≤5 menit, tanpa duplikat supervisor. Kapasitas darurat "±20 stream" [DF — PLAN §8] TIDAK diuji penuh di sini (butuh 20 stream) — dicatat sebagai [OQ] | Duplikat supervisor; atau pemindahan >15 menit |
| T1.3-G | Stabilitas 72 jam | Stream canary stabil 72 jam; 0 kematian tak direncanakan; 0 orphan; scorecard baris 3, 4, 11, 12 membaik/tetap | ≥1 kematian; ≥1 orphan; satu baris memburuk |

---

## 5. Uji DR + watchdog (Fase 1.4)

| ID | Uji | Lolos bila | Gagal bila |
|---|---|---|---|
| T1.4-A | DR replica sync | Sync malam lengkap; ukuran + checksum salinan cocok dengan sumber; DB backup `integrity_check` OK | Checksum tidak cocok; integrity_check gagal |
| T1.4-B | Watchdog + alarm | Heartbeat Master yang sengaja diputus → alarm dalam ≤90 dtk + 1 menit toleransi (total ≤2,5 mnt) | Alarm tidak berbunyi; atau >5 menit |
| T1.4-C | **Restore drill #1** | Aplikasi boot dari salinan DR di lingkungan terisolasi; DB terbaca (row count = baseline); 1 alur baca dashboard jalan. **RPO/RTO yang TERCAPAI dicatat apa adanya** (target ≤5 mnt/≤30 mnt adalah TARGET, bukan janji) | Boot gagal; DB tidak terbaca; atau drill tidak dijalankan (→ DR belum "selesai") |

---

## 6. Uji prep worker canary → penuh (Fase 1.5–1.6)

| ID | Uji | Lolos bila | Gagal bila |
|---|---|---|---|
| T1.5-A | Registrasi + klaim API | Live-01 terdaftar sebagai prep worker; klaim + heartbeat tercatat; SQLite tetap hanya di Master (0 koneksi DB dari VPS 8) | Koneksi DB dari VPS 8 terdeteksi; klaim gagal |
| T1.5-B | Ingestion canary (1 job) | 1 job ingestion COMPLETED end-to-end via API (klaim → proses → submit); 0 gangguan live selama job berjalan | Job gagal; live terganggu |
| T1.5-C | **Stage 2 (2 slot) + reassemble BERSAMAAN** | 2 job Stage 2 DONE + ≥1 job reassemble berjalan bersamaan; puncak CPU prep terukur ≤ anggaran (Stage 2 cgroup 4 + reassemble ±1 + live ±2 ≈ 7 core; headroom ±1 core tersisa); RAM prep ≤6 GB; **live tidak terganggu** (TX live stabil, 0 kematian encoder selama uji); 0 antrean Stage 2 yang tersangkut karena CPU | Puncak CPU >7 core (headroom habis); live terganggu; ≥1 job Stage 2 gagal karena perebutan CPU |
| T1.5-D | Guard lokal per box | Load gate: konkurensi prep turun saat load1 box ≥2,0/≥3,0 (uji beban); disk gate: klaim baru ditolak saat disk bebas <40 GB (uji ambang); setiap aksi tercatat alasannya | Guard tidak bereaksi; atau bereaksi tanpa log alasan |
| T1.5-E | TX throttle prep | Unggahan Drive prep dijeda saat TX live >50% (uji beban TX); lanjut otomatis saat TX turun | Throttle tidak bekerja; atau prep terus mengunggah saat TX live tinggi |
| T1.5-F | Worker crash | Worker prep dimatikan paksa (uji) → 0 duplikat job; job yang terinterupsi kembali ke antrean (lease habis) dan diselesaikan worker lain/Master; hasil tidak ganda | Duplikat job; hasil ganda; job hilang tanpa jejak |
| T1.5-G | Master disconnect | Koneksi Master↔VPS 8 diputus (uji) → prep worker pause klaim baru (tanpa error berulang); stream live TETAP jalan; sambung lagi → klaim lanjut normal; heartbeat stale >90 dtk memicu alarm | Live ikut mati; klaim berulang error; alarm tidak berbunyi |
| T1.5-H | Fencing + idempotency | Job dengan fencing token basi ditolak; submit ganda dengan idempotency key yang sama menghasilkan 1 hasil (tidak ganda) | Token basi diterima; hasil ganda |
| T1.5-I | Observasi 72 jam canary | Scorecard membaik/tetap; 0 error class baru; 0 gangguan live | Satu baris memburuk; ≥1 error class baru |
| T1.6-A | Prep penuh | Ingestion RUNNING ≥3 job saat backlog ada; Stage 2 4 slot jalan; throughput Stage 2/24 jam ≥ baseline (1.011/7 hari ≈ 144/hari) dan naik terukur; TX live tidak terganggu | Ingestion <3 saat backlog ada; throughput turun; live terganggu |
| T1.6-B | Broadcast window | [PR] Ingestion Master jeda 17:00–22:30 WIB; ingestion VPS 8 JALAN TERUS dalam jendela itu (guard broadcast hanya di Master) | VPS 8 ikut jeda; atau Master tidak jeda |

**Catatan T1.5-C:** uji gabungan ini WAJIB karena addendum
reassemble membuktikan anggaran lama (±6 core) salah — canary
Stage 2 sendirian TIDAK mewakili beban puncak nyata. [VF]
**Catatan T1.6-A:** target throughput "≥ baseline" adalah batas
minimum; "naik terukur" berarti angka 24 jam pasca-1.6 > angka
7 hari sebelumnya — tanpa target persen yang dikarang. [PR]

---

## 7. Uji guard lokal per box (lintas fase)

| ID | Guard | Box | Ambang uji | Lolos bila |
|---|---|---|---|---|
| G-1 | Slot render | Master | 2 + 1 longform | Job ke-4 antre (bukan bunuh) |
| G-2 | Kuota vault | Master | 8.400 unit (5 ch/vault) | Publish vault itu ditunda, vault lain jalan |
| G-3 | Load gate ingestion | Per box | load1 ≥2,0/≥3,0 | Konkurensi box itu saja yang turun |
| G-4 | Disk gate ingestion | Per box | <40 GB bebas | Klaim baru box itu saja yang ditolak |
| G-5 | Broadcast window | Master SAJA | 17:00–22:30 WIB | Lihat T1.6-B |
| G-6 | TX governor live | VPS 8 | TX total >70% port (420 Mbps) | Stream baru ditolak; TX prep dijeda |
| G-7 | Cgroup Stage 2 | VPS 8 | 4 core | Render antre, tidak melampaui cap |
| G-8 | Kuota disk co-tenant | VPS 8 | Sisakan ≥50 GB bebas | Co-tenant berhenti mengisi |

Semua ambang di atas: [DF — PLAN §5.4]. Uji G-3/G-4/G-6/G-7
membutuhkan beban sintetis — siapkan skrip beban SEBELUM uji,
jangan bebani produksi secara dadakan. [PR]

---

## 8. Gerbang Fase 1 (akhir)

Scorecard SEMUA 13 baris membaik atau tetap selama 72 jam
pasca-1.6; Master tidak lagi menjalankan live reguler; 0 error
class baru. [DF — PLAN §6] Lolos → VPS 8 dinyatakan SIAP
OPERASI PENUH (bukan sekadar "terpasang").

## 9. Log uji (diisi saat eksekusi)

| Tanggal/WIB | ID uji | Pelaksana | Hasil terukur | LOLOS/GAGAL/TIDAK TERUJI | Tindak lanjut |
|---|---|---|---|---|---|
| | | | | | |

## 10. Pertanyaan terbuka (OQ) rencana uji ini

1. **[OQ-201]** Ambang lolos bandwidth T1.1-B (usul ≥420 Mbps).
2. **[OQ-202]** Bagaimana memverifikasi "broadcast benar-benar live"
   dari sisi YouTube secara otomatis (T1.3-B) — cek manual vs API?
3. **[OQ-203]** Kapasitas darurat Master ±20 stream — uji penuh
   butuh 20 stream; kapan/di mana diuji?
4. **[OQ-204]** Skrip beban sintetis untuk G-3/G-4/G-6/G-7 — siapa
   yang menulis, dan di mana dijalankan agar tidak mengganggu produksi?
5. **[OQ-205]** Kriteria "watchdog menyelamatkan stall" (T0.5) bila
   tidak ada stall alami dalam 72 jam — uji stall buatan, atau terima
   "TIDAK TERUJI"?
