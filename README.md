# Aplikasi Peminjaman Buku

Aplikasi web berbasis Laravel untuk mengelola data buku dan peminjaman buku di perpustakaan sekolah.

## Deskripsi Program

Aplikasi ini mencatat data buku, data pengguna, dan transaksi peminjaman. Ada dua jenis pengguna, ditentukan oleh kolom `role` pada tabel `users`:

- **admin**: mengelola data buku dan peminjaman.
- **siswa** (default): pengguna yang meminjam buku. Memiliki data `kelas`.

Alur peminjaman:

1. Siswa meminjam buku, lalu sistem membuat data `loans` dengan `loan_date`, `due_date`, dan `status` bawaan `dipinjam`.
2. Saat buku dikembalikan, `returned_at` diisi tanggal pengembalian dan `status` diperbarui.
3. Jika pengembalian melewati `due_date`, denda dicatat pada kolom `fine`.

Status pengerjaan saat ini: struktur database (migration) dan model sudah dibuat. Route baru berisi halaman awal, sedangkan controller untuk buku dan peminjaman belum dibuat.

## Fitur

- **Books**: data buku (judul, penulis, penerbit, tahun, stok).
- **Loans**: data peminjaman (tanggal pinjam, batas kembali, tanggal kembali, status, denda).
- **Users**: pengguna dengan peran dan kelas.

## Teknologi

- PHP & Laravel
- MySQL (XAMPP)
- Composer

## ERD

```mermaid
erDiagram
  USERS ||--o{ LOANS : meminjam
  BOOKS ||--o{ LOANS : dipinjam
  USERS {
    bigint id PK
    string name
    string email UK
    string role
    string kelas
    string password
    timestamp email_verified_at
  }
  BOOKS {
    bigint id PK
    string title
    string author
    string publisher
    int year
    int stock
  }
  LOANS {
    bigint id PK
    bigint user_id FK
    bigint book_id FK
    date loan_date
    date due_date
    date returned_at
    string status
    int fine
  }
```

## Dokumentasi Fungsi / Prosedur

### Model `Book` (`app/Models/Book.php`)

| Elemen | Jenis | Keterangan |
|--------|-------|------------|
| `$fillable` | properti | Kolom yang boleh diisi massal: `title`, `author`, `publisher`, `year`, `stock`. |

### Model `Loan` (`app/Models/Loan.php`)

| Elemen | Jenis | Parameter | Return | Keterangan |
|--------|-------|-----------|--------|------------|
| `$fillable` | properti | - | - | `user_id`, `book_id`, `loan_date`, `due_date`, `returned_at`, `status`, `fine`. |
| `$casts` | properti | - | - | `loan_date`, `due_date`, `returned_at` otomatis menjadi objek tanggal (Carbon). |
| `user()` | fungsi relasi | - | `BelongsTo` | Mengambil user peminjam melalui `user_id`. Contoh: `$loan->user->name`. |
| `book()` | fungsi relasi | - | `BelongsTo` | Mengambil buku yang dipinjam melalui `book_id`. Contoh: `$loan->book->title`. |

### Model `User` (`app/Models/User.php`)

| Elemen | Jenis | Parameter | Return | Keterangan |
|--------|-------|-----------|--------|------------|
| `$fillable` | properti | - | - | `name`, `email`, `password`, `role`, `kelas`. |
| `$hidden` | properti | - | - | `password` dan `remember_token` disembunyikan saat data diubah ke array/JSON. |
| `casts()` | fungsi | - | `array` | `email_verified_at` menjadi datetime, `password` otomatis di-hash. |
| `isAdmin()` | fungsi | - | `bool` | `true` jika `role` bernilai `admin`, selain itu `false`. |

### Route (`routes/web.php`)

| Method | URL | Proses | Output |
|--------|-----|--------|--------|
| GET | `/` | Menjalankan closure | Menampilkan view `welcome` |

## Debugging

Catatan masalah yang ditemukan pada kode, penyebabnya, dan cara memperbaikinya.

### 1. Relasi balik belum ada pada `Book` dan `User`

- **Gejala:** `$book->loans` atau `$user->loans` menghasilkan error karena relasi tidak ditemukan.
- **Penyebab:** hanya model `Loan` yang punya relasi (`belongsTo`), sedangkan sisi sebaliknya belum dibuat.
- **Perbaikan:** tambahkan `hasMany` di kedua model.

```php
// app/Models/Book.php dan app/Models/User.php
use Illuminate\Database\Eloquent\Relations\HasMany;

public function loans(): HasMany
{
    return $this->hasMany(Loan::class);
}
```

### 2. Nilai `status` mudah salah ketik

- **Gejala:** data peminjaman tidak muncul saat difilter, misalnya mencari `dikembalikan` padahal tersimpan `dikembali`.
- **Penyebab:** `status` berupa string bebas.
- **Perbaikan:** simpan nilainya sebagai konstanta dan pakai di seluruh kode.

```php
// app/Models/Loan.php
public const STATUS_DIPINJAM = 'dipinjam';
public const STATUS_KEMBALI  = 'dikembalikan';

// pemakaian
Loan::where('status', Loan::STATUS_DIPINJAM)->get();
```

### 3. Stok buku belum terhubung dengan peminjaman

- **Gejala:** stok tidak berubah setelah buku dipinjam, atau bisa menjadi minus.
- **Penyebab:** belum ada logika yang mengubah `stock` saat pinjam dan kembali.
- **Usulan perbaikan:** cek stok, kurangi stok, dan simpan peminjaman dalam satu transaksi.

```php
use Illuminate\Support\Facades\DB;

DB::transaction(function () use ($book, $user) {
    if ($book->stock < 1) {
        abort(422, 'Stok buku habis');
    }

    $book->decrement('stock');

    Loan::create([
        'user_id'   => $user->id,
        'book_id'   => $book->id,
        'loan_date' => now(),
        'due_date'  => now()->addDays(7),
        'status'    => Loan::STATUS_DIPINJAM,
    ]);
});
```

### 4. Controller dan route belum dibuat

- **Gejala:** fitur books dan loans tidak bisa dibuka dari browser (halaman 404).
- **Penyebab:** `routes/web.php` baru berisi halaman `/`.
- **Perbaikan:** buat `BookController` dan `LoanController`, lalu daftarkan route-nya.

```bash
php artisan make:controller BookController --resource
php artisan make:controller LoanController
```

### Cara melacak bug

| Kebutuhan | Perintah |
|-----------|----------|
| Menampilkan isi variabel lalu menghentikan program | `dd($loan);` |
| Menampilkan isi variabel tanpa menghentikan program | `dump($request->all());` |
| Mencatat ke log | `Log::info('data pinjam', $data);` lalu baca `storage/logs/laravel.log` |
| Melihat daftar route | `php artisan route:list` |
| Melihat status migration | `php artisan migrate:status` |
| Mencoba model langsung | `php artisan tinker` lalu `Loan::with('user','book')->first();` |
| Menampilkan pesan error lengkap | atur `APP_DEBUG=true` di `.env` (hanya saat pengembangan) |

## Error Umum

**`SQLSTATE[HY000] [2002] Connection refused`**
MySQL belum menyala. Jalankan MySQL di XAMPP Control Panel, lalu cek `DB_HOST` dan `DB_PORT` di `.env`.

**`SQLSTATE[HY000] [1049] Unknown database`**
Database belum dibuat atau namanya berbeda. Buat database di phpMyAdmin dan samakan dengan `DB_DATABASE` di `.env`.

**`SQLSTATE[HY000] [1045] Access denied for user`**
Username atau password database di `.env` salah. Pada XAMPP bawaan: `DB_USERNAME=root` dan `DB_PASSWORD=` (kosong).

**`Failed to open the referenced table` / `Base table or view not found` saat migrate**
Migration `loans` berjalan sebelum `users` atau `books`. Pastikan timestamp pada nama file migration `loans` paling akhir, lalu jalankan `php artisan migrate:fresh`.

**`SQLSTATE[23000]: Integrity constraint violation`**
`user_id` atau `book_id` yang dimasukkan tidak ada di tabel induk. Pastikan data user dan buku sudah ada sebelum membuat peminjaman.

**`MassAssignmentException` atau data tidak tersimpan**
Kolom belum ada di `$fillable` pada model. Tambahkan kolom tersebut.

**`No application encryption key has been specified`**
Jalankan `php artisan key:generate`.

**`419 Page Expired` saat submit form**
Form belum memiliki token CSRF. Tambahkan `@csrf` di dalam tag `<form>`.

**`Class "App\Models\Loan" not found`**
Namespace atau nama file tidak sesuai. Periksa keduanya, lalu jalankan `composer dump-autoload`.

**`404 Not Found` pada halaman tertentu**
Route belum terdaftar atau salah ketik. Periksa dengan `php artisan route:list`.

**Perubahan `.env` tidak terbaca**
Jalankan `php artisan config:clear`.

## Instalasi

1. Clone repository

   ```bash
   git clone https://github.com/kalilasyifazzahra/peminjaman-buku.git
   cd peminjaman-buku
   ```

2. Install dependency

   ```bash
   composer install
   ```

3. Salin file environment dan generate key

   ```bash
   cp .env.example .env
   php artisan key:generate
   ```

4. Buat database (misalnya `peminjaman_buku`) di phpMyAdmin, lalu atur `.env`

   ```env
   DB_CONNECTION=mysql
   DB_HOST=127.0.0.1
   DB_PORT=3306
   DB_DATABASE=peminjaman_buku
   DB_USERNAME=root
   DB_PASSWORD=
   ```

5. Jalankan migrasi

   ```bash
   php artisan migrate
   ```

6. Jalankan aplikasi

   ```bash
   php artisan serve
   ```

   Buka `http://127.0.0.1:8000` di browser.

## Database

File database tersedia di `database/peminjaman_buku.sql`.
Import lewat phpMyAdmin (tab Import), atau jalankan `php artisan migrate`.

## Laporan Evaluasi

### Tujuan

Membuat aplikasi berbasis Laravel untuk mengelola data buku dan transaksi peminjaman buku perpustakaan.

### Hasil yang Dicapai

- Perancangan database: tabel `users`, `books`, dan `loans` beserta ERD (relasi one-to-many).
- Migration dan model `Book`, `Loan`, `User`, dengan relasi `Loan` ke `User` dan `Book`.
- Pembedaan peran pengguna lewat kolom `role` (`admin` dan `siswa`).
- Dokumentasi: deskripsi program, dokumentasi fungsi, dan panduan debugging.
- File database `database/peminjaman_buku.sql`.
- Project tersimpan di GitHub.

### Kendala

- Push ke GitHub sempat gagal (`Repository not found` dan `rejected`), diselesaikan dengan memperbaiki remote dan `git pull --rebase`.
- Urutan migration perlu dicek agar tabel `loans` dibuat setelah `users` dan `books`.

### Kekurangan

- Controller, route, dan tampilan untuk books dan loans belum dibuat.
- Relasi `loans()` pada model `Book` dan `User` belum ada.
- Logika stok dan perhitungan denda belum diimplementasikan.
- Belum ada pengujian fitur.

### Rencana Perbaikan

1. Tambahkan relasi `hasMany` pada `Book` dan `User`.
2. Buat `BookController` (CRUD) dan `LoanController` (pinjam, kembali, denda).
3. Kurangi stok saat pinjam dan tambah saat kembali, di dalam `DB::transaction()`.
4. Gunakan konstanta atau enum untuk `status`.
5. Tambahkan login dan pembatasan akses admin lewat `isAdmin()`.

### Kesimpulan

Fondasi aplikasi (struktur database, model, dan dokumentasi) sudah selesai. Tahap berikutnya adalah mengimplementasikan fitur utama agar aplikasi dapat digunakan.

## Lisensi

Proyek ini menggunakan lisensi [MIT](https://opensource.org/licenses/MIT).
