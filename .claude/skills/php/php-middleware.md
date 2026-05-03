---
name: php-middleware
language: php
layer: middleware
---

# PHP Middleware — Laravel

## Key Patterns

- JWT auth via Sanctum (`auth:sanctum`), Passport (`auth:api`), or tymon-jwt (`jwt.auth`) — pick one per project
- API key middleware reads `X-Api-Key` header and aborts with 401 on mismatch
- Register custom middleware in `app/Http/Kernel.php` under `$routeMiddleware`
- Use `$middlewareGroups` (`api`, `web`) for stacks applied to entire route files
- Throttle via named rate limiters defined in `RouteServiceProvider::configureRateLimiting()`

## Example

```php
// app/Http/Middleware/ApiKeyMiddleware.php
namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class ApiKeyMiddleware
{
    public function handle(Request $request, Closure $next): Response
    {
        $key = $request->header('X-Api-Key');

        if (empty($key) || $key !== config('services.internal_api.key')) {
            return response()->json(['message' => 'Unauthorized'], 401);
        }

        return $next($request);
    }
}

// app/Http/Kernel.php  — register alias
protected $routeMiddleware = [
    'auth.apikey' => \App\Http\Middleware\ApiKeyMiddleware::class,
    'jwt.auth'    => \Tymon\JWTAuth\Http\Middleware\Authenticate::class,   // tymon-jwt
];

// app/Providers/RouteServiceProvider.php  — named rate limiter
protected function configureRateLimiting(): void
{
    RateLimiter::for('api', function (Request $request) {
        return Limit::perMinute(120)->by($request->user()?->id ?: $request->ip());
    });
}
```

## Checklist

- [ ] Auth middleware alias matches the chosen package (`auth:sanctum` / `auth:api` / `jwt.auth`)
- [ ] API key read from `config()`, never hardcoded in the middleware class
- [ ] Custom middleware registered in `Kernel::$routeMiddleware` with a short alias
- [ ] Throttle rate limiter defined by name in `RouteServiceProvider`, applied by alias in routes
- [ ] Middleware returns `Response` — typed return hint helps catch missing `return $next($request)`
- [ ] `php artisan route:list --columns=middleware` used to verify middleware is applied correctly
- [ ] Token refresh / expiry handled gracefully (401 with `message` key matching API contract)

---
> Customize for your Laravel version and auth package (Sanctum / Passport / tymon-jwt).
