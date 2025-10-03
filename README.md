# panduan-absensi-project
Panduan Membuat Project Laravel (Absensi) Laravel 10 with breeze

💻 Langkah 1: Instalasi & Konfigurasi Awal
Kita akan memulai dengan instalasi proyek, Breeze, dan penyesuaian yang Anda sebutkan.

1.1. Instalasi Laravel dan Breeze
Jalankan perintah berikut di Laragon Anda untuk membuat proyek Laravel 10 dan menginstal Laravel Breeze versi 1.19.

# Buat proyek Laravel baru
```
composer create-project laravel/laravel absensi "10.*"
```
# Masuk ke direktori proyek
```
cd absensi
```
# Instal Laravel Breeze versi 1.19
```
composer require laravel/breeze:^1.19 --dev
```
# Jalankan Breeze dengan stack Blade
```
php artisan breeze:install blade
```
1.2. Penyesuaian postcss.config.js
Beberapa versi Node.js atau paket bundler mungkin memerlukan format CommonJS. Mengubah ekstensi file ini dapat mengatasi potensi kesalahan.

Ubah nama file postcss.config.js menjadi postcss.config.cjs.
# Ubah file postcss.js menjadi postcss.cjs
```
mv postcss.config.js postcss.config.cjs 
```
# sekarang kita liat hasil tampilannya dulu "is magic"
# jalan mesin php artisan serve di terminal pertama 
```
php artisan serve
```
# build frontend assets (jalan kan mesin untuk ui asset) di terminal kedua
```
npm run dev
```
⚙️ Langkah 2: Migrasi, Role, dan Seeder
Setelah proyek siap, kita akan membuat struktur data dasar untuk membedakan pengguna Admin dan Karyawan.

2.1. Menambahkan Kolom role ke Tabel users
Kita akan membuat migrasi baru untuk menambahkan kolom role pada tabel users yang sudah dibuat oleh Breeze.
### buat terminal ketiga untuk menangani perintah artisan
Perintah Artisan:

```Bash
php artisan make:migration add_details_to_users_table
```
Kode Migrasi:
Buka file migrasi yang baru saja dibuat di database/migrations/ dan tambahkan kode berikut di dalam metode up() dan down().

```PHP
use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::table('users', function (Blueprint $table) {
            // Kolom role untuk otorisasi
            $table->string('role', 20)->default('karyawan')->after('name');

            // Data tambahan untuk kebutuhan Karyawan/Absensi
            $table->string('phone', 15)->nullable()->after('email');
            $table->enum('gender', ['Laki-laki', 'Perempuan'])->nullable()->after('phone');
        });
    }

    public function down(): void
    {
        Schema::table('users', function (Blueprint $table) {
            $table->dropColumn(['role', 'phone', 'gender']);
        });
    }
};
```

### pastikan isi dari file .env begini sebelum migrate :
ubah var untuk APP_NAME dan DB_DATABASE
```
APP_NAME=Absensi
APP_ENV=local
APP_KEY=base64:adsaiosdiosdisdiodiosdsdsd
APP_DEBUG=true
APP_URL=http://localhost

LOG_CHANNEL=stack
LOG_DEPRECATIONS_CHANNEL=null
LOG_LEVEL=debug

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=absensi
DB_USERNAME=root
DB_PASSWORD=
```


Jalankan Migrasi:

```Bash
php artisan migrate
```

# 2.2. Membuat Seeder untuk Pengguna Awal
Kita akan membuat seeder untuk menambahkan pengguna admin dan karyawan secara otomatis ke database.

Perintah Artisan:

```Bash
php artisan make:seeder UserSeeder
```
Kode Seeder:
Buka file database/seeders/UserSeeder.php dan isi dengan kode berikut:

```PHP
<?php

namespace Database\Seeders;

use Illuminate\Database\Seeder;
use App\Models\User;
use Illuminate\Support\Facades\Hash;

class UserSeeder extends Seeder
{
    public function run(): void
    {
        // 1. Buat user Admin (Data tetap)
        User::create([
            'name' => 'Admin Utama',
            'email' => 'admin@gmail.com', // Revisi email
            'password' => Hash::make('password'),
            'role' => 'admin',
            'phone' => '081234567890',
            'gender' => 'Laki-laki',
        ]);

        // 2. Buat user Karyawan (Data tetap)
        User::create([
            'name' => 'Budi Karyawan',
            'email' => 'budi.karyawan@gmail.com', // Revisi email
            'password' => Hash::make('password'),
            'role' => 'karyawan',
            'phone' => '089876543210',
            'gender' => 'Laki-laki',
        ]);

        // 3. Buat 10 Data Dummy Karyawan menggunakan Factory
        User::factory()->count(10)->create();
    }
}
```
Integrasi Seeder ke DatabaseSeeder:
Buka file database/seeders/DatabaseSeeder.php dan tambahkan pemanggilan UserSeeder di dalam metode run().

```PHP
public function run(): void
{
    $this->call([
        UserSeeder::class,
    ]);
}
```

### Jalankan Seeder:
Jalankan perintah berikut untuk mengisi database Anda dengan data pengguna awal.

```Bash
php artisan db:seed
```


🔐 Langkah 3: Middleware Berbasis Role
Untuk melindungi rute dan halaman khusus admin, kita akan membuat middleware sederhana yang mengecek peran (role) pengguna.

3.1. Membuat Middleware
Perintah Artisan:

```Bash
php artisan make:middleware AdminMiddleware
```
Kode Middleware:
Buka file app/Http/Middleware/AdminMiddleware.php dan ubah metode handle() seperti ini:

```PHP
public function handle(Request $request, Closure $next): Response
{
    // Cek apakah pengguna terotentikasi dan memiliki role 'admin'
    if (auth()->check() && auth()->user()->role === 'admin') {
        return $next($request);
    }

    // Redirect atau beri respon 403 jika tidak memiliki role 'admin'
    return redirect('/dashboard')->with('error', 'Akses ditolak. Anda tidak memiliki hak akses administrator.');
}
```
3.2. Mendaftarkan Middleware
Buka file app/Http/Kernel.php dan daftarkan alias untuk middleware Anda.

Tambahkan baris berikut di dalam properti $middlewareAliases:

```PHP

protected $middlewareAliases = [
    // ... kode Breeze yang sudah ada
    'auth' => \App\Http\Middleware\Authenticate::class,
    'guest' => \App\Http\Middleware\RedirectIfAuthenticated::class,
    'admin' => \App\Http\Middleware\AdminMiddleware::class, // <-- Tambahkan baris ini
];
```
