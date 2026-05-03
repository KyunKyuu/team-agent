---
name: php-routes
language: php
layer: routing
---

# PHP Routes — Laravel

## Key Patterns

- All API routes live in `routes/api.php`; automatically prefixed with `/api`
- Version routes with a prefix group (`/v1`, `/v2`) to allow non-breaking evolution
- Apply middleware groups at the group level, not on individual routes
- Use `Route::apiResource()` to register the five standard RESTful routes in one line
- Name every route (`->name('policies.store')`) for `route()` helper use and test assertions

## Example

```php
// routes/api.php
use App\Http\Controllers\Api\V1\PolicyController;
use App\Http\Controllers\Api\V1\ClaimController;
use App\Http\Controllers\Api\V1\AuthController;

// Public routes
Route::prefix('v1')->name('v1.')->group(function () {

    Route::post('/auth/login',   [AuthController::class, 'login'])->name('auth.login');
    Route::post('/auth/refresh', [AuthController::class, 'refresh'])->name('auth.refresh');

    // Protected routes
    Route::middleware(['auth:sanctum', 'throttle:api'])->group(function () {

        Route::apiResource('policies', PolicyController::class);

        Route::prefix('policies/{policy}')->name('policies.')->group(function () {
            Route::apiResource('claims', ClaimController::class)->shallow();
        });

    });
});

// Internal / service-to-service routes (API key auth)
Route::prefix('internal/v1')->name('internal.v1.')
    ->middleware('auth.apikey')
    ->group(function () {
        Route::get('/policies/{id}/status', [PolicyController::class, 'status'])
             ->name('policies.status');
    });
```

## Checklist

- [ ] All routes in `routes/api.php`; no API routes in `routes/web.php`
- [ ] Version prefix applied to every group (`v1`, `v2`)
- [ ] Auth middleware applied at group level, not repeated on each route
- [ ] `Route::apiResource()` used for standard CRUD instead of five individual `Route::` calls
- [ ] Every route has a name for testability and `route()` helper usage
- [ ] Rate limiting configured via named throttle (`throttle:api`) defined in `RouteServiceProvider`
- [ ] `php artisan route:list` output reviewed after changes to catch conflicts or missing middleware

---
> Customize for your Laravel version and auth package (Sanctum / Passport / tymon-jwt).
