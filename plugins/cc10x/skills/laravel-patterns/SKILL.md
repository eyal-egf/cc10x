---
name: laravel-patterns
description: "Laravel PHP patterns: Eloquent, middleware, validation, testing. Complementary skill invoked via CLAUDE.md skill table."
allowed-tools: Read, Grep, Glob, Bash, LSP
---

# Laravel Patterns

## Overview

Laravel applications succeed when they embrace the framework's conventions. Fight the framework and you fight yourself. Every architectural decision should leverage Laravel's built-in tools before reaching for custom solutions.

**Core principle:** Convention over configuration. Use what Laravel gives you before building your own.

**Violating the letter of this process is violating the spirit of Laravel development.**

## Focus Areas (Reference Pattern)

- **Eloquent ORM** (relationships, scopes, accessors, query optimization)
- **Request lifecycle** (middleware, Form Requests, controllers, responses)
- **Database** (migrations, seeders, factories, query builder)
- **Testing** (Pest/PHPUnit, feature tests, unit tests)
- **Security** (CSRF, mass assignment, SQL injection, authorization)
- **Queues & Jobs** (async processing, retries, batches)
- **API Resources** (JSON responses, pagination, versioning)

## The Iron Law

```
NO DATABASE QUERY WITHOUT CONSIDERING THE N+1 PROBLEM
```

Every Eloquent relationship access in a loop is a potential N+1. If you haven't checked eager loading, you haven't finished the feature.

## When to Use

**Always when working with:**
- Laravel applications (any version, focus on 10+/11+)
- Eloquent models and relationships
- Laravel API development
- Blade templates or Livewire
- Artisan commands
- Laravel testing (Pest or PHPUnit)

## Universal Questions (Answer First)

1. **Does Laravel have this built-in?** Check framework features before building custom.
2. **Is this an N+1?** Any relationship access in a loop needs `with()`.
3. **Is user input validated?** All input must go through Form Requests or inline validation.
4. **Is this mass-assignable?** Check `$fillable` or `$guarded` on every model.
5. **Should this be async?** Long operations (email, file processing, API calls) belong in queues.

## Eloquent Patterns

### Model Definition

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Casts\Attribute;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\Relations\HasMany;
use Illuminate\Database\Eloquent\Builder;

class Order extends Model
{
    use HasFactory;

    protected $fillable = [
        'user_id',
        'status',
        'total_amount',
        'notes',
    ];

    protected $casts = [
        'total_amount' => 'decimal:2',
        'metadata' => 'array',
        'shipped_at' => 'datetime',
    ];

    // Relationships - always type-hint return
    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }

    public function items(): HasMany
    {
        return $this->hasMany(OrderItem::class);
    }

    // Scopes - prefix with 'scope'
    public function scopePending(Builder $query): Builder
    {
        return $query->where('status', 'pending');
    }

    public function scopeForUser(Builder $query, int $userId): Builder
    {
        return $query->where('user_id', $userId);
    }

    // Accessors (Laravel 10+ attribute casting)
    protected function formattedTotal(): Attribute
    {
        return Attribute::make(
            get: fn () => '$' . number_format($this->total_amount, 2),
        );
    }
}
```

### Eager Loading (N+1 Prevention)

```php
// BAD: N+1 - each iteration queries the database
$orders = Order::all();
foreach ($orders as $order) {
    echo $order->user->name; // Query per iteration!
}

// GOOD: Eager load
$orders = Order::with('user')->get();
foreach ($orders as $order) {
    echo $order->user->name; // No additional queries
}

// Nested eager loading
$orders = Order::with(['user', 'items.product'])->get();

// Constrained eager loading
$orders = Order::with(['items' => function ($query) {
    $query->where('quantity', '>', 1)->orderBy('price', 'desc');
}])->get();

// Prevent lazy loading in development (catches N+1 automatically)
// In AppServiceProvider::boot()
Model::preventLazyLoading(! $this->app->isProduction());
```

### Query Optimization

```php
// Select only needed columns
$users = User::select('id', 'name', 'email')->get();

// Use chunk for large datasets
User::where('active', true)->chunk(200, function ($users) {
    foreach ($users as $user) {
        // Process in batches of 200
    }
});

// Use cursor for memory efficiency (one row at a time)
foreach (User::where('active', true)->cursor() as $user) {
    // Minimal memory usage
}

// Aggregate without loading models
$count = Order::where('status', 'pending')->count();
$total = Order::where('user_id', $userId)->sum('total_amount');

// Exists check (faster than count > 0)
if (Order::where('user_id', $userId)->exists()) {
    // ...
}
```

## Controller Patterns

### Resource Controller

```php
<?php

namespace App\Http\Controllers;

use App\Http\Requests\StoreOrderRequest;
use App\Http\Requests\UpdateOrderRequest;
use App\Http\Resources\OrderResource;
use App\Models\Order;

class OrderController extends Controller
{
    public function index()
    {
        $orders = Order::with('user')
            ->latest()
            ->paginate(20);

        return OrderResource::collection($orders);
    }

    public function store(StoreOrderRequest $request)
    {
        $order = Order::create($request->validated());

        return new OrderResource($order);
    }

    public function show(Order $order)
    {
        $order->load(['items.product', 'user']);

        return new OrderResource($order);
    }

    public function update(UpdateOrderRequest $request, Order $order)
    {
        $order->update($request->validated());

        return new OrderResource($order);
    }

    public function destroy(Order $order)
    {
        $this->authorize('delete', $order);
        $order->delete();

        return response()->noContent();
    }
}
```

**Controller rules:**
- Keep controllers thin. Business logic belongs in Services or Actions.
- Always use Form Requests for validation (never validate in controllers).
- Use route model binding (`Order $order` in method signature).
- Return API Resources for JSON responses.

## Form Requests (Validation)

```php
<?php

namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Validation\Rule;

class StoreOrderRequest extends FormRequest
{
    public function authorize(): bool
    {
        return true; // Or use policies
    }

    public function rules(): array
    {
        return [
            'items' => ['required', 'array', 'min:1'],
            'items.*.product_id' => ['required', 'exists:products,id'],
            'items.*.quantity' => ['required', 'integer', 'min:1', 'max:100'],
            'notes' => ['nullable', 'string', 'max:500'],
            'shipping_address_id' => [
                'required',
                Rule::exists('addresses', 'id')->where('user_id', $this->user()->id),
            ],
        ];
    }

    public function messages(): array
    {
        return [
            'items.required' => 'At least one item is required.',
            'items.*.product_id.exists' => 'One or more products do not exist.',
        ];
    }
}
```

## API Resources

```php
<?php

namespace App\Http\Resources;

use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\JsonResource;

class OrderResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return [
            'id' => $this->id,
            'status' => $this->status,
            'total_amount' => $this->total_amount,
            'formatted_total' => $this->formatted_total,
            'user' => new UserResource($this->whenLoaded('user')),
            'items' => OrderItemResource::collection($this->whenLoaded('items')),
            'created_at' => $this->created_at->toISOString(),
            'updated_at' => $this->updated_at->toISOString(),
        ];
    }
}
```

**Resource rules:**
- Use `whenLoaded()` for relationships (avoids N+1 in resources)
- Never expose internal IDs or sensitive fields
- Use ISO 8601 for dates
- Create separate resources for list vs detail views if needed

## Middleware Patterns

```php
<?php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class EnsureUserIsSubscribed
{
    public function handle(Request $request, Closure $next): Response
    {
        if (! $request->user()?->subscribed()) {
            return response()->json([
                'message' => 'Subscription required.',
            ], 403);
        }

        return $next($request);
    }
}
```

**Middleware rules:**
- Single responsibility per middleware
- Return early on failure (guard clause pattern)
- Register in `bootstrap/app.php` (Laravel 11+) or `Kernel.php` (Laravel 10)

## Database Migrations

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('orders', function (Blueprint $table) {
            $table->id();
            $table->foreignId('user_id')->constrained()->cascadeOnDelete();
            $table->string('status')->default('pending')->index();
            $table->decimal('total_amount', 10, 2);
            $table->json('metadata')->nullable();
            $table->timestamp('shipped_at')->nullable();
            $table->timestamps();
            $table->softDeletes();

            // Composite index for common queries
            $table->index(['user_id', 'status']);
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('orders');
    }
};
```

**Migration rules:**
- Always add indexes for columns used in `WHERE`, `ORDER BY`, `JOIN`
- Use `foreignId()->constrained()` for foreign keys
- Add `down()` method for rollback capability
- Use `decimal` for money (never `float`)
- Add `softDeletes()` for data you might need to recover

## Testing Patterns

### Feature Test (Pest)

```php
<?php

use App\Models\Order;
use App\Models\User;

test('user can list their orders', function () {
    $user = User::factory()->create();
    Order::factory()->count(3)->for($user)->create();
    Order::factory()->count(2)->create(); // Other user's orders

    $response = $this->actingAs($user)
        ->getJson('/api/orders');

    $response
        ->assertOk()
        ->assertJsonCount(3, 'data');
});

test('user can create an order', function () {
    $user = User::factory()->create();
    $product = Product::factory()->create(['price' => 29.99]);

    $response = $this->actingAs($user)
        ->postJson('/api/orders', [
            'items' => [
                ['product_id' => $product->id, 'quantity' => 2],
            ],
        ]);

    $response->assertCreated();
    expect(Order::count())->toBe(1);
    expect(Order::first()->total_amount)->toBe(59.98);
});

test('unauthenticated users cannot create orders', function () {
    $this->postJson('/api/orders', [])
        ->assertUnauthorized();
});

test('validation rejects invalid order data', function () {
    $user = User::factory()->create();

    $this->actingAs($user)
        ->postJson('/api/orders', [
            'items' => [],
        ])
        ->assertUnprocessable()
        ->assertJsonValidationErrors(['items']);
});
```

### Unit Test

```php
<?php

use App\Models\Order;

test('formatted total returns currency string', function () {
    $order = Order::factory()->make(['total_amount' => 1234.50]);

    expect($order->formatted_total)->toBe('$1,234.50');
});

test('pending scope returns only pending orders', function () {
    Order::factory()->create(['status' => 'pending']);
    Order::factory()->create(['status' => 'completed']);

    expect(Order::pending()->count())->toBe(1);
});
```

**Testing rules:**
- Use `php artisan test` or `./vendor/bin/pest` (not `npm test`)
- Use factories for test data (never manual inserts)
- Test HTTP layer (feature tests) over internal methods (unit tests)
- Use `RefreshDatabase` trait (or Pest's `uses()`)
- Test validation rules explicitly
- Test authorization (who can/cannot access)

**Test commands:**
```bash
# Run all tests
php artisan test

# Run specific test file
php artisan test tests/Feature/OrderTest.php

# Run with filter
php artisan test --filter="user can create"

# Run with coverage
php artisan test --coverage --min=80
```

## Queue & Job Patterns

```php
<?php

namespace App\Jobs;

use App\Models\Order;
use App\Notifications\OrderShippedNotification;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;

class ProcessOrderShipment implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public int $tries = 3;
    public int $backoff = 60; // seconds between retries

    public function __construct(
        public readonly Order $order,
    ) {}

    public function handle(): void
    {
        // Process shipment
        $this->order->update(['status' => 'shipped', 'shipped_at' => now()]);
        $this->order->user->notify(new OrderShippedNotification($this->order));
    }

    public function failed(\Throwable $exception): void
    {
        // Log failure, notify admin, etc.
        logger()->error('Order shipment failed', [
            'order_id' => $this->order->id,
            'error' => $exception->getMessage(),
        ]);
    }
}

// Dispatching
ProcessOrderShipment::dispatch($order);
ProcessOrderShipment::dispatch($order)->onQueue('shipments');
ProcessOrderShipment::dispatch($order)->delay(now()->addMinutes(5));
```

**Queue rules:**
- Anything that takes > 1 second should be queued (emails, file processing, API calls)
- Always define `$tries` and `$backoff`
- Implement `failed()` method for error handling
- Use `readonly` constructor promotion for immutable job data
- Test with `Queue::fake()` in feature tests

## Authorization (Policies)

```php
<?php

namespace App\Policies;

use App\Models\Order;
use App\Models\User;

class OrderPolicy
{
    public function view(User $user, Order $order): bool
    {
        return $user->id === $order->user_id;
    }

    public function update(User $user, Order $order): bool
    {
        return $user->id === $order->user_id && $order->status === 'pending';
    }

    public function delete(User $user, Order $order): bool
    {
        return $user->id === $order->user_id && $order->status === 'pending';
    }
}
```

**Authorization rules:**
- One policy per model
- Use `$this->authorize()` in controllers or `Gate::allows()` in services
- Check authorization before any mutation
- Never rely on frontend-only authorization checks

## Service Pattern (For Complex Business Logic)

```php
<?php

namespace App\Services;

use App\Models\Order;
use App\Models\User;
use App\Jobs\ProcessOrderShipment;
use Illuminate\Support\Facades\DB;

class OrderService
{
    public function createOrder(User $user, array $items): Order
    {
        return DB::transaction(function () use ($user, $items) {
            $order = $user->orders()->create([
                'status' => 'pending',
                'total_amount' => $this->calculateTotal($items),
            ]);

            foreach ($items as $item) {
                $order->items()->create($item);
            }

            return $order->load('items');
        });
    }

    private function calculateTotal(array $items): float
    {
        return collect($items)->sum(fn ($item) =>
            $item['price'] * $item['quantity']
        );
    }
}
```

**Service rules:**
- Wrap multi-step operations in `DB::transaction()`
- Keep services focused on one domain
- Inject dependencies via constructor (Laravel auto-resolves)
- Return models or DTOs, not responses

## Security Checklist

| Check | What to Verify |
|-------|----------------|
| **Mass assignment** | All models have `$fillable` or `$guarded` |
| **CSRF** | All state-changing forms have `@csrf` |
| **SQL injection** | Using Eloquent/Query Builder (not raw SQL with user input) |
| **XSS** | Blade `{{ }}` auto-escapes. Never use `{!! !!}` with user input |
| **Authorization** | Every controller action checks permissions |
| **Rate limiting** | API endpoints have rate limits |
| **Validation** | All user input validated via Form Requests |
| **Sensitive data** | No secrets in `.env.example`, no passwords in logs |

## Red Flags - STOP and Reconsider

If you find yourself:

- Querying inside a loop without eager loading
- Writing raw SQL when Eloquent can do it
- Validating in controllers instead of Form Requests
- Skipping authorization checks
- Using `float` for money
- Putting business logic in controllers (> 20 lines per method)
- Not wrapping multi-model operations in transactions
- Returning Eloquent models directly from API endpoints

**STOP. Use the Laravel way.**

## Rationalization Prevention

| Excuse | Reality |
|--------|---------|
| "Eager loading is premature optimization" | N+1 is a correctness issue, not just performance. |
| "Raw SQL is faster" | Eloquent compiles to the same SQL. Use scopes. |
| "Validation in controller is simpler" | Until you need the same validation elsewhere. Use Form Requests. |
| "We'll add auth later" | Security is not a feature you bolt on. |
| "Transactions are overkill" | One failure in a multi-step operation = corrupted data. |
| "We don't need tests for this" | Every untested path is a bug waiting to happen. |

## Output Format

```markdown
## Laravel Review: [Feature/Component]

### Architecture
- [ ] Thin controllers, logic in services/actions
- [ ] Form Requests for validation
- [ ] API Resources for JSON responses
- [ ] Policies for authorization
- [ ] Jobs for async operations

### Database
- [ ] N+1 prevention (eager loading verified)
- [ ] Proper indexes on queried columns
- [ ] Transactions for multi-model operations
- [ ] Migrations have rollback (down method)

### Security
- [ ] Mass assignment protection ($fillable)
- [ ] Input validation (Form Requests)
- [ ] Authorization checks (policies)
- [ ] No raw SQL with user input

### Issues
| Severity | Issue | Location | Fix |
|----------|-------|----------|-----|
| [BLOCKS/IMPAIRS/MINOR] | [Issue] | `file:line` | [Fix] |

### Recommendations
1. [Most critical]
2. [Second]
```

## Final Check

Before completing Laravel work:

- [ ] N+1 queries prevented (eager loading with `with()`)
- [ ] All user input validated via Form Requests
- [ ] Mass assignment protected (`$fillable` on all models)
- [ ] Authorization checked (policies on all mutations)
- [ ] Multi-step operations wrapped in `DB::transaction()`
- [ ] Long operations dispatched to queues
- [ ] API responses use Resources (not raw models)
- [ ] Migrations have `down()` method
- [ ] Indexes added for queried columns
- [ ] Tests written (`php artisan test` passes)
