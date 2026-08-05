# Diengaktual24

Portal berita ringan (judul, narasi, kategori, foto) dengan panel admin **tersembunyi** dari frontend, dibangun untuk deploy **gratis** di **Cloudflare Pages + D1**. Tanpa framework berat, tanpa biaya server — cocok untuk hosting gratisan.

- **Frontend**: HTML/CSS/JS murni (statis) — `public/`
- **Backend**: Cloudflare Pages Functions (serverless) — `functions/`
- **Database**: Cloudflare D1 (SQLite gratis)
- **Penyimpanan foto**: Cloudflare R2 (gratis, tanpa biaya egress)
- **Admin**: `/admin-diengaktual24/` — tidak ditautkan dari menu manapun di frontend, dan diblokir dari mesin pencari lewat `robots.txt` + header `noindex`.

**Fitur lengkap:**
- Tulis, ubah, hapus, dan publikasikan/draf artikel — judul, narasi, kategori, foto, ringkasan.
- Unggah foto langsung dari perangkat (tersimpan di R2) atau tempel URL foto dari luar.
- Kelola kategori (tambah/hapus) langsung dari dashboard, tanpa perlu menyentuh API.
- URL artikel cantik & bisa dibagikan: `/artikel/<judul-berita>`, lengkap dengan **preview link (Open Graph)** yang otomatis terisi judul, ringkasan, dan foto saat dibagikan ke WhatsApp/Facebook/Twitter.
- `sitemap.xml` dan `robots.txt` dibuat otomatis (bukan file statis) sehingga selalu mengikuti artikel terbaru dan domain aktif.
- Halaman 404 kustom bertema situs.
- Pencarian judul, filter kategori, dan pagination di beranda.
- Pembersihan otomatis: saat artikel/foto dihapus atau diganti, foto lama di R2 ikut terhapus supaya tidak menumpuk sampah penyimpanan.

---

## 1. Persiapan

1. Buat akun gratis di https://dash.cloudflare.com
2. Install Node.js (v18+) di komputer Anda.
3. Install Wrangler (CLI Cloudflare):
   ```bash
   npm install -g wrangler
   wrangler login
   ```

## 2. Buat database D1 (gratis)

```bash
cd diengaktual24
wrangler d1 create diengaktual24-db
```

Perintah di atas menampilkan `database_id`. Salin nilai itu ke file **`wrangler.toml`**, ganti `ISI_DENGAN_ID_DATABASE_ANDA`.

Jalankan migrasi skema:

```bash
wrangler d1 execute diengaktual24-db --remote --file=./migrations/0001_init.sql
```

## 3. Buat bucket R2 untuk foto (gratis)

Cloudflare R2 gratis hingga 10GB penyimpanan/bulan dengan **tanpa biaya egress** — cocok untuk foto berita.

```bash
wrangler r2 bucket create diengaktual24-media
```

Nama bucket ini sudah cocok dengan binding di `wrangler.toml` (`BUCKET` → `diengaktual24-media`). Tidak perlu mengaktifkan akses publik R2 apa pun — foto disajikan lewat Pages Function `/media/:key`, jadi tetap aman dan tidak butuh domain tambahan.

## 4. Buat proyek Cloudflare Pages

```bash
wrangler pages project create diengaktual24
```

Pilih branch produksi (mis. `main`).

## 5. Set secret (rahasia) — WAJIB sebelum deploy pertama

```bash
wrangler pages secret put JWT_SECRET
# masukkan string acak panjang, mis. hasil dari: openssl rand -hex 32

wrangler pages secret put SETUP_KEY
# masukkan kata sandi rahasia sementara, hanya dipakai SEKALI untuk membuat akun admin pertama
```

> `SETUP_KEY` hanya berfungsi selagi tabel admin masih kosong. Setelah admin pertama dibuat, endpoint setup otomatis terkunci — aman untuk dibiarkan di secret.

## 6. Deploy

```bash
wrangler pages deploy public
```

Setelah selesai, Wrangler memberi URL seperti `https://diengaktual24.pages.dev`. Anda juga bisa menghubungkan domain kustom lewat dashboard Cloudflare Pages secara gratis.

> **Penting**: pastikan kedua binding berikut juga sudah dipasang di **dashboard Cloudflare Pages** → Settings → Functions (perlu diisi manual jika Anda deploy lewat GitHub/dashboard, bukan lewat `wrangler deploy`):
> - **D1 database binding**: nama binding `DB` → pilih `diengaktual24-db`
> - **R2 bucket binding**: nama binding `BUCKET` → pilih `diengaktual24-media`

## 7. Buat akun admin pertama

Buka:
```
https://<domain-anda>/admin-diengaktual24/setup.html
```
Isi **Setup Key** (sesuai secret `SETUP_KEY`), buat username & password admin. Setelah berhasil, halaman ini otomatis terkunci selamanya.

## 8. Login admin & mulai menulis

```
https://<domain-anda>/admin-diengaktual24/
```

Di dashboard Anda bisa membuat artikel dengan:
- **Judul**
- **Narasi/isi berita** (pisahkan paragraf dengan baris baru)
- **Kategori** (Nasional, Daerah, Politik, Ekonomi, Internasional, Olahraga, Teknologi, Hiburan — bisa ditambah/dihapus lewat menu "Kelola Kategori")
- **Foto** — unggah langsung dari perangkat (JPG/PNG/GIF/WEBP, maks. 5MB). Foto disimpan di R2 dan otomatis disajikan lewat `/media/<nama-file>`. Masih ada opsi "atau pakai URL foto dari luar" jika ingin memakai gambar yang sudah tersedia di internet.
- **Status**: Draf (belum tayang di frontend) atau Publikasikan

Klik **"Kelola Kategori"** di sidebar untuk menambah atau menghapus kategori kapan saja.

---

## Kenapa admin tersembunyi & aman?

- Path admin **tidak pernah ditautkan** di navigasi/menu frontend manapun.
- `robots.txt` dan header `X-Robots-Tag: noindex` mencegah mesin pencari mengindeksnya.
- Sesi admin memakai **cookie HttpOnly + Secure + SameSite=Strict** (tidak bisa dicuri lewat JavaScript/XSS).
- Password admin di-hash dengan **PBKDF2-SHA256 (100.000 iterasi)** + salt unik, tidak pernah disimpan dalam bentuk teks biasa.
- Semua endpoint tulis/ubah/hapus artikel memvalidasi sesi admin di server (bukan hanya disembunyikan di frontend).
- Endpoint `/api/setup` otomatis nonaktif setelah admin pertama dibuat.
- Upload foto (`/api/upload`) juga wajib sesi admin, membatasi ukuran (maks. 5MB), dan memvalidasi isi berkas lewat *magic bytes* — bukan sekadar percaya nama file atau header `Content-Type` dari klien.

## Struktur proyek

```
diengaktual24/
├── functions/              # Backend (Cloudflare Pages Functions)
│   ├── _lib/                #   util JWT, hash password, helper R2
│   ├── api/                  #   endpoint: login, logout, me, setup, categories, articles, upload
│   ├── artikel/[slug].js     #   halaman artikel SSR (meta Open Graph dinamis)
│   ├── media/[key].js        #   penyaji foto dari R2
│   ├── robots.txt.js         #   robots.txt dinamis
│   └── sitemap.xml.js        #   sitemap dinamis dari artikel published
├── public/                  # Frontend statis
│   ├── index.html            #   beranda
│   ├── artikel.html          #   halaman detail artikel (fallback lama, via ?slug=)
│   ├── 404.html               #   halaman tidak ditemukan
│   ├── favicon.svg
│   ├── admin-diengaktual24/  #   panel admin tersembunyi
│   └── assets/                #   css & js
├── migrations/0001_init.sql # Skema database D1
└── wrangler.toml
```

## Menjalankan secara lokal (opsional, untuk uji coba)

```bash
npm install
wrangler d1 execute diengaktual24-db --local --file=./migrations/0001_init.sql
npm run dev
```

Lalu buka `http://localhost:8788`.

## Pengembangan lanjutan (opsional)

- Tambah proteksi brute-force login dengan **Cloudflare Turnstile**.
- Tambah lebih dari satu akun admin lewat SQL langsung ke tabel `admins` (gunakan skrip hashing PBKDF2 yang sama).
- Tambah RSS feed (`/rss.xml`) dengan pola yang sama seperti `sitemap.xml.js` jika dibutuhkan.
