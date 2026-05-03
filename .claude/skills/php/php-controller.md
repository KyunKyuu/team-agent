---
name: php-controller
language: php
layer: controller
---

# PHP Controller — Laravel

## Key Patterns

- Controllers are thin dispatchers: receive request, call service, return Resource
- Use `FormRequest` for validation — no `$request->validate()` inline
- Inject the Service (not the Repository) via constructor or method injection
- Map domain exceptions to HTTP responses in `app/Exceptions/Handler.php`, not in the controller
- One controller per resource; one action per method (store, show, update, destroy, index)

## Example

```php
// app/Http/Controllers/Api/V1/PolicyController.php
namespace App\Http\Controllers\Api\V1;

use App\Http\Controllers\Controller;
use App\Http\Requests\StorePolicyRequest;
use App\Http\Resources\PolicyResource;
use App\Http\Resources\PolicyCollection;
use App\Services\PolicyService;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;

class PolicyController extends Controller
{
    public function __construct(private readonly PolicyService $service) {}

    public function index(Request $request): PolicyCollection
    {
        $policies = $this->service->listActive((int) $request->query('per_page', 20));
        return new PolicyCollection($policies);
    }

    public function store(StorePolicyRequest $request): JsonResponse
    {
        $policy = $this->service->create($request->validated(), $request->user());
        return (new PolicyResource($policy))->response()->setStatusCode(201);
    }

    public function show(int $id): PolicyResource
    {
        return new PolicyResource($this->service->get($id));
    }

    public function destroy(int $id, Request $request): JsonResponse
    {
        $this->service->cancel($id, $request->user());
        return response()->json(null, 204);
    }
}
```

## Checklist

- [ ] Controller methods are 5 lines or fewer (receive → call service → return resource)
- [ ] `FormRequest` class used for all create/update actions; no inline validation
- [ ] Service injected in constructor; repository never referenced directly
- [ ] Domain exceptions not caught here — mapped to HTTP in `Handler.php`
- [ ] Response status codes set explicitly (201 for create, 204 for delete)
- [ ] No business logic, DB queries, or model instantiation in the controller
- [ ] Controller namespaced under version prefix (`Api\V1\`) for route versioning

---
> Customize for your Laravel version and auth package (Sanctum / Passport / tymon-jwt).
