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
