---
name: php-service
language: php
layer: service
---

# PHP Service — Laravel

## Key Patterns

- Services own all business logic; controllers and models stay free of it
- Inject the repository interface (not the Eloquent class) via constructor
- Throw domain-specific exceptions (`PolicyNotFoundException`, `PolicyExpiredException`) — let a handler map them to HTTP responses
- Wrap multi-step writes in `DB::transaction()` to guarantee atomicity
- Return model instances or plain value objects; never return HTTP responses

## Example

```php
// app/Services/PolicyService.php
class PolicyService
{
    public function __construct(
        private readonly PolicyRepositoryInterface $policies,
        private readonly NotificationService $notifications,
    ) {}

    public function create(array $data, User $owner): Policy
    {
        if ($this->policies->findByPolicyNumber($data['policy_number'])) {
            throw new DuplicatePolicyNumberException($data['policy_number']);
        }

        return DB::transaction(function () use ($data, $owner) {
            $policy = new Policy(array_merge($data, ['user_id' => $owner->id]));
            $saved  = $this->policies->save($policy);
            $this->notifications->policyCreated($saved);
            return $saved;
        });
    }

    public function cancel(int $id, User $actor): Policy
    {
        $policy = $this->policies->findById($id)
            ?? throw new PolicyNotFoundException($id);

        if ($policy->status === PolicyStatus::Cancelled) {
            throw new PolicyAlreadyCancelledException($id);
        }

        Gate::forUser($actor)->authorize('cancel', $policy);

        $policy->status = PolicyStatus::Cancelled;
        return $this->policies->save($policy);
    }
}
```

## Checklist

- [ ] Service constructor injects interfaces, not concrete Eloquent classes
- [ ] All business rules (uniqueness, state transitions, authorization) live here, not in controllers or models
- [ ] Multi-step DB writes wrapped in `DB::transaction()`
- [ ] Domain exceptions thrown with meaningful names and messages
- [ ] No `response()`, `redirect()`, or HTTP status codes inside the service
- [ ] Return types declared on every public method
- [ ] Side effects (emails, events, queued jobs) dispatched from the service, not the controller

---
> Customize for your Laravel version and auth package (Sanctum / Passport / tymon-jwt).
