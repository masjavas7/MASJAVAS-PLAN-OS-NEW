# REPOSITORY ENGINEERING AUDIT & IMPROVEMENT SPRINT

**Status:** AKTIF — dimulai 22 Nov 2026 13:02 WIB.
**Pemicu:** review repo eksternal (dibaca user dari GitHub publik) +
mandat 24/7 "bangun repo sampai siap deploy" (22 Nov 12:30).
**Tujuan:** beralih dari *documentation building* ke *readiness
engineering* — menutup risiko yang ada, bukan menambah dokumen.
**Ruang lingkup:** dokumen di repo saja. Dokumen beku V2.0 TIDAK
disentuh; TANPA perubahan produksi (Level A — otonom penuh).

## Aturan sprint (mengikat)

1. **Verifikasi sebelum memperbaiki** — setiap klaim review dicek ke
   file repo sebelum diterima. Klaim yang tidak terbukti dicatat
   sebagai tidak terbukti, bukan diperbaiki.
2. **Status dari registry** — `STATUS_REGISTRY.yaml` adalah satu-satunya
   sumber status tiket + gerbang. README, dashboard, dan tracker tiket
   menampilkan ringkasan registry, bukan status manual.
3. **Kualitas evidence > persentase progress** — jangan biarkan dashboard
   mengatakan tiket selesai bila bukti yang mendasarinya berasal dari
   host yang salah atau belum terverifikasi.
4. **Optimasi, bukan penggantian** (prinsip user 12:40) — yang sudah
   solid dipertahankan; baseline (206 PUBLISHED/hari, 0 FAILED/BLOCKED,
   p90 30,2 mnt) adalah lantai.
5. **Label klaim** — setiap klaim: VERIFIED FACT / DOCUMENTED FACT NOT
   YET REVALIDATED / ASSUMPTION / PROPOSAL / OPEN QUESTION / OWNER
   DECISION REQUIRED. Jangan mengarang harga, benchmark, atau angka.
6. **Jangan kunci angka final sebelum benchmark** — concurrency, reserve,
   dan threshold adalah PROPOSAL sampai V2-008 mengukurnya.

## Temuan review — hasil verifikasi

| # | Klaim review | Hasil verifikasi |
|---|---|---|
| F1 | Status V2-001–V2-004 tidak konsisten antarfile | **TERKONFIRMASI.** README l.37 "V2-001–V2-004 lengkap" ambigu (tiket lengkap vs pekerjaan selesai); dashboard l.72 "V2-001 READY_FOR_REVIEW; menunggu Q5" BERTENTANGAN dengan l.84 "V2-001 EXECUTED (Q5/Q6 ditutup atas asumsi)"; angka "3/10 selesai dieksekusi" mencampur EXECUTED + READY_FOR_REVIEW. |
| F2 | Aturan rollback satu-metrik terlalu sensitif | **TERKONFIRMASI.** Rollback runbook l.18 + acceptance plan l.7–8: "Satu metrik scorecard memburuk = gerbang gagal = rollback" — tanpa threshold, durasi, atau klasifikasi. Spike CPU 3% sesaat dari backup akan memicu rollback. |
| F3 | VPS 8 headroom hanya ±1 core (7 dari 8 vCPU) | **TERKONFIRMASI** (addendum reassemble 12:10). Perlu capacity model + reserve + admission control. |
| F4 | Gerbang pembelian hanya berbasis jumlah stream | **TERKONFIRMASI** (plan §6). Perlu demand + operational risk + verified cost. |

## 8 item sprint (urutan roadmap review)

### P0

**1. Status consistency (STATUS_REGISTRY.yaml)** — satu sumber status
terstruktur untuk tiket + gerbang. Kriteria selesai: registry ada;
README + dashboard + tracker memakai status yang sama; kontradiksi
F1 hilang; V2-001 tercatat EXECUTED dengan `evidence_valid_for_target:
false` + reopen condition. **Catatan:** status memakai kosakata
direktif (NOT_STARTED/IN_PROGRESS/BLOCKED/READY_FOR_REVIEW/APPROVED/
EXECUTED/VERIFIED/ROLLED_BACK/CANCELLED) — TIDAK membuat status baru
seperti `EXECUTED_WITH_LIMITATION`; keterbatasan dicatat di field
`limitation`, bukan di nama status. `ticket_status` (COMPLETE/STUB)
dipisah dari `work_status` agar "lengkap" tidak lagi ambigu.

**2. CPU resource capacity model (`VPS8_RESOURCE_CAPACITY_MODEL.md`)**
— budget CPU per workload, reserve Live + OS/daemon, batas concurrent
reassemble + Stage 2, admission control berbasis CPU pressure + load,
simulasi normal/peak/failure. Kriteria selesai: model menghitung
ulang total puncak (termasuk koreksi reassemble) dan menyatakan
headroom tersisa; angka concurrency = PROPOSAL sampai V2-008.

**3. Purchase gate (`VPS8_PURCHASE_GATE.md`)** — keputusan pembelian
berdasarkan demand + operational risk + verified cost (bukan hanya
jumlah stream): CPU pressure, RAM, I/O wait, egress, backlog,
publish delay, SQLite lock, live stability, admission rejection.
Kriteria selesai: gerbang terukur dengan threshold + sumber data
per sinyal; tetap mengikat cap $34,67/bln.

### P1

**4. Orchestrator API contract (`ORCHESTRATOR_API_CONTRACT_V1.md`)**
— 7 endpoint (register, heartbeat, claim, job heartbeat, complete,
fail, release) dengan request/response schema, auth, idempotency,
timeout, retry, error code, lease ownership, fencing token, version
compatibility. Kriteria selesai: kasus uji stale-worker (A putus →
B klaim → A kembali mengirim hasil lama → ditolak fencing token)
terdefinisi sebagai kasus uji wajib.

**5. Failure injection test plan
(`VPS8_FAILURE_INJECTION_TEST_PLAN.md`)** — 11 skenario (Master API
mati, Tailscale putus, worker restart saat lease aktif, SQLite
terkunci, disk penuh, FFmpeg mati, output sebagian, publish timeout,
stale worker, backup gagal, restore gagal). Kriteria selesai: tiap
skenario punya kondisi awal, simulasi, hasil yang diharapkan, metrik,
kriteria lulus/gagal, prosedur pemulihan.

**6. Rollback threshold revision** — ganti aturan satu-metrik dengan
3 kategori: hard safety breach (stop/rollback darurat), SLO
degradation (hold + investigasi + keputusan owner), transient noise
(observasi). Kriteria selesai: rollback runbook + acceptance plan
diperbarui dengan threshold, durasi, dan jumlah sampel per metrik;
aturan lama dicatat SUPERSEDED dengan alasan.

**7. TCO model (`MASJAVAS_TCO_MODEL.md` + update cost register)** —
harga sebelum/sesudah pajak, setup fee, renewal, object storage,
retensi backup, snapshot, egress, domain/layanan pendukung, biaya
API/provider, cadangan tak terduga; 3 skenario (minimum/expected/
worst case) vs cap $34,67/bln. Kriteria selesai: setiap angka
berlabel; biaya yang belum dihitung tetap BELUM DIHITUNG (bukan
diasumsikan nol atau tercakup cap).

**8. SLO & error budget (`MASJAVAS_SLO_ERROR_BUDGET.md`)** — live
availability, publish success rate, job completion rate, worker
availability, API latency, claim latency, recovery time, backup
freshness; tiap SLO punya target, window, cara hitung, sumber data,
alert threshold, pemilik keputusan. Kriteria selesai: target angka
hanya dari baseline yang dapat dipercaya; tanpa baseline = OPEN
QUESTION, bukan target karangan.

## P2 (setelah P0–P1)

**9. Readiness automation** — validator registry/dependency/dokumen/
dashboard (mis. GitHub Actions untuk validasi saja — BUKAN deploy
produksi). Dirancang setelah registry stabil; bukan bagian sprint ini.

## Yang TIDAK dilakukan (dari review, disetujui)

- Jangan tambah VPS 8 untuk mengatasi semua bottleneck — masalah kode,
  guard, atribusi, dan job lifecycle tidak diperbaiki infrastruktur.
- Jangan migrasi ke PostgreSQL/Kafka/Celery sebelum bukti mengharuskan.
- Jangan buat dokumentasi paralel tanpa registry bersama.
