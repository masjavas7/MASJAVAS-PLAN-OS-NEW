# DESAIN COCKPIT SENTINEL 6-LANE SADAR-NODE + JANITOR OTONOM

**Tanggal:** 24 November 2026, 03:47 WIB
**Diminta oleh:** Owner (Abdul Aziz) — "buat agar Sentinel 6-Lane
Orchestration Cockpit juga memperlihatkan di VPS mana dia bekerja,
sehingga monitoring jadi lebih tepat dan Janitor bisa bekerja otonom
sebagaimana fungsinya"
**Status:** DESAIN — implementasi Fase 1 didelegasikan ke subagent
**Halaman target:** https://masjavastube.my.id/telemetry (panel Sentinel)

---

## 1. DIAGNOSIS — AMBIGUITAS TERBUKTI DI KODE (03:45–03:47 WIB)

Cockpit mengambil data dari `GET /api/telemetry/sentinel-cockpit` →
`probe_all_lanes()` → 6 probe di `core/telemetry/lane_probes.py`.
Semua probe berjalan di proses web Master dan membaca DB Master +
psutil LOKAL Master. Hasil audit per-lane:

| Lane | Node sebenarnya | Yang dibaca probe | Status |
|------|----------------|-------------------|--------|
| **L1 Publish** | Master (VPS 4+) | DB Master `operational_tasks` | ✅ BENAR — publish memang di Master |
| **L2 Ingestion** | VPS 8 (prep worker) | DB Master `ingestion_jobs` | ⚠️ Antrean benar (DB bersama), tapi PELAKSANA di VPS 8 tidak terlihat. Bila ingestion macet, cockpit tidak bisa membedakan antrean Master vs worker VPS 8. |
| **L3 Live** | VPS 8 (live-01) | DB Master (jumlah task) + **psutil ffmpeg LOKAL Master** | ❌ **RUSAK**: `active_streams` menghitung ffmpeg di Master = **selalu 0** (semua encoder di VPS 8). Cockpit menampilkan "Stream Aktif: 0" + WARNING padahal 3 stream sedang live. Ini ambiguitas utama. |
| **L4 LoopGen** | VPS 8 (Stage 2 prep) | DB Master `loopgen_jobs` + `loopgen_image_jobs` | ⚠️ Antrean benar (DB bersama), tapi eksekusi Stage 2 di VPS 8 tidak terlihat. |
| **L5 Janitor** | Master SAJA | DB Master `file_leases` + **psutil disk LOKAL Master** | ❌ **BUTA VPS 8**: disk VPS 8 (46/290 GB) + file VPS 8 tidak terlihat. Janitor tidak bisa membersihkan VPS 8 secara otonom — ia tidak tahu kondisi disk di sana. Bila disk VPS 8 penuh, tidak ada alert, tidak ada cleanup. |
| **L6 Vault** | Agnostik node (logis) | DB Master `oauth_masters` | ✅ BENAR — vault bukan terikat node. |

**Efek samping:** health score (`compute_health_score`) memakai probe
lane — WARNING palsu L3 ikut menekan skor kesehatan sistem.

---

## 2. SARAN & IDE PERBAIKAN (di luar cockpit)

| # | Saran | Alasan | Prioritas |
|---|-------|--------|-----------|
| S1 | **Cockpit sadar-node (tugas ini)** | L3 salah total, L5 buta VPS 8 — monitoring tidak bisa dipercaya tanpa ini | P0 |
| S2 | **Janitor per-node (Fase 2)** | Janitor harus DIEKSEKUSI di tiap node (worker janitor di VPS 8) agar cleanup otonom benar-benar jalan di sana, bukan sekadar terlihat | P1 |
| S3 | **Alert menyebut nama node** | Alert "disk > 80%" ambigu — harus "live-01 disk > 80%" agar operator tahu ke mana harus bertindak | P1 |
| S4 | **Registry node tunggal** | Pemetaan lane→node jangan hardcode selamanya — baca dari tabel `live_nodes` (desain arsitektur multi-node 23 Nov §4) begitu tabel itu ada | P2 |
| S5 | **Slot prep VPS 8 di L2/L4** | Tampilkan slot Stage 2/ingestion aktif (sudah ada di node telemetry) berdampingan dengan antrean, agar jelas kapasitas vs beban | P2 |
| S6 | **Label Host = "Master (VPS 4 Plus)"** | Section `host` di cockpit adalah metrik Master — beri label eksplisit agar tidak dikira metrik gabungan | P2 |

---

## 3. DESAIN FASE 1 — COCKPIT SADAR-NODE

### 3.1 Prinsip
1. **Aditif.** Semua kunci respons + tampilan yang ada TIDAK DIUBAH.
   Hanya tambah kunci `nodes` per lane + badge di kartu + sumber data
   yang benar untuk L3/L5.
2. **DB Master tetap sumber kebenaran antrean.** Antrean L1/L2/L4 tetap
   dari DB Master (sudah benar — antrean bersama).
3. **Data node dari `node_telemetry_samples`.** L3/L5 membaca sampel
   node terbaru (infrastruktur dari perbaikan telemetry 03:31) — TIDAK
   ada SSH/query langsung ke VPS 8 dari proses web.
4. **Degradasi anggun.** Bila tidak ada sampel node < 90 dtk: L3 fallback
   ke psutil lokal + tandai `node_stale: true`; L5 tampilkan hanya node
   Master + tandai VPS 8 "tidak ada data".
5. **Satu model pemetaan.** Pemetaan lane→node didefinisikan SATU KALI
   (konstanta di `lane_probes.py` atau modul baru `core/telemetry/nodes.py`)
   dan dipakai probe + endpoint + template.

### 3.2 Model pemetaan lane → node

```python
# core/telemetry/nodes.py (baru) ATAU konstanta di lane_probes.py
NODE_DISPLAY = {
    "master":  "VPS 4 Plus (Master)",
    "live-01": "VPS 8 (live-01)",
}
LANE_NODES = {
    "publish":   ["master"],            # L1 — worker --lane publish di Master
    "ingestion": ["live-01"],           # L2 — prep worker VPS 8
    "live":      ["live-01"],           # L3 — encoder di VPS 8
    "loopgen":   ["live-01"],           # L4 — Stage 2 di VPS 8
    "janitor":   ["master", "live-01"], # L5 — per-node (tiap node bersihkan sendiri)
    "vault":     [],                    # L6 — agnostik node
}
```

### 3.3 Perbaikan probe

#### L3 Live — baca stream dari node telemetry
```python
# probe_lane_live: ganti hitungan psutil lokal dengan:
# 1. SELECT streams_active, streams_total FROM node_telemetry_samples
#    WHERE node_id='live-01' ORDER BY ts DESC LIMIT 1
# 2. Bila sampel ada dan age < 90 dtk: active_streams = streams_active
# 3. Bila tidak ada/stale: fallback psutil lokal + node_stale=True
# 4. live_tasks_configured tetap dari DB Master (sumber kebenaran jumlah task)
# 5. details_json tambah: node_id, node_stale, encoder_deaths_1h, rtmp_timeouts_1h
```
Health status: HEALTHY bila active > 0; WARNING bila configured > 0
dan active = 0 (dengan data node yang benar, bukan 0 palsu);
IDLE bila configured = 0.

#### L5 Janitor — disk per-node
```python
# probe_lane_janitor: tambah pembacaan node telemetry:
# 1. Disk Master tetap dari psutil lokal (node master).
# 2. Untuk tiap node di node_telemetry_samples (sampel terbaru < 90 dtk):
#    ambil disk_free_gb + disk_percent.
# 3. details_json tambah: nodes: {"master": {free_gb, used_pct},
#    "live-01": {free_gb, used_pct, stale: bool}}
# 4. Health status = TERBURUK di semua node (threshold sama: CRITICAL
#    bila free < 15 GB atau used > 90%; WARNING bila free < 30 GB
#    atau used > 80%).
# 5. file_leases tetap dari DB Master (lease dicatat terpusat).
```

#### L1/L2/L4/L6 — tanpa perubahan logika
Hanya tambah kunci `nodes` (dari LANE_NODES) ke hasil probe.
Antrean + active_workers tidak berubah.

### 3.4 Endpoint `/api/telemetry/sentinel-cockpit`

Tambahkan ke tiap entri `lane_summary`:
```json
"live": {
  "health_status": "HEALTHY",
  "queue_depth": 0,
  "active_workers": 3,
  "error_rate": 0.0,
  "details": { "...": "...", "node_id": "live-01", "node_stale": false },
  "nodes": ["live-01"],
  "node_labels": ["VPS 8 (live-01)"]
}
```
Tambahkan kunci `host.node_id = "master"` dan
`host.node_label = "VPS 4 Plus (Master)"` (S6).

### 3.5 Template `telemetry.html` — badge node per kartu lane

Setiap kartu L1–L6 mendapat badge node di samping badge status:
```
┌──────────────────────────────────────────────┐
│ L3 · Live Stream        [HEALTHY] [VPS 8]    │
│ Stream Aktif: 3  •  Terjadwal: 3             │
│ Kematian encoder: 18/jam • RTMP timeout: 10/jam │
└──────────────────────────────────────────────┘
┌──────────────────────────────────────────────┐
│ L5 · Janitor            [WARNING] [Master+VPS 8]│
│ File Lease: 12 Aktif                         │
│ Disk Bebas: Master 18,6 GB • VPS 8 243,6 GB  │
└──────────────────────────────────────────────┘
```
- Badge node: warna berbeda per node (Master = slate/biru, VPS 8 = emerald/teal)
  agar mudah dibedakan sekilas.
- L3 tampilkan juga kematian encoder/jam + RTMP timeout/jam (dari details).
- L5 tampilkan disk per-node (dari details.nodes).
- JS `loadSentinelCockpit()` diperluas: baca `lane.nodes` /
  `lane.node_labels` + details baru. Pola fetch yang sama.
- Bila `node_stale: true`, badge node menampilkan "STALE" (kuning).

### 3.6 Janitor otonom — Fase 2 (DESAIN SAJA, tidak diimplementasi di tugas ini)

Fase 1 membuat Janitor MELIHAT per-node. Otonomi penuh butuh EKSEKUSI
per-node. Desain Fase 2 (untuk persetujuan owner terpisah):
1. **Janitor worker di VPS 8** — service `masjavas-janitor.service` yang
   menjalankan pembersihan lokal (hapus file Stage 2 lama, rotate log,
   bersihkan workspace sementara) dengan aturan yang sama seperti Janitor
   Master, dan melaporkan hasil via node-report (tambah kolom
   `janitor_last_run`, `janitor_freed_mb`).
2. **Aturan pembersihan terpusat** — threshold + retensi didefinisikan di
   Master (DB `system_settings`), dibaca janitor tiap node. Satu kebijakan,
   eksekusi lokal.
3. **Koordinasi lease** — file_leases tetap terpusat di DB Master; janitor
   VPS 8 TIDAK menghapus file yang masih ter-lease (cek via API Master
   sebelum hapus, atau lease disinkronkan ke DB VPS 8 via DR sync).
4. **Safeguard** — janitor TIDAK PERNAH menghapus: file yang sedang dipakai
   encoder aktif, backup DR, file < 1 jam. Dry-run dulu, log semua yang
   dihapus, alert bila yang dibebaskan > 10 GB dalam 1 run.

---

## 4. RENCANA IMPLEMENTASI FASE 1 (untuk subagent implementer)

### Urutan kerja
1. **Backup:** `cp` file yang akan diubah (`lane_probes.py`,
   `api/routes/telemetry.py`, `telemetry.html`, + modul baru bila ada)
   ke `/opt/masjavas-os/data/backups/cockpit-node-20261124-<timestamp>/`.
   TIDAK perlu backup DB (tidak ada perubahan skema — hanya baca
   `node_telemetry_samples` yang sudah ada).
2. **Model node:** buat `core/telemetry/nodes.py` (atau konstanta di
   `lane_probes.py` — pilih yang paling sesuai pola kode) berisi
   NODE_DISPLAY + LANE_NODES.
3. **Probe L3:** ubah `probe_lane_live` membaca `node_telemetry_samples`
   (sampel terbaru live-01) + fallback psutil + `node_stale`.
4. **Probe L5:** ubah `probe_lane_janitor` menambah disk per-node dari
   `node_telemetry_samples` + health terburuk.
5. **Semua probe:** tambah kunci `nodes` ke hasil (dari LANE_NODES).
   Pastikan `probe_all_lanes()` meneruskan kunci baru.
6. **Endpoint:** tambah `nodes` + `node_labels` ke `lane_summary`,
   `host.node_id` + `host.node_label`.
7. **Template:** badge node per kartu + detail L3 (kematian/RTMP) +
   detail L5 (disk per-node) + logika STALE di JS.
8. **Tes:** `py_compile` semua file; jalankan test suite telemetry
   (catat bila ada gagal pre-existing — buktikan dengan file backup).
9. **Deploy:** restart web SATU KALI (cek restart terakhir dulu; web
   terakhir restart 03:31:29 WIB — patuhi max 1 restart/15 menit).
   Verifikasi pasca-restart (§5).
10. **Commit + push** (Master): hanya file yang diubah tugas ini.
    Pesan commit jelas. JANGAN commit file Antigravity.
11. **Rollback (bila gagal):** restore file dari backup + restart web 1×.

### Larangan
- JANGAN ubah perilaku endpoint/probe yang ada selain yang dirancang di §3.
- JANGAN ubah skema DB.
- JANGAN sentuh live/worker/prep service. JANGAN restart apa pun kecuali web 1×.
- JANGAN cetak stream key / token / URL RTMP di log/laporan/chat.
- JANGAN query langsung ke VPS 8 dari proses web (semua data node lewat
  `node_telemetry_samples`).
- `lane_probes.py` kemungkinan root-owned — `sudo chown ubuntu:ubuntu`
  dulu bila perlu (pola AGENTS.md).

---

## 5. VERIFIKASI (wajib sebelum lapor selesai)

### Pra-deploy
- [ ] `py_compile` semua file Python yang diubah: OK.
- [ ] Test suite telemetry: hijau (atau gagal pre-existing terbukti).

### Pasca-deploy (web restart)
- [ ] Web responsif: `/` 200 < 2 dtk.
- [ ] `GET /api/telemetry/sentinel-cockpit` (via browser/cek): tiap lane
      mengandung `nodes` + `node_labels` yang benar (L1 master, L2/L3/L4
      live-01, L5 keduanya, L6 kosong).
- [ ] **L3 `active_workers` = 3** (bukan 0) — bukti perbaikan utama.
- [ ] **L3 `health_status` = HEALTHY** (bukan WARNING palsu).
- [ ] **L5 details.nodes** mengandung master + live-01 dengan angka disk
      yang cocok dengan node telemetry terbaru.
- [ ] Badge node tampil di 6 kartu lane (verifikasi template/HTML).
- [ ] Tidak ada error baru di journal web 10 menit pasca-restart.
- [ ] Publish tidak terganggu.

### Observasi
- [ ] Health score sistem membaik (WARNING palsu L3 hilang) — catat
      skor sebelum/sesudah bila bisa dibandingkan.
