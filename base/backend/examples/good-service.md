# Example: Good Service Pattern

Illustrates layering (see [`../architecture.md`](../architecture.md)) and
error handling (see [`../../shared/error-handling.md`](../../shared/error-handling.md)).

```typescript
// domain/order.ts — no framework/DB dependency
export class Order {
  constructor(private items: OrderItem[]) {}

  calculateTotal(): Money {
    return this.items.reduce((sum, i) => sum.add(i.price), Money.zero());
  }
}

// application/cancel-order.ts — orchestration only
export class CancelOrder {
  constructor(
    private orders: OrderRepository,
    private payments: PaymentGateway,
  ) {}

  async execute(orderId: string): Promise<Result<void, CancelOrderError>> {
    const order = await this.orders.findById(orderId);
    if (!order) return Result.err(new OrderNotFoundError(orderId));
    if (!order.isCancellable()) {
      return Result.err(new OrderNotCancellableError(orderId, order.status));
    }

    await this.payments.refund(order.paymentId);
    order.markCancelled();
    await this.orders.save(order);

    return Result.ok();
  }
}

// presentation/order-controller.ts — maps to HTTP, no business logic
router.post("/orders/:id/cancel", async (req, res) => {
  const result = await cancelOrder.execute(req.params.id);
  if (result.isErr()) return res.status(toHttpStatus(result.error)).json({ error: result.error.toResponse() });
  return res.status(204).send();
});
```

Why this is good:
- Domain (`Order`) has zero framework dependencies — testable in isolation.
- `CancelOrder` orchestrates without containing business rules itself.
- Errors are typed and mapped to stable HTTP responses at the boundary, per
  the shared error-handling contract — no raw exceptions leaking out.
