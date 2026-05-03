---
name: php-model
language: php
layer: model
---

# PHP Model — Laravel

## Key Patterns

- Declare `$fillable` (or `$guarded = []`) explicitly — never leave both unset
- Use `$casts` for type-safe attribute access (JSON columns, enums, dates, booleans)
- Relationships are the only logic allowed in a model; move all business logic to a Service
- Local scopes (`scopeActive`, `scopeByStatus`) keep query logic reusable without leaking into controllers
- Use `$hidden` to strip sensitive fields (passwords, tokens) from serialization automatically

## Example

```php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\SoftDeletes;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\Relations\HasMany;
use Illuminate\Database\Eloquent\Builder;

class Policy extends Model
{
    use SoftDeletes;

    protected $fillable = [
        'user_id', 'policy_number', 'status', 'coverage', 'meta', 'expires_at',
    ];

    protected $hidden = ['meta'];

    protected $casts = [
        'coverage'   => 'decimal:2',
        'meta'       => 'array',
        'expires_at' => 'datetime',
        'status'     => PolicyStatus::class, // backed enum
    ];

    // Relationships
    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }

    public function claims(): HasMany
    {
        return $this->hasMany(Claim::class);
    }

    // Local scopes — no business logic, only query constraints
    public function scopeActive(Builder $query): Builder
    {
        return $query->where('status', PolicyStatus::Active)
                     ->where('expires_at', '>', now());
    }

    public function scopeByStatus(Builder $query, PolicyStatus $status): Builder
    {
        return $query->where('status', $status);
    }
}
```

## Checklist

- [ ] `$fillable` declared; no mass-assignment vulnerabilities
- [ ] All non-string columns in `$casts` (JSON → `'array'`, booleans, decimals, dates)
- [ ] Sensitive attributes listed in `$hidden`
- [ ] Relationships defined as typed methods returning Eloquent `Relation` objects
- [ ] No HTTP, business, or validation logic inside the model
- [ ] Local scopes named `scope{Name}` and return `Builder`
- [ ] Soft deletes used where data retention is required (`SoftDeletes` trait + migration column)

---
> Customize for your Laravel version and auth package (Sanctum / Passport / tymon-jwt).
