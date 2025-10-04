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
            $table->string('role', 20)->default('karyawan')->after('email');
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
use App\Models\Employee; // <-- WAJIB TAMBAHKAN INI
use Illuminate\Support\Facades\Hash;
use Faker\Factory as Faker;

class UserSeeder extends Seeder
{
    public function run(): void
    {
        $faker = Faker::create('id_ID');

        // 1. Buat user Admin (HANYA data user, TANPA data employee)
        User::create([
            'name' => 'Admin Utama',
            'email' => 'admin@gmail.com',
            'password' => Hash::make('password'),
            'role' => 'admin', // Kolom 'role' tetap di tabel users untuk pembeda
        ]);

        // 2. Buat user Karyawan (buat user, LALU buat data employee-nya)
        $budiUser = User::create([
            'name' => 'Budi Karyawan', // Nama untuk login
            'email' => 'budi.karyawan@gmail.com',
            'password' => Hash::make('password'),
            'role' => 'karyawan',
        ]);

        Employee::create([
            'user_id' => $budiUser->id, // Hubungkan ke user Budi
            'full_name' => 'Budi Karyawan Setiawan', // Nama lengkap untuk HRD
            'employee_code' => 'K001',
            'position' => 'Staff IT',
            'department' => 'Teknologi Informasi',
            'hire_date' => now()->subMonths(6),
            'phone' => '089876543210',
            'address' => $faker->address,
        ]);

        // 3. Buat 13 Karyawan lainnya secara otomatis
        // Gunakan factory hanya untuk membuat user
        User::factory()->count(13)->create()->each(function ($user, $index) use ($faker) {
            // Lalu untuk setiap user, buat data employee-nya
            Employee::create([
                'user_id' => $user->id,
                'full_name' => $user->name,
                'employee_code' => 'K' . str_pad($index + 2, 3, '0', STR_PAD_LEFT), // K002, K003, ...
                'position' => $faker->randomElement(['Staff Marketing', 'Staff Keuangan', 'Developer']),
                'department' => $faker->randomElement(['Pemasaran', 'Keuangan', 'Teknologi Informasi']),
                'hire_date' => $faker->dateTimeBetween('-2 years', 'now'),
                'phone' => $faker->unique()->numerify('08##########'),
                'address' => $faker->address,
            ]);
        });
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
            'phone' => 'nullable|string|max:20',
            'gender' => 'nullable|in:Laki-laki,Perempuan',
        ]);

        User::create([
            'name' => $validated['name'],
            'email' => $validated['email'],
            'password' => Hash::make($validated['password']),
            'role' => $validated['role'],
            'phone' => $validated['phone'],
            'gender' => $validated['gender'],
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
            'phone' => 'nullable|string|max:20',
            'gender' => 'nullable|in:Laki-laki,Perempuan',
        ]);

        $user->name = $validated['name'];
        $user->email = $validated['email'];
        $user->role = $validated['role'];
        $user->phone = $validated['phone'];
        $user->gender = $validated['gender'];

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

### 📄 File: resources/views/admin/users/index.blade.php
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
                <div class="p-6 bg-white border-b border-gray-200">
                    <div class="flex justify-between items-center mb-4">
                        <h3 class="text-lg font-medium text-gray-900">Daftar Pengguna</h3>
                        <a href="{{ route('admin.users.create') }}" class="inline-flex items-center px-4 py-2 bg-gray-800 border border-transparent rounded-md font-semibold text-xs text-white uppercase tracking-widest hover:bg-gray-700 active:bg-gray-900 focus:outline-none focus:border-gray-900 focus:ring ring-gray-300 disabled:opacity-25 transition ease-in-out duration-150">
                            Tambah Pengguna
                        </a>
                    </div>
                    
                    @if(session('success'))
                        <div class="bg-green-100 border border-green-400 text-green-700 px-4 py-3 rounded relative mb-4" role="alert">
                            <span class="block sm:inline">{{ session('success') }}</span>
                        </div>
                    @endif

                    <div class="overflow-x-auto">
                        <table class="min-w-full divide-y divide-gray-200">
                            <thead class="bg-gray-50">
                                <tr>
                                    <th scope="col" class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Nama</th>
                                    <th scope="col" class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Email</th>
                                    <th scope="col" class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Peran</th>
                                    <th scope="col" class="relative px-6 py-3"><span class="sr-only">Aksi</span></th>
                                </tr>
                            </thead>
                            <tbody class="bg-white divide-y divide-gray-200">
                                @forelse($users as $user)
                                <tr>
                                    <td class="px-6 py-4 whitespace-nowrap">{{ $user->name }}</td>
                                    <td class="px-6 py-4 whitespace-nowrap">{{ $user->email }}</td>
                                    <td class="px-6 py-4 whitespace-nowrap">{{ ucfirst($user->role) }}</td>
                                    <td class="px-6 py-4 whitespace-nowrap text-right text-sm font-medium">
                                        <a href="{{ route('admin.users.show', $user->id) }}" class="text-blue-600 hover:text-blue-900 mr-2">Lihat</a>
                                        <a href="{{ route('admin.users.edit', $user->id) }}" class="text-indigo-600 hover:text-indigo-900 mr-2">Edit</a>
                                        <form action="{{ route('admin.users.destroy', $user->id) }}" method="POST" class="inline-block" onsubmit="return confirm('Apakah Anda yakin ingin menghapus pengguna ini?');">
                                            @csrf
                                            @method('DELETE')
                                            <button type="submit" class="text-red-600 hover:text-red-900">Hapus</button>
                                        </form>
                                    </td>
                                </tr>
                                @empty
                                <tr>
                                    <td colspan="4" class="px-6 py-4 whitespace-nowrap text-center text-gray-500">
                                        Tidak ada pengguna ditemukan.
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

### 📄 File: resources/views/admin/users/create.blade.php

```HTML
<x-app-layout>
    <x-slot name="header">
        <h2 class="font-semibold text-xl text-gray-800 leading-tight">
            {{ __('Manajemen Pengguna') }}
        </h2>
    </x-slot>

    <div class="py-12">
        <div class="max-w-7xl mx-auto sm:px-6 lg:px-8">
            <div class="bg-white overflow-hidden shadow-sm sm:rounded-lg">
                <div class="p-6 bg-white border-b border-gray-200">
                    <div class="flex justify-between items-center mb-4">
                        <h3 class="text-lg font-medium text-gray-900">Daftar Pengguna</h3>
                        <a href="{{ route('admin.users.create') }}" class="inline-flex items-center px-4 py-2 bg-gray-800 border border-transparent rounded-md font-semibold text-xs text-white uppercase tracking-widest hover:bg-gray-700 active:bg-gray-900 focus:outline-none focus:border-gray-900 focus:ring ring-gray-300 disabled:opacity-25 transition ease-in-out duration-150">
                            Tambah Pengguna
                        </a>
                    </div>
                    
                    @if(session('success'))
                        <div class="bg-green-100 border border-green-400 text-green-700 px-4 py-3 rounded relative mb-4" role="alert">
                            <span class="block sm:inline">{{ session('success') }}</span>
                        </div>
                    @endif

                    <div class="overflow-x-auto">
                        <table class="min-w-full divide-y divide-gray-200">
                            <thead class="bg-gray-50">
                                <tr>
                                    <th scope="col" class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Nama</th>
                                    <th scope="col" class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Email</th>
                                    <th scope="col" class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Peran</th>
                                    <th scope="col" class="relative px-6 py-3"><span class="sr-only">Aksi</span></th>
                                </tr>
                            </thead>
                            <tbody class="bg-white divide-y divide-gray-200">
                                @forelse($users as $user)
                                <tr>
                                    <td class="px-6 py-4 whitespace-nowrap">{{ $user->name }}</td>
                                    <td class="px-6 py-4 whitespace-nowrap">{{ $user->email }}</td>
                                    <td class="px-6 py-4 whitespace-nowrap">{{ ucfirst($user->role) }}</td>
                                    <td class="px-6 py-4 whitespace-nowrap text-right text-sm font-medium">
                                        <a href="{{ route('admin.users.show', $user->id) }}" class="text-blue-600 hover:text-blue-900 mr-2">Lihat</a>
                                        <a href="{{ route('admin.users.edit', $user->id) }}" class="text-indigo-600 hover:text-indigo-900 mr-2">Edit</a>
                                        <form action="{{ route('admin.users.destroy', $user->id) }}" method="POST" class="inline-block" onsubmit="return confirm('Apakah Anda yakin ingin menghapus pengguna ini?');">
                                            @csrf
                                            @method('DELETE')
                                            <button type="submit" class="text-red-600 hover:text-red-900">Hapus</button>
                                        </form>
                                    </td>
                                </tr>
                                @empty
                                <tr>
                                    <td colspan="4" class="px-6 py-4 whitespace-nowrap text-center text-gray-500">
                                        Tidak ada pengguna ditemukan.
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
### 📄 File: resources/views/admin/users/edit.blade.php

```HTML
<x-app-layout>
    <x-slot name="header">
        <h2 class="font-semibold text-xl text-gray-800 leading-tight">
            {{ __('Edit Pengguna') }}
        </h2>
    </x-slot>

    <div class="py-12">
        <div class="max-w-7xl mx-auto sm:px-6 lg:px-8">
            <div class="bg-white overflow-hidden shadow-sm sm:rounded-lg">
                <div class="p-6 bg-white border-b border-gray-200">
                    <form method="POST" action="{{ route('admin.users.update', $user->id) }}">
                        @csrf
                        @method('PUT')

                        <div>
                            <x-input-label for="name" :value="__('Nama')" />
                            <x-text-input id="name" class="block mt-1 w-full" type="text" name="name" :value="old('name', $user->name)" required autofocus />
                            <x-input-error :messages="$errors->get('name')" class="mt-2" />
                        </div>

                        <div class="mt-4">
                            <x-input-label for="email" :value="__('Email')" />
                            <x-text-input id="email" class="block mt-1 w-full" type="email" name="email" :value="old('email', $user->email)" required />
                            <x-input-error :messages="$errors->get('email')" class="mt-2" />
                        </div>
                        
                        <div class="mt-4">
                            <x-input-label for="role" :value="__('Peran')" />
                            <select id="role" name="role" class="block mt-1 w-full rounded-md shadow-sm border-gray-300 focus:border-indigo-300 focus:ring focus:ring-indigo-200 focus:ring-opacity-50">
                                <option value="karyawan" {{ old('role', $user->role) == 'karyawan' ? 'selected' : '' }}>Karyawan</option>
                                <option value="admin" {{ old('role', $user->role) == 'admin' ? 'selected' : '' }}>Admin</option>
                            </select>
                            <x-input-error :messages="$errors->get('role')" class="mt-2" />
                        </div>

                        <div class="mt-4">
                            <x-input-label for="gender" :value="__('Jenis Kelamin')" />
                            <select id="gender" name="gender" class="block mt-1 w-full rounded-md shadow-sm border-gray-300 focus:border-indigo-300 focus:ring focus:ring-indigo-200 focus:ring-opacity-50">
                                <option value="Laki-laki" {{ old('gender', $user->gender) == 'Laki-laki' ? 'selected' : '' }}>Laki-laki</option>
                                <option value="Perempuan" {{ old('gender', $user->gender) == 'Perempuan' ? 'selected' : '' }}>Perempuan</option>
                            </select>
                            <x-input-error :messages="$errors->get('gender')" class="mt-2" />
                        </div>

                        <div class="mt-4">
                            <x-input-label for="phone" :value="__('Nomor HP')" />
                            <x-text-input id="phone" class="block mt-1 w-full" type="text" name="phone" :value="old('phone', $user->phone)" required />
                            <x-input-error :messages="$errors->get('phone')" class="mt-2" />
                        </div>

                        <div class="mt-4">
                            <x-input-label for="password" :value="__('Password Baru (Kosongkan jika tidak ingin diubah)')" />
                            <x-text-input id="password" class="block mt-1 w-full" type="password" name="password" />
                            <x-input-error :messages="$errors->get('password')" class="mt-2" />
                        </div>

                        <div class="mt-4">
                            <x-input-label for="password_confirmation" :value="__('Konfirmasi Password Baru')" />
                            <x-text-input id="password_confirmation" class="block mt-1 w-full" type="password" name="password_confirmation" />
                            <x-input-error :messages="$errors->get('password_confirmation')" class="mt-2" />
                        </div>

                        <div class="flex items-center justify-end mt-4">
                            <x-primary-button>
                                {{ __('Perbarui Pengguna') }}
                            </x-primary-button>
                        </div>
                    </form>
                </div>
            </div>
        </div>
    </div>
</x-app-layout>
```

###📄 File: resources/views/admin/users/show.blade.php

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
                <div class="p-6 bg-white border-b border-gray-200">
                    <div class="mb-4">
                        <p class="text-gray-600">Nama: <span class="font-medium text-gray-900">{{ $user->name }}</span></p>
                        <p class="text-gray-600">Email: <span class="font-medium text-gray-900">{{ $user->email }}</span></p>
                        <p class="text-gray-600">Jenis Kelamin: <span class="font-medium text-gray-900">{{ $user->gender }}</span></p>
                        <p class="text-gray-600">Nomor HP: <span class="font-medium text-gray-900">{{ $user->phone }}</span></p>
                        <p class="text-gray-600">Peran: <span class="font-medium text-gray-900">{{ ucfirst($user->role) }}</span></p>
                        <p class="text-gray-600">Terdaftar Sejak: <span class="font-medium text-gray-900">{{ $user->created_at->format('d M Y H:i') }}</span></p>
                    </div>
                    
                    <a href="{{ route('admin.users.index') }}" class="inline-flex items-center px-4 py-2 bg-gray-200 border border-transparent rounded-md font-semibold text-xs text-gray-700 uppercase tracking-widest hover:bg-gray-300 focus:outline-none focus:border-gray-900 focus:ring ring-gray-300 disabled:opacity-25 transition ease-in-out duration-150">
                        Kembali
                    </a>
                </div>
            </div>
        </div>
    </div>
</x-app-layout>
```

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
                        <x-nav-link :href="route('admin.users.index')" :active="request()->routeIs('admin.users.index')">
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
                <x-responsive-nav-link :href="route('admin.users.index')" :active="request()->routeIs('admin.users.index')">
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
> Manajemen pengguna sudah selesai. Anda sudah bisa login sebagai admin@gmail.com dengan password "password", dan Anda bisa menambah, mengedit, melihat, dan menghapus pengguna lain melalui dashboard.

### Next lanjut Part 2
Untuk informasi lebih lanjut, kunjungi [Panduannya](https://github.com/ahmad-syaifuddin/part2-panduan-absensi-project.git).
