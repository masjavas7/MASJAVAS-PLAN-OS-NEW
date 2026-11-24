# Desain Notifikasi Live Profesional — MASJAVAS TUBE OS

Tanggal: 24 Nov 2026, 14:29 WIB. Perintah user: saat klik "Mulai Siar" notifikasi
saat ini tidak profesional; ganti dengan ikon/notifikasi profesional yang
menunjukkan aplikasi sedang memproses siaran; buat semuanya smooth layaknya
aplikasi modern dan mahal.

## 1. Diagnosis bottleneck (dicek langsung di kode, 24 Nov 14:29)

- `live_dashboard.html` (1.589 baris): ~20× `alert()` + ~5× `confirm()` untuk
  SEMUA umpan balik — start, stop, hapus, tautkan broadcast, terbitkan publik,
  peringatan kuota, timeout. Dialog native browser: tidak bisa di-style,
  memblokir seluruh tab, menampilkan origin mentah, terasa murah.
- `startLive()` (baris 635–672): tombol SUDAH punya loading state (spinner +
  "Memulai...") — bagian ini bagus. Tapi saat SUKSES: DIAM TOTAL, hanya
  `loadTasks()` (refresh daftar). User tidak mendapat konfirmasi apa pun bahwa
  siaran sedang dimulai.
- `confirm()` kapasitas menampilkan jargon internal ("Proyeksi CPU",
  "Admission Control", "Zero-CPU Stream Copy (~3% CPU)") — user tidak boleh
  melihat ini.
- `live_create.html` (2.076 baris): 26× `alert()`/`confirm()` lagi (validasi).
- KABAR BAIK: sistem toast "Global Elegant Toast Notification System" SUDAH ADA
  di `base.html` baris 60 (`window.showToast(msg, type)`, animasi slide/fade,
  backdrop-blur, rounded-2xl, ikon Font Awesome) dan dipakai halaman
  schedule/vaults/revenue/ingestion. Halaman live hanya BELUM MENGADOPSINYA.
  Jadi perbaikan ini sebagian besar ADOPSI, bukan bangun dari nol.

## 2. Prinsip desain

1. **Satu sistem notifikasi untuk seluruh aplikasi** — toast `base.html`.
   Tidak ada lagi `alert()`/`confirm()` di halaman mana pun.
2. **Setiap aksi punya 3 keadaan yang terlihat**: memproses → berhasil / gagal.
   Tidak ada lagi sukses yang diam.
3. **Bahasa manusia, tanpa jargon.** "Admission Control" → "Server sedang sibuk".
   "Timeout 45 detik" → "Server membutuhkan waktu lebih lama dari biasanya".
4. **Error selalu disertai aksi**: tombol "Coba lagi" di dalam toast, bukan
   sekadar teks.
5. **Transisi 200–300ms ease-out di semua perubahan keadaan** (kartu, modal,
   toast, indikator status) — ini yang membuat aplikasi terasa mahal.
6. **Verifikasi, bukan asumsi.** Setelah start sukses, konfirmasi 1× bahwa
   encoder benar-benar STREAMING (bukan hanya "perintah diterima").

## 3. Komponen (di `base.html`, dipakai semua halaman ke depan)

- `window.showToast(msg, type)` — SUDAH ADA. Tipe: success (hijau,
  fa-circle-check), error (merah, fa-circle-xmark), info (biru, fa-circle-info),
  warning (kuning, fa-triangle-exclamation). Auto-hilang ~4 detik.
- `window.showProgressToast(msg)` — BARU. Toast persisten (tidak auto-hilang)
  dengan spinner + pesan; mengembalikan handle `{update(msg), done(msg, type),
  fail(msg)}` untuk mengubah keadaan tanpa menumpuk toast baru.
- `window.showConfirm({title, message, confirmText, cancelText, danger})` —
  BARU. Modal bermerek (backdrop-blur, rounded-2xl, ikon) yang mengembalikan
  Promise<boolean>. Menggantikan semua `confirm()`.

## 4. Alur "Mulai Siar" yang baru

1. Klik → tombol jadi spinner "Memulai..." (sudah ada) + progress toast muncul:
   "Memulai siaran…".
2. Fetch POST start.
   - Sukses → progress toast berubah jadi sukses: "Siaran dimulai — mengudara
     ke YouTube…" + kartu task bertransisi ke status STREAMING dengan indikator
     LIVE (dot merah berdenyut `animate-ping` + label LIVE). Polling 1× setelah
     3 detik untuk konfirmasi encoder STREAMING; bila belum, toast info:
     "Encoder masih disiapkan — status diperbarui otomatis."
   - Kapasitas penuh → modal konfirmasi bermerek (bukan `confirm()`): judul
     "Server sedang sibuk", pesan manusiawi, tombol "Ya, mulai paksa" / "Batal".
   - Gagal → progress toast jadi error + tombol "Coba lagi" (memanggil
     `startLive` ulang).
   - Network error → toast error "Tidak dapat menghubungi server. Periksa
     koneksi Anda." + "Coba lagi".
3. Stop/hapus/terbitkan/tautkan: pola yang sama (modal konfirmasi bermerek +
   progress toast + toast hasil). Pesan sukses tidak lagi memuat ID teknis
   mentah di `alert()` — tampilkan judul + tautan "Lihat di YouTube" sebagai
   aksi toast.

## 5. Detail "mahal" (micro-interactions)

- Indikator LIVE: dot merah berdenyut + label LIVE (standar YouTube/Twitch).
- Kartu task: transisi status dengan fade (bukan ganti mendadak).
- Tombol: `active:scale-95` sudah ada — pertahankan; tambah hover lift halus
  pada kartu.
- Modal: masuk dengan scale 0.95→1 + fade 200ms; tutup dengan fade.
- Toast bertumpuk maksimal 3; yang lama keluar dengan slide-up.

## 6. Scope build

- `base.html`: tambah `showProgressToast` + `showConfirm` (komponen bersama).
- `live_dashboard.html`: ganti SEMUA `alert()`/`confirm()` → toast/modal;
  alur startLive baru (§4); indikator LIVE pulse; polling konfirmasi 3 detik.
- `live_create.html`: ganti 26 `alert()` validasi → toast (langkah 1; validasi
  inline adalah langkah 2 terpisah, bukan scope ini).
- TIDAK menyentuh: endpoint API, logika supervisor/encoder, file lane lain.
- Jinja auto-reload: perubahan template langsung live TANPA restart web.
  Verifikasi: (1) grep — 0 `alert(`/`confirm(` tersisa di kedua file;
  (2) halaman live ter-render 200 + tidak ada error JS (cek via curl + baca
  template); (3) screenshot Playwright 390px + desktop bila memungkinkan.
- Tes: tidak ada tes unit untuk template; verifikasi = checklist di atas +
  lapor dengan bukti (grep count + status render).
