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
php artisan make:migration add_role_to_users_table
```
Kode Migrasi:
Buka file migrasi yang baru saja dibuat di database/migrations/ dan tambahkan kode berikut di dalam metode up() dan down().

```PHP
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::table('users', function (Blueprint $table) {            
            // Data tambahan untuk kebutuhan Karyawan/Absensi
            $table->enum('role', ['admin', 'karyawan'])->default('karyawan')->after('email');
        });
    }

    public function down(): void
    {
        Schema::table('users', function (Blueprint $table) {
            $table->dropColumn(['role']);
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

Buat Model dan Migrasi Employee:
Tabel ini akan menyimpan data detail karyawan yang terhubung dengan tabel users.

```Bash
php artisan make:model Employee -m
```
Buka file migrasi employees yang baru dibuat dan isi dengan kode berikut:

```PHP

<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('employees', function (Blueprint $table) {
            $table->id();
            $table->foreignId('user_id')->constrained()->onDelete('cascade');
            $table->string('nama_lengkap');
            $table->string('nip')->unique();
            $table->string('posisi');
            $table->string('jabatan');
            $table->date('tanggal_perekrutan');
            $table->string('no_hp');
            $table->text('alamat');
            $table->enum('jenis_kelamin', ['Laki-laki', 'Perempuan']);
            $table->enum('status', ['aktif', 'tidak aktif', 'dihentikan'])->default('aktif');
            $table->timestamps();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('employees');
    }
};
```
Buat Model dan Migrasi Attendance & Holiday (Persiapan):
Kita buat sekarang agar struktur database lengkap.

```
php artisan make:model Attendance -m
```

```
php artisan make:model Holiday -m
```
Isi file migrasi attendances:

```PHP
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('attendances', function (Blueprint $table) {
            $table->id();
            $table->foreignId('employee_id')->constrained()->onDelete('cascade');
            $table->date('date');
            $table->time('time_in')->nullable();
            $table->time('time_out')->nullable();
            $table->enum('status', ['Hadir', 'Terlambat', 'Izin', 'Alpa'])->default('Alpa');
            $table->text('notes')->nullable();
            $table->timestamps();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('attendances');
    }
};
```
Isi file migrasi holidays:

```PHP
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('holidays', function (Blueprint $table) {
            $table->id();
            $table->date('date')->unique();
            $table->string('description');
            $table->timestamps();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('holidays');
    }
};
```
Jalankan Migrasi:
Sekarang, jalankan perintah ini untuk membuat semua tabel di database Anda.

```Bash
php artisan migrate
```
Langkah 3: Relasi Model, Factory, dan Seeder
Kita akan membuat data dummy agar aplikasi kita tidak kosong.

Definisikan Relasi Model:
Buka app/Models/User.php dan tambahkan relasi hasOne ke Employee. Juga tambahkan role ke $fillable.

```PHP
// app/Models/User.php
namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Relations\HasOne;
use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;
use Laravel\Sanctum\HasApiTokens;

class User extends Authenticatable
{
    use HasApiTokens, HasFactory, Notifiable;

    protected $fillable = [
        'name',
        'email',
        'password',
        'role', // Tambahkan ini
    ];

    // ... (kode lainnya)

    /**
     * Get the employee associated with the user.
     */
    public function employee(): HasOne
    {
        return $this->hasOne(Employee::class);
    }
}
```
Buka app/Models/Employee.php dan tambahkan relasi belongsTo ke User, serta definisikan properti $fillable.

```PHP
// app/Models/Employee.php
<?php
namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

class Employee extends Model
{
    use HasFactory;

    protected $fillable = [
        'user_id',
        'nama_lengkap',
        'nip',
        'posisi',
        'jabatan',
        'tanggal_perekrutan',
        'no_hp',
        'alamat',
        'jenis_kelamin',
        'status',
    ];

    /**
     * Get the user that owns the employee.
     */
    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }
}
```
Modifikasi UserFactory:
Buka database/factories/UserFactory.php dan tambahkan role.

```PHP
<?php

namespace Database\Factories;

use Illuminate\Support\Facades\Hash;
use Illuminate\Support\Str;
use Illuminate\Database\Eloquent\Factories\Factory;
use Faker\Factory as FakerFactory;

class UserFactory extends Factory
{
    protected static ?string $password;

    public function definition(): array
    {
        $faker = FakerFactory::create('id_ID');
        return [
            'name' => $faker->name(),
            'email' => $faker->unique()->safeEmail(),
            'email_verified_at' => now(),
            'password' => static::$password ??= Hash::make('password'),
            'remember_token' => Str::random(10),
            'role' => 'karyawan', // Default role adalah karyawan
        ];
    }

    public function unverified(): static
    {
        return $this->state(fn (array $attributes) => [
            'email_verified_at' => null,
        ]);
    }
}
```
Buat EmployeeFactory:

```Bash
php artisan make:factory EmployeeFactory --model=Employee
```
Buka database/factories/EmployeeFactory.php dan isi dengan logika untuk membuat data dummy karyawan.

```PHP
<?php
namespace Database\Factories;

// Tambahkan use statement ini
use Faker\Factory as FakerFactory;
use Illuminate\Database\Eloquent\Factories\Factory;

class EmployeeFactory extends Factory
{
    public function definition(): array
    {
        // Inisiasi Faker dengan lokal Indonesia
        $faker = FakerFactory::create('id_ID');

        return [
            // Gunakan $faker yang sudah disetel
            'nama_lengkap' => $faker->name(),
            'nip' => $faker->unique()->numerify('##################'), // NIP biasanya 18 digit
            'posisi' => $faker->jobTitle(),
            'jabatan' => $faker->randomElement(['Staff', 'Supervisor', 'Manajer', 'Direktur']),
            'tanggal_perekrutan' => $faker->date(),
            'no_hp' => $faker->phoneNumber(),
            'alamat' => $faker->address(),
            'jenis_kelamin' => $faker->randomElement(['Laki-laki', 'Perempuan']),
            'status' => 'aktif',
        ];
    }
}
```

Konfigurasi UserSeeder:
Buka database/seeders/UserSeeder.php. kita akan merevisi isi kodenya sebagai berikut :
```php
<?php

namespace Database\Seeders;

use App\Models\Employee;
use Illuminate\Database\Seeder;
use App\Models\User;
use Illuminate\Support\Facades\Hash;
use Illuminate\Support\Str;
use Faker\Factory as Faker;

class UserSeeder extends Seeder
{
    public function run(): void
    {
        // Membuat 1 User Admin
        User::factory()->create([
            'name' => 'Admin Utama',
            'email' => 'admin@gmail.com',
            'role' => 'admin',
            'password' => Hash::make('password'),
        ]);

        // Membuat 1 User Karyawan untuk testing
        User::factory()->create([
            'name' => 'Test Karyawan',
            'email' => 'karyawan@gmail.com',
            'role' => 'karyawan',
            'password' => Hash::make('password'),
        ]);

        // Membuat 10 Karyawan dummy menggunakan factory
        User::factory(10)->create()->each(function ($user) {
            // Untuk setiap user yang dibuat, buatkan juga data employee
            Employee::factory()->create(['user_id' => $user->id]);
        });
    }
}
```

Konfigurasi DatabaseSeeder:
Buka database/seeders/DatabaseSeeder.php. Kita akan membuat 1 Admin dan 10 Karyawan. Ganti email agar mudah diingat, dan pastikan setiap user memiliki data employee. Kita panggil (CALL)

```PHP
<?php

namespace Database\Seeders;

use App\Models\Employee; // Pastikan ini di-import
use App\Models\User;     // Pastikan ini di-import
use Illuminate\Database\Seeder;
use Illuminate\Support\Facades\Hash; // Pastikan ini di-import

class DatabaseSeeder extends Seeder
{
    public function run(): void
    {
        // Membuat 1 User Admin
        // Kita tidak perlu membuat data employee untuk admin
        User::factory()->create([
            'name' => 'Admin Utama',
            'email' => 'admin@gmail.com',
            'role' => 'admin',
            'password' => Hash::make('password'),
        ]);

        // Membuat 1 User Karyawan untuk testing dengan data employee
        User::factory()->has(Employee::factory()->state([
            'nama_lengkap' => 'Karyawan Test',
            'nip' => '199001012020121001'
        ]))->create([
            'name' => 'Karyawan Test',
            'email' => 'karyawan@gmail.com',
            'role' => 'karyawan',
            'password' => Hash::make('password'),
        ]);


        // Membuat 10 Karyawan dummy menggunakan factory
        // has(Employee::factory()) akan otomatis membuat data employee
        // untuk setiap user yang dibuat dan mengisikan user_id-nya.
        User::factory(10)->has(Employee::factory())->create();
    }
}
```

ubah kode model User.php seperti ini :
```php
<?php

namespace App\Models;

// use Illuminate\Contracts\Auth\MustVerifyEmail;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;
use Laravel\Sanctum\HasApiTokens;
use App\Models\Employee;

class User extends Authenticatable
{
    use HasApiTokens, HasFactory, Notifiable;

    /**
     * The attributes that are mass assignable.
     *
     * @var array<int, string>
     */
    protected $fillable = [
        'name',
        'email',
        'password',
        'role',
    ];

    /**
     * The attributes that should be hidden for serialization.
     *
     * @var array<int, string>
     */
    protected $hidden = [
        'password',
        'remember_token',
    ];

    public function employee()
    {
        return $this->hasOne(Employee::class);
    }

    /**
     * The attributes that should be cast.
     *
     * @var array<string, string>
     */
    protected $casts = [
        'email_verified_at' => 'datetime',
        'password' => 'hashed',
    ];
}
```

Jalankan Seeder:
Perintah ini akan menghapus semua data lama dan mengisi database dengan data dari seeder.

```Bash
php artisan migrate:fresh --seed
```
Sekarang database Anda sudah terisi dengan 1 admin, 1 karyawan testing, dan 10 karyawan dummy.

Admin Login: admin@gmail.com / password

Karyawan Login: karyawan@gmail.com / password

Sampai di sini, fondasi aplikasi kita sudah sangat kuat. Kita sudah punya:

Proyek Laravel dengan otentikasi.

Struktur database yang lengkap.

Data dummy untuk pengembangan.

---

## Langkah 4: Membuat Middleware untuk Role
Middleware ini berfungsi seperti satpam. Jika pengguna yang mencoba mengakses halaman manajemen pengguna bukan 'admin', sistem akan otomatis menolaknya.

### 4.1. Buat File Middleware
Jalankan perintah ini di terminal Anda:

```Bash
php artisan make:middleware RoleMiddleware
```
Perintah ini akan membuat file baru di app/Http/Middleware/RoleMiddleware.php.

### 4.2. Isi Logika Middleware
Buka file RoleMiddleware.php yang baru saja dibuat dan ubah isinya menjadi seperti ini:

```PHP
<?php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;
use Symfony\Component\HttpFoundation\Response;

class RoleMiddleware
{
    /**
     * Handle an incoming request.
     *
     * @param  \Closure(\Illuminate\Http\Request): (\Symfony\Component\HttpFoundation\Response)  $next
     */
    public function handle(Request $request, Closure $next, ...$roles): Response
    {
        // Cek dulu apakah pengguna sudah login atau belum.
        if (!Auth::check()) {
            return redirect('login');
        }

        // Ambil role dari pengguna yang sedang login.
        $userRole = Auth::user()->role;

        // Cek apakah role pengguna ada di dalam daftar role yang diizinkan ($roles).
        if (in_array($userRole, $roles)) {
            // Jika cocok, izinkan pengguna melanjutkan request.
            return $next($request);
        }

        // Jika tidak cocok, tolak akses (Forbidden).
        abort(403, 'ANDA TIDAK MEMILIKI AKSES KE HALAMAN INI.');
    }
}
```

### 4.3. Daftarkan Middleware
Agar bisa kita panggil dengan nama role, daftarkan middleware ini di app/Http/Kernel.php. Tambahkan satu baris di dalam array $middlewareAliases.

```PHP
// app/Http/Kernel.php

protected $middlewareAliases = [
    'auth' => \App\Http\Middleware\Authenticate::class,
    'auth.session' => \Illuminate\Session\Middleware\AuthenticateSession::class,
    'cache.headers' => \Illuminate\Http\Middleware\SetCacheHeaders::class,
    'can' => \Illuminate\Auth\Middleware\Authorize::class,
    'guest' => \App\Http\Middleware\RedirectIfAuthenticated::class,
    'precognitive' => \Illuminate\Foundation\Http\Middleware\HandlePrecognitiveRequests::class,
    'throttle' => \Illuminate\Routing\Middleware\ThrottleRequests::class,
    'verified' => \Illuminate\Auth\Middleware\EnsureEmailIsVerified::class,

    // Tambahkan baris ini di akhir
    'role' => \App\Http\Middleware\RoleMiddleware::class,
];
```

## Langkah 5: Controller dan Rute untuk Manajemen Pengguna
Di sini kita akan membuat "otak" (Controller) yang mengatur logika CRUD dan "alamat" (Rute) untuk mengakses halaman-halaman tersebut.

### 5.1. Buat Controller
Gunakan flag --resource agar Laravel otomatis membuatkan method-method standar CRUD (index, create, store, show, edit, update, destroy).

```Bash
php artisan make:controller UserController --resource
```
Ini akan membuat file app/Http/Controllers/UserController.php.

### 5.2. Definisikan Rute
Buka file routes/web.php dan tambahkan rute untuk manajemen pengguna. Kita akan meletakkannya di dalam grup middleware auth dan role:admin agar aman.

```PHP
<?php

use App\Http\Controllers\ProfileController;
use App\Http\Controllers\UserController; // <-- Tambahkan import ini
use Illuminate\Support\Facades\Route;

/*
|--------------------------------------------------------------------------
| Web Routes
|--------------------------------------------------------------------------
*/

Route::get('/', function () {
    return view('welcome');
});

Route::get('/dashboard', function () {
    return view('dashboard');
})->middleware(['auth', 'verified'])->name('dashboard');

// Rute bawaan Breeze untuk profil
Route::middleware('auth')->group(function () {
    Route::get('/profile', [ProfileController::class, 'edit'])->name('profile.edit');
    Route::patch('/profile', [ProfileController::class, 'update'])->name('profile.update');
    Route::delete('/profile', [ProfileController::class, 'destroy'])->name('profile.destroy');
});


// === RUTE KHUSUS ADMIN UNTUK MANAJEMEN PENGGUNA ===
Route::middleware(['auth', 'role:admin'])->group(function () {
    // Rute resource akan otomatis membuat semua URL CRUD untuk UserController
    // Contoh: /users, /users/create, /users/{user}/edit, dll.
    Route::resource('users', UserController::class);
});


require __DIR__.'/auth.php';
```

### 5.3. Isi Logika Awal di Controller (index)
Sekarang, kita isi method index di UserController.php untuk mengambil dan menampilkan daftar semua pengguna.

Buka app/Http/Controllers/UserController.php dan modifikasi seperti ini:

```PHP
<?php

namespace App\Http\Controllers;

use App\Models\User; // <-- Tambahkan import ini
use Illuminate\Http\Request;

class UserController extends Controller
{
    /**
     * Display a listing of the resource.
     */
    public function index()
    {
        // Ambil semua data user, diurutkan dari yang terbaru.
        // Gunakan 'with('employee')' untuk mengambil relasi employee (Eager Loading).
        // Gunakan paginate() untuk membatasi data per halaman.
        $users = User::with('employee')->latest()->paginate(10);

        // Kirim data users ke view 'users.index'
        return view('users.index', compact('users'));
    }

    // method-method lain akan kita isi pada tahap berikutnya...
}
```
Penjelasan Singkat:

User::with('employee'): Ini adalah teknik Eager Loading. Kita memberitahu Laravel untuk mengambil data User sekaligus data Employee yang terhubung. Ini sangat efisien dan mencegah banyak query ke database.

latest()->paginate(10): Mengurutkan data dari yang paling baru dibuat dan menampilkannya 10 data per halaman.

return view('users.index', ...): Mengirim data $users ke file view yang akan kita buat selanjutnya di resources/views/users/index.blade.php.

Selesai! 👏 Kita sudah berhasil:

Membuat Middleware untuk memproteksi rute admin.

Mempersiapkan Controller dan Rute untuk fitur manajemen pengguna.

Logika di belakang layar sudah siap. Langkah selanjutnya adalah bagian yang paling seru: membangun tampilan (view) CRUD menggunakan Modern Blade Component.

---

# Blade Views untuk CRUD
```
resources/views/users/index.blade.php
```

```
resources/views/users/create.blade.php
```

```
resources/views/users/edit.blade.php
```

```
resources/views/users/show.blade.php
```

# Tahap 3: Membangun View CRUD - Halaman Daftar Pengguna (Index)
Pada tahap ini, kita akan membuat file Blade untuk menampilkan data yang sudah kita siapkan di UserController@index.

## Langkah 6: Menambahkan Font Awesome
Sesuai rencana, kita akan menggunakan ikon dari Font Awesome. Cara termudah adalah menambahkannya melalui CDN ke layout utama aplikasi kita.

Buka file layout utama di resources/views/layouts/app.blade.php.

Tambahkan baris kode CDN berikut di dalam tag <head>, di bawah baris <link> yang sudah ada.

```HTML
<link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.5.2/css/all.min.css" integrity="sha512-SnH5WK+bZxgPHs44uWIX+LLJAJ9/2PkPKZ5QiAj6Ta86w+fsb2TkcmfRyVX3pBnMFcV7oQPJkl9QevSCWr3W6A==" crossorigin="anonymous" referrerpolicy="no-referrer" />
```
Contoh penempatannya:

```HTML
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <meta name="csrf-token" content="{{ csrf_token() }}">

    <title>{{ config('app.name', 'Laravel') }}</title>

    <link rel="preconnect" href="https://fonts.bunny.net">
    <link href="https://fonts.bunny.net/css?family=figtree:400,500,600&display=swap" rel="stylesheet" />

    @vite(['resources/css/app.css', 'resources/js/app.js'])

    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.5.2/css/all.min.css" integrity="sha512-SnH5WK+bZxgPHs44uWIX+LLJAJ9/2PkPKZ5QiAj6Ta86w+fsb2TkcmfRyVX3pBnMFcV7oQPJkl9QevSCWr3W6A==" crossorigin="anonymous" referrerpolicy="no-referrer" />
</head>
```

Sekarang kita akan membuat fungsionalitas di balik tombol "Tambah Pengguna". Ini melibatkan dua bagian utama: halaman formulir (create) untuk mengisi data, dan logika di controller (store) untuk memvalidasi dan menyimpan data tersebut ke database.

## Tahap 4: Fungsionalitas Tambah Pengguna
### Langkah 8: Membuat Form Tambah Pengguna (Metode create dan View)
Pertama, kita siapkan halaman yang berisi formulir input.

8.1. Update UserController (Metode create)

Buka app/Http/Controllers/UserController.php. Tugas metode create() sangat sederhana: hanya menampilkan file view yang berisi formulir.

```PHP

// app/Http/Controllers/UserController.php

// ... (method index) ...

/**
 * Show the form for creating a new resource.
 */
public function create()
{
    // Cukup kembalikan view yang berisi form
    return view('users.create');
}

// ... (method store dan lainnya) ...
```

### View Index.blade.php (manajemen user):
```blade
<x-app-layout>
    <x-slot name="header">
        <h2 class="font-semibold text-xl text-gray-800 leading-tight">
            {{ __('Manajemen Pengguna') }}
        </h2>
    </x-slot>

    <div class="py-12">
        <div class="max-w-7xl mx-auto sm:px-6 lg:px-8">
            <div class="bg-white overflow-hidden shadow-sm sm:rounded-lg">
                <div class="p-6 text-gray-900">

                    <div class="mb-4">
                        <a href="{{ route('users.create') }}" class="bg-blue-500 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded">
                            <i class="fas fa-plus"></i> Tambah Pengguna
                        </a>
                    </div>

                    <div class="overflow-x-auto">
                        <table class="min-w-full divide-y divide-gray-200">
                            <thead class="bg-gray-50">
                                <tr>
                                    <th scope="col" class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">No</th>
                                    <th scope="col" class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Nama & Email</th>
                                    <th scope="col" class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">NIP</th>
                                    <th scope="col" class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Role</th>
                                    <th scope="col" class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Aksi</th>
                                </tr>
                            </thead>
                            <tbody class="bg-white divide-y divide-gray-200">
                                @forelse ($users as $index => $user)
                                    <tr>
                                        <td class="px-6 py-4 whitespace-nowrap">{{ $index + $users->firstItem() }}</td>
                                        <td class="px-6 py-4 whitespace-nowrap">
                                            <div class="text-sm font-medium text-gray-900">{{ $user->name }}</div>
                                            <div class="text-sm text-gray-500">{{ $user->email }}</div>
                                        </td>
                                        <td class="px-6 py-4 whitespace-nowrap">
                                            <span class="px-2 inline-flex text-xs leading-5 font-semibold rounded-full bg-gray-100 text-gray-800">
                                                {{ $user->employee?->nip ?? '-' }}
                                            </span>
                                        </td>
                                        <td class="px-6 py-4 whitespace-nowrap">
                                            @if ($user->role == 'admin')
                                                <span class="px-2 inline-flex text-xs leading-5 font-semibold rounded-full bg-red-100 text-red-800">
                                                    Admin
                                                </span>
                                            @else
                                                <span class="px-2 inline-flex text-xs leading-5 font-semibold rounded-full bg-green-100 text-green-800">
                                                    Karyawan
                                                </span>
                                            @endif
                                        </td>
                                        <td class="px-6 py-4 whitespace-nowrap text-sm font-medium">
                                            <a href="{{ route('users.show', $user->id) }}" class="text-blue-600 hover:text-blue-900" title="Lihat"><i class="fas fa-eye"></i></a>
                                            <a href="{{ route('users.edit', $user->id) }}" class="text-indigo-600 hover:text-indigo-900 ml-4" title="Edit"><i class="fas fa-edit"></i></a>
                                            <form action="{{ route('users.destroy', $user->id) }}" method="POST" class="inline-block" onsubmit="return confirm('Apakah Anda yakin ingin menghapus pengguna ini?');">
                                                @csrf
                                                @method('DELETE')
                                                <button type="submit" class="text-red-600 hover:text-red-900 ml-4" title="Hapus"><i class="fas fa-trash"></i></button>
                                            </form>
                                        </td>
                                    </tr>
                                @empty
                                    <tr>
                                        <td colspan="5" class="px-6 py-4 whitespace-nowrap text-center text-sm text-gray-500">
                                            Data pengguna tidak ditemukan.
                                        </td>
                                    </tr>
                                @endforelse
                            </tbody>
                        </table>
                    </div>

                    <div class="mt-4">
                        {{ $users->links() }}
                    </div>

                </div>
            </div>
        </div>
    </div>
</x-app-layout>
```

## 8.2. Buat View users.create.blade.php

Buat file baru di resources/views/users/ bernama create.blade.php. Isi dengan kode berikut. Kode ini menggunakan komponen-komponen Blade yang sudah disediakan Breeze (x-input-label, x-text-input, dll.) agar tampilan kita konsisten.

```HTML
<x-app-layout>
    <x-slot name="header">
        <h2 class="font-semibold text-xl text-gray-800 leading-tight">
            {{ __('Tambah Pengguna Baru') }}
        </h2>
    </x-slot>

    <div class="py-12">
        <div class="max-w-7xl mx-auto sm:px-6 lg:px-8">
            <div class="bg-white overflow-hidden shadow-sm sm:rounded-lg">
                <div class="p-6 text-gray-900">
                    
                    @if ($errors->any())
                        <div class="mb-4 bg-red-100 border border-red-400 text-red-700 px-4 py-3 rounded relative" role="alert">
                            <strong class="font-bold">Oops!</strong>
                            <span class="block sm:inline">Ada beberapa masalah dengan input Anda.</span>
                            <ul class="mt-3 list-disc list-inside text-sm text-red-600">
                                @foreach ($errors->all() as $error)
                                    <li>{{ $error }}</li>
                                @endforeach
                            </ul>
                        </div>
                    @endif

                    <form method="POST" action="{{ route('users.store') }}">
                        @csrf

                        <div>
                            <x-input-label for="name" :value="__('Nama')" />
                            <x-text-input id="name" class="block mt-1 w-full" type="text" name="name" :value="old('name')" required autofocus />
                        </div>

                        <div class="mt-4">
                            <x-input-label for="email" :value="__('Email')" />
                            <x-text-input id="email" class="block mt-1 w-full" type="email" name="email" :value="old('email')" required />
                        </div>

                        <div class="mt-4">
                            <x-input-label for="role" :value="__('Role')" />
                            <select name="role" id="role" class="block mt-1 w-full border-gray-300 focus:border-indigo-500 focus:ring-indigo-500 rounded-md shadow-sm">
                                <option value="karyawan">Karyawan</option>
                                <option value="admin">Admin</option>
                            </select>
                        </div>

                        <div class="mt-4">
                            <x-input-label for="password" :value="__('Password')" />
                            <x-text-input id="password" class="block mt-1 w-full" type="password" name="password" required />
                        </div>

                        <div class="mt-4">
                            <x-input-label for="password_confirmation" :value="__('Konfirmasi Password')" />
                            <x-text-input id="password_confirmation" class="block mt-1 w-full" type="password" name="password_confirmation" required />
                        </div>

                        <div class="flex items-center justify-end mt-4">
                            <a href="{{ route('users.index') }}" class="text-sm text-gray-600 hover:text-gray-900 mr-4">
                                Batal
                            </a>
                            <x-primary-button>
                                {{ __('Simpan') }}
                            </x-primary-button>
                        </div>
                    </form>
                </div>
            </div>
        </div>
    </div>
</x-app-layout>
```
### Langkah 9: Logika Penyimpanan Data (Metode store)
Sekarang kita isi "otak"-nya. Metode store() akan menerima data dari form, melakukan validasi, dan jika lolos, menyimpannya ke database.

Buka kembali app/Http/Controllers/UserController.php dan isi method store():

```PHP
// app/Http/Controllers/UserController.php

use Illuminate\Http\Request;
use App\Models\User;
use Illuminate\Support\Facades\Hash; // <-- Tambahkan ini
use Illuminate\Validation\Rules;      // <-- Tambahkan ini

// ... (method create) ...

/**
 * Store a newly created resource in storage.
 */
public function store(Request $request)
{
    // 1. Validasi Input
    $request->validate([
        'name' => ['required', 'string', 'max:255'],
        'email' => ['required', 'string', 'email', 'max:255', 'unique:users'],
        'role' => ['required', 'in:admin,karyawan'],
        'password' => ['required', 'confirmed', Rules\Password::defaults()],
    ]);

    // 2. Buat User Baru
    User::create([
        'name' => $request->name,
        'email' => $request->email,
        'role' => $request->role,
        'password' => Hash::make($request->password), // Enkripsi password
    ]);

    // 3. Redirect ke halaman index dengan pesan sukses
    return redirect()->route('users.index')
                     ->with('success', 'Pengguna baru berhasil ditambahkan.');
}

// ... (method show dan lainnya) ...
```
Penjelasan Logika store():

$request->validate([...]): Ini adalah bagian validasi. Jika ada aturan yang tidak terpenuhi (misal: email sudah ada, atau password tidak sama dengan konfirmasinya), Laravel akan otomatis mengembalikan pengguna ke halaman form dan menampilkan pesan error.

User::create([...]): Jika validasi berhasil, data akan disimpan ke tabel users.

Hash::make(...): Ini sangat penting! Kita tidak pernah menyimpan password sebagai teks biasa. Hash::make akan mengenkripsi password sebelum menyimpannya ke database.

redirect()->...->with(...): Setelah berhasil menyimpan, kita arahkan admin kembali ke halaman daftar pengguna (users.index) sambil membawa "pesan" atau flash message bernama success.

### Langkah 10: Menampilkan Pesan Sukses (Flash Message)
Kita perlu menampilkan flash message yang kita kirim dari controller tadi.

Buka kembali view resources/views/users/index.blade.php dan tambahkan kode berikut tepat di atas tombol "Tambah Pengguna".
```HTML
@if (session('success'))
    <div class="mb-4 bg-green-100 border border-green-400 text-green-700 px-4 py-3 rounded relative" role="alert">
        <strong class="font-bold">Sukses!</strong>
        <span class="block sm:inline">{{ session('success') }}</span>
    </div>
@endif

<div class="mb-4">
    <a href="{{ route('users.create') }}" ...>
```

Uji Coba
Sekarang, mari kita tes fungsionalitas penuh:

Buka halaman /users.

Klik tombol "Tambah Pengguna". Anda akan diarahkan ke form.

Coba submit form dengan data kosong atau password yang tidak cocok. Anda akan melihat pesan error.

Isi formulir dengan benar, lalu klik "Simpan".

Jika berhasil, Anda akan kembali ke halaman daftar pengguna, melihat pesan sukses berwarna hijau, dan pengguna baru tersebut akan muncul di dalam tabel.

--- 

# Sip, kita hajar lagi! 💪
Kita lanjutkan dengan membuat fungsionalitas untuk mengedit data pengguna yang sudah ada. Prosesnya mirip dengan "Tambah Data", tapi kali ini formulirnya sudah terisi dengan data lama dan logikanya adalah untuk memperbarui (update) data, bukan membuat data baru.

## Tahap 5: Fungsionalitas Edit dan Update Pengguna
Langkah 11: Menyiapkan Form Edit (Metode edit dan View)
Pertama, kita siapkan halaman formulir yang sudah terisi data pengguna yang akan diedit.

### 11.1. Update UserController (Metode edit)

Buka app/Http/Controllers/UserController.php. Metode edit() bertugas mencari data pengguna berdasarkan ID yang dikirim melalui URL, lalu mengirimkan data tersebut ke view.

```PHP
// app/Http/Controllers/UserController.php

// ... (method store) ...

/**
 * Show the form for editing the specified resource.
 */
public function edit(User $user)
{
    // Variabel $user sudah otomatis berisi data user yang akan diedit
    // berkat Route Model Binding dari Laravel.
    return view('users.edit', compact('user'));
}

// ... (method update dan lainnya) ...
```
Catatan: Kita menggunakan fitur canggih Laravel bernama Route Model Binding. Dengan mendeklarasikan User $user di argumen metode, Laravel secara otomatis akan mencari User di database dengan id yang sesuai dari URL (/users/{user}/edit). Praktis, kan?

### 11.2. Buat View users.edit.blade.php

Untuk menghemat waktu, kita bisa duplikat file resources/views/users/create.blade.php dan menamainya edit.blade.php. Kemudian, modifikasi isinya agar sesuai untuk proses edit.

Buka file resources/views/users/edit.blade.php yang baru dan ubah isinya menjadi seperti ini:

```HTML
<x-app-layout>
    <x-slot name="header">
        <h2 class="font-semibold text-xl text-gray-800 leading-tight">
            {{ __('Edit Pengguna: ') }} {{ $user->name }}
        </h2>
    </x-slot>

    <div class="py-12">
        <div class="max-w-7xl mx-auto sm:px-6 lg:px-8">
            <div class="bg-white overflow-hidden shadow-sm sm:rounded-lg">
                <div class="p-6 text-gray-900">
                    
                    @if ($errors->any())
                        <div class="mb-4 bg-red-100 border border-red-400 text-red-700 px-4 py-3 rounded relative" role="alert">
                            <strong class="font-bold">Oops!</strong>
                            <ul class="mt-3 list-disc list-inside text-sm text-red-600">
                                @foreach ($errors->all() as $error)
                                    <li>{{ $error }}</li>
                                @endforeach
                            </ul>
                        </div>
                    @endif

                    <form method="POST" action="{{ route('users.update', $user->id) }}">
                        @csrf
                        @method('PUT') <div>
                            <x-input-label for="name" :value="__('Nama')" />
                            <x-text-input id="name" class="block mt-1 w-full" type="text" name="name" :value="old('name', $user->name)" required autofocus />
                        </div>

                        <div class="mt-4">
                            <x-input-label for="email" :value="__('Email')" />
                            <x-text-input id="email" class="block mt-1 w-full" type="email" name="email" :value="old('email', $user->email)" required />
                        </div>

                        <div class="mt-4">
                            <x-input-label for="role" :value="__('Role')" />
                            <select name="role" id="role" class="block mt-1 w-full border-gray-300 focus:border-indigo-500 focus:ring-indigo-500 rounded-md shadow-sm">
                                <option value="karyawan" {{ old('role', $user->role) == 'karyawan' ? 'selected' : '' }}>Karyawan</option>
                                <option value="admin" {{ old('role', $user->role) == 'admin' ? 'selected' : '' }}>Admin</option>
                            </select>
                        </div>

                        <div class="mt-4">
                            <x-input-label for="password" :value="__('Password (Opsional)')" />
                            <x-text-input id="password" class="block mt-1 w-full" type="password" name="password" />
                            <p class="text-sm text-gray-500 mt-1">Kosongkan jika tidak ingin mengubah password.</p>
                        </div>

                        <div class="mt-4">
                            <x-input-label for="password_confirmation" :value="__('Konfirmasi Password')" />
                            <x-text-input id="password_confirmation" class="block mt-1 w-full" type="password" name="password_confirmation" />
                        </div>

                        <div class="flex items-center justify-end mt-4">
                            <a href="{{ route('users.index') }}" class="text-sm text-gray-600 hover:text-gray-900 mr-4">
                                Batal
                            </a>
                            <x-primary-button>
                                {{ __('Update') }}
                            </x-primary-button>
                        </div>
                    </form>
                </div>
            </div>
        </div>
    </div>
</x-app-layout>
```
Perbedaan Utama dari Form create:

action="{{ route('users.update', $user->id) }}": Form akan dikirim ke rute update dengan menyertakan ID pengguna.

@method('PUT'): Karena form HTML hanya mendukung GET dan POST, kita menggunakan method spoofing ini untuk memberitahu Laravel bahwa ini adalah request PUT (untuk update).

**:value="old('name', $user->name)"**: Ini adalah triknya. Helper old() akan mencoba mengambil data dari sesi (jika ada error validasi), jika tidak ada, ia akan menggunakan nilai default yaitu $user-\>name\ (data dari database).

Password Opsional: Label dan petunjuk diubah untuk memberitahu admin bahwa password tidak wajib diisi jika tidak ingin diubah.

### Langkah 12: Logika Update Data (Metode update)
Sekarang kita isi logika untuk memproses data dari form edit.

Buka kembali app/Http/Controllers/UserController.php dan isi method update():

```PHP
// app/Http/Controllers/UserController.php

// ... (method edit) ...

use Illuminate\Validation\Rule; // <-- Tambahkan ini di atas

/**
 * Update the specified resource in storage.
 */
public function update(Request $request, User $user)
{
    // 1. Validasi Input
    $request->validate([
        'name' => ['required', 'string', 'max:255'],
        'email' => ['required', 'string', 'email', 'max:255', Rule::unique('users')->ignore($user->id)],
        'role' => ['required', 'in:admin,karyawan'],
        'password' => ['nullable', 'confirmed', Rules\Password::defaults()],
    ]);

    // 2. Siapkan data untuk di-update
    $updateData = [
        'name' => $request->name,
        'email' => $request->email,
        'role' => $request->role,
    ];

    // 3. Cek apakah password diisi
    if ($request->filled('password')) {
        $updateData['password'] = Hash::make($request->password);
    }

    // 4. Update data user
    $user->update($updateData);

    // 5. Redirect ke halaman index dengan pesan sukses
    return redirect()->route('users.index')
                     ->with('success', 'Data pengguna berhasil diperbarui.');
}

// ... (method destroy) ...
```
Penjelasan Logika update():

Rule::unique('users')->ignore($user->id): Ini adalah bagian validasi yang krusial. Aturan ini mengatakan, "Pastikan email ini unik di tabel users, KECUALI untuk baris data dengan ID milik pengguna ini sendiri". Ini mencegah error "email has already been taken" saat kita tidak mengubah email.

'password' => ['nullable', ...]: Aturan nullable berarti field password boleh kosong. Jika kosong, validasi untuk password akan dilewati.

if ($request->filled('password')): Kita hanya akan meng-update password jika field password di form diisi. Jika kosong, password lama tidak akan ditimpa.

$user->update(...): Memperbarui data pengguna di database dengan data yang sudah disiapkan.

Uji Coba
Waktunya mengetes!

Buka halaman /users.

Di salah satu baris data, klik ikon pensil (Edit). Anda akan diarahkan ke form edit yang sudah terisi data.

Ubah nama atau role, lalu klik "Update". Anda akan kembali ke halaman daftar dan melihat pesan sukses serta data yang sudah berubah.

Coba edit lagi, tapi kali ini biarkan password kosong. Klik "Update". Seharusnya data tetap ter-update tanpa mengubah password.

Coba edit lagi dan ubah passwordnya. Setelah update, coba logout dan login kembali dengan password baru untuk memastikan perubahannya berhasil.

---

# Tahap 6: Fungsionalitas Detail dan Hapus Pengguna
## Langkah 13: Menampilkan Detail Pengguna (Metode show dan View)
Fungsi ini berguna jika kita ingin melihat semua informasi terkait seorang pengguna dalam satu halaman, terutama detail data employee yang tidak semua ditampilkan di tabel utama.

### 13.1. Update UserController (Metode show)

Buka app/Http/Controllers/UserController.php. Sama seperti metode edit, show akan menggunakan Route Model Binding untuk mengambil data pengguna secara otomatis.

```PHP
// app/Http/Controllers/UserController.php

// ... (method index, create, store) ...

/**
 * Display the specified resource.
 */
public function show(User $user)
{
    // Kita panggil relasi 'employee' secara eksplisit
    // untuk memastikan datanya termuat.
    $user->load('employee');

    return view('users.show', compact('user'));
}

// ... (method edit, update, destroy) ...
```
Catatan: $user->load('employee') digunakan untuk memastikan relasi employee dimuat, bahkan jika belum dimuat sebelumnya. Ini adalah praktik yang baik untuk memastikan data selalu tersedia di view.

## 13.2. Buat View users.show.blade.php

Buat file baru di resources/views/users/ bernama show.blade.php. Isi dengan kode berikut untuk menampilkan data secara terstruktur.

```HTML
<x-app-layout>
    <x-slot name="header">
        <h2 class="font-semibold text-xl text-gray-800 leading-tight">
            {{ __('Detail Pengguna') }}
        </h2>
    </x-slot>

    <div class="py-12">
        <div class="max-w-7xl mx-auto sm:px-6 lg:px-8">
            <div class="bg-white overflow-hidden shadow-sm sm:rounded-lg">
                <div class="p-6 text-gray-900">
                    <div class="mb-6">
                        <a href="{{ route('users.index') }}" class="bg-gray-500 hover:bg-gray-700 text-white font-bold py-2 px-4 rounded">
                            <i class="fas fa-arrow-left"></i> Kembali
                        </a>
                    </div>
                    
                    <div class="border-t border-gray-200">
                        <dl>
                            <div class="bg-gray-50 px-4 py-5 sm:grid sm:grid-cols-3 sm:gap-4 sm:px-6">
                                <dt class="text-sm font-medium text-gray-500">Nama Lengkap</dt>
                                <dd class="mt-1 text-sm text-gray-900 sm:mt-0 sm:col-span-2">{{ $user->name }}</dd>
                            </div>
                            <div class="bg-white px-4 py-5 sm:grid sm:grid-cols-3 sm:gap-4 sm:px-6">
                                <dt class="text-sm font-medium text-gray-500">Alamat Email</dt>
                                <dd class="mt-1 text-sm text-gray-900 sm:mt-0 sm:col-span-2">{{ $user->email }}</dd>
                            </div>
                            <div class="bg-gray-50 px-4 py-5 sm:grid sm:grid-cols-3 sm:gap-4 sm:px-6">
                                <dt class="text-sm font-medium text-gray-500">Role</dt>
                                <dd class="mt-1 text-sm text-gray-900 sm:mt-0 sm:col-span-2">{{ ucfirst($user->role) }}</dd>
                            </div>
                            
                            @if ($user->employee)
                                <div class="bg-white px-4 py-5 sm:grid sm:grid-cols-3 sm:gap-4 sm:px-6">
                                    <dt class="text-sm font-medium text-gray-500">NIP</dt>
                                    <dd class="mt-1 text-sm text-gray-900 sm:mt-0 sm:col-span-2">{{ $user->employee->nip }}</dd>
                                </div>
                                <div class="bg-gray-50 px-4 py-5 sm:grid sm:grid-cols-3 sm:gap-4 sm:px-6">
                                    <dt class="text-sm font-medium text-gray-500">Posisi / Jabatan</dt>
                                    <dd class="mt-1 text-sm text-gray-900 sm:mt-0 sm:col-span-2">{{ $user->employee->posisi }} / {{ $user->employee->jabatan }}</dd>
                                </div>
                                <div class="bg-white px-4 py-5 sm:grid sm:grid-cols-3 sm:gap-4 sm:px-6">
                                    <dt class="text-sm font-medium text-gray-500">Tanggal Perekrutan</dt>
                                    <dd class="mt-1 text-sm text-gray-900 sm:mt-0 sm:col-span-2">{{ \Carbon\Carbon::parse($user->employee->tanggal_perekrutan)->isoFormat('D MMMM Y') }}</dd>
                                </div>
                                <div class="bg-gray-50 px-4 py-5 sm:grid sm:grid-cols-3 sm:gap-4 sm:px-6">
                                    <dt class="text-sm font-medium text-gray-500">No. HP & Alamat</dt>
                                    <dd class="mt-1 text-sm text-gray-900 sm:mt-0 sm:col-span-2">{{ $user->employee->no_hp }} <br> {{ $user->employee->alamat }}</dd>
                                </div>
                                <div class="bg-white px-4 py-5 sm:grid sm:grid-cols-3 sm:gap-4 sm:px-6">
                                    <dt class="text-sm font-medium text-gray-500">Jenis Kelamin</dt>
                                    <dd class="mt-1 text-sm text-gray-900 sm:mt-0 sm:col-span-2">{{ $user->employee->jenis_kelamin }}</dd>
                                </div>
                                <div class="bg-gray-50 px-4 py-5 sm:grid sm:grid-cols-3 sm:gap-4 sm:px-6">
                                    <dt class="text-sm font-medium text-gray-500">Status Karyawan</dt>
                                    <dd class="mt-1 text-sm text-gray-900 sm:mt-0 sm:col-span-2">{{ ucfirst($user->employee->status) }}</dd>
                                </div>
                            @else
                                <div class="bg-white px-4 py-5 sm:grid sm:grid-cols-3 sm:gap-4 sm:px-6">
                                    <dt class="text-sm font-medium text-gray-500">Data Karyawan</dt>
                                    <dd class="mt-1 text-sm text-gray-900 sm:mt-0 sm:col-span-2">Tidak ditemukan (Pengguna ini adalah Admin).</dd>
                                </div>
                            @endif
                        </dl>
                    </div>

                </div>
            </div>
        </div>
    </div>
</x-app-layout>
```

Catatan: Kita menggunakan Carbon (\Carbon\Carbon::...) untuk memformat tanggal perekrutan menjadi format yang lebih mudah dibaca (misal: "4 Oktober 2025").

## Langkah 14: Logika Hapus Data (Metode destroy)
Ini adalah langkah terakhir untuk fungsionalitas CRUD. Metode destroy akan menghapus pengguna dari database.

### 14.1. Pastikan Form Hapus Sudah Benar

Di file resources/views/users/index.blade.php, kita sudah membuat form untuk tombol hapus. Mari kita pastikan lagi kodenya.

```HTML
<form action="{{ route('users.destroy', $user->id) }}" method="POST" class="inline-block" onsubmit="return confirm('Apakah Anda yakin ingin menghapus pengguna ini?');">
    @csrf
    @method('DELETE') <button type="submit" class="text-red-600 hover:text-red-900 ml-4" title="Hapus"><i class="fas fa-trash"></i></button>
</form>
```
onsubmit="return confirm(...): Ini adalah Javascript sederhana untuk memunculkan kotak dialog konfirmasi sebelum form benar-benar dikirim. Ini adalah praktik keamanan yang baik untuk mencegah penghapusan data yang tidak disengaja.

### 14.2. Update UserController (Metode destroy)

Buka app/Http/Controllers/UserController.php dan isi method destroy():

```PHP
// app/Http/Controllers/UserController.php

// ... (method update) ...

/**
 * Remove the specified resource from storage.
 */
public function destroy(User $user)
{
    // Hapus pengguna dari database
    $user->delete();

    // Redirect kembali ke halaman index dengan pesan sukses
    return redirect()->route('users.index')
                     ->with('success', 'Pengguna berhasil dihapus.');
}
```
Logikanya sangat sederhana, $user->delete() akan menghapus baris data pengguna dari tabel users. Karena kita sudah mengatur onDelete('cascade') pada migrasi employees, maka data karyawan yang terkait juga akan otomatis ikut terhapus.

Uji Coba Akhir
Buka halaman /users.

Klik ikon mata (<i class="fas fa-eye"></i>) pada salah satu data karyawan. Anda akan diarahkan ke halaman detail yang menampilkan semua informasinya.

Kembali ke halaman /users.

Klik ikon tempat sampah (<i class="fas fa-trash"></i>) pada salah satu data.

Sebuah kotak konfirmasi akan muncul. Klik "OK".

Halaman akan me-refresh, pengguna tersebut akan hilang dari tabel, dan pesan sukses penghapusan akan muncul.

SELAMAT! 🎉 Anda telah berhasil membangun fungsionalitas CRUD (Create, Read, Update, Delete) penuh untuk manajemen pengguna.

Fondasi aplikasi absensi Anda kini sudah sangat kokoh. Kita sudah punya:

Sistem otentikasi.

Manajemen role admin dan karyawan.

Fitur lengkap untuk mengelola data pengguna.

---

# 🛠️ Revisi File navigation.blade.php
telah menerapkan logika pengecekan peran (role) menggunakan @if (Auth::user()->role === 'admin') dan menambahkan tautan placeholder untuk fitur absensi.

edit File: resources/views/layouts/navigation.blade.php

```HTML
<nav x-data="{ open: false }" class="bg-white border-b border-gray-100">
    <div class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8">
        <div class="flex justify-between h-16">
            <div class="flex">
                <div class="shrink-0 flex items-center">
                    <a href="{{ route('dashboard') }}">
                        <x-application-logo class="block h-9 w-auto fill-current text-gray-800" />
                    </a>
                </div>

                <div class="hidden space-x-8 sm:-my-px sm:ml-10 sm:flex">
                    <x-nav-link :href="route('dashboard')" :active="request()->routeIs('dashboard')">
                        {{ __('Dashboard') }}
                    </x-nav-link>

                    {{-- Link KHUSUS ADMIN --}}
                    @if (Auth::user()->role === 'admin')
                        <x-nav-link :href="route('users.index')" :active="request()->routeIs('users.index')">
                            {{ __('Manajemen Pengguna') }}
                        </x-nav-link>

                        {{-- PLACEHOLDER Admin: Absensi, Karyawan, Laporan --}}
                        <x-nav-link :href="'#'">
                            {{ __('Absensi Harian') }}
                        </x-nav-link>
                        <x-nav-link :href="'#'">
                            {{ __('Laporan') }}
                        </x-nav-link>
                    @endif

                    {{-- Link KHUSUS KARYAWAN --}}
                    @if (Auth::user()->role === 'karyawan')
                        <x-nav-link :href="'#'">
                            {{ __('Absensi Saya') }}
                        </x-nav-link>
                        <x-nav-link :href="'#'">
                            {{ __('Riwayat Absensi') }}
                        </x-nav-link>
                    @endif
                </div>
            </div>

            <div class="hidden sm:flex sm:items-center sm:ml-6">
                <x-dropdown align="right" width="48">
                    <x-slot name="trigger">
                        <button class="inline-flex items-center px-3 py-2 border border-transparent text-sm leading-4 font-medium rounded-md text-gray-500 bg-white hover:text-gray-700 focus:outline-none transition ease-in-out duration-150">
                            <div>{{ Auth::user()->name }}</div>
                            <div class="ml-1">
                                <svg class="fill-current h-4 w-4" xmlns="http://www.w3.org/2000/svg" viewBox="0 0 20 20">
                                    <path fill-rule="evenodd" d="M5.293 7.293a1 1 0 011.414 0L10 10.586l3.293-3.293a1 1 0 111.414 1.414l-4 4a1 1 0 01-1.414 0l-4-4a1 1 0 010-1.414z" clip-rule="evenodd" />
                                </svg>
                            </div>
                        </button>
                    </x-slot>

                    <x-slot name="content">
                        <x-dropdown-link :href="route('profile.edit')">
                            {{ __('Profile') }}
                        </x-dropdown-link>

                        <form method="POST" action="{{ route('logout') }}">
                            @csrf
                            <x-dropdown-link :href="route('logout')"
                                    onclick="event.preventDefault();
                                                this.closest('form').submit();">
                                {{ __('Log Out') }}
                            </x-dropdown-link>
                        </form>
                    </x-slot>
                </x-dropdown>
            </div>

            <div class="-mr-2 flex items-center sm:hidden">
                <button @click="open = ! open" class="inline-flex items-center justify-center p-2 rounded-md text-gray-400 hover:text-gray-500 hover:bg-gray-100 focus:outline-none focus:bg-gray-100 focus:text-gray-500 transition duration-150 ease-in-out">
                    <svg class="h-6 w-6" stroke="currentColor" fill="none" viewBox="0 0 24 24">
                        <path :class="{'hidden': open, 'inline-flex': ! open }" class="inline-flex" stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M4 6h16M4 12h16M4 18h16" />
                        <path :class="{'hidden': ! open, 'inline-flex': open }" class="hidden" stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M6 18L18 6M6 6l12 12" />
                    </svg>
                </button>
            </div>
        </div>
    </div>

    <div :class="{'block': open, 'hidden': ! open}" class="hidden sm:hidden">
        <div class="pt-2 pb-3 space-y-1">
            <x-responsive-nav-link :href="route('dashboard')" :active="request()->routeIs('dashboard')">
                {{ __('Dashboard') }}
            </x-responsive-nav-link>
            
            {{-- Link KHUSUS ADMIN RESPONSIVE --}}
            @if (Auth::user()->role === 'admin')
                <x-responsive-nav-link :href="route('users.index')" :active="request()->routeIs('users.index')">
                    {{ __('Manajemen Pengguna') }}
                </x-responsive-nav-link>
                <x-responsive-nav-link :href="'#'">
                    {{ __('Absensi Harian') }}
                </x-responsive-nav-link>
            @endif

            {{-- Link KHUSUS KARYAWAN RESPONSIVE --}}
            @if (Auth::user()->role === 'karyawan')
                <x-responsive-nav-link :href="'#'">
                    {{ __('Absensi Saya') }}
                </x-responsive-nav-link>
                <x-responsive-nav-link :href="'#'">
                    {{ __('Riwayat Absensi') }}
                </x-responsive-nav-link>
            @endif

        </div>

        <div class="pt-4 pb-1 border-t border-gray-200">
            <div class="px-4">
                <div class="font-medium text-base text-gray-800">{{ Auth::user()->name }}</div>
                <div class="font-medium text-sm text-gray-500">{{ Auth::user()->email }}</div>
            </div>

            <div class="mt-3 space-y-1">
                <x-responsive-nav-link :href="route('profile.edit')">
                    {{ __('Profile') }}
                </x-responsive-nav-link>

                <form method="POST" action="{{ route('logout') }}">
                    @csrf

                    <x-responsive-nav-link :href="route('logout')"
                            onclick="event.preventDefault();
                                            this.closest('form').submit();">
                        {{ __('Log Out') }}
                    </x-responsive-nav-link>
                </form>
            </div>
        </div>
    </div>
</nav>
```

>[!note]
> Status Saat Ini:
> Manajemen pengguna sudah selesai. Anda sudah bisa login sebagai admin@gmail.com dengan password "password", dan Anda bisa menambah, mengedit, melihat, dan menghapus pengguna lain.

### Next lanjut Part 2
Untuk informasi lebih lanjut, kunjungi [Panduannya](https://github.com/ahmad-syaifuddin/part2-panduan-absensi-project.git).
