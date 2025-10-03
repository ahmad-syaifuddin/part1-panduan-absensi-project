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

Perintah Artisan:

```Bash
php artisan make:migration add_role_to_users_table
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

Jalankan Migrasi:

```Bash
php artisan migrate
```
