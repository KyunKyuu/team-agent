---
name: php-resource
language: php
layer: presentation
---

# PHP Resource — Laravel

## Key Patterns

- `FormRequest` owns all validation rules and authorization; controllers stay thin
- `JsonResource` maps a model (or DTO) to the exact JSON shape the API contract requires
- Never expose raw model attributes — always go through a Resource
- `ResourceCollection` adds top-level meta (pagination, filters) to collection responses
- Use `whenLoaded()` for optional relationships to avoid N+1 queries

## Example

```php
// app/Http/Requests/StorePolicyRequest.php
class StorePolicyRequest extends FormRequest
{
    public function authorize(): bool
    {
        return $this->user()->can('create', Policy::class);
    }

    public function rules(): array
    {
        return [
            'policy_number' => ['required', 'string', 'unique:policies,policy_number'],
            'coverage'      => ['required', 'numeric', 'min:0'],
            'expires_at'    => ['required', 'date', 'after:today'],
            'meta'          => ['sometimes', 'array'],
        ];
    }
}

// app/Http/Resources/PolicyResource.php
class PolicyResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return [
            'id'            => $this->id,
            'policy_number' => $this->policy_number,
            'status'        => $this->status->value,
            'coverage'      => (float) $this->coverage,
            'expires_at'    => $this->expires_at->toIso8601String(),
            'user'          => new UserResource($this->whenLoaded('user')),
            'claims_count'  => $this->whenCounted('claims'),
        ];
    }
}

// app/Http/Resources/PolicyCollection.php
class PolicyCollection extends ResourceCollection
{
    public function toArray(Request $request): array
    {
        return [
            'data' => $this->collection,
            'meta' => ['total_active' => $this->collection->where('status', 'active')->count()],
        ];
    }
}
```

## Checklist

- [ ] `FormRequest` used for every mutating endpoint; no `$request->validate()` in controllers
- [ ] `authorize()` returns a meaningful policy/gate check, not a hardcoded `true`
- [ ] `JsonResource::toArray()` maps every field explicitly — no `parent::toArray()` leaking raw data
- [ ] `whenLoaded()` used for every relationship field to avoid accidental eager loading
- [ ] Dates serialized to ISO 8601 strings, not raw `Carbon` objects
- [ ] `ResourceCollection` used when adding envelope metadata alongside paginated results
- [ ] Validation error responses return HTTP 422 by default — do not override without reason

---
> Customize for your Laravel version and auth package (Sanctum / Passport / tymon-jwt).
