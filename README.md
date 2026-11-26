# Inputin - Catatan Pengeluaran

Aplikasi pencatat pengeluaran harian, multi-team, berbentuk PWA yang bisa dipasang di HP.

**Aplikasi live:** https://falconafk31.github.io/catatan-pengeluaran/

## Fitur

- **Tambah, ubah, hapus pengeluaran** - tanggal, jumlah, kategori, keterangan, dicatat oleh.
- **Team 1, 2, 3** - setiap transaksi masuk satu team; ada pil Team di form, chip Team di daftar, dan filter Team di Riwayat dan Laporan.
- **Beranda** - total bulan ini, jumlah transaksi, ringkasan per kategori, transaksi terakhir, skeleton saat memuat, dan cache data terakhir supaya tidak tampil Rp0 saat buka pertama.
- **Riwayat** - daftar per hari, cari, filter kategori dan team, muat lagi 50 baris.
- **Laporan** - Harian, 7 Hari, Bulanan, Tahunan; perbandingan dengan periode sebelumnya; seksi **Analisa**: kategori terbesar, team terbesar, rata-rata per hari, transaksi terbesar.
- **Unduh PDF dan CSV** - PDF dibuat di sisi klien (jsPDF); laporan yang disaring team hanya berisi team itu di semua bagian, termasuk tabel Per Team.
- **Kelola Kategori** - tambah, ubah, hapus kategori; ikon garis di lingkaran pastel; ubah nama kategori ikut mengubah transaksi lama.
- **Transaksi Otomatis** - aturan Harian, Mingguan, Bulanan; jalan saat aplikasi dibuka (tanpa pemicu terjadwal).
- **Mode gelap dan terang** - pilihan tersimpan di HP.
- **PWA** - bisa dipasang dari browser HP; shell aplikasi di-cache service worker.

## Cara pakai

1. Buka tautan aplikasi di atas.
2. Masuk dengan kata sandi aplikasi (disimpan di spreadsheet, lihat Spesifikasi Teknis). Sesi berlaku 30 hari, jadi tidak perlu masuk setiap hari.
3. Ketuk **Tambah**, pilih team, kategori, isi jumlah dan keterangan, lalu simpan.
4. Lihat ringkasan di **Beranda** dan **Laporan**; unduh PDF atau CSV dari tab Laporan bila perlu.

## Spesifikasi Teknis

### Arsitektur

```
HP / browser (PWA)
  -> GitHub Pages (frontend statis: index.html + manifest.json + sw.js)
  -> Google Apps Script API (endpoint /exec, JSON)
  -> Google Sheets (database)
```

Frontend dan backend terpisah penuh: frontend tidak menyimpan data, semua baca/tulis lewat API.

### Frontend

- Satu file `index.html` (Vue 3 lewat CDN, tanpa build step).
- PDF laporan dibuat di sisi klien dengan jsPDF 2.5.1 + jspdf-autotable 3.8.2.
- `manifest.json` + `sw.js`: PWA installable; service worker cache-first untuk shell aplikasi saja (API dan CDN selalu network). Nama cache saat ini: `inputin-v11`. **Setiap deploy yang mengubah `index.html` wajib menaikkan nama cache di `sw.js`**, kalau tidak PWA terpasang terus menyajikan tampilan lama.
- Mobile-first, lebar acuan 390px; tampilan gelap/terang lewat variabel CSS di `:root` dan `html.dark`.
- UI v10.0 (redesign Claude, 26 Nov 2026): navigasi bawah 5 tab (Beranda, Riwayat, tombol + tengah, Laporan, Lainnya); form Tambah memakai keypad angka; Beranda memakai kartu hero gradasi biru + diagram batang "Ke mana uangnya pergi"; Kelola Kategori dan Transaksi Otomatis pindah ke tab Lainnya; font Plus Jakarta Sans (Google Fonts, fallback font sistem bila gagal dimuat).
- Versi frontend saat ini: **v10.0-pages**.

### Backend (API)

- Google Apps Script, satu file `Code-api.gs` (578 baris), API versi `9.7-pages`, deployment Version 6.
- `doGet` melayani aksi baca lewat `?action=`; `doPost` menerima JSON `{action, args}` dengan Content-Type `text/plain` (Apps Script tidak menjawab preflight OPTIONS, jadi jangan pakai `application/json`).
- Balasan selalu `{ok, data}` atau `{ok:false, error}`.
- Login: kata sandi dicocokkan dengan sel `Pengaturan!B2` di spreadsheet; bila cocok, server membuat token UUID yang disimpan di `PropertiesService` dan berlaku 30 hari (tetap sejak login, bukan sliding). Semua fungsi data memeriksa token.
- Aksi baca: `getSummary`, `getExpenses`, `getReport`, `getCategories`, `getAutoRules`, `getDashboard`.
- Aksi tulis: `login`, `logout`, `addExpense`, `updateExpense`, `deleteExpense`, `addCategory`, `renameCategory`, `deleteCategory`, `addAutoRule`, `updateAutoRule`, `deleteAutoRule`, `runAutoRules`.
- `getReport` mendukung paging server (`offset`/`limit`, `limit 0` = semua baris untuk PDF) dan cursor `afterId`; parameter `team` untuk menyaring laporan per team.
- Tanggal dari sheet dibaca lewat helper `toDate_`: sel berformat angka dikembalikan Sheets sebagai nomor serial (bukan Date), jadi harus dikonversi manual.

### Database (Google Sheets)

Satu spreadsheet, zona waktu Asia/Jakarta, dengan sheet:

| Sheet | Isi |
| --- | --- |
| `Pengeluaran` | A ID, B Tanggal, C Jumlah (Rp), D Kategori, E Keterangan, F Dicatat Oleh, G Waktu Catat, H Team |
| `Kategori` | Daftar nama kategori |
| `Pengaturan` | B2 = kata sandi aplikasi |
| `Otomatis` | Aturan transaksi otomatis (dibuat server saat pertama dipakai) |
| `Rekap` | Rekap per bulan (rumus) |

Nilai team yang tidak dikenal dinormalisasi ke `Team 1` di sisi server.

### Aturan yang dikunci

- **PDF yang disaring hanya berisi team yang dipilih, di semua bagian** - termasuk tabel Per Team dan total kaki tabel. Server menghitung `perTeam` dari seluruh periode (untuk perbandingan), jadi frontend wajib menyaring baris tabel Per Team sendiri saat mengunduh PDF tersaring.
- **Semantik Analisa**: `perTeam` dihitung dari seluruh periode sebelum filter team; transaksi terbesar dihitung dari item yang tampil sesudah filter; baris "Team terbesar" disembunyikan bila laporan sedang disaring (persennya bisa di atas 100% karena pembandingnya global); bila team yang disaring tidak punya transaksi, seluruh seksi Analisa tidak tampil.
- **Jangan pernah memperpanjang range `setValues` melewati kolom yang tidak diniatkan** - satu kolom bergeser merusak kolom sebelah tanpa error (pernah terjadi: team tertulis ke kolom Waktu Catat).
- Update optimis di frontend: tambah/ubah/hapus langsung mengubah tampilan, gagal berarti data lama dikembalikan dari server.
- Kata sandi aplikasi hanya hidup di sel `Pengaturan!B2`; ganti kata sandi = ganti isi sel itu, tanpa deploy ulang.

### Struktur repo

```
index.html      Frontend (Vue 3, satu file)
manifest.json   Manifest PWA (nama "Inputin")
sw.js           Service worker (cache shell)
icon-48.png     Ikon PWA 48px
icon-180.png    Ikon Apple touch 180px
icon-192.png    Ikon PWA 192px
icon-512.png    Ikon PWA 512px
```

## Deploy

### Frontend (GitHub Pages)

1. Ubah file di repo ini, commit ke branch `main`.
2. GitHub Pages (Deploy from branch) menerbitkan ulang otomatis dalam sekitar satu menit.
3. Bila `index.html` berubah, naikkan dulu nama `CACHE` di `sw.js` (lihat aturan di atas).
4. Cek sesudah terbit: `index.html` yang disajikan harus memuat penanda versi baru, dan `sw.js` harus memuat nama cache baru.

### Backend (Apps Script)

1. Buka proyek Apps Script di `script.google.com`.
2. Tempel perubahan `Code-api.gs`.
3. **Edit deployment yang sudah ada ke Version baru** (Deploy > Manage deployments). Jangan buat deployment baru, supaya URL `/exec` tidak berubah.

## Riwayat versi

| Versi | Tanggal | Isi |
| --- | --- | --- |
| v9.0-pages | 24 Nov 2026 | Migrasi frontend ke GitHub Pages; backend jadi API JSON |
| v9.1-pages | 24 Nov 2026 | Optimasi backend (getDashboard satu baca, paging getReport) |
| v9.2-pages | 24 Nov 2026 | Perbaikan tanggal serial Sheets |
| v9.3-pages | 24 Nov 2026 | Cursor paging, ikon SVG kategori, skeleton, perbaikan UI |
| v9.4-pages | 24 Nov 2026 | Spinner semua status loading, update optimis |
| v9.5-pages | 24 Nov 2026 | Perbaikan flash Rp0/0 saat buka pertama |
| v9.6-pages | 24 Nov 2026 | Spinner hapus di dalam modal konfirmasi |
| v9.7-pages | 25 Nov 2026 | Fitur Team + Analisa; perbaikan kolom Team di updateExpense |
| v9.8-pages | 25 Nov 2026 | UI "Navy Modern" (ikon Lucide, chip pastel, tombol pil) |
| v9.9-pages | 25 Nov 2026 | Kelola Kategori: tombol Ubah/Hapus sejajar nama kategori |
| v10.0-pages | 26 Nov 2026 | Redesign UI/UX (Claude): nav 5 tab + tombol +, keypad Tambah, hero gradasi biru, tab Lainnya, font Plus Jakarta Sans; nama kategori panjang tetap 2 baris (perbaikan v9.9 dipertahankan) |

## Catatan keamanan

Repo ini publik. Jangan commit kata sandi aplikasi, URL endpoint API, ID spreadsheet, atau token apa pun ke repo ini. Kata sandi hanya disimpan di spreadsheet (sel `Pengaturan!B2`) dan tidak pernah ditulis di file repo.

Proyek pribadi, dibuat untuk pemakaian sendiri.
