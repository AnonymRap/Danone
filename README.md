# Papan Kerja — To-Do List Tim (Real-time)

Website to-do list minimalis untuk tim, dengan 3 status (**Belum Dikerjakan**,
**Sedang Dikerjakan**, **Selesai**), sinkronisasi real-time antar anggota tim,
dan login supaya hanya tim Anda yang bisa mengakses dan mengubah data.

Backend memakai **Supabase** (gratis, berbasis Postgres, sudah termasuk auth
dan real-time). File `index.html` ini murni statis, jadi bisa langsung
di-hosting dari repo GitHub via **GitHub Pages** — tidak perlu server sendiri.

Kenapa perlu Supabase dan bukan cuma file statis? Karena Anda minta data bisa
ditambah **real-time oleh banyak orang** dan **aman** — itu artinya perlu
database + aturan akses di belakang layar, bukan sekadar HTML/JS di browser.
Anon key Supabase memang publik dan boleh terlihat di kode (itu memang
desainnya), keamanan sebenarnya diatur lewat **Row Level Security (RLS)** di
langkah 3 — jadi walau repo-nya publik di GitHub, orang luar tetap tidak bisa
baca/ubah data tanpa login.

---

## 1. Buat project Supabase

1. Buka [supabase.com](https://supabase.com) → daftar/masuk → **New project**.
2. Isi nama project & password database (simpan baik-baik), pilih region
   terdekat (mis. Singapore), lalu **Create new project**. Tunggu ~2 menit.

## 2. Buat tabel `tasks`

Di dashboard project → menu **SQL Editor** → **New query** → tempel ini →
**Run**:

```sql
create table tasks (
  id uuid primary key default gen_random_uuid(),
  title text not null,
  assignee text,
  status text not null default 'todo' check (status in ('todo','progress','done')),
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);

alter table tasks enable row level security;

-- Hanya pengguna yang sudah login (anggota tim) yang boleh
-- membaca, menambah, mengubah, dan menghapus tugas.
create policy "authenticated can read" on tasks
  for select to authenticated using (true);
create policy "authenticated can insert" on tasks
  for insert to authenticated with check (true);
create policy "authenticated can update" on tasks
  for update to authenticated using (true);
create policy "authenticated can delete" on tasks
  for delete to authenticated using (true);

-- Aktifkan realtime untuk tabel ini
alter publication supabase_realtime add table tasks;
```

## 3. Atur akun tim (siapa yang boleh login)

Ada dua pilihan, pilih salah satu:

- **Opsi A — terkontrol (disarankan untuk keamanan):** matikan pendaftaran
  bebas. Di dashboard → **Authentication → Providers → Email**, matikan
  "Allow new users to sign up". Lalu tambahkan akun tim Anda satu per satu
  lewat **Authentication → Users → Add user**.
- **Opsi B — santai:** biarkan pendaftaran terbuka (tombol "Daftar" di
  website akan berfungsi), cocok kalau tim kecil dan saling percaya. Siapa
  pun yang tahu link website bisa membuat akun sendiri.

## 4. Ambil URL & anon key

Dashboard → **Project Settings → API**. Salin:
- **Project URL**
- **anon public key**

Buka `index.html`, cari bagian ini di paling atas `<script>`, dan ganti:

```js
const SUPABASE_URL = "GANTI_DENGAN_SUPABASE_URL";
const SUPABASE_ANON_KEY = "GANTI_DENGAN_SUPABASE_ANON_KEY";
```

## 5. Push ke GitHub & aktifkan GitHub Pages

```bash
git init
git add .
git commit -m "Papan kerja tim"
git branch -M main
git remote add origin https://github.com/USERNAME/NAMA_REPO.git
git push -u origin main
```

Lalu di GitHub: repo → **Settings → Pages** → Source: pilih branch `main`,
folder `/ (root)` → **Save**. Setelah beberapa menit, website Anda aktif di
`https://USERNAME.github.io/NAMA_REPO/`.

## 6. Coba

Buka linknya, daftar/masuk dengan akun yang sudah dibuat di langkah 3, lalu
tambahkan tugas. Buka di perangkat/tab lain dengan akun anggota tim lain —
perubahan status dan tugas baru akan muncul otomatis tanpa refresh.

---

### Catatan keamanan
- Jangan pernah menaruh **service_role key** Supabase di file ini — hanya
  pakai **anon public key**, itu memang dirancang untuk dipakai di sisi
  klien/browser dan aman selama RLS di langkah 2 sudah aktif.
- Repo boleh publik di GitHub; yang melindungi data Anda adalah kombinasi
  login (auth) + RLS, bukan menyembunyikan kode.
