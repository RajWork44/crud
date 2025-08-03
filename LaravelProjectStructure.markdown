# Laravel Project with Custom Admin and Employee Authentication

This guide outlines the creation of a Laravel project with custom authentication for admin and employee roles, including employee registration, middleware, and respective dashboards. The project uses Laravel's built-in authentication system without Breeze.

## Prerequisites
- Laravel 10.x or later
- PHP 8.1 or later
- Composer
- MySQL or any supported database

## Step 1: Project Setup
1. Create a new Laravel project:
```bash
composer create-project laravel/laravel employee-management
cd employee-management
```

2. Configure the `.env` file with your database credentials:
```env
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=employee_management
DB_USERNAME=root
DB_PASSWORD=
```

## Step 2: Database Schema
Create the necessary database tables.

### Migration for `users` table (for both admins and employees)
```php
// database/migrations/2025_08_03_000000_create_users_table.php
use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration {
    public function up(): void
    {
        Schema::create('users', function (Blueprint $table) {
            $table->id();
            $table->string('name');
            $table->string('email')->unique();
            $table->string('password');
            $table->enum('role', ['admin', 'employee'])->default('employee');
            $table->timestamps();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('users');
    }
};
```

### Migration for `leaves` table
```php
// database/migrations/2025_08_03_000001_create_leaves_table.php
use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration {
    public function up(): void
    {
        Schema::create('leaves', function (Blueprint $table) {
            $table->id();
            $table->foreignId('user_id')->constrained()->onDelete('cascade');
            $table->date('start_date');
            $table->date('end_date');
            $table->text('reason');
            $table->enum('status', ['pending', 'approved', 'rejected'])->default('pending');
            $table->timestamps();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('leaves');
    }
};
```

Run migrations:
```bash
php artisan migrate
```

## Step 3: Models
Update the `User` model and create a `Leave` model.

### User Model
```php
// app/Models/User.php
namespace App\Models;

use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;

class User extends Authenticatable
{
    use Notifiable;

    protected $fillable = [
        'name', 'email', 'password', 'role',
    ];

    protected $hidden = [
        'password', 'remember_token',
    ];

    public function leaves()
    {
        return $this->hasMany(Leave::class);
    }
}
```

### Leave Model
```php
// app/Models/Leave.php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Leave extends Model
{
    protected $fillable = [
        'user_id', 'start_date', 'end_date', 'reason', 'status',
    ];

    public function user()
    {
        return $this->belongsTo(User::class);
    }
}
```

## Step 4: Middleware
Create custom middleware for role-based access.

### Admin Middleware
```php
// app/Http/Middleware/AdminMiddleware.php
namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;

class AdminMiddleware
{
    public function handle(Request $request, Closure $next)
    {
        if (Auth::check() && Auth::user()->role === 'admin') {
            return $next($request);
        }

        return redirect('/login')->with('error', 'Unauthorized access.');
    }
}
```

### Employee Middleware
```php
// app/Http/Middleware/EmployeeMiddleware.php
namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;

class EmployeeMiddleware
{
    public function handle(Request $request, Closure $next)
    {
        if (Auth::check() && Auth::user()->role === 'employee') {
            return $next($request);
        }

        return redirect('/login')->with('error', 'Unauthorized access.');
    }
}
```

Register middleware in `app/Http/Kernel.php`:
```php
// app/Http/Kernel.php
protected $middlewareAliases = [
    // ...
    'admin' => \App\Http\Middleware\AdminMiddleware::class,
    'employee' => \App\Http\Middleware\EmployeeMiddleware::class,
];
```

## Step 5: Authentication Controllers
Create controllers for login and registration.

### Login Controller
```php
// app/Http/Controllers/Auth/LoginController.php
namespace App\Http\Controllers\Auth;

use App\Http\Controllers\Controller;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;

class LoginController extends Controller
{
    public function showLoginForm()
    {
        return view('auth.login');
    }

    public function login(Request $request)
    {
        $credentials = $request->validate([
            'email' => 'required|email',
            'password' => 'required',
        ]);

        if (Auth::attempt($credentials)) {
            $request->session()->regenerate();

            if (Auth::user()->role === 'admin') {
                return redirect()->route('admin.dashboard');
            } elseif (Auth::user()->role === 'employee') {
                return redirect()->route('employee.dashboard');
            }
        }

        return back()->withErrors([
            'email' => 'The provided credentials do not match our records.',
        ]);
    }

    public function logout(Request $request)
    {
        Auth::logout();
        $request->session()->invalidate();
        $request->session()->regenerateToken();
        return redirect('/login');
    }
}
```

### Register Controller
```php
// app/Http/Controllers/Auth/RegisterController.php
namespace App\Http\Controllers\Auth;

use App\Http\Controllers\Controller;
use App\Models\User;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Hash;

class RegisterController extends Controller
{
    public function showRegistrationForm()
    {
        return view('auth.register');
    }

    public function register(Request $request)
    {
        $request->validate([
            'name' => 'required|string|max:255',
            'email' => 'required|string|email|max:255|unique:users',
            'password' => 'required|string|min:8|confirmed',
        ]);

        User::create([
            'name' => $request->name,
            'email' => $request->email,
            'password' => Hash::make($request->password),
            'role' => 'employee', // Only employees can register
        ]);

        return redirect('/login')->with('success', 'Registration successful. Please login.');
    }
}
```

## Step 6: Dashboard Controllers
Create controllers for admin and employee dashboards.

### Admin Controller
```php
// app/Http/Controllers/AdminController.php
namespace App\Http\Controllers;

use App\Models\User;
use Illuminate\Http\Request;

class AdminController extends Controller
{
    public function dashboard()
    {
        $employees = User::where('role', 'employee')->get();
        return view('admin.dashboard', compact('employees'));
    }
}
```

### Employee Controller
```php
// app/Http/Controllers/EmployeeController.php
namespace App\Http\Controllers;

use App\Models\Leave;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;

class EmployeeController extends Controller
{
    public function dashboard()
    {
        $leaves = Leave::where('user_id', Auth::id())->get();
        return view('employee.dashboard', compact('leaves'));
    }
}
```

## Step 7: Routes
Define routes in `routes/web.php`:
```php
// routes/web.php
use App\Http\Controllers\Auth\LoginController;
use App\Http\Controllers\Auth\RegisterController;
use App\Http\Controllers\AdminController;
use App\Http\Controllers\EmployeeController;
use Illuminate\Support\Facades\Route;

Route::get('/login', [LoginController::class, 'showLoginForm'])->name('login');
Route::post('/login', [LoginController::class, 'login']);
Route::post('/logout', [LoginController::class, 'logout'])->name('logout');

Route::get('/register', [RegisterController::class, 'showRegistrationForm'])->name('register');
Route::post('/register', [RegisterController::class, 'register']);

Route::middleware(['auth', 'admin'])->group(function () {
    Route::get('/admin/dashboard', [AdminController::class, 'dashboard'])->name('admin.dashboard');
});

Route::middleware(['auth', 'employee'])->group(function () {
    Route::get('/employee/dashboard', [EmployeeController::class, 'dashboard'])->name('employee.dashboard');
});

Route::get('/', function () {
    return redirect()->route('login');
});
```

## Step 8: Views
Create the necessary Blade templates.

### Layout
```php
// resources/views/layouts/app.blade.php
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Employee Management</title>
    <link href="https://cdn.jsdelivr.net/npm/tailwindcss@2.2.19/dist/tailwind.min.css" rel="stylesheet">
</head>
<body class="bg-gray-100">
    <div class="container mx-auto p-4">
        @yield('content')
    </div>
</body>
</html>
```

### Login View
```php
// resources/views/auth/login.blade.php
@extends('layouts.app')

@section('content')
<div class="max-w-md mx-auto bg-white p-6 rounded-lg shadow-md">
    <h2 class="text-2xl font-bold mb-4">Login</h2>
    @if (session('error'))
        <div class="bg-red-100 text-red-700 p-2 mb-4 rounded">{{ session('error') }}</div>
    @endif
    @if ($errors->has('email'))
        <div class="bg-red-100 text-red-700 p-2 mb-4 rounded">{{ $errors->first('email') }}</div>
    @endif
    <form method="POST" action="{{ route('login') }}">
        @csrf
        <div class="mb-4">
            <label for="email" class="block text-gray-700">Email</label>
            <input type="email" name="email" id="email" class="w-full border rounded p-2" required>
        </div>
        <div class="mb-4">
            <label for="password" class="block text-gray-700">Password</label>
            <input type="password" name="password" id="password" class="w-full border rounded p-2" required>
        </div>
        <button type="submit" class="w-full bg-blue-500 text-white p-2 rounded hover:bg-blue-600">Login</button>
    </form>
    <p class="mt-4 text-center">Don't have an account? <a href="{{ route('register') }}" class="text-blue-500">Register</a></p>
</div>
@endsection
```

### Register View
```php
// resources/views/auth/register.blade.php
@extends('layouts.app')

@section('content')
<div class="max-w-md mx-auto bg-white p-6 rounded-lg shadow-md">
    <h2 class="text-2xl font-bold mb-4">Register</h2>
    @if (session('success'))
        <div class="bg-green-100 text-green-700 p-2 mb-4 rounded">{{ session('success') }}</div>
    @endif
    @if ($errors->any())
        <div class="bg-red-100 text-red-700 p-2 mb-4 rounded">
            @foreach ($errors->all() as $error)
                <p>{{ $error }}</p>
            @endforeach
        </div>
    @endif
    <form method="POST" action="{{ route('register') }}">
        @csrf
        <div class="mb-4">
            <label for="name" class="block text-gray-700">Name</label>
            <input type="text" name="name" id="name" class="w-full border rounded p-2" required>
        </div>
        <div class="mb-4">
            <label for="email" class="block text-gray-700">Email</label>
            <input type="email" name="email" id="email" class="w-full border rounded p-2" required>
        </div>
        <div class="mb-4">
            <label for="password" class="block text-gray-700">Password</label>
            <input type="password" name="password" id="password" class="w-full border rounded p-2" required>
        </div>
        <div class="mb-4">
            <label for="password_confirmation" class="block text-gray-700">Confirm Password</label>
            <input type="password" name="password_confirmation" id="password_confirmation" class="w-full border rounded p-2" required>
        </div>
        <button type="submit" class="w-full bg-blue-500 text-white p-2 rounded hover:bg-blue-600">Register</button>
    </form>
    <p class="mt-4 text-center">Already have an account? <a href="{{ route('login') }}" class="text-blue-500">Login</a></p>
</div>
@endsection
```

### Admin Dashboard
```php
// resources/views/admin/dashboard.blade.php
@extends('layouts.app')

@section('content')
<div class="max-w-4xl mx-auto bg-white p-6 rounded-lg shadow-md">
    <h2 class="text-2xl font-bold mb-4">Admin Dashboard</h2>
    <form method="POST" action="{{ route('logout') }}">
        @csrf
        <button type="submit" class="bg-red-500 text-white p-2 rounded hover:bg-red-600">Logout</button>
    </form>
    <h3 class="text-xl font-semibold mt-6 mb-2">Employee List</h3>
    <table class="w-full border-collapse border">
        <thead>
            <tr class="bg-gray-200">
                <th class="border p-2">ID</th>
                <th class="border p-2">Name</th>
                <th class="border p-2">Email</th>
            </tr>
        </thead>
        <tbody>
            @foreach ($employees as $employee)
                <tr>
                    <td class="border p-2">{{ $employee->id }}</td>
                    <td class="border p-2">{{ $employee->name }}</td>
                    <td class="border p-2">{{ $employee->email }}</td>
                </tr>
            @endforeach
        </tbody>
    </table>
</div>
@endsection
```

### Employee Dashboard
```php
// resources/views/employee/dashboard.blade.php
@extends('layouts.app')

@section('content')
<div class="max-w-4xl mx-auto bg-white p-6 rounded-lg shadow-md">
    <h2 class="text-2xl font-bold mb-4">Employee Dashboard</h2>
    <form method="POST" action="{{ route('logout') }}">
        @csrf
        <button type="submit" class="bg-red-500 text-white p-2 rounded hover:bg-red-600">Logout</button>
    </form>
    <h3 class="text-xl font-semibold mt-6 mb-2">My Leave Requests</h3>
    <table class="w-full border-collapse border">
        <thead>
            <tr class="bg-gray-200">
                <th class="border p-2">ID</th>
                <th class="border p-2">Start Date</th>
                <th class="border p-2">End Date</th>
                <th class="border p-2">Reason</th>
                <th class="border p-2">Status</th>
            </tr>
        </thead>
        <tbody>
            @foreach ($leaves as $leave)
                <tr>
                    <td class="border p-2">{{ $leave->id }}</td>
                    <td class="border p-2">{{ $leave->start_date }}</td>
                    <td class="border p-2">{{ $leave->end_date }}</td>
                    <td class="border p-2">{{ $leave->reason }}</td>
                    <td class="border p-2">{{ $leave->status }}</td>
                </tr>
            @endforeach
        </tbody>
    </table>
</div>
@endsection
```

## Step 9: Configuration
Update `config/auth.php` to use the custom User model:
```php
// config/auth.php
'providers' => [
    'users' => [
        'driver' => 'eloquent',
        'model' => App\Models\User::class,
    ],
],
```

## Step 10: Testing
1. Start the Laravel development server:
```bash
php artisan serve
```

2. Create an admin user manually in the database or via a seeder:
```php
// database/seeders/DatabaseSeeder.php
use App\Models\User;
use Illuminate\Database\Seeder;
use Illuminate\Support\Facades\Hash;

class DatabaseSeeder extends Seeder
{
    public function run(): void
    {
        User::create([
            'name' => 'Admin User',
            'email' => 'admin@example.com',
            'password' => Hash::make('password'),
            'role' => 'admin',
        ]);
    }
}
```

Run the seeder:
```bash
php artisan db:seed
```

3. Access the application:
- Visit `http://localhost:8000/login` to log in.
- Register a new employee at `http://localhost:8000/register`.
- Admin can log in with `admin@example.com` and password `password`.
- Admin dashboard: `http://localhost:8000/admin/dashboard` (shows employee list).
- Employee dashboard: `http://localhost:8000/employee/dashboard` (shows leave list).

## Notes
- This project uses Tailwind CSS for basic styling via CDN.
- The employee registration is restricted to the 'employee' role.
- Admins can view all employees, while employees can only view their own leave requests.
- Middleware ensures role-based access control.
- CSRF protection is included in all forms.
- The leave system can be extended with additional functionality (e.g., leave request submission) as needed.