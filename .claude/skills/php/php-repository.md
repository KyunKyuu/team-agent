---
name: php-repository
language: php
layer: repository
---

# PHP Repository — Laravel

## Key Patterns

- Define a typed interface; bind the Eloquent implementation in a `ServiceProvider`
- Controllers and Services depend on the interface, never the concrete class
- Repositories handle only persistence (find, save, delete, list); no business logic
- Delegate reusable query constraints to Eloquent local scopes, not inline lambdas
- Return model instances or collections — never raw query builder objects

## Example

```php
// app/Repositories/PolicyRepositoryInterface.php
interface PolicyRepositoryInterface
{
    public function findById(int $id): ?Policy;
    public function findByPolicyNumber(string $number): ?Policy;
    public function listActive(int $perPage = 20): LengthAwarePaginator;
    public function save(Policy $policy): Policy;
    public function delete(int $id): bool;
}

// app/Repositories/EloquentPolicyRepository.php
class EloquentPolicyRepository implements PolicyRepositoryInterface
{
    public function findById(int $id): ?Policy
    {
        return Policy::with('user')->find($id);
    }

    public function findByPolicyNumber(string $number): ?Policy
    {
        return Policy::where('policy_number', $number)->first();
    }

    public function listActive(int $perPage = 20): LengthAwarePaginator
    {
        return Policy::active()->with('user')->latest()->paginate($perPage);
    }

    public function save(Policy $policy): Policy
    {
        $policy->save();
        return $policy->refresh();
    }

    public function delete(int $id): bool
    {
        return Policy::destroy($id) > 0;
    }
}

// app/Providers/RepositoryServiceProvider.php
public function register(): void
{
    $this->app->bind(PolicyRepositoryInterface::class, EloquentPolicyRepository::class);
}
```

## Checklist

- [ ] Interface defined in `app/Repositories/` with return types on every method
- [ ] Eloquent implementation bound to the interface in a dedicated `ServiceProvider`
- [ ] No business rules or HTTP concerns inside the repository
- [ ] Eager loading (`with()`) declared in repository methods, not in controllers
- [ ] Local model scopes used for reusable WHERE clauses instead of inline closures
- [ ] Repository methods return typed values (`?Model`, `Collection`, `LengthAwarePaginator`)
- [ ] `RepositoryServiceProvider` registered in `config/app.php` providers array

---
> Customize for your Laravel version and auth package (Sanctum / Passport / tymon-jwt).
