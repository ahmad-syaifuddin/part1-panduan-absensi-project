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
            $table->string('role', 20)->default('karyawan')->after('email');
            $table->string('phone', 20)->nullable()->after('role');
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

## 2.2.1 UserFactory
Kita akan memodifikasi UserFactory agar dapat menghasilkan data dummy yang realistis termasuk role, phone, dan gender, serta memastikan domain email adalah @gmail.com.

Kode Factory:
Buka database/factories/UserFactory.php dan ubah metode definition():

```PHP
<?php

namespace Database\Factories;

use Illuminate\Database\Eloquent\Factories\Factory;
use Illuminate\Support\Facades\Hash;
use Illuminate\Support\Str;
use Faker\Factory as FakerFactory;

/**
 * @extends \Illuminate\Database\Eloquent\Factories\Factory<\App\Models\User>
 */
class UserFactory extends Factory
{
    /**
     * The current password being used by the factory.
     */
    protected static ?string $password;

    /**
     * Define the model's default state.
     *
     * @return array<string, mixed>
     */
    public function definition(): array
    {
        $faker = FakerFactory::create('id_ID'); // Pake lokal Indonesia

        $gender = $faker->randomElement(['Laki-laki', 'Perempuan']);
        $name   = $gender == 'Laki-laki' ? $faker->firstNameMale() : $faker->firstNameFemale();
        $lastName = $faker->lastName();

        return [
            'name' => $name . ' ' . $lastName,
            'email' => strtolower(
                str_replace(['.', ' ', '-'], '', $name) .
                    '.' .
                    str_replace(['.', ' ', '-'], '', $lastName) .
                    $faker->unique()->numerify('##')
            ) . '@gmail.com',
            'email_verified_at' => now(),
            'password' => Hash::make('password'),
            'remember_token' => Str::random(10),
            'role' => 'karyawan',
            'phone' => $faker->unique()->numerify('08##########'),
            'gender' => $gender,
        ];
    }

    /**
     * Indicate that the model's email address should be unverified.
     */
    public function unverified(): static
    {
        return $this->state(fn(array $attributes) => [
            'email_verified_at' => null,
        ]);
    }
}
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
use Illuminate\Support\Str;
use Faker\Factory as Faker;

class UserSeeder extends Seeder
{
    public function run(): void
    {
        $faker = Faker::create('id_ID');

        // 1. Buat user Admin (Data tetap)
        User::create([
            'name' => 'Admin Utama',
            'email' => 'admin@gmail.com', 
            'password' => Hash::make('password'),
            'role' => 'admin',
            'phone' => '081234567890',
            'gender' => 'Laki-laki',
        ]);

        // 2. Buat user Karyawan (Data tetap)
        User::create([
            'name' => 'Budi Karyawan',
            'email' => 'budi.karyawan@gmail.com', 
            'password' => Hash::make('password'),
            'role' => 'karyawan',
            'phone' => '089876543210',
            'gender' => 'Laki-laki',
        ]);

        // 3. 6 user nama custom
        $customNames = ['Haldi', 'Ryandy', 'Maulidi', 'Rio', 'Aldy', 'Ahmad S'];

        foreach ($customNames as $name) {
            User::create([
                'name' => $name,
                'email' => strtolower(str_replace(' ', '', $name)) . '@gmail.com',
                'password' => Hash::make('password'),
                'role' => 'karyawan',
                'phone' => $faker->unique()->numerify('08##########'),
                'gender' => 'Laki-laki', // <-- fix laki-laki
                'email_verified_at' => now(),
                'remember_token' => Str::random(10),
            ]);
        }

        // 4. 7 user random pakai Factory
        User::factory()->count(7)->create();
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
lalu cek isi data di tabel users dannn boom langsung terisi data dummy otomatis dengan data yg cukup realistis

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

# ✍️ Langkah 4: CRUD Manajemen Pengguna (Admin)
Sekarang kita akan membuat fitur lengkap untuk mengelola pengguna (CRUD).

## 4.1. Membuat Controller
Kita akan membuat UserController yang berisi semua logika untuk CRUD pengguna.

Perintah Artisan:

```Bash
php artisan make:controller Admin/UserController
```
## 4.2. Logika Controller
Buka app/Http/Controllers/Admin/UserController.php dan tambahkan kode untuk metode index, create, store, show, edit, update, dan destroy.
```PHP
<?php

namespace App\Http\Controllers\Admin;

use App\Http\Controllers\Controller;
use App\Models\User;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Hash;
use Illuminate\Validation\Rule;

class UserController extends Controller
{
    public function index()
    {
        $users = User::latest()->paginate(10);
        return view('admin.users.index', compact('users'));
    }

    public function create()
    {
        return view('admin.users.create');
    }

    public function store(Request $request)
    {
        $validated = $request->validate([
            'name' => 'required|string|max:255',
            'email' => 'required|string|email|max:255|unique:users',
            'password' => 'required|string|min:8|confirmed',
            'role' => ['required', 'string', Rule::in(['admin', 'karyawan'])],
        ]);

        User::create([
            'name' => $validated['name'],
            'email' => $validated['email'],
            'password' => Hash::make($validated['password']),
            'role' => $validated['role'],
        ]);

        return redirect()->route('admin.users.index')->with('success', 'Pengguna berhasil ditambahkan!');
    }
    
    public function show(User $user)
    {
        return view('admin.users.show', compact('user'));
    }

    public function edit(User $user)
    {
        return view('admin.users.edit', compact('user'));
    }

    public function update(Request $request, User $user)
    {
        $validated = $request->validate([
            'name' => 'required|string|max:255',
            'email' => [
                'required',
                'string',
                'email',
                'max:255',
                Rule::unique('users')->ignore($user->id),
            ],
            'password' => 'nullable|string|min:8|confirmed',
            'role' => ['required', 'string', Rule::in(['admin', 'karyawan'])],
        ]);

        $user->name = $validated['name'];
        $user->email = $validated['email'];
        $user->role = $validated['role'];

        if ($request->filled('password')) {
            $user->password = Hash::make($validated['password']);
        }
        
        $user->save();

        return redirect()->route('admin.users.index')->with('success', 'Data pengguna berhasil diperbarui!');
    }

    public function destroy(User $user)
    {
        $user->delete();

        return redirect()->route('admin.users.index')->with('success', 'Pengguna berhasil dihapus!');
    }
}
```

# 4.3. Rute (Routes)
Buka file routes/web.php dan tambahkan grup rute di bawah Route::middleware(['auth'])->group(...).

Gunakan alias middleware admin yang baru kita buat untuk melindungi semua rute ini.

```PHP
Route::middleware(['auth', 'admin'])->prefix('admin')->name('admin.')->group(function () {
    Route::resource('users', \App\Http\Controllers\Admin\UserController::class);
});
```

# 4.4. Blade Views untuk CRUD
Terakhir, kita buat file-file Blade untuk antarmuka pengguna. Pastikan untuk membuat folder admin/users di dalam resources/views.
```
resources/views/admin/users/index.blade.php
```

```
resources/views/admin/users/create.blade.php
```

```
resources/views/admin/users/edit.blade.php
```

```
resources/views/admin/users/show.blade.php
```
