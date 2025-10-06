<div align="center">

<p> <b>Visitors Count 👁️</b> </p>
<img src="https://profile-counter.deno.dev/part1-panduan-absensi-project/count.svg" alt="Profile Counter Repo :: Visitor's Count" />

</div>

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

## ➕ Langkah 8: Fungsionalitas Tambah Pengguna (Create & Store)

Sekarang kita akan membuat fungsionalitas di balik tombol "Tambah Pengguna". Ini melibatkan dua bagian: halaman formulir (create) dan logika penyimpanan (store).

### 8.1. Update UserController (Metode create)

Buka `app/Http/Controllers/UserController.php` dan tambahkan method `create()`:

```php
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

### 8.2. Buat View users.create.blade.php

Buat file baru `resources/views/users/create.blade.php` dengan kode berikut:

```blade
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
                    
                    {{-- Error Messages --}}
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

                    {{-- Form Tambah Pengguna --}}
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

---

## 💾 Langkah 9: Logika Penyimpanan Data (Metode store)

Sekarang kita isi logika untuk memproses data dari form.

### 9.1. Update UserController (Metode store)

Buka `app/Http/Controllers/UserController.php` dan tambahkan method `store()`:

```php
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

**Penjelasan Logika store():**

- `$request->validate([...])`: Validasi input. Jika ada aturan yang tidak terpenuhi, Laravel otomatis mengembalikan ke form dengan pesan error
- `User::create([...])`: Jika validasi berhasil, data disimpan ke database
- `Hash::make(...)`: Enkripsi password sebelum menyimpan (SANGAT PENTING!)
- `redirect()->...->with(...)`: Redirect ke halaman index dengan flash message

---

## 🔧 Langkah 10: Uji Coba Tambah Pengguna

### 10.1. Testing

1. Buka halaman `/users`
2. Klik tombol "Tambah Pengguna"
3. Coba submit form dengan data kosong atau password yang tidak cocok → Anda akan melihat pesan error
4. Isi formulir dengan benar, lalu klik "Simpan"
5. Jika berhasil, Anda akan kembali ke halaman daftar dengan pesan sukses hijau

---

## ✏️ Langkah 11: Fungsionalitas Edit Pengguna (Edit & Update)

Prosesnya mirip dengan "Tambah Data", tapi formulirnya sudah terisi dengan data lama.

### 11.1. Update UserController (Metode edit)

Buka `app/Http/Controllers/UserController.php` dan tambahkan method `edit()`:

```php
// app/Http/Controllers/UserController.php

// ... (method store) ...

/**
 * Show the form for editing the specified resource.
 */
public function edit(User $user)
{
    // Variabel $user sudah otomatis berisi data user yang akan diedit
    // berkat Route Model Binding dari Laravel
    return view('users.edit', compact('user'));
}

// ... (method update dan lainnya) ...
```

### 11.2. Buat View users.edit.blade.php

Buat file baru `resources/views/users/edit.blade.php`:

```blade
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
                    
                    {{-- Error Messages --}}
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

                    {{-- Form Edit Pengguna --}}
                    <form method="POST" action="{{ route('users.update', $user->id) }}">
                        @csrf
                        @method('PUT')

                        <div>
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

**Perbedaan Utama dari Form create:**

- `action="{{ route('users.update', $user->id) }}"`: Form dikirim ke rute update dengan ID pengguna
- `@method('PUT')`: Method spoofing untuk memberitahu Laravel ini adalah request PUT (update)
- `:value="old('name', $user->name)"`: Mengambil data lama jika ada error validasi, jika tidak ambil dari database
- Password Opsional: Label diubah untuk menunjukkan password tidak wajib diisi

---

## 🔄 Langkah 12: Logika Update Data (Metode update)

### 12.1. Update UserController (Metode update)

Buka `app/Http/Controllers/UserController.php` dan tambahkan method `update()`:

```php
// app/Http/Controllers/UserController.php

use Illuminate\Validation\Rule; // <-- Tambahkan ini di atas

// ... (method edit) ...

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

**Penjelasan Logika update():**

- `Rule::unique('users')->ignore($user->id)`: Pastikan email unik, KECUALI untuk pengguna ini sendiri
- `'password' => ['nullable', ...]`: Field password boleh kosong
- `if ($request->filled('password'))`: Hanya update password jika diisi
- `$user->update(...)`: Memperbarui data di database

---

## 👁️ Langkah 13: Menampilkan Detail Pengguna (Show)

Fungsi ini berguna untuk melihat semua informasi terkait seorang pengguna dalam satu halaman, terutama detail data employee.

### 13.1. Update UserController (Metode show)

Buka `app/Http/Controllers/UserController.php` dan tambahkan method `show()`:

```php
// app/Http/Controllers/UserController.php

// ... (method index, create, store) ...

/**
 * Display the specified resource.
 */
public function show(User $user)
{
    // Kita panggil relasi 'employee' secara eksplisit
    // untuk memastikan datanya termuat
    $user->load('employee');

    return view('users.show', compact('user'));
}

// ... (method edit, update, destroy) ...
```

**Catatan:** `$user->load('employee')` memastikan relasi employee dimuat, bahkan jika belum dimuat sebelumnya.

### 13.2. Buat View users.show.blade.php

Buat file baru `resources/views/users/show.blade.php`:

```blade
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
                    
                    {{-- Tombol Kembali --}}
                    <div class="mb-6">
                        <a href="{{ route('users.index') }}" class="bg-gray-500 hover:bg-gray-700 text-white font-bold py-2 px-4 rounded">
                            <i class="fas fa-arrow-left"></i> Kembali
                        </a>
                    </div>
                    
                    {{-- Detail Data Pengguna --}}
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

**Catatan:** Kita menggunakan Carbon untuk memformat tanggal menjadi format yang lebih mudah dibaca (misal: "4 Oktober 2025").

---

## 🗑️ Langkah 14: Logika Hapus Data (Destroy)

Ini adalah langkah terakhir untuk fungsionalitas CRUD.

### 14.1. Pastikan Form Hapus Sudah Benar

Di file `resources/views/users/index.blade.php`, kita sudah membuat form untuk tombol hapus:

```html
<form action="{{ route('users.destroy', $user->id) }}" method="POST" class="inline-block" onsubmit="return confirm('Apakah Anda yakin ingin menghapus pengguna ini?');">
    @csrf
    @method('DELETE')
    <button type="submit" class="text-red-600 hover:text-red-900 ml-4" title="Hapus">
        <i class="fas fa-trash"></i>
    </button>
</form>
```

**Catatan:** `onsubmit="return confirm(...)"` adalah Javascript sederhana untuk memunculkan kotak dialog konfirmasi sebelum penghapusan.

### 14.2. Update UserController (Metode destroy)

Buka `app/Http/Controllers/UserController.php` dan tambahkan method `destroy()`:

```php
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

**Logika sangat sederhana:** `$user->delete()` akan menghapus data. Karena kita sudah mengatur `onDelete('cascade')` pada migrasi employees, maka data karyawan terkait juga akan otomatis ikut terhapus.

---

## 🧭 Langkah 15: Update Navigation Menu

Sekarang kita update menu navigasi agar menampilkan link sesuai role pengguna.

### 15.1. Edit File navigation.blade.php

Buka file `resources/views/layouts/navigation.blade.php` dan ubah isinya:

```blade
<nav x-data="{ open: false }" class="bg-white border-b border-gray-100">
    <div class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8">
        <div class="flex justify-between h-16">
            <div class="flex">
                {{-- Logo --}}
                <div class="shrink-0 flex items-center">
                    <a href="{{ route('dashboard') }}">
                        <x-application-logo class="block h-9 w-auto fill-current text-gray-800" />
                    </a>
                </div>

                {{-- Navigation Links --}}
                <div class="hidden space-x-8 sm:-my-px sm:ml-10 sm:flex">
                    <x-nav-link :href="route('dashboard')" :active="request()->routeIs('dashboard')">
                        {{ __('Dashboard') }}
                    </x-nav-link>

                    {{-- Link KHUSUS ADMIN --}}
                    @if (Auth::user()->role === 'admin')
                        <x-nav-link :href="route('users.index')" :active="request()->routeIs('users.index')">
                            {{ __('Manajemen Pengguna') }}
                        </x-nav-link>

                        {{-- PLACEHOLDER Admin: Absensi, Laporan --}}
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

            {{-- Settings Dropdown --}}
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

            {{-- Hamburger Menu --}}
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

    {{-- Responsive Navigation Menu --}}
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

        {{-- Responsive Settings Options --}}
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

---

## 🎉 Uji Coba Akhir & Kesimpulan

### Testing Lengkap

1. **Buka halaman `/users`**
2. **Klik ikon mata** pada salah satu data karyawan - Anda akan melihat halaman detail lengkap
3. **Kembali ke `/users`**
4. **Klik ikon pensil** untuk edit data - Ubah nama atau role, lalu klik "Update"
5. **Klik ikon tempat sampah** untuk hapus - Konfirmasi, dan data akan terhapus

### Kredensial Login

- **Admin**: `admin@gmail.com` / `password`
- **Karyawan**: `karyawan@gmail.com` / `password`

---

## ✅ Apa yang Sudah Kita Capai?

Anda telah berhasil membangun fondasi aplikasi absensi yang kokoh:

- Sistem otentikasi lengkap dengan Laravel Breeze
- Manajemen role (Admin dan Karyawan)
- Fitur CRUD penuh untuk manajemen pengguna
- Middleware untuk proteksi rute
- Interface yang responsif dan modern dengan Tailwind CSS
- Validasi data yang ketat
- Flash messages untuk feedback pengguna
- Navigation menu yang dinamis berdasarkan role

---

## 🚀 Langkah Selanjutnya (Part 2)

Fondasi aplikasi sudah sangat kuat. Di **Part 2**, kita akan melanjutkan dengan:

- Fitur Absensi Harian untuk Admin
- Fitur Absensi Karyawan (Check-in/Check-out)
- Riwayat Absensi
- Laporan dan Statistik
- Management Hari Libur

---

## 📚 Informasi Part 2

Untuk melanjutkan pengembangan aplikasi absensi dengan fitur-fitur lanjutan yaitu ``**Fitur utama Absensi pembuatan Logik dan view blade untuk menyimpan data absensi karyawan**``, kunjungi:

**[Part 2 - Panduan Absensi Project](https://github.com/ahmad-syaifuddin/part2-panduan-absensi-project.git)**

---

## 💡 Tips Pengembangan

- Selalu backup database sebelum melakukan perubahan besar
- Gunakan `php artisan migrate:fresh --seed` untuk reset database saat development
- Manfaatkan Laravel Debugbar untuk debugging
- Dokumentasikan setiap perubahan yang Anda buat

---

**🎊 SELAMAT!** Anda telah menyelesaikan Part 1 dari tutorial ini dengan sukses!
