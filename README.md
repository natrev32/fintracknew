# Finance Tracker — Discord Sync

Pencatat keuangan pribadi berbasis HTML/JS murni (tanpa backend) dengan notifikasi otomatis ke Discord via Webhook, plus portal Tabungan & Investasi.

## Fitur

- Pencatatan transaksi masuk/keluar dengan buku kas & saldo berjalan
- Rekap per periode: bulan ini, bulan kemarin, tahun ini, atau bulan/tahun pilihan
- Portal Tabungan & Investasi: catat setor/tarik, saldo per portal, total dana terpisah
- Notifikasi Discord webhook dengan embed yang dapat dikustomisasi
- Login kata sandi + enkripsi AES-256-GCM untuk seluruh data

## Deploy ke Vercel via GitHub

1. Buat repository baru di GitHub, lalu push semua file di folder ini:
   ```bash
   git init
   git add .
   git commit -m "Finance Tracker with Discord webhook + security headers"
   git branch -M main
   git remote add origin https://github.com/USERNAME/REPO.git
   git push -u origin main
   ```
2. Buka [vercel.com/new](https://vercel.com/new), pilih repository tersebut, lalu klik **Deploy** (tanpa konfigurasi tambahan — `index.html` otomatis jadi halaman utama).
3. File `vercel.json` sudah berisi security headers (CSP, anti-clickjacking, HSTS, dll.) yang otomatis diterapkan Vercel.

## Deploy ke Rumahweb (cPanel)

1. Login cPanel Rumahweb → **File Manager** → masuk ke `public_html`.
2. Upload **`index.html`** dan **`.htaccess`** (file tersembunyi — aktifkan "Show Hidden Files" di File Manager bila tidak terlihat).
3. cPanel → **SSL/TLS Status** → jalankan **AutoSSL** untuk domain Anda. **HTTPS wajib aktif** — tanpa itu, enkripsi vault tidak berjalan (aplikasi turun ke mode plaintext) dan fitur "Lupa kata sandi" tidak dapat bekerja.
4. Selesai. `.htaccess` otomatis memaksa HTTPS dan menerapkan security headers yang sama dengan versi Vercel.

Catatan: `vercel.json` tidak dipakai di Rumahweb — boleh tidak di-upload.

## Struktur

| File | Fungsi |
|---|---|
| `index.html` | Aplikasi utama (halaman depan) |
| `finance-tracker.html` | Salinan identik untuk penggunaan lokal |
| `vercel.json` | Security headers untuk Vercel |
| `.htaccess` | Security headers + paksa HTTPS untuk Apache/Rumahweb |

## Keamanan

- **Kata sandi + enkripsi AES-256-GCM** — saat pertama kali dibuka, Anda diminta membuat kata sandi. Kata sandi diturunkan menjadi kunci enkripsi (PBKDF2, 310.000 iterasi), lalu seluruh data (transaksi, setelan webhook, log) disimpan **terenkripsi** di `localStorage`. Tanpa kata sandi, data tidak dapat dibaca siapa pun — bahkan dengan membuka DevTools.
- **Auto-lock 10 menit** — aplikasi mengunci sendiri setelah tidak ada aktivitas. Tombol kunci manual tersedia di navbar.
- **Batas percobaan gagal** — 5 kali salah kata sandi → cooldown 30 detik.
- **Data 100% lokal** — transaksi tidak pernah dikirim ke server mana pun kecuali embed notifikasi ke webhook Discord Anda sendiri.
- **URL Webhook tidak masuk repo** — webhook tersimpan terenkripsi per-browser; tidak ada rahasia di dalam kode, aman untuk repo publik.
- **Validasi URL webhook** — hanya `https://discord.com/api/webhooks/...` atau `https://discordapp.com/api/webhooks/...` yang diterima.
- **Anti-XSS** — semua teks dari input pengguna di-escape sebelum dirender.
- **Validasi data vault** — data korup/dimanipulasi otomatis disaring saat dimuat.
- **Rate limit webhook** — minimal 1,5 detik antar pengiriman untuk mencegah spam.
- **Security headers** — CSP, `X-Frame-Options: DENY`, `nosniff`, `Referrer-Policy: no-referrer`, HSTS, dan `Permissions-Policy` restriktif via `vercel.json`.

### Backup & Pindah Perangkat

- **Export** (tombol unduh di toolbar) → pilih format:
  - **Terenkripsi (disarankan)** — snapshot vault AES-256; hanya bisa dibuka dengan kata sandi Anda saat file diekspor.
  - **Plaintext** — JSON biasa tanpa enkripsi.
- **Impor** (tombol unggah di toolbar, atau tautan "Impor Backup Terenkripsi" di layar kunci):
  - File terenkripsi → mengganti seluruh data browser, lalu diminta kata sandi file tersebut. Bisa dilakukan bahkan sebelum setup (perangkat baru).
  - File plaintext (format lama) → hanya bisa diimpor setelah aplikasi terbuka kunci.
- **Pindah perangkat:** di perangkat lama → Export Terenkripsi → simpan/kirim file ke perangkat baru → buka website → *Impor Backup Terenkripsi* → masukkan kata sandi. Selesai.

### Sinkronisasi Cloud (Multi-Device) — Opsional

Secara default data hanya tersimpan di browser perangkat. Untuk akses dari semua device, aktifkan sinkronisasi cloud. Server (Supabase) **hanya menyimpan ciphertext** — data Anda tetap terenkripsi end-to-end dengan kata sandi vault.

**Setup sekali (pemilik website):**

1. Buat akun & project gratis di [supabase.com](https://supabase.com) (free tier cukup).
2. Di Supabase Dashboard → **SQL Editor**, jalankan:
   ```sql
   create table vaults (
     user_id uuid primary key references auth.users on delete cascade,
     blob jsonb not null,
     updated_at timestamptz not null default now()
   );
   alter table vaults enable row level security;
   create policy "own vault only" on vaults
     for all
     using (auth.uid() = user_id)
     with check (auth.uid() = user_id);

   create table profiles (
     user_id uuid primary key references auth.users on delete cascade,
     username text not null unique,
     email text not null,
     created_at timestamptz not null default now()
   );
   alter table profiles enable row level security;
   create policy "public read" on profiles for select using (true);
   create policy "own insert" on profiles for insert with check (auth.uid() = user_id);
   ```
3. Masih di dashboard → **Authentication → Providers → Email**: matikan opsi **Confirm email** (supaya pendaftaran langsung aktif; aktifkan lagi kalau mau verifikasi email).
4. Ambil di **Project Settings → API**: salin **Project URL** dan **anon public key**.

**PENTING untuk website publik — tanamkan konfigurasi server:**

Buka `index.html`, cari bagian ini di dalam `<script>` (cari kata `EMBEDDED_CLOUD_URL`):

```js
const EMBEDDED_CLOUD_URL = '';      // contoh: 'https://abcdefgh.supabase.co'
const EMBEDDED_CLOUD_ANON_KEY = ''; // anon public key (Project Settings → API)
```

Isi keduanya. Setelah itu panel "Pengaturan Server" otomatis hilang — pengguna publik tinggal daftar/masuk tanpa copy-paste apa pun. Anon key memang dirancang publik oleh Supabase; keamanan datanya ada di RLS + enkripsi vault.

**Setup reset kata sandi (Supabase → Authentication → URL Configuration):**
- **Site URL**: isi alamat website Anda (mis. `https://fintrackbeta.vercel.app` atau domain Rumahweb)
- **Redirect URLs**: tambahkan alamat yang sama
- Tanpa ini, tombol "Lupa kata sandi akun?" mengirim link yang tidak mengarah kembali ke website Anda.

**Cara pakai (per pengguna):**

**Satu kata sandi untuk semuanya** — kata sandi akun sekaligus menjadi kunci enkripsi data (AES-256, diturunkan di browser).

- **Daftar** (pengguna baru): tab *Daftar* → username, email, kata sandi → akun dibuat, vault terenkripsi langsung terbentuk & terunggah ke cloud.
- **Masuk** (perangkat lain/lama): tab *Masuk* → username + kata sandi → data terenkripsi diambil dari Supabase → dekripsi otomatis → aplikasi terbuka.
- **Kunjungan berikutnya di perangkat yang sama**: cukup masukkan kata sandi (dekripsi lokal, tanpa server).
- **Lupa kata sandi**: klik *Lupa kata sandi akun?* → link via email → kata sandi baru. Karena kata sandi = kunci enkripsi, data lama tidak bisa dibuka dengan kata sandi baru — saat masuk akan muncul opsi memasukkan **kata sandi lama** untuk menyelaraskan (data tetap utuh), atau memulai data baru.
- Di dalam aplikasi, ikon **awan** di navbar: status akun, *Sinkronkan Sekarang*, atau keluar akun. Setiap perubahan otomatis terunggah terenkripsi.
- **Pengguna lama dua-kata-sandi**: saat masuk, kata sandi akun tidak akan cocok dengan enkripsi lama — masukkan kata sandi vault lama di opsi penyelarasan, dan sistem akan mengenkripsi ulang dengan kata sandi akun.

**Catatan desain:**
- Login memakai **username** — aplikasi mencari email yang cocok di tabel `profiles` (perlu terbaca publik untuk keperluan login; konsekuensinya email akun terlihat oleh pengguna lain yang login).
- Kata sandi dikirim ke Supabase saat login (standar HTTPS) — server tepercaya Supabase memverifikasi, tapi secara teori operator server yang merekam kata sandi login dapat mendekripsi data. Ini trade-off kesederhanaan satu-kata-sandi.
- Konflik antar perangkat diselesaikan *last-write-wins* — jangan mengedit di dua perangkat bersamaan.
- URL & anon key Supabase bersifat publik (aman didesain begitu) — proteksi datanya ada di RLS + enkripsi.

- **Kata sandi tidak bisa dipulihkan.** Tidak ada fitur "lupa kata sandi" karena tidak ada server yang menyimpannya. Jika lupa, satu-satunya jalan adalah menghapus data (Clear Data) dan mulai dari nol — karena itu pertimbangkan mencadangkan kata sandi di password manager.
- **Backup plaintext tidak terenkripsi** — simpan di tempat aman; gunakan format terenkripsi jika ragu.
- **Backup terenkripsi adalah snapshot** — file lama tetap terbuka dengan kata sandi **saat file itu diekspor**, meski Anda kemudian mengganti kata sandi.
- Ganti kata sandi kapan saja lewat ikon kunci di navbar.

## Catatan

- Jika suatu saat mengganti CDN (Tailwind/FontAwesome/Google Fonts), sesuaikan juga domain di `Content-Security-Policy` pada `vercel.json`, jika tidak script akan diblokir browser.
- Webhook Discord bisa di-regenerate kapan saja dari Server Settings → Integrations → Webhooks jika URL-nya pernah bocor.
