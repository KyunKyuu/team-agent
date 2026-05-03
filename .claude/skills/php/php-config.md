---
name: php-config
language: php
layer: config
---

# PHP Config — Laravel

## Key Patterns

- All environment values live in `.env`; never read `env()` outside of `config/` files
- Publish config files under `config/` and access them exclusively via `config('file.key')`
- Validate required env values on boot using a `AppServiceProvider` boot check or a custom `ConfigValidator`
- Use typed accessors (`(int)`, `(bool)`, `explode`) inside config files so callers receive correct types
- Group related settings into a single config file (e.g. `config/services.php`, `config/app.php`)

## Example

```php
// config/payment.php
return [
    'gateway'    => env('PAYMENT_GATEWAY', 'midtrans'),
    'secret_key' => env('PAYMENT_SECRET_KEY'),
    'timeout'    => (int) env('PAYMENT_TIMEOUT', 30),
    'sandbox'    => (bool) env('PAYMENT_SANDBOX', false),
    'currencies' => explode(',', env('PAYMENT_CURRENCIES', 'IDR')),
];

// app/Providers/AppServiceProvider.php
public function boot(): void
{
    $required = ['payment.secret_key', 'services.some_api.key'];
    foreach ($required as $key) {
        if (empty(config($key))) {
            throw new \RuntimeException("Missing required config: {$key}");
        }
    }
}

// Usage anywhere in the app
$timeout = config('payment.timeout');        // int
$sandbox = config('payment.sandbox');        // bool
$gateway = config('payment.gateway', 'midtrans'); // with fallback
```

## Checklist

- [ ] No `env()` calls outside `config/` files
- [ ] All sensitive values (keys, passwords) are in `.env`, not hardcoded
- [ ] `.env.example` updated with every new variable (empty or safe default value)
- [ ] Required config keys validated on boot so the app fails fast on misconfiguration
- [ ] Config values are cast to their correct PHP types inside the config file
- [ ] Config cache (`php artisan config:cache`) works without errors in production
- [ ] New config file registered in `config/` — Laravel auto-discovers it; no manual registration needed

---
> Customize for your Laravel version and auth package (Sanctum / Passport / tymon-jwt).
