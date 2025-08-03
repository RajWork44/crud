# Laravel Employee Management System Setup

Below is a step-by-step guide to create a Laravel project for an Employee Management System with Admin and Employee panels, including authentication, employee CRUD, attendance, and leave management.

## Prerequisites
- PHP >= 8.1
- Composer
- Laravel CLI
- MySQL or any supported database

## Step 1: Create a New Laravel Project
Run the following commands to set up a new Laravel project:

```bash
composer create-project laravel/laravel EmployeeManagementSystem
cd EmployeeManagementSystem
```

## Step 2: Database Setup
1. Configure the `.env` file with your database credentials:
```env
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=employee_management
DB_USERNAME=root
DB_PASSWORD=
```

2. Create the database `employee_management` in MySQL:
```sql
CREATE DATABASE employee_management;
```

## Step 3: Install Laravel Breeze for Authentication
Laravel Breeze provides a simple authentication system.

```bash
composer require laravel/breeze --dev
php artisan breeze:install blade
php artisan migrate
npm install && npm run dev
```

## Step 4: Create Models and Migrations
Create models and migrations for User (with roles), Employee, Attendance, and Leave.

### User Model (with Role)
Update the `User` model to include a role field for admin/employee distinction.

```php
// app/Models/User.php
<?php
namespace App\Models;

use Illuminate\Foundation\Auth\User as Authenticatable;

class User extends Authenticatable
{
    protected $fillable = [
        'name', 'email', 'password', 'role',
    ];

    public function isAdmin()
    {
        return $this->role === 'admin';
    }

    public function isEmployee()
    {
        return $this->role === 'employee';
    }
}
```

Update the users migration to add a `role` column:

```php
// database/migrations/xxxx_create_users_table.php
Schema::create('users', function (Blueprint $table) {
    $table->id();
    $table->string('name');
    $table->string('email')->unique();
    $table->string('password');
    $table->string('role')->default('employee'); // Add role column
    $table->rememberToken();
    $table->timestamps();
});
```

### Employee Model
```bash
php artisan make:model Employee -m
```

```php
// app/Models/Employee.php
<?php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Employee extends Model
{
    protected $fillable = ['user_id', 'name', 'email', 'phone', 'department', 'position'];

    public function user()
    {
        return $this->belongsTo(User::class);
    }

    public function attendances()
    {
        return $this->hasMany(Attendance::class);
    }

    public function leaves()
    {
        return $this->hasMany(Leave::class);
    }
}
```

```php
// database/migrations/xxxx_create_employees_table.php
<?php
use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

class CreateEmployeesTable extends Migration
{
    public function up()
    {
        Schema::create('employees', function (Blueprint $table) {
            $table->id();
            $table->foreignId('user_id')->constrained()->onDelete('cascade');
            $table->string('name');
            $table->string('email')->unique();
            $table->string('phone')->nullable();
            $table->string('department')->nullable();
            $table->string('position')->nullable();
            $table->timestamps();
        });
    }

    public function down()
    {
        Schema::dropIfExists('employees');
    }
}
```

### Attendance Model
```bash
php artisan make:model Attendance -m
```

```php
// app/Models/Attendance.php
<?php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Attendance extends Model
{
    protected $fillable = ['employee_id', 'date', 'status'];

    public function employee()
    {
        return $this->belongsTo(Employee::class);
    }
}
```

```php
// database/migrations/xxxx_create_attendances_table.php
<?php
use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

class CreateAttendancesTable extends Migration
{
    public function up()
    {
        Schema::create('attendances', function (Blueprint $table) {
            $table->id();
            $table->foreignId('employee_id')->constrained()->onDelete('cascade');
            $table->date('date');
            $table->enum('status', ['present', 'absent', 'late']);
            $table->timestamps();
        });
    }

    public function down()
    {
        Schema::dropIfExists('attendances');
    }
}
```

### Leave Model
```bash
php artisan make:model Leave -m
```

```php
// app/Models/Leave.php
<?php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Leave extends Model
{
    protected $fillable = ['employee_id', 'start_date', 'end_date', 'reason', 'status'];

    public function employee()
    {
        return $this->belongsTo(Employee::class);
    }
}
```

```php
// database/migrations/xxxx_create_leaves_table.php
<?php
use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

class CreateLeavesTable extends Migration
{
    public function up()
    {
        Schema::create('leaves', function (Blueprint $table) {
            $table->id();
            $table->foreignId('employee_id')->constrained()->onDelete('cascade');
            $table->date('start_date');
            $table->date('end_date');
            $table->text('reason');
            $table->enum('status', ['pending', 'approved', 'rejected'])->default('pending');
            $table->timestamps();
        });
    }

    public function down()
    {
        Schema::dropIfExists('leaves');
    }
}
```

Run migrations:
```bash
php artisan migrate
```

## Step 5: Create Controllers
### Admin Controllers
```bash
php artisan make:controller Admin/EmployeeController
php artisan make:controller Admin/AttendanceController
php artisan make:controller Admin/LeaveController
```

```php
// app/Http/Controllers/Admin/EmployeeController.php
<?php
namespace App\Http\Controllers\Admin;

use App\Http\Controllers\Controller;
use App\Models\Employee;
use App\Models\User;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Hash;

class EmployeeController extends Controller
{
    public function index()
    {
        $employees = Employee::with('user')->get();
        return view('admin.employees.index', compact('employees'));
    }

    public function create()
    {
        return view('admin.employees.create');
    }

    public function store(Request $request)
    {
        $request->validate([
            'name' => 'required',
            'email' => 'required|email|unique:users,email',
            'password' => 'required|min:8',
            'phone' => 'nullable',
            'department' => 'nullable',
            'position' => 'nullable',
        ]);

        $user = User::create([
            'name' => $request->name,
            'email' => $request->email,
            'password' => Hash::make($request->password),
            'role' => 'employee',
        ]);

        Employee::create([
            'user_id' => $user->id,
            'name' => $request->name,
            'email' => $request->email,
            'phone' => $request->phone,
            'department' => $request->department,
            'position' => $request->position,
        ]);

        return redirect()->route('admin.employees.index')->with('success', 'Employee created successfully.');
    }

    public function edit(Employee $employee)
    {
        return view('admin.employees.edit', compact('employee'));
    }

    public function update(Request $request, Employee $employee)
    {
        $request->validate([
            'name' => 'required',
            'email' => 'required|email|unique:users,email,' . $employee->user_id,
            'phone' => 'nullable',
            'department' => 'nullable',
            'position' => 'nullable',
        ]);

        $employee->update($request->only(['name', 'email', 'phone', 'department', 'position']));
        $employee->user->update([
            'name' => $request->name,
            'email' => $request->email,
        ]);

        return redirect()->route('admin.employees.index')->with('success', 'Employee updated successfully.');
    }

    public function destroy(Employee $employee)
    {
        $employee->user->delete();
        $employee->delete();
        return redirect()->route('admin.employees.index')->with('success', 'Employee deleted successfully.');
    }
}
```

```php
// app/Http/Controllers/Admin/AttendanceController.php
<?php
namespace App\Http\Controllers\Admin;

use App\Http\Controllers\Controller;
use App\Models\Attendance;
use App\Models\Employee;
use Illuminate\Http\Request;

class AttendanceController extends Controller
{
    public function index()
    {
        $attendances = Attendance::with('employee')->get();
        return view('admin.attendances.index', compact('attendances'));
    }

    public function create()
    {
        $employees = Employee::all();
        return view('admin.attendances.create', compact('employees'));
    }

    public function store(Request $request)
    {
        $request->validate([
            'employee_id' => 'required|exists:employees,id',
            'date' => 'required|date',
            'status' => 'required|in:present,absent,late',
        ]);

        Attendance::create($request->all());
        return redirect()->route('admin.attendances.index')->with('success', 'Attendance recorded.');
    }

    public function edit(Attendance $attendance)
    {
        $employees = Employee::all();
        return view('admin.attendances.edit', compact('attendance', 'employees'));
    }

    public function update(Request $request, Attendance $attendance)
    {
        $request->validate([
            'employee_id' => 'required|exists:employees,id',
            'date' => 'required|date',
            'status' => 'required|in:present,absent,late',
        ]);

        $attendance->update($request->all());
        return redirect()->route('admin.attendances.index')->with('success', 'Attendance updated.');
    }

    public function destroy(Attendance $attendance)
    {
        $attendance->delete();
        return redirect()->route('admin.attendances.index')->with('success', 'Attendance deleted.');
    }
}
```

```php
// app/Http/Controllers/Admin/LeaveController.php
<?php
namespace App\Http\Controllers\Admin;

use App\Http\Controllers\Controller;
use App\Models\Leave;
use App\Models\Employee;
use Illuminate\Http\Request;

class LeaveController extends Controller
{
    public function index()
    {
        $leaves = Leave::with('employee')->get();
        return view('admin.leaves.index', compact('leaves'));
    }

    public function edit(Leave $leave)
    {
        return view('admin.leaves.edit', compact('leave'));
    }

    public function update(Request $request, Leave $leave)
    {
        $request->validate([
            'status' => 'required|in:pending,approved,rejected',
        ]);

        $leave->update(['status' => $request->status]);
        return redirect()->route('admin.leaves.index')->with('success', 'Leave status updated.');
    }

    public function destroy(Leave $leave)
    {
        $leave->delete();
        return redirect()->route('admin.leaves.index')->with('success', 'Leave deleted.');
    }
}
```

### Employee Controllers
```bash
php artisan make:controller Employee/LeaveController
php artisan make:controller Employee/AttendanceController
```

```php
// app/Http/Controllers/Employee/LeaveController.php
<?php
namespace App\Http\Controllers\Employee;

use App\Http\Controllers\Controller;
use App\Models\Leave;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;

class LeaveController extends Controller
{
    public function index()
    {
        $leaves = Leave::where('employee_id', Auth::user()->employee->id)->get();
        return view('employee.leaves.index', compact('leaves'));
    }

    public function create()
    {
        return view('employee.leaves.create');
    }

    public function store(Request $request)
    {
        $request->validate([
            'start_date' => 'required|date',
            'end_date' => 'required|date|after_or_equal:start_date',
            'reason' => 'required',
        ]);

        Leave::create([
            'employee_id' => Auth::user()->employee->id,
            'start_date' => $request->start_date,
            'end_date' => $request->end_date,
            'reason' => $request->reason,
            'status' => 'pending',
        ]);

        return redirect()->route('employee.leaves.index')->with('success', 'Leave request submitted.');
    }
}
```

```php
// app/Http/Controllers/Employee/AttendanceController.php
<?php
namespace App\Http\Controllers\Employee;

use App\Http\Controllers\Controller;
use App\Models\Attendance;
use Illuminate\Support\Facades\Auth;

class AttendanceController extends Controller
{
    public function index()
    {
        $attendances = Attendance::where('employee_id', Auth::user()->employee->id)->get();
        return view('employee.attendances.index', compact('attendances'));
    }
}
```

## Step 6: Middleware for Role-Based Access
Create middleware to restrict access based on roles.

```bash
php artisan make:middleware Role
```

```php
// app/Http/Middleware/Role.php
<?php
namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;

class Role
{
    public function handle(Request $request, Closure $next, $role)
    {
        if (!Auth::check() || Auth::user()->role !== $role) {
            return redirect('/home');
        }
        return $next($request);
    }
}
```

Register the middleware in `app/Http/Kernel.php`:

```php
protected $routeMiddleware = [
    // ...
    'role' => \App\Http\Middleware\Role::class,
];
```

## Step 7: Routes
Define routes in `routes/web.php`:

```php
<?php
use Illuminate\Support\Facades\Route;
use App\Http\Controllers\Admin\EmployeeController;
use App\Http\Controllers\Admin\AttendanceController;
use App\Http\Controllers\Admin\LeaveController;
use App\Http\Controllers\Employee\LeaveController as EmployeeLeaveController;
use App\Http\Controllers\Employee\AttendanceController as EmployeeAttendanceController;

Route::get('/', function () {
    return view('welcome');
});

Auth::routes();

Route::get('/home', [App\Http\Controllers\HomeController::class, 'index'])->name('home');

Route::middleware(['auth', 'role:admin'])->prefix('admin')->group(function () {
    Route::resource('employees', EmployeeController::class);
    Route::resource('attendances', AttendanceController::class);
    Route::resource('leaves', LeaveController::class);
});

Route::middleware(['auth', 'role:employee'])->prefix('employee')->group(function () {
    Route::resource('leaves', EmployeeLeaveController::class)->only(['index', 'create', 'store']);
    Route::resource('attendances', EmployeeAttendanceController::class)->only(['index']);
});
```

## Step 8: Blade Views
Create Blade views for Admin and Employee panels.

### Admin Views
```bash
mkdir -p resources/views/admin/employees
mkdir -p resources/views/admin/attendances
mkdir -p resources/views/admin/leaves
```

```php
// resources/views/admin/employees/index.blade.php
<x-app-layout>
    <div class="py-12">
        <div class="max-w-7xl mx-auto sm:px-6 lg:px-8">
            <div class="bg-white overflow-hidden shadow-sm sm:rounded-lg">
                <div class="p-6 bg-white border-b border-gray-200">
                    <h2>Employees</h2>
                    <a href="{{ route('admin.employees.create') }}" class="btn btn-primary">Add Employee</a>
                    <table class="table">
                        <thead>
                            <tr>
                                <th>Name</th>
                                <th>Email</th>
                                <th>Phone</th>
                                <th>Department</th>
                                <th>Position</th>
                                <th>Actions</th>
                            </tr>
                        </thead>
                        <tbody>
                            @foreach($employees as $employee)
                                <tr>
                                    <td>{{ $employee->name }}</td>
                                    <td>{{ $employee->email }}</td>
                                    <td>{{ $employee->phone }}</td>
                                    <td>{{ $employee->department }}</td>
                                    <td>{{ $employee->position }}</td>
                                    <td>
                                        <a href="{{ route('admin.employees.edit', $employee) }}" class="btn btn-sm btn-warning">Edit</a>
                                        <form action="{{ route('admin.employees.destroy', $employee) }}" method="POST" style="display:inline;">
                                            @csrf
                                            @method('DELETE')
                                            <button type="submit" class="btn btn-sm btn-danger">Delete</button>
                                        </form>
                                    </td>
                                </tr>
                            @endforeach
                        </tbody>
                    </table>
                </div>
            </div>
        </div>
    </div>
</x-app-layout>
```

```php
// resources/views/admin/employees/create.blade.php
<x-app-layout>
    <div class="py-12">
        <div class="max-w-7xl mx-auto sm:px-6 lg:px-8">
            <div class="bg-white overflow-hidden shadow-sm sm:rounded-lg">
                <div class="p-6 bg-white border-b border-gray-200">
                    <h2>Add Employee</h2>
                    <form method="POST" action="{{ route('admin.employees.store') }}">
                        @csrf
                        <div>
                            <label>Name</label>
                            <input type="text" name="name" class="form-control" required>
                        </div>
                        <div>
                            <label>Email</label>
                            <input type="email" name="email" class="form-control" required>
                        </div>
                        <div>
                            <label>Password</label>
                            <input type="password" name="password" class="form-control" required>
                        </div>
                        <div>
                            <label>Phone</label>
                            <input type="text" name="phone" class="form-control">
                        </div>
                        <div>
                            <label>Department</label>
                            <input type="text" name="department" class="form-control">
                        </div>
                        <div>
                            <label>Position</label>
                            <input type="text" name="position" class="form-control">
                        </div>
                        <button type="submit" class="btn btn-primary">Save</button>
                    </form>
                </div>
            </div>
        </div>
    </div>
</x-app-layout>
```

```php
// resources/views/admin/employees/edit.blade.php
<x-app-layout>
    <div class="py-12">
        <div class="max-w-7xl mx-auto sm:px-6 lg:px-8">
            <div class="bg-white overflow-hidden shadow-sm sm:rounded-lg">
                <div class="p-6 bg-white border-b border-gray-200">
                    <h2>Edit Employee</h2>
                    <form method="POST" action="{{ route('admin.employees.update', $employee) }}">
                        @csrf
                        @method('PUT')
                        <div>
                            <label>Name</label>
                            <input type="text" name="name" value="{{ $employee->name }}" class="form-control" required>
                        </div>
                        <div>
                            <label>Email</label>
                            <input type="email" name="email" value="{{ $employee->email }}" class="form-control" required>
                        </div>
                        <div>
                            <label>Phone</label>
                            <input type="text" name="phone" value="{{ $employee->phone }}" class="form-control">
                        </div>
                        <div>
                            <label>Department</label>
                            <input type="text" name="department" value="{{ $employee->department }}" class="form-control">
                        </div>
                        <div>
                            <label>Position</label>
                            <input type="text" name="position" value="{{ $employee->position }}" class="form-control">
                        </div>
                        <button type="submit" class="btn btn-primary">Update</button>
                    </form>
                </div>
            </div>
        </div>
    </div>
</x-app-layout>
```

```php
// resources/views/admin/attendances/index.blade.php
<x-app-layout>
    <div class="py-12">
        <div class="max-w-7xl mx-auto sm:px-6 lg:px-8">
            <div class="bg-white overflow-hidden shadow-sm sm:rounded-lg">
                <div class="p-6 bg-white border-b border-gray-200">
                    <h2>Attendances</h2>
                    <a href="{{ route('admin.attendances.create') }}" class="btn btn-primary">Add Attendance</a>
                    <table class="table">
                        <thead>
                            <tr>
                                <th>Employee</th>
                                <th>Date</th>
                                <th>Status</th>
                                <th>Actions</th>
                            </tr>
                        </thead>
                        <tbody>
                            @foreach($attendances as $attendance)
                                <tr>
                                    <td>{{ $attendance->employee->name }}</td>
                                    <td>{{ $attendance->date }}</td>
                                    <td>{{ $attendance->status }}</td>
                                    <td>
                                        <a href="{{ route('admin.attendances.edit', $attendance) }}" class="btn btn-sm btn-warning">Edit</a>
                                        <form action="{{ route('admin.attendances.destroy', $attendance) }}" method="POST" style="display:inline;">
                                            @csrf
                                            @method('DELETE')
                                            <button type="submit" class="btn btn-sm btn-danger">Delete</button>
                                        </form>
                                    </td>
                                </tr>
                            @endforeach
                        </tbody>
                    </table>
                </div>
            </div>
        </div>
    </div>
</x-app-layout>
```

```php
// resources/views/admin/attendances/create.blade.php
<x-app-layout>
    <div class="py-12">
        <div class="max-w-7xl mx-auto sm:px-6 lg:px-8">
            <div class="bg-white overflow-hidden shadow-sm sm:rounded-lg">
                <div class="p-6 bg-white border-b border-gray-200">
                    <h2>Add Attendance</h2>
                    <form method="POST" action="{{ route('admin.attendances.store') }}">
                        @csrf
                        <div>
                            <label>Employee</label>
                            <select name="employee_id" class="form-control" required>
                                @foreach($employees as $employee)
                                    <option value="{{ $employee->id }}">{{ $employee->name }}</option>
                                @endforeach
                            </select>
                        </div>
                        <div>
                            <label>Date</label>
                            <input type="date" name="date" class="form-control" required>
                        </div>
                        <div>
                            <label>Status</label>
                            <select name="status" class="form-control" required>
                                <option value="present">Present</option>
                                <option value="absent">Absent</option>
                                <option value="late">Late</option>
                            </select>
                        </div>
                        <button type="submit" class="btn btn-primary">Save</button>
                    </form>
                </div>
            </div>
        </div>
    </div>
</x-app-layout>
```

```php
// resources/views/admin/attendances/edit.blade.php
<x-app-layout>
    <div class="py-12">
        <div class="max-w-7xl mx-auto sm:px-6 lg:px-8">
            <div class="bg-white overflow-hidden shadow-sm sm:rounded-lg">
                <div class="p-6 bg-white border-b border-gray-200">
                    <h2>Edit Attendance</h2>
                    <form method="POST" action="{{ route('admin.attendances.update', $attendance) }}">
                        @csrf
                        @method('PUT')
                        <div>
                            <label>Employee</label>
                            <select name="employee_id" class="form-control" required>
                                @foreach($employees as $employee)
                                    <option value="{{ $employee->id }}" {{ $employee->id == $attendance->employee_id ? 'selected' : '' }}>{{ $employee->name }}</option>
                                @endforeach
                            </select>
                        </div>
                        <div>
                            <label>Date</label>
                            <input type="date" name="date" value="{{ $attendance->date }}" class="form-control" required>
                        </div>
                        <div>
                            <label>Status</label>
                            <select name="status" class="form-control" required>
                                <option value="present" {{ $attendance->status == 'present' ? 'selected' : '' }}>Present</option>
                                <option value="absent" {{ $attendance->status == 'absent' ? 'selected' : '' }}>Absent</option>
                                <option value="late" {{ $attendance->status == 'late' ? 'selected' : '' }}>Late</option>
                            </select>
                        </div>
                        <button type="submit" class="btn btn-primary">Update</button>
                    </form>
                </div>
            </div>
        </div>
    </div>
</x-app-layout>
```

```php
// resources/views/admin/leaves/index.blade.php
<x-app-layout>
    <div class="py-12">
        <div class="max-w-7xl mx-auto sm:px-6 lg:px-8">
            <div class="bg-white overflow-hidden shadow-sm sm:rounded-lg">
                <div class="p-6 bg-white border-b border-gray-200">
                    <h2>Leaves</h2>
                    <table class="table">
                        <thead>
                            <tr>
                                <th>Employee</th>
                                <th>Start Date</th>
                                <th>End Date</th>
                                <th>Reason</th>
                                <th>Status</th>
                                <th>Actions</th>
                            </tr>
                        </thead>
                        <tbody>
                            @foreach($leaves as $leave)
                                <tr>
                                    <td>{{ $leave->employee->name }}</td>
                                    <td>{{ $leave->start_date }}</td>
                                    <td>{{ $leave->end_date }}</td>
                                    <td>{{ $leave->reason }}</td>
                                    <td>{{ $leave->status }}</td>
                                    <td>
                                        <a href="{{ route('admin.leaves.edit', $leave) }}" class="btn btn-sm btn-warning">Edit</a>
                                        <form action="{{ route('admin.leaves.destroy', $leave) }}" method="POST" style="display:inline;">
                                            @csrf
                                            @method('DELETE')
                                            <button type="submit" class="btn btn-sm btn-danger">Delete</button>
                                        </form>
                                    </td>
                                </tr>
                            @endforeach
                        </tbody>
                    </table>
                </div>
            </div>
        </div>
    </div>
</x-app-layout>
```

```php
// resources/views/admin/leaves/edit.blade.php
<x-app-layout>
    <div class="py-12">
        <div class="max-w-7xl mx-auto sm:px-6 lg:px-8">
            <div class="bg-white overflow-hidden shadow-sm sm:rounded-lg">
                <div class="p-6 bg-white border-b border-gray-200">
                    <h2>Edit Leave</h2>
                    <form method="POST" action="{{ route('admin.leaves.update', $leave) }}">
                        @csrf
                        @method('PUT')
                        <div>
                            <label>Status</label>
                            <select name="status" class="form-control" required>
                                <option value="pending" {{ $leave->status == 'pending' ? 'selected' : '' }}>Pending</option>
                                <option value="approved" {{ $leave->status == 'approved' ? 'selected' : '' }}>Approved</option>
                                <option value="rejected" {{ $leave->status == 'rejected' ? 'selected' : '' }}>Rejected</option>
                            </select>
                        </div>
                        <button type="submit" class="btn btn-primary">Update</button>
                    </form>
                </div>
            </div>
        </div>
    </div>
</x-app-layout>
```

### Employee Views
```bash
mkdir -p resources/views/employee/leaves
mkdir -p resources/views/employee/attendances
```

```php
// resources/views/employee/leaves/index.blade.php
<x-app-layout>
    <div class="py-12">
        <div class="max-w-7xl mx-auto sm:px-6 lg:px-8">
            <div class="bg-white overflow-hidden shadow-sm sm:rounded-lg">
                <div class="p-6 bg-white border-b border-gray-200">
                    <h2>My Leaves</h2>
                    <a href="{{ route('employee.leaves.create') }}" class="btn btn-primary">Request Leave</a>
                    <table class="table">
                        <thead>
                            <tr>
                                <th>Start Date</th>
                                <th>End Date</th>
                                <th>Reason</th>
                                <th>Status</th>
                            </tr>
                        </thead>
                        <tbody>
                            @foreach($leaves as $leave)
                                <tr>
                                    <td>{{ $leave->start_date }}</td>
                                    <td>{{ $leave->end_date }}</td>
                                    <td>{{ $leave->reason }}</td>
                                    <td>{{ $leave->status }}</td>
                                </tr>
                            @endforeach
                        </tbody>
                    </table>
                </div>
            </div>
        </div>
    </div>
</x-app-layout>
```

```php
// resources/views/employee/leaves/create.blade.php
<x-app-layout>
    <div class="py-12">
        <div class="max-w-7xl mx-auto sm:px-6 lg:px-8">
            <div class="bg-white overflow-hidden shadow-sm sm:rounded-lg">
                <div class="p-6 bg-white border-b border-gray-200">
                    <h2>Request Leave</h2>
                    <form method="POST" action="{{ route('employee.leaves.store') }}">
                        @csrf
                        <div>
                            <label>Start Date</label>
                            <input type="date" name="start_date" class="form-control" required>
                        </div>
                        <div>
                            <label>End Date</label>
                            <input type="date" name="end_date" class="form-control" required>
                        </div>
                        <div>
                            <label>Reason</label>
                            <textarea name="reason" class="form-control" required></textarea>
                        </div>
                        <button type="submit" class="btn btn-primary">Submit</button>
                    </form>
                </div>
            </div>
        </div>
    </div>
</x-app-layout>
```

```php
// resources/views/employee/attendances/index.blade.php
<x-app-layout>
    <div class="py-12">
        <div class="max-w-7xl mx-auto sm:px-6 lg:px-8">
            <div class="bg-white overflow-hidden shadow-sm sm:rounded-lg">
                <div class="p-6 bg-white border-b border-gray-200">
                    <h2>My Attendances</h2>
                    <table class="table">
                        <thead>
                            <tr>
                                <th>Date</th>
                                <th>Status</th>
                            </tr>
                        </thead>
                        <tbody>
                            @foreach($attendances as $attendance)
                                <tr>
                                    <td>{{ $attendance->date }}</td>
                                    <td>{{ $attendance->status }}</td>
                                </tr>
                            @endforeach
                        </tbody>
                    </table>
                </div>
            </div>
        </div>
    </div>
</x-app-layout>
```

## Step 9: Update HomeController
Modify the `HomeController` to redirect users based on their role.

```php
// app/Http/Controllers/HomeController.php
<?php
namespace App\Http\Controllers;

use Illuminate\Support\Facades\Auth;

class HomeController extends Controller
{
    public function __construct()
    {
        $this->middleware('auth');
    }

    public function index()
    {
        if (Auth::user()->isAdmin()) {
            return redirect()->route('admin.employees.index');
        } elseif (Auth::user()->isEmployee()) {
            return redirect()->route('employee.leaves.index');
        }
        return view('home');
    }
}
```

## Step 10: Add CSS (Optional)
Add Bootstrap for styling in `resources/views/layouts/app.blade.php`:

```php
<!DOCTYPE html>
<html lang="{{ str_replace('_', '-', app()->getLocale()) }}">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title>{{ config('app.name', 'Laravel') }}</title>
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" rel="stylesheet">
</head>
<body>
    @yield('content')
    <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/js/bootstrap.bundle.min.js"></script>
</body>
</html>
```

## Step 11: Seed Database (Optional)
Create a seeder to add an admin user:

```bash
php artisan make:seeder AdminUserSeeder
```

```php
// database/seeders/AdminUserSeeder.php
<?php
namespace Database\Seeders;

use Illuminate\Database\Seeder;
use App\Models\User;
use App\Models\Employee;
use Illuminate\Support\Facades\Hash;

class AdminUserSeeder extends Seeder
{
    public function run()
    {
        $user = User::create([
            'name' => 'Admin',
            'email' => 'admin@example.com',
            'password' => Hash::make('password'),
            'role' => 'admin',
        ]);

        Employee::create([
            'user_id' => $user->id,
            'name' => 'Admin',
            'email' => 'admin@example.com',
        ]);
    }
}
```

Run the seeder:
```bash
php artisan db:seed --class=AdminUserSeeder
```

## Step 12: Run the Application
```bash
php artisan serve
```

Access the application at `http://localhost:8000`. Log in with:
- Admin: `admin@example.com` / `password`
- Create employee users via the admin panel.

## Features
- **Admin Panel**:
  - Employee CRUD (Create, Read, Update, Delete)
  - Attendance CRUD
  - Leave approval/rejection
- **Employee Panel**:
  - View attendance
  - Request leaves
  - View leave status

This setup provides a fully functional Employee Management System with role-based access control.