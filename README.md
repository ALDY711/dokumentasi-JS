ksfkfj# 📘 Dokumentasi Implementasi Teknis - Sistem Manajemen Gudang

## 🎯 Executive Summary

Sistem Manajemen Gudang adalah aplikasi berbasis web yang dibangun menggunakan **Laravel 10** dengan arsitektur MVC (Model-View-Controller). Aplikasi ini mengimplementasikan full CRUD operations untuk manajemen inventori, gudang, supplier, kategori produk, dan customer database.

**Tech Stack:**
- Backend: Laravel 10 (PHP 8.1+)
- Frontend: Blade Templates + Vanilla CSS/JS
- Database: MySQL/MariaDB
- Authentication: Laravel default
- Asset Bundling: Vite

---

## 📊 Database Design & Implementation

### Database Schema Overview

Database dirancang dengan normalisasi proper untuk menghindari duplikasi data dan memastikan integritas referensial.

### Detailed Table Structures

#### 1. `users` Table

**Purpose:** Menyimpan data pengguna sistem untuk autentikasi dan otorisasi.

```sql
CREATE TABLE users (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255) NOT NULL UNIQUE,
    email_verified_at TIMESTAMP NULL,
    password VARCHAR(255) NOT NULL,
    remember_token VARCHAR(100) NULL,
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL,
    INDEX idx_email (email)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

**Migration File:** `2014_10_12_000000_create_users_table.php`

**Model:** `App\Models\User`

**Relationships:**
- None (base authentication)

---

#### 2. `categories` Table

**Purpose:** Klasifikasi produk berdasarkan kategori untuk memudahkan organisasi dan pencarian.

```sql
CREATE TABLE categories (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    description TEXT NULL,
    status ENUM('active', 'inactive') NOT NULL DEFAULT 'active',
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL,
    INDEX idx_status (status),
    INDEX idx_name (name)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

**Migration File:** `2025_11_28_000001_create_categories_table.php`

**Model:** `App\Models\Category`

```php
class Category extends Model
{
    protected $fillable = ['name', 'description', 'status'];
    
    // Relationship: One category has many products
    public function products()
    {
        return $this->hasMany(Product::class);
    }
}
```

**Business Logic:**
- Soft status management (active/inactive)
- Cascade consideration: Saat category dihapus, products yang terkait akan memiliki `category_id = NULL` (set null on delete)

---

#### 3. `suppliers` Table

**Purpose:** Database vendor/supplier yang memasok produk ke perusahaan.

```sql
CREATE TABLE suppliers (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    contact_person VARCHAR(255) NULL,
    email VARCHAR(255) NULL,
    phone VARCHAR(20) NULL,
    address TEXT NULL,
    status ENUM('active', 'inactive') NOT NULL DEFAULT 'active',
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL,
    INDEX idx_status (status),
    INDEX idx_name (name)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

**Migration File:** `2025_11_28_000002_create_suppliers_table.php`

**Model:** `App\Models\Supplier`

```php
class Supplier extends Model
{
    protected $fillable = [
        'name', 'contact_person', 'email', 
        'phone', 'address', 'status'
    ];
    
    // Relationship: One supplier has many products
    public function products()
    {
        return $this->hasMany(Product::class);
    }
}
```

---

#### 4. `customers` Table

**Purpose:** Database pelanggan untuk sistem order (future implementation).

```sql
CREATE TABLE customers (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255) NULL,
    phone VARCHAR(20) NULL,
    address TEXT NULL,
    status ENUM('active', 'inactive') NOT NULL DEFAULT 'active',
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL,
    INDEX idx_status (status),
    INDEX idx_name (name),
    INDEX idx_email (email)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

**Migration File:** `2025_11_28_000003_create_customers_table.php`

**Model:** `App\Models\Customer`

```php
class Customer extends Model
{
    protected $fillable = [
        'name', 'email', 'phone', 'address', 'status'
    ];
    
    // Future: Relationship to orders
    // public function orders()
    // {
    //     return $this->hasMany(Order::class);
    // }
}
```

---

#### 5. `warehouses` Table

**Purpose:** Multi-warehouse management untuk tracking lokasi penyimpanan produk.

```sql
CREATE TABLE warehouses (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    code VARCHAR(255) NOT NULL UNIQUE,
    location VARCHAR(255) NULL,
    address TEXT NULL,
    phone VARCHAR(20) NULL,
    manager_name VARCHAR(255) NULL,
    capacity DECIMAL(10,2) NOT NULL DEFAULT 0 COMMENT 'Capacity in m²',
    status ENUM('active', 'inactive') NOT NULL DEFAULT 'active',
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL,
    UNIQUE KEY unique_code (code),
    INDEX idx_status (status),
    INDEX idx_location (location)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

**Migration File:** `2026_01_09_000000_create_warehouses_table.php`

**Model:** `App\Models\Warehouse`

```php
class Warehouse extends Model
{
    protected $fillable = [
        'name', 'code', 'location', 'address',
        'phone', 'manager_name', 'capacity', 'status'
    ];
    
    protected $casts = [
        'capacity' => 'decimal:2'
    ];
    
    // Future: Relationship to warehouse_stock pivot table
    public function products()
    {
        return $this->belongsToMany(Product::class, 'warehouse_stock')
                    ->withPivot('quantity', 'location_code')
                    ->withTimestamps();
    }
}
```

**Unique Constraint:**
- `code`: Kode warehouse harus unique untuk identifikasi

---

#### 6. `products` Table

**Purpose:** Master data produk dengan informasi lengkap untuk inventory management.

```sql
CREATE TABLE products (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    sku VARCHAR(255) NOT NULL UNIQUE,
    description TEXT NULL,
    category_id BIGINT UNSIGNED NULL,
    supplier_id BIGINT UNSIGNED NULL,
    price DECIMAL(15,2) NOT NULL DEFAULT 0,
    cost DECIMAL(15,2) NOT NULL DEFAULT 0,
    min_stock INT NOT NULL DEFAULT 0,
    max_stock INT NOT NULL DEFAULT 0,
    reorder_point INT NOT NULL DEFAULT 0,
    unit VARCHAR(50) NOT NULL DEFAULT 'pcs',
    track_batch BOOLEAN NOT NULL DEFAULT 0,
    track_expiry BOOLEAN NOT NULL DEFAULT 0,
    image VARCHAR(255) NULL,
    barcode VARCHAR(255) NULL,
    status ENUM('active', 'inactive') NOT NULL DEFAULT 'active',
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL,
    UNIQUE KEY unique_sku (sku),
    FOREIGN KEY (category_id) REFERENCES categories(id) ON DELETE SET NULL,
    FOREIGN KEY (supplier_id) REFERENCES suppliers(id) ON DELETE SET NULL,
    INDEX idx_status (status),
    INDEX idx_category (category_id),
    INDEX idx_supplier (supplier_id),
    INDEX idx_sku (sku),
    INDEX idx_barcode (barcode)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

**Migration File:** `2025_11_28_000004_create_products_table.php`

**Model:** `App\Models\Product`

```php
class Product extends Model
{
    protected $fillable = [
        'name', 'sku', 'description', 'category_id', 'supplier_id',
        'price', 'cost', 'min_stock', 'max_stock', 'reorder_point',
        'unit', 'track_batch', 'track_expiry', 'image', 'barcode', 'status'
    ];
    
    protected $casts = [
        'price' => 'decimal:2',
        'cost' => 'decimal:2',
        'min_stock' => 'integer',
        'max_stock' => 'integer',
        'reorder_point' => 'integer',
        'track_batch' => 'boolean',
        'track_expiry' => 'boolean'
    ];
    
    // Relationship: Product belongs to one category
    public function category()
    {
        return $this->belongsTo(Category::class);
    }
    
    // Relationship: Product belongs to one supplier
    public function supplier()
    {
        return $this->belongsTo(Supplier::class);
    }
}
```

**Foreign Keys:**
- `category_id`: References `categories.id` (ON DELETE SET NULL)
- `supplier_id`: References `suppliers.id` (ON DELETE SET NULL)

**Unique Constraints:**
- `sku`: Stock Keeping Unit harus unique

**Business Logic:**
- `track_batch`: Flag untuk tracking nomor batch
- `track_expiry`: Flag untuk tracking tanggal kadaluarsa
- `min_stock`, `max_stock`, `reorder_point`: Untuk inventory control

---

### Database Relationships Diagram

```
┌─────────────┐
│  Categories │
│    (1)      │
└──────┬──────┘
       │
       │ has many
       │
       ▼
┌─────────────┐         ┌─────────────┐
│  Products   │ ◄────── │  Suppliers  │
│    (N)      │ belongs │    (1)      │
└─────────────┘   to    └─────────────┘

Legend:
1 = One
N = Many
◄─ = belongs to
─► = has many
```

**Relational Integrity Rules:**

1. **Category → Product**: One-to-Many
   - Satu category bisa memiliki banyak products
   - Satu product hanya bisa belong to satu category
   - ON DELETE SET NULL: Jika category dihapus, product.category_id = NULL

2. **Supplier → Product**: One-to-Many
   - Satu supplier bisa supply banyak products
   - Satu product hanya dari satu supplier
   - ON DELETE SET NULL: Jika supplier dihapus, product.supplier_id = NULL

3. **Warehouse → Product** (Future): Many-to-Many via `warehouse_stock`
   - Satu warehouse bisa menyimpan banyak products
   - Satu product bisa ada di banyak warehouses
   - Pivot table akan menyimpan quantity per warehouse

---

## 🏗 Application Architecture

### MVC Pattern Implementation

```
┌───────────────────────────────────────────────────────┐
│                    HTTP REQUEST                       │
└────────────────────┬──────────────────────────────────┘
                     │
                     ▼
┌───────────────────────────────────────────────────────┐
│                   ROUTES (web.php)                    │
│  Route::get('/products', [ProductController, 'index'])│
└────────────────────┬──────────────────────────────────┘
                     │
                     ▼
┌───────────────────────────────────────────────────────┐
│                   MIDDLEWARE                          │
│  - Authentication (auth)                              │
│  - CSRF Protection                                    │
└────────────────────┬──────────────────────────────────┘
                     │
                     ▼
┌───────────────────────────────────────────────────────┐
│                   CONTROLLER                          │
│  ProductController@index()                            │
│  - Handle request                                     │
│  - Call Model                                         │
│  - Return View                                        │
└────────────────────┬──────────────────────────────────┘
                     │
                     ▼
┌───────────────────────────────────────────────────────┐
│                   MODEL (Eloquent)                    │
│  Product::with(['category','supplier'])->get()        │
│  - Database query                                     │
│  - Data manipulation                                  │
└────────────────────┬──────────────────────────────────┘
                     │
                     ▼
┌───────────────────────────────────────────────────────┐
│                   DATABASE (MySQL)                    │
│  SELECT * FROM products                               │
│  JOIN categories ON...                                │
│  JOIN suppliers ON...                                 │
└────────────────────┬──────────────────────────────────┘
                     │
                     ▼ (return data)
┌───────────────────────────────────────────────────────┐
│                   CONTROLLER                          │
│  return view('products.index', compact('products'))   │
└────────────────────┬──────────────────────────────────┘
                     │
                     ▼
┌───────────────────────────────────────────────────────┐
│                   VIEW (Blade)                        │
│  products/index.blade.php                             │
│  - Render HTML                                        │
│  - Display data                                       │
└────────────────────┬──────────────────────────────────┘
                     │
                     ▼
┌───────────────────────────────────────────────────────┐
│                   HTTP RESPONSE                       │
│  HTML Page to Browser                                 │
└───────────────────────────────────────────────────────┘
```

---

## 🔄 Detailed System Flows

### 1. Authentication Flow

```
┌──────────┐
│  User    │
└────┬─────┘
     │
     │ 1. Navigate to /login
     ▼
┌─────────────────┐
│  Login Page     │
│  (Blade View)   │
└────┬────────────┘
     │
     │ 2. Submit credentials
     │    POST /login
     ▼
┌──────────────────────────┐
│  LoginController@login   │
│  1. Validate input       │
│  2. Attempt auth         │
│  3. Create session       │
└────┬─────────────────────┘
     │
     ├─ Success
     │  │
     │  ▼
     │ ┌──────────────────┐
     │ │  Redirect to     │
     │ │  /dashboard      │
     │ └──────────────────┘
     │
     └─ Failed
        │
        ▼
       ┌──────────────────┐
       │  Back to /login  │
       │  with errors     │
       └──────────────────┘
```

**Implementation Code:**

```php
// routes/web.php
Route::get('/login', [LoginController::class, 'showLoginForm'])
    ->name('login');
Route::post('/login', [LoginController::class, 'login']);

// LoginController.php
public function login(Request $request)
{
    $credentials = $request->validate([
        'email' => 'required|email',
        'password' => 'required'
    ]);
    
    if (Auth::attempt($credentials, $request->filled('remember'))) {
        $request->session()->regenerate();
        return redirect()->intended('dashboard');
    }
    
    return back()->withErrors([
        'email' => 'The provided credentials do not match our records.',
    ]);
}
```

---

### 2. Product CRUD Flow

#### A. Create Product Flow

```
User Action: Click "Add Product"
     │
     ▼
Route: GET /products/create
     │
     ▼
┌────────────────────────────────┐
│ ProductController@create()     │
│ 1. Create empty Product model  │
│ 2. Load Categories from DB     │
│ 3. Load Suppliers from DB      │
└────────┬───────────────────────┘
         │
         ▼
┌────────────────────────────────┐
│ Return view('products.form')   │
│ with: $product, $categories,   │
│       $suppliers               │
└────────┬───────────────────────┘
         │
         ▼
┌────────────────────────────────┐
│ User fills form & clicks Save  │
│ POST /products                 │
└────────┬───────────────────────┘
         │
         ▼
┌────────────────────────────────────┐
│ ProductController@store()          │
│ 1. Validate input data             │
│ 2. Product::create($validated)     │
│ 3. Insert to database             │
│ 4. Redirect with success message  │
└────────┬───────────────────────────┘
         │
         ▼
┌────────────────────────────────┐
│ Redirect to /products          │
│ Show success notification      │
└────────────────────────────────┘
```

**Implementation Code:**

```php
// routes/web.php
Route::resource('products', ProductController::class);

// ProductController.php
public function create()
{
    $product = new Product();
    $categories = Category::where('status', 'active')->get();
    $suppliers = Supplier::where('status', 'active')->get();
    
    return view('products.form', compact('product', 'categories', 'suppliers'));
}

public function store(Request $request)
{
    $validated = $request->validate([
        'name' => 'required|string|max:255',
        'sku' => 'required|string|max:255|unique:products,sku',
        'description' => 'nullable|string',
        'category_id' => 'nullable|exists:categories,id',
        'supplier_id' => 'nullable|exists:suppliers,id',
        'price' => 'required|numeric|min:0',
        'cost' => 'nullable|numeric|min:0',
        'min_stock' => 'nullable|integer|min:0',
        'max_stock' => 'nullable|integer|min:0',
        'reorder_point' => 'nullable|integer|min:0',
        'unit' => 'required|string|max:50',
        'track_batch' => 'boolean',
        'track_expiry' => 'boolean',
        'barcode' => 'nullable|string|max:255',
        'status' => 'required|in:active,inactive',
    ]);
    
    Product::create($validated);
    
    return redirect()->route('products.index')
        ->with('success', 'Product created successfully');
}
```

#### B. Read/List Products Flow

```
User Action: Click "Products" menu
     │
     ▼
Route: GET /products
     │
     ▼
┌──────────────────────────────────────┐
│ ProductController@index()            │
│ 1. Build query with relationships    │
│ 2. Apply search filter (if any)      │
│ 3. Apply category filter (if any)    │
│ 4. Apply supplier filter (if any)    │
│ 5. Paginate results (15 per page)    │
└────────┬─────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────┐
│ Eloquent Query Execution             │
│ Product::with(['category','supplier'])│
│   ->where('name', 'like', "%search%")│
│   ->latest()                         │
│   ->paginate(15)                     │
└────────┬─────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────┐
│ Database Query (MySQL)               │
│ SELECT products.*, categories.name,  │
│        suppliers.name                │
│ FROM products                        │
│ LEFT JOIN categories ON...          │
│ LEFT JOIN suppliers ON...            │
│ WHERE products.name LIKE '%..%'      │
│ ORDER BY created_at DESC             │
│ LIMIT 15 OFFSET 0                    │
└────────┬─────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────┐
│ Return view('products.index')        │
│ with: $products (paginated)          │
│       $categories (for filter)       │
│       $suppliers (for filter)        │
└────────┬─────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────┐
│ Blade Template renders table         │
│ - Loop through $products             │
│ - Display pagination links           │
│ - Show action buttons (edit/delete)  │
└──────────────────────────────────────┘
```

**Implementation Code:**

```php
// ProductController.php
public function index(Request $request)
{
    $query = Product::with(['category', 'supplier']);
    
    // Search filter
    if ($search = $request->get('search')) {
        $query->where(function($q) use ($search) {
            $q->where('name', 'like', "%{$search}%")
              ->orWhere('sku', 'like', "%{$search}%")
              ->orWhere('barcode', 'like', "%{$search}%");
        });
    }
    
    // Category filter
    if ($request->filled('category')) {
        $query->where('category_id', $request->category);
    }
    
    // Supplier filter
    if ($request->filled('supplier')) {
        $query->where('supplier_id', $request->supplier);
    }
    
    // Status filter
    if ($request->filled('status')) {
        $query->where('status', $request->status);
    }
    
    $products = $query->latest()->paginate(15);
    $categories = Category::where('status', 'active')->get();
    $suppliers = Supplier::where('status', 'active')->get();
    
    return view('products.index', compact('products', 'categories', 'suppliers'));
}
```

#### C. Update Product Flow

```
User Action: Click Edit icon
     │
     ▼
Route: GET /products/{id}/edit
     │
     ▼
┌────────────────────────────────┐
│ ProductController@edit($id)    │
│ 1. Find Product by ID          │
│ 2. Load Categories             │
│ 3. Load Suppliers              │
└────────┬───────────────────────┘
         │
         ▼
┌────────────────────────────────┐
│ Return view('products.form')   │
│ with: $product (existing data) │
│       $categories, $suppliers  │
└────────┬───────────────────────┘
         │
         ▼
┌────────────────────────────────┐
│ User edits form & clicks Update│
│ PUT /products/{id}             │
└────────┬───────────────────────┘
         │
         ▼
┌──────────────────────────────────────┐
│ ProductController@update($id)        │
│ 1. Validate input data               │
│ 2. Find Product by ID                │
│ 3. $product->update($validated)      │
│ 4. Update in database               │
│ 5. Redirect with success message    │
└────────┬─────────────────────────────┘
         │
         ▼
┌────────────────────────────────┐
│ Redirect to /products          │
│ Show success notification      │
└────────────────────────────────┘
```

**Implementation Code:**

```php
// ProductController.php
public function edit(Product $product)
{
    $categories = Category::where('status', 'active')->get();
    $suppliers = Supplier::where('status', 'active')->get();
    
    return view('products.form', compact('product', 'categories', 'suppliers'));
}

public function update(Request $request, Product $product)
{
    $validated = $request->validate([
        'name' => 'required|string|max:255',
        'sku' => 'required|string|max:255|unique:products,sku,' . $product->id,
        // ... validation rules sama seperti store
    ]);
    
    $product->update($validated);
    
    return redirect()->route('products.index')
        ->with('success', 'Product updated successfully');
}
```

#### D. Delete Product Flow

```
User Action: Click Delete icon
     │
     ▼
JavaScript: Show confirmation dialog
     │
     ├─ User cancels → No action
     │
     └─ User confirms
        │
        ▼
   Submit DELETE form
   DELETE /products/{id}
        │
        ▼
┌────────────────────────────────┐
│ ProductController@destroy($id) │
│ 1. Find Product by ID          │
│ 2. $product->delete()          │
│ 3. Remove from database        │
└────────┬───────────────────────┘
         │
         ▼
┌────────────────────────────────┐
│ Redirect to /products          │
│ Show success notification      │
└────────────────────────────────┘
```

**Implementation Code:**

```php
// ProductController.php
public function destroy(Product $product)
{
    $product->delete();
    
    return redirect()->route('products.index')
        ->with('success', 'Product deleted successfully');
}
```

```javascript
// Blade template
<button onclick="confirmAction('Are you sure?', () => 
    document.getElementById('delete-form-{{ $product->id }}').submit()
)" class="btn btn-sm btn-ghost text-danger">
    <i class="fas fa-trash"></i>
</button>

<form id="delete-form-{{ $product->id }}" 
      action="{{ route('products.destroy', $product->id) }}" 
      method="POST" style="display: none;">
    @csrf
    @method('DELETE')
</form>
```

---

### 3. Search & Filter Flow

```
User types in search box
     │
     ▼
User clicks "Search" or presses Enter
     │
     ▼
Form submits: GET /products?search=laptop
     │
     ▼
┌────────────────────────────────────┐
│ ProductController@index()          │
│ 1. Get 'search' from request       │
│ 2. Build WHERE clause              │
│    WHERE name LIKE '%laptop%'      │
│    OR sku LIKE '%laptop%'          │
│    OR barcode LIKE '%laptop%'      │
└────────┬───────────────────────────┘
         │
         ▼
┌────────────────────────────────────┐
│ Execute query & return results     │
│ Only products matching search      │
└────────────────────────────────────┘
```

**Implementation Code:**

```php
// Blade template - Search form
<form action="{{ route('products.index') }}" method="GET">
    <input type="text" 
           name="search" 
           value="{{ request('search') }}"
           placeholder="Search...">
    <button type="submit">Search</button>
    <a href="{{ route('products.index') }}">Clear</a>
</form>

// Controller - Search logic
if ($search = $request->get('search')) {
    $query->where(function($q) use ($search) {
        $q->where('name', 'like', "%{$search}%")
          ->orWhere('sku', 'like', "%{$search}%")
          ->orWhere('barcode', 'like', "%{$search}%");
    });
}
```

---

## 📋 Controller Implementation Details

### CategoryController

```php
class CategoryController extends Controller
{
    // LIST: Show all categories with pagination
    public function index(Request $request)
    {
        $query = Category::withCount('products');
        
        if ($search = $request->get('search')) {
            $query->where(function($q) use ($search) {
                $q->where('name', 'like', "%{$search}%")
                  ->orWhere('description', 'like', "%{$search}%");
            });
        }
        
        $categories = $query->latest()->paginate(15);
        return view('categories.index', compact('categories'));
    }
    
    // CREATE: Show form for new category
    public function create()
    {
        $category = new Category();
        return view('categories.form', compact('category'));
    }
    
    // STORE: Save new category to database
    public function store(Request $request)
    {
        $validated = $request->validate([
            'name' => 'required|string|max:255',
            'description' => 'nullable|string',
            'status' => 'required|in:active,inactive',
        ]);
        
        Category::create($validated);
        
        return redirect()->route('categories.index')
            ->with('success', 'Category created successfully');
    }
    
    // EDIT: Show form to edit existing category
    public function edit(Category $category)
    {
        return view('categories.form', compact('category'));
    }
    
    // UPDATE: Save changes to database
    public function update(Request $request, Category $category)
    {
        $validated = $request->validate([
            'name' => 'required|string|max:255',
            'description' => 'nullable|string',
            'status' => 'required|in:active,inactive',
        ]);
        
        $category->update($validated);
        
        return redirect()->route('categories.index')
            ->with('success', 'Category updated successfully');
    }
    
    // DELETE: Remove category from database
    public function destroy(Category $category)
    {
        $category->delete();
        
        return redirect()->route('categories.index')
            ->with('success', 'Category deleted successfully');
    }
}
```

**Key Features:**
- **Route Model Binding**: Laravel automatically finds Category by ID
- **withCount('products')**: Eager load product count for each category
- **Validation**: Server-side validation untuk setiap input
- **Flash Messages**: Success/error messages via session

---

## 🎨 View/Blade Templates Implementation

### Blade Layout Structure

```
layouts/app.blade.php (Master Layout)
    ├── Header
    │   ├── Logo
    │   ├── Navigation Menu
    │   └── User Dropdown
    ├── Sidebar
    │   ├── Dashboard
    │   ├── Products
    │   ├── Categories
    │   ├── Suppliers
    │   ├── Warehouses
    │   └── Customers
    ├── Main Content (@yield('content'))
    └── Footer
```

**Example Blade Template:**

```blade
{{-- products/index.blade.php --}}
@extends('layouts.app')

@section('title', 'Products')

@section('content')
<!-- Breadcrumb -->
<nav class="breadcrumb">
    <div class="breadcrumb-item">
        <a href="{{ route('dashboard') }}">Dashboard</a>
    </div>
    <span>/</span>
    <div class="breadcrumb-item active">Products</div>
</nav>

<!-- Page Header -->
<div class="page-header">
    <h1>Products</h1>
    <a href="{{ route('products.create') }}" class="btn btn-primary">
        Add Product
    </a>
</div>

<!-- Search & Filters -->
<div class="card">
    <form method="GET">
        <input type="text" name="search" value="{{ request('search') }}">
        <button type="submit">Search</button>
    </form>
</div>

<!-- Products Table -->
<div class="card">
    <table class="table">
        <thead>
            <tr>
                <th>Name</th>
                <th>SKU</th>
                <th>Category</th>
                <th>Price</th>
                <th>Actions</th>
            </tr>
        </thead>
        <tbody>
            @forelse($products as $product)
            <tr>
                <td>{{ $product->name }}</td>
                <td>{{ $product->sku }}</td>
                <td>{{ $product->category->name ?? '-' }}</td>
                <td>Rp {{ number_format($product->price, 0, ',', '.') }}</td>
                <td>
                    <a href="{{ route('products.edit', $product) }}">Edit</a>
                    <form method="POST" action="{{ route('products.destroy', $product) }}">
                        @csrf
                        @method('DELETE')
                        <button type="submit">Delete</button>
                    </form>
                </td>
            </tr>
            @empty
            <tr>
                <td colspan="5">No products found</td>
            </tr>
            @endforelse
        </tbody>
    </table>
    
    <!-- Pagination -->
    {{ $products->links() }}
</div>
@endsection
```

---

## 🔐 Security Implementation

### 1. CSRF Protection

Laravel automatically protects against CSRF attacks:

```blade
<form method="POST" action="{{ route('products.store') }}">
    @csrf  {{-- Laravel generates CSRF token --}}
    <!-- form fields -->
</form>
```

### 2. SQL Injection Prevention

Eloquent ORM menggunakan prepared statements:

```php
// SAFE - Using Eloquent
Product::where('name', $userInput)->get();

// SAFE - Using Query Builder with bindings
DB::select('SELECT * FROM products WHERE name = ?', [$userInput]);

// UNSAFE - Direct concatenation (NEVER DO THIS)
// DB::select("SELECT * FROM products WHERE name = '$userInput'");
```

### 3. Mass Assignment Protection

Model menggunakan `$fillable` untuk whitelist:

```php
class Product extends Model
{
    protected $fillable = ['name', 'sku', 'price'];
    
    // Only these fields can be mass-assigned
    // Prevents injection of unauthorized fields
}
```

### 4. Validation

Server-side validation untuk semua input:

```php
$validated = $request->validate([
    'email' => 'required|email|max:255',
    'price' => 'required|numeric|min:0',
    'sku' => 'required|unique:products,sku'
]);
```

---

## 📦 Data Seeding Implementation

### Seeder Flow

```
php artisan db:seed
     │
     ▼
DatabaseSeeder::run()
     │
     ├─► UserSeeder::run()
     │        └─► Insert admin user
     │
     ├─► CategorySeeder::run()
     │        └─► Insert 7 categories
     │
     ├─► SupplierSeeder::run()
     │        └─► Insert 7 suppliers
     │
     ├─► CustomerSeeder::run()
     │        └─► Insert 8 customers
     │
     ├─► WarehouseSeeder::run()
     │        └─► Insert 5 warehouses
     │
     └─► ProductSeeder::run()
              └─► Insert 12 products with relationships
```

**Example Seeder:**

```php
class ProductSeeder extends Seeder
{
    public function run(): void
    {
        // Get related data first
        $elektronik = Category::where('name', 'Elektronik')->first();
        $supplier = Supplier::where('name', 'PT Elektronik Jaya')->first();
        
        // Create products with relationships
        Product::create([
            'name' => 'Laptop ASUS ROG',
            'sku' => 'ELK-001',
            'description' => 'Gaming laptop',
            'category_id' => $elektronik->id,
            'supplier_id' => $supplier->id,
            'price' => 15000000,
            'cost' => 13000000,
            'min_stock' => 5,
            'max_stock' => 50,
            'reorder_point' => 10,
            'unit' => 'pcs',
            'barcode' => '1234567890001',
            'status' => 'active'
        ]);
    }
}
```

---

## 🚦 Routing Implementation

### Route Structure

```php
// routes/web.php

// Guest routes
Route::middleware('guest')->group(function () {
    Route::get('/login', [LoginController::class, 'showLoginForm'])
        ->name('login');
    Route::post('/login', [LoginController::class, 'login']);
});

// Authenticated routes
Route::middleware('auth')->group(function () {
    // Dashboard
    Route::get('/dashboard', [DashboardController::class, 'index'])
        ->name('dashboard');
    
    // Resource routes (auto-generates 7 routes for CRUD)
    Route::resource('products', ProductController::class);
    Route::resource('categories', CategoryController::class);
    Route::resource('suppliers', SupplierController::class);
    Route::resource('warehouses', WarehouseController::class);
    Route::resource('customers', CustomerController::class);
    
    // Logout
    Route::post('/logout', [LoginController::class, 'logout'])
        ->name('logout');
});
```

**Resource Route auto-generates:**

| Verb | URI | Action | Route Name |
|------|-----|--------|------------|
| GET | /products | index | products.index |
| GET | /products/create | create | products.create |
| POST | /products | store | products.store |
| GET | /products/{id} | show | products.show |
| GET | /products/{id}/edit | edit | products.edit |
| PUT/PATCH | /products/{id} | update | products.update |
| DELETE | /products/{id} | destroy | products.destroy |

---

## 🎯 Best Practices Implemented

### 1. **DRY (Don't Repeat Yourself)**
- Reusable Blade components
- Shared layouts
- Helper functions

### 2. **Single Responsibility**
- Controllers hanya handle HTTP requests
- Models hanya untuk data logic
- Views hanya untuk presentation

### 3. **Naming Conventions**
- Controllers: PascalCase + "Controller" suffix
- Models: Singular PascalCase
- Tables: Plural snake_case
- Routes: kebab-case

### 4. **Error Handling**
```php
try {
    Product::create($validated);
    return redirect()->back()->with('success', 'Created!');
} catch (\Exception $e) {
    \Log::error($e->getMessage());
    return redirect()->back()->with('error', 'Failed to create product');
}
```

### 5. **Eager Loading**
```php
// Good: Prevents N+1 query problem
$products = Product::with(['category', 'supplier'])->get();

// Bad: Causes N+1 queries
$products = Product::all();
foreach ($products as $product) {
    echo $product->category->name; // Triggers separate query each time
}
```

---

## 📈 Performance Optimization

### 1. Database Indexing

```sql
-- Indexes untuk search performance
INDEX idx_products_name ON products(name)
INDEX idx_products_sku ON products(sku)
INDEX idx_products_status ON products(status)
INDEX idx_categories_name ON categories(name)
```

### 2. Pagination

```php
// Load only 15 items per page, not all data
$products = Product::paginate(15);
```

### 3. Query Optimization

```php
// Select only needed columns
Product::select('id', 'name', 'sku', 'price')->get();

// Use chunk for large datasets
Product::chunk(100, function ($products) {
    foreach ($products as $product) {
        // Process
    }
});
```

---

## 🧪 Testing Strategy

### Unit Tests Example

```php
class ProductTest extends TestCase
{
    public function test_can_create_product()
    {
        $product = Product::factory()->create([
            'name' => 'Test Product',
            'sku' => 'TEST-001'
        ]);
        
        $this->assertDatabaseHas('products', [
            'name' => 'Test Product',
            'sku' => 'TEST-001'
        ]);
    }
    
    public function test_product_belongs_to_category()
    {
        $category = Category::factory()->create();
        $product = Product::factory()->create([
            'category_id' => $category->id
        ]);
        
        $this->assertEquals($category->id, $product->category->id);
    }
}
```

---

## 📞 API Documentation (Future)

Untuk implementasi REST API di masa depan:

```php
// routes/api.php
Route::group(['prefix' => 'api/v1'], function () {
    Route::apiResource('products', ProductApiController::class);
});

// Response format
{
    "status": "success",
    "data": {
        "id": 1,
        "name": "Laptop",
        "sku": "ELK-001",
        "price": 15000000
    }
}
```

---

## 🎓 Kesimpulan

Sistem Manajemen Gudang ini dibangun dengan:
- **Arsitektur MVC** yang clean dan maintainable
- **Database normalisasi** untuk integritas data
- **Eloquent ORM** untuk query yang aman
- **Blade templating** untuk views yang reusable
- **Validation** di semua layer
- **Security** best practices
- **Performance optimization** dengan indexing dan pagination

Dokumentasi ini memberikan panduan lengkap untuk memahami implementasi teknis sistem dari database hingga presentasi layer.
