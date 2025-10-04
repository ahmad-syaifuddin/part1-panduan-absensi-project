# 📚 Panduan Lengkap: Membuat Aplikasi Absensi dengan Laravel 10 & Breeze

> **Catatan**: Panduan ini menggunakan Laravel 10 dengan Laravel Breeze untuk sistem autentikasi.

---

## 💻 Langkah 1: Instalasi & Konfigurasi Awal

Kita akan memulai dengan instalasi proyek, Breeze, dan penyesuaian yang diperlukan.

### 1.1. Instalasi Laravel dan Breeze

Jalankan perintah berikut di Laragon Anda untuk membuat proyek Laravel 10 dan menginstal Laravel Breeze versi 1.19.

**Buat proyek Laravel baru:**
```bash
composer create-project laravel/laravel absensi "10.*"
```

**Masuk ke direktori proyek:**
```bash
cd absensi
```

**Instal Laravel Breeze versi 1.19:**
```bash
composer require laravel/breeze:^1.19 --dev
```

**Jalankan Breeze dengan stack Blade:**
```bash
php artisan breeze:install blade
```

### 1.2. Penyesuaian postcss.config.js

Beberapa versi Node.js atau paket bundler mungkin memerlukan format CommonJS. Mengubah ekstensi file ini dapat mengatasi potensi kesalahan.

**Ubah nama file postcss.config.js menjadi postcss.config.cjs:**
```bash
mv postcss.config.js postcss.config.cjs
```

### 1.3. Menjalankan Aplikasi

Sekarang kita lihat hasil tampilannya!

**Terminal Pertama - Jalankan server PHP:**
```bash
php artisan serve
```

**Terminal Kedua - Build frontend assets:**
```bash
npm run dev
```

---

## ⚙️ Langkah 2: Migrasi, Role, dan Seeder

Setelah proyek siap, kita akan membuat struktur data dasar untuk membedakan pengguna Admin dan Karyawan.

### 2.1. Menambahkan Kolom Role ke Tabel Users

Kita akan membuat migrasi baru untuk menambahkan kolom `role` pada tabel `users` yang sudah dibuat oleh Breeze.

**Buat terminal ketiga untuk menangani perintah artisan:**
```bash
php artisan make:migration add_role_to_users_table
```

**Kode Migrasi:**

Buka file migrasi yang baru saja dibuat di `database/migrations/` dan tambahkan kode berikut:

```php
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

### 2.2. Konfigurasi Database

**Pastikan isi dari file `.env` sudah benar sebelum migrate:**

Ubah variabel untuk `APP_NAME` dan `DB_DATABASE`:

```env
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

**Jalankan Migrasi:**
```bash
php artisan migrate
```

### 2.3. Membuat Model dan Migrasi Employee

Tabel ini akan menyimpan data detail karyawan yang terhubung dengan tabel `users`.

**Buat Model dan Migrasi Employee:**
```bash
php artisan make:model Employee -m
```

**Buka file migrasi employees dan isi dengan kode berikut:**

```php
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

### 2.4. Membuat Model dan Migrasi Attendance & Holiday

Kita buat sekarang agar struktur database lengkap.

**Buat Model Attendance:**
```bash
php artisan make:model Attendance -m
```

**Buat Model Holiday:**
```bash
php artisan make:model Holiday -m
```

**Isi file migrasi `attendances`:**

```php
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

**Isi file migrasi `holidays`:**

```php
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

**Jalankan Migrasi:**
```bash
php artisan migrate
```

---

## 🔗 Langkah 3: Relasi Model, Factory, dan Seeder

Kita akan membuat data dummy agar aplikasi kita tidak kosong.

### 3.1. Definisikan Relasi Model User

Buka `app/Models/User.php` dan tambahkan relasi `hasOne` ke Employee. Juga tambahkan `role` ke `$fillable`.

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;
use Laravel\Sanctum\HasApiTokens;
use App\Models\Employee;

class User extends Authenticatable
{
    use HasApiTokens, HasFactory, Notifiable;

    protected $fillable = [
        'name',
        'email',
        'password',
        'role', // Tambahkan ini
    ];

    protected $hidden = [
        'password',
        'remember_token',
    ];

    protected $casts = [
        'email_verified_at' => 'datetime',
        'password' => 'hashed',
    ];

    /**
     * Get the employee associated with the user.
     */
    public function employee()
    {
        return $this->hasOne(Employee::class);
    }
}
```

### 3.2. Definisikan Relasi Model Employee

Buka `app/Models/Employee.php` dan tambahkan relasi `belongsTo` ke User.

```php
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

### 3.3. Modifikasi UserFactory

Buka `database/factories/UserFactory.php` dan tambahkan `role`.

```php
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

### 3.4. Buat EmployeeFactory

```bash
php artisan make:factory EmployeeFactory --model=Employee
```

Buka `database/factories/EmployeeFactory.php` dan isi dengan kode berikut:

```php
<?php

namespace Database\Factories;

use Faker\Factory as FakerFactory;
use Illuminate\Database\Eloquent\Factories\Factory;

class EmployeeFactory extends Factory
{
    public function definition(): array
    {
        // Inisiasi Faker dengan lokal Indonesia
        $faker = FakerFactory::create('id_ID');

        return [
            'nama_lengkap' => $faker->name(),
            'nip' => $faker->unique()->numerify('##################'), // NIP 18 digit
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

### 3.5. Buat UserSeeder

```bash
php artisan make:seeder UserSeeder
```

Buka `database/seeders/UserSeeder.php` dan isi dengan kode berikut:

```php
<?php

namespace Database\Seeders;

use App\Models\Employee;
use Illuminate\Database\Seeder;
use App\Models\User;
use Illuminate\Support\Facades\Hash;

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
            Employee::factory()->create(['user_id' => $user->id]);
        });
    }
}
```

### 3.6. Konfigurasi DatabaseSeeder

Buka `database/seeders/DatabaseSeeder.php` dan ubah isinya:

```php
<?php

namespace Database\Seeders;

use App\Models\Employee;
use App\Models\User;
use Illuminate\Database\Seeder;
use Illuminate\Support\Facades\Hash;

class DatabaseSeeder extends Seeder
{
    public function run(): void
    {
        // Membuat 1 User Admin (tanpa data employee)
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

        // Membuat 10 Karyawan dummy
        User::factory(10)->has(Employee::factory())->create();
    }
}
```

### 3.7. Jalankan Seeder

Perintah ini akan menghapus semua data lama dan mengisi database dengan data dari seeder:

```bash
php artisan migrate:fresh --seed
```

**🎉 Database Anda sudah terisi!**

- **Admin Login**: `admin@gmail.com` / `password`
- **Karyawan Login**: `karyawan@gmail.com` / `password`

---

✅ **Checkpoint**: Fondasi aplikasi sudah sangat kuat. Kita sudah punya:
- Proyek Laravel dengan otentikasi
- Struktur database yang lengkap
- Data dummy untuk pengembangan

---

## 🛡️ Langkah 4: Membuat Middleware untuk Role

Middleware ini berfungsi seperti satpam. Jika pengguna yang mencoba mengakses halaman manajemen pengguna bukan 'admin', sistem akan otomatis menolaknya.

### 4.1. Buat File Middleware

Jalankan perintah ini di terminal Anda:

```bash
php artisan make:middleware RoleMiddleware
```

Perintah ini akan membuat file baru di `app/Http/Middleware/RoleMiddleware.php`.

### 4.2. Isi Logika Middleware

Buka file `RoleMiddleware.php` yang baru saja dibuat dan ubah isinya menjadi:

```php
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
        // Cek dulu apakah pengguna sudah login atau belum
        if (!Auth::check()) {
            return redirect('login');
        }

        // Ambil role dari pengguna yang sedang login
        $userRole = Auth::user()->role;

        // Cek apakah role pengguna ada di dalam daftar role yang diizinkan
        if (in_array($userRole, $roles)) {
            // Jika cocok, izinkan pengguna melanjutkan request
            return $next($request);
        }

        // Jika tidak cocok, tolak akses (Forbidden)
        abort(403, 'ANDA TIDAK MEMILIKI AKSES KE HALAMAN INI.');
    }
}
```

### 4.3. Daftarkan Middleware

Agar bisa dipanggil dengan nama `role`, daftarkan middleware ini di `app/Http/Kernel.php`. Tambahkan satu baris di dalam array `$middlewareAliases`.

```php
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

---

## 🎯 Langkah 5: Controller dan Rute untuk Manajemen Pengguna

Di sini kita akan membuat "otak" (Controller) yang mengatur logika CRUD dan "alamat" (Rute) untuk mengakses halaman-halaman tersebut.

### 5.1. Buat Controller

Gunakan flag `--resource` agar Laravel otomatis membuatkan method-method standar CRUD (index, create, store, show, edit, update, destroy).

```bash
php artisan make:controller UserController --resource
```

Ini akan membuat file `app/Http/Controllers/UserController.php`.

### 5.2. Definisikan Rute

Buka file `routes/web.php` dan tambahkan rute untuk manajemen pengguna. Kita akan meletakkannya di dalam grup middleware `auth` dan `role:admin` agar aman.

```php
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

Buka `app/Http/Controllers/UserController.php` dan modifikasi seperti ini:

```php
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
        // Ambil semua data user, diurutkan dari yang terbaru
        // Gunakan 'with('employee')' untuk mengambil relasi employee (Eager Loading)
        // Gunakan paginate() untuk membatasi data per halaman
        $users = User::with('employee')->latest()->paginate(10);

        // Kirim data users ke view 'users.index'
        return view('users.index', compact('users'));
    }

    // method-method lain akan kita isi pada tahap berikutnya...
}
```

**Penjelasan Singkat:**

- `User::with('employee')`: Teknik Eager Loading. Kita mengambil data User sekaligus data Employee yang terhubung. Sangat efisien!
- `latest()->paginate(10)`: Mengurutkan data dari yang paling baru dan menampilkannya 10 data per halaman
- `return view('users.index', ...)`: Mengirim data `$users` ke file view yang akan kita buat selanjutnya

---

## 🎨 Langkah 6: Menambahkan Font Awesome

Sesuai rencana, kita akan menggunakan ikon dari Font Awesome. Cara termudah adalah menambahkannya melalui CDN ke layout utama aplikasi.

### 6.1. Update Layout Utama

Buka file layout utama di `resources/views/layouts/app.blade.php`.

Tambahkan baris kode CDN berikut di dalam tag `<head>`, di bawah baris `<link>` yang sudah ada:

```html
<link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.5.2/css/all.min.css" integrity="sha512-SnH5WK+bZxgPHs44uWIX+LLJAJ9/2PkPKZ5QiAj6Ta86w+fsb2TkcmfRyVX3pBnMFcV7oQPJkl9QevSCWr3W6A==" crossorigin="anonymous" referrerpolicy="no-referrer" />
```

**Contoh penempatannya:**

```html
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <meta name="csrf-token" content="{{ csrf_token() }}">

    <title>{{ config('app.name', 'Laravel') }}</title>

    <link rel="preconnect" href="https://fonts.bunny.net">
    <link href="https://fonts.bunny.net/css?family=figtree:400,500,600&display=swap" rel="stylesheet" />

    @vite(['resources/css/app.css', 'resources/js/app.js'])

    <!-- Font Awesome CDN -->
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.5.2/css/all.min.css" integrity="sha512-SnH5WK+bZxgPHs44uWIX+LLJAJ9/2PkPKZ5QiAj6Ta86w+fsb2TkcmfRyVX3pBnMFcV7oQPJkl9QevSCWr3W6A==" crossorigin="anonymous" referrerpolicy="no-referrer" />
</head>
```

---

## 📋 Langkah 7: Membuat View Index (Daftar Pengguna)

Sekarang kita buat halaman untuk menampilkan daftar pengguna.

### 7.1. Buat Folder dan File

Buat folder baru `users` di `resources/views/` dan buat file `index.blade.php` di dalamnya.

**Path lengkap:** `resources/views/users/index.blade.php`

### 7.2. Isi View Index

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

                    {{-- Flash Message Sukses --}}
                    @if (session('success'))
                        <div class="mb-4 bg-green-100 border border-green-400 text-green-700 px-4 py-3 rounded relative" role="alert">
                            <strong class="font-bold">Sukses!</strong>
                            <span class="block sm:inline">{{ session('success') }}</span>
                        </div>
                    @endif

                    {{-- Tombol Tambah Pengguna --}}
                    <div class="mb-4">
                        <a href="{{ route('users.create') }}" class="bg-blue-500 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded">
                            <i class="fas fa-plus"></i> Tambah Pengguna
                        </a>
                    </div>

                    {{-- Tabel Data Pengguna --}}
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

                    {{-- Pagination Links --}}
                    <div class="mt-4">
                        {{ $users->links() }}
                    </div>

                </div>
            </div>
        </div>
    </div>
</x-app-layout>
```

---
