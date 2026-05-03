---
name: php-testing
language: php
layer: testing
---

# PHP Testing — Laravel

## Key Patterns

- Feature tests cover the full HTTP stack (routes → controller → service → DB); unit tests cover isolated logic
- `RefreshDatabase` resets DB state per test — use `DatabaseTransactions` when tests must share a migrated schema for speed
- Mock the Service in controller tests using `$this->mock()`; never hit real external APIs in tests
- `assertJsonStructure` verifies shape; `assertJsonFragment` verifies specific values
- Use Pest's `it()` syntax for readable test descriptions; PHPUnit `test_` prefix is fine too

## Example

```php
// tests/Feature/Api/V1/PolicyTest.php  (Pest)
use App\Models\User;
use App\Models\Policy;
use App\Services\PolicyService;

uses(\Illuminate\Foundation\Testing\RefreshDatabase::class);

beforeEach(function () {
    $this->user = User::factory()->create();
    $this->actingAs($this->user, 'sanctum');
});

it('returns paginated active policies', function () {
    Policy::factory(3)->active()->for($this->user)->create();

    $this->getJson(route('v1.policies.index'))
         ->assertOk()
         ->assertJsonStructure([
             'data' => [['id', 'policy_number', 'status', 'coverage', 'expires_at']],
         ]);
});

it('creates a policy and returns 201', function () {
    $payload = Policy::factory()->make()->only(['policy_number', 'coverage', 'expires_at']);

    $this->postJson(route('v1.policies.store'), $payload)
         ->assertCreated()
         ->assertJsonFragment(['policy_number' => $payload['policy_number']]);

    $this->assertDatabaseHas('policies', ['policy_number' => $payload['policy_number']]);
});

it('returns 422 when policy_number is missing', function () {
    $this->postJson(route('v1.policies.store'), [])
         ->assertUnprocessable()
         ->assertJsonValidationErrors(['policy_number', 'coverage', 'expires_at']);
});

// Mocking the service in a unit-style feature test
it('delegates cancellation to PolicyService', function () {
    $policy  = Policy::factory()->create(['user_id' => $this->user->id]);
    $service = $this->mock(PolicyService::class);
    $service->shouldReceive('cancel')->once()->with($policy->id, \Mockery::type(User::class));

    $this->deleteJson(route('v1.policies.destroy', $policy))->assertNoContent();
});
```

## Checklist

- [ ] `RefreshDatabase` (or `DatabaseTransactions`) trait on every Feature test class
- [ ] Factories defined for every model; states (`active()`, `expired()`) cover common scenarios
- [ ] Auth set up with `actingAs($user, 'sanctum')` before protected route calls
- [ ] Both happy path and validation/error paths tested for each endpoint
- [ ] External services (payment, email, SMS) mocked — no real network calls in tests
- [ ] `assertDatabaseHas` / `assertDatabaseMissing` used to verify persistence side effects
- [ ] `php artisan test --coverage` run in CI; target ≥ 80% coverage on Service and Repository layers

---
> Customize for your Laravel version and auth package (Sanctum / Passport / tymon-jwt).
