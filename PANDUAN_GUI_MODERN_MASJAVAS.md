# PANDUAN GUI MODERN — MASJAVAS TUBE OS

| | |
|---|---|
| **Tanggal** | 25 Nov 2026 |
| **Status** | BERLAKU — semua UI baru (termasuk fitur edit jadwal) wajib mengikuti panduan ini |
| **Alasan** | Audit 25 Nov 2026: `live_dashboard.html` memakai 10 warna aksen, teks didominasi 9–11px (114 pemakaian vs 8 pemakaian ≥14px), tombol 5–8 buah per baris, tanpa token/komponen bersama. Hasilnya: ramai, sulit dipindai, tidak konsisten antar halaman |

---

## 1. Prinsip (4 kata)

**Tenang, jelas, konsisten, cepat.**
Dashboard operasi dibaca ratusan kali sehari dalam keadaan lelah/jam 3 pagi. Setiap elemen harus langsung terbaca maknanya tanpa berpikir. Dekorasi yang tidak membantu keputusan = dihapus.

---

## 2. Token desain (satu-satunya sumber kebenaran)

Definisikan di `tailwind.config` di `base.html`. DILARANG memakai warna/ukuran mentah di luar token ini pada UI baru; UI lama dimigrasikan bertahap saat disentuh.

### 2.1 Warna — 1 aksen + 4 status (bukan 10 warna)

| Token | Nilai | Dipakai untuk |
|---|---|---|
| `bg` | slate-950 `#020617` | Latar halaman |
| `surface` | slate-900 `#0f172a` | Kartu, panel, baris tabel |
| `surface-2` | slate-800 `#1e293b` | Hover, elemen terangkat, input |
| `border` | slate-800 `#1e293b` | Garis kartu/pemisah (1px, tanpa bayangan ganda) |
| `text` | slate-100 `#f1f5f9` | Teks utama |
| `muted` | slate-400 `#94a3b8` | Teks sekunder, label, timestamp |
| `faint` | slate-500 `#64748b` | Placeholder, teks tersier |
| `accent` | emerald-500 `#10b981` | SATU-SATUNYA aksen brand/aksi utama |
| `accent-soft` | emerald-500/15 bg + emerald-400 text | Badge/hover aksen |
| `success` | emerald-400 | LIVE, berhasil |
| `warning` | amber-400 | RECONNECTING, perlu perhatian |
| `danger` | rose-500 | ERROR, STOPPED, hapus |
| `info` | sky-400 | Info netral, IDLE |
| `on-accent` | white | Teks di atas tombol aksen |

**Aturan keras:**
- Rose HANYA untuk bahaya/error — jangan pernah untuk tombol utama (tombol "Buat Live" yang sekarang gradasi rose→red harus jadi `accent`).
- Indigo, cyan, violet, teal, purple, lime, orange, pink: DILARANG pada UI baru (sisa pemakaian lama dimigrasikan saat halaman disentuh).
- Gradasi: DILARANG (kecuali elemen brand di landing page). Tombol = warna datar + bayangan halus.
- `backdrop-blur` + transparansi `/60`: DILARANG pada kartu data (menyulitkan baca). Kartu = `surface` solid + `border` 1px.

### 2.2 Tipografi — 3 ukuran UI + mono untuk data

| Token | Ukuran/berat | Dipakai untuk |
|---|---|---|
| `title` | 18px / 600 | Judul halaman & kartu utama |
| `body` | 14px / 400–500 | Teks utama, label field, isi tabel |
| `small` | 12px / 400 | Teks sekunder, keterangan, timestamp |
| `mono` | 12px / JetBrains Mono | ID, stream key (mask), angka telemetri, timestamp teknis |

**Aturan keras:**
- DILARANG teks 9–11px pada UI baru (yang lama: naikkan ke 12px saat disentuh).
- DILARANG `font-extrabold` untuk judul biasa (maks 600/semibold; 800 hanya angka hero statistik).
- Angka yang dibandingkan (CPU, Mbps, jumlah) memakai `font-mono` + `tabular-nums` agar tidak goyang saat berubah.
- Judul halaman: 1 baris (judul + aksi kanan), tanpa subjudul panjang.

### 2.3 Spasi, radius, bayangan

- Grid 8px: padding kartu `p-4` (16px) atau `p-5` (20px); jarak antar kartu `gap-4`; jarak antar section `gap-6`.
- Radius: kartu `rounded-xl` (12px); tombol/input `rounded-lg` (8px); badge `rounded-full`.
- Bayangan: kartu tanpa bayangan (mengandalkan border); tombol utama `shadow-sm`; modal `shadow-2xl`. DILARANG `shadow-lg shadow-rose-600/25` berwarna.
- Tinggi baris tabel: nyaman — `py-3`, bukan rapat.

---

## 3. Komponen standar (bangun sekali, pakai di mana-mana)

Implementasikan sebagai **macro Jinja** (`web/templates/components.html`, baru) atau kelas CSS bersama. Semua UI baru WAJIB memakai ini, bukan meracik utility per tempat.

### 3.1 Tombol (4 varian, 2 ukuran)

| Varian | Gaya | Dipakai untuk |
|---|---|---|
| Primary | bg `accent`, text white, `px-4 py-2` | Aksi utama per layar/modal (MAKS 1 per layar) |
| Secondary | bg `surface-2`, border, text `text` | Aksi sekunder |
| Ghost | transparan, text `muted` → `text` saat hover | Aksi tersier (detail, lihat) |
| Danger | bg `danger`, text white | Hapus / hentikan (selalu + konfirmasi) |

Ukuran: default `py-2 text-sm`; kecil `py-1.5 px-3 text-xs` (hanya di dalam tabel). Target sentuh minimum 36px. DILARANG tombol gradasi, DILARANG `active:scale-95` (efek mainan).

### 3.2 Badge status (1 pola, dot + label)

```
● LIVE        emerald dot (pulse halus) + teks emerald-400
● RECONNECTING amber dot + teks amber-400
● ERROR       rose dot + teks rose-400
● STOPPED     slate dot + teks slate-400
● IDLE        sky dot + teks sky-400
● SCHEDULED   sky dot outline + teks sky-400
```

Satu badge per status, selalu dot + label (bukan warna saja — aksesibilitas). Maks 3 badge per baris tabel; info lain masuk kolom/tooltip.

### 3.3 Kartu

`surface` solid + `border` 1px + `rounded-xl` + `p-4/5`. Header kartu: judul 14px/600 kiri + aksi kanan (maks 1 tombol + menu ⋯). Tanpa blur, tanpa gradasi, tanpa bayangan berwarna.

### 3.4 Input & form

- Label di atas field, 12px `muted`, wajib ditandai `*` merah muda (bukan teks "(wajib)").
- Input: bg `surface-2`, border, `rounded-lg`, `px-3 py-2`, text 14px; focus: ring `accent` 1px + border `accent`.
- Validasi inline: ✅/❌ + pesan 12px di bawah field (bukan alert); field salah = border `danger`.
- Select/dropdown: gaya sama dengan input; tanpa styling kustom per halaman.

### 3.5 Modal (1 kerangka)

Header (judul + ✕) · isi (maks 560px lebar, scroll dalam) · footer (tombol sekunder kiri, primer kanan). Backdrop gelap 60% + klik-di-luar menutup (kecuali form berisi perubahan belum disimpan → konfirmasi). Semua modal baru (edit jadwal, quick-fix, template, riwayat) memakai kerangka ini.

### 3.6 Tabel

Header 12px uppercase `muted` + tracking; baris `py-3`, hover `surface-2`, pemisah `border`. Kolom aksi: MAKS 2 tombol terlihat + menu ⋯ untuk sisanya (lihat §4.3). Angka rata kanan, mono. Baris yang bermasalah: penanda kiri 3px `warning`/`danger` (bukan seluruh baris merah).

### 3.7 Toast & umpan balik

- Sukses/error/loading via toast kanan atas (sistem toast yang sudah ada — pakai, jangan buat baru).
- Aksi async: tombol menunjukkan spinner + nonaktif selama proses; JANGAN biarkan klik ganda.
- Status order/antrean (misal hot-swap): tampilkan state kecil di baris ("Terkirim… → diterapkan ✓ / gagal ✗").

---

## 4. Aturan layout per halaman

### 4.1 Dashboard Live (perbaikan bertahap, mulai dari fitur baru)

- Header: judul + 1 tombol utama ("Buat Live", `accent`) + menu ⋯ untuk aksi sekunder (QA, depot, arsip). Hapus deretan 3–4 tombol warna-warni di header.
- Kartu ringkasan (4 angka hero: LIVE, RECONNECTING, kapasitas, load): angka 24px mono/700, label 12px `muted`.
- Tabel task: kolom = Status · Channel · Judul · Video/Audio (1 kolom ringkas) · Node · Uptime · Aksi (2 tombol + ⋯). Hapus kolom yang jarang dibaca (pindah ke modal detail).
- Baris bermasalah: penanda kiri + panel quick-fix di bawah baris (expand), bukan tombol baru di baris.

### 4.2 Form buat/edit jadwal (2136 baris → wizard)

Pecah `live_create.html` menjadi **wizard 4 langkah** + panel ringkasan lengket di kanan:
1. Channel & judul → 2. Video → 3. Audio → 4. Jadwal & tayang → Tinjau.
- Indikator langkah di atas (1–4 + centang). Validasi per langkah (tidak bisa lanjut bila langkah ini salah). Tombol "Kembali" selalu ada.
- **Modal edit memakai langkah yang sama** (komponen dipakai ulang — edit = wizard dengan nilai terisi + penanda dampak per field: 🟢 langsung / 🟡 hot-swap / 🔴 butuh restart).
- Panel ringkasan kanan: menampilkan konfigurasi yang dipilih secara live (channel, video, audio, jadwal, node) — operator selalu tahu apa yang akan disimpan.

### 4.3 Aksi per baris (aturan keras)

Maks 2 tombol terlihat per baris + menu ⋯. Contoh baris STREAMING: [Edit] [Stop] ⋯ (Pause, Restart, Metadata, Riwayat, Hapus). Ini saja memotong ~60% kebisingan visual dashboard.

### 4.4 Responsif & aksesibilitas

- Tabel → kartu di layar <768px (tiap task jadi 1 kartu dengan info kunci + aksi).
- Kontras minimum: teks `muted` (slate-400) di atas `surface` sudah lolos; jangan pakai slate-500 untuk teks penting.
- Semua tombol/icon-only punya `title` + `aria-label`.
- Focus ring terlihat (ring `accent`) untuk navigasi keyboard.

---

## 5. Yang TIDAK dilakukan (anti-pola)

- ❌ Menambah warna aksen baru ("biru untuk fitur ini, ungu untuk itu").
- ❌ Teks <12px, judul extrabold di mana-mana, gradasi tombol, blur kartu data.
- ❌ 5+ tombol dalam satu baris; 4+ badge dalam satu baris.
- ❌ Modal berbeda gaya per fitur — semua pakai kerangka §3.5.
- ❌ Animasi dekoratif (bounce, scale, pulse di mana-mana) — hanya dot LIVE yang berdenyut.
- ❌ Mengubah `base.html` secara besar-besaran dalam satu PR — migrasi per halaman, mulai dari halaman live.

---

## 6. Ceklis penerimaan UI (wajib sebelum merge)

- [ ] Hanya warna dari §2.1 (grep: tidak ada indigo/cyan/violet/teal/purple/pink/orange/lime baru).
- [ ] Tidak ada teks <12px; judul maks semibold.
- [ ] Tombol memakai 4 varian §3.1; maks 1 primary per layar.
- [ ] Maks 2 tombol terlihat per baris tabel (+ menu ⋯).
- [ ] Modal memakai kerangka §3.5; form memakai input §3.4.
- [ ] Status memakai badge §3.2 (dot + label).
- [ ] Screenshot perbandingan sebelum/sesudah dilampirkan di laporan (desktop 1440px + mobile 390px).
- [ ] Tidak ada regresi fungsional (tes + klik manual alur utama).

---

## 7. Rencana migrasi (bertahap, tidak big-bang)

| Tahap | Isi |
|---|---|
| 1 (bersama fitur edit jadwal) | Token §2 di `tailwind.config` + `components.html` + semua UI BARU (modal edit, quick-fix, template, riwayat) mengikuti panduan |
| 2 | Dashboard live: header + kartu ringkasan + tabel (aturan §4.1, §4.3) |
| 3 | `live_create.html` → wizard §4.2 |
| 4 | Halaman live lain (monitor, depot, node, arsip) + hapus warna terlarang yang tersisa |
| 5 | Halaman non-live (bertahap, saat disentuh) — target akhir: 1 sistem desain di semua halaman |
