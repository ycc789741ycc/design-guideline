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
