# Example: Backend Anti-Patterns

## Swallowed errors

```typescript
// ❌ Bad — error silently disappears
async function cancelOrder(id: string) {
  try {
    await orders.cancel(id);
  } catch (e) {
    console.log("something went wrong");
  }
}
```

```typescript
// ✅ Good — typed error propagated to caller
async function cancelOrder(id: string): Promise<Result<void, CancelOrderError>> {
  try {
    await orders.cancel(id);
    return Result.ok();
  } catch (e) {
    return Result.err(new CancelOrderError(id, e));
  }
}
```

## Business logic leaking into controllers

```typescript
// ❌ Bad — discount calculation, a domain rule, lives in the HTTP layer
router.post("/orders", async (req, res) => {
  let total = req.body.items.reduce((s, i) => s + i.price, 0);
  if (req.body.items.length > 10) total *= 0.9; // bulk discount — a domain rule
  const order = await db.orders.insert({ ...req.body, total });
  res.json(order);
});
```

```typescript
// ✅ Good — discount rule lives in the domain, controller just orchestrates
router.post("/orders", async (req, res) => {
  const result = await createOrder.execute(req.body);
  res.json(result);
});
```

## Reaching into a component's internals

```typescript
// ❌ Bad — the route imports the orders component's private repository
// src/api/routes/orders.ts
import { OrderRepository } from "../../bookstore/orders/order-repository";

router.get("/orders/:id", async (req, res) => {
  res.json(await orderRepository.findById(req.params.id));
});
```

```typescript
// ✅ Good — only the component's entry point is imported
// src/api/routes/orders.ts
import { GetOrder } from "../../bookstore/orders";

router.get("/orders/:id", async (req, res) => {
  const result = await getOrder.execute(req.params.id);
  if (result.isErr()) return res.status(toHttpStatus(result.error)).json({ error: result.error.toResponse() });
  return res.json(result.value);
});
```

## N+1 queries

```typescript
// ❌ Bad
const orders = await db.orders.findAll();
for (const order of orders) {
  order.customer = await db.customers.findById(order.customerId);
}
```

```typescript
// ✅ Good
const orders = await db.orders.findAllWithCustomers(); // single joined query
```
