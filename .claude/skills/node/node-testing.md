---
name: node-testing
language: node
layer: testing
---

# Node Testing

## Key Patterns

- Unit tests: mock all I/O (DB, HTTP, time); test one class/function at a time
- Integration/HTTP tests: use `supertest` against the full Express app with a real test DB
- Arrange-Act-Assert structure in every test block
- Shared fixtures in `test/fixtures/` — avoid duplicating test data setup inline
- Clear DB state between tests; use transactions rolled back per test or `beforeEach` truncation

## Example

```ts
// test/unit/user.service.test.ts
import { UserService } from "../../src/services/user.service";
import { IUserRepository } from "../../src/repositories/user.repository";

const mockRepo: jest.Mocked<IUserRepository> = {
  findById: jest.fn(),
  findByEmail: jest.fn(),
  save: jest.fn(),
  delete: jest.fn(),
};

describe("UserService.create", () => {
  const svc = new UserService(mockRepo);

  beforeEach(() => jest.clearAllMocks());

  it("throws ConflictError when email already exists", async () => {
    mockRepo.findByEmail.mockResolvedValue({ id: "1", email: "a@b.com" } as any);
    await expect(svc.create({ email: "a@b.com", password: "pass1234", fullName: "A" }))
      .rejects.toThrow("Email already in use");
  });

  it("hashes password before saving", async () => {
    mockRepo.findByEmail.mockResolvedValue(null);
    mockRepo.save.mockResolvedValue({ id: "2" } as any);
    await svc.create({ email: "b@c.com", password: "pass1234", fullName: "B" });
    expect(mockRepo.save).toHaveBeenCalledWith(expect.objectContaining({ passwordHash: expect.any(String) }));
  });
});
```

## Checklist

- [ ] Unit tests import only the module under test; all deps are mocked
- [ ] `jest.clearAllMocks()` called in `beforeEach` to prevent state bleed
- [ ] HTTP tests use `supertest(app)` — no server `listen()` needed
- [ ] Test DB uses a separate `DATABASE_URL` from `.env.test`
- [ ] Each test has a single assertion focus — one reason to fail
- [ ] Error path tested for every service method (not just happy path)
- [ ] CI runs `jest --coverage`; coverage threshold enforced in `jest.config.ts`

---
> Customize this file for your specific stack (e.g., Sequelize vs TypeORM, Express vs Fastify).
